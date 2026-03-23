---
ticketId: ENG-6
ticketTitle: "Too much repeated code : Using a validation library like Joi or Zod can indeed help improve code readability and provide a DRY code structure"
status: draft
createdAt: 2026-03-23T07:11:15.243Z
---

> Selected repository: `Zivl9090/node-express-realworld-example-app`
> Reasoning: This Node.js/Express backend repository contains the validation logic shown in the ticket that needs refactoring to use Joi or Zod for DRY code structure.

## What to do

Introduce **Joi** for request validation and centralize it in a `validate` middleware. Replace the manual field-checking pattern in `routes/api/users.js` (and any other routes doing the same thing) with declarative schemas. This keeps error shape consistent with the existing `{ errors: { field: [messages] } }` contract.

---

## Changes

### 1. `src/middleware/validate.js` — **new file**

A reusable Express middleware factory. Takes a Joi schema and validates `req.body` against it. On failure, transforms Joi's error output into the app's existing error shape and throws (or calls `next`) with status 422.

```js
// Public interface:
module.exports = (schema) => (req, res, next) => { ... }
```

Transformation logic:

- Iterate `error.details`
- Each detail has `context.key` (field name) and `message` (string)
- Strip Joi's surrounding quotes from messages
- Build `{ errors: { [key]: [message] } }` and return `res.status(422).json(...)`

Use `schema.validate(req.body, { abortEarly: false })` so all field errors surface at once, not just the first.

---

### 2. `src/validators/user.js` — **new file**

Joi schemas for user-related routes.

```js
const Joi = require('joi');

const register = Joi.object({
  user: Joi.object({
    email: Joi.string().trim().required(),
    username: Joi.string().trim().required(),
    password: Joi.string().trim().required(),
  }).required(),
});

const login = Joi.object({
  user: Joi.object({
    email: Joi.string().trim().required(),
    password: Joi.string().trim().required(),
  }).required(),
});

const update = Joi.object({
  user: Joi.object({
    email: Joi.string().trim(),
    username: Joi.string().trim(),
    password: Joi.string().trim(),
    image: Joi.string().allow('', null),
    bio: Joi.string().allow('', null),
  }).min(1).required(),
});

module.exports = { register, login, update };
```

Note the nested `user:` wrapper — confirm against the actual request body shape in `routes/api/users.js` before finalising. The RealWorld spec wraps all user payloads in a `user` key.

---

### 3. `routes/api/users.js` — **modify**

- `npm install joi` first.
- Import `validate` middleware and `userValidators`.
- On the `POST /users` (register) route: add `validate(userValidators.register)` before the handler. Remove the manual `if (!email)` / `if (!username)` / `if (!password)` block and the `.trim()` calls — Joi handles both.
- On the `POST /users/login` route: add `validate(userValidators.login)`.
- On the `PUT /user` route: add `validate(userValidators.update)`.

The handler bodies should shrink to just the business logic (hash password, create user, sign JWT, etc.).

---

### 4. `src/validators/article.js` — **new file** (if articles route has same pattern)

If `routes/api/articles.js` contains similar manual blank-checks, add an article schema here. Check the file — if it does, same treatment: schema + `validate` middleware on create/update routes.

---

## Gotchas

- **Error message format**: Joi default messages look like `"email" is not allowed to be empty`. Strip the surrounding quotes and ensure the message matches what tests expect. If existing tests assert on specific strings, check them before changing message text — or just assert on status code + field key presence.
- **Nested body shape**: RealWorld wraps bodies as `{ user: { ... } }` and `{ article: { ... } }`. Your Joi schema must mirror this nesting exactly, or validation will always fail.
- **`.trim()` side effect**: The existing code trims and then uses the trimmed value. Joi's `.trim()` modifier mutates the validated value — use `const { error, value } = schema.validate(req.body, ...)` and replace `req.body` with `value` in the middleware so trimmed values flow into the handler.
- **`abortEarly: false`**: Without this, only the first error is returned. Add it so multi-field errors work like the existing per-field error shape implies.
- **Do not change the User or Article Mongoose models** — this refactor is purely in the route/middleware layer.
- **Install Joi**: `npm install joi` — add to `package.json` dependencies, not devDependencies.

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.
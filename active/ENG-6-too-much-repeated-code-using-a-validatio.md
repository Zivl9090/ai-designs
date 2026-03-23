---
ticketId: ENG-6
ticketTitle: "Too much repeated code : Using a validation library like Joi or Zod can indeed help improve code readability and provide a DRY code structure"
status: draft
createdAt: 2026-03-23T09:17:59.695Z
---

> Selected repository: `Zivl9090/node-express-realworld-example-app`
> Reasoning: This Node.js/Express backend repository contains the validation code shown in the ticket that needs refactoring to use Joi or Zod for DRY validation logic.

## What to do

Introduce **Joi** as the validation library and replace the manual field-by-field validation in user routes with declarative schemas + a shared `validate` middleware. The middleware maps Joi errors to the existing `{ errors: { field: [messages] } }` 422 shape so nothing changes for callers.

Joi is the right pick here — the ticket shows a Joi example, it's battle-tested with Express, and Zod would require more wiring with Mongoose-style error shapes.

---

## Changes

### 1. `src/middleware/validate.js` — **new file**

Purpose: Express middleware factory that runs a Joi schema against `req.body` and calls `next()` on success or throws a 422 in the existing error shape on failure.

```js
// Public interface:
// validateBody(schema) → Express middleware (req, res, next)
```

Implementation notes:
- Use `schema.validate(req.body, { abortEarly: false })` so all errors surface at once.
- Map `error.details` to `{ errors: { [field]: [message, ...] } }` where `field` is `detail.path[0]`.
- Call `res.status(422).json(...)` on failure; call `next()` on success.
- Strip Joi's default quoting from messages (Joi wraps field names in quotes by default — pass `{ errors: { wrap: { label: false } } }` to `Joi.object()` or handle in the message mapping).

### 2. `src/validators/user.js` — **new file**

Joi schemas for user operations:

```js
const registerSchema = Joi.object({
  email:    Joi.string().email().required(),
  username: Joi.string().required(),
  password: Joi.string().required(),
});

const loginSchema = Joi.object({
  email:    Joi.string().email().required(),
  password: Joi.string().required(),
});

const updateSchema = Joi.object({
  email:    Joi.string().email(),
  username: Joi.string(),
  password: Joi.string(),
  image:    Joi.string().uri().allow('', null),
  bio:      Joi.string().allow('', null),
}).min(1); // at least one field required on update
```

Export all three as named exports.

### 3. `routes/api/users.js` — **modify**

- Import `validateBody` from `../../middleware/validate`.
- Import `{ registerSchema, loginSchema, updateSchema }` from `../../validators/user`.
- Add `validateBody(registerSchema)` as middleware on `POST /users` (registration), before the route handler.
- Add `validateBody(loginSchema)` on `POST /users/login`.
- Add `validateBody(updateSchema)` on `PUT /user`.
- **Remove** the manual `if (!email)` / `if (!username)` / `if (!password)` blocks and the `.trim()` calls that exist purely for blank-checking. Keep any business-logic that happens after validation (DB lookup, password hashing, JWT generation).

---

## Gotchas

- The existing error shape uses arrays for messages: `{ errors: { email: ["can't be blank"] } }`. Joi's default messages differ — map them. For a required field, Joi emits `"email" is required` — strip the quotes and preserve the text, or override with `.messages({ 'any.required': "can't be blank" })` on each field. Using `.messages()` is cleaner.
- `req.body` in the registration route is nested: the client sends `{ user: { email, ... } }`. Check whether the route destructures `req.body.user` — if so, the schema should validate that object, not `req.body`. Pass `req.body.user` to the validator or adjust the schema to wrap in `Joi.object({ user: registerSchema })`. Look at the existing handler to confirm the shape before deciding.
- Joi `.email()` rejects addresses that the existing code accepted if they were non-blank. If tests use fake emails like `test@test`, add `.email({ tlds: { allow: false } })` to avoid false failures.
- Do **not** modify `models/User.js` or the Mongoose schema — validation stays in the route layer.
- Install Joi first: `npm install joi`. Confirm it's not already a dependency before adding.

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.
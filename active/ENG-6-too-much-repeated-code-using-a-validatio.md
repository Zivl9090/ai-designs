---
ticketId: ENG-6
ticketTitle: "Too much repeated code : Using a validation library like Joi or Zod can indeed help improve code readability and provide a DRY code structure"
status: draft
createdAt: 2026-03-22T17:07:36.773Z
---

> Selected repository: `Zivl9090/node-express-realworld-example-app`
> Reasoning: This Node.js/Express backend repository contains the validation code shown in the ticket (Joi schemas and manual validation checks) that needs to be refactored to use a validation library consistently.

## What to do

Introduce **Joi** as the validation library and centralise all request-body validation behind a single `validate` middleware. Replace the manual field-by-field checks in the user and article routes with Joi schemas. The error shape must stay `{ errors: { field: ["message"] } }` with status 422.

---

## Changes

### 1. Install dependency

```bash
npm install joi
```

---

### 2. New file: `src/middleware/validate.js`

A generic Express middleware factory that takes a Joi schema, validates `req.body`, and throws/returns a 422 in the existing error shape on failure.

```js
const validate = (schema) => (req, res, next) => {
  const { error } = schema.validate(req.body, { abortEarly: false });
  if (!error) return next();

  const errors = {};
  error.details.forEach(({ path, message }) => {
    const field = path[0];
    errors[field] = [message.replace(/['"]/g, '')];
  });
  return res.status(422).json({ errors });
};

module.exports = validate;
```

- `abortEarly: false` collects all failures at once.
- `path[0]` maps to the top-level field name, matching the existing error shape.

---

### 3. New file: `src/middleware/schemas.js`

All Joi schemas live here. Define one object per request type.

```js
const Joi = require('joi');

const userRegistration = Joi.object({
  user: Joi.object({
    email: Joi.string().email().trim().required(),
    username: Joi.string().trim().required(),
    password: Joi.string().trim().required(),
  }).required(),
});

const userLogin = Joi.object({
  user: Joi.object({
    email: Joi.string().trim().required(),
    password: Joi.string().trim().required(),
  }).required(),
});

const userUpdate = Joi.object({
  user: Joi.object({
    email: Joi.string().email().trim(),
    username: Joi.string().trim(),
    password: Joi.string().trim(),
    image: Joi.string().uri().allow('', null),
    bio: Joi.string().allow('', null),
  }).min(1),
});

const articleCreate = Joi.object({
  article: Joi.object({
    title: Joi.string().required(),
    description: Joi.string().required(),
    body: Joi.string().required(),
    tagList: Joi.array().items(Joi.string()),
  }).required(),
});

const articleUpdate = Joi.object({
  article: Joi.object({
    title: Joi.string(),
    description: Joi.string(),
    body: Joi.string(),
    tagList: Joi.array().items(Joi.string()),
  }).min(1),
});

const commentCreate = Joi.object({
  comment: Joi.object({
    body: Joi.string().required(),
  }).required(),
});

module.exports = {
  userRegistration,
  userLogin,
  userUpdate,
  articleCreate,
  articleUpdate,
  commentCreate,
};
```

---

### 4. `src/routes/api/users.js`

- Import `validate` and the schemas.
- Add `validate(schemas.userRegistration)` before the `POST /users` handler body.
- Add `validate(schemas.userLogin)` before the `POST /users/login` handler body.
- Add `validate(schemas.userUpdate)` before the `PUT /user` handler body.
- **Remove** all manual `if (!email)`, `if (!username)`, `if (!password)` checks and the corresponding `.trim()` assignments — Joi handles trimming and presence.

Route wiring example:

```js
const validate = require('../../middleware/validate');
const schemas = require('../../middleware/schemas');

router.post('/users', validate(schemas.userRegistration), async (req, res, next) => { ... });
router.post('/users/login', validate(schemas.userLogin), async (req, res, next) => { ... });
router.put('/user', auth.required, validate(schemas.userUpdate), async (req, res, next) => { ... });
```

Inside the handlers, read from `req.body.user` as before — Joi validation doesn't reshape the object.

---

### 5. `src/routes/api/articles.js`

- Add `validate(schemas.articleCreate)` before the `POST /articles` handler.
- Add `validate(schemas.articleUpdate)` before the `PUT /articles/:slug` handler.
- Remove any manual required-field checks for `title`, `description`, `body`.

---

### 6. `src/routes/api/comments.js`

- Add `validate(schemas.commentCreate)` before the `POST /articles/:slug/comments` handler.
- Remove any manual check for `body` being blank.

---

## Gotchas

- **Nested body shape**: The API wraps payloads — `{ user: { email, ... } }`, `{ article: { ... } }`, `{ comment: { ... } }`. The Joi schemas must mirror this nesting, and the validate middleware must validate `req.body` (the whole envelope), not `req.body.user`. If you validate only the inner object, the outer wrapper is unchecked.
- **Error path for nested objects**: With the nested shape, `error.details[].path` will be `['user', 'email']` not `['email']`. Change the middleware's path extraction to `path[path.length - 1]` (the leaf field name) so the error response stays `{ errors: { email: [...] } }`.
- **Trimming side-effect**: The existing code does `input.email?.trim()` before using the value. Joi's `.trim()` modifier validates the trimmed value but does **not** mutate `req.body`. Either keep one explicit `.trim()` in the handler before DB calls, or set `convert: true` in `schema.validate(req.body, { abortEarly: false, convert: true })` — `convert: true` is Joi's default so trimming will mutate the validated value only if you use the returned `value` object. Switch handlers to use the `value` returned by Joi if you want auto-trimming; otherwise keep the `.trim()` calls in the handler. The simplest fix: use the `value` from the middleware by attaching it — `req.body = value` — before calling `next()`.
- **Do not change the User or Article Mongoose models** — this refactor is purely at the route/middleware layer.
- **Existing tests** validate the 422 error shape — run `npm test` after changes to confirm the shape hasn't drifted.

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.
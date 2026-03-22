---
ticketId: ENG-6
ticketTitle: "Too much repeated code : Using a validation library like Joi or Zod can indeed help improve code readability and provide a DRY code structure"
status: approved
createdAt: 2026-03-22T17:01:04.688Z
---

> Selected repository: `Zivl9090/node-express-realworld-example-app`
> Reasoning: This is a Node.js/Express backend repository that contains the validation code shown in the ticket (Joi validation for password updates and field validation), making it the direct target for refactoring to use a validation library like Zod.

## What to do

Introduce **Joi** as the validation library and replace all manual field-checking in the user routes with a shared `validate` middleware. Joi is already referenced in the ticket example and fits Express middleware patterns naturally.

---

## Changes

### 1. `package.json`

Add `joi` as a dependency.

```bash
npm install joi
```

---

### 2. New file: `src/middleware/validate.js`

Purpose: generic Express middleware factory that validates `req.body` against a Joi schema and throws a 422 in the RealWorld error shape if validation fails.

```js
const validate = (schema) => (req, res, next) => {
  const { error, value } = schema.validate(req.body, {
    abortEarly: false,   // collect all errors, not just the first
    allowUnknown: true,  // don't reject fields not in the schema
    stripUnknown: false,
  });

  if (!error) {
    req.body = value;    // use the coerced/trimmed values
    return next();
  }

  const errors = {};
  error.details.forEach(({ path, message }) => {
    const field = path[0];
    errors[field] = errors[field] || [];
    errors[field].push(message.replace(/['"]/g, ''));
  });

  return res.status(422).json({ errors });
};

module.exports = validate;
```

---

### 3. New file: `src/middleware/schemas/user.schemas.js`

Purpose: Joi schemas that mirror the existing manual checks in user-related route handlers.

```js
const Joi = require('joi');

const register = Joi.object({
  email:    Joi.string().trim().email().required()
              .messages({ 'any.required': "can't be blank", 'string.empty': "can't be blank" }),
  username: Joi.string().trim().required()
              .messages({ 'any.required': "can't be blank", 'string.empty': "can't be blank" }),
  password: Joi.string().trim().required()
              .messages({ 'any.required': "can't be blank", 'string.empty': "can't be blank" }),
}).unknown(true);

const login = Joi.object({
  email:    Joi.string().trim().email().required()
              .messages({ 'any.required': "can't be blank", 'string.empty': "can't be blank" }),
  password: Joi.string().trim().required()
              .messages({ 'any.required': "can't be blank", 'string.empty': "can't be blank" }),
}).unknown(true);

const update = Joi.object({
  email:    Joi.string().trim().email(),
  username: Joi.string().trim(),
  password: Joi.string().trim(),
  image:    Joi.string().allow('', null),
  bio:      Joi.string().allow('', null),
}).unknown(true);

module.exports = { register, login, update };
```

---

### 4. `src/routes/api/users.js`

**What to change:**

1. Import `validate` and the `user.schemas`.
2. Add `validate(schemas.register)` as middleware on `POST /users` before the handler.
3. Add `validate(schemas.login)` as middleware on `POST /users/login` before the handler.
4. Add `validate(schemas.update)` as middleware on `PUT /user` before the handler.
5. **Remove** the manual `if (!email)` / `if (!username)` / `if (!password)` blocks and the corresponding `trim()` calls from inside the handlers — Joi's coercion via `stripUnknown: false` + `trim()` in schema means `req.body` already has clean values after the middleware runs.

Example before/after for the register handler:

```js
// BEFORE
router.post('/users', async (req, res, next) => {
  const email    = req.body.user?.email?.trim();
  const username = req.body.user?.username?.trim();
  const password = req.body.user?.password?.trim();
  if (!email)    throw new HttpException(422, { errors: { email:    ["can't be blank"] } });
  if (!username) throw new HttpException(422, { errors: { username: ["can't be blank"] } });
  if (!password) throw new HttpException(422, { errors: { password: ["can't be blank"] } });
  ...
});

// AFTER
router.post('/users', validate(schemas.register), async (req, res, next) => {
  const { email, username, password } = req.body.user ?? req.body;
  ...
});
```

> Adjust the destructuring path (`req.body.user` vs `req.body`) to match whatever the actual handlers use — inspect the existing code before removing anything.

---

## Gotchas

- **RealWorld wraps inputs in a `user` key** (`{ "user": { "email": "...", ... } }`). Check whether the existing handlers destructure from `req.body.user` or `req.body` directly. If it's `req.body.user`, pass `req.body.user` into `validate` or adjust the middleware to unwrap it: `schema.validate(req.body.user ?? req.body, ...)`. Pick one convention and be consistent.
- **Error message format must match exactly.** Existing tests likely assert `{ errors: { email: ["can't be blank"] } }`. The `.messages()` calls on each field must produce that exact string. Run the test suite after wiring up to verify.
- **Do not touch `models/User.js`** — Mongoose-level validation is separate and should stay unchanged.
- **`allowUnknown: true`** is intentional — the RealWorld body may include fields the schema doesn't enumerate (e.g. `demo`, `image`, `bio` on register), and rejecting unknown fields would break the API contract.

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.
---
ticketId: ENG-6
ticketTitle: "Too much repeated code : Using a validation library like Joi or Zod can indeed help improve code readability and provide a DRY code structure"
status: draft
createdAt: 2026-03-26T17:24:11.757Z
---

> Selected repository: `Zivl9090/node-express-realworld-example-app`
> Reasoning: This backend repository contains the validation code shown in the ticket and needs refactoring to use Joi/Zod to eliminate repeated validation logic.

## What to do

Introduce **Joi** as a validation layer and replace the scattered manual field checks in the user-related route handlers with a shared `validate` middleware. Joi is already referenced in the ticket and is the better fit here (Zod requires TypeScript types to be useful; this is a plain JS Express app).

---

## Changes

### 1. `package.json`

Add `joi` as a dependency:

```bash
npm install joi
```

---

### 2. New file: `src/middleware/validate.js`

Purpose: a generic Express middleware factory that validates `req.body` (or other parts of the request) against a Joi schema and returns the project's standard 422 error shape on failure.

```js
const validate = (schema) => (req, res, next) => {
  const { error, value } = schema.validate(req.body, {
    abortEarly: false,   // collect all errors, not just first
    stripUnknown: true,  // drop fields not in schema
  });

  if (!error) {
    req.body = value;    // use sanitised/coerced values downstream
    return next();
  }

  const errors = {};
  for (const detail of error.details) {
    const key = detail.path[0] ?? 'base';
    if (!errors[key]) errors[key] = [];
    errors[key].push(detail.message.replace(/['"]/g, ''));
  }

  return res.status(422).json({ errors });
};

module.exports = validate;
```

---

### 3. New file: `src/validators/user.js`

Purpose: all Joi schemas for user-facing inputs. Export named schemas; routes import what they need.

```js
const Joi = require('joi');

const register = Joi.object({
  user: Joi.object({
    email:    Joi.string().email().trim().required().messages({
      'string.empty': "can't be blank",
      'any.required': "can't be blank",
    }),
    username: Joi.string().trim().required().messages({
      'string.empty': "can't be blank",
      'any.required': "can't be blank",
    }),
    password: Joi.string().min(1).required().messages({
      'string.empty': "can't be blank",
      'any.required': "can't be blank",
    }),
  }).required(),
});

const login = Joi.object({
  user: Joi.object({
    email:    Joi.string().email().trim().required().messages({
      'string.empty': "can't be blank",
      'any.required': "can't be blank",
    }),
    password: Joi.string().required().messages({
      'string.empty': "can't be blank",
      'any.required': "can't be blank",
    }),
  }).required(),
});

const update = Joi.object({
  user: Joi.object({
    email:    Joi.string().email().trim(),
    username: Joi.string().trim(),
    password: Joi.string().min(1),
    image:    Joi.string().uri().allow('', null),
    bio:      Joi.string().allow('', null),
  }).required(),
});

module.exports = { register, login, update };
```

---

### 4. `src/routes/api/users.js`

- Import `validate` from `../../middleware/validate` and the three schemas from `../../validators/user`.
- **Remove** all manual `if (!email)`, `if (!username)`, `if (!password)` checks and the manual `.trim()` calls at the top of each handler (the schema + middleware handles them now).
- Wire the middleware **before** each handler:

```js
const validate       = require('../../middleware/validate');
const userSchemas    = require('../../validators/user');

// POST /users  (register)
router.post('/', validate(userSchemas.register), async (req, res, next) => { ... });

// POST /users/login
router.post('/login', validate(userSchemas.login), async (req, res, next) => { ... });

// PUT /user   (update — auth.required stays, validate comes after)
router.put('/', auth.required, validate(userSchemas.update), async (req, res, next) => { ... });
```

After the middleware runs, `req.body` already contains trimmed, validated values — use them directly (e.g. `const { email, username, password } = req.body.user`).

---

## Gotchas

- **Error message format**: existing tests likely assert `{ errors: { email: ["can't be blank"] } }`. The custom `.messages()` in the schemas above reproduce those exact strings. Verify against the test suite before tweaking.
- **Nested `user` key**: the RealWorld spec wraps all user payloads in `{ user: { ... } }`. The Joi schema must mirror this nesting; the `validate` middleware validates `req.body` which includes the wrapper. Access `req.body.user.*` in handlers as before.
- **`update` schema is partial**: all fields are optional (PATCH semantics). Do not add `.required()` to any field in `update`.
- **Do not touch `models/User.js`** — Mongoose-level uniqueness errors (duplicate email/username) are already handled by the existing error handler; no change needed there.
- **`stripUnknown: true`** in the middleware will silently drop unknown top-level keys. This is intentional and safe for these endpoints, but note it if debugging unexpected missing fields.

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.
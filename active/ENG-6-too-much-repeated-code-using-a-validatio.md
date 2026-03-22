---
ticketId: ENG-6
ticketTitle: "Too much repeated code : Using a validation library like Joi or Zod can indeed help improve code readability and provide a DRY code structure"
status: draft
createdAt: 2026-03-22T16:43:33.414Z
---

> Selected repository: `Zivl9090/node-express-realworld-example-app`
> Reasoning: This Node.js + Express backend repository contains the validation code shown in the ticket that needs refactoring to use Joi/Zod instead of manual validation checks.

## What to do

Introduce **Joi** as the validation library and centralize all request validation behind a reusable `validate` middleware. Replace the inline field-by-field checks in user-related route handlers with Joi schemas. All validation errors must continue to emit `{ errors: { field: ["message"] } }` with status 422.

## Changes

### 1. Install Joi

```bash
npm install joi
```

---

### 2. New file: `src/middleware/validate.js`

Express middleware factory that takes a Joi schema, validates `req.body`, and forwards a 422 error in the existing error shape if validation fails.

```js
const validate = (schema) => (req, res, next) => {
  const { error, value } = schema.validate(req.body, {
    abortEarly: false,   // collect all errors, not just the first
    stripUnknown: true,  // drop fields not in schema
  });

  if (!error) {
    req.body = value;    // replace body with coerced/trimmed values
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

Joi schemas for every user-facing body payload. Keep them in one place so routes just import what they need.

```js
const Joi = require('joi');

const register = Joi.object({
  user: Joi.object({
    email: Joi.string().email().trim().required().messages({
      'string.empty': "can't be blank",
      'any.required': "can't be blank",
      'string.email': 'is invalid',
    }),
    username: Joi.string().trim().required().messages({
      'string.empty': "can't be blank",
      'any.required': "can't be blank",
    }),
    password: Joi.string().min(8).required().messages({
      'string.empty': "can't be blank",
      'any.required': "can't be blank",
      'string.min': 'is too short (minimum 8 characters)',
    }),
  }).required(),
});

const login = Joi.object({
  user: Joi.object({
    email: Joi.string().email().trim().required().messages({
      'string.empty': "can't be blank",
      'any.required': "can't be blank",
      'string.email': 'is invalid',
    }),
    password: Joi.string().required().messages({
      'string.empty': "can't be blank",
      'any.required': "can't be blank",
    }),
  }).required(),
});

const update = Joi.object({
  user: Joi.object({
    email: Joi.string().email().trim().messages({ 'string.email': 'is invalid' }),
    username: Joi.string().trim(),
    password: Joi.string().min(8).messages({
      'string.min': 'is too short (minimum 8 characters)',
    }),
    image: Joi.string().uri().allow('', null),
    bio: Joi.string().allow('', null),
  }).required(),
});

module.exports = { register, login, update };
```

> Adjust minimum password length or other constraints to match whatever the existing code enforces — the above uses 8 as a sensible default. If the current code has no minimum, remove `min(8)`.

---

### 4. Modify `src/routes/api/users.js`

Remove all manual `if (!email)` / `if (!username)` / `if (!password)` checks and `.trim()` calls. Wire the `validate` middleware and schemas onto the three affected routes.

**POST /users** (registration):
```js
router.post('/users', validate(userSchemas.register), async (req, res, next) => { … });
```

**POST /users/login**:
```js
router.post('/users/login', validate(userSchemas.login), async (req, res, next) => { … });
```

**PUT /user** (update, already behind `auth.required`):
```js
router.put('/user', auth.required, validate(userSchemas.update), async (req, res, next) => { … });
```

Inside each handler, `req.body.user` is already validated and trimmed — delete the redundant destructuring-and-check blocks. Keep all logic that calls Mongoose / builds the response unchanged.

Imports to add at the top of `users.js`:
```js
const validate = require('../../middleware/validate');
const userSchemas = require('../../middleware/schemas/user.schemas');
```

---

## Gotchas

- **Nested body shape**: the RealWorld spec wraps payloads in `{ user: { … } }`. The Joi schemas above reflect this. The `validate` middleware validates `req.body`, so the top-level schema key must be `user`. Inside handlers, field access stays `req.body.user.email`.
- **Error message format**: existing tests likely assert `{ errors: { email: ["can't be blank"] } }`. Joi's default messages include quotes around field names — the `replace(/['"]/g, '')` in the middleware strips them. Verify test assertions still pass.
- **`stripUnknown`**: setting this strips `demo` or other unknown fields automatically; no need to handle them explicitly in the handler.
- **`update` route fields are all optional**: Joi `.required()` should NOT be added to `update` schema fields — a partial update sending only `bio` is valid.
- **Do not change `routes/auth.js`** or the User/Article/Comment Mongoose models.
- **Other routes** (articles, comments, profiles) are out of scope for this ticket. Don't add Joi schemas for them unless the ticket is expanded.

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.
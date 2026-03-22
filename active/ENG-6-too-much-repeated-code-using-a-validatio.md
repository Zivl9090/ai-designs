---
ticketId: ENG-6
ticketTitle: "Too much repeated code : Using a validation library like Joi or Zod can indeed help improve code readability and provide a DRY code structure"
status: draft
createdAt: 2026-03-22T16:31:32.066Z
---

> Selected repository: `Zivl9090/node-express-realworld-example-app`
> Reasoning: This backend repository contains the validation code shown in the ticket (Joi schema for updatePassword and manual validation checks) that needs to be refactored to use a validation library like Joi or Zod to reduce code duplication.

## What to do

Introduce **Joi** as a validation layer and replace all manual field-checking in user-facing route handlers with a shared `validate` middleware. Joi is already referenced in the ticket example and is well-suited to the Express + object-shape validation pattern here.

---

## Changes

### 1. Install Joi

```bash
npm install joi
```

---

### 2. New file: `src/middleware/validate.js`

A generic Express middleware factory that takes a Joi schema and returns a middleware function. On failure it formats errors into the existing `{ errors: { field: [messages] } }` shape with 422.

```js
const Joi = require('joi');

/**
 * @param {Joi.ObjectSchema} schema - Joi schema with optional `body`, `query`, `params` keys
 * @returns Express middleware
 */
function validate(schema) {
  return (req, res, next) => {
    const toValidate = {};
    if (schema.body) toValidate.body = req.body;
    if (schema.params) toValidate.params = req.params;
    if (schema.query) toValidate.query = req.query;

    const { error } = Joi.object(schema).validate(toValidate, { abortEarly: false });
    if (!error) return next();

    const errors = {};
    for (const detail of error.details) {
      // detail.path is e.g. ['body', 'email'] — strip the top-level key
      const field = detail.path.slice(1).join('.');
      if (!errors[field]) errors[field] = [];
      errors[field].push(detail.message.replace(/['"]/g, ''));
    }
    return res.status(422).json({ errors });
  };
}

module.exports = validate;
```

---

### 3. New file: `src/middleware/schemas/userSchemas.js`

Centralise all user-related Joi schemas here.

```js
const Joi = require('joi');

const register = {
  body: Joi.object({
    user: Joi.object({
      email: Joi.string().trim().email().required().messages({
        'string.empty': "can't be blank",
        'any.required': "can't be blank",
      }),
      username: Joi.string().trim().required().messages({
        'string.empty': "can't be blank",
        'any.required': "can't be blank",
      }),
      password: Joi.string().trim().required().messages({
        'string.empty': "can't be blank",
        'any.required': "can't be blank",
      }),
    }).required(),
  }),
};

const login = {
  body: Joi.object({
    user: Joi.object({
      email: Joi.string().trim().email().required().messages({
        'string.empty': "can't be blank",
        'any.required': "can't be blank",
      }),
      password: Joi.string().trim().required().messages({
        'string.empty': "can't be blank",
        'any.required': "can't be blank",
      }),
    }).required(),
  }),
};

const update = {
  body: Joi.object({
    user: Joi.object({
      email: Joi.string().trim().email().optional(),
      username: Joi.string().trim().optional(),
      password: Joi.string().trim().optional(),
      image: Joi.string().uri().allow('', null).optional(),
      bio: Joi.string().allow('', null).optional(),
    }).required(),
  }),
};

module.exports = { register, login, update };
```

---

### 4. Modify: `src/routes/api/users.js`

Wire the middleware into the three routes. Remove all manual `if (!email)` / `if (!username)` / `if (!password)` blocks and the `.trim()` calls that precede them — Joi's `.trim()` modifier handles that.

**Registration route** (`POST /users`):
```js
const validate = require('../../middleware/validate');
const userSchemas = require('../../middleware/schemas/userSchemas');

router.post('/', validate(userSchemas.register), async (req, res, next) => {
  // req.body.user.email / .username / .password are guaranteed present and trimmed
  // remove the manual blank-checks that were here
  ...
});
```

**Login route** (`POST /users/login`):
```js
router.post('/login', validate(userSchemas.login), async (req, res, next) => {
  // remove manual blank-checks
  ...
});
```

**Update route** (`PUT /user`):
```js
router.put('/', auth.required, validate(userSchemas.update), async (req, res, next) => {
  // all fields optional; remove any manual presence checks
  ...
});
```

---

## Gotchas

- **Body shape**: the Conduit API wraps payloads in a `user` key — `{ user: { email, password, ... } }`. The schema must reflect this nesting; validate `body.user.*` not `body.*` directly.
- **Error path stripping**: Joi reports paths like `['body', 'user', 'email']`. The middleware above slices off the first segment (`body`) so the error key becomes `user.email`. If the frontend expects just `email`, slice two segments (`detail.path.slice(2)`). Check against the existing error responses in the test suite before deciding.
- **`.trim()` in Joi vs in handler**: once Joi validates with `.trim()`, the trimmed value is in `error`-free result object but **not mutated onto `req.body`** unless you use `{ convert: true }` (the default) and read from the Joi result. The easiest fix is to pass `{ convert: true }` (already default) and assign `req.body = value` from the Joi result inside the middleware before calling `next()`. Add this to the `validate` middleware:
  ```js
  const { error, value } = Joi.object(schema).validate(toValidate, { abortEarly: false });
  if (!error) {
    if (schema.body) req.body = value.body;
    return next();
  }
  ```
- **Do not change the User or Article Mongoose models** — this ticket is routing/middleware only.
- **Existing tests** hit the real DB and check exact error shapes — run `npm test` after each route change to confirm the 422 response format hasn't drifted.

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.
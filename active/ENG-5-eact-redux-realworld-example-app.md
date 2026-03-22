---
ticketId: ENG-5
ticketTitle: "eact-redux-realworld-example-app"
status: draft
createdAt: 2026-03-22T14:12:18.060Z
---

> Selected repository: `Zivl9090/react-redux-realworld-example-app`
> Reasoning: This is the React frontend application mentioned in the ticket that needs to be modified to accept API_ROOT from environment variables instead of hardcoding it in src/agent.js.

# Design Document: ENG-5 — Runtime-Configurable `API_ROOT` for Production Builds

---

## Classification

- **Type:** feature
- **Scope:** single-repo
- **Complexity:** low
- **Estimated effort:** 1–2 hours

---

## Design

### Recommended Approach

The core problem is that Create React App (CRA) bakes environment variables into the JavaScript bundle at **build time** via `process.env.REACT_APP_*`. Once the bundle is compiled, those values are frozen into the static JS files. This is fundamentally incompatible with the 12-factor "build once, promote through environments" pattern.

The standard solution for CRA apps running in containers is to **inject a runtime configuration object into `index.html` via a server-side entrypoint script**, and have `src/agent.js` read from that object instead of (or falling back from) `process.env`.

#### Concrete implementation:

**Step 1 — Add a `window._env_` config object placeholder**

Create a file `public/env-config.js`:
```javascript
window._env_ = {
  API_ROOT: 'https://conduit.productionready.io/api'
};
```
This file ships with the build but serves as a default. It is the only file that needs to change per environment at container startup — not a rebuild.

**Step 2 — Load it in `public/index.html`**

Add before the closing `</head>` tag in `public/index.html`:
```html
<script src="%PUBLIC_URL%/env-config.js"></script>
```

**Step 3 — Add a container entrypoint script**

Create `docker-entrypoint.sh` (or equivalent OpenShift init script):
```bash
#!/bin/sh
# Overwrite env-config.js at container startup using real env vars
cat > /usr/share/nginx/html/env-config.js <<EOF
window._env_ = {
  API_ROOT: "${API_ROOT:-https://conduit.productionready.io/api}"
};
EOF
exec "$@"
```
This script runs once when the container starts, before nginx serves requests. It writes the correct `API_ROOT` value (from the container's environment) into the static file.

**Step 4 — Update `src/agent.js`**

Currently in `src/agent.js` (line ~3):
```javascript
const API_ROOT = 'https://conduit.productionready.io/api';
```

Change to:
```javascript
const API_ROOT =
  (window._env_ && window._env_.API_ROOT) ||
  process.env.REACT_APP_BACKEND_URL ||
  'https://conduit.productionready.io/api';
```

This preserves backward compatibility:
- `window._env_.API_ROOT` is used when injected at container runtime (production/staging containers)
- `REACT_APP_BACKEND_URL` continues to work for local development (`npm start`)
- The hardcoded default is the final fallback

---

### Alternatives Considered

**1. Build-time `REACT_APP_BACKEND_URL` only**
Already partially in place per `CLAUDE.md`. Works for local dev but is incompatible with 12-factor: you must rebuild the image per environment. Rejected because it violates the explicit constraint.

**2. Per-environment Dockerfiles / build args**
Building separate images per environment (`docker build --build-arg REACT_APP_BACKEND_URL=...`). This is the most common anti-pattern the ticket is trying to escape. Rejected for the same reason as above.

**3. Nginx `sub_filter` to rewrite the placeholder at runtime**
Nginx can do string substitution in served files. This works but requires nginx configuration changes and is fragile (string matching on bundle content). The `env-config.js` approach is simpler and explicit.

**4. Fetch a `/config.json` from the backend at app boot**
The app could `fetch('/config.json')` on startup and read `API_ROOT` from the response. This requires making `agent.js` asynchronous before any requests can be made, which is a significant structural change and has a chicken-and-egg problem (which server do you fetch config from?). Rejected as disproportionately complex.

**5. CRA's `REACT_APP_*` with `NODE_ENV=production` rebuild on deploy**
Some teams do a "build step" as part of their deploy pipeline. The ticket explicitly rules this out ("built once then promoted").

---

### Key Assumptions

1. The app is served as static files from a container (e.g., nginx), so the entrypoint script can overwrite `env-config.js` before the server starts accepting requests.
2. There is a `Dockerfile` or equivalent container definition where an entrypoint script can be hooked in. (None exists in the repo currently — it will need to be created.)
3. The StackOverflow link referenced in the ticket ([SO #49975735](https://stackoverflow.com/q/49975735/329496)) points to the same pattern — injecting `window._env_` via a shell script at container start. The answer there confirms this is the accepted approach for CRA + Docker.
4. `REACT_APP_BACKEND_URL` should continue to work for local development without changes to developer workflow.
5. Only `API_ROOT` needs to be runtime-configurable for now, though the `window._env_` object is naturally extensible to other config values later.

---

### Affected Areas

| File | Change |
|---|---|
| `src/agent.js` | Change `API_ROOT` constant (line ~3) to read from `window._env_.API_ROOT` with fallbacks |
| `public/index.html` | Add `<script src="%PUBLIC_URL%/env-config.js"></script>` |
| `public/env-config.js` | **New file** — default `window._env_` definition for local/build-time fallback |
| `docker-entrypoint.sh` | **New file** — container startup script that rewrites `env-config.js` from env vars |
| `Dockerfile` | **New file** (or existing if one exists outside the repo) — must `COPY` the entrypoint and set it as `ENTRYPOINT` |
| `.gitignore` | No changes needed — `env-config.js` should be committed (it contains the default) |
| `README.md` | Update "Making requests to the backend API" section to document the new mechanism |

No Redux reducers, components, or API method signatures change. This is purely an infrastructure/configuration concern isolated to `agent.js` and the container startup sequence.

---

## Risk and Confidence

- **Confidence in approach:** High — this is the canonical pattern for runtime config in CRA/static-file apps as confirmed by the referenced SO question and widely used in production.
- **Confidence in implementation feasibility:** High — the change to `src/agent.js` is a single line; the main work is the Dockerfile/entrypoint which is straightforward.
- **Known risks:**
  - **`window._env_` access before DOM ready:** Not a concern here — `env-config.js` is loaded synchronously in `<head>` before the React bundle, so `window._env_` is always defined by the time `agent.js` executes.
  - **CRA test environment:** Jest runs in Node, not a browser, so `window._env_` will be `undefined` in tests. The fallback chain (`|| process.env.REACT_APP_BACKEND_URL || hardcoded default`) handles this correctly, but test setup should be verified.
  - **`public/env-config.js` accidentally committed with a staging/prod URL:** The file should always contain the safe default (the public demo API). The per-environment URL only ever lives in container environment variables, never in git.
  - **Cache headers on `env-config.js`:** Nginx must serve `env-config.js` with `Cache-Control: no-store` or a very short TTL; otherwise old API roots can be cached in browsers across environment promotions. This is a deployment config concern.
  - **OpenShift-specific:** On OpenShift, if the container runs as a non-root user (common), the entrypoint script must have write permission to wherever the static files are served from. The Dockerfile must account for this (e.g., `RUN chown -R app:app /usr/share/nginx/html`).

---

## Open Questions

1. **Is there an existing `Dockerfile` for this app?** It was not found in the repository. If one exists in a separate infra/helm repo, the entrypoint changes must be coordinated there.
2. **What is the container base image?** The entrypoint script above assumes nginx. If it's Apache httpd, Caddy, or a Node server (`serve`, `http-server`), the script path for static files differs.
3. **Are there other values in `agent.js` or elsewhere that also need to be runtime-configurable** (e.g., feature flags, analytics keys)? If so, they should be added to `window._env_` at the same time rather than doing this twice.
4. **Should `REACT_APP_BACKEND_URL` be deprecated for local dev in favour of `API_ROOT` (the non-`REACT_APP_` name)?** Keeping `REACT_APP_BACKEND_URL` for local dev is fine, but the naming inconsistency between the env var name and the JS variable name could confuse new contributors. Worth deciding on a canonical name.
5. **Is there a CI pipeline that needs updating?** If CI runs a build and pushes the image, the pipeline itself does not change — but integration/smoke tests that run against a deployed container need to ensure they set `API_ROOT` in the container env, not rely on build-time values.

---

## Key Review Points

When reviewing the Phase 2 PR, validate:

- [ ] `src/agent.js`: `API_ROOT` resolution uses `window._env_.API_ROOT` first, then `process.env.REACT_APP_BACKEND_URL`, then hardcoded default — **in that order**. Confirm no other file reads `API_ROOT` directly (it is only defined and used within `agent.js`).
- [ ] `public/env-config.js`: Confirm the default URL is the safe public demo API, not a staging/prod URL.
- [ ] `public/index.html`: Confirm the `<script>` tag for `env-config.js` appears **before** the main bundle scripts so `window._env_` is defined when `agent.js` runs.
- [ ] `docker-entrypoint.sh`: Confirm it uses a shell default (`${API_ROOT:-<fallback>}`) so the container starts safely even if `API_ROOT` is not set in the environment.
- [ ] Jest tests: Run `npm test` and confirm no tests break due to `window._env_` being undefined in the Node/jsdom environment. The fallback chain should absorb this silently.
- [ ] `README.md`: Confirm the "Making requests to the backend API" section is updated to reflect that `src/agent.js` no longer needs manual editing, and documents both the `REACT_APP_BACKEND_URL` (dev) and `API_ROOT` (container) env var approaches.
- [ ] No secrets or environment-specific URLs appear anywhere in committed files.
- [ ] Cache headers for `env-config.js` are documented or configured in the nginx config within the Dockerfile.

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.
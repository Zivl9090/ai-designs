---
ticketId: ENG-5
ticketTitle: "eact-redux-realworld-example-app"
status: draft
createdAt: 2026-03-22T13:25:01.785Z
---

> Selected repository: `Zivl9090/react-redux-realworld-example-app`
> Reasoning: This is the frontend React application mentioned in the ticket that needs to be modified to read API_ROOT from an environment variable instead of a hardcoded value in src/agent.js.

# Design Document: ENG-5 — Runtime-Configurable `API_ROOT` via Environment Variable

---

## Classification
- **Type:** feature
- **Scope:** single-repo
- **Complexity:** low
- **Estimated effort:** 1–2 hours (small change, but requires understanding the CRA build model and the 12-factor constraint)

---

## Design

### Recommended Approach

The core tension here is that **Create React App (CRA) bakes `REACT_APP_*` environment variables into the static JS bundle at build time** via Webpack's `DefinePlugin`. This means a naïve `process.env.REACT_APP_BACKEND_URL` will not satisfy the 12-factor requirement of "build once, run anywhere" — the value is frozen at `npm run build` time, not at container startup.

The Stack Overflow question the ticket references ([https://stackoverflow.com/q/49975735/329496](https://stackoverflow.com/q/49975735/329496)) is specifically about this problem with CRA builds in containerised environments.

The standard solution for CRA in containers is to **inject a runtime configuration object via a small inline `<script>` block** in `public/index.html`, written by a shell script (or entrypoint) that runs when the container starts. The JS bundle reads from that `window`-level object rather than from `process.env`.

#### Concrete steps

**1. Add a runtime config script to `public/index.html`**

Add a placeholder `<script>` block that the container entrypoint will populate:

```html
<!-- public/index.html -->
<script>
  window.__RUNTIME_CONFIG__ = {
    API_ROOT: "%REACT_APP_BACKEND_URL%"
  };
</script>
```

The `%REACT_APP_BACKEND_URL%` syntax is CRA's own [HTML env-var interpolation](https://create-react-app.dev/docs/adding-custom-environment-variables/#referencing-environment-variables-in-the-html) — this gives a sensible default for local dev (where you still set the var before `npm start`) while keeping the mechanism open for runtime override.

**2. Write a container entrypoint script (`docker-entrypoint.sh`)**

```bash
#!/bin/sh
# Replaces the placeholder in the already-built index.html with the
# value of API_ROOT supplied at container runtime.
sed -i "s|%REACT_APP_BACKEND_URL%|${API_ROOT}|g" /usr/share/nginx/html/index.html
exec "$@"
```

This script runs once at container start (before the static file server starts), mutating only the tiny `<script>` block — the rest of the built JS is untouched.

**3. Update `src/agent.js` to read from `window.__RUNTIME_CONFIG__` with a build-time fallback**

Current code in `src/agent.js`:
```js
const API_ROOT = 'https://conduit.productionready.io/api';
```

Replace with:
```js
const API_ROOT =
  (window.__RUNTIME_CONFIG__ && window.__RUNTIME_CONFIG__.API_ROOT) ||
  process.env.REACT_APP_BACKEND_URL ||
  'https://conduit.productionready.io/api';
```

This preserves full backward compatibility:
- Local dev with `REACT_APP_BACKEND_URL` set → works as before
- Production container with `API_ROOT` injected at startup → uses runtime value
- Bare `npm start` with nothing set → falls back to the hardcoded default

**4. Wire it into the Dockerfile / OpenShift `DeploymentConfig`**

```dockerfile
# Dockerfile (illustrative — repo has none currently)
FROM node:18 AS build
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

In OpenShift, set `API_ROOT` as an environment variable in the `DeploymentConfig` or via a `ConfigMap`. No new image is needed per environment.

---

### Alternatives Considered

| Alternative | Why set aside |
|---|---|
| **Multiple `REACT_APP_BACKEND_URL` values per environment at build time** (one image per env) | Violates the stated 12-factor constraint. Requires a rebuild for every environment promotion, which is exactly what the ticket wants to avoid. |
| **CRA's `REACT_APP_*` env vars alone** | These are inlined at build time by Webpack. A single built image cannot have different values in different environments without a rebuild. |
| **Fetch a `/config.json` at runtime** (e.g., `fetch('/config.json')` on app init) | Technically valid and used in some projects, but adds an async loading step before `agent.js` is usable, requires a loading state/spinner, and complicates error handling. The `window.__RUNTIME_CONFIG__` approach is synchronous and simpler. |
| **Nginx `sub_filter` to rewrite on the fly** | Clever but pushes logic into Nginx config, harder to read and test. The `sed`-on-startup approach is universally understood. |
| **Using `__webpack_public_path__` or other Webpack escape hatches** | CRA does not expose Webpack config without ejecting; ejecting is a significant maintenance burden. |
| **Ejecting CRA to customise DefinePlugin** | Too invasive for what is a small operational concern. |

---

### Key Assumptions

1. The production deployment uses a container image built from `npm run build` output served by a static file server (Nginx or similar). The entrypoint `sed` approach only works if the container has write access to `index.html` at startup — standard for Docker/OpenShift writable container layers.
2. There is currently **no `Dockerfile`** in the repository; one will need to be added alongside this change. If one already exists in a separate infra repo, the `ENTRYPOINT` change and `docker-entrypoint.sh` should go there instead.
3. The only URL that needs runtime injection is `API_ROOT` in `src/agent.js`. The codebase has no other hardcoded backend URLs — confirmed by reviewing `agent.js`, which is the single source of truth for all API calls per the CLAUDE.md.
4. Local development workflow (using `REACT_APP_BACKEND_URL` as a build-time CRA env var) should remain unchanged.
5. OpenShift `ConfigMap` or `DeploymentConfig` environment variables are the mechanism for supplying `API_ROOT` per environment — not baked-in secrets or build arguments.

---

### Affected Areas

| File | Change |
|---|---|
| `src/agent.js` | Change `API_ROOT` constant to read from `window.__RUNTIME_CONFIG__.API_ROOT` with fallback chain |
| `public/index.html` | Add inline `<script>` block with `window.__RUNTIME_CONFIG__` placeholder |
| `docker-entrypoint.sh` *(new file)* | Shell script to `sed`-replace the placeholder at container startup |
| `Dockerfile` *(new file)* | Multi-stage build + entrypoint wiring (if not already in an infra repo) |
| `.env.example` *(optional, new)* | Document `REACT_APP_BACKEND_URL` for local dev and `API_ROOT` for container runtime |
| `README.md` | Update "Making requests to the backend API" section to document the new mechanism |

No Redux reducers, no React components, no routing changes required.

---

## Risk and Confidence

- **Confidence in approach:** High — this is the canonical solution to this exact CRA + 12-factor problem, matching what the referenced Stack Overflow thread converges on.
- **Confidence in implementation feasibility:** High — the change to `agent.js` is 3 lines; the `public/index.html` addition is 5 lines; the entrypoint script is 3 lines. No dependency changes needed.
- **Known risks:**
  - **Content Security Policy (CSP):** If the deployed app has a strict CSP that disallows inline scripts, the `<script>` block in `index.html` will be blocked. Check nginx/OpenShift CSP headers and add a `nonce` or `sha256` hash if needed.
  - **`index.html` caching:** If the static server or a CDN aggressively caches `index.html`, the runtime-injected value won't update on redeploy. Ensure `index.html` is served with `Cache-Control: no-cache`. (Built JS/CSS chunks are content-hashed by CRA and safe to cache.)
  - **`sed` portability:** The `sed -i` flag behaves differently on macOS (`sed -i ''`) vs GNU/Linux (`sed -i`). The Dockerfile uses a Linux base image so this is fine in production, but developers running the script locally on macOS need to be aware.
  - **`window.__RUNTIME_CONFIG__` is synchronous but global** — any other future code that reads from it must do so after `index.html` is parsed. This is naturally satisfied since the `<script>` block appears before the app bundle.

---

## Open Questions

1. **Does a Dockerfile already exist** in a separate infra/helm/GitOps repo for this service? If yes, the `docker-entrypoint.sh` and `ENTRYPOINT` changes should go there rather than in this repo, and only the `src/agent.js` + `public/index.html` changes live here.
2. **Is the static build served directly by Nginx, or via a Node.js server** (e.g., `serve` or `express`)? If a Node server is used instead of Nginx, the entrypoint `sed` approach still works but the Dockerfile differs.
3. **Are there other environment-specific values** beyond `API_ROOT` that might need the same treatment (e.g., analytics keys, feature flags, OAuth client IDs)? If yes, extend `window.__RUNTIME_CONFIG__` now rather than revisiting the mechanism later.
4. **What is the expected local dev experience?** The current `REACT_APP_BACKEND_URL` CRA variable works fine for `npm start`. Confirm that local devs should continue using that and not the `window.__RUNTIME_CONFIG__` path directly.
5. **Is there a test environment where the container image is run without any `API_ROOT` set** (e.g., in a CI smoke-test step)? If so, verify the fallback to the hardcoded default or `REACT_APP_BACKEND_URL` is acceptable for that case.

---

## Key Review Points

When reviewing the Phase 2 PR, validate:

- [ ] `src/agent.js`: `API_ROOT` uses the three-level fallback chain (`window.__RUNTIME_CONFIG__.API_ROOT` → `process.env.REACT_APP_BACKEND_URL` → hardcoded default). No other files reference the hardcoded `conduit.productionready.io` URL.
- [ ] `public/index.html`: The `<script>` block appears **before** the `<div id="root">` and before any `<script src=...>` bundle tags, so it is evaluated first.
- [ ] `docker-entrypoint.sh`: The `sed` replacement correctly handles URLs containing `/` (use `|` as the sed delimiter to avoid escaping issues with URL slashes — e.g., `sed -i "s|placeholder|${API_ROOT}|g"`).
- [ ] `docker-entrypoint.sh`: The script ends with `exec "$@"` so the container's CMD (nginx) runs as PID 1 and receives signals correctly.
- [ ] `Dockerfile`: Uses a multi-stage build so the final image does not contain Node, `node_modules`, or source files — only the built static assets.
- [ ] `README.md` or a new `DEPLOYMENT.md`: Documents the `API_ROOT` environment variable, its expected format (full URL including `/api` suffix or not?), and that `index.html` must not be cached.
- [ ] **Existing tests still pass** — `npm test` should be green. The fallback chain in `agent.js` means tests that don't set `window.__RUNTIME_CONFIG__` will use `process.env.REACT_APP_BACKEND_URL` or the default, which is the current behaviour.
- [ ] Verify there are no other places in the codebase that hardcode the backend URL outside `agent.js` (search for `productionready.io` and `conduit` across all `src/` files).

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.
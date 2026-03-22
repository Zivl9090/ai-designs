---
ticketId: ENG-5
ticketTitle: "eact-redux-realworld-example-app"
status: draft
createdAt: 2026-03-22T13:50:28.982Z
---

> Selected repository: `Zivl9090/react-redux-realworld-example-app`
> Reasoning: This is the exact repository mentioned in the ticket that needs to be modified to support environment variable configuration for API_ROOT instead of hardcoded values.

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

The core problem is that Create React App (CRA) bakes `process.env.REACT_APP_*` values into the JavaScript bundle **at build time**. A container image built once cannot have its `REACT_APP_BACKEND_URL` changed at runtime — the value is already inlined into the compiled JS.

The standard 12-factor-compatible solution for CRA apps in containers is to **serve a small, dynamically-written runtime config file** from the web server, and read it in the app before any API calls are made. The two most common patterns are:

1. **`window._env_` injected by an entrypoint shell script** (recommended — idiomatic for Docker/OpenShift)
2. A `/config.json` endpoint fetched asynchronously at app start (adds async complexity, see Alternatives)

#### Recommended: `window._env_` via entrypoint script

**How it works:**

1. An entrypoint shell script (run when the container starts) writes a `env-config.js` file that sets `window._env_ = { API_ROOT: "$API_ROOT" }`, substituting actual environment variables from the container's runtime environment.
2. `index.html` loads that script via a `<script>` tag before the React bundle.
3. `src/agent.js` reads `window._env_.API_ROOT` (with a fallback to the current build-time value for local development).

This is exactly the pattern described in the StackOverflow question referenced in the ticket ([https://stackoverflow.com/q/49975735/329496](https://stackoverflow.com/q/49975735/329496)).

---

#### Concrete file changes

**1. `public/index.html`** — add a script tag before the main bundle:

```html
<script src="%PUBLIC_URL%/env-config.js"></script>
```

This file is served statically by the container's web server. It must load before React boots.

**2. `docker-entrypoint.sh`** (new file, at repo root) — generates `env-config.js` at container startup from actual runtime env vars:

```sh
#!/bin/sh
# Write runtime env vars into a JS file the browser can load
cat <<EOF > /usr/share/nginx/html/env-config.js
window._env_ = {
  API_ROOT: "${API_ROOT:-https://conduit.productionready.io/api}"
};
EOF

exec "$@"
```

The `:-` default means local/dev environments without the variable set still work.

**3. `public/env-config.js`** (new file, checked into repo as a stub) — needed so `npm start` (CRA dev server) doesn't 404 on the script tag:

```js
// This file is replaced at container startup by docker-entrypoint.sh.
// It provides a fallback for local development.
window._env_ = {
  API_ROOT: "https://conduit.productionready.io/api"
};
```

**4. `src/agent.js`** — change the `API_ROOT` constant (currently line 1–3 area):

Current:
```js
const API_ROOT = 'https://conduit.productionready.io/api';
```

New:
```js
const API_ROOT =
  (window._env_ && window._env_.API_ROOT) ||
  process.env.REACT_APP_BACKEND_URL ||
  'https://conduit.productionready.io/api';
```

This preserves full backward compatibility:
- **Container (production):** `window._env_.API_ROOT` wins (set by entrypoint script).
- **`npm start` with `REACT_APP_BACKEND_URL` set:** falls through to `process.env` (existing local dev pattern per `CLAUDE.md`).
- **Plain `npm start` with no config:** falls back to the hardcoded default.

**5. `Dockerfile`** (new or existing — check if one exists; none is visible in the repo) — example nginx-based production Dockerfile:

```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

**6. `.gitignore`** — optionally add `public/env-config.js` to gitignore if you don't want the stub committed, but it's cleaner to commit the stub so `npm start` works without extra setup.

---

### Alternatives Considered

| Alternative | Why set aside |
|---|---|
| **Build per environment** (`REACT_APP_BACKEND_URL` at build time, one image per env) | Violates 12-factor / "build once, promote" requirement. Already the status quo — ticket explicitly rejects this. |
| **Switch statement / named environments in code** | Explicitly rejected in the ticket. Requires code changes to add a new environment. |
| **Fetch `/config.json` async at app start** | Works, but requires wrapping the Redux store initialization and all of `agent.js` initialization in a `fetch` promise. Adds boot complexity and a network round-trip. The `window._env_` pattern is simpler and synchronous. |
| **Nginx `sub_filter` to rewrite the bundle** | Fragile — depends on the exact string being present in the minified bundle. Breaks on any variable renaming during minification. |
| **`envsubst` on the built JS files** | Similar fragility. Also requires knowing which chunk file to edit. |
| **CRA `REACT_APP_*` with `.env.production`** | Still build-time. Doesn't help for runtime promotion. |

---

### Key Assumptions

1. The app is (or will be) containerized with a web server (nginx or similar) as the static file server — the entrypoint script must be able to write to the served directory at startup. If the web server is something other than nginx (e.g., Apache, a Node static server), the Dockerfile and entrypoint paths will differ, but the pattern is identical.
2. The container orchestration (OpenShift) injects the `API_ROOT` value as an environment variable into the container at runtime — this is standard OpenShift `DeploymentConfig`/`Deployment` env var injection.
3. There is no existing `Dockerfile` in this repo (none is visible in the provided file tree). One will need to be created.
4. The app is currently running as a CRA build served as static files, not via a Node SSR server.
5. `public/env-config.js` checked into the repo as a stub is acceptable. If it is not (e.g., the team doesn't want a hardcoded URL in source), the stub can use an empty string and the app's fallback chain handles it.

---

### Affected Areas

| File | Change |
|---|---|
| `src/agent.js` | Change `API_ROOT` constant to read from `window._env_.API_ROOT` with fallbacks |
| `public/index.html` | Add `<script src="%PUBLIC_URL%/env-config.js"></script>` |
| `public/env-config.js` | New stub file for local dev |
| `docker-entrypoint.sh` | New entrypoint script that writes `env-config.js` at container start |
| `Dockerfile` | New (or update existing) to copy entrypoint script and set `ENTRYPOINT` |
| `CLAUDE.md` | Update "How to run" section to document `API_ROOT` env var pattern |
| `.gitignore` | Optional: exclude `public/env-config.js` if stub should not be committed |

No Redux reducers, components, or API method signatures change. The only runtime behavior change is where `API_ROOT` is read from.

---

## Risk and Confidence

- **Confidence in approach:** High — this is the well-established pattern for CRA + Docker + 12-factor, directly matching the StackOverflow question the ticket author linked. The `window._env_` approach is synchronous, requires no changes to the React component tree, and has no effect on the Redux store or component architecture.
- **Confidence in implementation feasibility:** High — all changes are confined to one constant in `agent.js`, two lines in `index.html`, one new shell script, and one new stub JS file. No build tooling changes required.
- **Known risks:**
  - **`window._env_` is undefined if `env-config.js` fails to load** (e.g., 404). The guard `(window._env_ && window._env_.API_ROOT)` prevents a JS crash, but API calls will silently fall back to the default URL. Consider adding an explicit console warning if `window._env_` is absent.
  - **`public/env-config.js` stub contains a hardcoded URL** — if this URL is wrong for a developer's local setup, the existing `REACT_APP_BACKEND_URL` fallback covers it, but the developer needs to know the priority order.
  - **Shell variable quoting** in `docker-entrypoint.sh`: if `API_ROOT` contains special characters, the `cat <<EOF` heredoc may not escape them correctly. Use `envsubst` as an alternative to the `cat` approach for robustness.
  - **OpenShift security policies** may restrict entrypoint scripts from writing to the served directory at runtime. Verify the nginx container's filesystem permissions in the target OpenShift environment. Running as non-root (OpenShift default) may require the write target to be a volume or explicitly writable.
  - **CRA caching**: browsers may cache `env-config.js` aggressively. Add `Cache-Control: no-store` for this specific file in the nginx config so environment switches take effect immediately.

---

## Open Questions

1. **Does a `Dockerfile` already exist** in the deployment pipeline (perhaps in a separate infra repo or CI config) that this change needs to be coordinated with? The repo itself doesn't appear to contain one.
2. **What web server serves the static build in production?** The entrypoint script's write path (`/usr/share/nginx/html/`) assumes nginx. If it's Apache or a Node server, the path differs.
3. **Should `REACT_APP_BACKEND_URL` (the existing build-time env var documented in `CLAUDE.md`) be deprecated** in favor of the new `API_ROOT` runtime var, or kept as a parallel mechanism for local dev? The fallback chain in the proposed `agent.js` change preserves both, but it may be cleaner to align on one name across environments.
4. **OpenShift write permissions**: Can the container's entrypoint script write to the nginx document root? Or should a `ConfigMap`-mounted `env-config.js` be used instead (an OpenShift-native alternative that avoids the entrypoint write entirely)?
5. **Should `env-config.js` be in `.gitignore`?** If yes, developers need a setup step (`cp public/env-config.example.js public/env-config.js` or similar) documented in the README.

---

## Key Review Points

- [ ] `src/agent.js`: `API_ROOT` reads `window._env_.API_ROOT` with correct guard (`window._env_ &&`) before property access — no uncaught TypeError if the script tag fails to load.
- [ ] `src/agent.js`: Fallback chain is `window._env_.API_ROOT` → `process.env.REACT_APP_BACKEND_URL` → hardcoded default. Confirm order matches intended priority.
- [ ] `public/index.html`: `<script src="%PUBLIC_URL%/env-config.js"></script>` appears **before** the React bundle scripts (CRA injects bundle scripts at end of `<body>`; this tag should be in `<head>` or early `<body>`).
- [ ] `public/env-config.js`: Stub file exists and is valid JS so `npm start` doesn't produce a 404 console error.
- [ ] `docker-entrypoint.sh`: Is executable (`chmod +x`) and uses `#!/bin/sh` (not `bash`) for Alpine Linux compatibility.
- [ ] `docker-entrypoint.sh`: Falls back to a sane default URL if `API_ROOT` env var is unset (using `${API_ROOT:-<default>}`).
- [ ] `Dockerfile`: Entrypoint script is copied in and `ENTRYPOINT` is set — not overriding the CMD (nginx startup command).
- [ ] Nginx config (if customized): `env-config.js` has `Cache-Control: no-store` or equivalent to prevent stale environment config being served from browser cache.
- [ ] `CLAUDE.md` or `README.md`: Updated to document the `API_ROOT` runtime env var and the two-tier config mechanism so future developers understand the priority order.
- [ ] Manual smoke test: Build the image, run it locally with `-e API_ROOT=http://localhost:3000/api`, open browser devtools and confirm `window._env_.API_ROOT` is set correctly and network requests go to the right host.

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.
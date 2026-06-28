# Nuxt Frontend

How we build **Nuxt 4** apps. Not the Vue 3 + Vite SPA pattern in [vue-frontend.md](vue-frontend.md).

We ship two production shapes — pick one up front:

| Pattern | Use when | Deploy |
|---------|----------|--------|
| **A — Static site** | Marketing, blog, portfolio; content in repo; no API | `nuxt generate` → nginx serves `.output/public` |
| **B — Spring SPA** | App UI with Java session auth + optional Nitro routes | Nitro `node-server`; pair with [java-spring-backend.md](java-spring-backend.md) |

Per-repo adoption: [assessments/nuxt-frontend.md](assessments/nuxt-frontend.md).

## Shared conventions

- **Nuxt 4** (`nuxt` ^4.4), Vue 3.5, TypeScript.
- **Tailwind via module.** `@nuxtjs/tailwindcss` — exception to vue-frontend's "no CSS framework" rule.
- **Lint:** `@nuxt/eslint` + `eslint.config.mjs`.
- **File-based routing** under `pages/` (repo root or `app/pages/` — both appear in org repos; pick one layout per project and keep to it).
- **No Pinia by default.** Use `useState` / composables unless the app clearly needs a store.

---

## Pattern A — Static site

Flat-file or repo-local content, no backend. Single image, nginx in production.

### Layout (repo root)

```
./
  nuxt.config.ts
  package.json
  app.vue
  Dockerfile              # generate → nginx, not node-server
  nginx.conf
  pages/
    index.vue
    blog/
      index.vue
      [...slug].vue
  components/
  data/                   # TypeScript content modules (projects, posts, …)
  server/
    routes/
      rss.get.ts          # build-time / prerender only when using generate
  public/
  helm/
  oglimmer.sh             # single-image shape
```

Repo may live at the **monorepo root** (not under `frontend/`).

### `package.json` scripts

```json
{
  "scripts": {
    "dev": "nuxt dev",
    "build": "nuxt build",
    "generate": "nuxt generate",
    "preview": "nuxt preview",
    "postinstall": "nuxt prepare",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "typecheck": "nuxi typecheck"
  }
}
```

CI at repo root: `npm ci` → `lint` → `typecheck` → `build` (or `generate` if Docker uses that — **keep CI and Dockerfile aligned**).

### Content

Store structured content in `data/*.ts` (typed arrays). Blog slugs drive `pages/blog/[...slug].vue`. No CMS, no API fetch for primary content.

### Server routes with static deploy

`server/routes/*.ts` runs at **build/prerender** time when using `nuxt generate`. Routes that must run on **every request** (live feeds, auth, webhooks) need pattern B (`node-server`) instead.

### Docker

Multi-stage: Node build → nginx alpine serving `.output/public`:

```dockerfile
# build: npm ci && npm run generate
# run:   nginx on port 8080 (non-root), custom nginx.conf
COPY --from=builder /app/.output/public /usr/share/nginx/html
```

Aligns with [docker.md](docker.md) Vue SPA nginx shape — Nuxt generate output replaces Vite `dist/`.

### Ingress

Catch-all `/` to the frontend Service. No `/api` split.

---

## Pattern B — Spring SPA

App UI in a `frontend/` subfolder; session cookies + CSRF to a Spring Boot API.

### Layout

```
frontend/
  nuxt.config.ts
  package.json
  Dockerfile              # Nitro .output node-server
  app/
    app.vue
    pages/
    components/
    composables/
      useApi.ts
      useAuth.ts
  server/
    api/
      generate.get.ts     # Nitro-owned — NOT proxied to Spring
  public/
```

### Stack

| Layer | Choice |
|-------|--------|
| Server | Nitro (`preset: 'node-server'`) |
| Test | Vitest (optional; fixture regen via `renovate:regen`) |
| Auth | Spring session + OAuth2 (GitHub/GitLab) |

### `nuxt.config.ts`

```ts
nitro: {
  preset: 'node-server',
  devProxy: { /* enumerated Spring routes */ },
  prerender: {
    crawlLinks: true,
    routes: ['/', '/developers', '/generate', '/dashboard', ...]
  }
}
```

**Why node-server:** routes like `/api/generate` must run in Node at request time. Prerendered pages still land in `.output/public`.

### Dev proxy — enumerate, don't blanket

```ts
devProxy: {
  '/api/feedback': { target: 'http://localhost:8080/api/feedback', changeOrigin: true },
  '/api/me': { target: 'http://localhost:8080/api/me', changeOrigin: true },
  // oauth2, dashboard, repos, actuator — each path explicit
}
```

Do **not** proxy routes Nitro owns (e.g. `/api/generate`). Spring `context-path: /api` → targets `http://localhost:8080/api/...`.

### API client (`useApi`)

```ts
export function useApi() {
  async function request(path: string, options: RequestInit = {}): Promise<Response> {
    const headers = new Headers(options.headers || {})
    headers.set('X-Requested-With', 'XMLHttpRequest')
    if (method is mutating) {
      const csrf = readCookie('XSRF-TOKEN')
      if (csrf) headers.set('X-XSRF-TOKEN', csrf)
    }
    return fetch(`/api${path}`, { ...options, headers, credentials: 'include' })
  }
  return { request }
}
```

### Auth (`useAuth`)

- `useState` for `user`, `loading`, `providers`.
- `fetchMe()` → `GET /api/me` with `credentials: 'include'`.
- `login(provider)` → top-level nav to `/api/oauth2/authorization/github`.
- No JWT in `localStorage` — see [java-spring-backend.md](java-spring-backend.md).

### Docker (pattern B)

```dockerfile
# build: npm run build  →  .output/
# run:   node .output/server/index.mjs
ENV NITRO_PORT=8080 NITRO_HOST=0.0.0.0
USER node
EXPOSE 8080
```

### CI (pattern B)

[github-actions.md](github-actions.md) `frontend` job with `working-directory: frontend`:

- `npm run lint`
- `npm run type-check`
- `npm run test` (when Vitest is configured)

---

## Relationship to other docs

| Doc | Pattern A | Pattern B |
|-----|-----------|-----------|
| [vue-frontend.md](vue-frontend.md) | Different toolchain | Different auth model |
| [java-spring-backend.md](java-spring-backend.md) | N/A | Required |
| [docker.md](docker.md) | nginx + `.output/public` | Nitro node-server |
| [helm.md](helm.md) | SPA catch-all `/` | Split `/api/*` vs `/` (+ Nitro routes) |
| [oglimmer-sh.md](oglimmer-sh.md) | Single image | `-fe` / `-be` with backend |

## New-repo checklists

**Pattern A (static):**

1. `nuxi init` at repo root (or dedicated package dir).
2. Add `data/` modules + `pages/` routes.
3. `npm run generate` in Dockerfile; nginx on 8080.
4. CI: lint + typecheck + build/generate.
5. Single-image `oglimmer.sh` + Helm chart.

**Pattern B (Spring SPA):**

1. Scaffold `frontend/` with Nuxt 4; `nitro.preset: 'node-server'`.
2. Add `useApi` + `useAuth` (cookie + CSRF).
3. Enumerate `devProxy` for every Spring route the browser calls.
4. Add `server/api/` only for Nitro-owned routes.
5. Nitro Dockerfile; wire CI; align Ingress with Spring `context-path`.

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Use `node-server` when everything can be static | Pattern A: `generate` + nginx |
| Use `generate` + nginx when routes must run per request | Pattern B: `node-server` |
| Copy vue-frontend JWT / Pinia auth (pattern B) | `useAuth` + session cookies |
| Blanket `devProxy: '/api' → backend` | Enumerate; exclude Nitro-owned routes |
| CI runs `build` but Docker runs `generate` (or vice versa) | One command for both gates |
| `views/` directory (Vue SPA habit) | `pages/` (Nuxt) |
| Assume `vite.config.ts` dev proxy | Nuxt `nitro.devProxy` only (pattern B) |
# Vue Frontend

How we build SPAs. Vue 3 + TypeScript + Vite, no UI framework, hand-rolled fetch client, CSS custom properties for theming.

**Scope:** Vue 3 + Vite SPAs only. Does **not** apply to Nuxt ([nuxt-frontend.md](nuxt-frontend.md)), Svelte, or server-rendered apps. Per-repo adoption and path mapping: [assessments/vue-frontend.md](assessments/vue-frontend.md), [assessments/repo-map.md](assessments/repo-map.md).

## Philosophy

- **Composition API only.** `<script setup lang="ts">` everywhere. No Options API.
- **No UI framework.** Design system via CSS custom properties in `styles.css` — zero runtime cost, full control.
- **Thin views, fat composables.** Route targets render and wire; reusable logic lives in `composables/` and `lib/`.
- **Same-origin API paths.** Relative `/api/…` in dev (Vite proxy) and production (Ingress or nginx proxy) — no CORS configuration in the app.
- **Typed boundaries.** Shared interfaces in `types.ts`; API functions return typed promises; router meta is augmented.
- **Test the logic, not the pixels.** Vitest for `lib/`, `api.ts`, and composables; component tests where behaviour matters.

## Stack

| Layer | Choice |
|-------|--------|
| Framework | Vue 3.5+ |
| Language | TypeScript (strict) |
| Build | Vite |
| State | Pinia (setup-store factories) |
| Routing | vue-router 5 |
| Testing | Vitest + jsdom + `@vue/test-utils` / Testing Library |
| Lint | ESLint flat config (`eslint-plugin-vue` + `@vue/eslint-config-typescript`) |
| Fonts | `@fontsource-variable/*` |

No axios, no Prettier, no Tailwind, no Nuxt (unless the project explicitly is SSR).

## Layout

```
frontend/
  package.json          # version = single source of truth for releases
  vite.config.ts
  tsconfig.json
  eslint.config.js
  index.html            # inline theme boot script (no flash)
  nginx.conf            # compose dev: proxy /api + SPA fallback
  nginx-security-headers.conf
  Dockerfile
  src/
    main.ts             # createApp → Pinia → router → mount
    App.vue             # shell chrome, global dialogs, <RouterView>
    api.ts              # fetch wrapper, ApiError, domain API functions
    router.ts           # routes + beforeEach guard
    types.ts            # shared interfaces
    styles.css          # design system (:root + html[data-theme])
    theme.ts            # theme registry + localStorage
    build-info.ts       # types for Vite-injected globals
    test-setup.ts       # Vitest localStorage shim
    components/         # PascalCase reusables (ConfirmDialog, UserMenu, …)
    composables/        # use* stateful logic (useConfirm, useAsyncData, …)
    stores/             # use*Store Pinia factories
    views/              # *View.vue route targets
    lib/                # pure functions + colocated *.test.ts
```

### Naming

| Kind | Convention | Example |
|------|------------|---------|
| Component file | PascalCase | `ConfirmDialog.vue` |
| View file | `*View.vue` | `EventListView.vue` |
| Composable | `use*` | `useAsyncData.ts` |
| Store | `use*Store` | `stores/auth.ts` → `useAuthStore` |
| Pure util | verb/noun in `lib/` | `lib/datetime.ts` |

`vue/multi-word-component-names` is off — short names like `App.vue` are fine.

## `package.json` scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview",
    "typecheck": "vue-tsc --noEmit",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "test": "vitest run --passWithNoTests",
    "test:watch": "vitest",
    "check": "npm run typecheck && npm run lint && npm run test"
  }
}
```

- `build` runs typecheck first — a type error never ships.
- `check` is the local equivalent of CI's frontend job.
- `--passWithNoTests` lets new repos boot without tests yet.

## Vite config

### Dev proxy

Forward backend paths to `localhost:8080` so the SPA uses same-origin relative URLs:

```ts
server: {
  port: 5173,
  proxy: {
    '/api': 'http://localhost:8080',
    '/healthz': 'http://localhost:8080',
  },
},
```

Add every path the backend serves that the browser calls directly (`/mcp`, `/oauth`, `/.well-known/…`). Keep proxy list in sync with [docker.md](docker.md) `nginx.conf` and [helm.md](helm.md) ingress paths.

### Build metadata

Inject version info via `define` (not `import.meta.env` alone — works in tests and non-Vite contexts):

```ts
define: {
  __APP_VERSION__: JSON.stringify(appVersion),
  __GIT_COMMIT__: JSON.stringify(gitCommit),
  __BUILD_TIME__: JSON.stringify(buildTime),
},
```

Resolution order: `VITE_APP_VERSION` / `VITE_GIT_COMMIT` / `VITE_BUILD_TIME` env vars (Docker build-args from `oglimmer.sh`) → `package.json` version + `git rev-parse` → fallbacks.

Declare globals in `build-info.ts`; surface via `useBuildInfo()` composable and the site footer. Release bumps, tags, and registry rules: [versioning-release.md](versioning-release.md).

### Vitest

```ts
test: {
  environment: 'jsdom',
  globals: true,
  setupFiles: ['./src/test-setup.ts'],
},
```

## TypeScript

`tsconfig.json` baseline:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "types": ["vitest/globals"]
  },
  "include": ["src/**/*.ts", "src/**/*.vue"]
}
```

- Import types with `import type { … }`.
- Unused vars prefixed with `_` are allowed (ESLint rule).
- No `any` unless bridging untyped third-party code — comment why.

## API client (`api.ts`)

Hand-rolled `fetch` wrapper — no axios.

### Core types and helpers

```ts
export class ApiError extends Error {
  status: number
  constructor(status: number, message: string) { … }
}

export function errMsg(e: unknown, fallback?: string): string
export function errStatus(e: unknown): number | undefined
export function isJwtExpired(tok: string): boolean
```

- Backend errors are JSON `{ "error": "…" }` — parse and throw `ApiError`.
- `204 No Content` → return `undefined`.
- `errMsg` / `errStatus` at call sites — never assume `e instanceof Error` alone.

### Request path

1. `authHeaders()` — attach `Authorization: Bearer <token>` from `localStorage`; throw `ApiError(401)` if JWT `exp` is past (client-side gate before network).
2. JSON requests — set `Content-Type: application/json`.
3. `FormData` / blob uploads — **do not** set `Content-Type`; let the browser set the multipart boundary.
4. Export a single `api` object with domain methods (`api.listEvents()`, `api.me()`, …).

### Auth token storage

- `localStorage` keys: `token`, `user` (cached profile JSON).
- Auth store owns session lifecycle; `api.ts` only reads the token.
- On 401 from `/api/me`, auth store clears session and redirects to login.

## Pinia stores

Setup-store factories only:

```ts
export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(loadUser())
  const token = ref<string | null>(loadToken())

  function setSession(t: string, u: User) { … }
  function logout() { … }

  return { user, token, setSession, logout, … }
})
```

Rules:

- Persist auth to `localStorage`; rehydrate on load.
- Discard expired JWTs on load — don't wait for the server.
- Cache in-flight promises for one-shot fetches (`ensureMode`, `ensureFreshUser`) to avoid duplicate calls.
- Mutate lists immutably: `list.value = list.value.map(…)`.
- Stores expose actions; views call store methods, not raw `api` for auth-sensitive flows.

## Router (`router.ts`)

### Route definitions

```ts
{
  path: '/admin/events/:id',
  component: () => import('./views/EventDashboardView.vue'),
  props: true,                    // route params → component props
  meta: { requiresAuth: true, requiresAdmin: true },
}
```

- **Lazy-load every view** — `() => import('./views/…')`.
- **`props: true`** when the view needs `:id`, `:slug`, etc. as props.
- **`props: (route) => ({ … })`** for query-driven views (e.g. error code from `?code=`).
- Declare `RouteMeta` via module augmentation:

  ```ts
  declare module 'vue-router' {
    interface RouteMeta {
      requiresAuth?: boolean
      requiresAdmin?: boolean
      hideChrome?: boolean
    }
  }
  ```

### Single `beforeEach` guard

Centralize all redirect logic:

1. Unauthenticated → `/login?redirect=…`
2. `ensureFreshUser()` — 401 clears session; other errors → `/error?code=…`
3. Profile onboarding gate (e.g. unconfirmed profile → `/welcome`)
4. Admin gate → `/error?code=403`
5. Catch-all → `ErrorView` with `code: 404`

Don't scatter `if (!user)` checks in individual views — the guard owns access control.

## Composables

Extract repeated view logic into `composables/use*.ts`.

### `useAsyncData` — list/detail fetching

```ts
const { data, loading, error, reload } = useAsyncData(
  () => api.listEvents(scope.value),
  [],
)
watch(scope, reload)
```

- Starts `loading: true` on mount when `immediate` (default) — avoids empty-state flicker.
- Errors funnel through `errMsg`.
- Call `reload()` after mutations or when inputs change.

### `useConfirm` — promise-based modals

```ts
const { confirm } = useConfirm()
if (await confirm({ message: 'Delete?', danger: true })) { … }
```

- One `<ConfirmDialog>` in `App.vue`, module-level state, `<Teleport to="body">`.
- `danger: true` focuses Cancel so Enter doesn't confirm destructive actions.
- Split API: `useConfirm()` for callers, `useConfirmDialog()` for the dialog component.

### When to add a composable

| Signal | Action |
|--------|--------|
| Same loading/error/fetch block in 2+ views | `useAsyncData` or domain-specific variant |
| Modal/prompt needed from async flow | `useConfirm` / `usePrompt` |
| Column sort/filter/persist | `useColumnSort`, `useColumnConfig` |
| Polling or auto-refresh | `useAutoReload` |

Keep composables testable — pure logic in `lib/`, composable orchestrates refs and lifecycle.

## Views and components

### Views (`*View.vue`)

- One view per route (or per major tab within a layout).
- `<script setup>`: import api/store/composables, minimal local state.
- Template: loading / error / empty / data states — always handle all four.
- Styles: `<style scoped>` using design tokens (`var(--text)`, `var(--accent)`, …).

### Components (`components/`)

- Reusable across views — dialogs, menus, logos, filters.
- Props in, events out. No direct router pushes in leaf components unless they are navigation chrome.
- Heavy components delegate to composables (file upload orchestration, etc.).

### `App.vue`

- App shell: header, nav, `<RouterView>`, footer.
- Global overlays: `ConfirmDialog`, theme switcher.
- Respect `route.meta.hideChrome` for login/onboarding/full-screen views.

## Styling and theming

### Design tokens in `styles.css`

```css
:root {
  --bg: #fcf9ed;
  --text: #000161;
  --accent: #f5ac11;
  --accent-rgb: 245 172 17;   /* for rgb(var(--accent-rgb) / 0.12) tints */
  --danger: #c2491f;
  --radius: 0;
  font-family: var(--mono);
  color-scheme: light;
}

html[data-theme="dark"] {
  --bg: #0a0c10;
  /* full palette override */
}
```

Rules:

- **Solid tokens** (`--bg`, `--text`) for direct use.
- **RGB channel triples** (`--accent-rgb`) for alpha tints: `rgb(var(--accent-rgb) / 0.12)`.
- **Derived tokens** (`--placeholder`, `--glow-1`) when they aren't a simple alpha of a base colour.
- Every theme block redefines the **full palette** — partial overrides leave holes.
- Component styles use `scoped` + tokens; no hard-coded hex in views.

### `theme.ts` + `index.html` boot script

- `THEMES` registry maps `id` → `html[data-theme="…"]` block in CSS.
- `applyTheme(getStoredTheme())` in `main.ts` before mount.
- **Inline script in `index.html`** applies theme from `localStorage` before the bundle loads — prevents flash-of-wrong-theme. Keep theme id lists in sync between `theme.ts` and `index.html`.

## Testing

### What to test

| Target | Tool | Example |
|--------|------|---------|
| `lib/*.ts` | Vitest | `datetime.test.ts`, `submissionRules.test.ts` |
| `api.ts` helpers | Vitest | `isJwtExpired`, `errMsg`, `errStatus` |
| Composables | Vitest | `useConfirm.test.ts`, `useAsyncData.test.ts` |
| Components | `@vue/test-utils` or Testing Library | Interaction flows |
| Theme/CSS tokens | Vitest | `theme.test.ts` — token presence per theme |

### `test-setup.ts`

Install an in-memory `localStorage` shim on `globalThis` and `window` — Node's experimental built-in shadows jsdom's, and the app uses bare `localStorage` (not `window.localStorage` explicitly).

### MSW

Use `msw` for fetch mocking in integration-style tests when needed. Colocate handlers with the test file or a `src/mocks/` module.

### CI mirror

Frontend CI job ([github-actions.md](github-actions.md)): `npm ci` → lint → typecheck → test → build.
Pre-commit ([pre-commit.md](pre-commit.md)): eslint on commit, typecheck + vitest on pre-push.

## nginx (compose / Docker)

The frontend image serves static files. `nginx.conf` in the frontend directory handles:

- **Proxy** `/api/`, `/healthz`, `/mcp`, `/oauth/`, `/.well-known/…` → backend (compose only — Helm uses Ingress).
- **SPA fallback** `try_files $uri /index.html`.
- **Cache** `/assets/*` → `immutable`; `/index.html` → `no-cache`.
- **Security headers** via included snippet.
- **Body size** aligned with backend upload cap.

When adding a backend path, update **three places**: Vite proxy, `nginx.conf`, Helm ingress ([helm.md](helm.md)).

## ESLint

Flat config (`eslint.config.js`):

```js
export default [
  { ignores: ['dist/**', 'node_modules/**'] },
  ...vue.configs['flat/recommended'],
  ...vueTsConfig(),
  {
    rules: {
      'vue/multi-word-component-names': 'off',
      'vue/singleline-html-element-content-newline': 'off',
      'vue/max-attributes-per-line': 'off',
    },
  },
]
```

Formatting is editor responsibility — ESLint catches logic and Vue correctness, not line wraps.

## New-project checklist

1. Scaffold `frontend/` with Vite Vue-TS template; strip demo code.
2. Add Pinia, vue-router, Vitest, ESLint flat config, font packages.
3. Create `src/` layout per [Layout](#layout) above.
4. Wire `api.ts` fetch wrapper matching backend error shape.
5. Add `useAuthStore` (or equivalent) + `router.ts` guard.
6. Add `styles.css` tokens + at least light/dark themes.
7. Add `theme.ts` + inline boot script in `index.html`.
8. Configure Vite dev proxy for `/api`.
9. Add `nginx.conf` with proxy + SPA fallback.
10. Add `Dockerfile` per [docker.md](docker.md).
11. Wire `ci.yml` frontend job and pre-commit hooks per [pre-commit.md](pre-commit.md).
12. Enable Renovate for npm ([renovate.md](renovate.md)).

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Options API (`export default { data() … }`) | `<script setup lang="ts">` |
| axios / ofetch / custom HTTP libs | Hand-rolled `fetch` in `api.ts` |
| Tailwind / Vuetify / PrimeVue (default) | CSS custom properties design system |
| Hard-coded `http://localhost:8080/api` | Relative `/api/…` + Vite proxy |
| Auth checks in every view | Single `router.beforeEach` guard |
| `localStorage` in components | Auth store owns session |
| Skip loading/error/empty states | Four-state template pattern |
| Set `Content-Type` on `FormData` | Let browser set multipart boundary |
| Global axios interceptors | `authHeaders()` + `ApiError` in one module |
| Theme toggle only in JS | `index.html` inline script + `theme.ts` |
| Eager-import all views | `() => import('./views/…')` lazy routes |
| `npm run build` without typecheck | `vue-tsc --noEmit && vite build` |
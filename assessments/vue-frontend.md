# Vue frontend — project assessment

Per-repo adoption for [../vue-frontend.md](../vue-frontend.md). Last verified: June 2026.

| Repo | Path | Scripts | `api.ts` pattern | Router/views | Notes |
|------|------|---------|------------------|--------------|-------|
| irl-planner-pro | `frontend/` | ✅ incl. `check` | ✅ single `api.ts` + `ApiError` | `views/` + guard | Target |
| plugin-skill-hosting | `frontend/` | ⚠️ `build` no `vue-tsc` | ✅ | `views/` | **Gap:** align `build` with `vue-tsc && vite build` |
| trivia | `frontend/` | ✅ `check` | 🔀 `services/api/*` | **`pages/`** not `views/` | Legacy layout |
| linky | `client/` | ❌ no lint/test/check | 🔀 `api/client.ts` wrapper | `views/` | Not `backend/`/`frontend/` names |
| yt-infographics | — | ❌ **Svelte** | n/a | n/a | **Out of scope** for this doc |
| easy-host-k8s | — | ❌ no SPA | n/a | Go templates | **Out of scope** |
| deep-digest-rss | `news-frontend/` | ⚠️ Prettier | ⚠️ split `api/*.ts` | `pages/` | Prettier present (target says no Prettier) |
| start-renovate | — | ❌ **Nuxt 4** | n/a | `app/pages/` | **Out of scope** — [../nuxt-frontend.md](../nuxt-frontend.md) pattern B |
| homepage-v4 | — | ❌ **Nuxt 4** | n/a | `pages/` at root | **Out of scope** — [../nuxt-frontend.md](../nuxt-frontend.md) pattern A |
| coffee-diary | `frontend/` | ✅ `build` + type-check | ⚠️ `services/*.ts` | `pages/` + meta guards | Session cookies + Pinia; oxlint+eslint; Playwright e2e not Vitest |
| boardwalk-billionaire | `client/` | ✅ `vue-tsc` + build | 🔀 STOMP (`@stomp/stompjs`) | `components/`; no vue-router | Pinia stores; lobby REST + `/ws` proxy; no unit tests |
| cybernight | `frontend/` | ✅ type-check + build | 🔀 STOMP + SockJS | `views/` + Vitest + Playwright | Prettier present; **also ships Pixi `frontend2/`** (oglimmer default) |
| picz | `frontend/` | ✅ type-check + build | ❌ **axios** + OIDC client | `views/` + Vitest | Bootstrap + FontAwesome; `VITE_*` baked in prod Dockerfile |
| picz2 | `frontend/` | ✅ `vue-tsc` + build | ✅ `useApi` fetch composables | `views/` | TUS uploads; no Pinia; closer to guideline fetch pattern |
| video-msg | `frontend/` | ✅ type-check + build | ⚠️ `ApiService` fetch class | `views/` + Vitest + Playwright | Pinia; Prettier present; screen recording upload flow |
| status-tacos | `frontend/` | ✅ type-check + build | ⚠️ `ApiService` fetch + OIDC | `views/` + Vitest + Playwright | Pinia; Prettier; multi-tenant dashboards |

**Disclosed gaps:** align the `plugin-skill-hosting` `build` script; document or migrate the `pages/` vs `views/` naming in `trivia` and `deep-digest-rss`; add lint/test scripts to `linky/client`.

**Valid deviations:** split API modules (`services/api/`, `api/news.ts`) are fine when `ApiError` + auth header logic stay centralized; `pages/` vs `views/` is cosmetic if router lazy-loads consistently.
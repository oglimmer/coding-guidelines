# Nuxt frontend — project assessment

Per-repo adoption for [../nuxt-frontend.md](../nuxt-frontend.md). Last verified: June 2026.

**In scope:** Nuxt 4 repos only. Vue 3 + Vite apps (e.g. `deep-digest-rss/news-frontend`) belong in [vue-frontend.md](vue-frontend.md) assessment.

| Topic | homepage-v4 | start-renovate |
|-------|-------------|----------------|
| Pattern | A — static | B — Spring SPA |
| Framework | ✅ Nuxt 4 | ✅ Nuxt 4 |
| App location | repo root | `frontend/` subfolder |
| Pages layout | `pages/` at root | `app/pages/` |
| Production deploy | `generate` + nginx | Nitro `node-server` |
| Spring / API | ❌ none | ✅ Java `backend/` |
| `server/` routes | `server/routes/rss.get.ts` | `server/api/generate.get.ts` |
| CI | ✅ `ci.yml` (lint, typecheck, build) | ✅ Java + Nuxt in `ci.yml` |
| `build.yml` / ARC | ❌ | ✅ |
| oglimmer image | `homepage-oglimmer-2025` | `start-renovate-fe` |

**homepage-v4 gaps:** no `build.yml` (image push manual or external); CI runs `nuxt build` but Dockerfile runs `nuxt generate` — align before treating as template; `/rss` server route may not exist in static output unless prerendered.

**start-renovate:** reference implementation for pattern B (Nuxt + Spring).
# Coding Guidelines

Precise, opinionated conventions for how we build software. One markdown file per concept.

These documents state **what we build toward** — stack choices, layout, CI shape, deploy plumbing. They stand on their own and do not assume a particular repo layout beyond the paths named in each doc.

**Project assessments** — which live repos match which guidelines, path mapping, gaps, and copy sources: [assessments/](assessments/).

## Docs

| Concept | File | Applies when |
|---------|------|--------------|
| GitHub Actions | [github-actions.md](github-actions.md) | Repo ships workflows |
| Renovate | [renovate.md](renovate.md) | Repo has dependencies |
| Pre-commit | [pre-commit.md](pre-commit.md) | Vue + Go + Helm full-stack |
| Docker | [docker.md](docker.md) | Repo builds container images |
| Helm | [helm.md](helm.md) | Repo deploys to Kubernetes |
| Vue frontend | [vue-frontend.md](vue-frontend.md) | Vue 3 + Vite SPA |
| Nuxt frontend | [nuxt-frontend.md](nuxt-frontend.md) | Nuxt 4 — static (pattern A) or Spring SPA (pattern B) |
| Go backend | [go-backend.md](go-backend.md) | Go HTTP service |
| Java Spring backend | [java-spring-backend.md](java-spring-backend.md) | Spring Boot HTTP API |
| oglimmer.sh | [oglimmer-sh.md](oglimmer-sh.md) | Repo has `oglimmer.sh` |
| Versioning & release | [versioning-release.md](versioning-release.md) | Repo ships versioned artifacts (images, charts, tags) |

## Stack routing

| You are building… | Read |
|-------------------|------|
| Vue 3 SPA + Go API + Helm | [vue-frontend.md](vue-frontend.md) + [go-backend.md](go-backend.md) + [helm.md](helm.md) |
| Vue 3 SPA + Go API + MariaDB | Same docs — see [assessments/repo-map.md](assessments/repo-map.md) (`coffee-diary`) |
| Nuxt 4 static site (blog/marketing) | [nuxt-frontend.md](nuxt-frontend.md) pattern A |
| Nuxt 4 + Spring Boot SPA | [nuxt-frontend.md](nuxt-frontend.md) pattern B + [java-spring-backend.md](java-spring-backend.md) |
| Vue SPA + Spring Boot API | [vue-frontend.md](vue-frontend.md) + [java-spring-backend.md](java-spring-backend.md) |
| Vue `client/` + Spring `server/` (e.g. WebSocket game) | Same docs — see [assessments/repo-map.md](assessments/repo-map.md) (`boardwalk-billionaire`) |
| Vue/Pixi + Spring `backend/` (WebSocket + MariaDB) | Same docs — see [assessments/repo-map.md](assessments/repo-map.md) (`cybernight`) |
| Photo gallery — legacy (`picz`) or K8s rewrite (`picz2`) | [assessments/repo-map.md](assessments/repo-map.md) — **prefer `picz2`** for new work |
| Screen recording (Vue + Go `backend-go/` + FFmpeg) | [vue-frontend.md](vue-frontend.md) + [go-backend.md](go-backend.md) — see `video-msg` in [assessments/repo-map.md](assessments/repo-map.md) |
| Status monitoring (Vue + Spring + OIDC) | [vue-frontend.md](vue-frontend.md) + [java-spring-backend.md](java-spring-backend.md) — see `status-tacos` in [assessments/repo-map.md](assessments/repo-map.md) |

For a specific project's stack and path names, start at [assessments/repo-map.md](assessments/repo-map.md).
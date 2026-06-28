# Project assessments

How live repos compare to the coding guidelines. Guidelines live in the parent directory and stay repo-agnostic; this folder tracks adoption, gaps, and where to copy working examples from.

Last verified: June 2026 (files on disk under `/Users/oli/dev/`).

**Start here:** [repo-map.md](repo-map.md) — stack types, path mapping, maturity tiers, release variants, priority fixes.

## Audited repos

| Repo | Stack | Guidelines that apply |
|------|-------|----------------------|
| **irl-planner-pro** | Vue 3 + Go + Helm + Postgres | **All** — closest full match |
| **plugin-skill-hosting** | Vue 3 + Go + Helm + Postgres | All except full pre-commit; adds e2e CI |
| **trivia** | Vue 3 + Go + Helm + Postgres | Most; legacy frontend/backend layout |
| **linky** | Vue client + Go server + **MySQL** + Helm | Actions, Renovate, Helm, oglimmer (custom paths) |
| **yt-infographics** | **Svelte** + Go (no DB) | Actions, Docker, oglimmer, Helm partially |
| **easy-host-k8s** | Go server-rendered HTML + **MySQL** (`backend-go/`) | Helm, oglimmer partially; **CI is stale** |
| **deep-digest-rss** | Vue + **Java Spring** + **Python** scraper | Actions, Renovate, Docker, Helm, oglimmer (multi-component) |
| **start-renovate** | **Nuxt 4** + **Java Spring Boot** | Actions, Renovate, Docker, Helm, oglimmer |
| **homepage-v4** | **Nuxt 4** static site (no API) | Actions, Renovate, Docker, Helm, oglimmer (single image) |
| **coffee-diary** | Vue 3 + Go + **MariaDB** + Helm; **SwiftUI** `ios/` | Renovate, Docker, Helm, oglimmer; **no workflows on disk** |
| **boardwalk-billionaire** | Vue 3 `client/` + **Java Spring** `server/` | Actions, Renovate, Docker, Helm, oglimmer; STOMP/WebSocket game, no DB |
| **cybernight** | Vue `frontend/` + **PixiJS** `frontend2/` + **Java Spring** `backend/` | Renovate, Docker, Helm, oglimmer, pre-commit; MariaDB; **no CI workflows** |
| **picz** | Vue `frontend/` + **Gradle Spring** `backend/` | Renovate, Docker, compose, pre-commit; **AWS Terraform/Ansible** deploy; legacy |
| **picz2** | Vue `frontend/` + **Maven Spring** `server/` + **SwiftUI** `ios/` | Renovate, Docker, Helm (5 workloads), oglimmer, pre-commit; MinIO+MariaDB; **no CI** |
| **video-msg** | Vue `frontend/` + Go `backend-go/` (+ deprecated Java `backend/`) | Renovate, Docker, Helm, oglimmer, pre-commit; MariaDB; **stale `ci.yml`** (tests Java only) |
| **status-tacos** | Vue `frontend/` + **Java Spring** `backend/` + **Swift** `notifier-app/` | Renovate, Docker, Helm, oglimmer, pre-commit; MariaDB+OIDC; **no CI** |

## Adoption matrix

Legend: ✅ matches target · ⚠️ partial · ❌ missing or N/A · 🔀 different stack

| Area | irl-planner-pro | plugin-skill-hosting | trivia | linky | yt-infographics | easy-host-k8s | deep-digest-rss | start-renovate | homepage-v4 | coffee-diary | boardwalk-billionaire | cybernight | picz | picz2 | video-msg | status-tacos |
|------|-----------------|----------------------|--------|-------|-----------------|---------------|-----------------|----------------|--------------|----------------|---------------------|------------|------|------|-----------|--------------|
| GitHub Actions | ✅ | ⚠️ no actionlint | ⚠️ no release/cleanup | ⚠️ old action pins | ⚠️ no release/cleanup | ❌ build on hosted runner | 🔀 `pr.yml` not `ci.yml` | ⚠️ Java+Nuxt CI | ⚠️ `ci.yml` only, no `build.yml` | ❌ **no `.github/`** | ⚠️ `ci.yaml`; ARC+hook ✅ | ❌ **no workflows** | ❌ | ❌ | ⚠️ **stale `ci.yml`** | ❌ **no workflows** |
| Renovate | ✅ | ⚠️ no PG major freeze | ⚠️ no PG major freeze | ✅ | ❌ no file | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Pre-commit | ✅ | ⚠️ minimal | ⚠️ minimal | ❌ | ❌ | ❌ | 🔀 Java hooks | 🔀 Java+Nuxt hooks | ❌ | ❌ | ❌ | 🔀 Java+Vue; no yamllint | 🔀 Gradle+Vue | 🔀 Maven+Vue; helm-lint | 🔀 Java on deprecated BE; no go | 🔀 Maven+Vue; helm-lint |
| Docker | ✅ | ✅ | ⚠️ no .dockerignore | 🔀 MariaDB compose | ⚠️ single-stage BE | 🔀 `backend-go/` | 🔀 3 components | 🔀 Java+Nuxt images | ✅ generate+nginx | ⚠️ alpine BE; FE `.dockerignore` only | ✅ both; Java JAR image | ⚠️ unpinned tags | ✅ Gradle JAR | ✅ Maven JAR | 🔀 Go alpine+ffmpeg; split compose | ✅ JAR; FE skips lint in image |
| Helm | ✅ | ✅ | ⚠️ no `helm/argocd/` | ⚠️ no argocd/ | ⚠️ no argocd/ | ⚠️ single app chart | 🔀 multi-deploy chart | ⚠️ seal at `helm/` root | ✅ single app | ✅ in-chart SealedSecret | ✅ `/ws` before `/api` | ✅ SealedSecret | ❌ **AWS deploy** | 🔀 **5 workloads** (api/worker/tusd/…) | ✅ PVC storage; seal at root | ✅ chart at `helm/` root |
| Vue frontend | ✅ | ⚠️ build skips tsc | ⚠️ `pages/`, split API | ⚠️ axios-style client | ❌ Svelte | ❌ no SPA | ⚠️ `news-frontend` | ❌ use nuxt-frontend | ❌ Nuxt | ⚠️ `pages/`, session+Pinia, Playwright | 🔀 `client/`; STOMP | ⚠️ + Pixi `frontend2/` | ⚠️ **axios** + Bootstrap | ⚠️ fetch composables; TUS; no Pinia | ⚠️ `ApiService` fetch; Pinia | ⚠️ fetch + OIDC; Pinia |
| Nuxt frontend | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ pattern B | ✅ pattern A | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Go backend | ✅ | ⚠️ `cmd/marketplace` | ⚠️ pgxpool, fs migrations | 🔀 MySQL/sqlx | ⚠️ root `main.go` | 🔀 MySQL, templates | ❌ use java-spring | ❌ use java-spring | ❌ | 🔀 MariaDB | ❌ | ❌ | ❌ | ❌ | 🔀 **`backend-go/`** MariaDB | ❌ |
| Java Spring backend | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ `news-backend` | ✅ `backend/` | ❌ | ❌ | ⚠️ `server/`; WebSocket | ✅ `backend/`; STOMP | 🔀 **Gradle** `backend/` | ✅ `server/`; MinIO+MariaDB | 🔀 deprecated `backend/` | ✅ `backend/`; OIDC |
| oglimmer.sh | ✅ | ✅ | ⚠️ `K8S_NAMESPACE` | 🔀 client/server | ✅ | ❌ no restart hook | ✅ multi-comp B | ✅ `-fe`/`-be` | ✅ shape C | ⚠️ kubectl only | ✅ A′ + hook | ⚠️ vue\|pixi; kubectl | ❌ **none** (`build/` AWS) | ⚠️ api+worker; kubectl only | ⚠️ `backend-go`; kubectl | ⚠️ kubectl; `release` |

## Per-topic assessments

| Guideline | Assessment |
|-----------|------------|
| GitHub Actions | [github-actions.md](github-actions.md) |
| Renovate | [renovate.md](renovate.md) |
| Pre-commit | [pre-commit.md](pre-commit.md) |
| Docker | [docker.md](docker.md) |
| Helm | [helm.md](helm.md) |
| Vue frontend | [vue-frontend.md](vue-frontend.md) |
| Nuxt frontend | [nuxt-frontend.md](nuxt-frontend.md) |
| Go backend | [go-backend.md](go-backend.md) |
| Java Spring backend | [java-spring-backend.md](java-spring-backend.md) |
| oglimmer.sh | [oglimmer-sh.md](oglimmer-sh.md) |
| Versioning & release | [versioning-release.md](../versioning-release.md) + [repo-map.md](repo-map.md) release variants |

## Reference repos by topic

| Topic | Copy from |
|-------|-----------|
| Full stack (everything) | `irl-planner-pro` |
| Closest Vue+Go sibling | `plugin-skill-hosting` |
| Multi-component images + deploy | `deep-digest-rss/oglimmer.sh` |
| Private registry + ARC | `trivia`, `start-renovate`, `deep-digest-rss` |
| Nuxt static site (generate + nginx) | `homepage-v4` |
| Nuxt + Spring SPA | `start-renovate` |
| Vue + Spring API | `deep-digest-rss` |
| ghcr.io + release + cleanup | `irl-planner-pro`, `plugin-skill-hosting` |
| Vue + Go + MariaDB session (OIDC) | `coffee-diary` |
| Vue `client/` + Spring `server/` (WebSocket) | `boardwalk-billionaire` |
| Vue/Pixi + Spring `backend/` (WebSocket + MariaDB) | `cybernight` |
| Legacy photo app (Gradle + AWS) | `picz` |
| Photo upload v2 (multi-workload Helm + TUS) | `picz2` |
| Screen recording (Vue + Go `backend-go/` + FFmpeg) | `video-msg` |
| Status monitoring (Vue + Spring + OIDC + MariaDB) | `status-tacos` |

## Known org-wide gaps (June 2026)

1. **Only `irl-planner-pro` has the full GitHub Actions quintet** (`ci`, `build`, `release`, `cleanup-images`, `actionlint`). Siblings lack `release.yml`, `cleanup-images.yml`, and/or `actionlint.yaml`. No audited repo yet ships optional **security scan** (Trivy/OSV) or **manual environment e2e** workflows — see [github-actions.md](../github-actions.md) supplementary section.
2. **Pre-commit trio** (`.pre-commit-config.yaml`, `.yamllint.yaml`, `.shellcheckrc`) exists only on `irl-planner-pro`.
3. **`helm/argocd/` GitOps layout** exists only on `irl-planner-pro` and `plugin-skill-hosting`. Others seal secrets in-chart or at `helm/` root.
4. **Postgres major freeze in `renovate.json`** exists only on `irl-planner-pro`, despite bundled Postgres on `trivia` and `plugin-skill-hosting`.
5. **`yt-infographics` has no `renovate.json`** and no pre-commit.
6. **`easy-host-k8s` CI is broken/stale** — `.github/workflows/build.yml` runs Maven in a `backend/` directory that no longer exists; app code is `backend-go/`. Also builds on `ubuntu-latest` instead of ARC + `oglimmer.sh`; `oglimmer.sh` has kubectl-only restart (no hook).
7. **`coffee-diary` has no `.github/workflows/`** — README references `build.yml` that is not on disk; add `ci.yml` + ARC `build.yml`, restart hook in `oglimmer.sh`, pre-commit baseline.
8. **`cybernight` has no `.github/workflows/`** — add `ci.yml` + ARC `build.yml` + restart hook; pin Docker base images; document `frontend2/` (Pixi) vs `frontend/` (Vue) deploy default.
9. **`picz2` has no `.github/workflows/`** — add `ci.yml` + ARC `build.yml` + restart hook; onboard `arc-picz2` if missing.
10. **`picz` is legacy** — Gradle (not Maven) backend; deploy via `build/terraform` + `build/ansible` (AWS), not oglimmer/Helm; successor is `picz2`.
11. **`video-msg` CI is stale** — `ci.yml` runs Maven tests in deprecated Java `backend/` on `ubuntu-latest`; active code is Go `backend-go/`. Rewrite CI + add ARC `build.yml` + restart hook.
12. **`status-tacos` has no `.github/workflows/`** — add `ci.yml` + ARC `build.yml` + restart hook.
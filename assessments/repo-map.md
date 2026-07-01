# Repo map

How audited org repos map to the coding guidelines. Read this before copying patterns from a sibling — **path names and stacks differ**.

Last verified: June 2026 (files on disk under `/Users/oli/dev/`).

## Which docs apply?

```
Does the repo ship a Nuxt 4 app (repo root or frontend/)?
  yes → ../nuxt-frontend.md (homepage-v4: pattern A static; start-renovate: pattern B + Spring)

Does the repo ship a Vue 3 SPA in frontend/ (or equivalent)?
  yes → ../vue-frontend.md (with path mapping below)
  no  → skip (Svelte: yt-infographics; none: easy-host-k8s)

Does the repo ship a Java Spring Boot API in backend/ (or equivalent)?
  yes → ../java-spring-backend.md (path: `backend/`, `news-backend/`, or `server/`)

Does the repo ship a Go HTTP API in backend/ (or equivalent)?
  yes → ../go-backend.md (with path mapping below)
  no  → skip (MySQL/MariaDB Go: linky, easy-host-k8s, coffee-diary)

Does the repo build container images?
  yes → ../docker.md + ../oglimmer-sh.md

Does the repo deploy to Kubernetes?
  yes → ../helm.md

All repos with dependencies:
  ../renovate.md

All repos with .github/workflows/:
  ../github-actions.md

Only irl-planner-pro today (target for Vue+Go siblings):
  ../pre-commit.md full stack
```

## Stack reference

| Repo | UI | API | DB | Deploy | Registry |
|------|----|-----|-----|--------|----------|
| irl-planner-pro | Vue 3 `frontend/` | Go `backend/` | Postgres | Helm + argocd/ | ghcr.io |
| plugin-skill-hosting | Vue 3 `frontend/` | Go `backend/` (`cmd/marketplace`) | Postgres | Helm + argocd/ | ghcr.io |
| trivia | Vue 3 `frontend/` (`pages/`) | Go `backend/` (`internal/api`) | Postgres | Helm | registry.oglimmer.com |
| linky | Vue 3 `client/` | Go `server/` | **MySQL** | Helm | registry.oglimmer.com |
| yt-infographics | **Svelte** `frontend/` | Go `backend/` (thin) | none | Helm | registry.oglimmer.com |
| easy-host-k8s | Go templates `backend-go/templates/` | Go `backend-go/` | **MySQL** | Helm single chart | registry.oglimmer.com |
| deep-digest-rss | Vue `news-frontend/` | **Java** `news-backend/` | **MariaDB** + Redis | Helm multi-deploy | registry.oglimmer.com |
| start-renovate | **Nuxt 4** `frontend/` | **Java** `backend/` | Postgres | Helm | registry.oglimmer.com |
| homepage-v4 | **Nuxt 4** (repo root) | — | — | Helm | registry.oglimmer.com |
| coffee-diary | Vue 3 `frontend/` + **SwiftUI** `ios/` | Go `backend/` | **MariaDB** | Helm | registry.oglimmer.com |
| boardwalk-billionaire | Vue 3 `client/` | **Java Spring** `server/` | none (in-memory) | Helm | registry.oglimmer.com |
| cybernight | Vue `frontend/` + Pixi `frontend2/` | **Java Spring** `backend/` | **MariaDB** (ext) | Helm | registry.oglimmer.com |
| picz | Vue `frontend/` | **Gradle Spring** `backend/` | **MariaDB** + S3 | **AWS** (Terraform/Ansible) | — |
| picz2 | Vue `frontend/` + **SwiftUI** `ios/` | **Maven Spring** `server/` | **MariaDB** + **MinIO** | Helm (5 workloads) | registry.oglimmer.com |
| video-msg | Vue `frontend/` | Go **`backend-go/`** (+ deprecated Java `backend/`) | **MariaDB** | Helm `video-msg` | registry.oglimmer.com |
| status-tacos | Vue `frontend/` + **Swift** `notifier-app/` | **Java Spring** `backend/` | **MariaDB** | Helm (chart at `helm/` root) | registry.oglimmer.com |

**picz → picz2:** same product family; `picz` is the legacy stack, `picz2` (`photo-upload` chart) is the K8s rewrite.

**video-msg:** the Go rewrite in `backend-go/` is active; Java `backend/` is deprecated but still wired into a stale `ci.yml` and pre-commit.

## Path mapping (guideline path → repo path)

| Guideline assumes | irl-planner-pro | plugin-skill-hosting | trivia | linky | yt-infographics | easy-host-k8s | deep-digest-rss | start-renovate | homepage-v4 | coffee-diary | boardwalk-billionaire | cybernight | picz | picz2 | video-msg | status-tacos |
|-------------------|-----------------|----------------------|--------|-------|-----------------|---------------|-----------------|----------------|--------------|----------------|---------------------|------------|------|------|-----------|--------------|
| `frontend/` | ✅ | ✅ | ✅ | `client/` | ✅ (Svelte) | — | `news-frontend/` | ✅ (Nuxt) | **repo root** | ✅ | `client/` | ✅ (+ `frontend2/`) | ✅ | ✅ | ✅ | ✅ |
| `backend/` | ✅ | ✅ | ✅ | `server/` (Go) | ✅ | `backend-go/` | `news-backend/` (Java) | `backend/` (Java) | — | ✅ | `server/` (Java) | ✅ (Java) | ✅ (**Gradle**) | `server/` (Java) | **`backend-go/`** (Go) | ✅ (Java) |
| `cmd/server/` | ✅ | `cmd/marketplace/` | ✅ | `cmd/linky/` | ❌ `main.go` | `backend-go/cmd/server/` | — | — | — | ✅ | — | — | — | — | ✅ | — |
| `internal/server/` | ✅ | ✅ | ❌ `internal/api` | `server/internal/` | partial | `handler/`/`store/` | — | — | — | ❌ `internal/handler` | — | — | — | — | ❌ `internal/handler` | — |
| `helm/<chart>/` | `irl-planner-pro` | `plugin-skill-hosting` | `trivia` | `linky` | `yt-infographics` | `easy-host` | `deep-digest-rss` | `start-renovate` | `helm/` | `coffee-diary` | `boardwalk-billionaire` | `cybernight` | — | `photo-upload` | `video-msg` | **`helm/`** (root) |
| `helm/argocd/` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Default branch | `main` | `master` | `master` | `master` | `master` | `main` | `main` | `main` | `main` | `master` | `main` | `main` | `master` | `main` | `main` | `main` |

## Maturity tiers (deploy plumbing)

Use tiers to plan bring-up work — they are not a judgment of app quality.

| Tier | Definition | Repos today |
|------|------------|-------------|
| **3 — Full** | ci + build (ARC) + release + cleanup + actionlint + full pre-commit + argocd/ + oglimmer restart hook | **irl-planner-pro** |
| **2 — Deployable** | ci + build (ARC) + oglimmer restart hook + renovate + helm; missing release/cleanup/actionlint/pre-commit | plugin-skill-hosting, trivia, linky, yt-infographics, deep-digest-rss, start-renovate, **boardwalk-billionaire** |
| **1 — Broken plumbing** | oglimmer or CI out of sync with repo layout | **easy-host-k8s** (CI builds non-existent `backend/` with Maven; code is `backend-go/`); **video-msg** (same pattern — `ci.yml` still tests deprecated Java `backend/`); **coffee-diary**, **cybernight**, **status-tacos** (no `.github/workflows/`) |
| **0 — Out of scope** | Doc targets a different stack | Svelte/MySQL Go; **picz** AWS/Gradle legacy; Java/Nuxt in dedicated docs |

## `oglimmer.sh` shape by repo

| Shape | Repos | Commands | Notes |
|-------|-------|----------|-------|
| **A** full-stack | irl-planner-pro, plugin-skill-hosting, trivia, yt-infographics, start-renovate, **coffee-diary**, **cybernight**, **video-msg**, **status-tacos** | `build`, `release`‡, `helm-push`†, dev cmds§ | `-f`/`-b`/`-a`; `PLATFORM`; cybernight adds `--vue`/`--pixi`; video-msg builds **`backend-go/`** |
| **A′** client/server | linky, **boardwalk-billionaire** | same | `-f`=client, `-b`=server; linky ~1400 lines |
| **B** multi-component | deep-digest-rss, **picz2** | `build` (+ `release` on picz2) | picz2: api+worker same image; tusd + retention in Helm only |
| **C** single image | easy-host-k8s, **homepage-v4** | `build` | one component; homepage = static Nuxt only |

† `helm-push`: irl-planner-pro, plugin-skill-hosting. start-renovate / coffee-diary release does not helm-push.  
‡ coffee-diary `release`: local tag + `execute_build` only (no `git push`, no `release.yml`) — uses `backend/VERSION` not only `package.json`.  
§ Dev `start`/`stop`/…: irl-planner-pro, plugin-skill-hosting (Go local loop).

## Release flow variants

The `release` commands differ from repo to repo:

| Variant | Repos | What happens |
|---------|-------|--------------|
| **Full GitOps** | irl-planner-pro, plugin-skill-hosting | bump `package.json` + `Chart.yaml` → commit → tag `v*` → **push tag** → `release.yml` multi-arch → `helm-push` |
| **Local tag + build** | yt-infographics | bump → commit → tag → **`execute_build` only** — no `git push`, no `release.yml`, no helm-push (tags exist in git history) |
| **Interactive bump + build** | start-renovate, **coffee-diary**, **video-msg**, **status-tacos** | tag + build locally; no `helm-push`; video-msg bumps `backend-go/VERSION`; status-tacos bumps `backend/pom.xml` + `package.json` |
| **None** | deep-digest-rss, linky, trivia, easy-host-k8s | manual tags or no release command |

## CI workflow naming

| Pattern | Repos | Notes |
|---------|-------|-------|
| `ci.yml` (push + PR) | irl-planner-pro, plugin-skill-hosting, trivia, linky, yt-infographics, start-renovate | Standard target |
| `ci.yaml` (push + PR) | boardwalk-billionaire | Same shape; nonstandard filename |
| `pr.yml` (PR only) | deep-digest-rss | No push-to-main CI gate — only PR checks |
| `build.yml` only | easy-host-k8s | **Stale:** Maven test + docker on `ubuntu-latest`; no `oglimmer.sh`, no ARC |
| `ci.yml` (stale backend path) | **video-msg** | Runs Maven in deprecated Java `backend/` on `ubuntu-latest` — active code is Go `backend-go/` |
| **None** | **coffee-diary**, **cybernight**, **picz**, **picz2**, **status-tacos** | No `.github/workflows/` on disk (video-msg has only stale `ci.yml`) |

## Priority fixes (disclosed gaps)

Ordered by impact:

1. **easy-host-k8s** — Rewrite `.github/workflows/build.yml`: drop Maven `backend/`, use ARC + `./oglimmer.sh build --platform auto`, add restart hook to `oglimmer.sh`.
2. **yt-infographics** — Add `renovate.json`; add `release.yml` or document intentional local-only release; add pre-commit baseline.
3. **trivia + plugin-skill-hosting** — Add Postgres major-freeze rule to `renovate.json` (both bundle Postgres in Helm).
4. **Vue+Go siblings** — Copy `irl-planner-pro` pre-commit trio; add `actionlint.yaml` where ARC labels exist.
5. **ghcr.io apps** — Add `cleanup-images.yml` where missing (only irl-planner-pro + plugin-skill-hosting have it today).
6. **deep-digest-rss** — Decide: rename `pr.yml` → `ci.yml` + main push, or document PR-only CI as intentional.
7. **coffee-diary** — Add `ci.yml` + ARC `build.yml`; add restart hook to `oglimmer.sh`; align README with on-disk workflows.
8. **cybernight** — Add `ci.yml` + ARC `build.yml` + restart hook; pin `nginx:latest` / `mariadb:latest` in Docker/compose.
9. **picz2** — Add `ci.yml` + ARC `build.yml` + restart hook.
10. **picz** — Legacy; new work should target `picz2`. Do not copy AWS `build/` as org default.
11. **video-msg** — Rewrite `ci.yml` for Go `backend-go/`; add ARC `build.yml` + restart hook; add gofmt/vet pre-commit hooks; remove or quarantine deprecated Java `backend/`.
12. **status-tacos** — Add `ci.yml` + ARC `build.yml` + restart hook.

## Safe copy sources

| You need… | Copy from |
|-----------|-----------|
| Everything | `irl-planner-pro` |
| Vue+Go sibling closest to target | `plugin-skill-hosting` |
| Private registry ARC build | `trivia` or `start-renovate` `build.yml` |
| Multi-component oglimmer | `deep-digest-rss/oglimmer.sh` |
| Nuxt 4 static site (generate + nginx) | `homepage-v4` → [nuxt-frontend.md](../nuxt-frontend.md) pattern A |
| Nuxt 4 + Spring SPA (Nitro) | `start-renovate/frontend` → [nuxt-frontend.md](../nuxt-frontend.md) pattern B |
| Spring Boot OAuth SPA backend | `start-renovate/backend` → [java-spring-backend.md](../java-spring-backend.md) |
| Spring Boot API + Redis session + MCP | `deep-digest-rss/news-backend` — deviates from canonical DB/auth/layout, copy MCP add-on only; see [assessments/java-spring-backend.md](java-spring-backend.md) |
| ghcr release + cleanup | `irl-planner-pro` or `plugin-skill-hosting` `release.yml` |
| Vue + Go + MariaDB (session/OIDC) | `coffee-diary` → [vue-frontend.md](../vue-frontend.md) + [go-backend.md](../go-backend.md) (MySQL paths) |
| Vue `client/` + Spring `server/` (STOMP) | `boardwalk-billionaire` → [vue-frontend.md](../vue-frontend.md) + [java-spring-backend.md](../java-spring-backend.md) |
| Vue + Spring `backend/` (STOMP + MariaDB) | `cybernight` → [vue-frontend.md](../vue-frontend.md) + [java-spring-backend.md](../java-spring-backend.md) (MariaDB is a documented deviation, see [assessments/java-spring-backend.md](java-spring-backend.md)) |
| Photo gallery (Helm api+worker+tusd) | `picz2` → [java-spring-backend.md](../java-spring-backend.md) + [helm.md](../helm.md) multi-deploy |
| Legacy Gradle + AWS photo app | `picz` — reference only; prefer `picz2` |
| Vue + Go `backend-go/` + MariaDB (FFmpeg processing) | `video-msg` → [vue-frontend.md](../vue-frontend.md) + [go-backend.md](../go-backend.md) |
| Vue + Spring status monitoring (OIDC, multi-tenant) | `status-tacos` → [vue-frontend.md](../vue-frontend.md) + [java-spring-backend.md](../java-spring-backend.md) |

**Out of scope:** `coffee-diary/ios/`, `picz2/ios/` (SwiftUI); `status-tacos/notifier-app/` (Swift); `cybernight/frontend-ex-unity/` (Unity); `cybernight/computer-player/`; `picz/build/terraform` (AWS); `video-msg/backend/` (deprecated Java) — no dedicated guidelines yet.

**Do not copy** `easy-host-k8s/.github/workflows/build.yml` until it is rewritten.
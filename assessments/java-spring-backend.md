# Java Spring backend — project assessment

Per-repo adoption for the single canonical pattern in [../java-spring-backend.md](../java-spring-backend.md): Postgres + Flyway, OAuth2 login + session, global `/api` context-path, `controller`/`service`/`repository`/`entity`/`dto` layout. Last verified: July 2026.

## Canonical-pattern deviations at a glance

| Repo | Path | DB | Auth | Context-path | Layout | Matches canonical |
|------|------|----|------|---------------|--------|--------------------|
| start-renovate | `backend/` | ✅ Postgres + Flyway | ✅ OAuth2 (GitHub) + session | ✅ global `/api` | ✅ `controller`/`service`/`repository`/`entity`/`dto` | ✅ full match — this is the reference implementation |
| deep-digest-rss | `news-backend/` | ❌ MariaDB + `flyway-mysql` | ❌ form login + Redis session + API keys | ❌ no global context-path; `/api/v1/...` per-controller | ❌ `web`/`db` (flattened) | ⚠️ intentional deviation, see below |
| boardwalk-billionaire | `server/` | ❌ none (in-memory `SessionManager`, no Flyway) | ❌ REST lobby + STOMP/WebSocket game, no OAuth2 | N/A | 🔀 game-server shape, not this doc's target | 🔀 out of scope for this doc |
| cybernight | `backend/` | ⚠️ MariaDB + Flyway + JPA (Postgres is canonical) | ⚠️ OAuth2 (Discord) + STOMP/WebSocket game | ✅ `/api` (+ `/ws`, `/oauth2`, `/login`) | Unverified | ⚠️ partial — DB dialect deviation |
| picz | `backend/` (Gradle) | ⚠️ MariaDB + S3, **Gradle** not Maven | ❌ OIDC (Keycloak) | Unverified | Unverified | 🔀 legacy, AWS deploy — do not use as a reference |
| picz2 | `server/` | ⚠️ MariaDB + MinIO | Unverified | Unverified | Unverified | ⚠️ multi-workload shape (api + worker), not a plain API service |
| status-tacos | `backend/` | ⚠️ MariaDB + Flyway + JPA | ❌ OIDC bearer tokens | ✅ `/api` | Unverified | ⚠️ partial — DB dialect + auth deviation |
| video-msg | `backend/` (deprecated) | N/A | N/A | N/A | N/A | Deprecated — active code is Go `backend-go/`; do not copy Java patterns from here |

### start-renovate (`backend/`) — reference implementation

| Topic | Status |
|-------|--------|
| CI gate | ✅ `ci.yml` push+PR |
| Spotless | ✅ |
| Testcontainers in CI | H2 tests |
| Non-root Docker user | ✅ |

### deep-digest-rss (`news-backend/`) — documented deviation

Built before the org standardized on one Spring pattern. Keep running as-is; **do not copy its DB/auth/layout choices into new repos** — new Spring services follow the canonical pattern in [../java-spring-backend.md](../java-spring-backend.md) and add Spring AI/MCP as an optional add-on on top of it, not as a reason to also switch DB/auth/layout.

| Topic | Status | Why it deviates |
|-------|--------|------------------|
| DB | MariaDB + `flyway-mysql` | Predates the Postgres default; no plan to migrate a live DB for a doc-conformance reason alone |
| Auth | Form login + Redis session + API keys | API keys needed for MCP machine clients; form login predates the OAuth2 default |
| Context-path | No global `/api`; `/api/v1/...` via per-controller `@RequestMapping` | Predates the context-path convention |
| Layout | `web/` (controllers+dto) and `db/` (entities+repositories), not the 5-package split | Predates the layout convention |
| MCP / Spring AI | ✅ present | This part **is** the model for the "optional add-on" section in the canonical doc — copy this if you're adding MCP to a new repo, just onto the canonical DB/auth/layout, not this repo's variant |
| CI gate | ⚠️ `pr.yml` PR-only | |
| Testcontainers in CI | ✅ `RUN_INTEGRATION_TESTS` | |
| Non-root Docker user | ❌ gap | |

### boardwalk-billionaire (`server/`)

| Topic | Detail |
|-------|--------|
| Path | `server/` (not `backend/`) |
| CI gate | ✅ `ci.yaml` + `build.yml` ARC |
| Spotless | ❌ |
| Persistence | ❌ in-memory (`SessionManager`); no Flyway |
| API style | REST lobby + **STOMP/WebSocket** game — out of scope for the SPA/OAuth2 pattern this doc targets |
| Docker | ✅ non-root JRE alpine |
| Ingress | ✅ `/ws`, `/api`, `/admin` before SPA `/` |

### cybernight (`backend/`)

| Topic | Detail |
|-------|--------|
| Path | `backend/` |
| CI gate | ❌ no workflows on disk |
| Spotless | ✅ (pre-commit + pom) |
| Persistence | MariaDB + Flyway + JPA — **deviates from canonical Postgres**; no migration planned, real-time game state doesn't need it urgently |
| API style | STOMP/WebSocket game + OAuth2 (Discord) |
| Docker | ✅ non-root Temurin 25 JAR |
| Ingress | ✅ `/ws`, `/api`, `/oauth2`, `/login` before SPA |
| Extras | `computer-player/` Java bot; not in standard two-image chart |

### picz (`backend/` — Gradle)

| Topic | Detail |
|-------|--------|
| Path | `backend/` |
| Build | **Gradle** (`./gradlew`), not Maven — java-spring doc assumes `mvnw` |
| CI / deploy | ❌ no workflows; **AWS** via `build/terraform` + `build/ansible` |
| Spotless | ✅ Gradle plugin (pre-commit) |
| Persistence | MariaDB + S3 (s3fs-go in compose) |
| Auth | OIDC (Keycloak) — frontend uses `oidc-client-ts` |
| Status | Legacy; successor is picz2. Do not use as a reference for new repos |

### picz2 (`server/`)

| Topic | Detail |
|-------|--------|
| Path | `server/` |
| CI gate | ❌ no workflows on disk |
| Spotless | ✅ Maven (pre-commit) |
| Persistence | MariaDB + **MinIO** (S3-compatible) |
| Workloads | Same JAR → **api** + **worker** pods (`SPRING_PROFILES_ACTIVE`); tusd + retention CronJob in Helm |
| Docker | `picz2-be` / `picz2-fe` via oglimmer |
| Extras | `ios/` SwiftUI primary client; TUS resumable uploads |

### status-tacos (`backend/`)

| Topic | Detail |
|-------|--------|
| Path | `backend/` |
| CI gate | ❌ no workflows on disk |
| Spotless | ✅ Maven (pre-commit) |
| Persistence | MariaDB + Flyway + JPA — **deviates from canonical Postgres** |
| API style | REST + OIDC bearer tokens (not OAuth2 login + session) — **deviates from canonical auth** |
| Context-path | ✅ `/api` |
| Docker | ✅ non-root JRE alpine |
| Ingress | ✅ `/api` before SPA `/` |
| Extras | `notifier-app/` Swift iOS alert client; Teams notification integration |

### video-msg (`backend/` — deprecated)

| Topic | Detail |
|-------|--------|
| Path | `backend/` (deprecated) — active code is Go `backend-go/` |
| CI gate | ⚠️ stale `ci.yml` still runs Maven tests here |
| Note | Do not copy Java patterns; migrate CI/pre-commit to `backend-go/` |

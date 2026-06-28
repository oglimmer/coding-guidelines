# Java Spring backend — project assessment

Per-repo adoption for [../java-spring-backend.md](../java-spring-backend.md). Last verified: June 2026.

| Topic | start-renovate | deep-digest-rss |
|-------|----------------|-----------------|
| Path | `backend/` | `news-backend/` |
| CI gate | ✅ `ci.yml` push+PR | ⚠️ `pr.yml` PR-only |
| Spotless | ✅ | ✅ |
| Testcontainers in CI | H2 tests | ✅ `RUN_INTEGRATION_TESTS` |
| Context-path `/api` | ✅ | ❌ (path on controllers) |
| Non-root Docker user | ✅ | ❌ gap |
| MCP / Spring AI | ❌ | ✅ |
| Matches go-backend doc | ❌ intentional | ❌ intentional |

### boardwalk-billionaire (`server/`)

| Topic | boardwalk-billionaire |
|-------|----------------------|
| Path | `server/` (not `backend/`) |
| CI gate | ✅ `ci.yaml` + `build.yml` ARC |
| Spotless | ❌ |
| Persistence | ❌ in-memory (`SessionManager`); no Flyway |
| API style | REST lobby + **STOMP/WebSocket** game — not OAuth SPA session doc |
| Docker | ✅ non-root JRE alpine |
| Ingress | ✅ `/ws`, `/api`, `/admin` before SPA `/` |

### cybernight (`backend/`)

| Topic | cybernight |
|-------|------------|
| Path | `backend/` |
| CI gate | ❌ no workflows on disk |
| Spotless | ✅ (pre-commit + pom) |
| Persistence | ✅ MariaDB + Flyway + JPA |
| API style | STOMP/WebSocket game + OAuth2 (Discord); variant B-ish |
| Docker | ✅ non-root Temurin 25 JAR |
| Ingress | ✅ `/ws`, `/api`, `/oauth2`, `/login` before SPA |
| Extras | `computer-player/` Java bot; not in standard two-image chart |

### picz (`backend/` — Gradle)

| Topic | picz |
|-------|------|
| Path | `backend/` |
| Build | **Gradle** (`./gradlew`), not Maven — java-spring doc assumes `mvnw` |
| CI / deploy | ❌ no workflows; **AWS** via `build/terraform` + `build/ansible` |
| Spotless | ✅ Gradle plugin (pre-commit) |
| Persistence | MariaDB + S3 (s3fs-go in compose) |
| Auth | OIDC (Keycloak) — frontend uses `oidc-client-ts` |

### picz2 (`server/`)

| Topic | picz2 |
|-------|------|
| Path | `server/` |
| CI gate | ❌ no workflows on disk |
| Spotless | ✅ Maven (pre-commit) |
| Persistence | MariaDB + **MinIO** (S3-compatible) |
| Workloads | Same JAR → **api** + **worker** pods (`SPRING_PROFILES_ACTIVE`); tusd + retention CronJob in Helm |
| Docker | `picz2-be` / `picz2-fe` via oglimmer |
| Extras | `ios/` SwiftUI primary client; TUS resumable uploads |

### status-tacos (`backend/`)

| Topic | status-tacos |
|-------|--------------|
| Path | `backend/` |
| CI gate | ❌ no workflows on disk |
| Spotless | ✅ Maven (pre-commit) |
| Persistence | ✅ MariaDB + Flyway + JPA |
| API style | REST + OIDC bearer tokens; context-path `/api` |
| Docker | ✅ non-root JRE alpine |
| Ingress | ✅ `/api` before SPA `/` |
| Extras | `notifier-app/` Swift iOS alert client; Teams notification integration |

### video-msg (`backend/` — deprecated)

| Topic | video-msg |
|-------|-----------|
| Path | `backend/` (deprecated) — active code is Go `backend-go/` |
| CI gate | ⚠️ stale `ci.yml` still runs Maven tests here |
| Note | Do not copy Java patterns; migrate CI/pre-commit to `backend-go/` |
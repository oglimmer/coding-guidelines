# Go backend — project assessment

Per-repo adoption for [../go-backend.md](../go-backend.md). Last verified: July 2026 (DB/pooler config re-checked against disk).

| Repo | Layout | Router/HTTP | DB | Migrations | Notes |
|------|--------|-------------|-----|------------|-------|
| irl-planner-pro | ✅ `cmd/server` + `internal/server` | chi + `server/` | ⚠️ pgx **stdlib adapter**, pooler-safe ✅ | ✅ embedded | Dual-env safe; stdlib adapter is legacy but not a bug |
| plugin-skill-hosting | ⚠️ `cmd/marketplace` | ✅ `internal/server` | ⚠️ pgx **stdlib adapter**, pooler-safe ✅ | ✅ embedded | Dual-env safe; rename cmd is cosmetic |
| trivia | ⚠️ `internal/api` not `server` | chi in api pkg | ✅ pgxpool — ❌ **not pooler-safe** | 🔀 **filesystem** `migrations/` | **Breaks behind PgBouncer** — see gaps below |
| linky | 🔀 `server/`, `cmd/linky` | chi | **MySQL** + sqlx | golang-migrate embed | **Different stack** |
| yt-infographics | ⚠️ root `main.go` | stdlib mux, CORS `*` | pgxpool, no migrations | ❌ | **Gap:** no `cmd/server`, no schema layer |
| easy-host-k8s | 🔀 `backend-go/` | handler/store | **MySQL** | migrate in cmd | Server-rendered HTML |
| deep-digest-rss | ❌ Java | Spring | JPA/Flyway | — | **Out of scope** |
| start-renovate | ❌ Java | Spring Boot | — | — | **Out of scope** |
| coffee-diary | ✅ `cmd/server` | chi in `internal/handler` | **MariaDB** + `database/sql` | 🔀 `migrations/` on disk | OIDC session; not pgx; also **SwiftUI** `ios/` client |
| boardwalk-billionaire | ❌ Java | Spring Boot | — | — | **Out of scope** |
| cybernight | ❌ Java | Spring Boot | JPA/MariaDB | Flyway | **Out of scope** |
| picz | ❌ Gradle Spring | — | MariaDB | Flyway | **Out of scope** |
| picz2 | ❌ Java | Spring Boot | MariaDB+MinIO | Flyway | **Out of scope** |
| video-msg | 🔀 **`backend-go/`** | chi `internal/handler` | **MariaDB** `database/sql` | golang-migrate SQL | FFmpeg re-encoding; deprecated Java `backend/` still in CI/pre-commit |

**Disclosed gaps:**

- **`trivia` — would fail in the company (PgBouncer) environment.** `backend/internal/db/db.go:27` builds a `pgxpool` without `DefaultQueryExecMode = QueryExecModeExec` or the disabled statement/description caches, so pgx uses server-side named prepared statements. That works against the direct `postgres.default.svc` in k3s and errors behind transaction pooling. Its `pgxpool` use is now the *target* shape — only the three `ConnConfig` settings are missing. Highest-value fix in this table. Also: move to `internal/server`, embedded migrations.
- `irl-planner-pro` / `plugin-skill-hosting` use the `database/sql` adapter rather than `pgxpool`. Both set the pooler-safe options, so both are dual-environment correct — this is a style gap, **not** a defect, and does not need migrating on its own.
- `yt-infographics` needs a proper layout if it grows beyond a thin API; its `pgxpool` has no migrations layer.

**Valid deviations:** `plugin-skill-hosting` `cmd/marketplace` is historical naming. MySQL repos (`linky`, `easy-host-k8s`) correctly use their own drivers — do not force pgx patterns from this doc.
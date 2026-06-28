# Go backend — project assessment

Per-repo adoption for [../go-backend.md](../go-backend.md). Last verified: June 2026.

| Repo | Layout | Router/HTTP | DB | Migrations | Notes |
|------|--------|-------------|-----|------------|-------|
| irl-planner-pro | ✅ `cmd/server` + `internal/server` | chi + `server/` | pgx stdlib + PgBouncer mode | ✅ embedded | Target |
| plugin-skill-hosting | ⚠️ `cmd/marketplace` | ✅ `internal/server` | ✅ pgx | ✅ embedded | Rename cmd is cosmetic |
| trivia | ⚠️ `internal/api` not `server` | chi in api pkg | ⚠️ **pgxpool** direct | 🔀 **filesystem** `migrations/` | Legacy — migrate toward embedded + stdlib adapter |
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

**Disclosed gaps:** `trivia` should move toward `internal/server` + embedded migrations + pgx stdlib settings when touched; `yt-infographics` needs proper layout if it grows beyond a thin API.

**Valid deviations:** `plugin-skill-hosting` `cmd/marketplace` is historical naming. MySQL repos (`linky`, `easy-host-k8s`) correctly use their own drivers — do not force pgx patterns from this doc.
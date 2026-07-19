# Go Backend

How we build HTTP APIs in Go. It's one of our backend options — some projects ship Java Spring Boot services instead — but it's the default when pairing with a Vue SPA.

**Scope:** Go HTTP + Postgres services. Does **not** apply to Java Spring ([java-spring-backend.md](java-spring-backend.md)) or MySQL-backed Go apps (use sqlx/golang-migrate patterns instead of pgx). Per-repo adoption: [assessments/go-backend.md](assessments/go-backend.md).

## When to use Go vs Java

| Choose Go when… | Choose Java Spring Boot when… |
|-----------------|-------------------------------|
| Pairing with our standard Vue + Helm stack | The org already has Spring expertise/infrastructure |
| Small binary, distroless image, low memory | Heavy enterprise integrations (JPA ecosystem, batch frameworks) |
| Single static binary, embedded migrations | Existing Spring codebase or mandated JVM platform |

This doc covers Go application code only. Java Spring backends share the deploy plumbing ([oglimmer-sh.md](oglimmer-sh.md), [helm.md](helm.md), [github-actions.md](github-actions.md)) but not the layout — see [java-spring-backend.md](java-spring-backend.md). Path mapping: [assessments/repo-map.md](assessments/repo-map.md).

## Philosophy

- **Thin entrypoint, fat `internal/`.** `cmd/server/main.go` wires config → DB → migrations → jobs → HTTP; everything else lives in packages.
- **Transport separate from domain.** `internal/server/` is HTTP wiring only. Integrations (`email`, `slack`) and business rules get their own packages, testable without `httptest`.
- **Env-only configuration.** One `Config` struct, no config files in production. Sensible dev defaults; reject insecure defaults at startup unless explicitly allowed.
- **JSON API contract.** Errors are always `{"error": "…"}`. Browsers hitting unmatched routes get styled HTML via content negotiation.
- **Postgres via pgx.** Compatible with PgBouncer transaction pooling, but never dependent on it — the same binary runs against a direct Postgres. SQL migrations are embedded and run at startup.
- **stdlib logging.** Single-line `log.Printf` with level prefixes. No structured logging framework until the project outgrows this.

## Stack

| Layer | Choice |
|-------|--------|
| Language | Go 1.26+ |
| Router | `go-chi/chi/v5` |
| Database | `jackc/pgx/v5` native (`pgxpool`), configured pooler-safe |
| Auth | `golang-jwt/jwt/v5` (session JWTs) + `coreos/go-oidc/v3` (OIDC) |
| Metrics | `prometheus/client_golang` |
| CORS | `go-chi/cors` |
| Rate limiting | `go-chi/httprate` on auth/OAuth endpoints |
| Migrations | Embedded `.sql` files, applied at startup |
| Logging | stdlib `log` |

No Gin, no Echo, no GORM, no Viper, no zap (unless a project has a specific reason).

## Layout

```
backend/
  go.mod
  go.sum
  Dockerfile
  .dockerignore
  cmd/
    server/
      main.go           # ~100 lines: config → db → migrate → jobs → serve → shutdown
  internal/
    config/
      config.go         # Config struct + Load() + validation
    db/
      db.go             # Open(), Migrate(), Exec interface
      migrations/
        0001_init.sql
        0002_….sql
    server/
      app.go            # App struct, writeJSON/writeErr, ready probe
      router.go         # chi routes + middleware stack
      store.go          # data-access helpers (*Store)
      auth.go           # JWT issue/parse, middleware
      errors.go         # content-negotiated HTML/JSON errors
      *.go              # handlers grouped by domain (events, users, …)
      *_test.go         # colocated tests
    buildinfo/
      buildinfo.go      # Version, Commit, Time (ldflags)
    metrics/
      metrics.go        # Prometheus middleware + /metrics handler
    email/              # SMTP sender (stdlib net/smtp)
    slack/              # Slack Web API notifier
    <domain>/           # other integrations / pure domain logic
```

The module path matches the repo (`irlplanner`, `pluginskillhosting`, …). Keep `internal/` so nothing outside the module can import it.

## `cmd/server/main.go`

The entrypoint follows a fixed sequence:

1. `config.Load()`
2. `db.Open()` + `defer pool.Close()`
3. `db.Migrate()`
4. Construct `server.App` with config, DB, `Store`, and collaborators (`email.Sender`, `slack.Notifier`, …)
5. `signal.NotifyContext` for SIGINT/SIGTERM
6. Optional init (OIDC discovery, …) on `rootCtx`
7. Start background workers (`StartReminders`, `StartOAuthGC`, …) tracked by `sync.WaitGroup`
8. `app.MarkReady()` — flips `/readyz` to 200
9. `http.Server` with `ReadHeaderTimeout` + `server.NewRouter(app)`
10. Graceful shutdown: `srv.Shutdown` then wait for background workers (with timeout)

Distroless images have no zoneinfo, so embed IANA tzdata when the app uses `time.LoadLocation`:

```go
import _ "time/tzdata"
```

## Configuration (`internal/config`)

### Rules

- One `Config` struct; `Load()` is the only public entry.
- Read env with typed helpers: `getenv(key, fallback)`, `parseInt`, `parseDuration`, `parseBool`, and comma-separated lists.
- **Validate on load:** bad values log `WARN` and fall back — never crash on a malformed duration.
- **Reject insecure secrets at startup:** e.g. a JWT secret shorter than 32 chars or matching a published dev default, unless `ALLOW_INSECURE_JWT_SECRET=true` (local dev only).
- Put decision logic on the struct as methods (`MCPEnabled()`, `RequiresUserApproval()`) rather than scattering it across handlers.
- Document every env var in `.env.example` at the repo root.

### Dev vs production defaults

| Setting | Dev default | Production |
|---------|-------------|------------|
| `DATABASE_URL` | `postgres://…@localhost:5432/…` | From secret / Helm |
| `JWT_SECRET` | Placeholder (rejected unless `ALLOW_INSECURE_JWT_SECRET`) | `openssl rand -hex 32` |
| `AUTH_MODE` | `oidc` or `password` stub for zero-config local boot | `oidc` only |
| `PUBLIC_BASE_URL` | `http://localhost:8080` | HTTPS URL from Helm |

## HTTP layer (`internal/server`)

### `App` struct

```go
type App struct {
    Cfg   config.Config
    DB    *sql.DB
    Store *Store
    Email email.Sender    // zero value = not configured
    Slack slack.Notifier
    OIDC  *oidcRuntime     // nil unless auth.mode=oidc
    ready atomic.Bool
}
```

Handlers are methods on `*App`. JSON helpers:

```go
writeJSON(w, status, v)
writeErr(w, status, msg)           // → {"error": msg}
serverErr(w, r, err, publicMsg)    // log with request ID; return generic 500
```

### Middleware order

```go
r.Use(middleware.RequestID)
r.Use(middleware.RealIP)
r.Use(skipLogger("/healthz", "/readyz"))
r.Use(middleware.Recoverer)
r.Use(metrics.HTTPMiddleware)
r.Use(securityHeaders)
r.Use(cors.Handler(…))
```

- **Probes:** `/healthz` always `ok`; `/readyz` returns 503 until `MarkReady()`.
- **Metrics:** `GET /metrics` — gated by `METRICS_TOKEN` when set.
- **Security headers** on every response (`X-Content-Type-Options`, `X-Frame-Options`, CSP for error/OAuth HTML pages).
- **CORS:** `AllowedOrigins` from config; include MCP-specific headers when `/mcp` is enabled.

### Route structure

All API routes under `/api`. Nested chi groups by privilege:

```go
r.Route("/api", func(r chi.Router) {
    // Public
    r.Get("/auth/config", …)
    r.Get("/events/{slug}/image", …)   // public asset

    // Auth endpoints — rate-limited per IP
    r.Group(func(r chi.Router) {
        r.Use(httprate.LimitByIP(60, time.Minute))
        // OIDC login/callback/logout OR dev-login
    })

    // Authenticated
    r.Group(func(r chi.Router) {
        r.Use(app.authMiddleware)
        r.Get("/me", …)

        // Admin-only
        r.Group(func(r chi.Router) {
            r.Use(app.requireAdminMiddleware)
            r.Get("/admin/events", …)
        })
    })
})
```

Rules:

- **Admin routes** under `/api/admin/…` so they don't collide with slug-keyed attendee routes (`/api/events/{slug}`).
- **Rate-limit** login and OAuth token endpoints per IP.
- **Mount optional surfaces** (MCP, OAuth discovery) only when config enables them — see [mcp.md](mcp.md) for the full MCP server + OAuth 2.1 AS pattern.
- Set `r.NotFound` and `r.MethodNotAllowed` to content-negotiated handlers.

### Auth middleware

- Bearer token in `Authorization` header.
- JWT session tokens: `sub` (user ID), `ver` (revocation counter), `iat`, `exp` (30-day TTL).
- Resolve user from DB on each request; compare `ver` to `token_version` column — bump version to revoke all sessions.
- Distinguish "bad credential" (401 + message) from "DB error" (500) in token resolution.
- OIDC: `go-oidc` discovery, state+nonce cookies, ID token verification, RP-initiated logout. Optional Google Workspace `hd` allowlist (only for Google issuers).
- `password` auth mode is a **local dev stub only** — never enable on shared deployments.

### JSON ↔ Vue contract

The Vue frontend ([vue-frontend.md](vue-frontend.md)) expects:

- Error body: `{"error": "human-readable message"}`
- `204 No Content` for deletes with no body
- camelCase JSON field names on structs (`json:"firstName"`)
- `GET /api/auth/config` bootstrap payload for sign-in UI
- `GET /api/version` or `/api/...` for build info

## Database (`internal/db`)

Postgres-native features — JSONB, full-text search, upserts, `SKIP LOCKED` queues, advisory locks, bulk loads — are covered in [postgres-for-golang.md](postgres-for-golang.md). This doc owns the connection, the `Exec` interface, and the migration runner; that doc owns the SQL.

### Connection

**Use `pgxpool` with pgx's native interface** — not `lib/pq`, and not the `database/sql` adapter unless a third-party library forces `*sql.DB` on you.

Services run against a direct Postgres in the k3s cluster and behind a **transaction-mode PgBouncer** in the company environment. One binary must satisfy both, so these three settings are mandatory. They are inert on a direct connection:

```go
cfg, err := pgxpool.ParseConfig(url)
cfg.ConnConfig.DefaultQueryExecMode     = pgx.QueryExecModeExec
cfg.ConnConfig.StatementCacheCapacity   = 0
cfg.ConnConfig.DescriptionCacheCapacity = 0

cfg.MaxConns = 10
cfg.MinConns = 2
cfg.MaxConnLifetime = 30 * time.Minute
cfg.MaxConnIdleTime = 5 * time.Minute
```

Set them in code, not the DSN — a pooler-safety guarantee should not live in environment config where it can be dropped in a copy-paste. This also rules out session-scoped state (`LISTEN`, `pg_advisory_lock`, bare `SET`); see [§0 of postgres-for-golang.md](postgres-for-golang.md#0-the-two-environments-read-this-first) for the full list and the replacements.

Ping in a retry loop at startup (~30×, 1s backoff) so the service tolerates a slow-booting Postgres.

Legacy note: `irl-planner-pro` and `plugin-skill-hosting` use the `database/sql` adapter with these same three settings. They are pooler-safe and need no migration — new services use `pgxpool` directly.

### Migrations

- Numbered files: `0001_init.sql`, `0002_profile_names.sql`, …
- Embedded via `//go:embed migrations/….sql`
- Registered in a `migrations` slice — adding a file requires three steps: (1) the SQL file, (2) the embed var, (3) the slice entry
- Each script is idempotent (`CREATE TABLE IF NOT EXISTS`, `ALTER … IF NOT EXISTS`)
- `Migrate()` runs all migrations in order at every startup; errors wrap the migration name

### `Exec` interface

```go
type Exec interface {
    Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
    Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
    QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}
```

This is satisfied by both `*pgxpool.Pool` and `pgx.Tx`, so functions that accept `db.Exec` work inside and outside a transaction. Use `pgx.Tx` (via `pool.Begin` or `pgx.BeginTxFunc`) for multi-statement writes.

On the two repos still using the `database/sql` adapter this interface is the `sql.Result`/`*sql.Rows` equivalent (`ExecContext`/`QueryContext`/`QueryRowContext`). Same idea, same call sites — don't mix the two shapes within one repo.

### Schema conventions

- UUID primary keys (`gen_random_uuid()` via `pgcrypto`)
- `TIMESTAMPTZ` stored UTC; `DATE` for event-local calendar days
- Enum-like fields enforced with `CHECK` constraints
- **Soft-delete** user-facing records where appropriate; don't hard-delete audit trails
- Index foreign keys and lookup columns

### `Store`

`internal/server/store.go` holds data-access helpers on `*Store`. New read queries go on `Store`; transactional writes may still use `a.DB` directly while the data layer is extracted incrementally. The goal is a testable data layer without forcing a full repository pattern on day one.

## Domain packages (`internal/<domain>/`)

Integrations and pure logic live outside `server/`:

```go
// internal/email — stdlib SMTP, zero value = not configured
type Sender struct { Host, Port, Username, Password, From string; … }
func (s Sender) Configured() bool
func (s Sender) Send(to []string, subject, body string) error
```

Rules:

- The zero value means "not configured" — callers check `Configured()` before sending.
- No HTTP imports in domain packages.
- Colocate `*_test.go` with the package.
- Inject into `App` at construction time in `main.go`.

## Background jobs

Ticker + context cancellation, tracked by `sync.WaitGroup`:

```go
func (a *App) StartReminders(ctx context.Context, wg *sync.WaitGroup) {
    if !a.Cfg.RemindersEnabled { return }
    wg.Add(1)
    go func() {
        defer wg.Done()
        a.runReminderTick(ctx, time.Now())  // immediate pass at startup
        ticker := time.NewTicker(a.Cfg.ReminderTickInterval)
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done(): return
            case now := <-ticker.C: a.runReminderTick(ctx, now)
            }
        }
    }()
}
```

Rules:

- Log and continue on per-tick failures — one error never kills the loop.
- Keep the work **idempotent** (claim rows, `ON CONFLICT DO NOTHING`, period keys).
- Skip scheduled work when the integration is unconfigured (no SMTP → no reminder emails).
- Derive all worker contexts from the shutdown `rootCtx`.

## Build

Static binary for distroless:

```dockerfile
RUN CGO_ENABLED=0 go build -trimpath \
    -ldflags "-s -w \
      -X <module>/internal/buildinfo.Version=${VERSION} \
      -X <module>/internal/buildinfo.Commit=${GIT_COMMIT} \
      -X <module>/internal/buildinfo.Time=${BUILD_TIME}" \
    -o /out/server ./cmd/server
```

See [docker.md](docker.md) for the full multi-stage Dockerfile.

Expose build info at `GET /api/version` (or similar) for the Vue footer. Full release and semver flow: [versioning-release.md](versioning-release.md).

## Testing

### Unit tests (no DB)

Colocated `*_test.go`. Test pure functions directly:

```go
func testApp() *App {
    return &App{Cfg: config.Config{JWTSecret: "test-secret-at-least-32-characters-long"}}
}
```

JWT roundtrips, parsers, validators — no database required.

### HTTP handler tests

`httptest.NewRecorder()` + `httptest.NewRequest()`. Inject auth via context or `Authorization` header. Assert status, headers, and `json.Unmarshal` the body.

### DB-backed tests

Opt in via an env var, and self-skip when it's unset (so `go test ./...` works without Postgres):

```go
func testDB(t *testing.T) *sql.DB {
    url := os.Getenv("IRL_TEST_DATABASE_URL")
    if url == "" {
        t.Skip("set IRL_TEST_DATABASE_URL to run DB-backed tests")
    }
    d, _ := db.Open(url)
    db.Migrate(d)
    // TRUNCATE … RESTART IDENTITY CASCADE
    t.Cleanup(func() { d.Close() })
    return d
}
```

CI sets the env var with a `services.postgres` block ([github-actions.md](github-actions.md)). Locally, run `compose up` and export the URL.

Run with race detector in CI: `go test -race ./...`.

### What to test

| Layer | Focus |
|-------|-------|
| `internal/<domain>/` | Integration clients, parsing, edge cases |
| `server/` auth | Token issue/parse, middleware rejection paths |
| `server/` handlers | Status codes, JSON shape, auth gates |
| `server/` + DB | Business rules, idempotency, constraints |

## CI & local tooling

| Tool | What it runs |
|------|--------------|
| `gofmt -s` | Formatting (pre-commit + CI) |
| `go vet` | Static analysis |
| `go build ./...` | Compile all packages |
| `go test -race ./...` | Tests (+ Postgres service in CI) |
| `shellcheck` | If the repo has `oglimmer.sh` |

See [pre-commit.md](pre-commit.md) and [github-actions.md](github-actions.md).

## New-project checklist

1. `go mod init <module>`; pin Go 1.26+ in `go.mod`.
2. Create `cmd/server/main.go` with the standard startup sequence.
3. Add `internal/config` with `Load()` and `.env.example`.
4. Add `internal/db` with `Open`, `Migrate`, `Exec`, and `0001_init.sql`.
5. Add `internal/buildinfo` and `internal/metrics`.
6. Add `internal/server` with `App`, `router.go`, `auth.go`, `errors.go`.
7. Add `internal/email` or other integrations as needed.
8. Wire routes under `/api` with nested auth groups.
9. Add `backend/Dockerfile` per [docker.md](docker.md).
10. Add path-filtered backend job to `ci.yml`.
11. Add gofmt/vet/build/test to pre-commit.
12. Enable Renovate `gomod` manager ([renovate.md](renovate.md)).
13. Pair with Vue frontend per [vue-frontend.md](vue-frontend.md) and Helm chart per [helm.md](helm.md).

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Put business logic in `main.go` | `internal/<domain>/` or handler files with clear scope |
| Use `lib/pq` | `pgx/v5` native (`pgxpool`) with the pooler-safe settings |
| Return stack traces to clients | `serverErr` logs internally; return `{"error": "…"}` |
| Crash on bad env var format | Warn + fall back to default |
| Ship with default JWT secret in prod | Reject at startup unless `ALLOW_INSECURE_JWT_SECRET` |
| Global mutable state | Pass `*App` or dependencies explicitly |
| Hand-run migrations separately | Embedded migrations at startup |
| `LISTEN` or `pg_advisory_lock` (session-scoped) | Poll; `pg_try_advisory_xact_lock` — session state breaks behind PgBouncer |
| `password` auth in production | OIDC only on shared deployments |
| Enable `/metrics` on public ingress without token | Require `METRICS_TOKEN` |
| Hard-delete audit data | Soft-delete or append-only log tables |
| Skip `-race` in CI | `go test -race ./...` |
| Use an ORM by default | Raw SQL via pgx (`CollectRows` for scanning) |
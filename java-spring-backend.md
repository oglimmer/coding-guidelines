# Java Spring Boot Backend

How we build HTTP APIs in Java Spring Boot. Alternative to [go-backend.md](go-backend.md) — use Go for the standard Vue+Helm stack; use Spring when the app already lives here or needs the JVM ecosystem (JPA, Spring Security OAuth2, Spring AI, MCP).

**Not in scope:** Python scrapers or other non-JVM workers. Frontend pairing: [nuxt-frontend.md](nuxt-frontend.md) (Nuxt + session) or [vue-frontend.md](vue-frontend.md) (Vue SPA + session). Per-repo adoption: [assessments/java-spring-backend.md](assessments/java-spring-backend.md).

## Philosophy

- **Spring Boot conventions.** Starters, auto-config, `application.yml` — don't fight the framework.
- **Layered packages.** `web` (controllers) → `service` → `db` (entities + repositories). DTOs for API boundaries.
- **Schema via Flyway.** `ddl-auto: validate` — never `create` or `update` in production.
- **Session auth for SPAs.** Cookie sessions + CSRF for browser clients; API keys or OAuth2 bearer where needed — not JWT-in-localStorage (that's the Go+Vue pattern).
- **Context-path discipline.** Either mount the whole API under `/api` (variant A) or keep root paths and let Ingress split (variant B) — document which you chose.
- **Format in CI.** Spotless `check` before `verify` — same gate locally (pre-commit) and in GitHub Actions.

## Stack

| Layer | Variant A (OAuth SPA) | Variant B (API + Redis) |
|-------|----------------|-----------------|
| Boot | 4.0.5 | 4.1.0 |
| Java | 21 | 21 |
| Build | Maven (`./mvnw`) | Maven (`./mvnw`) |
| DB | PostgreSQL + Flyway | MariaDB + `flyway-mysql` |
| ORM | Spring Data JPA | Spring Data JPA |
| Auth | OAuth2 login (GitHub/GitLab) + session | Form login + Redis session + API keys |
| Extras | WebClient (DeepSeek, GitHub API) | Spring AI, MCP server, Redis, Bucket4j |
| Health | Actuator `/api/actuator/health` | Actuator + k8s probes |

## Layout

### Variant A — OAuth SPA backend (`backend/`, context-path `/api`)

```
backend/
  pom.xml
  mvnw / .mvn/
  Dockerfile
  src/main/java/com/oglimmer/start_renovate/
    StartRenovateApplication.java
    config/           # SecurityConfig, WebClient, properties
    controller/       # REST endpoints + GlobalExceptionHandler
    service/          # business logic
    repository/       # Spring Data JPA
    entity/           # JPA entities
    dto/              # request/response records
    security/         # OAuth2 user service, CSRF SPA helpers
  src/main/resources/
    application.yaml
    db/migration/     # Flyway SQL
  src/test/java/      # @SpringBootTest, MockMvc, Testcontainers optional
```

### Variant B — API + Redis session (`news-backend/`)

```
news-backend/
  pom.xml
  mvnw
  Dockerfile
  src/main/java/de/oglimmer/news/
    NewsApplication.java
    config/           # SecurityConfiguration, Cors, Redis session
    web/              # controllers + dto/
    service/
    db/               # entities + repositories (not separate entity/ pkg)
    mcp/              # MCP tool surface
  src/main/resources/
    application.yml
    news-properties.yml
    db/migration/     # Flyway V{version}__description.sql
  src/test/java/
```

Package naming differs (`controller` vs `web`, `repository` vs `db`) — pick one layout per repo and stay consistent.

## Configuration

### Externalize everything

`application.yaml` / `application.yml` with env var placeholders:

```yaml
spring:
  datasource:
    url: "${DB_URL:jdbc:postgresql://localhost:5432/renovate}"
    username: "${DB_USERNAME:renovate}"
    password: "${DB_PASSWORD:renovate}"
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false   # true only when you accept the trade-off (DDRSS today)
  flyway:
    enabled: true
```

Custom app keys under `app:` or a dedicated prefix (`deepseek:`, `github:`).

### Variant A specifics

- **`server.servlet.context-path: /api`** — all controllers are relative to `/api`. Ingress and Nuxt devProxy must include this prefix.
- **`server.forward-headers-strategy: framework`** — trust `X-Forwarded-*` behind Ingress.
- **OAuth2 client** registration for GitHub (and optional GitLab) under `spring.security.oauth2.client`.

### Variant B specifics

- **MariaDB** JDBC URL in `spring.datasource` (not Postgres).
- **Redis** for Spring Session (`spring.session.store-type: redis`).
- **Split config:** `spring.config.import: classpath:news-properties.yml` for tunables.
- **No global context-path** — API lives at `/api/v1/...` via controller `@RequestMapping`.

Mirror every secret and URL in Helm values comments and `.env.example` at repo root.

## Security (SPA + session)

Both production Spring apps serve a browser SPA. Pattern:

| Concern | Approach |
|---------|----------|
| Login | OAuth2 (variant A) or form login (variant B) |
| Session | Servlet session cookie (`credentials: 'include'` on fetch) |
| CSRF | `CookieCsrfTokenRepository` + SPA reads `XSRF-TOKEN`, sends `X-XSRF-TOKEN` on mutating requests |
| XHR vs redirect | `X-Requested-With: XMLHttpRequest` → **401** instead of 302 to login |
| CORS | Explicit allowed origin(s) — not `*` when cookies are used |
| Public endpoints | Explicit `permitAll()` matchers (health, `/feedback`, OAuth callbacks) |

Variant A `SecurityConfig` highlights:

- CSRF cookie path `/` (not `/api`) so `document.cookie` on the SPA host can read it.
- `SpaCsrfTokenRequestHandler` for Angular/Vue/Nuxt-style CSRF header.
- `POST /feedback` exempt from CSRF (cross-origin, rate-limited public API).

Variant B adds:

- `ApiKeyAuthFilter` for machine clients.
- Optional CSRF via `app.security.csrf-enabled`.
- Separate filter chains / `@Order` when MCP OAuth endpoints need different rules.

**Do not** copy Go backend JWT-in-`localStorage` auth into Spring SPAs without a deliberate redesign.

## HTTP API contract

Align with what the frontend expects:

- Errors: consistent JSON body (e.g. `GlobalExceptionHandler` → `{"error": "…"}` or problem-details — match the SPA's parser).
- **401** for unauthenticated XHR, not an HTML login redirect.
- Version/build info via Actuator `info` or a dedicated endpoint if the UI footer needs it.

Document public vs authenticated routes in controller JavaDoc — Ingress path lists depend on it.

## Database & Flyway

| Rule | Detail |
|------|--------|
| Migrations only | SQL files under `src/main/resources/db/migration/` |
| Naming | `V1__init.sql` (Flyway versioned) — DDRSS uses `V18.0.1__...` style |
| Idempotent where possible | `IF NOT EXISTS` for extensions/tables when bootstrapping |
| `ddl-auto` | `validate` in all deployed profiles |
| Postgres major bumps | Disable in [renovate.md](renovate.md) when chart bundles Postgres |

Variant A: PostgreSQL + `flyway-database-postgresql` + Boot 4 `spring-boot-flyway` starter (required for migrations to run).

Variant B: `flyway-mysql` + MariaDB driver; Redis is separate infrastructure (not Flyway).

## Docker

Multi-stage Maven build → JRE runtime:

```dockerfile
# build: eclipse-temurin JDK + ./mvnw package -DskipTests
# run:   eclipse-temurin JRE + java -jar app.jar
```

| Variant | Base image notes |
|---------|------------------|
| A | `jammy` JRE, non-root `spring` user, `curl` for HEALTHCHECK on `/api/actuator/health` |
| B | `alpine` JRE, `-XX:UseSVE=0` for arm64; prefer non-root user |

Bake no secrets. Configure via env vars from Helm / SealedSecret.

See [docker.md](docker.md) for registry tagging via [oglimmer-sh.md](oglimmer-sh.md).

## CI

Standard Java job shape ([github-actions.md](github-actions.md)):

```yaml
- uses: actions/setup-java@v5
  with:
    distribution: temurin
    java-version: '21'
    cache: maven
- run: ./mvnw -B spotless:check
- run: ./mvnw -B verify   # or test with RUN_INTEGRATION_TESTS=true + Testcontainers
```

Path filters must match the backend directory (`backend/**` or `news-backend/**`, etc.).

## Pre-commit

Java repos use **Spotless + verify**, not gofmt:

```yaml
- id: spotless-java
  entry: bash -c "cd backend && ./mvnw -B spotless:check"
- id: backend-verify
  entry: bash -c "cd backend && ./mvnw -B verify"
```

Java repos typically run Spotless locally and in CI; add gitleaks/trufflehog as needed.

## Deploy integration

| Concern | How |
|---------|-----|
| Image names | `<project>-be` or org convention via [oglimmer-sh.md](oglimmer-sh.md) |
| Registry | `registry.oglimmer.com` for both |
| Health probes | Actuator health; context-path affects probe URL when using variant A |
| Helm | [helm.md](helm.md) — secrets for DB, OAuth client secrets, API keys, Redis |

## New-repo checklist

1. `spring init` or copy from an existing org Spring project (OAuth SPA vs API+session).
2. Add `mvnw`, Flyway, Spotless, Actuator.
3. Set `ddl-auto: validate`, `open-in-view: false` unless you need otherwise.
4. Implement `SecurityFilterChain` with SPA 401/CSRF behaviour.
5. Add `Dockerfile` (JDK build → JRE run).
6. Wire `ci.yml` job: spotless → verify.
7. Add pre-commit Spotless + verify hooks.
8. Add `renovate.json` with Maven + Docker managers ([renovate.md](renovate.md)).
9. Point `oglimmer.sh` / `build.yml` at the correct directory and image name.

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| `ddl-auto: update` in prod | Flyway + `validate` |
| `open-in-view: true` by default | `false`; enable only with eyes open |
| JWT in localStorage for SPA | Session cookie + CSRF |
| 302 to login on XHR | 401 when `X-Requested-With: XMLHttpRequest` |
| Skip Spotless in CI | `spotless:check` before `verify` |
| Hard-code secrets in `application.yml` | `${ENV_VAR}` from Helm secret |
| Copy Go `internal/server` layout literally | Spring `controller`/`service`/`repository` |
| One Flyway dialect switch mid-repo | Postgres **or** MySQL per service — DDRSS is MariaDB |
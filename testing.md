# Testing

How we test full-stack apps: fast hermetic unit tests on every push, integration tests against a real DB behind a gate, and a thin layer of **Playwright** end-to-end journeys. One philosophy across Go, Java/Spring, and npm; different tools per stack.

Covers the three backends and the Vue/Nuxt frontend in [go-backend.md](go-backend.md), [java-spring-backend.md](java-spring-backend.md), [vue-frontend.md](vue-frontend.md), [nuxt-frontend.md](nuxt-frontend.md). CI wiring: [github-actions.md](github-actions.md). Per-repo adoption and best copy sources: [assessments/](assessments/) and [assessments/repo-map.md](assessments/repo-map.md).

## Philosophy

- **The default `test` run is hermetic.** `go test ./...`, `./mvnw test`, `npm run test` must pass with **no Docker, no database, no network**. Anything needing a real DB self-skips unless explicitly enabled. This is the PR gate.
- **Pyramid, not ice-cream-cone.** Many unit tests, fewer integration tests, a handful of e2e journeys. E2e is for "does the wired-up stack work," not for covering business rules — push those down to unit tests.
- **Test behaviour, not implementation.** Status codes, JSON shape, business rules, redirect-vs-401 — not private method internals or pixel positions ([vue-frontend.md](vue-frontend.md)).
- **Real dependencies beat mocks at the edges.** Integration tests run the app's **own migrations** against a real DB (Testcontainers or a CI service), never a hand-maintained parallel schema. Mock only what you don't own (outbound HTTP, third-party APIs).
- **One gate mechanism per stack.** Integration tests opt in via a single documented env var — `*_TEST_DATABASE_URL` (Go), `RUN_INTEGRATION_TESTS=true` (Java). No per-repo invention.
- **e2e selects by role and semantics, never `data-testid`.** `getByRole`, labels, and stable ids/CSS. If a test needs a testid hook, the UI probably lacks an accessible name — fix that instead.
- **Tests run in CI or they rot.** Several repos carry Playwright configs and `*_test.go`/`*Test.java` files that no workflow ever runs — those decay into commented-out dead code. A test not in `ci.yml` is a liability, not coverage.

## The layers

| Layer | Go | Java/Spring | Frontend (npm) | E2E |
|-------|----|-----|----|-----|
| **Unit** | stdlib `testing`, table-driven | JUnit 5 + Mockito + AssertJ | Vitest + jsdom | — |
| **Component / handler** | `net/http/httptest` + in-memory fake store | `@SpringBootTest` + `MockMvc` | `@testing-library/vue` + `@pinia/testing` + MSW | — |
| **Integration (real DB)** | env-gated `*_test.go` (CI Postgres service) or `//go:build integration` + Testcontainers | `RUN_INTEGRATION_TESTS` + Testcontainers MariaDB | — | — |
| **End-to-end** | — | — | — | Playwright (chromium) |

## Go

stdlib `testing` is the default — **no testify unless a repo already committed to it** (`video-msg`, `coffee-diary` do; greenfield uses plain `t.Errorf`/`t.Fatalf`). Tests are **colocated white-box** (`internal/<pkg>/*_test.go`, same package).

### Unit & handler tests

- **Table-driven** for pure functions (`TestValidateQuestion`, parsers, validators, PKCE/JWT).
- **Handlers via `httptest`** — `httptest.NewRecorder()` + `httptest.NewRequest()`, assert status, headers, and `json.Unmarshal` the body ([go-backend.md](go-backend.md)).
- **Hand-written in-memory fakes, never gomock/mockery.** A `fakeStore` implementing the `Store` interface, with a compile-time assertion:
  ```go
  type fakeStore struct{ /* maps + sync.Mutex */ }
  var _ Store = (*fakeStore)(nil)   // breaks the build if the interface drifts
  ```
  This keeps handler tests DB-free and fast. `trivia` (`internal/api/fake_store_test.go`) and `yt-infographics` (`fakeUserStore`) are the reference.

### Integration (real DB) — two sanctioned shapes

Both keep the default `go test ./...` hermetic; pick by whether CI provides a service DB or you want fully self-contained Docker.

**A. Env-gated + `t.Skip` (default — matches [go-backend.md](go-backend.md)).** `irl-planner-pro`, `plugin-skill-hosting`:
```go
func testDB(t *testing.T) *sql.DB {
    url := os.Getenv("IRL_TEST_DATABASE_URL")
    if url == "" {
        t.Skip("set IRL_TEST_DATABASE_URL to run DB-backed tests")
    }
    d, _ := db.Open(url); db.Migrate(d)
    // TRUNCATE … RESTART IDENTITY CASCADE between tests
    t.Cleanup(func() { d.Close() })
    return d
}
```
CI supplies the URL via a `services.postgres` block ([github-actions.md](github-actions.md)); the same `*_test.go` is a unit test locally and an integration test in CI.

**B. Build-tag + Testcontainers (fully self-contained).** `trivia` (`internal/api/integration_test.go`):
```go
//go:build integration
// Opt in: go test -tags=integration ./...   (requires a Docker daemon)
```
`tcpg.Run(ctx, "postgres:16-alpine", …)` with `wait.ForLog("ready to accept connections").WithOccurrence(2)`, then the app's own `db.Migrate`. Drives full flows over real WebSockets against `httptest.NewServer`. Gated on the **release** path (`oglimmer.sh` runs `go test -tags=integration` before building images), not the PR gate.

### Running

- `./oglimmer.sh test` → `go test ./...` (hermetic subset) — the canonical local entrypoint.
- CI: `go test -race ./...` always; add `-count=1` to defeat the cache for flaky-prone suites.
- `plugin-skill-hosting` is the coverage reference: `-coverprofile=coverage.out` → `go tool cover -func` → artifact + PR summary.

## Java / Spring Boot

JUnit 5 (Jupiter) + **AssertJ** (`assertThat`, `assertThatThrownBy`) universally; never JUnit 4. Maven + `mvnw` (only legacy `picz` is Gradle). Group with `@Nested` + `@DisplayName`.

### Unit tests — Mockito

Service layer is the primary unit-test target:
```java
@ExtendWith(MockitoExtension.class)
class MonitorServiceTest {
    @Mock MonitorRepository repo;
    @InjectMocks MonitorService service;
}
```
Pure domain/POJO logic (game rules, mappers) gets plain JUnit + AssertJ with no Spring context at all (`cybernight` `game/*Test.java`). **Don't bootstrap Spring for logic that doesn't need it.**

### Component tests — `@SpringBootTest` + MockMvc

The house pattern is a full context with **MockMvc built manually**, not `@WebMvcTest`/`@DataJpaTest` slices:
```java
@SpringBootTest
@ActiveProfiles("test")
@Transactional
class AlertContactControllerTest {
    MockMvc mvc = MockMvcBuilders.webAppContextSetup(context).build();
    // @WithMockUser / SecurityMockMvcRequestPostProcessors.oauth2Login()/csrf()
}
```
Reuse a `TestSecurityConfig` (`@Import(TestSecurityConfig.class)`) for auth stubbing. Use the current `@MockitoBean`, **not deprecated `@MockBean`**.

> On Spring Boot 4 the umbrella `spring-boot-starter-test` still works and stays the default. The split starters (`spring-boot-starter-webmvc-test`, `…-actuator-test`) are an option but not the standard — don't mix conventions within the org.

### Integration (real DB) — Testcontainers, env-gated

**Standardize on `RUN_INTEGRATION_TESTS=true` with Testcontainers MariaDB** (`deep-digest-rss` is the reference). Do **not** use the `-Drun.testcontainers` system-property variant (`picz2`) in new work — one gate, one name:
```java
@SpringBootTest
@Testcontainers
@EnabledIfEnvironmentVariable(named = "RUN_INTEGRATION_TESTS", matches = "true")
@TestPropertySource(properties = "spring.jpa.hibernate.ddl-auto=validate")
class SessionAuthFlowIntegrationTest {
    @Container @ServiceConnection
    static MariaDBContainer<?> db = new MariaDBContainer<>("mariadb:11");
    // extra container (Redis) via @DynamicPropertySource when needed
}
```
`ddl-auto=validate` means **Flyway migrations build the schema against real MariaDB** and Hibernate only validates — the same migrations that run in production. Default `./mvnw test` skips these; CI runs `RUN_INTEGRATION_TESTS=true ./mvnw verify` ([java-spring-backend.md](java-spring-backend.md), [github-actions.md](github-actions.md)).

**H2 in-memory** (`ddl-auto: create-drop`, `flyway.enabled: false` in `application-test.yml`) is the **fast fallback** when a repo can't run Docker in CI — but it drifts from MySQL-dialect Flyway scripts. Prefer Testcontainers; don't hand-maintain a parallel H2 `data.sql` (the `status-tacos` drift trap).

- Outbound HTTP is stubbed with okhttp `MockWebServer`, not rest-assured.
- Naming: `*Test` (surefire). If you split slow integration tests into failsafe, suffix `*IT` consistently — today the repos are inconsistent here, so **pick `*IT` = container-backed** and stick to it.
- Run: `./mvnw -B spotless:check` then `./mvnw -B verify`.

## Frontend (npm)

Vue 3 + Vite + **Vitest** + **jsdom** everywhere. The mature setup (`plugin-skill-hosting`, `irl-planner-pro`) is the standard; the bare `@vue/test-utils` scaffold left by `create-vue` (`status-tacos`, `cybernight`, `picz`) is **not** — those carry zero real tests.

Standard unit/component stack:
```jsonc
// devDependencies
"@testing-library/vue", "@testing-library/user-event", "@pinia/testing",
"msw", "jsdom", "vitest", "@vitest/coverage-v8"
```
```ts
// vitest.config.ts
test: { environment: 'jsdom', globals: true, setupFiles: ['./src/test/setup.ts'] }
```
- **Co-located `*.test.ts`** next to source — covers `lib/` utils, composables, Pinia stores, components, views, `api.ts`, `theme.ts` ([vue-frontend.md](vue-frontend.md)).
- **MSW for API mocking**, set up once in `src/test/setup.ts` with `server.listen({ onUnhandledRequest: 'error' })` + `cleanup()`. Shared `factories.ts` for fixtures.
- `@pinia/testing` for store-backed component tests.
- Scripts: `"test": "vitest run"`, `"test:coverage": "vitest run --coverage"`, `"test:watch": "vitest"`, and a `"check": "typecheck && lint && test"` aggregate.
- **Test the logic, not the pixels.** Pure `lib/`, composables, and `api.ts` helpers are the highest-value targets; component tests where behaviour (not layout) matters.

Nuxt repos run Vitest on pure lib logic (`start-renovate` regenerates a shared cross-language **fixture corpus** with `UPDATE_FIXTURES=1 vitest run`, consumed by both the Nuxt and the Java side — wire it into CI). Static Nuxt (`homepage-v4`) needs no tests beyond typecheck + build.

## End-to-end — Playwright

Playwright (chromium-first) for real user journeys. Two sanctioned topologies — pick by what you're testing:

**A. Isolated `e2e/` package against the full compose stack (preferred for CI).** `plugin-skill-hosting/e2e/` is the reference: its own `package.json` (`@playwright/test` only), `globalSetup` runs `docker compose up -d --build` from a clean DB and polls `/api/auth/config` until ready, `baseURL` is the **nginx single origin** (`:8080`) so the test hits frontend + backend + Postgres exactly as production wires them. `workers: 1` when journeys mutate shared state; reusable `helpers.ts` (`register`/`login`/`createPlugin`).

**B. In-frontend, `webServer` auto-start + `storageState` (lighter, for auth reuse).** `coffee-diary/frontend/` is the reference:
```ts
projects: [
  { name: 'setup', testMatch: /.*\.setup\.ts/ },
  { name: 'chromium', use: { ...devices['Desktop Chrome'], storageState: 'e2e/.auth/state.json' }, dependencies: ['setup'] },
],
webServer: { command: 'npm run dev', url: 'http://localhost:5173', reuseExistingServer: !process.env.CI },
```
`auth.setup.ts` does a **real Keycloak OIDC login** once (`id.oglimmer.de`, creds from `E2E_USER`/`E2E_PASS`) and persists `storageState`; every spec reuses it instead of logging in each time.

Conventions either way:
- `testDir: './e2e'` (or `./tests`), `forbidOnly: !!CI`, `retries: CI ? 1-2 : 0`, `workers: CI ? 1`, `reporter: 'html'`, `trace: 'on-first-retry'`, `screenshot: 'only-on-failure'`.
- **Selectors: `getByRole` + semantic labels / stable ids — no `data-testid`.**
- Journeys only: auth round-trip, wrong-password rejection, core CRUD, unauth→login redirect. Not field-level validation matrices (those are unit tests).
- `E2E_NO_STACK=1` / `reuseExistingServer` to point at an already-running stack during local iteration.

### Playwright in CI

A dedicated, path-filtered `e2e` job (`plugin-skill-hosting/.github/workflows/ci.yml`):
```yaml
e2e:
  needs: changes
  if: needs.changes.outputs.e2e == 'true'
  runs-on: ubuntu-latest
  defaults: { run: { working-directory: e2e } }
  steps:
    - uses: actions/checkout@v6
    - uses: actions/setup-node@v6
      with: { node-version: 24.16.0, cache: npm, cache-dependency-path: e2e/package-lock.json }
    - run: npm ci
    - run: npx playwright install --with-deps chromium
    - run: npm test
    - if: failure()
      uses: actions/upload-artifact@v4
      with: { name: playwright-report, path: e2e/playwright-report/ }
```
Cache `~/.cache/ms-playwright`; `playwright install --with-deps` only on cache miss; upload `playwright-report/` `if: always()`. Slow journeys against shared dev/stage/prod go in a `workflow_dispatch` `e2e.yml` instead of the PR critical path ([github-actions.md](github-actions.md)).

## Reference repos (copy sources)

| You need… | Copy from |
|-----------|-----------|
| Go env-gated DB tests | `irl-planner-pro/backend`, `plugin-skill-hosting/backend` |
| Go Testcontainers + build tag | `trivia/backend` (`internal/api/integration_test.go`) |
| Go in-memory fake store | `trivia` `fakeStore`, `yt-infographics` `fakeUserStore` |
| Go coverage in CI | `plugin-skill-hosting/backend` |
| Spring Mockito unit tests | `status-tacos/backend`, `picz2/server` |
| Spring Testcontainers integration | `deep-digest-rss/news-backend` (MariaDB + Redis) |
| Spring MockMvc + test security config | `status-tacos/backend` (`TestSecurityConfig`) |
| Vue unit/component (testing-library + MSW + coverage) | `plugin-skill-hosting/frontend` |
| Vue lib/composable unit tests | `irl-planner-pro/frontend` |
| Playwright against compose stack + CI job | `plugin-skill-hosting/e2e/` |
| Playwright `webServer` + OIDC `storageState` | `coffee-diary/frontend/` |
| Cross-language fixture corpus (Vitest ↔ Java) | `start-renovate` (`renovate:regen`) |

## New-repo checklist

1. **Hermetic by default** — `go test ./...` / `./mvnw test` / `npm run test` pass with no Docker or DB.
2. **Go:** colocated `*_test.go`, table-driven units, `httptest` handlers, an in-memory fake store with a `var _ Store` assertion.
3. **Go DB tests:** env-gate with `t.Skip` (CI Postgres service) **or** `//go:build integration` + Testcontainers — never an ungated DB dependency.
4. **Spring:** Mockito service units; `@SpringBootTest` + manual `MockMvc` for controllers; `@MockitoBean` not `@MockBean`.
5. **Spring DB tests:** `@EnabledIfEnvironmentVariable("RUN_INTEGRATION_TESTS")` + Testcontainers MariaDB + `@ServiceConnection`, Flyway builds schema (`ddl-auto=validate`).
6. **Frontend:** Vitest + jsdom + `@testing-library/vue` + `@pinia/testing` + MSW; co-located `*.test.ts`; `src/test/setup.ts`; `test` + `test:coverage` scripts.
7. **E2e:** one Playwright topology (isolated `e2e/` against compose, or in-frontend `webServer` + `storageState`); `getByRole` selectors; reusable login helper.
8. **Wire everything into `ci.yml`** — `-race` for Go, `spotless:check` + `verify` for Java, `test:coverage` for frontend, a path-filtered `e2e` job. A test not in CI doesn't count.
9. **Add `./oglimmer.sh test`** (and `e2e` where applicable) as the local entrypoint ([oglimmer-sh.md](oglimmer-sh.md)).
10. Run the integration suite on the **release** path at minimum if it's too slow for every PR (`trivia` gates release on `-tags=integration`).

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Require Docker/DB for the default test run | Self-skip via env gate; CI opts in |
| Hand-maintain an H2 `data.sql` parallel to Flyway | Testcontainers + real migrations (`ddl-auto=validate`) |
| `data-testid` hooks for e2e selectors | `getByRole` + accessible names / stable ids |
| Bootstrap Spring (`@SpringBootTest`) for pure logic | Plain JUnit + Mockito on the service/domain |
| gomock/mockery codegen in Go | Hand-written in-memory fake + `var _ Iface` assertion |
| Invent a per-repo integration gate | `*_TEST_DATABASE_URL` (Go) / `RUN_INTEGRATION_TESTS` (Java) |
| Leave Playwright/`*_test` files unrun in CI | Wire into `ci.yml` or delete them — dead tests rot |
| Cover business rules through slow e2e | Push rules down to unit tests; e2e = wiring only |
| Commit `create-vue` scaffold tests as "coverage" | Real `*.test.ts` with testing-library + MSW |
| Deprecated `@MockBean` | `@MockitoBean` |
| `npm test` hitting a live deployed URL by default | `webServer` auto-start or compose `globalSetup` |

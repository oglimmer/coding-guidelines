# Java Spring Boot Backend

How we build HTTP APIs in Java Spring Boot. An alternative approach is [go-backend.md](go-backend.md): Go is lighter on memory footprint and startup time.

**Not in scope:** Python scrapers or other non-JVM workers. Frontend pairing: [nuxt-frontend.md](nuxt-frontend.md) (Nuxt + session) or [vue-frontend.md](vue-frontend.md) (Vue SPA + session). Per-repo adoption and documented deviations: [assessments/java-spring-backend.md](assessments/java-spring-backend.md).

## Philosophy

- **Spring Boot conventions.** Starters, auto-config, `application.yaml` — don't fight the framework.
- **Layered packages.** `controller` → `service` → `repository`, with `entity` and `dto` as their own packages.
- **Schema via Flyway.** `ddl-auto: validate` — never `create` or `update` in production.
- **Session auth for SPAs.** OAuth2/OIDC login — any standards-compliant provider, our own Keycloak by default in production — plus cookie session + CSRF for browser clients. Not JWT-in-localStorage (that's the Go+Vue pattern).
- **One context-path.** Mount the whole API under `server.servlet.context-path: /api` — one setting instead of per-controller path discipline. Ingress and the frontend dev proxy both include this prefix.
- **Format in CI.** Spotless `check` before `verify` — same gate locally (pre-commit) and in GitHub Actions.

There is one way to build a Spring backend here, described below. Add MCP/Spring AI, Redis-backed sessions, API keys, or rate limiting only when a project's requirements call for them (see [Optional add-ons](#optional-add-ons)) — they extend this pattern, they don't replace it. A repo that needs MariaDB instead of Postgres, or form login instead of OAuth2, is a documented deviation, not a second variant — track it in [assessments/java-spring-backend.md](assessments/java-spring-backend.md).

## Stack

| Layer | Choice |
|-------|--------|
| Boot | 4.1.x (latest 4.x) |
| Java | 21 |
| Build | Maven (`./mvnw`) |
| DB | PostgreSQL + Flyway |
| ORM | Spring Data JPA |
| Auth | OAuth2/OIDC login (Keycloak `id.oglimmer.de` in production; GitHub/GitLab/Google/any OIDC provider otherwise) + session |
| Extras | WebClient for outbound HTTP calls (third-party APIs) |
| Utilities | Lombok — `@RequiredArgsConstructor`, `@Slf4j`, `@Getter`/`@Setter`, `@SneakyThrows`, `@Builder` (see [Lombok](#lombok)) |
| Reporting | JaCoCo, Checkstyle, PMD, SpotBugs/find-sec-bugs, OWASP dependency-check via `mvn site` (see [Static analysis & mvn site](#static-analysis--mvn-site)) |
| Health | Actuator `/api/actuator/health` |

No MariaDB, no form-login-only auth, no ad-hoc `RestTemplate` — unless a project has a specific reason (existing infra, mandated platform). Document the reason in the assessment.

## Layout

```
backend/
  pom.xml
  mvnw / .mvn/
  Dockerfile
  src/main/java/com/oglimmer/<project>/
    <Project>Application.java
    config/           # SecurityConfig, WebClient, properties
    controller/       # REST endpoints + GlobalExceptionHandler
    service/          # business logic
    repository/       # Spring Data JPA
    entity/           # JPA entities
    dto/              # request/response records
    mapper/           # optional: entity<->dto mapping reused across services
    exception/        # ApiException + named subclasses (NotFoundException, …)
    security/         # OAuth2 user service, CSRF SPA helpers
  src/main/resources/
    application.yaml
    db/migration/     # Flyway SQL, V{version}__description.sql
  src/test/java/      # @SpringBootTest, MockMvc, Testcontainers optional
```

The package name matches the repo/org (`com.oglimmer.<project>`). Keep `controller`/`service`/`repository`/`entity`/`dto` as separate packages rather than flattening controllers+DTOs into one `web` package or entities+repositories into one `db` package — it keeps the layered-package philosophy visible in the tree.

## DTO mapping

**`@Entity` never crosses the controller boundary.** No controller method takes an entity as a `@RequestBody`, and none returns one — every request/response is a DTO. This isn't optional or style preference:

- **Mass assignment:** binding a request body straight to an entity lets a client set any field the class exposes — `id`, `createdAt`, ownership/role columns — not just the ones the endpoint intends to accept.
- **Serialization failures:** Jackson serializing an entity walks its associations; a lazy-loaded one throws `LazyInitializationException` outside a session, and a bidirectional relationship recurses until it blows the stack (or Jackson's cycle detection kicks in and produces garbage output).
- **Schema coupling:** the API contract becomes whatever the JPA schema happens to look like today. A column rename or a new `@ManyToOne` in a migration silently changes the public API instead of being an internal refactor.

A DTO at the boundary is what makes the entity/DTO split in [Layout](#layout) meaningful — if a controller can return an entity, the `dto/` package is decorative. See below for how the conversion itself is written.

**No mapping framework** — no MapStruct, ModelMapper, Dozer, or Orika. Write the entity↔DTO conversion by hand.

The reason is specific to how this code gets written and maintained: MapStruct generates the actual mapping implementation at compile time into `target/generated-sources`, which never shows up in a diff or a code review — including a review by Claude, which can't see generated code without running a full build first. ModelMapper is worse: it maps by reflection/convention, so a renamed or added field fails silently at runtime with no compiler signal. Manual mapping is more boilerplate, but boilerplate is cheap for whoever (human or Claude) is writing this code, and every line of it is visible, reviewable, and fails at compile time like any other Java code.

- **Simple 1:1 conversion:** a static factory method on the DTO record itself.
  ```java
  public record UserDto(UUID id, String email, String displayName) {
      public static UserDto from(User entity) {
          return new UserDto(entity.getId(), entity.getEmail(), entity.getDisplayName());
      }
  }
  ```
- **Reused across services, or assembled from more than one entity:** a dedicated `<Entity>Mapper` class with static methods, in `mapper/`.
  ```java
  public final class OrderMapper {
      private OrderMapper() {}
      public static OrderDto toDto(Order order, Customer customer) { … }
  }
  ```
- Keep mappers **static and side-effect-free**. If converting a DTO needs an injected collaborator (e.g. a lookup call), that's a `service` method, not a mapper — don't turn `mapper/` into a Spring-managed layer with its own DI graph.
- Don't build a single catch-all `Mapper`/`Converter` utility class for the whole app — one mapper per entity (or skip `mapper/` entirely and keep the factory method on the DTO) keeps each conversion next to the type it belongs to.

## Lombok

Use Lombok for boilerplate that's mechanical and fully predictable regardless of the class it's applied to — not for anything that encodes business logic (that's still hand-written, see [DTO mapping](#dto-mapping)).

| Where | Use | Skip |
|-------|-----|------|
| `service` / `config` / other Spring components | `@RequiredArgsConstructor` on `final` fields for constructor injection; `@Slf4j` for the logger field | Field injection (`@Autowired` on fields), manual constructors written just for DI |
| `entity` | `@Getter @Setter @NoArgsConstructor @AllArgsConstructor` | `@Data`, `@EqualsAndHashCode`, `@ToString` — see below |
| `dto` | Nothing — DTOs are `record`s, which already give immutability plus `equals`/`hashCode`/`toString` for free | `@Value` (redundant with records) |
| A method calling a checked-exception-throwing library API | `@SneakyThrows` (see [Exception handling](#exception-handling)) | `try/catch` wrappers |
| Classes with several optional fields, or test fixtures | `@Builder` | — |

**Never `@Data` on a JPA entity**, and skip `@EqualsAndHashCode`/`@ToString` there by default. `@ToString` walking a lazy association triggers an extra query (or a `LazyInitializationException` outside a session). `@EqualsAndHashCode` over all fields breaks once Hibernate assigns the ID after persist — an entity hashed into a `Set` before persist hashes differently afterward — and can recurse across bidirectional relationships. If an entity genuinely needs `equals`/`hashCode`, key it explicitly to the ID:

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Order {
    @Id
    @EqualsAndHashCode.Include
    private UUID id;

    private String status;
    // …
}
```

`@Data` is banned everywhere, not just on entities — it silently turns on `equals`/`hashCode`/`toString` together in one annotation, which is exactly the kind of implicit behavior this doc otherwise avoids. Compose the specific annotations you actually want instead.

## Configuration

### Externalize everything

`application.yaml` with env var placeholders:

```yaml
server:
  servlet:
    context-path: /api
  forward-headers-strategy: framework   # trust X-Forwarded-* behind Ingress
spring:
  datasource:
    url: "${DB_URL:jdbc:postgresql://localhost:5432/app}"
    username: "${DB_USERNAME:app}"
    password: "${DB_PASSWORD:app}"
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false   # true only when you accept the trade-off, and say so in the assessment
  flyway:
    enabled: true
  security:
    oauth2:
      client:
        registration:
          keycloak:                                  # production default — any OIDC provider works the same way
            client-id: "${OIDC_CLIENT_ID}"
            client-secret: "${OIDC_CLIENT_SECRET}"
            scope: openid,profile,email
            provider: keycloak
          github:                                    # dev-tool-style apps where GitHub is the natural identity
            client-id: "${GITHUB_CLIENT_ID}"
            client-secret: "${GITHUB_CLIENT_SECRET}"
        provider:
          keycloak:
            issuer-uri: "${OIDC_ISSUER_URI:https://id.oglimmer.de/realms/<realm>}"
```

Spring Security's OAuth2 client treats every provider the same way: a `registration` (client id/secret/scope) plus a `provider` (an `issuer-uri`, or a Spring-recognized shorthand like `github`/`gitlab`/`google` that already knows its own endpoints). **This isn't GitHub/GitLab-specific** — it's whatever OIDC provider fits the app. Our own Keycloak (`id.oglimmer.de`) is the production default; GitHub/GitLab/Google are reasonable choices for internal tools whose users already have those accounts. Only register the providers an app actually uses.

Custom app keys under `app:` or a dedicated integration prefix (e.g. `deepseek:`, `github:`). Mirror every secret and URL in Helm values comments and in `.env.example` at the repo root.

## Security (SPA + session)

Every Spring app here serves a browser SPA. The pattern:

| Concern | Approach |
|---------|----------|
| Login | OAuth2/OIDC — any provider; Keycloak (`id.oglimmer.de`) is the production default |
| Session | Servlet session cookie (`credentials: 'include'` on fetch) |
| CSRF | `CookieCsrfTokenRepository` + SPA reads `XSRF-TOKEN`, sends `X-XSRF-TOKEN` on mutating requests |
| XHR vs redirect | `X-Requested-With: XMLHttpRequest` → **401** instead of 302 to login |
| CORS | Explicit allowed origin(s) — not `*` when cookies are used |
| Public endpoints | Explicit `permitAll()` matchers (health, feedback forms, OAuth callbacks) |

`SecurityConfig` highlights:

- CSRF cookie path `/` (not `/api`) so `document.cookie` on the SPA host can read it.
- `SpaCsrfTokenRequestHandler` for Vue/Nuxt-style CSRF header.
- Rate-limited, CSRF-exempt public endpoints (e.g. `POST /feedback`) declared explicitly, not by disabling CSRF broadly.

**Do not** copy the Go backend's JWT-in-`localStorage` auth into Spring SPAs without a deliberate redesign.

## HTTP API contract

Match what the frontend expects:

- Errors: `{"error": "…"}` from `GlobalExceptionHandler` — see [Exception handling](#exception-handling). `vue-frontend.md` parses exactly this shape; don't switch to RFC 7807 problem-details.
- **401** for unauthenticated XHR, not an HTML login redirect.
- Version/build info via Actuator `info`, or a dedicated endpoint if the UI footer needs it.

Document public vs authenticated routes in controller JavaDoc — Ingress path lists depend on it.

## Exception handling

**Unchecked only.** No `throws` clauses on service/business methods. Checked exceptions don't compose with streams/lambdas, and Spring's own `DataAccessException` hierarchy is already unchecked — declaring checked exceptions elsewhere fights that convention for no benefit.

A small hierarchy: a base `ApiException` carrying an `HttpStatus`, with one named subclass per case — each hardcodes its own status so call sites never pass `HttpStatus.X` literals around.

```java
public sealed class ApiException extends RuntimeException
        permits NotFoundException, ConflictException, ValidationException {
    private final HttpStatus status;
    protected ApiException(HttpStatus status, String message) {
        super(message);
        this.status = status;
    }
    public HttpStatus getStatus() { return status; }
}

public final class NotFoundException extends ApiException {
    public NotFoundException(String message) { super(HttpStatus.NOT_FOUND, message); }
}
```

One `@RestControllerAdvice` maps the hierarchy to the JSON contract, plus a catch-all for anything unexpected:

```java
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(ApiException.class)
    ResponseEntity<Map<String, String>> handleApi(ApiException ex) {
        return ResponseEntity.status(ex.getStatus()).body(Map.of("error", ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    ResponseEntity<Map<String, String>> handleUnexpected(Exception ex) {
        log.error("Unhandled exception", ex);
        return ResponseEntity.internalServerError().body(Map.of("error", "Internal server error"));
    }
}
```

**Checked exceptions from libraries** (`IOException`, `GeneralSecurityException`, JSON processing, …) — use Lombok's `@SneakyThrows` on the method that calls them, not a `try/catch` wrapper. It keeps the call site readable and doesn't force a `throws` clause onto a caller that can't do anything about it anyway. The exception still escapes as its original checked type; the catch-all `Exception.class` handler above turns it into a 500, logs the full exception server-side, and returns a generic message to the client.

```java
@SneakyThrows
private byte[] digest(String input) {
    return MessageDigest.getInstance("SHA-256").digest(input.getBytes(StandardCharsets.UTF_8));
}
```

- Put `@SneakyThrows` on the smallest method around the library call, not broadly across a class.
- If a checked exception needs a specific business status or message rather than a generic 500, don't `@SneakyThrows` it — catch it and throw the matching `ApiException` subclass instead.
- See [Lombok](#lombok) for the rest of the annotations in use — `@SneakyThrows` is one of several, not a special case.

## Transactions & persistence context

**`@Transactional` lives on `service` methods only** — never on controllers, never on repository methods. Spring Data already wraps every repository call in its own transaction; adding `@Transactional` again on top just changes propagation/isolation without buying anything. A controller opens no transaction of its own — it calls a service method that does.

**One `@Transactional` service method = one persistence context = one unit of work.** The moment an operation touches more than one repository call, or reads then writes based on what it read, it needs an explicit `@Transactional` service method — not because Spring demands it, but because without it each repository call gets its own transaction and its own consistency snapshot, and a failure partway through leaves partial writes committed.

**Map to DTO before the method returns.** With `open-in-view: false` ([Configuration](#configuration)), the persistence context closes when the transactional method returns — touching a lazy association afterward (e.g. in the controller) throws `LazyInitializationException`. Combined with [entities never crossing the controller boundary](#dto-mapping): the `@Transactional` service method is where entity and DTO meet. Load, touch whatever lazy data the DTO needs, map, return the DTO — a service method's return type is a DTO, not an entity, for this reason as much as the boundary rule.

### Self-invocation breaks it silently

`@Transactional` works through a proxy. Calling a `@Transactional` method from another method **in the same class** bypasses the proxy — no transaction opens, nothing rolls back on failure, and there's no error or warning to notice it:

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    public void placeOrder(...) {
        this.persistOrder(...);   // NOT transactional — same-class call skips the proxy
    }

    @Transactional
    void persistOrder(...) { ... }
}
```

Fix it by moving `persistOrder` to a separate collaborator bean and injecting it — don't self-inject a proxy of the same class just to route around this.

### Rollback and `@SneakyThrows`

Spring's default rollback rule: a transaction rolls back on an unchecked exception (`RuntimeException`/`Error`) and **commits** on a checked one. The [`ApiException`](#exception-handling) hierarchy is unchecked, so throwing `NotFoundException`/`ConflictException`/etc. from inside a `@Transactional` method rolls back correctly with no extra configuration.

**`@SneakyThrows` breaks this if the same method is `@Transactional`.** `@SneakyThrows` doesn't change what the exception *is* — a checked exception rethrown via `@SneakyThrows` is still checked by class, so Spring's default rollback rule does **not** roll back for it, even though the compiler no longer sees a `throws` clause. A `@Transactional` method that can `@SneakyThrows` a checked exception has to say so explicitly:

```java
@Transactional(rollbackFor = Exception.class)
@SneakyThrows
public void importRecord(...) { ... }
```

Better still: keep the checked-exception-throwing call outside the `@Transactional` method entirely, so the two concerns never collide.

### Read-only queries

`@Transactional(readOnly = true)` on service methods that only read — it skips Hibernate's dirty-checking flush and documents intent at the call site. Leave mutating methods at the default (`readOnly = false`).

### Don't hold a transaction open across an external call

Never call `WebClient` (or any outbound network call) from inside a `@Transactional` method — it holds a DB connection, and any row locks already acquired, for as long as the external call takes, which starves the connection pool under load. Structure the method so the external call happens outside the transactional boundary: do the external call first and write the result in a short transaction afterward, or the reverse — never both inside one transaction.

## Database & Flyway

| Rule | Detail |
|------|--------|
| Migrations only | SQL files under `src/main/resources/db/migration/` |
| Naming | `V1__init.sql` (Flyway versioned) |
| Idempotent where possible | `IF NOT EXISTS` for extensions/tables when bootstrapping |
| `ddl-auto` | `validate` in all deployed profiles |
| Driver | `flyway-database-postgresql` + the Boot 4 `spring-boot-flyway` starter (required for migrations to run) |
| Postgres major bumps | Disable in [renovate.md](renovate.md) when the chart bundles Postgres |

## Optional add-ons

Add these when a project's requirements call for them — they layer onto the pattern above, they don't fork it into a variant:

| Need | Add |
|------|-----|
| Machine clients (no browser session) | `ApiKeyAuthFilter` in a separate filter chain / `@Order`, alongside the OAuth2 chain |
| Horizontally-scaled sessions | `spring.session.store-type: redis` instead of the default in-memory servlet session |
| Rate limiting | Bucket4j, applied to auth/OAuth endpoints and any public unauthenticated routes |
| AI tool endpoints | Spring AI + an MCP server under its own package (`mcp/`); give MCP OAuth endpoints their own filter chain if their rules differ from the SPA's — full pattern in [mcp.md](mcp.md) |
| Existing MariaDB infra | `flyway-mysql` + MariaDB driver instead of the Postgres stack — document why in the assessment |

## Docker

Multi-stage Maven build → JRE runtime:

```dockerfile
# build: eclipse-temurin JDK + ./mvnw package -DskipTests
# run:   eclipse-temurin JRE + java -jar app.jar
```

`jammy` JRE, non-root `spring` user, `curl` for `HEALTHCHECK` on `/api/actuator/health`. Alpine JRE is fine for a smaller image if you need it — set `-XX:UseSVE=0` on arm64.

Bake in no secrets. Configure via env vars from Helm / SealedSecret.

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

Path filters must match the backend directory (`backend/**`, or whatever this repo calls it).

## Static analysis & mvn site

**Non-blocking, informational.** Spotless (above) is the only enforced style gate. Checkstyle, PMD, SpotBugs, and dependency-check run as Maven `<reporting>` plugins, assembled by `mvn site` into `target/site/`, and published as a browsable report — they never fail `verify` or block a PR.

| Tool | Binding | Reports |
|------|---------|---------|
| JaCoCo | `<build>`: `prepare-agent` + `report` on the `test` phase | Line/branch coverage |
| Checkstyle | `<reporting>` only | Style + complexity |
| PMD | `<reporting>` only | Default ruleset |
| SpotBugs + find-sec-bugs | `<reporting>` only | Security-focused static analysis |
| OWASP dependency-check | `<build>`: `aggregate` execution, plus `<reporting>` | CVE scan of dependencies |
| git-commit-id-maven-plugin | `<build>` | Git SHA/branch/time into Actuator `/info` |
| maven-project-info-reports-plugin | `<reporting>` only | Index + dependency list pages |

### Checkstyle: keep complexity checks, drop Javadoc and style nitpicks

Base config is the Checkstyle plugin's bundled `sun_checks.xml`, with complexity modules added to its `<TreeWalker>`: `BooleanExpressionComplexity`, `ClassDataAbstractionCoupling`, `ClassFanOutComplexity`, `CyclomaticComplexity`, `JavaNCSS`, `NPathComplexity`. Then suppress everything that doesn't fit how this org writes Java — apps, not libraries, so no Javadoc requirement:

```xml
<!-- checkstyle-suppressions.xml -->
<suppressions>
    <suppress checks="JavadocPackage" files=".*" />
    <suppress checks="JavadocStyle" files=".*" />
    <suppress checks="JavadocVariable" files=".*" />
    <suppress checks="MissingJavadocMethod" files=".*" />
    <suppress checks="JavadocMethod" files=".*" />
    <suppress checks="DesignForExtension" files=".*" />
    <suppress checks="HiddenField" files=".*" />                  <!-- constructor params shadowing fields is fine -->
    <suppress checks="MagicNumber" files=".*" />
    <suppress checks="HideUtilityClassConstructor" files=".*" />  <!-- not compatible with Spring beans -->
    <suppress checks="AvoidStarImport" files=".*" />
    <suppress checks="FinalParameters" files=".*" />
    <suppress checks="LineLength" files=".*" />
    <suppress checks="NewlineAtEndOfFile" files=".*" />
</suppressions>
```

### SpotBugs: exclude the Lombok false positives

```xml
<!-- spotbugs-security-exclude.xml -->
<FindBugsFilter>
    <Match><Bug pattern="EI_EXPOSE_REP"/></Match>   <!-- fires on every Lombok @Getter returning a mutable field -->
    <Match><Bug pattern="EI_EXPOSE_REP2"/></Match>  <!-- fires on every Lombok @Setter accepting a mutable reference -->
</FindBugsFilter>
```

### Dependency-check: never hardcode the NVD key

```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <configuration>
        <nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>
        <outputDirectory>${project.build.directory}/site/</outputDirectory>
    </configuration>
</plugin>
```

`NVD_API_KEY` comes from a GitHub Actions secret. A literal key value in `pom.xml` is a checked-in credential — never do that, even for a rate-limit-only key.

### Publishing the site

A second, minimal image serves `target/site/` — built and pushed from a **post-merge** job, not PR CI, so pull requests stay fast:

```dockerfile
# Dockerfile-site
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY target/site/ /usr/share/nginx/html/
EXPOSE 80
```

```yaml
# post-merge job only — PR CI stays on spotless:check + verify
- run: ./mvnw -B verify site
- uses: docker/build-push-action@v7
  with:
    context: ./backend
    file: ./backend/Dockerfile-site
    push: true
    tags: <registry>/<project>-backend-site:latest
```

## Pre-commit

Java repos use **Spotless + verify**, not gofmt:

```yaml
- id: spotless-java
  entry: bash -c "cd backend && ./mvnw -B spotless:check"
- id: backend-verify
  entry: bash -c "cd backend && ./mvnw -B verify"
```

Java repos typically run Spotless both locally and in CI; add gitleaks/trufflehog as needed.

## Deploy integration

| Concern | How |
|---------|-----|
| Image names | `<project>-be` or org convention via [oglimmer-sh.md](oglimmer-sh.md) |
| Registry | `registry.oglimmer.com` |
| Health probes | Actuator health at `/api/actuator/health` (context-path affects the probe URL) |
| Helm | [helm.md](helm.md) — secrets for DB, OAuth client secrets, and any optional add-ons in use |

## New-repo checklist

1. `spring init` or copy from an existing org Spring project on this pattern.
2. Add `mvnw`, Flyway (`flyway-database-postgresql`), Spotless, Actuator.
3. Set `server.servlet.context-path: /api`, `ddl-auto: validate`, `open-in-view: false` unless you need otherwise.
4. Implement `SecurityFilterChain` with OAuth2 login + SPA 401/CSRF behaviour.
5. Add `Dockerfile` (JDK build → JRE run).
6. Wire `ci.yml` job: spotless → verify.
7. Add pre-commit Spotless + verify hooks.
8. Add `renovate.json` with Maven + Docker managers ([renovate.md](renovate.md)).
9. Point `oglimmer.sh` / `build.yml` at the correct directory and image name.
10. Add JaCoCo, Checkstyle, PMD, SpotBugs/find-sec-bugs, and OWASP dependency-check reporting plugins; copy `checkstyle.xml`/`checkstyle-suppressions.xml`/`spotbugs-security-exclude.xml`; wire a post-merge `mvn verify site` job publishing `<project>-backend-site`.
11. Add any optional add-ons the project actually needs (see above) — and note them in the assessment.

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| `ddl-auto: update` in prod | Flyway + `validate` |
| `open-in-view: true` by default | `false`; enable only with eyes open |
| JWT in localStorage for SPA | Session cookie + CSRF |
| 302 to login on XHR | 401 when `X-Requested-With: XMLHttpRequest` |
| Skip Spotless in CI | `spotless:check` before `verify` |
| Hard-code secrets in `application.yaml` | `${ENV_VAR}` from Helm secret |
| Copy Go `internal/server` layout literally | Spring `controller`/`service`/`repository` |
| Flatten `controller`+`dto` into `web`, or `entity`+`repository` into `db` | Keep the five packages separate |
| Introduce a second DB dialect or auth style "because this repo is different" without recording it | Document the deviation in [assessments/java-spring-backend.md](assessments/java-spring-backend.md) |
| MapStruct, ModelMapper, Dozer, or Orika for entity↔DTO mapping | Static factory method on the DTO, or a hand-written `<Entity>Mapper` |
| `@Entity` as a controller's `@RequestBody` or return type | Convert to/from a DTO at the boundary — always |
| `@Transactional` on a controller or repository method | Service layer only |
| Calling a `@Transactional` method from another method in the same class | Move it to a separate collaborator bean |
| `@SneakyThrows` inside `@Transactional` without `rollbackFor` | `rollbackFor = Exception.class`, or move the checked call outside the transaction |
| `WebClient`/external call inside a `@Transactional` method | Keep external I/O outside the transaction boundary |
| `throws` clauses on service/business methods | Unchecked `ApiException` subclasses |
| `try/catch` wrapper around a checked-exception library call | `@SneakyThrows` on that method |
| Leaking raw exception messages or stack traces to the client | Catch-all `Exception.class` handler: log server-side, fixed generic message to the client |
| `@Data`, or `@EqualsAndHashCode`/`@ToString` on a JPA entity | Explicit `@Getter @Setter @NoArgsConstructor @AllArgsConstructor`; key `equals`/`hashCode` to the ID only if truly needed |
| `@Autowired` field injection | `@RequiredArgsConstructor` on `final` fields |
| Hardcode the NVD API key (or any credential) in `pom.xml` | `${env.NVD_API_KEY}` from a CI secret |
| Bind Checkstyle/PMD/SpotBugs `:check` goals to the PR gate | Reporting-only via `mvn site`; Spotless stays the one enforced gate |

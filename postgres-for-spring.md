# Postgres for Spring Boot

Reference for using Postgres-native features from Spring Boot. Assumes Spring Boot 3.2+/4.x, Hibernate 6.4+, PostgreSQL 15+, and Flyway.

**Scope:** the database layer of a Spring Boot service — JSONB, full-text search, upserts, `RETURNING`, `SKIP LOCKED` job queues, advisory locks, arrays, exclusion constraints, isolation/retry, bulk writes, and Flyway migration pitfalls. Layers onto [java-spring-backend.md](java-spring-backend.md), which owns the service shape (layout, config, auth, build). Go equivalent: [postgres-for-golang.md](postgres-for-golang.md). Applies to `start-renovate` and `deep-digest-rss`; our other Spring services (cybernight, picz, picz2, status-tacos) run MariaDB, so check the DB in [assessments/repo-map.md](assessments/repo-map.md) before applying this.

**Every service must run against both a direct Postgres and a transaction-mode PgBouncer — start at [§0](#0-the-two-environments-read-this-first).**

**Prime directive:** JPA is a portability abstraction. Postgres's best features are non-portable. When they conflict, drop to native SQL or `JdbcClient` rather than contorting JPA. Mixing is fine and normal.

## 0. The two environments (read this first)

Our services must run in both, from one build:

| | k3s cluster (`kubectl`) | Company environment |
|---|---|---|
| Postgres access | direct to `postgres.default.svc` | **PgBouncer, transaction mode** |

**Write PgBouncer-compatible code; never require PgBouncer.** pgjdbc switches to server-side named prepared statements after `prepareThreshold` executions (default 5), which transaction pooling breaks. Disabling it is inert on a direct connection, so there is one configuration for both:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}?prepareThreshold=0
    hikari:
      maximum-pool-size: 10
      leak-detection-threshold: 60000
```

Everything statement-scoped or client-side works unchanged behind the pooler: JSONB, full-text search, upserts, `RETURNING`, `SKIP LOCKED` queues, arrays, exclusion constraints, `unnest` bulk writes, Flyway.

What breaks is **session state**, because transaction pooling recycles the connection between transactions:

| Feature | Status | Use instead |
|---|---|---|
| `LISTEN` (receiving) | ❌ broken | poll; see §7 |
| `NOTIFY` (sending) | ✅ fine | — |
| `pg_advisory_lock` (session) | ❌ leaks across clients | `pg_try_advisory_xact_lock` (§6) |
| `SET foo = …` | ❌ leaks to other clients | `SET LOCAL` inside a transaction |
| Session temp tables, `WITH HOLD` cursors | ❌ broken | keep within one transaction |

**Rule:** any feature marked session-scoped is unavailable to a service that must also run in the company environment. If a single-environment service uses one, say so in its README — it pins the service to one environment.

Do **not** add PgBouncer to the k3s cluster to "match" the company setup. Hikari pools in-process; compatibility is a property of the code, not the deployment.

## Table of contents

0. [The two environments](#0-the-two-environments-read-this-first)
1. [JSONB](#1-jsonb)
2. [Full-text search](#2-full-text-search)
3. [Upsert (ON CONFLICT)](#3-upsert-on-conflict)
4. [RETURNING](#4-returning)
5. [Job queues with SKIP LOCKED](#5-job-queues-with-skip-locked)
6. [Advisory locks](#6-advisory-locks)
7. [LISTEN/NOTIFY](#7-listennotify)
8. [Arrays](#8-arrays)
9. [Ranges and exclusion constraints](#9-ranges-and-exclusion-constraints)
10. [Isolation levels and retries](#10-isolation-levels-and-retries)
11. [Bulk writes](#11-bulk-writes)
12. [Flyway and Postgres](#12-flyway-and-postgres)
13. [Driver, pool, and observability](#13-driver-pool-and-observability)
14. [Extensions worth knowing](#14-extensions-worth-knowing)
15. [Testing](#15-testing)

---

## 1. JSONB

Use for genuinely open-ended data (webhook payloads, per-tenant custom fields, scraped metadata). Do **not** use it as an escape hatch for columns you will filter and join on — those should be real columns.

### Mapping

Hibernate 6 maps JSON natively. **Do not add `hypersistence-utils`** for this — it was needed for Hibernate 5 and is now redundant.

```java
@Entity
public class Article {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private Metadata metadata;          // POJO, Map<String,Object>, or List<T>
}

public record Metadata(String source, List<String> tags, Instant scrapedAt) {}
```

Serialization goes through Hibernate's `FormatMapper`. To make it use your Spring `ObjectMapper` (so your Java Time / naming-strategy config applies):

```java
@Bean
HibernatePropertiesCustomizer jsonMapper(ObjectMapper objectMapper) {
    return props -> props.put(AvailableSettings.JSON_FORMAT_MAPPER,
                              new JacksonJsonFormatMapper(objectMapper));
}
```

For Spring Data JDBC / `JdbcClient`, register converters instead:

```java
@WritingConverter
class MetadataToPg implements Converter<Metadata, PGobject> {
    public PGobject convert(Metadata m) {
        var o = new PGobject();
        o.setType("jsonb");
        o.setValue(MAPPER.writeValueAsString(m));   // handle JsonProcessingException
        return o;
    }
}
```

### Indexing and querying

```sql
-- containment/existence only, ~2-3x smaller and faster to build
CREATE INDEX idx_article_meta ON articles USING GIN (metadata jsonb_path_ops);

-- default GIN: also supports ? / ?| / ?& key-existence operators
CREATE INDEX idx_article_meta ON articles USING GIN (metadata);

-- one hot scalar path, much cheaper than a whole-doc GIN
CREATE INDEX idx_article_source ON articles ((metadata->>'source'));
```

```java
@Query(value = """
        SELECT * FROM articles
        WHERE metadata @> CAST(:filter AS jsonb)
        """, nativeQuery = true)
List<Article> findByMetadata(@Param("filter") String filterJson);
```

**Gotchas**
- `@>` uses the GIN index. `metadata->>'source' = 'heise'` does **not**, unless you created the expression index above.
- Bind JSONB params as `String` + explicit `CAST(:p AS jsonb)`, or the driver sends `unknown` type and Postgres complains about ambiguous operators.
- Hibernate cannot dirty-check inside a JSON document reliably across all mappers. Reassign the field (`entity.setMetadata(newValue)`) rather than mutating the object graph in place.
- Never use JSONB for anything you'd want a foreign key on.

## 2. Full-text search

Covers the 80% case without adding Elasticsearch. Pick the right config for the language — `'german'` does German stemming and stopwords, `'simple'` does neither.

```sql
ALTER TABLE articles ADD COLUMN ts tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('german', coalesce(title,'')), 'A') ||
    setweight(to_tsvector('german', coalesce(body,'')),  'B')
  ) STORED;

CREATE INDEX idx_articles_ts ON articles USING GIN (ts);
```

Generated columns keep the vector in sync with no trigger and no application code. `coalesce` is required — `NULL` anywhere poisons the whole concatenation.

```java
@Query(value = """
        SELECT *, ts_rank(ts, websearch_to_tsquery('german', :q)) AS rank
        FROM articles
        WHERE ts @@ websearch_to_tsquery('german', :q)
        ORDER BY rank DESC
        LIMIT :limit
        """, nativeQuery = true)
List<Article> search(@Param("q") String query, @Param("limit") int limit);
```

- `websearch_to_tsquery` accepts user input safely (`quoted phrases`, `or`, `-negation`). `to_tsquery` throws on malformed input — never feed it raw user text.
- `plainto_tsquery` ANDs all terms; fine for simple cases.
- For typo tolerance use `pg_trgm` (`similarity()`, `%`, GIN/GiST on `gin_trgm_ops`). FTS and trigram solve different problems: FTS = stemmed word matching, trigram = fuzzy string matching. Often you want both.
- To project the `rank` alongside the entity, use an interface projection or `@SqlResultSetMapping`; a bare `List<Object[]>` works but ages badly.

## 3. Upsert (ON CONFLICT)

The single most useful Postgres feature for ingest pipelines.

```java
@Modifying
@Query(value = """
        INSERT INTO articles (url, title, fetched_at)
        VALUES (:url, :title, now())
        ON CONFLICT (url) DO UPDATE
          SET title = EXCLUDED.title,
              fetched_at = now()
        WHERE articles.title IS DISTINCT FROM EXCLUDED.title
        """, nativeQuery = true)
int upsert(@Param("url") String url, @Param("title") String title);
```

- **Do not use `JpaRepository.save()` for upserts.** It does a SELECT then INSERT-or-UPDATE, which is an extra round trip and racy under concurrency — two threads can both see "absent" and both INSERT.
- `ON CONFLICT` needs a unique index/constraint on the conflict target.
- `DO NOTHING` returns 0 rows and no `RETURNING` row — if you need the id either way, use `ON CONFLICT DO UPDATE SET url = EXCLUDED.url RETURNING id`, or do a follow-up SELECT.
- The `WHERE ... IS DISTINCT FROM` clause avoids no-op writes, which avoids bloat and spurious `updated_at` churn.
- `ON CONFLICT DO UPDATE` still burns a sequence value on conflict — id gaps are normal, don't chase them.
- After a native modifying query, entities in the persistence context are stale. Use `@Modifying(clearAutomatically = true, flushBeforeExecution = true)` or keep upserts out of JPA-managed flows entirely.

## 4. RETURNING

Saves a round trip and closes race windows.

```java
Long id = jdbcClient.sql("""
        INSERT INTO articles (url, title) VALUES (?, ?)
        ON CONFLICT (url) DO UPDATE SET title = EXCLUDED.title
        RETURNING id
        """)
    .params(url, title)
    .query(Long.class)
    .single();
```

Works on INSERT / UPDATE / DELETE. Combine with a CTE for atomic move-between-tables:

```sql
WITH moved AS (
  DELETE FROM inbox WHERE id = $1 RETURNING *
)
INSERT INTO archive SELECT * FROM moved RETURNING id;
```

Hibernate 6.5+ can use `RETURNING` for generated values via `@Generated(event = INSERT)`; for anything more interesting, use `JdbcClient`.

## 5. Job queues with SKIP LOCKED

This is why many services don't need Kafka or RabbitMQ. N workers, disjoint batches, no contention.

```java
@Transactional
public List<Job> claimBatch(int size) {
    return jdbcClient.sql("""
            UPDATE jobs SET status = 'running', claimed_at = now(), claimed_by = ?
            WHERE id IN (
              SELECT id FROM jobs
              WHERE status = 'pending' AND run_after <= now()
              ORDER BY priority DESC, run_after
              LIMIT ?
              FOR UPDATE SKIP LOCKED
            )
            RETURNING *
            """)
        .params(instanceId, size)
        .query(Job.class)
        .list();
}
```

- The claim and the work must be in the **same transaction**, or the lock is released before the work is done. If work is long, claim-and-commit (set `status='running'`) then process outside the transaction, with a reaper for rows stuck in `running` past a timeout.
- `SKIP LOCKED` skips locked rows silently — that's the point, but it means `LIMIT n` may return fewer than `n`.
- Partial index for the hot path: `CREATE INDEX ON jobs (priority DESC, run_after) WHERE status = 'pending';`
- JPA equivalent is `@Lock(LockModeType.PESSIMISTIC_WRITE)` + `@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2"))` where `-2` means SKIP LOCKED. It's obscure enough that native SQL is usually clearer.
- The `jobs` table gets hot and bloated; make sure autovacuum is tuned for it (`autovacuum_vacuum_scale_factor = 0.01` on that table).

## 6. Advisory locks

Cluster-wide mutex with no table. The standard way to make a `@Scheduled` task run on exactly one pod.

```java
boolean got = jdbcClient.sql("SELECT pg_try_advisory_xact_lock(?)")
        .param(lockKey)          // long, or two ints
        .query(Boolean.class).single();
```

- **Always use `pg_try_advisory_xact_lock`** (transaction-scoped, auto-released on commit/rollback), never `pg_advisory_lock` (session-scoped). With Hikari, a session lock that isn't explicitly unlocked stays held on a pooled connection and is leaked forever; behind PgBouncer it leaks across unrelated clients (§0). Transaction-scoped locks are safe in both environments.
- `try_` variants return immediately; the non-`try_` variants block indefinitely.
- Key is an arbitrary `bigint` you choose — hash your logical name, and write the mapping down somewhere, because collisions are silent deadlocks.
- **ShedLock** with `JdbcTemplateLockProvider` is the ergonomic option for scheduled tasks; it uses a table by default, which is easier to debug than advisory locks. Use plain advisory locks for ad-hoc critical sections.

## 7. LISTEN/NOTIFY

> **⚠️ Session-scoped — `LISTEN` does not work behind PgBouncer in transaction mode (§0).** A service that must also run in the company environment cannot rely on it. `NOTIFY` (sending) is statement-scoped and fine everywhere. Combined with how awkward it already is in Java, the practical answer here is **poll**.

Awkward in Java — worth knowing, rarely worth using.

```java
PGConnection pg = dataSource.getConnection().unwrap(PGConnection.class);
// LISTEN on a dedicated, non-pooled connection, then poll:
PGNotification[] n = pg.getNotifications(5000);
```

- Requires a raw `PGConnection` outside the pool, plus a polling thread. HikariCP will not give you a stable long-lived connection for this without carving out a separate `DataSource` with `maximumPoolSize=1` and a huge `maxLifetime`.
- Fire-and-forget: no delivery guarantee if no listener is connected. Never use it as the sole trigger for work — pair it with polling (`NOTIFY` = "check now", polling = "check eventually").
- Payload capped at 8000 bytes.
- Notifications are delivered on commit, not on the `NOTIFY` statement.
- If you want push-based, this is usually the moment to consider actually adding a broker — or to look at R2DBC, where the postgres driver exposes notifications as a `Flux`.

## 8. Arrays

Good for tags and other small, unordered, non-FK sets.

```java
@JdbcTypeCode(SqlTypes.ARRAY)
@Column(columnDefinition = "text[]")
private List<String> tags;
```

```sql
CREATE INDEX idx_tags ON articles USING GIN (tags);
SELECT * FROM articles WHERE tags && ARRAY['klima','energie'];  -- overlaps (OR)
SELECT * FROM articles WHERE tags @> ARRAY['klima'];            -- contains (AND)
```

- Cheaper than a join table when you never need referential integrity or per-tag metadata. The moment you want `tag_id → tags(id)`, use the join table.
- No FK constraints on elements. No per-element uniqueness. Rewriting one element rewrites the whole row.
- `unnest()` turns arrays into rows for joining/aggregating.
- Native queries taking array params need `connection.createArrayOf("text", arr)` or a `SqlParameterValue`; passing a raw `List` to a native `@Query` will not work.

## 9. Ranges and exclusion constraints

Database-enforced invariants that are very hard to get right in application code.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE bookings ADD CONSTRAINT no_double_booking
  EXCLUDE USING gist (room_id WITH =, during WITH &&);
```

No double-booking, guaranteed, under any concurrency, with no locking in your code. `btree_gist` is needed because `room_id` is scalar (`=`) rather than a range.

- Map `tstzrange` as a String or with a custom `UserType`; Hibernate has no built-in range type.
- Violation surfaces as SQLSTATE `23P01` (`exclusion_violation`) → Spring wraps it as `DataIntegrityViolationException`. Check the SQLSTATE, not the message text.
- `&&` = overlaps, `@>` = contains, `-|-` = adjacent. Bound inclusivity matters: `'[)'` is the sane default for time ranges.

## 10. Isolation levels and retries

Postgres defaults to READ COMMITTED. Spring's `@Transactional` default (`Isolation.DEFAULT`) inherits that.

- **REPEATABLE READ / SERIALIZABLE** use SSI: they never block on reads, but they abort transactions with SQLSTATE `40001` (`serialization_failure`). `40P01` is deadlock.
- **Spring does not retry these for you.** `@Transactional(isolation = SERIALIZABLE)` without a retry wrapper is a latent production incident.

```java
@Retryable(retryFor = {CannotSerializeTransactionException.class,
                       DeadlockLoserDataAccessException.class},
           maxAttempts = 4,
           backoff = @Backoff(delay = 50, multiplier = 2, random = true))
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transfer(...) { ... }
```

The retry **must wrap** the transaction, not live inside it — a proxy ordering mistake here means you retry inside an already-doomed transaction. Put `@Retryable` on an outer bean that calls the `@Transactional` one, or verify the advisor order.

In practice: stay on READ COMMITTED, and get correctness from unique constraints, `ON CONFLICT`, `FOR UPDATE`, and exclusion constraints. Reach for SERIALIZABLE only for genuinely multi-row invariants.

## 11. Bulk writes

```yaml
spring:
  jpa:
    properties:
      hibernate.jdbc.batch_size: 100
      hibernate.order_inserts: true
      hibernate.order_updates: true
  datasource:
    hikari:
      data-source-properties:
        reWriteBatchedInserts: true   # pgjdbc collapses N inserts into one multi-row statement
```

`reWriteBatchedInserts=true` is a large win and is off by default. Note that JPA batching is silently disabled by `GenerationType.IDENTITY` — Hibernate must round-trip for each id. Use `GenerationType.SEQUENCE` with a pooled optimizer (`allocationSize = 50`) if you batch-insert.

For real bulk, skip JPA:

```java
// One statement, arbitrary N, no batching machinery
jdbcClient.sql("""
        INSERT INTO articles (url, title)
        SELECT * FROM unnest(?::text[], ?::text[])
        ON CONFLICT (url) DO NOTHING
        """)
    .params(urlArray, titleArray)
    .update();
```

For very large loads, `COPY` via `CopyManager` (`pgConnection.getCopyAPI().copyIn(...)`) is an order of magnitude faster than anything else.

## 12. Flyway and Postgres

- **`CREATE INDEX CONCURRENTLY` cannot run in a transaction.** Flyway wraps migrations in one by default. Either name the migration so it's detected, or explicitly disable:
  ```sql
  -- V12__add_index.sql
  -- lock:none  (Flyway 9.7+ / or set flyway.executeInTransaction=false)
  CREATE INDEX CONCURRENTLY idx_articles_source ON articles (source);
  ```
  A failed `CONCURRENTLY` build leaves an **invalid** index behind that must be dropped manually — check `pg_index.indisvalid` after a failed deploy.
- Long `ALTER TABLE` takes an ACCESS EXCLUSIVE lock and queues behind/ahead of every other query. Always set `SET lock_timeout = '3s';` at the top of DDL migrations and retry, rather than freezing the whole app.
- `ADD COLUMN ... DEFAULT x` is instant since PG11 (no table rewrite). `ADD COLUMN ... NOT NULL` without a default is not.
- Adding a `NOT NULL` to an existing column: add a `CHECK (col IS NOT NULL) NOT VALID`, `VALIDATE CONSTRAINT` (takes only a SHARE UPDATE EXCLUSIVE lock), then `SET NOT NULL` (PG12+ recognizes the validated check and skips the scan).
- Keep `hibernate.hbm2ddl.auto=validate` in every environment. Flyway owns the schema; Hibernate only checks it agrees.

## 13. Driver, pool, and observability

- pgjdbc: `prepareThreshold=0` is **mandatory** for us, not a tuning knob — see §0. After 5 executions the driver otherwise switches to a server-side named prepared statement, which transaction pooling breaks. Inert on a direct connection.
- **PgBouncer in transaction mode** is how the company environment runs; the k3s cluster connects direct. Write for both (§0) rather than deploying a pooler to match.
- Hikari: `maximumPoolSize` should be small (`~ 2 * cores + effective_spindle_count`, often 10). Bigger pools are slower. `leakDetectionThreshold: 60000` catches connections held across a long HTTP call.
- Always set `statement_timeout` and `idle_in_transaction_session_timeout` at the role level. A stuck `idle in transaction` connection blocks vacuum and holds locks indefinitely.
- `pg_stat_statements` is the first thing to enable on any Postgres you own. `EXPLAIN (ANALYZE, BUFFERS)` is the second thing to learn.
- `spring.jpa.properties.hibernate.generate_statistics=true` plus Micrometer surfaces N+1 queries in Grafana before your users find them.

## 14. Extensions worth knowing

| Extension | Use for |
|---|---|
| `pg_trgm` | fuzzy/typo-tolerant matching, `LIKE '%x%'` acceleration |
| `pgvector` | embeddings; HNSW index, `<=>` cosine distance |
| `btree_gist` | needed to mix scalars into exclusion constraints |
| `pg_partman` | time-based partition maintenance |
| `pg_cron` | in-database scheduling (careful: bypasses your app's observability) |
| `pg_stat_statements` | query-level performance stats. Enable it. |

`uuid-ossp` is obsolete for most uses: `gen_random_uuid()` is built in since PG13. Prefer **UUIDv7** (`uuidv7()`, native in PG18) over v4 for primary keys — time-ordered means far better B-tree locality and less WAL.

## 15. Testing

Testcontainers, always. H2 in Postgres-compat mode does not have JSONB, FTS, `SKIP LOCKED`, or partial indexes, so it validates nothing that matters here.

```java
@Testcontainers
@SpringBootTest
class ArticleRepositoryTest {
    @Container @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17-alpine");
}
```

`@ServiceConnection` (Boot 3.1+) removes the need for `@DynamicPropertySource`. Reuse the container across the suite (`.withReuse(true)` + `testcontainers.reuse.enable=true` in `~/.testcontainers.properties`) — startup dominates the runtime otherwise. Let Flyway run the real migrations against it; that tests the migrations too.

---

## Decision quick-reference

| If you're about to... | Do this instead |
|---|---|
| `save()` in a loop for ingest | `INSERT ... ON CONFLICT` |
| add Elasticsearch for search | `tsvector` generated column + GIN |
| add Kafka for a work queue | `FOR UPDATE SKIP LOCKED` |
| add Redis for a distributed lock | `pg_try_advisory_xact_lock` |
| add a `tags` join table for display-only labels | `text[]` + GIN |
| write overlap-checking logic in Java | `EXCLUDE USING gist` |
| use `SERIALIZABLE` | first try a unique constraint |
| use H2 for tests | Testcontainers |
| add PgBouncer to k3s to match the company env | set `prepareThreshold=0` — compatibility is a code property (§0) |
| `LISTEN` for push updates | poll; `LISTEN` is unavailable behind the pooler (§0) |

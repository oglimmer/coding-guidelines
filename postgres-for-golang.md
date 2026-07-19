# Postgres for Go

Reference for using Postgres-native features from Go. Assumes `github.com/jackc/pgx/v5` and PostgreSQL 15+.

**Scope:** the database layer of a Go HTTP service — JSONB, full-text search, upserts, `RETURNING`, `SKIP LOCKED` job queues, advisory locks, `LISTEN/NOTIFY`, arrays, exclusion constraints, `CopyFrom` bulk loads, isolation/retry, and pgxpool tuning. Layers onto [go-backend.md](go-backend.md), which owns the service shape (chi, config, auth, layout, migration runner). Spring equivalent: [postgres-for-spring.md](postgres-for-spring.md). Does **not** apply to our MySQL/MariaDB Go services (`linky`, `easy-host-k8s`, `coffee-diary`) — those use sqlx, not pgx.

**Every service must run against both a direct Postgres and a transaction-mode PgBouncer — start at [§0](#0-the-two-environments-read-this-first).**

**Prime directive:** use **pgx's native interface** (`pgxpool`), not `database/sql`, and configure it so the same binary runs in both our environments. Going through `database/sql` costs you the binary protocol, the full Postgres type system, `CopyFrom`, and `LISTEN/NOTIFY`. Use the `stdlib` shim only when a third-party library forces `*sql.DB` on you.

## 0. The two environments (read this first)

Our Go services must run in both, from one build:

| | k3s cluster (`kubectl`) | Company environment |
|---|---|---|
| Postgres access | direct to `postgres.default.svc` | **PgBouncer, transaction mode** |

**Write PgBouncer-compatible code; never require PgBouncer.** The settings that make a connection pooler-safe are inert on a direct connection, so there is exactly one configuration and no build flag:

```go
cfg, err := pgxpool.ParseConfig(dsn)

// PgBouncer transaction mode forbids server-side named prepared statements.
// QueryExecModeExec pipelines Parse+Bind+Describe+Execute+Sync as one message
// group, so the exchange completes inside a single pooler-owned transaction.
// Inert on a direct connection — always set it.
cfg.ConnConfig.DefaultQueryExecMode      = pgx.QueryExecModeExec
cfg.ConnConfig.StatementCacheCapacity    = 0
cfg.ConnConfig.DescriptionCacheCapacity  = 0

cfg.MaxConns = 10
pool, err := pgxpool.NewWithConfig(ctx, cfg)
```

This is the *only* thing `database/sql` was ever buying us — it is not tied to the adapter, and `pgxpool` takes the identical settings via `cfg.ConnConfig`. Pgx-native and pooler-safe are not in tension.

### What is safe in both environments

Everything statement-scoped or client-side — the bulk of this document. JSONB, full-text search, upserts, `RETURNING`, `SKIP LOCKED` queues, arrays, exclusion constraints, `CollectRows`/`RowToStructByName`, `pgtype.Range`, `CopyFrom`, and `unnest` bulk loads all work unchanged behind a transaction pooler.

### What breaks under PgBouncer — session state

These hold state *across* transactions, which is exactly what transaction pooling recycles:

| Feature | Status | Use instead |
|---|---|---|
| `LISTEN` (receiving) | ❌ broken | poll; see §8 |
| `NOTIFY` (sending) | ✅ fine | — |
| `pg_advisory_lock` (session) | ❌ leaks | `pg_try_advisory_xact_lock` (§7) |
| `SET foo = …` | ❌ silently leaks to other clients | `SET LOCAL` inside a transaction |
| `WITH HOLD` cursors, session temp tables | ❌ broken | keep within one transaction |

**Rule:** if a feature in this doc is marked session-scoped, it is unavailable to any service that must also run in the company environment. Single-environment services may use it — say so explicitly in the repo's README, because it pins that service to one environment.

## Table of contents

0. [The two environments](#0-the-two-environments-read-this-first)
1. [Driver setup](#1-driver-setup)
2. [JSONB](#2-jsonb)
3. [Full-text search](#3-full-text-search)
4. [Upsert (ON CONFLICT)](#4-upsert-on-conflict)
5. [RETURNING](#5-returning)
6. [Job queues with SKIP LOCKED](#6-job-queues-with-skip-locked)
7. [Advisory locks](#7-advisory-locks)
8. [LISTEN/NOTIFY](#8-listennotify)
9. [Arrays](#9-arrays)
10. [Ranges and exclusion constraints](#10-ranges-and-exclusion-constraints)
11. [Isolation levels and retries](#11-isolation-levels-and-retries)
12. [Bulk writes: CopyFrom and unnest](#12-bulk-writes-copyfrom-and-unnest)
13. [Error handling](#13-error-handling)
14. [Migrations](#14-migrations)
15. [Pool tuning and observability](#15-pool-tuning-and-observability)
16. [Extensions worth knowing](#16-extensions-worth-knowing)
17. [Testing](#17-testing)

---

## 1. Driver setup

```go
cfg, err := pgxpool.ParseConfig(dsn)

// Pooler-safe; inert on a direct connection. See §0 — always set these three.
cfg.ConnConfig.DefaultQueryExecMode     = pgx.QueryExecModeExec
cfg.ConnConfig.StatementCacheCapacity   = 0
cfg.ConnConfig.DescriptionCacheCapacity = 0

cfg.MaxConns = 10
cfg.MinConns = 2
cfg.MaxConnLifetime = time.Hour
cfg.MaxConnIdleTime = 30 * time.Minute
cfg.HealthCheckPeriod = time.Minute

pool, err := pgxpool.NewWithConfig(ctx, cfg)
defer pool.Close()
```

Ping in a retry loop at startup (~30×, 1s backoff) so the service tolerates a slow-booting Postgres.

Pass a `context.Context` with a deadline to every query. pgx cancels the query server-side on context cancellation — this is the mechanism that keeps a slow query from outliving its HTTP request.

**Scanning.** Use `github.com/jackc/pgx/v5/pgxutil` or, far more commonly, `pgx.CollectRows` with `pgx.RowToStructByName`:

```go
rows, err := pool.Query(ctx, `SELECT id, url, title FROM articles WHERE source = $1`, src)
if err != nil { return nil, err }
articles, err := pgx.CollectRows(rows, pgx.RowToStructByName[Article])
```

`RowToStructByName` matches on the `db:"..."` struct tag. `RowToStructByPos` matches on order — brittle, avoid. For a single row: `pgx.CollectOneRow`, which returns `pgx.ErrNoRows` when empty.

**Query layer choice**
- **pgx directly** — full control, no magic. Default choice.
- **sqlc** — generates type-safe Go from your SQL, with a pgx/v5 driver target. Excellent fit for Postgres-heavy code: you write the real SQL, you get the structs for free. Strongly recommended for anything non-trivial.
- **GORM** — hides exactly the features in this document. If you're reading this doc, GORM is working against you.

## 2. JSONB

Use for genuinely open-ended data. Not as an escape hatch for columns you filter and join on.

```go
type Metadata struct {
    Source    string    `json:"source"`
    Tags      []string  `json:"tags"`
    ScrapedAt time.Time `json:"scrapedAt"`
}

// pgx marshals/unmarshals any struct to jsonb automatically — no Valuer/Scanner needed.
var m Metadata
err := pool.QueryRow(ctx, `SELECT metadata FROM articles WHERE id = $1`, id).Scan(&m)

_, err = pool.Exec(ctx,
    `INSERT INTO articles (url, metadata) VALUES ($1, $2)`, url, m)
```

pgx v5 handles `jsonb` ↔ Go struct via `encoding/json` out of the box for any type it doesn't otherwise recognize. Implement `driver.Valuer`/`sql.Scanner` only if you need custom encoding, or register a codec on the type map for a custom JSON library:

```go
cfg.AfterConnect = func(ctx context.Context, c *pgx.Conn) error {
    c.TypeMap().RegisterType(&pgtype.Type{
        Name:  "jsonb",
        OID:   pgtype.JSONBOID,
        Codec: &pgtype.JSONBCodec{Marshal: sonic.Marshal, Unmarshal: sonic.Unmarshal},
    })
    return nil
}
```

### Indexing and querying

```sql
-- containment/existence only; smaller and faster to build
CREATE INDEX idx_article_meta ON articles USING GIN (metadata jsonb_path_ops);

-- default GIN: also supports ? / ?| / ?& key-existence operators
CREATE INDEX idx_article_meta ON articles USING GIN (metadata);

-- one hot scalar path, much cheaper than a whole-doc GIN
CREATE INDEX idx_article_source ON articles ((metadata->>'source'));
```

```go
rows, _ := pool.Query(ctx,
    `SELECT * FROM articles WHERE metadata @> $1`,
    map[string]any{"source": "heise"})   // pgx encodes the map as jsonb
```

**Gotchas**
- `@>` uses the GIN index; `metadata->>'source' = 'heise'` does not, unless the expression index exists.
- `?` is the JSONB key-existence operator **and** pgx's... no — pgx uses `$1` placeholders, so `?` is unambiguous. (This *is* a real problem in `database/sql` wrappers that rewrite `?`; another reason to use pgx natively.)
- A `NULL` jsonb column scanned into a non-pointer struct errors. Use `*Metadata` or `pgtype.JSONB`.
- `jsonb` reorders keys and drops duplicates. `json` preserves the input text but can't be indexed usefully. Always use `jsonb`.

## 3. Full-text search

Covers the 80% case without adding Elasticsearch. Pick the right config — `'german'` does German stemming and stopwords, `'simple'` does neither.

```sql
ALTER TABLE articles ADD COLUMN ts tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('german', coalesce(title,'')), 'A') ||
    setweight(to_tsvector('german', coalesce(body,'')),  'B')
  ) STORED;

CREATE INDEX idx_articles_ts ON articles USING GIN (ts);
```

Generated columns keep the vector in sync with no trigger and no application code. `coalesce` is required — `NULL` anywhere poisons the whole concatenation.

```go
rows, err := pool.Query(ctx, `
    SELECT id, title, ts_rank(ts, websearch_to_tsquery('german', $1)) AS rank
    FROM articles
    WHERE ts @@ websearch_to_tsquery('german', $1)
    ORDER BY rank DESC
    LIMIT $2`, userQuery, limit)
```

- `websearch_to_tsquery` accepts raw user input safely (`quoted phrases`, `or`, `-negation`). `to_tsquery` **panics on malformed input** — never feed it user text.
- For typo tolerance use `pg_trgm` (`similarity()`, `%`). FTS = stemmed word matching, trigram = fuzzy string matching. Frequently you want both, combined with `GREATEST(ts_rank(...), similarity(...))`.
- `ts_headline()` gives you highlighted snippets, but it's expensive — run it only on the page of results you're returning, in a subquery/CTE after `LIMIT`.

## 4. Upsert (ON CONFLICT)

The single most useful Postgres feature for ingest pipelines.

```go
var id int64
err := pool.QueryRow(ctx, `
    INSERT INTO articles (url, title, fetched_at)
    VALUES ($1, $2, now())
    ON CONFLICT (url) DO UPDATE
      SET title = EXCLUDED.title, fetched_at = now()
      WHERE articles.title IS DISTINCT FROM EXCLUDED.title
    RETURNING id`, url, title).Scan(&id)
```

- Requires a unique index/constraint on the conflict target.
- `DO NOTHING` returns **no row**, so `RETURNING id` + `Scan` gives you `pgx.ErrNoRows` on conflict. If you always need the id, use the `DO UPDATE SET url = EXCLUDED.url` no-op trick, or handle `ErrNoRows` with a follow-up SELECT.
- The `WHERE ... IS DISTINCT FROM` guard skips no-op writes → less bloat, no spurious `updated_at` churn, and `DO NOTHING`-like semantics with a `RETURNING` row still available when it does fire (careful: when the guard fails, no row is returned).
- Sequence values are consumed even on conflict. Id gaps are normal.
- `ON CONFLICT` only sees one conflict target. If you have two unique constraints that could both fire, you need `ON CONFLICT ON CONSTRAINT name` or explicit error handling per constraint.

## 5. RETURNING

Saves a round trip and closes race windows. Works on INSERT / UPDATE / DELETE.

```go
var id int64
var createdAt time.Time
err := pool.QueryRow(ctx,
    `DELETE FROM inbox WHERE id = $1 RETURNING id, created_at`, id).
    Scan(&id, &createdAt)
```

Combine with a CTE for an atomic move:

```sql
WITH moved AS (
  DELETE FROM inbox WHERE id = $1 RETURNING *
)
INSERT INTO archive SELECT * FROM moved RETURNING id;
```

Note that all CTEs in a statement see the same snapshot — `moved` cannot see rows the INSERT writes, and vice versa. This is a feature (it makes the statement atomic) and a trap (self-referential CTEs don't do what you expect).

## 6. Job queues with SKIP LOCKED

This is why many Go services don't need Kafka or NATS. N workers, disjoint batches, no contention.

```go
func claimBatch(ctx context.Context, pool *pgxpool.Pool, n int) ([]Job, error) {
    rows, err := pool.Query(ctx, `
        UPDATE jobs SET status = 'running', claimed_at = now(), claimed_by = $1
        WHERE id IN (
          SELECT id FROM jobs
          WHERE status = 'pending' AND run_after <= now()
          ORDER BY priority DESC, run_after
          LIMIT $2
          FOR UPDATE SKIP LOCKED
        )
        RETURNING id, payload, attempts`, instanceID, n)
    if err != nil { return nil, err }
    return pgx.CollectRows(rows, pgx.RowToStructByName[Job])
}
```

Two shapes:
- **Claim-and-hold** — `BEGIN; SELECT ... FOR UPDATE SKIP LOCKED; <do work>; COMMIT;`. Crash-safe (lock dies with the connection) but holds a transaction open for the whole job. Only for short work: a long-open transaction blocks vacuum cluster-wide.
- **Claim-and-commit** — the version above. Mark `running`, commit, work outside the transaction, mark `done`. Needs a reaper for rows stuck in `running` past a timeout, and your handler must be idempotent (at-least-once).

Notes
- `SKIP LOCKED` silently skips locked rows, so `LIMIT n` may return fewer than `n`. That's the point.
- Partial index for the hot path: `CREATE INDEX ON jobs (priority DESC, run_after) WHERE status = 'pending';`
- The `jobs` table gets hot and bloats. Tune autovacuum per-table: `ALTER TABLE jobs SET (autovacuum_vacuum_scale_factor = 0.01);`
- `river` (by the pgx author) and `gue` are solid off-the-shelf implementations if you'd rather not hand-roll it.

## 7. Advisory locks

Cluster-wide mutex with no table. The standard way to make a cron/leader task run on exactly one pod.

```go
// Transaction-scoped: released automatically on commit/rollback. Prefer this.
tx, err := pool.Begin(ctx)
defer tx.Rollback(ctx)

var got bool
err = tx.QueryRow(ctx, `SELECT pg_try_advisory_xact_lock($1)`, lockKey).Scan(&got)
if !got { return nil }  // someone else has it
// ... critical section ...
return tx.Commit(ctx)
```

- **Always use `pg_try_advisory_xact_lock`.** Transaction-scoped locks are safe in both environments (§0). Session-scoped `pg_advisory_lock` leaks on a *pooled* connection — the connection returns to the pool still holding the lock and the next borrower inherits it — and under PgBouncer it leaks across unrelated *clients*, which is worse and near-impossible to debug. Treat session-scoped advisory locks as unavailable to us.
- `try_` returns immediately; the non-`try_` variants block indefinitely with no context cancellation on the lock wait itself.
- The key is an arbitrary `bigint` you choose. Hash your logical name (`hash/fnv`) and keep a registry — collisions are silent, mysterious serialization.
- `pg_locks` shows held advisory locks (`locktype = 'advisory'`), which makes debugging tractable.

## 8. LISTEN/NOTIFY

> **⚠️ Session-scoped — `LISTEN` does not work behind PgBouncer in transaction mode.** A service that must run in the company environment (§0) cannot rely on it. `NOTIFY` (sending) is statement-scoped and fine everywhere.
>
> Options, in order of preference: (a) **poll** — the correctness baseline you need regardless, see below; (b) open a **second, direct** connection to Postgres that bypasses the pooler *only* for the listener, and degrade to polling when that isn't available; (c) accept single-environment deployment and document it in the repo README.

This is where Go clearly beats the JVM — pgx exposes it cleanly.

```go
conn, err := pool.Acquire(ctx)          // dedicated connection, NOT returned to the pool
defer conn.Release()

_, err = conn.Exec(ctx, "LISTEN article_inserted")

for {
    n, err := conn.Conn().WaitForNotification(ctx)
    if err != nil {
        if ctx.Err() != nil { return ctx.Err() }
        // connection dropped: reconnect, re-LISTEN, and re-scan for missed work
        return err
    }
    handle(n.Payload)
}
```

- `WaitForNotification` blocks and respects context cancellation. No polling thread, no goroutine tricks.
- **Fire-and-forget.** If no one is listening, the notification is gone. Never make it the sole trigger for work — pair it with periodic polling. `NOTIFY` means "check now"; polling means "check eventually". This combination gives you low latency *and* correctness.
- On reconnect you have a gap. Always re-scan for pending work after re-LISTENing.
- Payload capped at 8000 bytes. Send an id, not the data.
- Delivered on commit, not at the `NOTIFY` statement.
- The listening connection is out of the pool for its whole life — size `MaxConns` accordingly.

## 9. Arrays

Good for tags and other small, unordered, non-FK sets. pgx maps Go slices to Postgres arrays natively.

```go
type Article struct {
    ID   int64    `db:"id"`
    Tags []string `db:"tags"`   // text[] — just works
}

rows, _ := pool.Query(ctx,
    `SELECT * FROM articles WHERE tags && $1`, []string{"klima", "energie"})
```

```sql
CREATE INDEX idx_tags ON articles USING GIN (tags);
SELECT * FROM articles WHERE tags && ARRAY['klima','energie'];  -- overlaps (OR)
SELECT * FROM articles WHERE tags @> ARRAY['klima'];            -- contains (AND)
```

- Cheaper than a join table when you never need referential integrity or per-tag metadata. The moment you want `tag_id → tags(id)`, use the join table.
- Nullable elements need `[]*string` or `[]pgtype.Text`; a `NULL` element scanned into `[]string` errors.
- `unnest()` turns arrays into rows — the basis of the bulk-insert trick in §12.
- `= ANY($1)` is the idiomatic Postgres replacement for building a dynamic `IN (?,?,?)` list. Use it; never string-concatenate an IN list.

## 10. Ranges and exclusion constraints

Database-enforced invariants that are very hard to get right in application code.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE bookings ADD CONSTRAINT no_double_booking
  EXCLUDE USING gist (room_id WITH =, during WITH &&);
```

No double-booking, guaranteed, under any concurrency, with zero locking in your code. `btree_gist` is required because `room_id` is scalar (`=`) rather than a range.

```go
r := pgtype.Range[pgtype.Timestamptz]{
    Lower:     pgtype.Timestamptz{Time: start, Valid: true},
    Upper:     pgtype.Timestamptz{Time: end, Valid: true},
    LowerType: pgtype.Inclusive,
    UpperType: pgtype.Exclusive,
    Valid:     true,
}
_, err := pool.Exec(ctx, `INSERT INTO bookings (room_id, during) VALUES ($1, $2)`, roomID, r)
```

- pgx has first-class range support via `pgtype.Range[T]` — a real advantage over Hibernate here.
- `'[)'` (inclusive lower, exclusive upper) is the sane default for time ranges.
- Violation → SQLSTATE `23P01`. Check the code, not the message (see §13).
- `&&` overlaps, `@>` contains, `-|-` adjacent.

## 11. Isolation levels and retries

Postgres defaults to READ COMMITTED.

- **REPEATABLE READ / SERIALIZABLE** use SSI: they never block on reads, but abort with SQLSTATE `40001` (`serialization_failure`). `40P01` is deadlock. Both are **retryable** and you must retry them.

```go
func withRetry(ctx context.Context, pool *pgxpool.Pool, fn func(pgx.Tx) error) error {
    const maxAttempts = 4
    for attempt := 1; ; attempt++ {
        err := pgx.BeginTxFunc(ctx, pool,
            pgx.TxOptions{IsoLevel: pgx.Serializable}, fn)
        if err == nil { return nil }

        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) &&
            (pgErr.Code == "40001" || pgErr.Code == "40P01") &&
            attempt < maxAttempts {
            // exponential backoff with jitter
            d := time.Duration(1<<attempt) * 10 * time.Millisecond
            select {
            case <-time.After(d + time.Duration(rand.Int63n(int64(d)))):
                continue
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return err
    }
}
```

`pgx.BeginTxFunc` handles commit/rollback for you — use it rather than hand-rolling `defer tx.Rollback(ctx)` (though that defer is harmless after a successful commit; `Rollback` on a committed tx returns `pgx.ErrTxClosed`, which you ignore).

In practice: stay on READ COMMITTED, and get correctness from unique constraints, `ON CONFLICT`, `FOR UPDATE`, and exclusion constraints. Reach for SERIALIZABLE only for genuinely multi-row invariants.

## 12. Bulk writes: CopyFrom and unnest

Two tiers.

**`unnest()`** — one round trip, works with `ON CONFLICT`:

```go
_, err := pool.Exec(ctx, `
    INSERT INTO articles (url, title)
    SELECT * FROM unnest($1::text[], $2::text[])
    ON CONFLICT (url) DO NOTHING`, urls, titles)
```

This is the workhorse. It's a single statement, arbitrary N, and it supports upsert — which `COPY` does not.

**`CopyFrom`** — the COPY protocol, fastest possible, no `ON CONFLICT`:

```go
n, err := pool.CopyFrom(ctx,
    pgx.Identifier{"articles"},
    []string{"url", "title", "fetched_at"},
    pgx.CopyFromSlice(len(rows), func(i int) ([]any, error) {
        return []any{rows[i].URL, rows[i].Title, rows[i].FetchedAt}, nil
    }))
```

Order of magnitude faster than batched inserts for large loads. It fails the whole batch on any constraint violation. Standard pattern for upsert-at-scale: `COPY` into a `TEMP` table, then `INSERT ... SELECT ... ON CONFLICT` from it.

`CopyFrom` itself is safe in both environments. The `TEMP` table variant is not, unless the `CREATE TEMP TABLE`, the `COPY`, and the `INSERT ... SELECT` all run **inside one explicit transaction** on the same `pgx.Tx` — behind PgBouncer the connection is recycled at transaction end and the temp table vanishes. Use `ON COMMIT DROP` and keep it to one `tx`.

**`SendBatch`** — pipelines N statements in one round trip. Good for heterogeneous statements; not a substitute for `unnest`/`COPY` on homogeneous bulk.

```go
batch := &pgx.Batch{}
for _, a := range articles {
    batch.Queue(`INSERT INTO articles (url) VALUES ($1)`, a.URL)
}
br := pool.SendBatch(ctx, batch)
defer br.Close()
```

Note `br.Close()` returns an error you must check — it surfaces failures from queued statements you never explicitly read.

## 13. Error handling

Never match on error strings. Always use `*pgconn.PgError` and the SQLSTATE.

```go
var pgErr *pgconn.PgError
if errors.As(err, &pgErr) {
    switch pgErr.Code {
    case "23505": // unique_violation
        return ErrDuplicate            // pgErr.ConstraintName tells you which one
    case "23503": // foreign_key_violation
        return ErrMissingParent
    case "23P01": // exclusion_violation
        return ErrOverlap
    case "40001", "40P01": // serialization_failure, deadlock_detected
        return ErrRetryable
    case "57014": // query_canceled (usually your statement_timeout)
        return ErrTimeout
    }
}
if errors.Is(err, pgx.ErrNoRows) { return ErrNotFound }
```

`pgErr.ConstraintName` is what lets you map "which unique constraint fired" to a meaningful domain error. Name your constraints deliberately for this reason.

## 14. Migrations

**Our services do not use a migration library.** They embed numbered SQL via `//go:embed`, register it in a `migrations` slice, and run it at startup — the pattern is specified in [go-backend.md](go-backend.md#migrations), which owns it. Follow that; the guidance below is about the SQL itself and applies whatever runs it.

- **`CREATE INDEX CONCURRENTLY` cannot run inside a transaction.** With our startup runner each migration is `Exec`'d on its own, so a migration containing *only* that statement works. Put it in its own file — as soon as a file has multiple statements, pgx sends them as one implicit transaction and `CONCURRENTLY` fails.
- A failed `CONCURRENTLY` build leaves an **invalid index** behind that must be dropped manually. Check `SELECT * FROM pg_index WHERE NOT indisvalid;` after a failed deploy.
- Always `SET lock_timeout = '3s';` at the top of DDL migrations. An `ALTER TABLE` waiting on an ACCESS EXCLUSIVE lock queues *every subsequent query* behind it — a 3-second timeout and a retry is infinitely better than a total outage.
- `ADD COLUMN ... DEFAULT x` is instant since PG11. `ADD COLUMN ... NOT NULL` without a default is not.
- Adding `NOT NULL` to an existing column: `ADD CONSTRAINT ... CHECK (col IS NOT NULL) NOT VALID` → `VALIDATE CONSTRAINT` (weak lock) → `SET NOT NULL` (PG12+ skips the scan).
- Migrations must stay idempotent (`CREATE TABLE IF NOT EXISTS`, `ALTER … IF NOT EXISTS`) — `Migrate()` re-runs every file on every startup, and every pod starts.
- If you're using sqlc, migrations are the source of truth it generates from — keep them clean.

## 15. Pool tuning and observability

- `MaxConns` should be **small**. `~2 × cores` is a good start; often 10 is plenty. Bigger pools are slower under load, not faster.
- Set `statement_timeout` and `idle_in_transaction_session_timeout` at the role level. An `idle in transaction` connection holds locks and blocks vacuum indefinitely.
- **PgBouncer in transaction mode** is not optional for us — it is how the company environment runs (§0). pgx uses named prepared statements by default, so the three `ConnConfig` settings from §0 are mandatory in every service. Setting them in code rather than the DSN keeps the guarantee out of environment config, where it can be lost in a copy-paste.
- Do **not** add PgBouncer to the k3s cluster to "match" the company environment. `pgxpool` handles pooling in-process; a pooler earns its place only at high pod counts. Compatibility is a code property, not a deployment one.
- `pgxpool.Pool.Stat()` exposes `AcquireCount`, `AcquireDuration`, `EmptyAcquireCount` — export these to Prometheus. `EmptyAcquireCount` climbing means the pool is too small (or you're leaking connections).
- Tracing: implement `pgx.QueryTracer` or use `github.com/exaring/otelpgx` for OpenTelemetry spans per query.
- `pg_stat_statements` is the first thing to enable on any Postgres you own. `EXPLAIN (ANALYZE, BUFFERS)` is the second thing to learn.

## 16. Extensions worth knowing

| Extension | Use for |
|---|---|
| `pg_trgm` | fuzzy/typo-tolerant matching, `LIKE '%x%'` acceleration |
| `pgvector` | embeddings; HNSW index, `<=>` cosine distance (`pgvector-go` for pgx types) |
| `btree_gist` | needed to mix scalars into exclusion constraints |
| `pg_partman` | time-based partition maintenance |
| `pg_cron` | in-database scheduling (careful: bypasses your app's observability) |
| `pg_stat_statements` | query-level performance stats. Enable it. |

`uuid-ossp` is obsolete: `gen_random_uuid()` is built in since PG13. Prefer **UUIDv7** (`uuidv7()`, native in PG18; `github.com/google/uuid` v1.6+ has `NewV7()` client-side) over v4 for primary keys — time-ordered means far better B-tree locality and less WAL.

## 17. Testing

Testcontainers, always. There is no in-memory Postgres, and SQLite validates nothing that matters here.

```go
func TestMain(m *testing.M) {
    ctx := context.Background()
    pg, err := postgres.Run(ctx, "postgres:17-alpine",
        postgres.WithDatabase("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2)),
    )
    // ... run migrations against pg.ConnectionString(ctx), then m.Run()
}
```

- The `WithOccurrence(2)` wait strategy matters: the Postgres image logs "ready" once during its init phase before restarting. Waiting for the first occurrence gives you a connection that dies immediately.
- One container per package, `TestMain`, migrations run once. Per-test isolation via `TRUNCATE ... RESTART IDENTITY CASCADE` or a transaction-per-test that always rolls back.
- `postgres.WithSnapshot`/`pg.Restore(ctx)` gives fast per-test resets against a seeded baseline.

---

## Decision quick-reference

| If you're about to... | Do this instead |
|---|---|
| loop `INSERT` for ingest | `unnest()` one-shot, or `CopyFrom` |
| use `database/sql` + `lib/pq` | pgx v5 native (`lib/pq` is in maintenance mode) |
| add PgBouncer to k3s to match the company env | set the three §0 `ConnConfig` options — compatibility is a code property |
| use `LISTEN` as the only trigger for work | poll as the baseline; `LISTEN` is a latency optimisation you may not have (§0) |
| `SET search_path = …` on a pooled connection | `SET LOCAL` inside a transaction |
| hand-write scan boilerplate | sqlc, or `pgx.CollectRows` + `RowToStructByName` |
| add Elasticsearch for search | `tsvector` generated column + GIN |
| add Kafka/NATS for a work queue | `FOR UPDATE SKIP LOCKED` (or `river`) |
| add Redis for a distributed lock | `pg_try_advisory_xact_lock` |
| build an `IN (?,?,?)` string | `= ANY($1)` with a slice |
| match `err.Error()` against "duplicate key" | `errors.As` → `pgErr.Code == "23505"` |
| write overlap-checking logic in Go | `EXCLUDE USING gist` |
| use SQLite/mocks for repo tests | Testcontainers |

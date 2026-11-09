# Juju Schema and Query Benchmark Harness

## Status

Initial planning document. This will be expanded as requirements and design
inputs are collected.

## Goal

Design a benchmark harness that represents Juju's database schema and query
profile closely enough to measure database and query-layer changes against a
stable, reproducible baseline.

## Baseline

The initial baseline uses the versions currently pinned by Juju:

- `github.com/canonical/go-dqlite/v3` at `v3.0.3`.
- `github.com/canonical/sqlair` at
  `v0.0.0-20260218132926-bd54c4999dea`.
- Juju's current controller and model schemas.
- Juju's current Sqlair queries and transaction patterns.

Exact module versions, Juju revision, schema hashes, and harness configuration
must be captured with every benchmark result so that runs can be reproduced.

The current Juju module also specifies Go `1.26.5`. The Go version is part of
the baseline because Sqlair delegates prepared-statement connection management
to `database/sql`.

## Current Execution and Caching Behaviour

The word "plan" needs care in this document. There are three different things
that can otherwise be conflated:

- Juju's `domain.StateBase` caches parsed `sqlair.Statement` objects by SQLair
  query text. This avoids repeatedly parsing and type-binding the SQLair
  expression, but it is not a database query plan.
- Sqlair caches `database/sql.Stmt` objects. These represent prepared SQL and
  may own a different driver-prepared statement for each physical connection
  used by the standard-library connection pool.
- Dqlite prepares the SQL into SQLite VM bytecode and a query plan. In the Go
  driver this is represented by a connection-bound remote statement handle.

### Assertion Assessment

| Assertion | Assessment | Current behaviour |
| --- | --- | --- |
| The Dqlite Go driver relies on `database/sql` for query-plan reuse. | Qualified | The driver has no Go-side statement cache. It exposes both explicit prepare/statement operations and direct raw-SQL query/exec operations. `database/sql` manages the connection-specific prepared statements only when the caller owns a `sql.Stmt`. Direct calls bypass that lifecycle. |
| Sqlair caches query plans using standard-library behaviour. | Confirmed with terminology adjusted | Sqlair caches `*sql.Stmt` values, keyed by the Sqlair DB, Sqlair statement, and generated SQL. `database/sql` then prepares and retains a driver statement per physical connection as required. Sqlair itself does not cache an independent SQLite plan. |
| Juju's explicit transactions mean cached plans do not survive transactions. | Correct for the normal Juju path, but not universally | Juju's normal transaction-only usage does not populate Sqlair's `*sql.Stmt` cache, so there is usually no cached driver statement to reuse. An entry that was first prepared through `sqlair.DB.Query`, however, can be adapted to a transaction and its connection-specific driver statement can remain in the parent `sql.Stmt` after commit. |

### Baseline Cache-Miss Path

The normal Juju domain path is:

```text
domain.StateBase.Prepare
    -> cached parsed sqlair.Statement
TxnRunner.Txn
    -> sqlair.DB.Begin
sqlair.TX.Query
    -> Sqlair driver-statement cache miss
sql.Tx.QueryContext / ExecContext
    -> go-dqlite Conn.QueryContext / ExecContext
    -> Dqlite raw-SQL QUERY_SQL / EXEC_SQL request
```

The Dqlite connection implements the optional `driver.QueryerContext` and
`driver.ExecerContext` interfaces. Consequently, `database/sql` uses those
direct operations and does not invoke its prepare-execute-close fallback. The
Go driver sends SQL text in the request and retains no client-side prepared
statement handle for the next call.

Sqlair's transaction path deliberately behaves as follows:

- On a cache hit, call `sql.Tx.Stmt` with the cached DB-level `*sql.Stmt` and
  execute the transaction-specific wrapper.
- On a cache miss, call `sql.Tx.QueryContext` or `sql.Tx.ExecContext` directly.
  This path does not add a prepared statement to Sqlair's cache.

This means that repeated uses of the same `sqlair.Statement`, either within one
transaction or across successive transactions, continue to take the direct
raw-SQL path if that statement was never run through `sqlair.DB.Query`.

Juju reinforces this behaviour. Its domain database abstraction exposes a
transaction runner, and the normal SQLair runner begins a new transaction
before invoking domain state code. The long-lived state object therefore
retains Sqlair parsing work, but normal domain execution does not prime
Sqlair's underlying driver-statement cache.

### Prepared-Statement Exception

If a Sqlair statement is first executed outside a transaction through
`sqlair.DB.Query`, Sqlair prepares and caches a DB-level `*sql.Stmt`. A later
transaction can use it through `sql.Tx.Stmt`.

`database/sql` reuses its driver statement when it is already prepared on the
transaction's physical connection. Otherwise it prepares the same parent
statement on that connection and records it there. Committing or rolling back
closes the transaction-specific wrapper, but deliberately leaves a statement
owned by the parent DB-level `*sql.Stmt` available for later use. It therefore
is not strictly true that prepared plans cannot survive a transaction; the
normal Juju path simply does not create the cache entry needed to reach this
behaviour.

The lifetime still ends when Sqlair evicts or finalizes the statement, the
Sqlair DB is finalized, the driver connection closes, or Dqlite invalidates the
remote statement.

### Source Trail

- `github.com/canonical/go-dqlite/v3/driver/driver.go`: direct SQL operations,
  explicit prepare, statement-ID operations, and connection-bound statements.
- `github.com/canonical/sqlair/cache.go`: the DB/statement cache of
  `*sql.Stmt` values.
- `github.com/canonical/sqlair/sqlair.go`: distinct `DB.Query` and `TX.Query`
  cache-hit and cache-miss paths.
- Go `database/sql/sql.go`: `DB.PrepareContext`, `Tx.Stmt`, and per-connection
  statement handling.
- `domain/state.go`: Juju's parsed Sqlair statement cache.
- `internal/database/txn.go` and `internal/database/txn/transaction.go`:
  Juju's transaction-only domain execution path and retry semantics.

## Baseline Benchmark Consequences

The baseline must reproduce the transaction-only path rather than accidentally
warming Sqlair's DB-level statement cache during setup. At minimum, it should
separate these cases:

- first execution in a fresh transaction;
- repeated execution of one statement within a transaction;
- repeated execution across successive transactions;
- a deliberately DB-prepared statement reused by transactions, as a control;
- stable generated SQL versus argument-dependent generated SQL, such as
  expanded slice inputs.

The harness should count or otherwise observe:

- Dqlite raw-SQL `QUERY_SQL` and `EXEC_SQL` operations;
- Dqlite `PREPARE`, statement-ID query/exec, and `FINALIZE` operations;
- transaction begin, commit, rollback, and retry attempts;
- latency, throughput, allocations, and result correctness for each case.

The benchmark metadata must include Go version and `database/sql` pool settings
in addition to the Dqlite and Sqlair versions. Physical connection selection
determines whether a DB-level `sql.Stmt` can reuse an existing Dqlite prepared
statement or must prepare it on another connection.

## First Variant: Connection-Local Lazy Preparation in the Go Driver

This variant precedes the Dqlite concurrent-preparation change. It rewrites the
Go Dqlite driver while holding the Dqlite server/library, Sqlair, Go, Juju,
schema, and workload versions constant.

The intended driver behaviour is:

- `Conn.PrepareContext` performs no Dqlite protocol operation. It returns a
  lightweight `driver.Stmt` containing the SQL and its owning connection.
- Before any query or execution, the driver looks for a prepared Dqlite
  statement for that exact SQL in a cache owned by the physical connection.
- A miss prepares the statement on that connection and stores the remote
  statement handle. A hit executes the existing handle by statement ID.
- The cache survives transaction boundaries but never connection boundaries.

"Prepare is a no-op" therefore means no remote preparation at Go's
`PrepareContext` call. The method must still return an object satisfying the
`database/sql/driver.Stmt` contract.

### Does Sqlair Need to Change?

No Sqlair modification is required if the connection-local cache is applied to
every driver execution path.

This qualification matters because current Sqlair has two paths:

| Sqlair path | Standard-library call | Driver requirement |
| --- | --- | --- |
| DB-level Sqlair statement cache hit | `sql.Stmt.QueryContext` or `ExecContext` | The returned driver `Stmt` must lazily look up or prepare the connection-cached remote statement before execution. |
| Transaction cache miss, which is normal for Juju | `sql.Tx.QueryContext` or `ExecContext` | Dqlite `Conn.QueryContext` and `Conn.ExecContext` must use the same connection-local cache. |

An alternative driver-only implementation is for the direct connection methods
to return `driver.ErrSkip`, causing `database/sql` to use its
prepare-execute-close fallback. This also avoids a Sqlair change, but only if
closing the temporary logical driver statement does not discard the
connection-owned remote cache entry.

Changing only `Conn.PrepareContext` and the driver `Stmt` execution methods is
not sufficient while leaving Dqlite's direct `Conn.QueryContext` and
`Conn.ExecContext` operations unchanged. Normal Juju SQLair transactions would
continue to take those direct operations on every cache miss and would not use
the new connection-local prepared statement. In that incomplete driver design,
Sqlair would need a transaction-path change to force preparation. Keeping the
policy entirely inside the driver is the preferred boundary.

Sqlair's existing DB-level `*sql.Stmt` cache becomes partly redundant but is
not incompatible. It can continue to cache standard-library statement objects;
the driver decides whether those logical objects share an already-prepared
remote statement on each physical connection.

### Driver Contract Requirements

The connection must own the Dqlite statement handle. A logical `driver.Stmt`
returned to `database/sql` must not independently finalize a shared handle when
`Stmt.Close` is called. The driver needs ownership or reference-counting rules
that ensure:

- Sqlair eviction or finalization cannot invalidate a handle still used by
  another logical statement;
- transaction-specific `sql.Stmt` wrappers can close at commit or rollback
  without removing the connection cache entry;
- active rows keep the statement protected from eviction until `Rows.Close`;
- all cached handles are finalized when the physical connection closes;
- bounded eviction finalizes only idle handles.

Because remote preparation no longer happens before the first execution,
`Stmt.NumInput` cannot initially use Dqlite's returned parameter count. The
driver can return `-1`, as permitted by `database/sql/driver`, and let Dqlite
validate bindings during lazy preparation/execution. This changes the timing of
invalid-SQL and argument-count errors from prepare time to execution time and
must be covered by compatibility tests.

The cache also needs an explicit policy for:

- maximum entries or memory per physical connection;
- exact SQL cache keys, including Sqlair-generated slice and bulk variants;
- schema changes and stale-statement reprepare behaviour;
- connection loss and `driver.ErrBadConn` recovery;
- reset and binding cleanup between executions;
- whether transaction-control, DDL, and pragma statements are cached;
- cancellation during the lazy prepare operation.

### Driver Variant Comparison

Compare the current and rewritten Go drivers against the same current Dqlite
version:

| Go driver | Expected Dqlite operations for a repeated statement on one connection |
| --- | --- |
| Current | Normal Juju transaction misses use repeated raw-SQL `QUERY_SQL` or `EXEC_SQL` operations. |
| Connection-local lazy cache | The first execution sends `PREPARE` followed by statement-ID query/exec; later transactions use statement-ID operations without another prepare. |

Measure cold first execution separately from warm reuse. Report cache hits,
misses, evictions, remote prepares, finalizes, and the number of physical
connections that have seen each SQL string. Latency and throughput results
without those counts cannot show whether the intended driver path was used.

### Compatibility Tests

Before benchmarking, verify with current Sqlair that:

- a repeated `sqlair.TX.Query` prepares once per physical connection and reuses
  the remote statement across transactions;
- repeated use within one transaction also prepares only once;
- the Sqlair DB-cache-hit path and transaction-cache-miss path converge on the
  same driver cache entry;
- separate physical connections prepare independent remote statements;
- connection replacement produces a new prepare and no stale statement ID;
- cache eviction and logical `Stmt.Close` do not break active or later users;
- schema changes either retain a valid statement or trigger safe reprepare;
- invalid SQL and incorrect parameter counts return errors at execution;
- transaction retry, rollback, and cancellation preserve correctness.

## Second Dqlite Variant: Concurrent Preparation

Version: to be recorded when released.

Current Dqlite follows SQLite's concurrency model: reads can execute
concurrently, while write execution is serialized. In addition, the current
Dqlite threading model performs query preparation on its scheduling thread.
Read execution may move to parallel execution threads, but the preparation of
those reads first passes through this serial scheduling point.

The anticipated Dqlite version removes the scheduling-thread constraint from
query preparation. It is expected to follow the connection-local lazy driver,
so the release-to-release comparison must hold that rewritten driver constant.
The benchmark must measure the effect of the Dqlite change without attributing
SQLite's unchanged single-writer behaviour to it.

### Expected Effect

With the original driver, the normal Juju transaction path sends raw SQL for
each Sqlair cache miss, repeatedly exercising Dqlite query preparation. The
connection-local lazy driver removes most repeated preparation for a stable hot
query set. Measurements of the later Dqlite change must therefore include both
the representative warm-cache workload and a controlled preparation-pressure
workload.

The primary hypotheses are:

- With current Dqlite, cache-miss read throughput will stop scaling when serial
  preparation saturates the scheduling thread, even if read execution still
  has spare parallel capacity.
- With concurrent preparation, cache-miss reads will scale further before
  another resource becomes limiting, and queueing-related tail latency will
  improve.
- The difference between Dqlite versions will be smaller for explicitly
  prepared statements because their execution does not repeatedly pay the
  preparation cost.
- Write execution will remain single-writer limited. Concurrent preparation may
  reduce preparation delay around writes, but it must not be presented as
  parallel write execution.
- Mixed workloads may show a secondary benefit if read preparation no longer
  occupies the scheduling thread ahead of write-related work.

### Comparison Matrix

Hold the Go driver at the connection-local lazy version and hold the Juju
revision, schema, data set, Go version, Sqlair version, Dqlite topology,
connection-pool configuration, hardware, and query mix constant. Compare:

| Dqlite | Driver cache state | Purpose |
| --- | --- | --- |
| Current baseline | Cold or forced misses | Expose serialized preparation with the rewritten driver. |
| Concurrent-preparation version | Cold or forced misses | Measure the effect of removing preparation from the scheduling-thread bottleneck. |
| Current baseline | Representative warm cache | Establish product behaviour after the driver rewrite. |
| Concurrent-preparation version | Representative warm cache | Measure the residual product benefit once hot statements avoid preparation. |

The first comparison isolates the intended Dqlite change. The warm-cache
comparison states its practical effect after the earlier driver improvement.
Retain the original driver as an additional diagnostic control if it remains
compatible with both Dqlite versions; it naturally creates sustained
preparation pressure through raw-SQL operations.

### Workload Shape

Run separate read-only, write-only, and mixed phases. For each phase:

- sweep client concurrency from one worker through and beyond the available
  execution parallelism;
- include both repeated identical SQL and a representative distribution of
  distinct Juju queries;
- create preparation pressure through cold connections, controlled eviction,
  or a query working set larger than the connection cache, without changing the
  SQL distribution between Dqlite versions;
- stratify reads by preparation complexity and execution cost so that a cheap
  point lookup is not averaged together with a large join or view query;
- compare repeated statements within one transaction and across successive
  transactions;
- keep DDL and schema invalidation outside the timed interval;
- run a single Dqlite process diagnostic first, followed by the representative
  clustered topology, so network or topology effects do not hide the local
  scheduling-thread result.

Use both a concurrency saturation curve and a fixed offered-load test. The
first shows scaling and maximum throughput; the second exposes queue growth and
tail latency before saturation.

Mixed phases must use several read/write ratios derived from Juju's measured
query profile. The exact ratios remain a design input. A write-only phase is
still required to demonstrate the unchanged single-writer ceiling and to
identify any preparation-only improvement for writes.

### Measurements

In addition to the baseline metrics, collect where instrumentation permits:

- query preparation count, duration, and queueing time;
- scheduling-thread CPU utilization and runnable/blocked time;
- read execution concurrency;
- throughput scaling efficiency per added client and CPU;
- read and write p50, p95, p99, and maximum latency separately;
- writer latency while concurrent readers are preparing and executing;
- raw-SQL versus statement-ID protocol operation counts;
- CPU profiles or per-thread samples at the scaling knee.

If direct preparation queueing telemetry is not available in both versions,
the raw-SQL versus prepared-statement delta provides the attribution control.
End-to-end latency and throughput remain the primary product measurements.

### Interpretation Guardrails

- Compare latency distributions at equal offered load as well as each
  version's peak throughput. Comparing only at saturation can hide queueing.
- Do not infer preparation improvements from a workload dominated by execution
  time or write-lock contention.
- Do not warm the current Sqlair DB-level statement cache for the normal-path
  cases.
- Verify result correctness and transaction outcomes at every concurrency
  level; failed or retried operations must not inflate throughput.
- Record transaction retries separately from fresh operations.
- Treat any prepared-statement improvement as a possible unrelated Dqlite
  change until profiles or preparation telemetry support attribution.

## Third Variant: Sqlair-Owned Statement Cache and Streamlined API

This variant is based on the ideas illustrated by the last three commits of
the `manadart/juju` branch `dqlite-streamline-sqlair`:

- [`2cc2e9e`](https://github.com/manadart/juju/commit/2cc2e9e818a55781b10e1f37049d0c9e0aa3f10c)
  introduces generic `In`, `Out`, and `OutM` argument processors.
- [`3918e741`](https://github.com/manadart/juju/commit/3918e741923acf2be0579305a31c7fbe6b9b0b37)
  adds `StateBase.QueryRow`, `Query`, and `Exec` wrappers that derive Sqlair
  type samples, prepare implicitly, and use Juju's statement cache.
- [`93d511bf`](https://github.com/manadart/juju/commit/93d511bf5a7d8d2d652a87a75ef7a44b186a6797)
  demonstrates the syntax in several domain state packages.

The commits are an API prototype rather than the proposed implementation. The
new design moves Juju's parsed `sqlair.Statement` cache into Sqlair first, then
implements the streamlined syntax in Sqlair. Juju call sites should no longer
need to prepare statements explicitly or own the cache.

This parsed-statement cache is distinct from both Sqlair's existing cache of
DB-level `*sql.Stmt` objects and the proposed Dqlite driver cache of remote
statements per physical connection.

### Intended Boundary

The new Sqlair API should:

- accept SQLair query text plus input and output argument descriptors;
- derive the input values, result destinations, and type samples;
- retrieve or create the parsed and type-bound `sqlair.Statement`;
- execute it against the supplied `sqlair.TX`;
- provide single-row query, multi-row query, and execution/result forms.

The cache key must include the query text and sufficient concrete type identity
to distinguish every valid type binding. Juju's current `StateBase` cache is
keyed only by query text and explicitly documents the risk of returning the
wrong statement when identical text is prepared with different types. Moving
the cache is an opportunity to remove that limitation, not copy it into a
broader scope.

The intended change should not alter the standard-library or Dqlite protocol
path. If the new Sqlair cache also changes whether execution uses a DB-level
`*sql.Stmt`, that must be treated as a separate variant because it overlaps the
driver preparation experiments.

### Expected Effect

The end-to-end effect is expected to be negligible for warm, database-bound
Juju operations. The hypotheses are:

- Warm query latency and throughput remain effectively equivalent because SQL
  execution and Dqlite communication dominate the small API-layer change.
- Moving the cache into Sqlair may reduce duplicate parsing across short-lived
  or separate Juju state objects if the new cache has a broader lifetime.
- A broader shared cache may introduce lock contention or retain more entries.
- The `In`, `Out`, and `OutM` style may add slice construction, interface
  conversion, closures, or allocations on each call. These may be visible in a
  focused microbenchmark even when invisible end to end.
- Cold calls may change because cache-key construction and lookup are added,
  while parsing and type binding still dominate a true miss.
- Dqlite protocol operation counts remain identical between equivalent warm
  cases.

"Negligible" must be defined as an equivalence margin before results are
collected. A statistically non-significant result by itself does not establish
equivalence.

### Staged Comparison

Hold Dqlite, the Go driver, Go, schema, data, generated SQL, connection pool,
and transaction boundaries constant. Compare:

| Stage | Parsed-statement cache | Call syntax | Purpose |
| --- | --- | --- | --- |
| Current | Juju `StateBase` | Explicit `Prepare`, `TX.Query`, and `Get`/`GetAll`/`Run` | Baseline. |
| Cache relocation | Sqlair | Existing syntax through a compatibility path | Isolate cache ownership, key construction, lifetime, and contention. |
| Streamlined API | Sqlair | Sqlair-owned `In`/`Out`/`OutM` and query/exec helpers | Measure the incremental syntax and argument-processing cost. |

If a compatibility stage is impractical, retain focused cache-only
microbenchmarks so the combined end-to-end comparison can still be explained.
An optional no-cache control can quantify the parsing and type-binding work
that both cached designs avoid, but it is not a product candidate.

Run this comparison with one fixed Dqlite driver/server pair. It can be repeated
on the final stack later, but Dqlite or driver changes must not vary within a
Sqlair result set.

### Benchmark Layers

Use three layers so database latency does not hide API costs:

1. A cache and argument-processing microbenchmark with no database I/O,
   reporting nanoseconds and allocations per operation.
2. A transaction benchmark using a controlled driver or local database,
   verifying the complete Sqlair path with minimal network noise.
3. The representative Juju/Dqlite workload, reporting the practical
   end-to-end effect.

Each layer should include:

- cold misses and steady-state hot hits;
- one hot query and a representative multi-query working set;
- no-input, single-input, multiple-input, single-output, multi-output, and
  execution-result forms;
- map, struct, slice, and bulk arguments used by current Juju queries;
- a long-lived state object and churn through many state objects;
- single-threaded access and concurrent cache access;
- identical query text used with distinct type signatures;
- multiple controller/model database namespaces where relevant.

### Measurements

Collect:

- cache hits, misses, entries, evictions, and duplicate preparations;
- cache lookup, key-construction, parse, and type-binding duration;
- cache lock wait or contention profiles;
- nanoseconds, allocations, and allocated bytes per API call;
- end-to-end throughput and latency distributions;
- transaction duration;
- Dqlite raw-SQL, prepare, statement-ID execution, and finalize counts;
- retained cache memory after state-object and query churn.

Use repeated, interleaved runs and report effect sizes with confidence
intervals. For Go microbenchmarks, retain raw benchmark output and use a
statistical comparison tool such as `benchstat`. Record compiler, CPU,
frequency-governor, and process-placement details.

### Correctness and Attribution Guardrails

- Verify that cache entries never cross incompatible type bindings or database
  lifetimes.
- Preserve error behaviour for no rows, multiple rows, invalid arguments, and
  execution outcomes such as rows affected.
- Verify transaction retry safety for all input and output holders.
- Keep the final SQL sent to Dqlite byte-for-byte equivalent where possible.
  If wildcard expansion or another syntax feature changes generated SQL,
  measure that separately.
- Keep equivalent Go-side work inside or outside the transaction. Some example
  branch conversions move result transformation out of the transaction, which
  would otherwise confound transaction-duration results.
- Do not infer a cache or syntax improvement from changed Dqlite protocol
  counts; unexpected count changes indicate that another execution path also
  changed.
- Evaluate ergonomics and code reduction separately from runtime performance.

## Design Inputs

The plan needs to define:

- the representative schema and data populations;
- the query and transaction workload;
- the expected mix of cache hits, cache misses, and generated SQL variants;
- the Dqlite topology and runtime environment;
- the metrics and comparison method;
- setup, warm-up, run, and teardown phases;
- result metadata and reporting;
- controls for repeatability and variance.

## Open Questions

To be developed from further input.

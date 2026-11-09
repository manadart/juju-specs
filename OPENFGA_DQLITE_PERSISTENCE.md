# OpenFGA Persistence on Juju Dqlite

## Status

Recommendation for discussion. This note records the findings and design
options considered for embedding OpenFGA in Juju while persisting its data in
Juju's Dqlite-backed controller database.

## Executive recommendation

Implement a Juju-owned `storage.OpenFGADatastore` adapter over Juju's
`database.TxnRunner`.

The implementation should:

- use OpenFGA's public storage contract and preserve its datastore semantics;
- use an OpenFGA-shaped logical schema, with Juju-owned table names and schema
  migrations;
- execute all database operations through Juju's database accessor so that
  Dqlite retries, connection replacement, lifecycle, metrics and tracing
  continue to apply;
- port or reimplement the required query behaviour rather than copying the
  OpenFGA SQLite adapter wholesale; and
- pin an exact OpenFGA version and treat upgrades as storage-contract changes.

This is effectively a fourth option, although it is closest to option 2. The
key distinction is that Juju should adopt the OpenFGA storage *contract and
semantics*, not its concrete SQLite implementation.

The proposed boundary is:

```text
Embedded OpenFGA
        |
        v
Juju OpenFGADatastore adapter
        |
        v
database.TxnRunner
        |
        v
Dqlite
```

## Requirements and constraints

### Juju owns Dqlite transaction semantics

Juju deliberately exposes a transaction runner rather than a raw SQL database
handle. `database.TxnRunner` supplies SQLair and standard-library transaction
closures and exposes database lifecycle through `Dying`:

- [`core/database/runner.go`](core/database/runner.go)

The tracked database worker wraps each transaction in a retry loop, ties it to
the worker lifecycle, supports connection replacement, and records metrics:

- [`internal/worker/dbaccessor/tracker.go`](internal/worker/dbaccessor/tracker.go)

Juju recognises Dqlite-specific transient failures including locked or busy
databases, a transaction already in progress, bad connections and checkpoints
in progress:

- [`internal/database/drivererrors/errors.go`](internal/database/drivererrors/errors.go)
- [`internal/database/txn/transaction.go`](internal/database/txn/transaction.go)

These semantics require retrying the complete transaction closure. A lower
level SQL driver can translate an error, but cannot safely reconstruct and
replay arbitrary statements after a query or commit failure.

### OpenFGA provides a public datastore contract

OpenFGA embeds storage behind `storage.OpenFGADatastore`. The interface covers:

- relationship tuple reads and writes;
- paginated reads and tuple iterators;
- authorization models;
- stores;
- assertions;
- the tuple changelog;
- readiness and lifecycle; and
- limits such as maximum tuples per write.

The interface is the appropriate integration point:

- [OpenFGA storage contract](https://github.com/openfga/openfga/blob/main/pkg/storage/storage.go)

OpenFGA also provides a public storage conformance suite intended for custom
datastore implementations:

- [OpenFGA storage conformance tests](https://github.com/openfga/openfga/blob/main/pkg/storage/test/storage.go)
- [OpenFGA storage testing guidance](https://github.com/openfga/openfga/blob/main/CONTRIBUTING.md#1-storage-layer)

## Findings from the LXD implementation

LXD's embedded OpenFGA datastore is not a generic port of OpenFGA's SQLite
datastore to Dqlite. It is a specialised virtual datastore over LXD's existing
entity, authorization group and permission tables.

Notable characteristics are:

- tuples are synthesised from LXD entities and permissions rather than stored
  in the upstream OpenFGA tuple schema;
- some structural relationships are derived from entity URLs without querying
  the database;
- tuple writes are not implemented;
- paginated tuple reads are not implemented;
- store creation, deletion, retrieval and listing are not implemented;
- assertions and change reading are not implemented;
- the implementation uses one dummy store identifier;
- the authorization model is held in memory; and
- readiness always reports success.

See:

- [LXD OpenFGA datastore](https://github.com/canonical/lxd/blob/main/lxd/db/openfga/openfga.go)

LXD also disables OpenFGA's ListObjects pipeline optimisation because that
pipeline sends query shapes that the specialised datastore does not yet
support:

- [LXD embedded OpenFGA setup](https://github.com/canonical/lxd/blob/main/lxd/auth/drivers/openfga.go#L75-L81)

This design is suitable when all of the following are intentional:

- the application's native authorization schema is the source of truth;
- OpenFGA is used only as an evaluator;
- the authorization model is fixed and controlled by the application;
- OpenFGA's tuple, store, assertion and change APIs will not be exposed; and
- OpenFGA upgrades are tested against the precise query shapes used by the
  application.

It is not a drop-in general-purpose OpenFGA datastore. An implementation based
on observed query patterns can break when an OpenFGA optimisation changes those
patterns even if the public API continues to compile.

## Findings from the OpenFGA SQLite adapter

OpenFGA's SQLite datastore accepts an existing `*sql.DB` through `NewWithDB`,
but that does not make it compatible with Juju's database accessor boundary.

The SQLite adapter:

- stores and queries a raw `*sql.DB`;
- begins and commits its own transactions;
- closes the supplied database from `Close`;
- returns lazy SQL-backed tuple iterators for some reads;
- performs a limited SQLite busy retry in selected operations;
- recognises errors from `modernc.org/sqlite`; and
- maps SQLite constraint errors using modernc-specific error codes.

See:

- [OpenFGA SQLite datastore](https://github.com/openfga/openfga/blob/main/pkg/storage/sqlite/sqlite.go)
- [OpenFGA SQLite construction and ownership](https://github.com/openfga/openfga/blob/main/pkg/storage/sqlite/sqlite.go#L94-L136)
- [OpenFGA SQLite transaction handling](https://github.com/openfga/openfga/blob/main/pkg/storage/sqlite/sqlite.go#L365-L644)
- [OpenFGA SQLite error handling](https://github.com/openfga/openfga/blob/main/pkg/storage/sqlite/sqlite.go#L1251-L1309)

Passing a raw Dqlite-backed `*sql.DB` to this adapter would therefore bypass or
conflict with Juju's handling of:

- whole-transaction retries;
- Dqlite-specific error classification;
- connection replacement;
- database worker shutdown;
- transaction metrics and tracing; and
- ownership of the database handle.

Adapting errors alone would not solve these problems. In particular, retrying
only `BEGIN`, `COMMIT`, or an individual statement is not equivalent to replaying
the entire transaction.

## Options considered

### Option 1: Use an LXD-style application persistence schema

Implement OpenFGA's storage interface as a projection over Juju's native
authorization and resource tables.

#### Advantages

- Juju data remains the only source of truth.
- Structural relationships can be derived directly from Juju resources.
- No duplicated resource hierarchy tuples are required.
- Resource and authorization changes can be made atomically in existing Juju
  state transactions.
- The schema can be designed around Juju's authorization workflows.

#### Disadvantages

- The adapter becomes coupled to one Juju authorization model.
- It may only support OpenFGA query shapes observed today.
- OpenFGA optimisations or new resolver behaviour can require adapter changes.
- Standard tuple, model, store, assertion and changelog APIs may be incomplete
  or unavailable.
- OpenFGA's general storage conformance suite may not pass.
- A specialised adapter requires strong functional tests for Check,
  ListObjects and ListUsers on every dependency upgrade.

#### Recommendation

Choose this only if Juju explicitly decides that its own authorization schema
is authoritative and OpenFGA is solely an embedded evaluator for a fixed Juju
model. Do not select it merely because LXD also uses Dqlite.

### Option 2: Copy OpenFGA's SQLite schema and query implementation

Copy the upstream schema and SQLite datastore into Juju, then replace raw SQL
database access with Juju's database accessor.

#### Advantages

- Starts from behaviour already exercised by OpenFGA.
- Retains conventional OpenFGA tuple, model, store and changelog semantics.
- Makes comparison against the upstream SQLite backend straightforward.

#### Disadvantages

- A literal copy will drift from upstream bug fixes and semantic changes.
- Upstream migrations do not fit Juju's controller schema lifecycle.
- Upstream SQL contains assumptions about its SQLite driver and SQLite
  version.
- Upstream lazy iterators do not fit transaction closures that may be retried.
- Copying implementation details creates a larger maintenance and security
  review surface than implementing the public contract.

#### Recommendation

Use the upstream schema, indexes, constraints, algorithms and tests as design
references. Do not maintain an unstructured source copy. Reimplement the
contract behind Juju's transaction boundary, document the upstream version
used for each area, and use differential tests to detect divergence.

### Option 3: Adapt the driver or fork OpenFGA

Make the OpenFGA SQLite adapter accept Dqlite errors, expose a raw database
connection, or maintain a Juju fork of OpenFGA.

#### Advantages

- Potentially reuses more upstream query code.
- A sufficiently general upstream refactor could benefit other embedded
  deployments.

#### Disadvantages

- A driver cannot replay an arbitrary transaction after a mid-transaction
  failure.
- Returning `driver.ErrBadConn` is not enough to resolve ambiguous commits or
  repeat application logic safely.
- Exposing a raw database handle weakens Juju's existing accessor boundary.
- OpenFGA's adapter would still own transaction and connection lifecycle.
- A fork would need frequent rebasing for storage fixes and interface changes.
- Juju-specific Dqlite logic would be unlikely to belong in OpenFGA's generic
  SQLite implementation.

#### Recommendation

Do not fork OpenFGA or solve this solely at the SQL driver layer.

It is worth discussing an upstream, database-neutral transaction-runner and
error-mapping extension with OpenFGA. Such a change would need to support whole
operation retries and address lazy iterators; a simple injected error
classifier would not be sufficient. Juju's initial implementation should not
depend on that work being accepted.

### Option 4: Juju-owned OpenFGADatastore with OpenFGA semantics

Implement the OpenFGA storage interface in Juju, backed by a Juju-owned schema
and `database.TxnRunner`.

#### Advantages

- Preserves Juju's Dqlite correctness and lifecycle boundary.
- Preserves general OpenFGA storage semantics.
- Can pass OpenFGA's storage conformance suite.
- Does not require an OpenFGA fork.
- Makes error translation explicit at the correct layer.
- Allows Juju to control schema upgrades with the rest of the controller
  database.

#### Disadvantages

- Juju owns a non-trivial datastore implementation.
- Query and schema behaviour must be reviewed on every OpenFGA upgrade.
- Materialising tuple iterators can use more memory than streaming rows.
- Correct retry behaviour, pagination and changelog ordering require careful
  implementation and tests.

#### Recommendation

This is the preferred embedded and self-contained approach.

## Recommended design details

### Schema ownership

Use Juju controller schema patches rather than OpenFGA's Goose migrations.
Authorization normally spans controller and model resources, so the controller
database is the likely location. Avoid splitting one authorization graph across
controller and model databases because Juju cannot provide atomicity across
those databases.

Use prefixed names to avoid collisions and make ownership clear, for example:

- `openfga_store`;
- `openfga_authorization_model`;
- `openfga_tuple`;
- `openfga_changelog`; and
- `openfga_assertion` if assertion APIs are supported.

The initial schema should closely follow the logical fields, uniqueness
constraints and indexes in the pinned OpenFGA SQLite schema. It need not retain
identical table names or OpenFGA's migration metadata.

Persist a stable store identifier if Juju uses a single store per controller.
Do not generate a process-local dummy identifier unless store persistence and
all store-related APIs are intentionally unsupported.

### Database operations

Execute every database interaction through `TxnRunner.StdTxn` or
`TxnRunner.Txn`.

For reads that must return an OpenFGA `TupleIterator`:

1. query and scan all selected rows inside the transaction closure;
2. close the SQL rows inside that closure;
3. allow the transaction to commit successfully; and
4. return a static or in-memory iterator over the materialised tuples.

No SQL rows, statements, transactions or database handles should escape the
retryable transaction closure. Otherwise a retry could occur after partial
results have already been observed by OpenFGA.

Pagination should use deterministic, indexed keys compatible with OpenFGA's
continuation-token contract. Changelog ordering and pagination should preserve
the ULID semantics expected by `ReadChanges`.

### Error handling

Do not convert a Dqlite error into a generic OpenFGA error before Juju's
transaction runner has had an opportunity to classify and retry it.

The intended order is:

1. return raw or wrapping-preserving database errors from the transaction
   closure;
2. let the database accessor retry recognised transient failures;
3. after the operation succeeds or retries are exhausted, translate terminal
   storage outcomes to OpenFGA errors such as `ErrNotFound`, `ErrCollision`,
   invalid write input or transactional write failure; and
4. preserve the original cause when returning infrastructure failures.

Constraint translation must recognise Juju's Dqlite and test SQLite driver
errors, rather than relying on `modernc.org/sqlite` error types.

### Retry-safe writes

The complete write operation must be safe when the transaction closure is
executed more than once.

This includes:

- resetting mutable result slices or maps where a previous attempt could leave
  partial state;
- using stable operation, tuple and changelog identifiers across attempts;
- writing tuple changes and their changelog entries in one transaction;
- handling an ambiguous commit without producing duplicate changelog entries;
- distinguishing a replay of the same operation from an independent duplicate
  request where required by OpenFGA semantics; and
- ensuring timestamps and continuation keys remain consistent on replay.

A persisted operation identifier or equivalent replay marker may be needed to
prove that an exact existing changelog record came from a previous attempt of
the same operation.

### Lifecycle and readiness

The OpenFGA adapter must not close the underlying Juju database. Its `Close`
method should release only resources owned by the adapter and otherwise be a
no-op.

`IsReady` should perform an accessor-backed health or schema check and respect
context cancellation and `TxnRunner.Dying`. It should not merely return true.

### Caching and multiple controller agents

Each controller agent can host its own embedded OpenFGA server while sharing
the replicated Dqlite database. Process-local cross-request authorization
caches can therefore become stale when another controller writes tuples or a
new model.

Initially disable shared cross-request Check caches, or use a configuration
whose correctness does not depend on immediate invalidation. If such caching
is enabled later, invalidate it from a cluster-visible tuple/model change
stream. Per-request caches remain useful and do not have the same cross-node
staleness problem.

## Atomicity with Juju domain changes

Embedding OpenFGA does not automatically make a Juju domain mutation and an
OpenFGA tuple write atomic. The OpenFGA `Write` contract owns its transaction
and does not accept a transaction opened by the host application. This is a
known limitation for embedded deployments:

- [OpenFGA issue: host transaction participation](https://github.com/openfga/openfga/issues/3302)

For system-managed relationships, choose one of these designs deliberately:

1. **Derive structural relationships from Juju data.** This avoids duplicate
   resource and hierarchy tuples and resembles the useful part of LXD's
   projection approach.
2. **Write resource and tuple state through a shared Juju state transaction.**
   Use common transaction-level helpers rather than calling the embedded
   OpenFGA server from inside a domain state transaction.
3. **Use an outbox and reconciler.** Persist the intended authorization change
   with the domain mutation, then apply it to tuple tables asynchronously. This
   must fail closed while convergence is incomplete.

The first two choices give stronger immediate consistency. An outbox is
appropriate only where temporary denial is acceptable and reconciliation is
observable and reliable. A failure must never temporarily grant access.

## Validation strategy

### Storage conformance

Run OpenFGA's complete public storage test suite against the Juju adapter and a
real Dqlite test cluster. A compile-time assertion that the adapter implements
`storage.OpenFGADatastore` is necessary but not sufficient.

### Differential tests

For supported operations, apply identical inputs to:

- the pinned upstream OpenFGA SQLite datastore; and
- the Juju Dqlite datastore.

Compare API-visible results and errors rather than SQL text or physical table
layout. Cover:

- exact and partial tuple filters;
- usersets, wildcards and tuple conditions;
- pagination in both directions where supported;
- duplicate writes and missing deletes with all write options;
- authorization model ordering and lookup;
- stores and assertions;
- changelog ordering, horizons and continuation tokens; and
- concurrent conflicting writes.

### Dqlite failure tests

Inject retryable errors at:

- transaction begin;
- the first and later reads;
- tuple deletion;
- tuple insertion;
- changelog insertion; and
- commit.

Verify that callers observe one logical operation, with no missing tuple,
duplicate tuple, missing changelog entry or duplicate changelog entry.

Run multi-node leader-loss and connection-replacement tests. Because embedded
OpenFGA and the database accessor use concurrency, run relevant tests with
`-race` and stress the failover and cancellation cases.

### OpenFGA functional tests

Storage conformance alone does not establish resolver compatibility. Exercise
the Juju authorization model through:

- Check and BatchCheck;
- ListObjects;
- ListUsers;
- contextual tuples;
- conditions;
- recursive usersets and tuple-to-userset relationships; and
- resolver pipeline optimisations enabled in production.

These tests are particularly important for any specialised or hybrid datastore
that derives some tuples from Juju resources.

### Upgrade gate

For every OpenFGA dependency update:

1. review changes to `storage.OpenFGADatastore` and its subinterfaces;
2. review upstream SQL schema and migrations;
3. review SQLite and SQL-common storage fixes;
4. run storage conformance tests;
5. run differential tests;
6. run Juju model functional tests; and
7. verify that new OpenFGA query optimisations do not require unsupported
   datastore query shapes.

## Alternative deployment

If Juju can accept an additional external service and database dependency, the
lowest datastore maintenance option is to deploy an independent OpenFGA server
with an upstream-supported PostgreSQL or MySQL backend. That avoids maintaining
a custom datastore, but introduces new installation, upgrade, high-availability,
backup and failure-mode requirements and weakens Juju's self-contained
controller deployment model.

For an embedded, self-contained controller using Dqlite, the Juju-owned
`OpenFGADatastore` remains the preferred option.

## Final ranking

1. **Juju-owned `OpenFGADatastore` using `TxnRunner` and an OpenFGA-shaped,
   Juju-migrated schema.**
2. **External OpenFGA with an upstream-supported database**, if the operational
   dependency is acceptable.
3. **LXD-style projection**, only if Juju's native authorization schema is
   deliberately the source of truth and the supported OpenFGA surface is
   intentionally narrow.
4. **Driver-only adaptation or an OpenFGA fork.** Avoid as the primary design.


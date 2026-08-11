# Cross-model macaroon discharge caching

## Status

This note records the investigation of cross-model secret latency and
macaroon caching, and proposes a controller-scoped cache worker as the
long-term fix.

The source inspected for the 4.0 work was:

- branch: `4.0-macaroon-cache`
- 4.0 base: `2914d4a9e2124d5988135ed87d1b502adbb3f6c5`
- current change: `73ea3e47e9698b2986ceecc3496d9b00a5f4d6f0`
- current change is the cherry-pick of
  [PR 22973](https://github.com/juju/juju/pull/22973)

The current main branch was also inspected at
`06aa0598f6182460e6571ca1d6c920b373ffab35`. It has the same ineffective
cache ownership and application UUID cache key described below.

## Original incorrect behavior

Cross-model offer permission macaroons expire after three minutes. When a
secret read presents an expired relation macaroon, the offering controller
returns `CodeDischargeRequired` with a new macaroon to discharge.

Before PR 22973, `GetRemoteSecretContentInfo` handled this response from the
`retry.Call` `IsFatalError` callback. The callback updated the request with the
discharged macaroon, but the retry framework then waited its normal
three-second retry delay before making the next API call.

This caused secret reads around credential expiry to take approximately three
seconds. The latency was not proportional to the number of secrets. The
reported workload repeatedly read all previously created secrets and happened
to cross the three-minute credential expiry boundary.

PR 22973 moved discharge handling into the retry function. A
discharge-required response is now discharged and retried immediately within
the same retry attempt. The ordinary three-second delay remains available for
transient remote-controller errors.

This fixes the visible three-second pause. It does not make the discharged
macaroon reusable by later secret reads.

## Remaining secret cache defect

`api/controller/crossmodelsecrets.Client` contains a `MacaroonCache` and
`NewClient` creates a fresh cache for every client. There is already a
`NewClientWithCache` constructor which permits a cache to be injected.

The production secrets path does not use that constructor. The
`SecretsManager` facade does the following for every remote secret read:

1. Look up connection information for the secret's source model.
2. Open a new external controller connection.
3. Construct a new `crossmodelsecrets.Client`, including a new cache.
4. Fetch the relation macaroon and request the secret content.
5. If required, discharge and insert the result into the new client's cache.
6. Retry successfully with the discharged macaroon.
7. Close the client and external connection when the API call returns.

The cache is therefore discarded immediately after the call which populated
it. The next secret read creates an empty cache, presents the original expired
relation macaroon, and performs another discharge. PR 22973 makes each of
these repeated discharges fast, but does not prevent them.

Retaining the external client or connection is not the right correction.
Secret reads can target different remote models, and cross-controller
connections need deterministic closure and reconnection handling. The cache
and connection have different required lifetimes.

## Incorrect current cache key

The secrets client currently looks up and stores discharged macaroons using
the consuming application UUID.

That is not the authorization scope. The offering controller extracts the
offer and relation identity from the supplied macaroon and checks a
relation-specific `relate` operation. One consuming application can have
multiple cross-model relations, potentially to multiple source models.

If the existing cache is given a longer lifetime without changing the key, a
discharged macaroon for one relation can replace the entry for another
relation belonging to the same application. At best this creates additional
discharge or permission failures; it must not be the basis of a shared cache.

The cross-model relations client also uses untyped strings such as relation
tokens and offer UUIDs as cache keys. These work only within the assumptions
of the individual client. Promoting that map to controller scope would create
additional namespaces and operation classes in the same key space.

## Other production clients inspected

The production constructors using the same cache-bearing cross-model clients
are:

- `apiserver/facades/agent/secretsmanager/register.go` creates a
  `crossmodelsecrets.Client` for a remote secret API call.
- `internal/worker/remoterelationconsumer/manifold.go` creates a
  `crossmodelrelations.Client` from the remote relation caller.
- `internal/worker/firewaller/shim.go` creates a
  `crossmodelrelations.Client` around a newly opened external controller
  connection.

The exact client lifetime differs between these users, but each constructor
currently creates its own cache. Consequently, a valid discharge learned by
one client cannot be reused by another client, even when both present the same
underlying relation credential.

## Options considered

### Client-owned cache

This is sufficient only for a client which is retained by a long-running
worker. It does not cover clients created inside API facade methods or other
short-lived connection paths.

### SecretsManager facade-owned cache

The API root caches a `SecretsManagerAPI` facade for the lifetime of an
inbound agent API connection. A cache captured by that facade and supplied to
`NewClientWithCache` would prevent repeated discharge on that connection.

This is a useful minimal fix, but it still creates separate caches for every
agent connection and every other worker. It also loses all entries when an
agent reconnects. It cannot reuse a discharge already obtained by a remote
relation worker using the same credential.

### Persistent or distributed cache

Discharged macaroons are short-lived bearer credentials. Persisting them in
model state would add migration, revocation, security, and HA consistency
semantics for little benefit. This is not proposed.

### Controller-scoped cache worker

This is the recommended option. It provides the widest safe cache reuse while
remaining process-local and ephemeral.

## Proposed architecture

Create one macaroon cache worker in the controller agent dependency engine.
The API server and model-worker manager depend on it:

```text
macaroon-cache
|-- API server
|   `-- SecretsManager-created cross-model secrets clients
`-- model-worker manager
    `-- per-model dependency engine
        |-- remote-relation consumer clients
        `-- firewaller cross-model clients
```

The API server and model-worker manager are sibling controller-agent
manifolds. The model-worker manager already passes shared controller
dependencies into each nested model engine, so the cache can follow the same
path.

For the API server, pass the cache through the API server configuration into
`sharedServerContext`. Expose it to facades through a narrow facade-context
capability. `SecretsManager` can then inject it into every short-lived client
with `crossmodelsecrets.NewClientWithCache`. External connections remain
request-scoped and are still closed after use.

The reusable cache API should live in a neutral macaroon package usable by
API clients and the API server, rather than under
`api/controller/crossmodelrelations`. The lifecycle wrapper belongs under
`internal/worker`.

## Cache identity

Use a structured key derived from the credential, not from the requested
application, relation method, or secret:

```text
{
    target model UUID,
    SHA-256(original primary macaroon ID),
}
```

The Bakery macaroon ID contains a random nonce and the authorized operation.
Hashing avoids retaining or logging the raw identifier. The target model UUID
provides an explicit remote-authority namespace.

The key must be calculated from the original credential supplied by the
caller, before the request argument is replaced with a cached discharged
slice. Cache entries remain associated with that original key after a
successful discharge.

No method or facade name should be part of the key. A discharged credential
may legitimately authorize several cross-model relation methods. Omitting the
method permits safe reuse between relation publication, relation watchers,
firewall operations, consumed-secret watchers, and remote secret reads when
they use the same original credential.

A replacement relation macaroon has a new macaroon ID and therefore cannot
accidentally reuse an entry belonging to the old credential.

## Concurrency and lifecycle

A process-wide cache will receive concurrent calls. A mutex around the map is
necessary but not sufficient: two callers can both observe a miss and perform
the same discharge.

The cache should provide a context-aware, per-key acquisition operation, for
example `GetOrDischarge`. It should recheck the cache after becoming the
single acquirer for a key, perform one supplied discharge function, publish
the result, and allow other callers to reuse it. Waiting callers must be able
to return when their contexts are cancelled.

The cache worker should use the normal Juju worker lifecycle rather than the
current finalizer-managed goroutine:

- deterministic `Kill` and `Wait` behavior;
- periodic expiry cleanup;
- lazy expiry checking on lookup;
- no normal runtime failure path which would unnecessarily restart the API
  server and all model workers.

Only macaroons with a known expiry should be cached. Alternatively, a strict
maximum cache TTL must be applied. A size bound should also prevent a
controller with many historical relations from retaining unlimited entries.

If a cached credential is rejected as stale, the client should evict it and
retry once with the original credential. This must be bounded so that genuine
permission failures do not create an authentication loop.

## Scope and HA behavior

This design maximizes safe reuse within one controller process. Each
controller node in an HA controller has its own cache. API traffic reaching a
different controller node may cause one discharge on that node.

That boundary is intentional. A cluster-wide cache would require persistence
or distributed coordination for short-lived bearer credentials and is not
justified by the three-minute lifetime.

## Incremental implementation

The work can remain focused on secrets first while establishing the common
foundation:

1. Introduce the neutral cache key and cache interface.
2. Implement the lifecycle-managed cache worker with bounded storage,
   expiration, and per-key acquisition coalescing.
3. Add the cache as a controller-agent dependency of the API server.
4. Pass it through `sharedServerContext` to `SecretsManager`.
5. Change the secrets client to derive its key from the original relation
   macaroon and inject the shared cache into short-lived clients.
6. In subsequent work, pass the same dependency through the model-worker
   manager and convert remote-relation consumer and firewaller clients.
7. Remove the token-specific cache keys as each client is converted.

## Required verification

The implementation should demonstrate:

- two separate secret client instances using the same original credential
  cause only one discharge;
- different relation credentials for the same application do not collide;
- replacement credentials do not reuse old entries;
- expiry causes one new discharge and concurrent misses are coalesced;
- cancelled waiters return without cancelling the shared acquisition for
  unrelated callers;
- all short-lived external connections are still closed;
- cache worker shutdown is deterministic;
- worker and cache concurrency tests pass with `-race` and stress.

When the other clients are converted, add a test showing that a discharge
populated through a cross-model relation client can be reused by the secrets
client for the same credential and target model.

# Reverse-direction cross-model secret sharing

Status: exploratory design note

This document records the problem described in
[juju/juju#21248](https://github.com/juju/juju/issues/21248), the proposals made
in that discussion, and the design constraints found while examining the Juju
4.0 implementation. It is an artefact for continuing the design later. It is
not an accepted implementation design, and the names of APIs, domain methods,
and tables are illustrative.

## Problem statement

Cross-model relation (CMR) secrets currently work when an application on the
offering side owns a secret and grants it to an application on the consuming
side. They do not work in the reverse direction.

The reported example has the following shape:

- The offering model contains `data-integrator`, which offers its requirer
  endpoint.
- The consuming model contains `postgresql-k8s`, which provides the endpoint
  and relates to the consumed offer.
- `postgresql-k8s` creates a secret and grants it to the remote
  `data-integrator` application.
- Juju rejects the grant because sharing a consumer-side secret across a CMR is
  not supported.

The same direction is required when a provider supplies credentials to an
offered requirer, and for protocols such as mutual TLS where the requirer must
share certificate material with the provider.

For this document:

- **Owner side** means the consuming model and controller on which the secret
  is created. This is `postgresql-k8s` in the example.
- **Reader side** means the offering model and controller containing the unit
  that needs to read the secret. This is `data-integrator` in the example.
- **Forward direction** means the already-supported offerer-to-consumer secret
  flow.
- **Reverse direction** means the proposed consumer-to-offerer secret flow.

These terms describe the secret flow, not which endpoint is a relation
provider or requirer.

## Important network asymmetry

The consuming controller establishes and maintains the CMR connection to the
offering controller. Existing deployments also normally arrange for consuming
workloads to reach offered workloads. Neither fact guarantees that a unit in
the offering model can reach the consuming controller, its Kubernetes API, or
its Vault service.

This asymmetry is the main distinction between the two content-delivery
proposals:

- A mailbox can use the existing consuming-controller-to-offering-controller
  connection and does not require a new inbound route to the consuming side.
- A direct backend read is simpler, but requires the reader unit to reach a
  content endpoint or the source external backend on the owner side.

The eventual choice is therefore partly a product-level connectivity contract,
not merely an implementation preference.

## Design goals

The design should:

- allow a consuming-side application to grant a secret to the offered
  application for the life of their relation;
- preserve relation-scoped, least-privilege access checks;
- support latest, explicit-revision, refresh, and peek semantics consistently
  with local and forward-direction secrets;
- propagate revision changes so that reader units receive `secret-changed`;
- propagate revocation, secret deletion, relation teardown, and expiry without
  leaving durable access behind;
- retain old revisions while a remote reader legitimately tracks them;
- converge after controller, worker, or unit-agent restart;
- behave correctly with controller high availability, duplicate delivery, and
  reordered work;
- keep secret writes, rotation, and deletion under the owner model's existing
  domain and hook-commit workflows; and
- negotiate the feature explicitly across the two controllers and unit agent.

The design should not:

- make a reverse grant a model-wide or controller-wide entitlement;
- put bearer credentials or secret content in relation data;
- give a remote unit a general controller API credential;
- allow the reader to write directly to an internal Juju secret backend; or
- weaken or replace the existing forward-direction path merely to make both
  directions superficially identical.

## Existing implementation shape

Juju separates secret metadata and revision state from secret content.
`secret_metadata`, `secret_revision`, `secret_content`, and
`secret_value_ref` describe locally owned secrets. `secret_reference` describes
a secret owned by another model. `secret_unit_consumer` records the revision a
local unit is consuming, while `secret_remote_unit_consumer` records revisions
tracked by units across a CMR.

For unit-side reads, `internal/secrets/backend.go` first obtains content
information. Inline content is returned immediately. A `ValueRef` causes the
unit to construct the named backend from restricted configuration and retrieve
the content directly.

The Kubernetes and Vault providers can issue restricted configuration and
short-lived access tokens. The internal Juju provider is currently a dummy
backend: it does not issue tokens, its restricted configuration contains no
usable content endpoint, and its backend cannot retrieve content. The current
cross-model facade therefore handles internal content separately.

The current CMR domain, schema comments, facade calls, and workers assume that
the remote secret owner is on the offering side. Reverse sharing first fails
in the grant path, before content delivery is selected. A new content path by
itself therefore cannot solve the issue.

## Common control plane

Both content-delivery proposals need the same backend-neutral control-plane
state on the reader side. That state should express an access entitlement and
the current secret lifecycle. It should not itself be a bearer capability.

### State propagated from the owner side

The owner controller needs to reconcile the following information to the
reader controller before either data plane can work:

| Information | Purpose | Persistence on the reader side |
| --- | --- | --- |
| Full secret URI | Identifies the source model UUID and secret UUID without relying on a local alias. | `secret` and `secret_reference`. |
| Owner application identity | Associates the secret with the synthetic representation of the consuming-side owner application. | Existing CMR application mapping plus `secret_reference.owner_application_uuid`. |
| Local grantee application | Identifies the offered application whose units may read the secret. | A new relation-scoped remote grant record. |
| Offer and connection identity | Binds the update to one authenticated offer consumption. | Existing `offer` and `offer_connection` rows. |
| Relation identity on both sides | Maps the owner-side relation to the corresponding reader-side relation and scopes cleanup. | Existing CMR mapping plus the new remote grant record. |
| Role | Records that the remote application has `view` access. | The new remote grant record, using `secret_role`. |
| Latest numeric revision | Drives `secret-changed` and latest-revision reads. | `secret_reference.latest_revision`. |
| Grant and secret lifecycle | Represents grant, revoke, deletion, and relation removal. | Presence or absence in a reconciled snapshot; optional explicit tombstones only if snapshots are not used. |
| Ordering generation | Rejects stale snapshots and protects empty revoke snapshots from reordering. | New per-relation sync state, unless an existing durable monotonic sequence can be reused. |
| Protocol capabilities | States which reverse-secret control and content protocols both sides support. | New sync state or an extension of the CMR connection state. |

The receiver must derive the local offered application, local relation, and
remote application mapping from the authenticated CMR context. It must not
accept arbitrary application or relation UUIDs supplied by the caller.

### Snapshot reconciliation

The preferred control contract is an idempotent full snapshot per relation,
conceptually:

```text
ApplyRemoteSecretSnapshot(
    relation,
    generation,
    protocolVersion,
    []RemoteSecretGrant{
        URI,
        ownerApplicationToken,
        role,
        latestRevision,
    },
)
```

The reader-side domain service resolves all local identities and applies one
generation transactionally. Applying a snapshot should:

1. reject a generation older than or equal to the last applied generation;
2. upsert its secret references and remote grants;
3. update changed latest revisions;
4. remove grants absent from the new snapshot;
5. remove an unreferenced remote secret only after its final relation grant and
   local consumer state are gone; and
6. update the relation's last-applied generation in the same transaction.

A change-log watcher should only notify a worker that state may have changed.
The worker then re-queries the full authoritative state. This follows Juju's
watcher model and makes duplicate or coalesced notifications safe.

Full snapshots avoid a durable incremental outbox for individual grant,
revision, and revoke events. They do not eliminate ordering state. The owner
side needs a durable monotonic generation, or an equivalent source epoch and
sequence, and the reader side must store the last accepted value. If no
existing change-log sequence has the required lifetime and scope, sync-state
storage is needed on both sides.

### Authoritative access

The owner-side `secret_permission` remains the authority for a grant. The
reader-side mirror exists so that Juju can:

- authorize the local unit before starting a content read;
- route the request to the correct remote relation;
- filter revision-change notifications;
- remove access promptly when a reconciled snapshot revokes it; and
- clean up state when the relation departs.

The owner side must re-check the live secret, relation, grant, requested
revision, and reader identity when serving each content request. A stale mirror
must never be sufficient to read content.

## Schema implications

The exact schema should be designed with the domain operations, migrations,
and watcher change-log entries. The following identifies where current tables
can be reused and where new state appears necessary.

### Existing tables that can be reused

- **`application_remote_consumer`**: Represents the consuming-side application
  inside the offering model. In reverse sharing this synthetic application can
  represent the secret owner.
- **`offer_connection`**: Associates the remote application and reader-side
  relation with an offer.
- **`relation`**: Supplies the local relation lifecycle and scope.
- **`secret` and `secret_reference`**: Represent a remote secret and its latest
  revision. The code and comments need to stop assuming that the remote owner
  is necessarily on the offering side.
- **`secret_unit_consumer`**: Records the local reader unit, source model UUID,
  label, and currently tracked revision. Its existing latest-revision watcher
  model can drive `secret-changed` from `secret_reference` updates.
- **`secret_permission`**: Remains the authoritative grant on the owner side.
  Owner-side resolution must permit the remote-offerer orientation.
- **`secret_remote_unit_consumer`**: Can continue to record the revision used by
  the remote reader on the owner side. Direction-specific queries and comments
  need generalizing. The uniqueness of anonymized remote unit names across
  relations should be verified; otherwise relation identity must be added.

### New reader-side remote grant table

`secret_permission` cannot safely be reused for mirrored remote grants. Its
secret foreign key points to `secret_metadata`, which deliberately exists only
for locally owned secrets. A remote secret has a `secret_reference` instead.
Removing that ownership distinction from `secret_permission` would blur the
authority boundary.

A dedicated table therefore appears necessary. An illustrative shape is:

```sql
CREATE TABLE secret_remote_grant (
    secret_id TEXT NOT NULL,
    relation_uuid TEXT NOT NULL,
    grantee_application_uuid TEXT NOT NULL,
    role_id INT NOT NULL,
    generation INT NOT NULL,
    updated_at DATETIME NOT NULL,
    PRIMARY KEY (
        secret_id,
        relation_uuid,
        grantee_application_uuid
    ),
    FOREIGN KEY (secret_id) REFERENCES secret (id),
    FOREIGN KEY (relation_uuid) REFERENCES relation (uuid),
    FOREIGN KEY (grantee_application_uuid) REFERENCES application (uuid),
    FOREIGN KEY (role_id) REFERENCES secret_role (id)
);
```

This table is a relation-scoped authorization mirror, not an alternative
authority for content. It needs change-log triggers or equivalent watcher
support. Cascading behavior must be explicit so that relation teardown cannot
leave a grant behind.

### New synchronization state

The receiver needs the last applied generation and negotiated protocol version.
An illustrative table is:

```sql
CREATE TABLE secret_remote_sync_state (
    relation_uuid TEXT NOT NULL PRIMARY KEY,
    applied_generation INT NOT NULL,
    protocol_version INT NOT NULL,
    reconciled_at DATETIME NOT NULL,
    FOREIGN KEY (relation_uuid) REFERENCES relation (uuid)
);
```

The same schema could carry an `outbound_generation` for a model acting as the
owner-side sender, or sender state could use a separate table. Extending
`offer_connection` is possible, but a dedicated table keeps secret protocol
state out of the general CMR entity. The key invariant is that the sender's
generation survives restart and the receiver can reject delayed work from an
older generation.

### Reciprocal tracked-revision propagation

Control state also has to flow from the reader side back to the owner side.
When a reader unit first reads, refreshes, or changes revision,
`secret_unit_consumer.current_revision` changes in the reader model. The owner
must learn that value and update `secret_remote_unit_consumer`; otherwise it can
prune a revision that remains in use remotely.

The consuming controller already owns the outbound CMR connection. The
offering controller can expose a relation-scoped watcher and snapshot of
changed reader revisions on an appropriate facade. The consuming-side worker
uses the existing connection to watch, fetch, and reconcile that snapshot into
owner-side state.

No new consumer-tracking table appears necessary if
`secret_remote_unit_consumer` can safely identify the reader unit. A relation
UUID or stronger opaque reader token may need to be added if its current
`(secret_id, unit_name)` identity is not collision-safe for reverse sharing.

## Data plane A: encrypted mailbox

The issue's final comment proposes a pull-based mailbox hosted by the reader
controller. The owner controller services the mailbox using its existing
outbound connection to that controller.

### Proposed flow

1. The reader unit creates an ephemeral encryption key pair.
2. The unit asks its controller to create a request containing the secret URI,
   relation identity, requested revision and read flags, and public key.
3. A consuming-side worker watches requests for the authenticated relation over
   the existing CMR connection.
4. The owner controller claims the request, revalidates the relation and grant,
   obtains the content from the authoritative backend, and encrypts an
   authenticated response for the reader key.
5. The owner controller publishes the ciphertext or a terminal error to the
   mailbox.
6. The reader's watcher fires and the unit fetches and decrypts the result.
7. The unit acknowledges the result. The reader controller deletes the request
   and response, with expiry-based garbage collection as a fallback.

### Mailbox state

One persisted state machine is preferable to unrelated request and result
collections. An illustrative `secret_read_exchange` would contain:

- request UUID;
- local reader unit UUID and relation UUID;
- full secret URI;
- optional explicit revision plus `refresh` and `peek` flags;
- reader public key and cryptographic algorithm version;
- state such as `pending`, `claimed`, `completed`, `failed`,
  `acknowledged`, or `expired`;
- encrypted result or a bounded terminal error;
- creation, claim, completion, acknowledgement, and expiry times; and
- an owner response identity or attempt number for idempotency.

This is a new table if the mailbox path is selected. It requires domain methods
for create, claim, complete, acknowledge, expire, and reconcile. Facades should
remain thin adapters over those methods.

### Mailbox advantages

- It works with the existing CMR connection direction.
- The owner side continues to access Kubernetes, Vault, or internal content
  through its local connectivity and credentials.
- Secret plaintext need not be stored on the reader controller; it can store
  only authenticated ciphertext.
- The owner revalidates every read, so revocation does not depend on the expiry
  of a previously issued backend token.

### Mailbox caveats

- The grant and revision control plane is still required; the mailbox only
  transports reads.
- Persistent request state does not by itself survive a full unit-agent restart
  if the private key was only in memory. Either the key must be stored securely,
  or the restarted agent must abandon the old request and create a new one
  while expiry removes the old exchange.
- Claim and completion must be idempotent under worker restart and controller
  failover. A request needs bounded retries, deadlines, terminal errors, and
  queue limits.
- Encryption protects against accidental plaintext storage and passive
  observation. It does not automatically protect against a malicious reader
  controller that substitutes the unit's public key. The threat model must say
  which controllers are trusted, and any proof-of-possession or unit-key
  binding must be explicit.
- The protocol should use a standard authenticated hybrid-encryption scheme.
  Its associated data should bind the request UUID, secret URI, relation,
  reader, revision, and protocol version to prevent replay or substitution.
- A completed ciphertext must remain available long enough for disconnection
  and restart, but not indefinitely. Acknowledgement and TTL garbage collection
  are both necessary.
- Reads have additional queueing and controller round trips. Back-pressure and
  denial-of-service behavior need limits per relation and unit.
- Errors must distinguish retryable disconnection from denied, revoked,
  deleted, expired, and unsupported requests without leaking unrelated secret
  existence.

## Data plane B: a real internal Juju content backend

The alternative is to make the internal Juju backend behave like the existing
Kubernetes and Vault backends from the unit agent's perspective. After the
control-plane grant reaches the reader side, the unit receives a `ValueRef` and
restricted backend configuration and obtains content using its backend client.

### Proposed flow

1. The reader unit asks its local `SecretsManager` for content information.
2. The reader controller checks the local relation and mirrored grant and
   returns a synthetic internal `ValueRef` plus restricted configuration.
3. `internal/secrets/provider/juju` constructs a real, read-only backend.
4. That backend connects to a dedicated owner-side secret-content service using
   a narrowly scoped capability.
5. The endpoint validates the live relation, grant, capability, unit identity,
   revision, and read flags, then obtains content from the owner model's
   authoritative backend.
6. The endpoint returns the content directly to the reader unit and records the
   tracked revision on the owner side.

The endpoint may share the controller's TLS listener and routing
infrastructure, but it should be a content-only authorization surface. The
credential must not be usable as a general model or controller API login.

Internal backend `SaveContent` and `DeleteContent` should remain unsupported
for unit-side clients. Secret writes, rotation, and deletion continue through
the established hook and domain transactions.

### Capability and restricted configuration

A capability should be short-lived and restricted to:

- one owner model;
- one offer connection and relation;
- one remote application or unit identity;
- read-only secret operations;
- an expiry and explicit audience;
- the negotiated protocol version; and, if feasible,
- proof-of-possession by a unit-held key rather than an unbound bearer token.

The owner endpoint must still check current domain state on every request. A
token proves which relation and reader are making the request; it is not a
cached grant. Immediate revocation should therefore be possible without
waiting for token expiry.

Restricted configuration may need endpoint addresses, server identity or CA
material, source model UUID, capability, expiry, backend type and ID, and a
revision identifier. The minimum required set depends on whether the unit
contacts the owner controller or the original external backend.

Bearer material must not be placed in relation data. It should be issued to an
authenticated unit through the agent facade and kept only as long as necessary.

### Two direct-backend variants

The direct approach has two possible scopes:

1. **Internal content gateway**: The reader unit always contacts a dedicated
   owner-controller content endpoint. The owner controller reads the actual
   internal, Kubernetes, or Vault backend locally. This gives one remote
   protocol and exposes no source-backend credentials, but requires the unit to
   reach the owner controller and puts content traffic through it.
2. **Source backend access**: The owner issues the same kind of restricted
   configuration used for a local Kubernetes or Vault reader, and the remote
   unit contacts that backend directly. This avoids proxying content through
   the owner controller, but additionally requires network reachability to the
   source backend and safely exportable backend credentials and CA material.

The internal content gateway is the closer analogue for Juju's currently
dummy backend and gives the cleanest uniform unit-side interface. It does not
require all source backends to be remotely addressable.

### Direct-backend schema

The common remote grant and synchronization tables are still required.
Additional tables depend on credential lifetime:

- An owner-side `secret_backend_issued_token` table is likely if tokens need
  explicit revocation, audit, or renewal. It would record a token UUID,
  relation, remote application or unit, backend, audience, expiry, and optional
  key binding. Existing backend-token persistence has a TODO in this area.
- A reader-side `secret_remote_backend_access` table may be needed if restricted
  configuration must survive controller or unit restart. Avoid persisting a
  bearer token there if it can be reissued safely on demand.

These tables are not needed for the mailbox path. Conversely,
`secret_read_exchange` is not needed for direct reads.

### Direct-backend advantages

- It uses the unit agent's existing backend abstraction instead of adding a
  special asynchronous read API.
- Reads are request-response and require less persisted exchange state.
- Latest, refresh, peek, and explicit revision semantics can live at one
  authoritative content endpoint.
- A dedicated service can expose a smaller attack surface than the general
  controller API.
- If connectivity is a supported premise, it is the simpler long-term data
  path.

### Direct-backend caveats

- It cannot be assumed to work with today's CMR network topology. The offering
  unit may have no route to the consuming controller, Kubernetes API, or Vault.
- The owner endpoint needs stable, advertised, externally routable addresses
  and correct TLS identity across controller HA changes, NAT, and firewalls.
- Token issuance, delivery, expiry, renewal, revocation, and restart behavior
  become security-critical protocol state.
- Existing Kubernetes and Vault restricted tokens enumerate exact readable
  revision IDs and are short-lived. Rotation therefore still requires fresh
  configuration or an endpoint that resolves revisions dynamically.
- Direct access does not remove metadata propagation. The reader controller
  still needs grant, latest-revision, revoke, delete, relation-life, and
  capability state to authorize local API calls and generate
  `secret-changed`.
- A content gateway deliberately lets the owner controller process plaintext.
  Current forward CMR handling already passes some content through controller
  code, so any claim that a design hides content from a controller must define
  whether it concerns storage at rest, normal process visibility, or a
  malicious controller.

## Data-plane comparison

| Property | Encrypted mailbox | Direct internal backend |
| --- | --- | --- |
| Required network path | Existing owner-controller to reader-controller CMR connection. | Reader unit to owner content endpoint; source-backend variant needs more routes. |
| Read model | Asynchronous request, claim, result, and acknowledgement. | Synchronous backend `GetContent`. |
| New durable data | Read exchange state and garbage-collection metadata. | Potential issued-token and restricted-config state. |
| Owner authorization | Rechecked while servicing every request. | Rechecked at the endpoint on every read. |
| Plaintext at reader controller | Avoidable; ciphertext only. | Avoidable if configuration passes through but unit talks directly to owner. |
| Restart complexity | Exchange state plus unit private-key recovery or retry. | Token/config renewal and endpoint rediscovery. |
| Rotation notification | Common control-plane snapshot. | Common control-plane snapshot. |
| Revision retention feedback | Reciprocal tracked-revision snapshot. | Can be recorded during direct read, with reconciliation still required. |
| Main advantage | Works with existing CMR connection direction. | Simpler and consistent with the unit backend abstraction. |
| Main risk | Stateful asynchronous protocol and cryptographic lifecycle. | Unproven or unsupported reverse network reachability. |

The direct backend is preferable if Juju explicitly requires and validates
reader-unit reachability to the owner content endpoint. If reverse secret
sharing must work under the existing CMR network contract, the design needs a
relay, tunnel, or mailbox; a direct path cannot be selected on code structure
alone.

## Lifecycle semantics shared by both paths

### Grant and first read

The owner transaction that creates a relation-scoped grant becomes visible to
an owner-side watcher. The worker reconciles a new full snapshot to the reader
controller. The local reader unit is authorized only after that snapshot is
applied. Content delivery then revalidates the authoritative grant.

This means the current explicit rejection of a consumer-side CMR grant must be
replaced by relation and application resolution, not simply bypassed.

### Revision and `secret-changed`

When the owner adds a revision, the snapshot's latest revision increases. The
reader-side transaction updates `secret_reference.latest_revision`, and the
existing consumed-secret watcher pattern notifies each affected local unit.
The unit re-queries and decides whether to refresh according to current secret
semantics.

Notifications are hints, not payloads. Coalescing multiple changes is safe
because the reader re-queries current state.

### Tracked revisions

A non-peek read updates the reader unit's tracked revision. That update is
reconciled to the owner-side `secret_remote_unit_consumer` record. Peek must not
accidentally pin a revision if current local semantics do not do so. Refresh
must atomically select the intended latest revision and update tracking.

Pruning on the owner must consider both local and remote consumers. Disconnect
should delay pruning conservatively until tracked-reader state is reconciled or
the relation is conclusively gone.

### Revoke, delete, and relation teardown

A revoked grant disappears from the next snapshot. Applying that snapshot
removes the reader-side remote grant and wakes affected watchers. The owner
rejects any later content request even if the reader has stale mirror state or
an unexpired capability.

Secret deletion and relation teardown need the same fail-closed behavior. Any
mailbox exchanges become terminal or expire, issued capabilities stop working,
remote consumer tracking is removed, and reader-side references are removed
when no longer shared by another live relation.

### Expiry and rotation

Explicit revision expiry is authoritative on the owner side and must be checked
at read time. Rotation updates the latest-revision control snapshot but does
not independently grant access. Restricted external-backend configuration may
also need renewal when the set of permitted revision IDs changes.

## Failure, restart, and high-availability behavior

Workers should reconcile durable domain state and be safe to restart. They
must not own authoritative state only in memory.

At minimum, the future design should specify behavior for:

- duplicate, delayed, and reordered snapshots;
- reconnect after one side missed several revisions and revocations;
- owner or reader controller leadership changes during reconciliation;
- unit restart during a content read;
- relation departure while a read is in flight;
- secret deletion after authorization but before backend retrieval;
- backend timeout after a mailbox request is claimed;
- capability expiry during a direct request;
- an unavailable source backend; and
- model migration while sync, tokens, or exchanges exist.

Full-snapshot reconciliation plus monotonic generations should make control
state converge. Content reads should either return one authorized revision or
a well-defined error. A retry must never revive a revoked grant.

Mailbox claims and completions need compare-and-swap state transitions. Direct
tokens should be reissuable and short-lived so that losing ephemeral delivery
state is recoverable.

## Security and trust model

Before implementation, the design must state what it protects against:

- an unauthenticated network peer;
- a unit from another application or relation;
- a compromised unit within the granted application;
- a stale or replayed controller request;
- accidental secret persistence in controller state; and
- a malicious or compromised owner or reader controller.

Some goals are incompatible with the existing controller trust model. An owner
controller necessarily sees locally managed plaintext. A reader controller can
usually influence the identity and credentials delivered to its units. End-to-
end mailbox encryption can prevent routine plaintext storage on that
controller, but cannot by itself prove that a public key belongs to an honest
unit if the controller is malicious.

Regardless of data plane:

- authenticate the CMR relation and derive scope server-side;
- use TLS with verified endpoint identity;
- bind capabilities or ciphertext to relation, reader, secret, revision,
  audience, and protocol version;
- make all access read-only and time-bounded;
- re-check authoritative access at content-read time;
- prevent token and ciphertext replay across relations;
- bound request sizes, concurrency, and error detail;
- avoid logging content, credentials, or unrestricted backend configuration;
  and
- provide audit events for grant, revoke, denied read, and successful remote
  read without recording secret content.

## Capability negotiation and rollout

A charm `assumes` declaration can prevent a feature-dependent charm from being
deployed to an obviously unsupported controller, but it is not sufficient for
this protocol. Two controllers and a unit agent participate, and they may be
at different versions during upgrade.

The protocol needs facade capability or version negotiation for at least:

- reverse-grant snapshot reconciliation;
- reciprocal tracked-revision reconciliation;
- mailbox exchanges, if selected;
- direct internal backend configuration and content reads, if selected; and
- cryptographic or token format versions.

Unsupported combinations should fail before a charm believes a grant is
usable. The existing forward flow must remain unchanged. Rolling upgrade and
downgrade behavior must define when the feature becomes available and what
happens to existing reverse grants if a participant loses support.

Model export, import, and migration must include the common grant and sync
state. Short-lived direct tokens should normally be reissued rather than
migrated. Pending mailbox exchanges can either be migrated with their state or
failed explicitly so that reader units retry; silently losing them is not
sufficient.

## Juju 4.0 and Juju 3.6

The issue was reported against Juju 3.6, whose persistence and worker
implementation differs from Juju 4.0. In 4.0, new workflows should live in
domain services with state hidden behind domain interfaces, Dqlite schema and
migrations, change-log watchers, thin facades, and restartable reconciliation
workers. Facades must not contain the workflow or receive raw database access.

A 3.6 backport would need equivalent Mongo collections and legacy worker and
facade integration. The shared wire protocol can be designed once, but the
persistence implementation should not force 4.0 back into legacy Mongo
patterns. Backport feasibility is a separate decision from the 4.0 design.

## Open decisions

The following decisions remain before an implementation design can be
accepted:

1. Does Juju guarantee that offering-side units can reach a consuming-side
   secret content endpoint? If not, is a relay or mailbox required?
2. If direct reads are selected, does the unit contact one owner-controller
   gateway or the original Kubernetes or Vault backend?
3. What exact trust property is required regarding plaintext visibility to the
   reader controller?
4. What stable identity represents a remote reader unit, and is the current
   anonymized unit name collision-safe across relations and models?
5. Which current change-log sequence, if any, can supply a durable per-relation
   generation? If none can, what sender and receiver sync-state schema is used?
6. How are capability issue, proof-of-possession, renewal, revocation, audit,
   and key rotation implemented for direct reads?
7. How does a mailbox unit recover its private key or abandon and retry an
   exchange after restart?
8. What are the exact latest, explicit revision, refresh, and peek semantics at
   the new boundary?
9. How quickly must revoke and delete become effective during controller
   disconnection, and which side fails closed?
10. How are new state and in-flight reads handled by model migration, relation
    departure, rolling upgrade, and downgrade?
11. Is the feature targeted only at Juju 4.x, or must its wire contract support
    a Juju 3.6 backport?

## Suggested future design sequence

If work resumes, a useful sequence is:

1. Specify the reverse grant and reciprocal tracked-revision state machines,
   identities, snapshots, generations, domain interfaces, and schema.
2. Prototype and test connectivity from an offering-side machine and
   Kubernetes unit to a consuming controller in supported CMR topologies.
3. Select the mailbox or direct backend data plane from the resulting network
   contract and threat model.
4. Specify its content-read protocol, lifecycle, errors, and persisted state.
5. Add facade capability negotiation and mixed-version behavior.
6. Implement domain state, migrations, watchers, and restartable workers before
   wiring thin facades and agent clients.
7. Test internal, Kubernetes, and Vault content across machine and Kubernetes
   models, including HA failover, rotation, prune, revoke, relation teardown,
   disconnection, restart, and migration.

The control-plane work is not throwaway: both data planes require it. Settling
that contract first reduces the later choice to content transport, credentials,
and connectivity rather than mixing it with grant semantics.

## Relevant Juju 4.0 code

The current implementation areas most relevant to a future investigation are:

- `domain/schema/model/sql/0012-secret.sql`
- `domain/schema/model/sql/0034-cross-model-relation.sql`
- `domain/crossmodelrelation/state/model/secrets.go`
- `domain/secret/state/state.go`
- `domain/secret/service/watcher.go`
- `apiserver/facades/controller/crossmodelsecrets/crossmodelsecrets.go`
- `apiserver/facades/agent/secretsmanager/secrets.go`
- `internal/worker/remoterelationconsumer/localconsumerworker.go`
- `internal/secrets/backend.go`
- `internal/secrets/provider/juju/provider.go`
- `internal/secrets/provider/juju/backend.go`
- `internal/secrets/provider/kubernetes/provider.go`
- `internal/secrets/provider/vault/provider.go`
- `domain/secretbackend/service/service.go`


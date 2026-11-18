# Storage resize implementation planning

Juju needs a cohesive contract for changing storage allocation defaults and
resizing existing storage. The initial assessment is that application storage
directives remain useful as allocation defaults, but the 3.6 implementation does
not provide the guarantees needed for resizing existing storage safely.

Status: working prototype and cloud resize infrastructure plan, updated on
2026-11-13. Workload preparation remains an explicitly invoked charm action;
new hooks and uniter coordination are out of scope. The selected workflow is `application-storage` followed by
`juju resize-storage <unit>`; see the command decision below.

## Prototype location and current additions

The prototype is on
[4.1-ephemeral-storage-resize-prototype](https://github.com/manadart/juju/tree/4.1-ephemeral-storage-resize-prototype).
It is a throw-away proof of concept, with no new automated tests requested.

The implementation through `0745bd5c422` provides:

- Durable resize intent, one pending request per storage instance, retained
  completed/failed results and a stable request UUID.
- State, service and client API methods for submission and result lookup.
  `resize-storage --wait` waits for the recorded outcome;
  `show-storage-resize <request-id>` retrieves it later.
- Machine-authorised execution APIs, persisted executor/attachment identity,
  lifecycle revalidation and atomic reporting of measured capacities.
- Execution in the existing machine storage provisioner, serialized with its
  attachment/removal operations. Loop backing files grow in place, the device
  capacity is refreshed, and whole-device managed ext4 filesystems can grow.
- Cancellation, repeatable growth stages and retries after ambiguous reporting
  failures. Unsupported providers and layouts produce explicit results.

The execution additions are split into state (`ce7bd98ede0`), service
(`f887bc902a7`), API (`de57eec40b4`) and worker (`0745bd5c422`) commits.
Existing state/service/schema and worker tests passed with the race detector;
24 bounded worker stress runs reported no failures. Scratch database and
backing-file checks passed, with device commands simulated in the latter.
No live device or cloud resize was validated. Full lint was unavailable because
`golangci-lint` was missing; plain vet also found an existing unkeyed test literal.

## Directive-driven unit resize increment

The command workflow now uses `application-storage <app> <name>=<size>`
followed by `resize-storage <unit> [--wait]`. Submission stages or reuses the
existing per-instance requests and reports already satisfied instances. The
capacity target comes from the current application directive. Existing provider
and machine execution stages are unchanged. See the command decision below for
atomic staging, conflict handling and allocation ordering.

The state and service changes are in `9c15120885d` and `5c741408cda`. Version 23
of the Application facade identifies allocation-only directive updates; Storage
version 10 adds unit submission. Earlier storage request APIs remain available
for compatibility. The CLI requires those versions for the new workflow and
supports size-only updates, preserving the pool and count through API encoding
and persistence. Pool, count and compound directives are rejected. After a
successful update, the command reads back and displays the affected directives
in the same format as a query, including their new sizes and existing pool/count.

Existing state, service, client API, facade and command suites pass with the race
detector. Forty bounded stress runs of the existing storage command suite pass.
A disposable SQLite model check exercised the new state and service paths:
size-only updates preserve pool/count, create no requests, reject stale writes
and allocations, stage multiple instances, reuse matching pending requests,
reject conflicting batches, detect attachment changes and skip sufficient
observed capacity. Raw block no-op detection requires both backend and machine
device capacity, so partial backend growth still stages attachment work. The check caught and verified a SQLair projection correction
in `e981db49466`. No new repository tests were added; existing mocks and the
command-list fixture were updated. Static checks pass through schema and import
validation, then stop because `golangci-lint` is not installed.

The size-only refinement passes the existing core storage, application service,
client API, facade and command suites with the race detector. Disposable command
and SQLite checks confirm the update output matches a single-directive query,
zero counts are preserved, pool/count edits are rejected, and failed updates
produce no success output. No new repository tests were added.

The live controller has not been upgraded and the new command workflow has not
been run against it. Use the LXD verification commands below after installing
the new agent.

## Kubernetes allocation directive increment

Size-only directive updates now support PVC-backed Kubernetes storage. Updating
a directive changes allocation defaults; it does not resize existing claims.
Before scaling, the application provisioner creates missing PVCs for the desired
ordinals using current defaults or persisted unit allocation sizes. It uses the
existing StatefulSet's claim-template names without replacing the StatefulSet.
Existing claims retain their capacity and binding, including across retries and
concurrent claim creation. Claim preparation must succeed before scaling.

The state, service and worker/provider changes are in `86d5c181a35`,
`b9073e29400` and `ada546b0044`. Kubernetes resizing of existing claims and
rootfs/tmpfs directive updates remain outside this increment.

Existing application state/service, storage provisioning state/service,
Kubernetes provider, application provisioner and command suites pass with the
race detector. Forty bounded application provisioner stress runs pass. No new
repository tests were added; existing mock and call fixtures were updated.
Disposable SQLite checks cover allocation snapshots, stale directive rejection,
zero counts and unsupported storage types. A fake Kubernetes client check covers
new ordinal sizes, persisted allocation sizes, unchanged existing claims and
StatefulSet identity/templates, retry and create races, cancellation, zero-count
defaults and claim-creation failures preventing scale changes. These checks do
not establish live Kubernetes behaviour; no cluster has been upgraded or tested.
Repository static checks pass through schema, import and context validation,
then stop because `golangci-lint` is not installed. Licence and shell-format
checks are skipped because `go-licenses` and `shfmt` are also unavailable.

The CLI and controller agent build as `4.1-beta3.6` in `.bin`. Command help now
identifies PVC-backed Kubernetes directive updates as supported and keeps
existing-storage resizing restricted to machine models.

## Integer resize request identifiers

Each resize request now has an internal UUID and a model-scoped integer natural
key. User commands display and accept the integer ID. A dedicated resize
sequence allocates IDs in the same transaction that creates the requests;
matching pending requests retain their original IDs when reused. A cumulative
schema patch numbers existing requests while preserving their UUIDs, execution
state and results.

There is no parent operation or operation/task hierarchy. When
`resize-storage <unit>` finds several storage instances below their current
application targets, it stages independent requests atomically and returns an
indented list of request IDs and storage names. Instance IDs disambiguate
multiple instances of the same storage name. For example:

```text
Storage resize requests for postgresql/0:
  - request 41 for pgdata (pgdata/0)
  - request 42 for pgwal (pgwal/1)
Check request status with 'juju show-storage-resize <id>'
```

Storage that already satisfies its target is reported separately without a
request ID. YAML and JSON output retain the storage-to-request mapping and use
the integer IDs. `--wait` follows each individual request, and
`show-storage-resize 41` retrieves its retained result. Worker coordination and
reporting continue to use UUIDs.

Storage facade version 11 exposes integer-ID lookup; the client requires that
version before submitting unit requests. Earlier UUID-based API methods remain
available for older clients. Existing race-enabled schema, storage state/service,
client API, facade and command suites pass. Disposable database and command
checks cover cumulative upgrades, ID uniqueness and lookup, retry reuse,
retained removal results, multiple instances of one storage name, formatted
output, no-op submissions and partial failures. Forty bounded stress runs of
the existing storage command suite pass. No new repository tests were added.
No live controller has been upgraded or exercised with these changes.

## Resize coordination with storage lifecycle

The storage provisioner serialises resize reconciliation steps with creation,
attachment, detachment and removal in its worker loop. For local ephemeral
storage, this prevents the worker from unmounting a filesystem, detaching its
loop device or deleting its backing file while a resize step is using it.
Three cases still require explicit handling:

1. **Lifecycle changes in controller state.** Storage or its attachments can
   become dying while the worker is executing a step. Queued resize work must
   revalidate lifecycle and resource identity before mutation. Completion
   reports must reject stale state, even though the physical operations in the
   worker are serialised.
2. **Removal between resize stages.** Backend growth and device/filesystem
   growth are separate steps. Removal can intervene between them. Further
   resize steps must stop, and the request must retain a terminal result
   reflecting the interruption and any growth already completed.
3. **Asynchronous provider operations.** A provider resize may continue after
   its method returns. Serialising method calls does not prevent a subsequent
   removal from overlapping that remote operation. Future provider support
   must coordinate conflicting operations for the actual lifetime of the
   remote work, including across model/machine workers and worker restarts.
   Reconciliation calls remain bounded; the worker loop must not wait for the
   whole remote operation to finish.

The intended guarantee is that removal prevents further resize steps from
starting, any executing step settles before conflicting teardown proceeds,
and the resize request receives a retained terminal result. Serial execution
within one worker is the local foundation; durable coordination for remote
operations remains a provider implementation requirement.

## Cloud resize infrastructure increment

The initial cloud increment added optional provider contracts and coordination
without provider implementations. The LXD ZFS increment below implements the
first provider. A source that does not implement its optional resize interface
must produce an unsupported result during checks.
Provider inspection errors remain retryable errors, not capability decisions.

The request remains the user-facing operation. Generalise the single machine
executor into a persisted backend stage and, when needed, an attachment stage:

1. Resolve the provider resource, scope and relevant attachment identities.
2. The backend worker checks eligibility and reports prerequisites and follow-up
   requirements. The machine checks the device and managed filesystem.
3. The service approves the target only after all required checks pass against
   unchanged identities. Requested, approved and observed sizes stay distinct.
4. The backend worker reconciles growth through idempotent ensure calls.
   Acceptance is persisted as progress and does not complete the stage.
5. After backend completion, the machine refreshes device capacity and performs
   supported filesystem growth. The service completes the request only after
   every required stage has reported sufficient observed capacity.

State retains stage checks, requirements, progress and partial observations.
Reports are conditional on the checked resource/attachment identity and stage;
repeated reports are harmless and cannot regress a completed stage. Backend
observations update volume capacity, device observations update block-device
capacity, and filesystem observations update filesystem capacity. Partial growth
survives failures. Removal retains the request result as before.

Services coordinate the workflow through transactional state operations. Model
workers use domain services directly; machine workers use their authorised API.
The existing request/result commands remain usable. The model provisioner
resolves the correct pool/source and invokes its optional `Resizer` contract.
Volumes and provider-native filesystems share `ResizeRequest`; its `ProviderID`
identifies the resource within the selected source. Resize source resolution
must not use the creation-specific `Dynamic()` gate. Each reconciliation is
bounded, with backoff for pending/blocked results rather than waiting for remote
completion inside the provisioner loop.

Machine execution is split into local backend growth and attachment expansion.
Loop storage uses both stages on its machine; cloud storage uses a model backend
stage followed by machine attachment work. Reuse device/filesystem validation
and growth helpers. Start with online growth and one writable attachment;
reject unsupported disruption, partition or shared-storage requirements before
backend mutation. Do not infer workload readiness or invoke charm actions.

The initial infrastructure keeps unsupported node transports/layouts explicit.
Provider-native filesystems use the shared contract through their filesystem
source. Unattached storage needing machine work remains pending; backend-only
storage can complete after the required provider observations.
Further provider implementations, offline coordination, multi-attachment
execution and migration of active resize operations remain follow-up work.

The implementation is split into documentation, provider contracts,
state/schema, service coordination, API, and worker/executor commits.
`storage_resize_plan` binds the resource snapshot and approved size;
`storage_resize_stage` records backend and attachment checks and observations.
The machine facade is `StorageResizer` version 2. Version 1 methods cannot bypass
a staged plan. Machines can plan local loop work themselves, so cloud credential
availability does not gate local resizing.

The first cloud attachment executor handles whole SCSI devices with a stable
`/dev/disk/by-id/` link and a kernel rescan interface, plus whole-device ext4.
Other transports, partitioned layouts, shared attachments and disruption
requirements remain explicit failures before backend growth. Provider-native
filesystems use a backend-only stage. Providers other than LXD still fail their
capability check because their sources do not implement the optional `Resizer`.

The agent builds and existing race suites pass. Scratch service
checks exercise approval, partial completion, separate volume/device observations
and rejection of offline requirements. The loop backing-file smoke check still
preserves contents with device commands simulated. These checks do not establish
live SCSI, ext4 or cloud resize behaviour. The later LXD validation is below.

## LXD ZFS provider increment

The LXD filesystem source implements `Resizer` for custom filesystem
volumes backed by ZFS datasets on standalone LXD servers. It grows the existing
volume's `size` quota through the LXD API; no copy, replacement, detach, unmount,
or machine-side filesystem growth is involved. This follows LXD's documented
[ZFS quota semantics](https://canonical.com/lxd/docs/latest/reference/storage_zfs/#quotas).

Checks are read-only. Both check and ensure inspect the actual pool driver,
volume content type, model ownership, stable volume UUID and current quota.
Other pool drivers, ZFS block mode, clustered servers, shared volumes and
unlimited quotas are explicitly unsupported in this prototype. LXD inspection
errors remain retryable. The implementation preserves the existing description
and configuration, sends the current ETag with the quota update and honours the
worker's bounded context for API calls and operation completion.

Ensure re-reads the volume after LXD completes the update and verifies that its
UUID is unchanged. It reports the provider's committed quota, not the requested
value or a reservation of free space in the backing pool. A retry or smaller
target is a no-op when the current quota already satisfies the target. No
separate provider operation journal is required for this convergent quota
update. A timeout or ambiguous response is retried by inspecting the same
volume and target again. LXD 5.21 updates are synchronous; newer LXD versions
may return an operation, whose acknowledgement uses the bounded context.

The existing model-scoped flow is unchanged: record intent, check the provider,
approve the plan, ensure the quota, persist observed filesystem size and finish
the request. The controller agent must contain this implementation. Rebuilding
only the CLI or the workload machine agent does not enable it.

Live provider verification on LXD 5.21.9 used a disposable container and custom
volume in the local `default` ZFS pool. The actual provider grew the mounted
filesystem from 64 MiB to 128 MiB, verified with `df -B1`. A 1 MiB random file
retained its SHA-256 checksum; the volume UUID, attachment and other settings
were unchanged. Repeated ensure, a smaller target, cancellation and rejection of
a `dir` pool were also checked. The disposable resources were removed. This
verified the provider directly; the live Juju request/approval/reporting path
still needs to be exercised with an updated controller agent. No new automated
tests were added for this prototype.

The existing LXD provider suite passes with the race detector, as do 40 bounded
stress runs of its storage suite. Targeted vet passes with the composites check
disabled; plain vet reports existing unkeyed test literals. Repository static
checks reach Go lint, which remains unavailable because `golangci-lint` is not
installed.

To exercise that path after updating or re-bootstrapping the test controller,
use a Juju pool that explicitly selects the existing ZFS pool:

```sh
export JUJU_DATA="$PWD/.juju-data"
./.bin/juju create-storage-pool -m lxd:work resize-zfs lxd \
  driver=zfs lxd-pool=default
./.bin/juju deploy -m lxd:work ./testcharms/charms/dummy-storage-fs \
  resize-zfs-test --base ubuntu@24.04 --storage data=resize-zfs,64M
./.bin/juju storage -m lxd:work
```

Wait for storage attachment. Write a file to `/srv/data`, record its checksum
and filesystem size, then update the default and apply it to the unit:

```sh
./.bin/juju application-storage -m lxd:work resize-zfs-test data=128M
./.bin/juju resize-storage -m lxd:work resize-zfs-test/0 --wait
./.bin/juju show-storage -m lxd:work data/<id>
```

Check the checksum and `df -B1 /srv/data` again on the unit, and use
`lxc storage volume show default <provider-volume-name>` to confirm the same
volume UUID and updated quota. Repeating the same target must complete without
replacing storage. The pre-existing `lxd-zfs` Juju pool in this environment has
a conflicting `zfs.pool_name`, so the commands above use a separate pool.

The initial review baseline is branch `3.6` at
`1cefd614fe63861595d5d43dacf1365de4819f4e`. Links in the existing behaviour and
initial findings sections refer to that revision, including files removed from
4.1. The subsequent baseline verification and provider design discussion use
branch `4.1` at `bf2364777c95828dbd9d6d32f037835f10cb46e9`. The target release
and detailed implementation design remain open. Findings are based on source
inspection; the validation limits are recorded below.

## Existing 3.6 behaviour

The original intention was to change an application's storage directive so that
new units would receive the updated allocation. Existing storage was to retain
its current size. More precisely, the directive governs new allocations:
reusing existing storage need not satisfy the latest default.

Three changes establish the relevant behaviour. Their descriptions and ancestry
were inspected in local Git history.

| Change | Behaviour |
| --- | --- |
| PR #20604, merge `b6cb35a1a41` | Adds the API for reading and updating application storage directives. |
| PR #21309, merge `6b8cbf075b8` | Wires the storage watcher into the Kubernetes application provisioner and recreates StatefulSets with updated claim templates. |
| PR #22388, merge `f3dac7dce98` | Watches PVC changes and updates Juju's recorded sizes and provider identities, including after manual PVC replacement. |

The command and API persist the updated directives through
[`Application.UpdateStorageConstraints`](state/application.go#L3657).
Machine unit creation reads the application defaults in
[`addUnitOps`](state/application.go#L2451). On Kubernetes, the
[application worker](internal/worker/caasapplicationprovisioner/application.go#L390)
reacts asynchronously and calls
[`EnsureStorage`](internal/provider/kubernetes/application/application.go#L2555).

When claim templates change, `EnsureStorage` saves the replica count, deletes
the StatefulSet with orphan propagation, and recreates it with updated claim
templates. Existing pods and PVCs are intended to survive. New claims use the
new template. This operation does not patch existing PVCs to request expansion.

Recreating the StatefulSet while preserving its pods is an established way to
change immutable claim templates. Google's
[StatefulSet volume expansion guidance](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/volume-expansion)
distinguishes that operation from expanding existing PVCs. The replacement of
the StatefulSet is therefore not inherently an invalid approach; its ordering,
recovery and preservation guarantees need scrutiny.

The [scale down and back up test](tests/suites/storage_k8s/resize.sh#L183)
explicitly expects reused PVCs to retain their original sizes. Consequently,
"new units get the new size" is too broad a contract when units reuse storage.

## Initial findings

### F1 PVC replacement does not establish data preservation

The [manual replacement test](tests/suites/storage_k8s/resize.sh#L930) changes the
application directive, deletes the existing PVC and pod, and checks that Juju
reports the new size under the same storage ID. It does not write data and
verify that the data survives.

The state layer explicitly allows changes to
[filesystem identity](state/filesystem.go#L1322) and
[volume identity](state/volume.go#L1504) in Kubernetes models. The provisioner
[updates the existing volume record](apiserver/facades/controller/caasapplicationprovisioner/provisioner.go#L1157)
and corresponding filesystem record with the newly observed provider identity.

This can present replacement storage as the original Juju storage becoming
larger. A storage class with a `Delete` reclaim policy can delete the original
backing storage. With `Retain`, preserving the old volume still does not
automatically bind it to the replacement claim. Juju defaults to retention for
[storage classes it constructs](internal/provider/kubernetes/storage/volumes.go#L101),
but existing cluster storage classes can have different policies.

Kubernetes documents that normal
[PVC expansion](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims)
grows the existing backing volume. Its
[reclaim behaviour](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reclaiming)
explains why deleting a claim has different consequences.

Observing external replacement is useful, but resizing must preserve data and
storage identity. A replacement or migration workflow needs an explicit
contract of its own.

### F2 Directive updates and scaling have no application ordering barrier

The API returns after persisting the directives. Storage changes and scaling
arrive through independent watcher channels in the
[application loop](internal/worker/caasapplicationprovisioner/application.go#L262).
The [operation interlock](internal/worker/caasapplicationprovisioner/ops.go#L787)
blocks scaling when a storage update is already running. It does not prevent
scaling from starting before the storage notification has been processed.

The resulting source-level failure scenario is:

1. An application directive update commits and returns successfully.
2. A subsequent scale request reaches the worker before storage reconciliation.
3. The StatefulSet creates fresh claims using its old template.
4. Storage reconciliation later changes the template but leaves those claims
   unchanged.

The desired contract needs an explicit point after which new allocations use
the accepted directive. A persisted directive revision and an applied revision
are one possible mechanism. Serialising operations that have already started
does not establish ordering between accepted requests.

### F3 The storage watcher retains a stale charm revision

[`WatchStorageConstraints`](state/watcher.go#L2031) accepts collection events for
any charm revision belonging to the application. However, its content filter
calls `StorageConstraints()` on the original `Application` object without
refreshing that object's charm URL.

Storage constraints are
[keyed by application and charm URL](state/application.go#L205). After a charm
refresh, the watcher therefore continues reading the old revision's document.
Changes to the current revision can be filtered out as unchanged. If the old
document disappears, subsequent reads of that key still do not switch the
watcher to the new revision.

The watcher must resolve the current application revision when evaluating a
change, or leave that resolution to the consumer. Recovery must not depend on
restarting the worker to obtain a fresh application object.

### F4 Directive updates can lose concurrent changes

[`UpdateStorageConstraints`](state/application.go#L3697) reads the entire map,
merges the requested fields, and replaces the entire map. The
[replacement transaction](state/storage.go#L1742) asserts only that the document
exists. It does not assert the revision or contents that were read.

Two concurrent updates to different fields can therefore overwrite each other.
For example, a size update and a pool update can both read the same original
map; the later whole-map write can discard the earlier update.

The transaction also lacks an application assertion tying the write to the
validated charm revision and application life. During a concurrent charm
refresh, existing units can retain the old revision's constraints document. A
write to that obsolete document can then succeed rather than retry against the
current application. See the
[reference-counted constraints lifetime](state/charmref.go#L65).

Checking application life and refreshing the object only on a retry is
insufficient when the transaction has no assertion that forces the retry.
Directive updates need atomic validation and concurrency protection.

### F5 Filesystem size reports the requested capacity

[`FilesystemInfo`](internal/provider/kubernetes/storage/storage.go#L226) derives
filesystem size from `pvc.spec.resources.requests.storage`.
[`FilesystemStatus`](internal/provider/kubernetes/storage/storage.go#L36) maps a
bound claim to `attached`. Volume size is separately derived from
[PV capacity](internal/provider/kubernetes/storage/storage.go#L117).

During expansion, a PVC can request 20 GiB while the usable filesystem remains
10 GiB. The resize may still be pending or may have failed. Recording the PVC
request as filesystem size can therefore overstate available capacity and
disagree with the volume record. A bound claim does not establish completion of
filesystem expansion.

Requested capacity, observed volume capacity, observed filesystem capacity and
operation progress need distinct meanings. Completion must be based on the
provider and filesystem observations required by the operation.

### F6 StatefulSet deletion can block the worker indefinitely

[`StatefulSetWithOrphanDelete.Delete`](internal/provider/kubernetes/resources/statefulset.go#L147)
handles NotFound from the delete request, but discards other deletion errors and
then polls until the StatefulSet disappears. The
[caller](internal/provider/kubernetes/application/application.go#L2646) supplies
`context.TODO()`.

If deletion is forbidden while reads continue to succeed, the object remains
and polling can continue indefinitely. A finalizer that prevents completion
can have the same effect. The synchronous call blocks the application worker's
normal reconciliation and does not receive its shutdown cancellation.

Persisting the current operation and replica count is useful recovery
machinery. It needs error propagation and cancellation throughout provider
operations, including waits. Cancellation of a worker must be distinguished
from reversal of a storage expansion that has already started.

## Verified 4.1 baseline

At the original reviewed 4.1 baseline, the standalone application storage
directive update and Kubernetes claim-template workflow from 3.6 were unavailable. The
`application-storage` command is absent, and the retained
[`UpdateApplicationStorage` API implementation](apiserver/facades/client/application/application.go#L2459)
returns a not-implemented error. The storage-constraint watcher, provider
`EnsureStorage` operation and resize integration suite have been removed.

Some related code remains: PVC observation, the unused storage-update operation
value and orphan-deletion helper, and directive reconciliation during charm
refresh. Their presence does not establish support for the standalone update
or existing-storage resize workflow. This verification was source inspection;
no live resizing was exercised.

## Proposed design principles

These are starting points for planning, rather than agreed API or schema
decisions.

- Keep application directives as defaults for future allocations. Give an
  operation on existing storage an explicit target and durable intent.
- Preserve storage identity and data during resize. Treat replacement and data
  migration as separate operations with their own guarantees.
- Define requested and observed capacity at the volume and filesystem layers.
  Report pending, failed and completed work without turning a request into an
  observation.
- Accept requests as durable asynchronous operations. Workers check provider
  capabilities and constraints before authorising resize execution. Represent
  limitations on online growth, filesystem growth and shrinking explicitly;
  do not assume parity across providers.
- Make reconciliation idempotent and recoverable after controller or worker
  restart. Re-read current state after notifications and protect concurrent
  intent updates.
- Define ordering with allocation, scaling, attachment, detachment, charm
  refresh, removal and model migration. Honour lifecycle changes and shutdown
  without losing the record of work already accepted by a provider.
- Keep facades responsible for authentication and wire conversion, domain/state
  logic responsible for validation and durable intent, and workers responsible
  for provider reconciliation.
- Keep workload preparation outside the resize operation. Charm authors can
  provide actions that users explicitly invoke before resizing, with recovery
  handled through the charm's documented procedure. New uniter coordination,
  hooks and generic maintenance mode are outside the current resize scope.

## Command decision

`application-storage` displays the application's storage allocation defaults and
updates their sizes. Only `<storage-name>=<size>` updates are supported; pool and
count cannot be changed with this command. Updating validates and persists the
supplied sizes; it does not submit resize requests or mutate existing storage.
New units allocate storage using
the current directive. Pool and count retain their current values, including a
count of zero. Successful updates print the affected directives using the same
output format as a query. For example:

```text
$ juju application-storage ceph-osd cache-devices=20GiB
Storage        Pool  Size    Count
cache-devices  loop  20 GiB  0
```

`juju resize-storage <unit> [--wait]` applies the current application capacity
targets to that unit's existing storage instances. The command takes a unit,
not a storage ID or an explicit size. Match instances to application directives
by storage name, including every instance when a name has multiple allocations.
Read the application directive rather than the unit's older directive copy.
The existing `add-storage` command still defaults from the unit directive copy;
changing that allocation path is outside this increment.
Resizing never moves storage, creates additional instances or removes surplus
instances.

Stage a unit's requests atomically against a consistent snapshot of its live
attachments and the current application directives. Generate request UUIDs in
the service and revalidate the snapshot when committing. A changed directive,
charm, lifecycle or attachment causes retry or an explicit conflict rather than
partial submission. Targets are fixed once recorded, so later directive edits
do not change work already underway. Reuse matching pending requests; conflicting
pending targets reject the entire submission. Instances with sufficient observed
capacity are reported as already satisfied, without shrinking or new requests.

The result maps storage IDs to integer request IDs and their snapshotted target
sizes.
`--wait` waits for every staged/reused request and reports individual results,
including partial execution failure. `show-storage-resize <request-id>` remains
the way to inspect an individual request later. No parent operation is required.
Request staging is atomic; physical expansion across several instances is not.

For example, after any explicitly invoked charm preparation:

```sh
juju application-storage postgresql pgdata=20G
juju resize-storage postgresql/0 --wait
juju resize-storage postgresql/1 --wait
juju resize-storage postgresql/2 --wait
```

This permits a user-controlled rolling resize and gives a later unit the larger
allocation default. Unit creation must validate the directive snapshot in its
insertion transaction, so an in-flight add-unit cannot commit an obsolete size.
Concurrent charm refresh must likewise preserve or reject a changed directive.

Kubernetes size-only directive updates are supported for PVC-backed storage.
Before scaling, the application provisioner reads current allocation defaults
and persisted unit allocations, then ensures claims for every desired ordinal.
New ordinals use the current default; existing unit allocations retain their
recorded sizes. Claims use the existing StatefulSet's template names, including
legacy names, so its older capacity defaults do not determine new allocations.
Initial deployment and recovery use the same preparation step before scaling.
Claims are created with create-only semantics: existing or concurrently created
claims are checked but never resized or replaced. A created claim fixes its size
across retries and later directive changes. An add-unit transaction rejects a
stale directive snapshot. No directive watcher or StatefulSet replacement is
needed for capacity updates. Existing PVC reuse preserves its size and binding.

Kubernetes rootfs/tmpfs (`emptyDir`) directive updates are excluded: their sizes
are part of the shared pod template and need separate handling. The existing
single-allocation-per-storage-name Kubernetes limitation is retained; multiple
persisted allocations of the same name on a unit are rejected explicitly.

Kubernetes resizing of existing storage remains follow-up work and resize
submission still requires a machine model. A future Kubernetes resize executor
will patch the individual bound PVC, preserve PVC/PV identity, and wait for
Kubernetes controller and node expansion to finish.
It will not require a Juju machine resize worker. Existing provider contracts,
per-instance requests and execution stages remain the basis for both models.

## Asynchronous resize operation approach

The chosen direction is to execute provider checks asynchronously through
workers, at model and machine level as needed, much like provisioning. The user
receives a durable resize operation identifier and can wait for completion or
query its progress and result later. Returning that identifier means the
request has been recorded for checking; it does not mean provider checks have
passed or storage has grown.

Where a workload needs preparation, the user first follows the charm's
documented procedure and explicitly invokes its preparation action. The resize
operation does not invoke that action or wait for a new uniter acknowledgement.

The resize flow is:

1. The user issues `juju resize-storage`. The API authenticates and authorises the
   request, and the domain service validates its structure and resolves the
   target storage from model state. These checks can return immediate errors.
2. Persist an operation containing the requested size, stable storage identity
   and relevant revision and attachment references. Commit the operation before
   returning its identifier. Provider and machine inspection belongs to the
   worker phase, rather than the submitting API call.
3. Workers read the operation and perform the checks appropriate to their
   execution scope. Model-level workers check model-scoped provider operations,
   including Kubernetes. Machine-level workers check local devices, mounts and
   filesystems. Their results and requirements contribute to the same operation.
4. Once all required checks pass and their requirements can be fulfilled,
   transactionally approve the target for execution. Verify that the operation
   generation, resource identity, lifecycle and relevant state still match the
   checked request before allowing mutation.
5. Workers converge the approved target, recording progress through the backend,
   attachment and filesystem stages that apply. Completion requires all stages
   needed for the storage instance; backend request acceptance is insufficient.
   This includes restoring storage access where the operation changed it.
6. The user obtains the result by operation identifier, either through a status
   query or by waiting. Rejection during checking and failure during execution
   are recorded with their phase and reason.

The user handles any subsequent workload recovery through the charm's
documented procedure, including a recovery action where provided. This is
outside the resize operation's completion criteria.

The operation's requested size remains distinct from the approved resize target
until checks pass. Rejection during checking leaves the storage's desired size
unchanged. An already-satisfied request can complete after the necessary
observations without issuing a resize mutation.

A conceptual lifecycle is `pending -> checking -> resizing -> completed`.
Workload preparation and recovery are outside that lifecycle. Rejection or
failure is recorded against the phase where it occurred. Exact state names
and retry policy remain open.
Temporary problems, such as an unavailable machine, must be distinguishable
from definitive rejection; the operation needs an explicit blocked or retrying
policy for them.

The user-facing contract resembles actions: submission returns an identifier,
execution is asynchronous, and progress and results are retrievable. This does
not yet select the existing action implementation or schema for reuse.

Checks cannot reserve the external world. Workers must revalidate relevant
conditions before acting and handle changes between checking and execution.
If execution fails after some stages succeed, the result must retain those
observations and completed stages; failure does not imply rollback of growth.

## Worker coordination and out-of-band workload preparation

### Scope decision

New uniter behaviour for resizing is explicitly out of scope. Introducing new
hooks requires wider consultation with the charming ecosystem. Generic
maintenance mode remains a separate proposal and is not a prerequisite for
storage resizing.

For now, preparation uses the existing charm action mechanism, following the
pattern used for operations such as upgrades: the charm author supplies a
preparation action and the user explicitly executes it outside the resize
workflow. The uniter continues to execute ordinary actions through its existing
machinery; resizing adds no preparation hooks, readiness acknowledgements,
scheduling restrictions or completion notifications.

The current uniter has no resize coordination. Its
[storage resolver](internal/worker/uniter/storage/resolver.go#L173) ignores
further changes once `storage-attached` has committed, except lifecycle
changes. The 3.6 resolver had the same behaviour. Its current
[storage snapshot](internal/worker/uniter/remotestate/snapshot.go#L132) contains
attachment state and location, but no capacity or resize-operation information.
Observing a changed PVC or recorded size therefore does not itself notify a
charm or establish that the workload is ready for disruption.

### Responsibilities

| Participant | Responsibility |
| --- | --- |
| Domain service and persisted operation | Record storage identities, attachments, requirements, worker progress and results; enforce valid transitions and conflicting-operation rules. |
| Model storage worker | Check and grow model-scoped storage, including requesting Kubernetes PVC expansion. |
| Machine storage worker | Check local devices and filesystems; perform required device, partition, filesystem and mount operations. |
| Kubernetes application provisioner | Coordinate any required pod disruption and conflicting scaling or application changes. |
| Charm author | Supply and document any preparation action and workload recovery procedure, including how the prepared condition is maintained. |
| User | Explicitly execute preparation for the affected workloads, submit and monitor the resize, then perform the documented workload recovery. |

These responsibilities do not yet prescribe new worker types or select a
coordinator implementation. Coordination should use durable operation state and
validated transitions. Watcher notifications prompt participants to re-read
that state; receiving a notification does not establish another worker's
readiness or completion.

### Preparation and recovery responsibilities

The charm's action may stop writes, flush data, transfer a workload role or stop
a service. Its documented procedure must cover the affected consumers and keep
the workload prepared for the required interval. Ordinary hooks and actions
continue to run under existing uniter scheduling, so the charm must preserve
its prepared condition when handling them and relevant restarts.

The resize operation does not discover preparation actions, execute them or
interpret their results as a Juju-managed readiness guarantee. Users follow
the charm's preparation procedure before requesting disruptive resizing.
Storage workers still check and establish the physical prerequisites they
control, such as an unmounted filesystem, and revalidate relevant conditions
before mutation. An out-of-band action does not replace those checks.

After resizing, the user follows the charm's recovery procedure once storage
access is safe. This also applies after rejection or failure: preparation may
already have changed the workload. The resize result must expose partial
growth and outstanding storage problems so that recovery can take account of
the actual outcome. Workload recovery is not performed or certified by the
resize operation.

Reusing `storage-detaching` for preparation would be incorrect: its
[commit path](internal/worker/uniter/storage/attachments.go#L124) participates in
removing the attachment. Temporary unmounting for growth must preserve the
logical attachment and storage identity.

### Kubernetes coordination and the 3.6 findings

For Kubernetes, the CSI driver and kubelet perform backend and node expansion.
See the documented
[CSI expansion responsibilities](https://kubernetes-csi.github.io/docs/volume-expansion.html).
If the selected expansion path requires pod recreation, the charm's documented
preparation must account for it, and storage workers must recover their durable
progress after disruption. The prepared workload condition remains the charm's
responsibility. Storage identity and contents must survive; pod recreation is
not a general solution for ephemeral storage.

The earlier findings constrain this coordination:

| Finding | Required coordination property |
| --- | --- |
| F1 Replacement presented as resize | Keep the original storage identity through expansion and restoration of access; reject stale references to replaced resources. |
| F2 Missing ordering barrier | Gate conflicting storage and application work on persisted operation conditions. If defaults are also changed, retain an explicit ordering contract for fresh allocations. |
| F3 Stale charm revision | Re-read current state after notifications and validate that worker checks apply to the current relevant revision. |
| F4 Lost concurrent updates | Protect operation transitions, attachment changes and target updates transactionally. |
| F5 Requested size reported as actual | Record backend, node and filesystem observations separately; do not report resize completion on request acceptance alone. |
| F6 Unbounded blocking | Propagate cancellation and errors through each worker stage; preserve progress and storage recovery obligations when execution stops. |

The model worker, machine worker and Kubernetes application provisioner must
respect the applicable operation conditions. Excluding new uniter coordination
does not remove these storage guarantees or automatically restrict resizing to
online growth. Provider scope and supported interruption requirements remain to
be selected within this preparation model.

## Proposed provider resize contract

The existing
[`Provider`, `VolumeSource` and `FilesystemSource` interfaces](internal/storage/interface.go#L54)
provide a suitable extension point, including for Kubernetes. They currently
have no resize methods. An optional `Resizer` interface allows sources to add
support without implying that every storage source can resize:

```go
type Resizer interface {
    CheckResize(context.Context, ResizeRequest) (ResizeRequirements, error)
    EnsureSize(context.Context, ResizeRequest) (ResizeProgress, error)
}

type ResizeRequest struct {
    OperationID string
    ProviderID  string
    SizeMiB     uint64
    Attachments []ResizeAttachment
}
```

These declarations describe the prototype contract. The selected volume or
filesystem source determines the resource kind; `ProviderID` identifies the
resource within that source. Requests carry stable operation identity, target
capacity and relevant attachment references. Checks and ensure calls share the
same contract for both kinds of source.
The existing `Provider.Dynamic()` describes creation after machine provisioning;
it must not be treated as a resize capability.

### Meaning of the check method

`CheckResize` assesses whether the particular resource can reach the requested
size, and identifies any requirements. Support can depend on the pool, backend,
filesystem, attachment state and target size, even when the source implements
the optional interface.

The method runs in the worker phase of the asynchronous operation. A check can
be a normal call within its worker; the API does not wait for its execution.

| Outcome | Meaning |
| --- | --- |
| Unsupported | The provider cannot resize this resource, or the request violates a constraint such as growth only. |
| Temporarily blocked | The operation may become possible, for example after another operation completes or a suitable attachment is available. |
| Supported with requirements | Growth needs additional work or coordination, such as unmounting, workload downtime, device discovery or subsequent partition and filesystem growth. |
| Already satisfied | Observed capacity has reached the requested target at the layer being checked. |

Checks must have no side effects. A successful check establishes eligibility at
that moment; it does not reserve capacity or guarantee that a later provider
operation will succeed. Failure to inspect a resource must remain an error,
rather than being reported as a definitive lack of support. The exact encoding
of outcomes in `ResizeRequirements` and typed errors remains to be designed.

Requirements must distinguish prerequisites for starting an operation from
follow-up work needed to complete it. An already-satisfied volume check does
not establish that its filesystem or every attached machine has observed the
larger size. The coordinator must assess the complete storage instance.

`EnsureSize` must be idempotent, honour cancellation and handle conditions that
changed after the check. It must distinguish a request accepted by the provider
from completed growth, returning observed progress for reconciliation. Retries
must preserve data and identity and must not repeat destructive creation work.

### Execution scope and persistence

Execution scope and storage lifetime are separate concerns. The existing
provider scope determines whether work runs through the model provisioner or
on a machine. Persistence describes whether storage survives its current
machine or workload. Model-scoped management does not imply persistent storage:
the Kubernetes `rootfs` and `tmpfs` providers also use model scope.

An operation can span both scopes. For a cloud volume containing a Juju-managed
filesystem, the model worker requests backend growth, then machine-side work
observes the larger device and grows the partition and filesystem. Raw block
storage also needs any required node-side expansion or capacity refresh, even
though there is no filesystem to grow. Whether that work needs an additional
attachment-level contract is an open design question.

| Storage form | Proposed resize responsibilities |
| --- | --- |
| Cloud block volume with managed filesystem | Grow the backend volume, observe the device size on the machine, then grow the partition and filesystem. |
| Raw block volume | Grow the backend and complete the required node-side work for relevant attachments. |
| Machine-local loop device | Grow the backing file, refresh loop-device capacity, then grow any partition and filesystem. |
| Machine-local tmpfs | Change the mount size limit and persist the new mount configuration for later attachment. This changes a limit, not a reservation of physical memory. |
| Provider-native filesystem | Change capacity or quota through the filesystem source, subject to that provider's semantics. |
| Kubernetes PVC-backed storage | Request expansion through the bound PVC and observe backend and node/filesystem completion managed by Kubernetes and its driver. |

The Linux interfaces support
[refreshing loop-device capacity](https://man7.org/linux/man-pages/man8/losetup.8.html)
and [changing tmpfs limits by remounting](https://docs.kernel.org/filesystems/tmpfs.html).
These are possible implementation mechanisms, not existing Juju resize support.
Ephemeral storage must preserve its current contents during an in-place resize;
its shorter lifetime does not permit replacement to masquerade as expansion.

### Kubernetes and storage without an independent allocation

Kubernetes fits the filesystem-source contract, but its identities need careful
handling. The current provider represents a
[filesystem with a PV identity](internal/provider/kubernetes/storage.go#L486)
and a [filesystem attachment with a PVC identity](internal/provider/kubernetes/storage.go#L595).
The resize operation must resolve the correct claim and verify its binding and
identity before mutation. It must not assume that the filesystem provider ID
names a PVC. A retained PV without a suitable claim needs an explicit blocked
or unsupported outcome.

For PVC-backed storage, request expansion on the existing claim, subject to the
storage class and driver's support. Observe completion separately from the
requested size. Kubernetes documents both
[PVC expansion](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims)
and the separate
[CSI controller and node expansion stages](https://kubernetes-csi.github.io/docs/volume-expansion.html).
Changing a StatefulSet template alone does not perform those operations.

Machine `rootfs` storage is a
[directory on an existing filesystem](internal/storage/provider/rootfs.go#L190),
without an independent allocation or quota in the current implementation.
Growing the containing root disk affects other users of that filesystem and
needs its own contract; an individual directory resize must not silently imply
that operation.

Kubernetes `rootfs` and `tmpfs` storage use `emptyDir`. Eligibility must follow
the actual volume medium and cluster capabilities. Upstream documents a gated
[in-place resize mechanism for memory-backed emptyDir](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/#resizing-memory-backed-emptydir-volumes)
through the pod resize API. That mechanism has cluster-version, feature-gate
and node requirements and does not cover disk-backed `emptyDir`. Pod recreation
can lose `emptyDir` data and cannot satisfy an in-place preservation contract.

### Durable reconciliation

The domain service should persist the operation's requested target and
generation, then approve execution after the required worker checks pass.
Workers should execute and record the required provider and machine stages.
Requested capacity, approved target, observed capacity and operation progress
need distinct meanings in state. Recovery must cover checking, resizing and
restoration of storage access across controller and machine restarts, partial
completion, changed attachments, superseding requests and owner removal.
Workload preparation and recovery remain outside this reconciliation contract,
as described above.

The current storage worker skips already-provisioned
[volumes](internal/worker/storageprovisioner/volume_events.go#L457) and
[filesystems](internal/worker/storageprovisioner/filesystem_events.go#L355)
for creation purposes. It needs an explicit reconciliation path for resizing
existing resources. Reusing
[managed filesystem creation](internal/storage/provider/managedfs.go#L102)
would be unsafe: that path can destroy partitions and format the filesystem.

The proposed first scope is growth of individual existing storage instances.
Application allocation defaults remain a separate concern. If a later command
combines both operations, it needs an explicit ordering and partial-failure
contract. This is the selected prototype scope.

## Open design decisions

1. Which release and storage providers should the first implementation target?
2. The prototype supports growth only. Which providers can grow volumes and
   filesystems online, and which require workload interruption?
   Which interruption requirements can be supported with out-of-band workload
   preparation and the physical prerequisites established by storage workers?
3. The prototype targets an individual storage ID. If an application-wide
   operation is later offered, when is its target set fixed?
4. What should an existing allocation below the latest application default mean
   when it is reattached or reused by a recreated unit?
5. `resize-storage --wait` and `show-storage-resize` expose results. How should
   detailed stage progress be presented, and how long should results be retained?
6. What durable revisions or operation records order directives, new
   allocations and repeated resize requests? How are superseding requests and
   retries reconciled?
7. How should the resize command and operation result explain preparation
   requirements and the storage conditions relevant to out-of-band workload
   recovery? New charm hooks and uniter readiness coordination are out of scope.
8. How should out-of-band expansion or replacement be detected and represented?
   Which observed identity changes require an explicit replacement workflow?
9. How will the removal ordering above be enforced for asynchronous provider
   operations? What state must survive model migration, backup and recovery?
10. What retry, blocked and timeout policies apply during checking and execution,
    particularly when a required machine is unavailable?

## Validation work

| Finding or contract | Required evidence |
| --- | --- |
| F1 Data and identity preservation | Write identifiable data, resize, and verify the same backing storage and intact data. Exercise external replacement separately with both reclaim policies. |
| F2 Allocation ordering | Control notification delivery so scaling arrives before storage reconciliation after an acknowledged directive update. Verify fresh allocations meet the defined ordering contract. |
| F2 Storage reuse | Scale down and back up, and explicitly attach existing storage. Verify the documented treatment of older allocations. |
| F3 Charm refresh | Keep a watcher alive across refresh through a separately loaded application object, then change the new revision's directives and verify reconciliation occurs. |
| F4 Concurrent updates | Interleave independent directive updates and refresh while an old unit retains the old constraints document. Verify no lost update or successful write to an obsolete revision. |
| F5 Expansion observations | Hold PVC requested capacity, PV capacity and filesystem capacity at different values. Exercise delayed and failed expansion and verify sizes, status and completion. |
| F6 Deletion failures | Deny deletion while allowing reads, and hold deletion behind a finalizer. Verify error handling, cancellation and worker shutdown. |
| Restart and lifecycle | Interrupt each durable stage, restart the worker, supersede the request, and remove the owner. Verify idempotency and defined recovery behaviour. |
| Provider compatibility | Exercise supported and unsupported combinations of provider, storage class, volume type, filesystem and attachment state. |
| Asynchronous submission | Verify the operation is durable before its ID is returned and provider checks run through workers. Query and wait for rejection, failure and completion under that same ID. |
| Check coordination | Hold one required worker check pending or reject it. Verify resizing cannot start and rejection leaves the desired size unchanged. Invalidate completed checks through identity, lifecycle or revision changes and verify stale results cannot authorise execution. |
| Check contract | Exercise every outcome, inspection errors and conditions changing between check and ensure. Verify checks have no side effects and do not promise later success. |
| Out-of-band preparation | Explicitly run a test charm's preparation action, resize and perform its documented recovery. Verify the resize workflow invokes no charm actions or new hooks and enforces observable storage prerequisites. Exercise ordinary hooks and restarts to validate the test charm's preparation procedure. |
| Storage recovery | Fail expansion at each stage and restart workers. Verify storage progress and restoration obligations survive, partial growth remains visible, and the result supports the user's workload recovery procedure. |
| Attachment changes | Add or remove an attachment, refresh a charm, or supersede the request after checks complete. Verify stale check results cannot authorise work on changed resources. |
| Kubernetes pod disruption | With out-of-band preparation completed, exercise required pod replacement and conflicting scale changes. Verify durable storage coordination, preserved identity and contents, and recovery of worker progress. |
| Multi-stage completion | Complete backend growth while holding node or filesystem growth pending. Verify the overall operation remains incomplete and resumes safely. |
| Ephemeral storage | Grow loop and tmpfs storage while preserving contents. Verify persisted configuration and explicit rejection of unsupported rootfs or emptyDir operations. |
| Kubernetes identity | Exercise correct and stale PVC/PV bindings, claim replacement and retained volumes without claims. Verify an old request cannot resize a different resource. |

Concurrency tests must run with `-race`; worker tests also need bounded stress
runs under the repository guidance. Live provider tests must verify usable
capacity and data preservation, beyond API objects and reported sizes.

## Validation limits of the initial assessment

The six findings are source-level observations and inferred failure scenarios.
They have not yet been reproduced as part of this investigation. Existing
integration tests were inspected but not executed against Kubernetes.

A focused race-test attempt for
`ApplicationSuite.TestWatchStorageConstraints` did not execute tests. The
default Go toolchain was 1.25.7, below the module's 1.26.10 requirement. Retrying
with cached Go 1.27.3 stopped during dependency setup: the required
`github.com/juju/replicaset/v3` v3.0.2 module was unavailable locally and its
download-cache lock could not be created on the read-only module cache.
No race or stress result is established by those attempts.

# Storage resize implementation planning

Juju needs a cohesive contract for changing storage allocation defaults and
resizing existing storage. The initial assessment is that application storage
directives remain useful as allocation defaults, but the 3.6 implementation does
not provide the guarantees needed for resizing existing storage safely.

Status: initial findings, asynchronous operation approach, worker coordination
and proposed provider contracts, recorded on 2026-11-10.

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

The standalone application storage directive update and Kubernetes claim-template
update workflow from 3.6 is unavailable at the reviewed 4.1 revision. The
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

## Asynchronous resize operation approach

The chosen direction is to execute provider checks asynchronously through
workers, at model and machine level as needed, much like provisioning. The user
receives a durable resize operation identifier and can wait for completion or
query its progress and result later. Returning that identifier means the
request has been recorded for checking; it does not mean provider checks have
passed or storage has grown.

The flow is:

1. The user issues a resize command. The API authenticates and authorises the
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
5. Where workload interruption is required, prepare the affected workloads and
   record readiness before permitting disruptive storage work. Preparation is
   a separate stage after the side-effect-free checks.
6. Workers converge the approved target, recording progress through the backend,
   attachment and filesystem stages that apply. Completion requires all stages
   needed for the storage instance; backend request acceptance is insufficient.
   Restore workload access and perform any required charm recovery afterwards.
7. The user obtains the result by operation identifier, either through a status
   query or by waiting. Rejection during checking and failure during execution
   are recorded with their phase and reason.

The operation's requested size remains distinct from the approved resize target
until checks pass. Rejection during checking leaves the storage's desired size
unchanged. An already-satisfied request can complete after the necessary
observations without issuing a resize mutation.

A conceptual lifecycle is `pending -> checking -> resizing -> completed`, with
additional preparation and recovery stages when workload coordination is
required, as described below. Rejection or failure is recorded against the
phase where it occurred. Exact state names and retry policy remain open.
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

## Worker and uniter coordination

The uniter needs to participate when resizing requires workload coordination.
Fully online growth may need no uniter involvement beyond any completion
notification we choose to provide. The uniter owns interaction with the charm;
storage workers own the physical storage changes.

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
| Domain service and persisted operation | Record participants, requirements, phase and acknowledgements; enforce valid transitions and conflicting-operation rules. |
| Model storage worker | Check and grow model-scoped storage, including requesting Kubernetes PVC expansion. |
| Machine storage worker | Check local devices and filesystems; perform required device, partition, filesystem and mount operations. |
| Uniter | Coordinate charm preparation and recovery when the workload must participate; deliver any resize notification. |
| Kubernetes application provisioner | Coordinate any required pod disruption and conflicting scaling or application changes. |

These responsibilities do not yet prescribe new worker types or select a
coordinator implementation. Coordination should use durable operation state and
validated transitions. Watcher notifications prompt participants to re-read
that state; receiving a notification does not establish another worker's
readiness or completion.

### Preparation and recovery

For a resize requiring interruption, the conceptual execution sequence is:

`checking -> preparing workload -> resizing -> restoring workload -> completed`

Preparation changes workload state and belongs after the side-effect-free
checks. The charm may need to stop writes, flush data or stop a service. The
uniter records readiness for the specific operation, generation and attachment.
Storage workers can then establish physical prerequisites, such as an
unmounted filesystem, and proceed with the required expansion stages. After
storage access is restored, the uniter coordinates workload resumption.

The preparation contract needs the following guarantees:

- Readiness remains true until released. A completed preparation hook is
  insufficient if a later action or hook can restart the workload. Uniter
  scheduling and the charm contract must preserve the prepared condition,
  including across agent restarts. The set of operations permitted while
  prepared remains a design decision.
- All affected consumers are accounted for. Shared storage may require several
  unit acknowledgements. New attachments or changed participants must invalidate
  readiness or be prevented during the relevant phase. A stale acknowledgement
  must not authorise work on a different attachment or operation generation.
- Preparation has explicit charm support and readiness semantics. Existing
  [hook execution treats a missing hook as success](internal/worker/uniter/operation/runhook.go#L157).
  That behaviour cannot prove that an older charm has quiesced its workload.
- Failure includes recovery obligations. If preparation succeeds but expansion
  fails, the operation must retain the information needed to restore the
  workload safely. Completed growth must remain recorded; a failed operation
  does not imply that the original size or workload state has been restored.

A dedicated resize coordination contract is needed if workload preparation is
supported. Hook names, capability declarations and acknowledgement mechanisms
remain open. Reusing `storage-detaching` would be incorrect: its
[commit path](internal/worker/uniter/storage/attachments.go#L124) participates in
removing the attachment. Temporary unmounting for growth must preserve the
logical attachment and storage identity.

### Kubernetes coordination and the 3.6 findings

For Kubernetes, the CSI driver and kubelet perform backend and node expansion;
the uniter handles any charm coordination. See the documented
[CSI expansion responsibilities](https://kubernetes-csi.github.io/docs/volume-expansion.html).
If the selected expansion path requires pod recreation, preparation must be
persisted before terminating the pod, and the replacement agent must recover
the operation and its remaining obligations. Storage identity and contents must
survive; pod recreation is not a general solution for ephemeral storage.

The earlier findings constrain this coordination:

| Finding | Required coordination property |
| --- | --- |
| F1 Replacement presented as resize | Keep the original storage identity through preparation, expansion and recovery; reject stale references to replaced resources. |
| F2 Missing ordering barrier | Gate conflicting work on persisted conditions and acknowledgements. If defaults are also changed, retain an explicit ordering contract for fresh allocations. |
| F3 Stale charm revision | Re-read current state after notifications and validate that checks and acknowledgements apply to the current relevant revision. |
| F4 Lost concurrent updates | Protect operation transitions, participant changes and target updates transactionally. |
| F5 Requested size reported as actual | Record backend, node and filesystem observations separately; do not release completion notifications on request acceptance alone. |
| F6 Unbounded blocking | Propagate cancellation and errors through each worker stage; preserve progress and workload recovery obligations when execution stops. |

Adding a hook alone does not establish these properties. The model worker,
machine worker, uniter and Kubernetes application provisioner must all respect
the applicable operation conditions.

One possible initial scope is growth that requires no workload interruption.
That would defer the preparation protocol. Whether to notify the charm after
growth, and whether notification or workload recovery must succeed before the
operation reports completion, remain explicit design decisions. This scope
restriction has not been selected.

## Proposed provider resize contract

The existing
[`Provider`, `VolumeSource` and `FilesystemSource` interfaces](internal/storage/interface.go#L54)
provide a suitable extension point, including for Kubernetes. They currently
have no resize methods. Optional interfaces would allow providers to add
support without implying that every storage source can resize:

```go
type VolumeResizer interface {
    CheckVolumeResize(context.Context, VolumeResizeRequest) (ResizeRequirements, error)
    EnsureVolumeSize(context.Context, VolumeResizeRequest) (ResizeProgress, error)
}

type FilesystemResizer interface {
    CheckFilesystemResize(context.Context, FilesystemResizeRequest) (ResizeRequirements, error)
    EnsureFilesystemSize(context.Context, FilesystemResizeRequest) (ResizeProgress, error)
}
```

These are schematic contracts, not agreed Go declarations. Requests need stable
resource identity, target capacity and relevant attachment references. Types,
batching, error representation and the division of node-side work remain open.
The existing `Provider.Dynamic()` describes creation after machine provisioning;
it must not be treated as a resize capability.

### Meaning of the check methods

`CheckVolumeResize` and `CheckFilesystemResize` assess whether the particular
resource can reach the requested size, and identify any requirements. Support
can depend on the pool, backend, filesystem, attachment state and target size,
even when the source implements the optional interface.

The methods run in the worker phase of the asynchronous operation. A check can
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

`Ensure*Size` must be idempotent, honour cancellation and handle conditions that
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
need distinct meanings in state. Recovery must cover both checking and resizing
as well as any workload preparation and restoration across controller and
machine restarts, partial completion, changed attachments, superseding requests
and owner removal.

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
contract. This is a recommendation for planning, not a settled scope decision.

## Open design decisions

1. Which release and storage providers should the first implementation target?
2. Should the initial operation support growth only? Which providers can grow
   volumes and filesystems online, and which require workload interruption?
   Should the first implementation exclude operations needing interruption?
3. How should users target a storage instance? If an application-wide operation
   is offered, when is its target set fixed, and does it also change defaults?
4. What should an existing allocation below the latest application default mean
   when it is reattached or reused by a recreated unit?
5. Which commands and API methods expose operation status, waiting and results?
   Submission returns a durable operation identifier; how should waiting report
   rejection, partial failure and completion, and how long are results retained?
6. What durable revisions or operation records order directives, new
   allocations and repeated resize requests? How are superseding requests and
   retries reconciled?
7. What explicit charm capability and readiness contract supports preparation
   and recovery? Which hooks and actions can run while prepared, and how do
   shared storage and participant changes affect readiness? Is completion
   notification required, and how does its failure affect the operation result?
8. How should out-of-band expansion or replacement be detected and represented?
   Which observed identity changes require an explicit replacement workflow?
9. What happens when storage or its owner is removed during expansion? What
   state must survive model migration, backup and recovery?
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
| Workload preparation | Hold one unit's readiness pending, omit charm support, and attempt conflicting hooks or actions. Verify disruptive work waits for valid readiness from all required consumers. |
| Preparation recovery | Restart agents after preparation, fail expansion at each stage, and fail workload restoration. Verify prepared state and recovery obligations survive and partial growth remains visible. |
| Participant changes | Add or remove an attachment, refresh a charm, or supersede the request after readiness is recorded. Verify stale acknowledgements cannot authorise disruption. |
| Kubernetes pod disruption | Hold preparation pending while attempting pod replacement or conflicting scale changes. Verify required acknowledgements precede disruption and the replacement agent recovers the operation without replacing storage. |
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

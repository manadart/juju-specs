# Potential Kubernetes Controller Restore Approach

**Status:** Design note; command surface not approved

**Related specification:**
[`2610/JUJU-9652/spec.md`](2610/JUJU-9652/spec.md)

## Summary

Kubernetes controller recovery should use two cooperating components:

1. An operator-side orchestrator running outside the controller and talking
   directly to the Kubernetes API.
2. The provider-free `restore-backup` utility running inside a temporary
   maintenance pod against the replacement controller's PVC.

The orchestrator must remain alive while the normal controller StatefulSet is
scaled to zero. The core restore utility should continue to own archive
validation, Dqlite mutation, object restoration and agent-config regeneration,
so IAAS and Kubernetes share the same logical restore path.

An illustrative operator interface is:

```text
juju restore-controller \
  --controller recovery-controller \
  --backup ./juju-backup.tar.gz \
  --kubeconfig ~/.kube/config
```

The name and exact packaging are proposals only. The command must not depend on
the recovery controller API remaining available after preflight.

## Problem Statement

A controller process inside its normal Kubernetes pod cannot safely perform
the whole recovery. Scaling the StatefulSet to zero would terminate that
process, while merely stopping `jujuagentd` inside the pod would leave probe
restart and PVC-exclusion concerns unresolved.

The lifecycle owner must therefore be outside the StatefulSet. It must stop the
normal pod, establish exclusive access to the controller PVC, run a restricted
restore process, rebind Kubernetes metadata from the temporary recovery
identity to the backed-up identity, and restart the StatefulSet only when the
restore state permits it.

## Scope and Assumptions

- The source and replacement are same-release, single-member Kubernetes
  controllers.
- The source and replacement use the same Kubernetes cluster. Cross-cluster,
  cross-provider and cross-substrate restoration are rejected.
- The replacement is freshly bootstrapped and disposable.
- The backed-up controller and controller-model UUIDs, CA and logical state are
  authoritative.
- The replacement contributes its Dqlite membership, cluster credential,
  Service networking, PVC, passwords and Kubernetes resource identity.
- The source API endpoint is moved to the replacement separately by the
  operator; endpoint movement is not performed by the core restorer.

## Current-State Evidence

- Kubernetes bootstrap creates a one-replica controller StatefulSet, a bound
  PVC and a publish-not-ready headless Service used for stable Dqlite DNS in
  `internal/provider/kubernetes/bootstrap.go`.
- The controller and controller-unit bootstrap templates come from a ConfigMap,
  while writable configs on the PVC take precedence when present.
- The current restore implementation has a provider-free
  `cmd/restore-backup`, but its reduced runtime is still machine-oriented.
- No supported Kubernetes restore orchestrator exists at the referenced
  specification baseline.

## Ownership Boundaries

### Operator-side orchestrator

The orchestrator owns Kubernetes API access and lifecycle operations:

- identifying and validating the fresh target;
- inventorying its Kubernetes resources;
- scaling the controller StatefulSet to zero;
- proving that the normal controller pod no longer owns the PVC;
- creating and supervising the maintenance pod;
- streaming or otherwise privately supplying the backup archive;
- rebinding Kubernetes identity metadata after core success;
- deleting the maintenance pod and restoring one replica; and
- reporting required external API-endpoint movement.

### Core restorer

The in-pod `restore-backup` process remains provider-free and owns:

- complete archive and compatibility validation;
- staging and verifying referenced file-object blobs;
- starting only the reduced Dqlite runtime;
- capturing target database facts before mutation;
- importing and overlaying model and controller databases;
- regenerating writable controller and controller-unit configs;
- atomically installing restored file objects; and
- recording durable core restore phases on the PVC.

The core restorer must not call the Kubernetes API or start API, provider,
model or reconciliation workers.

## Proposed Workflow

1. The operator bootstraps a fresh same-release controller in the source
   Kubernetes cluster.
2. The orchestrator reads the backup metadata and validates the requested
   target, Kubernetes endpoint and cluster CA before disrupting the target.
3. It inventories the target Namespace, StatefulSet, controller Services, PVC,
   ConfigMap, Secrets, ServiceAccount and relevant RBAC resources, including
   names, UIDs, resource versions, replicas and identity metadata.
4. It scales the StatefulSet to zero and waits for the normal controller pod to
   be deleted. ReadWriteOnce alone is not accepted as proof of process
   exclusion.
5. It creates a probe-free, one-shot maintenance pod using the target
   controller image, service account and security context. The pod mounts the
   existing controller PVC and assumes the missing `controller-0` hostname and
   headless-Service subdomain required by Dqlite.
6. The archive is supplied through a private stream or temporary volume. It is
   not stored in a ConfigMap or Secret because it may be large and contains
   controller credentials.
7. The orchestrator invokes `restore-backup` in the maintenance pod. The core
   process validates again, records its phase, performs the logical restore and
   exits without starting the controller.
8. After durable core success, the orchestrator updates controller and
   controller-model identity metadata on the inventoried resources. Updates use
   UID and resource-version preconditions and preserve resource names, UIDs,
   selectors, Service networking, PVC binding and Secret data.
9. The orchestrator records finalization progress, removes the maintenance pod
   and restores the StatefulSet to one replica.
10. It waits for the controller pod to become ready and reports that the
    operator must route a source API endpoint to the replacement and reconnect
    using the source controller alias and CA.

## Durable Restore Phases

The phase record must live on persistent target storage and contain at least
the archive checksum, source identities, temporary target identities and
substrate. Its exact representation remains a design decision.

| Phase | Meaning | Permitted recovery |
| --- | --- | --- |
| `prepared` | Validation and staging completed; no database commit is possible yet. | Clean up or retry. |
| `mutating` | Written durably before the first possible database commit. | Keep the controller stopped and replace the disposable target. |
| `core-complete` | Databases, objects and writable configs are complete. | Resume Kubernetes finalization only. |
| `finalizing` | Kubernetes identity rebinding or restart is underway. | Resume idempotent finalization. |
| `complete` | Core restore and Kubernetes finalization succeeded. | Starting or observing the restored controller is permitted. |

Any `mutating` record that is not followed by `core-complete` is treated as a
failed mutation even if the process died before it could record an error. This
conservative rule makes a post-mutation crash distinguishable from a safe
pre-mutation failure.

Normal Kubernetes startup must fail closed whenever the phase record shows
that restoration crossed the mutation boundary but is not complete. It must
not run `bootstrap-state` or recreate writable configs from the recovery
ConfigMap in that state.

## Kubernetes Identity Rebinding

The backed-up controller UUID and controller-model UUID become the logical
identity of the recovered controller. The temporary replacement UUIDs are used
only to inventory and safely match the fresh target.

The final resource inventory and metadata-key map must be specified before
implementation. At minimum it must account for identity-bearing metadata on
the controller Namespace, StatefulSet and pod template, Services, PVC,
ConfigMap, Secrets, ServiceAccount and cluster-scoped RBAC. Rebinding changes
only identity metadata; it does not recreate stable resources or substitute
source-side Service, PVC or credential data.

## Why Not an End-to-End `jujuagentd` Command

A `jujuagentd` command analogous to safe mode could host the reduced database
runtime, but it cannot own the complete Kubernetes lifecycle while running in
the StatefulSet that must be scaled to zero. It would either terminate itself
or leave the normal pod and its probes active.

For that reason an external orchestrator is required regardless of whether the
core restore entry point remains a standalone binary or is eventually exposed
as a `jujuagentd` subcommand. Keeping `restore-backup` standalone preserves a
single provider-free mutation path for IAAS and Kubernetes.

## Security and Operational Constraints

- The archive is credential-bearing and must use private temporary storage,
  restrictive permissions and redacted logs.
- The maintenance pod and normal controller pod must never coexist.
- Resource updates must refuse UID or ownership mismatches rather than adopting
  an object recreated after inventory.
- The recovery controller alias becomes invalid when the backed-up CA and UUID
  are restored. Success instructions must direct the operator to the source
  alias and endpoint.
- The target remains stopped after any uncertain or failed mutation.

## Open Questions

1. Should the orchestrator be a `juju restore-controller` command, another
   Juju-distributed executable, or a separately documented administration
   tool?
2. How should the command select the Kubernetes context and target namespace
   without relying on a live controller API after preflight?
3. Should `restore-backup` accept the archive on standard input and spool it
   privately, or should the orchestrator use a temporary pod volume?
4. What is the complete resource and metadata-key inventory for each supported
   Kubernetes label version?
5. Where and in what format should restore and finalization phases be persisted
   on the PVC?
6. Which external endpoint mechanisms need operator documentation for the
   initial MicroK8s support envelope?

## Suggested Next Decisions

Approve the operator command surface and archive transport, then complete the
Kubernetes resource-rebinding inventory. Once those decisions are recorded,
this approach can be divided into implementation plans for core phase tracking,
Kubernetes config support, orchestration, finalization and end-to-end recovery.

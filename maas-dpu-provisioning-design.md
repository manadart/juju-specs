# MAAS DPU provisioning in Juju

Status: exploratory design note

This document records the current Juju and MAAS provisioning behavior, the
limitations that behavior creates for Data Processing Unit (DPU) support, and
the proposals under discussion. It is not an accepted implementation design.
Names for constraints, provider methods, state entities, and Terraform grammar
are illustrative. The design is scoped to bare-metal MAAS machines.

## Goals and provisioning modes

MAAS is adding support for DPUs used for hardware-accelerated networking. Juju
needs to support three distinct provisioning modes.

1. **Direct DPU provisioning**: An infrastructure-only model can request a DPU
   independently of its host. This model is operated by a networking
   infrastructure persona and is independent of workflow models.
2. **Secure parent provisioning**: A credential that cannot see DPUs can request
   a parent machine that has a DPU. MAAS provisions the parent and performs any
   required DPU work, but Juju receives and records only the parent.
3. **Managed parent and DPU provisioning**: A credential that can see DPUs can
   request a parent that has DPUs. MAAS establishes the parent-DPU relationship,
   and Juju records the parent and all visible attached DPUs so that applications
   can be deployed to the DPUs.

The selectors for these modes must remain semantically distinct. In particular,
"allocate a DPU" and "allocate a parent that has a DPU" select different
resources. The secure and managed parent cases must also express whether Juju is
expected to discover and manage the attached DPUs.

## Current Juju provisioning API

The provider boundary is `environs.InstanceBroker`. Its provisioning operation
is:

```go
StartInstance(
    context.Context,
    environs.StartInstanceParams,
) (*environs.StartInstanceResult, error)
```

The contract is singular in both directions: one Juju machine produces one
provider request, and a successful request returns one provider instance.

### Current request

`StartInstanceParams` combines several kinds of input for one machine:

- selection information: constraints, provider placement, and availability
  zone;
- storage information: root disk, new volumes, and existing attachments;
- networking information: interfaces, endpoint bindings, and subnet-to-zone
  topology;
- boot information: image metadata, Juju tools, and `InstanceConfig`, including
  the machine ID, nonce, base, and agent configuration; and
- operation information: status callbacks, cleanup, and cancellation.

For MAAS, Juju principally translates the following selectors into an allocate
request:

- architecture;
- minimum CPU core count and memory;
- required and excluded tags;
- required and excluded spaces;
- root disk and requested MAAS storage volumes;
- availability zone;
- explicit hostname or system ID placement; and
- the model UUID as the MAAS agent name.

An `image-id` constraint selects the series used by the MAAS deploy operation.

### Current response

`StartInstanceResult` describes one instance and can contain:

- a display name;
- one `instances.Instance`, whose MAAS identity is the machine system ID;
- hardware characteristics;
- network interface information;
- created volumes and attachments; and
- an optional updated environment configuration used by some providers.

The compute provisioner extracts `result.Instance.Id()` and records that single
provider ID against the Juju machine being provisioned. There is no collection
of instances, relationship metadata, or result role such as parent or DPU.

## Current MAAS calls for `StartInstance`

The normal bare-metal machine success path makes the following MAAS calls in
order. Conditional calls are identified explicitly.

1. `GET zones/` validates `AvailabilityZone` when one was supplied.
2. `GET spaces/` resolves positive and negative space constraints. This lookup
   occurs even when there is no explicit space constraint.
3. `GET zones/` is made by the spaces helper when MAAS returned at least one
   space.
4. `POST machines/?op=allocate` allocates one matching machine. The response is
   one MAAS machine plus maps identifying the storage devices and interfaces
   that matched labelled constraints.
5. `GET spaces/` rebuilds the subnet-to-space map used for the returned Juju
   network configuration.
6. `GET zones/` is again made by the spaces helper when MAAS returned at least
   one space.
7. `POST <machine-resource>/?op=deploy` deploys the allocated machine using the
   selected series and the one machine's Juju cloud-init data.
8. `GET domains/` obtains DNS search domains for the network information in the
   result.
9. `POST <machine-resource>/?op=set_owner_data` writes Juju controller, model,
   and machine ownership data.

The owner-data update is best effort: its failure is logged but does not fail
`StartInstance`. Hardware, hostname, system ID, interface data, and matched
storage are read from the allocate and deploy response objects; they do not
cause additional machine lookups.

For an ordinary model machine, success means that MAAS accepted the deploy
operation. `StartInstance` does not wait for the machine to reach `Deployed`.
Bootstrap adds a separate deployment wait.

If an error occurs after allocation, the deferred failure path releases the
one allocated instance. No release call occurs on success.

## Limitations for DPU provisioning

The existing provisioning contract assumes all of the following:

- every constraint describes the same single instance that Juju will record;
- allocation has one primary provider identity;
- one `InstanceConfig` and one cloud-init payload configure that identity;
- success and rollback concern that one identity; and
- one Juju machine is updated from the response.

This contract can represent direct DPU provisioning if MAAS exposes a DPU as an
allocatable machine. It can also represent secure parent provisioning because
the DPU remains an implementation detail behind the parent constraint.

It cannot directly represent a managed parent request that produces a parent
and an arbitrary number of visible DPUs. Extending `StartInstanceResult` with a
second fixed instance would still fail for multiple DPUs and would make generic
provider provisioning responsible for provider-specific topology.

There is also a bootstrapping problem. The parent request carries cloud-init for
the parent Juju machine only. A DPU recorded as a separate Juju machine needs a
different machine ID, nonce, tools selection, and cloud-init payload. MAAS
cannot fully enrol multiple DPU machine agents from the parent's current
request.

## Parent-child machine precedent

`juju add-machine lxd` provides a useful precedent without being an exact model
for DPUs. Juju transactionally creates a parent and an LXD child, for example
machines `3` and `3/lxd/0`. A provisioner running on the parent later starts the
child through an LXD implementation of `InstanceBroker`. The child receives its
own `StartInstanceParams` and is recorded independently.

Two parts of that precedent are useful:

- parent and child are separate Juju machines with separate instance identity
  and boot configuration; and
- an asynchronous, retryable provisioner reconciles the child after the parent
  exists.

The existing implementation must not simply be extended by treating a DPU as a
container. Although a name such as `3/dpu/0` satisfies the current machine-name
grammar and the database has a generic `machine_parent` relation, many consumers
equate any slash-qualified machine name with a container. Current APIs, watchers,
reboot behavior, provisioning selection, CLI checks, and the
`machine_container_type` schema support LXD-specific semantics. DPU children
need an explicit child or provisioning kind and an audit of these consumers.

## Proposed controller-side DPU reconciliation

The current direction keeps the ordinary `StartInstance` contract singular and
introduces a controller-side DPU provisioner or reconciler backed by an optional
provider capability. It runs on the controller because MAAS relationships are
visible through the model's cloud credential; it does not need to run on the
parent machine.

Calling this component only a broker understates its responsibility. A broker
adapts provider operations, while the worker must watch desired and observed
state, retry idempotently, honour cancellation, register instance information,
and coordinate removal.

### DPU cardinality and durable intent

An early proposal was to pre-create one DPU child together with the parent, as
`add-machine lxd` does. This is insufficient because the number of DPUs attached
to the selected parent may not be known until MAAS allocates the parent. A
parent may have zero, one, or several attached DPUs.

Juju should instead persist parent-scoped DPU management intent before
provisioning. Individual child machines are materialised after discovery. The
reconciler should correlate children by stable MAAS DPU system ID, not by the
assigned `3/dpu/N` ordinal, so retries and controller restarts cannot create
duplicates.

A managed provisioning sequence is therefore:

1. Juju creates the parent machine and persists that its visible DPUs are to be
   discovered and managed.
2. The normal compute provisioner calls `StartInstance` for the parent and
   records the parent's MAAS system ID.
3. The controller-side DPU reconciler observes the parent instance ID and asks
   the MAAS provider for all DPUs associated with it.
4. The reconciler transactionally upserts one child machine for each stable DPU
   identity, producing names such as `3/dpu/0` and `3/dpu/1`.
5. Each DPU child receives its own provisioning information and is deployed or
   enrolled independently.
6. Application placement reconciliation deploys declared DPU workloads onto
   the discovered children.

### Provider capability

Because the cardinality is not necessarily one, the discovery operation should
be plural. An illustrative provider capability is:

```go
type DPUProvider interface {
    DPUsForParent(
        context.Context,
        instance.Id,
    ) ([]DPUInstance, error)
}
```

The concrete result type is undecided. At minimum, each discovered DPU needs a
stable provider identity and enough lifecycle and capability information to
decide whether it can be enrolled.

Discovery alone is sufficient only if a DPU has already received valid Juju
machine-agent configuration. Otherwise Juju needs a second operation that
accepts child-specific provisioning data. For example:

```go
type DPUProvisioner interface {
    StartDPUForParent(
        context.Context,
        instance.Id,
        instance.Id,
        environs.StartInstanceParams,
    ) (*environs.StartInstanceResult, error)
}
```

Alternatively, `DPUsForParent` can return associated but undeployed MAAS machine
identities and the provider can use the existing MAAS deploy operation for each
DPU. The preferred boundary depends on what MAAS means by bringing up a DPU as
part of parent deployment.

The interfaces above are deliberately DPU-specific while requirements remain
provider-specific. They can be generalised to related compute resources later
if a second concrete use case establishes the right abstraction.

## Terraform and dynamic DPU placement

Terraform cannot reliably enumerate DPU machine resources in one plan when the
number and identities of DPUs are unknown until the parent is allocated.
Terraform `for_each` keys derived from that discovery would be unknown during
planning and commonly require a second apply.

Terraform can nevertheless declare an application placement policy that Juju
owns and reconciles. Terraform tracks one stable policy rather than every
discovered DPU child. Illustrative grammar is:

```hcl
resource "juju_application" "dpu_networking" {
  name  = "dpu-networking"
  charm = "..."

  placement {
    kind     = "dpu"
    parent   = juju_machine.host.machine_id
    strategy = "all"
  }
}
```

The grammar is not a proposal for the current Terraform Provider for Juju
schema. It demonstrates the required declarative meaning: deploy the application
to all matching DPUs discovered for this parent, rather than deploy to a list of
machine IDs known at plan time.

This resembles a topology-aware fan-out policy:

- discovery of an additional matching DPU creates a child and adds a unit;
- disappearance or release of a DPU initiates defined unit draining and child
  removal behavior;
- zero matches leaves the policy pending rather than inventing a machine; and
- discovered children and units are observed results of the policy, not
  independent Terraform resources that Terraform attempts to recreate.

The policy must specify whether `all` continuously follows topology changes or
captures a snapshot at initial convergence. Continuous convergence is the more
declarative interpretation, but its application scaling and removal semantics
need explicit design.

## Proposed behavior for each provisioning mode

### Direct DPU

- Juju creates a normal top-level machine.
- A DPU-specific resource selector is passed through normal `StartInstance`.
- MAAS allocates and deploys one DPU.
- Juju records one ordinary provider instance result.
- No parent-child discovery worker is involved.

### Secure parent with hidden DPU

- Juju creates only a top-level parent machine.
- A parent capability constraint requires a host with a DPU.
- MAAS performs any hidden DPU preparation using its own privileges.
- Juju records only the parent and does not probe for attached DPUs.
- A permission failure from DPU discovery is therefore not part of this mode's
  normal workflow.

### Managed parent with visible DPUs

- Juju creates the parent and persists DPU management or placement intent.
- Normal `StartInstance` provisions and records the parent.
- The controller-side DPU reconciler discovers zero or more related DPUs.
- Juju creates and independently enrols one child machine per stable DPU
  identity.
- Application placement policies fan out units across the resulting children.

## Desired MAAS API behavior

The principal new relationship operation from Juju's point of view is "give me
the DPUs associated with this machine." It should be plural unless MAAS can
guarantee a one-to-one relationship.

The operation should provide:

- lookup by stable parent machine system ID;
- zero-to-many stable DPU machine identities;
- an idempotent response suitable for reconciliation after controller restart;
- lifecycle or readiness information, or errors that distinguish not-yet-ready
  from a permanent absence;
- clear authorization behavior for credentials that cannot see DPUs; and
- sufficient relationship metadata to prove that each DPU belongs to the
  requested parent.

The existing allocation API also needs selectors for:

- allocating a DPU directly; and
- allocating a parent that has one or more DPUs.

Depending on existing MAAS behavior, further API support may be necessary for
per-DPU deployment with Juju user-data. MAAS and Juju must also agree on a
single owner for release behavior: releasing a parent might release all attached
DPUs, or Juju might release each DPU child explicitly, but both sides must not
independently race the same ownership transition.

## Reconciliation and lifecycle requirements

Any implementation must define the following before code is added:

- **Stable correlation**: The tuple of model, parent provider ID, and DPU
  provider ID maps to at most one Juju child.
- **Restart safety**: A worker restart between discovery and state registration
  does not leak or duplicate DPU children.
- **Typed absence**: No DPU, DPU not yet ready, DPU hidden by authorization, and
  provider failure have distinct outcomes.
- **Partial failure**: One failed DPU does not lose the identities or status of
  its siblings, and the parent result remains recorded.
- **Parent removal**: Parent lifecycle changes drive child draining and cleanup
  in a deterministic order.
- **DPU removal**: A disappeared or failed DPU has defined unit evacuation,
  status, and removal behavior.
- **Application convergence**: Dynamic placement has defined behavior when the
  matching DPU set grows or shrinks.
- **Credential changes**: Loss or restoration of DPU visibility does not cause
  accidental deletion or duplicate adoption.
- **Migration**: Model export, import, and controller migration preserve DPU
  intent, child identity, and parent relationships.

## Open questions

1. Does MAAS expose an attached DPU as an ordinary machine resource with a
   system ID and the normal deploy, status, and release operations?
2. When MAAS "brings up" a DPU with its parent, is the DPU merely powered or
   commissioned, or is an operating system already deployed?
3. Can Juju deploy child-specific user-data after MAAS establishes the
   relationship, or must a compound request provide all boot payloads up front?
4. Can a parent have multiple DPUs, and can the set change after allocation?
5. Which MAAS operation owns reservation and release of an attached DPU?
6. Does direct DPU allocation detach or otherwise reserve the DPU independently
   of its parent?
7. What child-kind representation should replace the current assumption that
   every slash-qualified machine is an LXD container?
8. Does an application placement policy continuously target all matching DPUs,
   or select a fixed set during initial deployment?
9. What should Terraform report when a policy has zero matching DPUs or some DPU
   units fail to converge?
10. Which constraints or placement directives distinguish direct, secure
    parent-only, and managed parent-plus-DPU intent?

## Current source locations

The observations in this note are grounded in the following files on the branch
where the note was written:

- `environs/broker.go`: `InstanceBroker`, `StartInstanceParams`, and
  `StartInstanceResult`;
- `internal/provider/maas/environ.go`: MAAS allocation, deployment, topology
  lookup, owner data, and result construction;
- `internal/provider/maas/constraints.go`: conversion from Juju constraints to
  MAAS allocation arguments;
- `internal/provider/maas/instance.go`: MAAS system ID and hardware result
  mapping;
- `internal/provider/maas/interfaces.go`: conversion of allocated-machine
  interface data;
- `internal/provider/maas/volumes.go`: conversion of allocation constraint
  matches into Juju volumes and attachments;
- `internal/provisionertask/provisioner_task.go`: singular provider invocation
  and instance-info registration;
- `domain/machine/state/placement.go`: transactional parent and LXD child
  creation;
- `internal/container/broker/lxd-broker.go`: independent child configuration and
  provisioning;
- `core/machine/machine.go`: slash-qualified machine names and current
  container classification; and
- `domain/machine/service/watcher.go`: watcher assumptions around container
  children and top-level machines.

At the time of writing, the checkout was `main` at commit
`ab33dc55e790aeee8064a14369a87df5b583e064`, and it used
`github.com/juju/gomaasapi/v3 v3.0.0`.

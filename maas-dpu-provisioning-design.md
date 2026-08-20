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

## Current proposal: preparation before `StartInstance`

The current proposal introduces a durable provider preparation phase before a
machine becomes eligible for the existing `StartInstance` path. The working
name for this phase and its user-visible status is `recon`, preceding `pending`.
The name and whether this should be a machine status at all remain undecided.

The phase is processed by a controller worker through a new provider method.
Providers without preparation work return a no-op result, after which the
machine advances immediately to `pending`. For the managed MAAS parent case,
preparation performs the allocation-only part of today's `StartInstance`, asks
MAAS for every DPU belonging to the allocated parent, and records the resulting
topology before any of those machines boot.

This ordering solves the cloud-init identity problem. Every DPU gets a Juju
machine entity, machine ID, password, nonce, and agent configuration before its
operating system is deployed. The later `StartInstance` invocation can then
carry child-specific cloud-init rather than attempting to enrol all children
with the parent's user-data.

Calling the controller component only a broker understates its responsibility.
It must watch durable preparation intent, retry provider operations
idempotently, persist discovered topology, advance provisioning phases, honour
cancellation, and coordinate cleanup. It does not need to run on the parent
machine because it uses the model's MAAS credential from the controller.

### Proposed managed-parent sequence

1. Juju creates a parent machine in the tentative `recon` phase and persists
   that its visible DPUs are to be managed.
2. The preparation worker asks the MAAS provider to prepare the machine. MAAS
   runs the allocation portion of `StartInstance`, but does not deploy the
   allocated parent.
3. MAAS is queried for the zero-to-many DPUs associated with that allocated
   parent. Preparation must return stable MAAS identities for the parent and
   every visible DPU.
4. A Juju domain operation records the parent's prepared reservation and
   transactionally upserts one child machine for each DPU. Children are
   correlated by stable MAAS DPU identity, not by an assigned `3/dpu/N`
   ordinal.
5. The parent and child machines advance to `pending` once their machine
   entities and prepared reservations are durable.
6. The compute provisioner handles each machine separately. It creates that
   machine's `InstanceConfig` and invokes `StartInstance` with an explicit
   prepared reservation, conceptually similar to a resolved `--to` placement.
7. MAAS adopts the already allocated resource, skips allocation, and deploys it
   with that machine's cloud-init. Successful deployment records the ordinary
   final instance ID and hardware/network information.
8. Application placement reconciliation deploys declared DPU workloads onto
   the discovered child machines.

The sequence retains the singular `StartInstance` request and response: there
is one call, boot payload, result, and Juju machine per parent or DPU. The new
plurality exists in the earlier preparation result and in the domain operation
that materialises related machines.

### Provider capability

An illustrative provider-neutral capability is:

```go
type InstancePreparer interface {
    PrepareInstance(
        context.Context,
        PrepareInstanceParams,
    ) (*PrepareInstanceResult, error)
}

type PrepareInstanceResult struct {
    Primary PreparedInstance
    Related []RelatedPreparedInstance
}
```

`PrepareInstanceParams` must carry a per-Juju-machine idempotency identity and
the selection information needed for allocation, but not final cloud-init.
Each prepared result needs at least a stable provider resource identity, a
reservation or adoption token, a relationship role, and enough observed
hardware information to create and provision the corresponding Juju machine.

Providers that do not implement the optional capability can be treated as
having no preparation work. This may be preferable to adding explicit no-op
implementations to every provider. Whether the interface should remain generic
or be MAAS/DPU-specific depends on whether a second provider use case emerges.

For MAAS, the underlying API requirement remains plural: after allocating a
parent, return all DPUs associated with that machine. The later MAAS
`StartInstance` path also needs an explicit prepared-resource input so it can
adopt and deploy an allocated machine without issuing another allocate request.

### Prepared reservation is not a final instance ID

The proposal says to associate MAAS IDs with the Juju machines during
preparation, but this cannot use the existing final instance-ID field without a
semantic change. The current compute provisioner treats a machine with an
instance ID as already started and does not call `StartInstance`. The current
schema likewise describes that ID as unset until the instance has actually
been created.

Juju therefore needs to distinguish two facts:

- **prepared reservation**: MAAS has allocated or resolved a resource for this
  Juju machine, but `StartInstance` must still deploy it; and
- **instance ID**: deployment has been accepted and the normal provisioner must
  not start another instance.

A separate prepared-reservation entity is preferable to overloading
`machine_placement`. Placement records the user's original directive, while a
reservation records a provider-resolved resource and its lifecycle. The
prepared identity may equal the eventual MAAS system ID, but its state-machine
meaning is different.

Current MAAS placement by `system-id` is also not sufficient by itself. The
MAAS `StartInstance` implementation still calls allocate after resolving that
placement, whereas a prepared machine is already allocated and may no longer be
eligible for allocation. `StartInstanceParams` therefore needs an explicit
prepared reservation or adoption token, not merely a reconstructed placement
string.

### Earlier post-start discovery alternative

The previous proposal provisioned the parent normally, then queried MAAS for
its DPUs and created child machines. That remains a possible inventory model,
but is not sufficient when MAAS has already deployed the DPUs during the parent
operation. A separately managed DPU requires its Juju identity and cloud-init
before first boot.

The preparation proposal moves discovery to after parent allocation but before
parent or DPU deployment. This still accommodates unknown cardinality: a parent
may have zero, one, or many DPUs, and child machines are materialised only after
MAAS reports the actual set.

## Terraform and dynamic DPU placement

Terraform cannot reliably enumerate DPU machine resources in one plan when the
number and identities of DPUs are unknown until the parent is prepared during
apply. Terraform `for_each` keys derived from that discovery would be unknown
during planning and commonly require a second apply.

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

- Juju creates a normal top-level machine in the preparation phase.
- A DPU-specific resource selector causes MAAS preparation to allocate one DPU
  as the primary prepared resource, with no related children.
- The machine advances to `pending`, and `StartInstance` adopts and deploys the
  prepared DPU with its Juju machine-specific cloud-init.
- Juju records one ordinary provider instance result.
- No parent-child materialisation is involved.

### Secure parent with hidden DPU

- Juju creates only a top-level parent machine in the preparation phase.
- A parent capability constraint causes MAAS preparation to allocate a host
  with a DPU, but Juju does not request or receive the hidden DPU identities.
- The parent advances to `pending`, and `StartInstance` adopts and deploys it.
- MAAS performs any hidden DPU preparation using its own privileges.
- Juju records only the parent and does not probe for attached DPUs.
- A permission failure from DPU discovery is therefore not part of this mode's
  normal workflow.

### Managed parent with visible DPUs

- Juju creates the parent in the preparation phase and persists DPU management
  or placement intent.
- MAAS preparation allocates the parent and returns zero or more related DPU
  reservations.
- Juju creates one child machine per stable DPU identity and durably associates
  every parent and child with a prepared reservation.
- The parent and children advance to `pending`; the compute provisioner invokes
  `StartInstance` separately for each prepared resource and records each final
  instance result.
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

For the preparation design, the relationship response also needs to make clear
whether each DPU is reserved for Juju, merely observable, or implicitly owned
by the parent's reservation. Juju must be able to retry the operation and adopt
the same resources after a controller failure.

The existing allocation API also needs selectors for:

- allocating a DPU directly; and
- allocating a parent that has one or more DPUs.

The existing MAAS allocate and deploy operations may be sufficient for the two
halves of preparation and start, but the Juju MAAS provider needs a supported
way to deploy an already allocated machine without allocating it again. Further
MAAS API support may be necessary if attached DPUs cannot use the ordinary
per-machine deploy operation with child-specific user-data.

MAAS and Juju must also agree on a single owner for release behavior: releasing
a parent might release all attached DPUs, or Juju might release each DPU child
explicitly, but both sides must not independently race the same ownership
transition.

## Reconciliation and lifecycle requirements

Any implementation must define the following before code is added:

- **Phase semantics**: Preparation is distinct from machine-agent `pending`
  status, and all readers agree which phase makes a machine eligible for
  `StartInstance`.
- **Reservation semantics**: Prepared provider identity is stored separately
  from the final instance ID and from the user's original placement directive.
- **Provider idempotency**: Retrying preparation for one Juju machine returns
  the same allocation and related DPU set rather than reserving another parent.
- **Side-effect recovery**: If MAAS allocation succeeds and the controller
  fails before state is committed, a retry can find, adopt, or release the
  orphaned reservation. The current model-wide MAAS agent name is not a unique
  per-machine idempotency key.
- **Stable correlation**: The tuple of model, parent provider ID, and DPU
  provider ID maps to at most one Juju child.
- **Atomic domain update**: Parent reservation, child machine creation, child
  reservations, relationships, and phase transitions become durable together.
- **Restart safety**: A worker restart at any boundary between allocation,
  discovery, state registration, and deployment does not leak resources,
  duplicate children, or boot with the wrong cloud-init.
- **Typed absence**: No DPU, DPU not yet ready, DPU hidden by authorization, and
  provider failure have distinct outcomes.
- **Partial failure**: One failed DPU preparation or deployment does not lose
  the identities or status of its siblings, and the parent has a defined
  ability to proceed or roll back.
- **Deployment ordering**: MAAS and Juju agree whether parent and DPU deployment
  may run independently, must be ordered, or requires a compound coordination
  operation.
- **Prepared cleanup**: Cancellation and removal release resources that were
  allocated but never deployed, including partially recorded related DPUs.
- **Parent removal**: Parent lifecycle changes drive child draining and cleanup
  in a deterministic order.
- **DPU removal**: A disappeared or failed DPU has defined unit evacuation,
  status, and removal behavior.
- **Application convergence**: Dynamic placement has defined behavior when the
  matching DPU set grows or shrinks.
- **Credential changes**: Loss or restoration of DPU visibility does not cause
  accidental deletion or duplicate adoption.
- **Provisioner selection**: DPU children are watched and started despite the
  current compute watcher excluding slash-qualified machine names as
  containers.
- **Hardware data**: Preparation and `StartInstance` have a single, defined
  owner for storing parent and DPU hardware characteristics without losing
  information or reporting it twice.
- **Migration**: Model export, import, and controller migration preserve DPU
  intent, prepared reservations, child identity, and parent relationships.

## Open questions

1. Does MAAS expose an attached DPU as an ordinary machine resource with a
   system ID and the normal deploy, status, and release operations?
2. When MAAS "brings up" a DPU with its parent, is the DPU merely powered or
   commissioned, or is an operating system already deployed?
3. Does allocating a parent make its DPU identities and reservation ownership
   immediately available, before either resource is deployed?
4. Can Juju deploy child-specific user-data after MAAS establishes the
   relationship, or must a compound request provide all boot payloads up front?
5. Can an already allocated parent or DPU be adopted and deployed by stable ID
   without another allocate call?
6. What provider-side idempotency key can correlate an allocation with one Juju
   machine across controller failure and retry?
7. Can a parent have multiple DPUs, and can the set change between preparation,
   deployment, and later operation?
8. Which MAAS operation owns reservation and release of an attached DPU?
9. Does direct DPU allocation detach or otherwise reserve the DPU independently
   of its parent?
10. Should `recon` be a user-visible machine status, or a separate provisioning
    phase that leaves agent status semantics unchanged?
11. Should provider preparation be an optional generic capability, or an
    explicit method implemented as a no-op by every provider?
12. What child-kind representation should replace the current assumption that
    every slash-qualified machine is an LXD container?
13. What deployment ordering or failure policy applies when the parent starts
    successfully but one or more DPUs do not?
14. Does an application placement policy continuously target all matching DPUs,
    or select a fixed set during initial deployment?
15. What should Terraform report when a policy has zero matching DPUs or some
    DPU units fail to converge?
16. Which constraints or placement directives distinguish direct, secure
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
  and instance-info registration, including the current rule that an existing
  instance ID prevents `StartInstance`;
- `domain/schema/model/sql/0017-machine-cloud-instance.sql`: current final
  instance-ID storage;
- `domain/schema/model/sql/0018-machine.sql`: provider placement and generic
  parent relationships;
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

# Juju OpenStack network offload prototype

Recorded on 16 November 2026. The agreed interface is a new boolean
`network-offload` constraint, applied to every network port Juju creates for a
machine. An initial implementation is present and uncommitted on branch
`4.1-ovn-offloaded-ports-prototype`, based on commit
`31baaa9c4bb909f98712e69bb64bf34b920dbfb9` (`4.1-beta3`).

The prototype builds locally. Successful provisioning and network behaviour
on an offload-capable cloud remain to be validated. The user is investigating
access to such an environment and explicitly deferred tests for this iteration.

## Requirement and motivation

PS6 has been deployed with OVN offloading support. The request is for Juju's
OpenStack provider to create instances using offloaded ports. Today, the
reported workaround is to create those instances manually and add them to a
Juju model through the manual provider.

The requested ports retain normal networking behaviour: they belong to normal
subnets, have port security, and are subject to security-group rules. Juju
should provision and manage the machines through its OpenStack provider,
including their networking lifecycle.

The initial request suggested a per-model knob, perhaps also a per-controller
knob. The subsequent discussion selected a constraint as the interface, giving
model defaults and more specific application or machine requests.

[Launchpad bug 1910120](https://bugs.launchpad.net/juju/+bug/1910120),
"[openstack] support for modelling port binding attributes", describes the
underlying request. It also envisages mixing ordinary and offloaded ports
according to workload requirements, because hardware-backed ports are finite
resources. The January 2024 follow-up highlights the manual provisioning cost
for workloads needing high network throughput and, in some cases, disk I/O.
The agreed prototype supports choosing different policies for different
machines; selecting individual ports within one machine is deferred.

## Understanding of the OpenStack mechanism

The bug links to the Queens-era
[Open vSwitch hardware-offloading guide](https://docs.openstack.org/neutron/queens/admin/config-ovs-offload.html#validate-open-vswitch-hardware-offloading).
Its provisioning sequence is to create a Neutron port with:

```text
--vnic-type=direct
--binding-profile '{"capabilities": ["switchdev"]}'
```

Nova then creates the server with that port's ID. These attributes therefore
need to be supplied before the instance boots. Creating a port record alone
does not establish that Nova can allocate a suitable device or connect a
working instance.

The request's distinction from SR-IOV concerns networking semantics. The
documented implementation still uses SR-IOV virtual functions underneath,
along with switchdev and OVS offloading. OVN can connect the accelerated ports
to the OVS integration bridge so that instances participate in ordinary tenant
networks. The cloud already supplies that infrastructure capability; the Juju
change requests and manages its use.
[OVN hardware-offloading guide](https://docs.openstack.org/charm-guide/latest/admin/networking/ovn/hardware-offload.html)

Requiring an offload-capable port does not assert that every possible flow or
security rule is executed in hardware. Actual offloading and the required
network behaviour must be checked on the target cloud.

## Discussion and agreed decisions

A constraint was selected because this is a requirement on the machine's
networking. It lets an operator reserve scarce offloaded ports for workloads
that need them, while also providing the model-wide default originally
requested. Existing networking-related constraints, including
`allocate-public-ip`, provide precedent for this kind of provisioning choice.

The user explicitly approved the name `network-offload` and the machine-wide
scope, then authorised implementation without tests for now.

| Decision | Agreed behaviour |
| --- | --- |
| Interface | A boolean Juju constraint named `network-offload`. |
| Scope | Every Neutron port Juju creates for the new machine. Existing network and space selection still determines which networks or subnets are used. |
| Unspecified | Inherit the applicable default. With no inherited request, use ordinary ports. |
| `true` | Require offload-capable ports. Report a provisioning failure when the cloud cannot supply them; do not silently fall back to ordinary ports. |
| `false` | Request ordinary ports, overriding an inherited `true`. |
| Overrides | Use Juju's existing constraint precedence for model defaults and more specific application or machine requests. |
| Existing machines | Changing a default does not reconfigure existing interfaces. Explicit placement must account for the machine's existing provisioning choice. |
| Tests | Defer test additions and execution for this prototype iteration. Build and static checks are permitted. |

The intended usage on a controller built from this prototype is:

```text
# Default for newly provisioned machines in this model.
juju set-model-constraints network-offload=true

# Request offloading for a particular application.
juju deploy <charm> --constraints network-offload=true

# Override an enabled model default for another application.
juju deploy <charm> --constraints network-offload=false

# Request it when adding a machine directly.
juju add-machine --constraints network-offload=true
```

A controller's own machine constraints do not provide a default for every
hosted model. A controller-wide default across models remains a separate
design question; the prototype does not introduce one.

## Local Sunbeam environment and validation strategy

The available local OpenStack environment is the LXD VM named `sunbeam`.
Read-only inspection during this discussion found OpenStack 2024.1, two virtio
network interfaces, and no SR-IOV-capable PCI devices exposed to the VM.

That environment is useful for developing the Juju integration, but its current
configuration cannot supply the virtual functions required by the documented
offload path. The distinction between possible local checks and hardware
validation is:

| Area | Local Sunbeam VM | Offload-capable cloud |
| --- | --- | --- |
| Neutron acceptance and retention of port attributes | Can be investigated, subject to extensions and permissions. | Confirms the target cloud's API contract. |
| Provisioning with explicitly created ordinary ports | Can exercise shared lifecycle behaviour. | Also applicable. |
| Failure reporting and resource cleanup | Can exercise unsupported or unavailable offload cases. | Needed for realistic hardware allocation and binding failures. |
| Successful provisioning with offloaded ports | Not available with the VM's current devices. | Required. |
| Connectivity, security-group enforcement and actual offloading | Ordinary networking can be checked locally. | Required for the offloaded datapath. |

OpenStack also documents
[PCI and SR-IOV testing with an emulated Intel igb NIC](https://docs.openstack.org/nova/latest/contributor/testing/pci-passthrough-sriov.html).
This can provide additional PCI/SR-IOV coverage without special host hardware.
It does not establish an equivalent of the OVN switchdev datapath, and adapting
it to this LXD/Sunbeam installation would be additional work. We have not
selected or configured that approach.

The agreed direction is to develop with the existing environment and obtain
access to a suitable cloud for the final provisioning and networking checks.
The Sunbeam API and a live prototype deployment have not yet been exercised.

## Current implementation

The implementation carries the nullable boolean through constraint parsing,
JSON/YAML representations, model/application/machine persistence, provisioning
reads, status, and the current export, migration-import and recovery
representations. Preserving unset separately from explicit `false` is necessary
for inheritance and overrides. Older imported models leave the field unset.

The OpenStack offload path selects networks or subnets using the existing
model configuration and space-binding logic. It creates the ports before
booting, requests `direct` and `switchdev`, enables port security, and assigns
Juju's security-group IDs directly to the ports. It checks that Neutron's
response retains the requested attributes. Nova receives port IDs without
network IDs or a separate security-group assignment for those ports.

Ports have model and machine ownership in their names. Failure cleanup attempts
to remove ports created by that attempt before removing the machine security
group, and leaves ports attached to a live instance alone. Subsequent attempts
can recover matching unbound ports. Instance termination captures owned port
IDs before Nova detaches them, then deletes the server and removes the ports,
with bounded retries for conflicts while deletion completes. Model destruction
also attempts to remove remaining unbound offload ports.
The termination ordering change also affects existing Juju-owned ports created
for space-aware networking.

The following are implementation choices in this first prototype, rather than
additional requirements approved independently:

- Networks with port security disabled are rejected for offloaded provisioning.
- Neutron is required; OpenStack GBP and an explicit `virt-type=lxd` request are
  rejected for this path. Container placement with `network-offload=true` is
  also rejected.
- Other real providers reject `network-offload=true` through their constraint
  validators. Explicit `false` remains acceptable.
- Existing-machine placement checks the recorded machine provisioning
  constraint. It does not inspect the live NIC or prove hardware offloading.
  A machine without a recorded offload request cannot satisfy `true`.
- The Goose version in this checkout lacks the port-binding fields. A local
  OpenStack request type supplies those fields through its authenticated
  request client; no dependency version was changed.

The implementation changes the development 4.1 base schema. Use a fresh
prototype controller; an in-place upgrade from an existing beta database has
not been implemented or validated for this change.

Relevant source locations:

- [Constraint definition and parsing](core/constraints/constraints.go).
- [Core/domain conversion](domain/constraints/constraints.go).
- [Constraint schema](domain/schema/model/sql/0014-constraint.sql).
- [Existing-machine placement](domain/machine/state/placement.go).
- [Offloaded port creation and attempt cleanup](internal/provider/openstack/offload.go).
- [Instance provisioning and termination](internal/provider/openstack/provider.go).
- [Constraint reference](docs/reference/constraint.md).

## Validation completed

- `make go-build` passed, including the Juju client and machine agent. Binaries
  are in `_build/linux_amd64/bin/`.
- Schema, export/import and recovery generation completed. The client and
  agent API schemas match independent regeneration from the working source.
- Focused `go vet -tags=libsqlite3` initially reported existing unkeyed struct
  literals. The same package set passed with `-composites=false`.
- `gci`, `gofmt` and `git diff --check` completed successfully.
- No test files were changed and no unit or integration tests were executed.

`STATIC_ANALYSIS=1 make pre-check` stopped at its schema check because that check
compares generated schemas against committed `HEAD`. The working schemas
contain the new constraint, while the changes remain uncommitted. Full lint is
also unavailable because `golangci-lint` is missing; the static runner reported
that licence analysis was skipped because `go-licenses` is missing.

These results establish buildability and limited static validation. They do not
establish successful cloud provisioning, security-group enforcement, cleanup
under real failures, migration/recovery behaviour, or hardware acceleration.

## Outstanding decisions and next validation

The first live checks should establish that the target credentials can request
and read the binding attributes, that an instance reaches an active and
reachable state, and that each requested port has the expected binding and
security groups. Connectivity and firewall changes must work on the offloaded
interfaces, and the cloud's flow diagnostics should confirm actual offloading.

Lifecycle validation should cover multi-interface and space-bound machines,
explicit `false` overriding a model default, unavailable hardware, partial port
creation, failed server creation, controller interruption, and repeated
termination. Port and security-group cleanup must be checked at the provider
after each failure or removal.

Per-port selection, a controller-wide default for hosted models, conversion of
existing interfaces, and richer discovery of offload capability on manually
created machines remain outside the agreed prototype. Automated coverage and
production hardening can follow once the prototype's cloud behaviour and scope
have been reviewed.

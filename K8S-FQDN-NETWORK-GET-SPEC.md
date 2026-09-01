# Kubernetes Unit FQDNs for `network-get`

## Summary

Persist Kubernetes unit DNS names in Juju's existing `fqdn_address` model tables
and expose them only through the `network-get` hook tool path.

Today, CAAS unit address reporting stores the pod IP through the cloud-container
`ip_address` path. The schema already has `fqdn_address` and
`net_node_fqdn_address`, but current unit-network reads do not use those tables.
This spec activates those tables for one narrow consumer: the uniter
`NetworkInfo` facade used by `network-get`.

The Kubernetes provider must keep reporting pod IPs as IP addresses. It must not
overload the existing CAAS unit `Address` field with a DNS name, because that
field flows through IP/subnet-oriented persistence. Instead, Kubernetes FQDNs are
recorded through a separate FQDN path and linked to the unit's existing
`net_node`.

## Goals

- Store stable Kubernetes pod DNS names for CAAS units in `fqdn_address`.
- Link each stored FQDN to the unit's `net_node` using `net_node_fqdn_address`.
- Expose those FQDNs only via the uniter `NetworkInfo` path used by
  `network-get`.
- Prefer the unit FQDN for `network-get --bind-address` and fallback
  `--ingress-address` on CAAS units when an FQDN is available.
- Preserve existing pod-IP persistence and existing non-`network-get` address
  APIs.
- Use explicit address provenance to set `fqdn_address.scope_id`.

## Non-Goals

- Changing controller API address publication.
- Changing `apiaddresssetter`, `controller_api_address`, or
  `GetControllerAPIAddresses`.
- Broadly activating `fqdn_address` for all unit address APIs.
- Replacing pod IPs in status, relation network state, or existing
  cloud-container address storage.
- Changing IAAS unit address behavior.
- Inferring scope from DNS suffixes such as `.svc` or `.cluster.local`.

## Current State

The relevant current path is:

1. The Kubernetes provider reports `caas.Unit.Address` as `p.Status.PodIP`.
2. The CAAS application provisioner sends that value as
   `UpdateCAASUnitParams.Address`.
3. The application state layer wraps the value in `network.NewSpaceAddress` and
   writes it through the cloud-container `ip_address` path.
4. Network reads behind `network-get` use `domain/network` state queries based on
   `v_unit_relation_network`, which currently reads IP address data.

The existing schema includes:

- `fqdn_address(uuid, address, scope_id)`.
- `net_node_fqdn_address(net_node_uuid, address_uuid)`.

Those tables are exported, imported, and removed with model data, but they are
not currently part of the `network-get` read path.

## Address Scope Contract

`fqdn_address.scope_id` must be set from the source of the DNS name, not from
the string shape.

Use existing Kubernetes provider precedent:

- Headless Service pod DNS: `local-cloud` (`scope_id = 1`).
- ClusterIP Service DNS: `local-cloud` (`scope_id = 1`).
- LoadBalancer ingress hostname: `public` (`scope_id = 2`).
- ExternalName Service hostname: `public` (`scope_id = 2`).
- NodePort external hostname or external IP: `public` (`scope_id = 2`).

For this change, the intended writer is the headless-Service pod DNS path, so it
must write `scope_id = 1`.

Do not infer scope from the DNS suffix. Kubernetes cluster DNS domains are
configurable, and Juju's address model treats both internal and external DNS
names as hostnames unless the writer supplies scope explicitly.

## Proposed Solution

### 1. Extend CAAS Unit Address Reporting

Add a separate FQDN field to the CAAS unit update path, conceptually:

```go
type UpdateCAASUnitParams struct {
    Address     *string // existing pod IP path
    FQDNAddress *ScopedFQDNAddress
}

type ScopedFQDNAddress struct {
    Value string
    Scope network.Scope
}
```

The exact type names are not prescribed. The important contract is that FQDN
values and scope travel separately from the existing pod-IP field.

For Kubernetes StatefulSet units, the provider should report the stable pod DNS
name associated with the unit, for example:

```text
<pod-name>.<headless-service>.<namespace>.svc
```

The writer must set this as `local-cloud`.

The existing `Address` field remains the pod IP and continues to write through
the existing `ip_address` path.

### 2. Persist FQDNs on the Unit Net Node

In the application state update path for CAAS units:

- Upsert the FQDN into `fqdn_address`.
- Upsert the link into `net_node_fqdn_address` for the unit's existing
  `net_node_uuid`.
- Preserve the existing pod IP write to `ip_address`.

Use idempotent writes:

- Repeated updates with the same FQDN must not duplicate rows.
- Updating the FQDN must leave only the current FQDN linked to the unit net node
  for this source.
- Removing or omitting an FQDN should be explicitly defined. For StatefulSet pod
  DNS, retaining the last value is acceptable because the identity is stable,
  but the implementation must make that choice visible in tests.

### 3. Activate FQDNs Only for `NetworkInfo`

Include FQDN rows only in the state queries behind uniter `NetworkInfo`:

- `GetUnitEndpointNetworkInfo`.
- `GetUnitNetworkInfo`.

Do not change broader address views or APIs such as:

- `v_all_unit_address`.
- `GetUnitAndK8sServiceAddresses`.
- `GetControllerAPIAddresses`.
- controller API address publication paths.

The read implementation may use either:

- private query additions in the two `NetworkInfo` state methods, or
- a private view used only by those methods.

The result should produce ordinary `UnitAddress` values whose address value is
the FQDN and whose type is hostname.

### 4. Ordering for `network-get`

`network-get --bind-address` returns the first address in the first network
info result. `network-get --ingress-address` can also fall back to the first
address when no explicit ingress address is selected.

For CAAS units, order addresses so FQDNs are preferred when present:

1. FQDN addresses linked to the unit net node.
2. Existing provider pod IP addresses.
3. Other current addresses.

This ordering applies only to the `NetworkInfo` result used by `network-get`.

Do not use FQDN values to calculate egress subnets. Egress subnets should remain
derived from configured relation/model settings or IP addresses.

## User-Visible Behavior

For a CAAS unit with a recorded Kubernetes FQDN:

- `network-get <binding> --bind-address` returns the unit FQDN.
- `network-get <binding> --ingress-address` returns the selected ingress
  address as today, but if it falls back to the address list, the FQDN is chosen
  before the pod IP.
- `network-get <binding>` includes the FQDN in the interface/address list for
  that binding.

For IAAS units and CAAS units without a recorded FQDN, behavior remains
unchanged.

## Compatibility

This is a CAAS-only behavior change for `network-get`.

Existing pod-IP storage remains intact, so consumers outside uniter
`NetworkInfo` continue to see current behavior. Charms that inspect the complete
`network-get` output may see an additional hostname address or a hostname
preferred over the pod IP for bind-address selection.

No facade schema change is required if the existing `NetworkInfo` wire type can
represent hostname addresses, which it already should through address strings
and hostname address type handling.

## Failure Handling

- If the provider cannot determine a unit FQDN, do not fail the CAAS unit
  update. Continue storing the pod IP as today.
- If FQDN persistence fails after pod-IP persistence would have succeeded, the
  update should fail transactionally rather than partially committing ambiguous
  network state.
- If `NetworkInfo` sees malformed or unusable FQDN data, it should ignore that
  FQDN row and continue returning existing IP-based results rather than breaking
  hook execution.

## Test Matrix

- CAAS unit update writes pod IP to `ip_address` and FQDN to `fqdn_address`.
- CAAS unit update links the FQDN to the unit's `net_node` via
  `net_node_fqdn_address`.
- Repeated CAAS unit updates with the same FQDN are idempotent.
- Updating a unit FQDN replaces the active linked FQDN for that unit net node.
- Headless-Service pod DNS is persisted with `scope_id = local-cloud`.
- A simulated LoadBalancer or ExternalName FQDN, if supported by the same helper,
  is persisted with `scope_id = public`.
- `GetUnitEndpointNetworkInfo` includes FQDNs only for the targeted
  `NetworkInfo` path.
- `GetUnitNetworkInfo` includes FQDNs for the no-provider-networking fallback.
- `network-get --bind-address` returns the FQDN before the pod IP for CAAS.
- `network-get --ingress-address` fallback prefers the FQDN when no explicit
  ingress address exists.
- Existing `GetUnitAndK8sServiceAddresses` and `GetControllerAPIAddresses` tests
  remain unchanged and do not start returning FQDN rows.
- IAAS `network-get` behavior remains unchanged.

## Open Questions

- What exact Kubernetes object should be the source of the headless-Service name
  for each unit FQDN? The implementation should avoid reconstructing names from
  ad hoc string conventions when provider state already has the governing
  Service name.
- Should omission of a previously recorded StatefulSet FQDN retain or delete the
  old row? Retention is defensible for stable pod identity, but the final choice
  must be explicit.
- Should `fqdn_address` links record source/provenance in a new table or
  column? This spec avoids schema expansion and relies on the CAAS writer owning
  the unit FQDN link, but future multi-source FQDN writes may need provenance.

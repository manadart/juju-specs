# Dual-stack networking findings

Date: 2026-03-05

## Scope reviewed

- `core/network`
- `domain/network` (`service` and `state`)
- related controller/API address prioritization in `domain/controllernode`
- adjacent call sites consuming network prioritization

## Executive summary

Juju networking logic is broadly dual-stack capable at representation/storage level, but the active selection/prioritization logic is intentionally IPv4-first in multiple core and service paths. The biggest practical effect is that when both IPv4 and IPv6 are available, many selectors return IPv4 first (or only one family bucket), while some paths preserve both families in priority order.

Spaces behavior is largely address-family agnostic; constraints/bindings operate on spaces/subnets and not directly on IP family.

## Core prioritization model (IPv4-first)

- `core/network/address.go` defines `ScopeMatch` tiers with explicit IPv4-preferred buckets:
  - `exactScopeIPv4`, `firstFallbackScopeIPv4`, `secondFallbackScopeIPv4`
  - See: `core/network/address.go:106`
- Match hierarchy checks IPv4 buckets first:
  - `scopeMatchHierarchy()` order starts with `exactScopeIPv4`, then `exactScope`, etc.
  - See: `core/network/address.go:1015`
- Public/cloud-local matchers encode IPv4 preference:
  - `ScopeMatchPublic`: `core/network/address.go:789`
  - `ScopeMatchCloudLocal`: `core/network/address.go:824`
- General sort order also puts IPv4 ahead of IPv6 within scope buckets:
  - `SortOrderMostPublic` increments order for IPv6
  - See: `core/network/address.go:182`, `core/network/address.go:208`

## Important selector behavior split

Two selector families have different behavior:

- `indexesForScope` returns only the first non-empty match bucket:
  - can collapse to a single family bucket
  - `core/network/address.go:974`
- `indexesByScopeMatch` returns all match buckets in priority order:
  - keeps both families if present, ordered
  - `core/network/address.go:988`

This distinction is key when assessing dual-stack outcomes at call sites.

## Where `domain/network` consumes this

### Unit/machine address APIs

- Unit private address:
  - matches `ScopeMatchCloudLocal`, then returns first after sort
  - `domain/network/service/unitaddress.go:25`
- Unit public addresses:
  - matches `ScopeMatchPublic`, sorted, returned
  - `domain/network/service/unitaddress.go:78`
- Unit public single address:
  - first element of public address list
  - `domain/network/service/unitaddress.go:62`
- Machine public/private single address:
  - uses `OneMatchingScope` (`ScopeMatchPublic` / `ScopeMatchCloudLocal`)
  - `domain/network/service/machineaddress.go:31`
  - `domain/network/service/machineaddress.go:51`

Net effect: these paths inherit core IPv4-first semantics directly.

### Endpoint network info

- `buildUnitNetworkFromAddresses`:
  - ingress uses `AllMatchingScope(ScopeMatchCloudLocal).Values()`
  - `domain/network/state/unitinfo.go:133`
- For providers not supporting networking, `GetUnitNetwork` is reused for every endpoint:
  - `domain/network/service/unitinfo.go:82`

Potential effect: ingress lists may collapse to the first scope-match bucket rather than preserving a balanced dual-stack set.

### Controller unit API addresses in network service

- `GetControllerAPIAddresses` only removes machine-local addresses; it does not sort/prioritize itself:
  - `domain/network/service/unitaddress.go:106`
- Any later prioritization is done by downstream consumers.

## Spaces behavior (mostly family-agnostic)

- Space constraints, app bindings, and subnet moves are space/subnet based:
  - space requirements and device selection:
    - `domain/network/service/container.go:157`
    - `domain/network/state/container.go:19`
    - `domain/network/state/container.go:132`
  - subnet move validation and enforcement:
    - `domain/network/state/movesubnets.go:20`
    - `domain/network/state/movesubnets.go:91`
    - `domain/network/state/movesubnets.go:163`

These flows reason about spaces/subnets, not explicit IPv4-vs-IPv6 preference.

## Storage/query and determinism notes

- Address query views do not impose ordering:
  - `v_ip_address_with_names`: `domain/schema/model/sql/0021-network.sql:351`
  - `v_all_unit_address`: `domain/schema/model/sql/0021-network.sql:421`
- State address fetchers also have no `ORDER BY`:
  - `domain/network/state/address.go:17`
  - `domain/network/state/unitaddress.go:29`
  - `domain/network/state/unitaddress.go:66`
- Some service fallbacks return "first address" when no scope match:
  - `domain/network/service/unitaddress.go:43`

This can make fallback behavior sensitive to DB row order.

## Adjacent consumers that repeat the same bias

- `core/network/hostport` prioritization uses the same match hierarchy:
  - `PrioritizedForScope`: `core/network/hostport.go:74`
  - `AllMatchingScope`: `core/network/hostport.go:314`
- Agent config API hostport ordering for cloud-local:
  - `agent/agent.go:688`
- Bootstrap controller-url public address fallback:
  - `internal/bootstrap/deployer.go:427`
- Cross-model relation egress fallback picks first public match per unit:
  - `domain/crossmodelrelation/service/service.go:535`

## Controller-node domain has duplicated IPv4-first logic

`domain/controllernode` defines its own `ScopeMatch` model and hierarchy mirroring core behavior:

- matching and hierarchy:
  - `domain/controllernode/types.go:86`
  - `domain/controllernode/types.go:115`
  - `domain/controllernode/types.go:146`
  - `domain/controllernode/types.go:194`
- service usage:
  - agents by controller: `domain/controllernode/service/service.go:270`
  - all agents: `domain/controllernode/service/service.go:287`
  - all clients (public): `domain/controllernode/service/service.go:330`
  - clients-by-controller currently use cloud-local matcher:
    - `domain/controllernode/service/service.go:353`
    - `domain/controllernode/service/service.go:361`

## Other dual-stack-relevant observations

- Observed network discovery still skips IPv6 link-local addresses:
  - `core/network/discovery.go:148`
- Default route detection parses `ip route show` output:
  - `core/network/gateway.go:19`
  - no explicit `-6` handling in this path
- Placeholder subnets for k8s dual-stack are explicit (`0.0.0.0/0` and `::/0`):
  - `core/network/subnet.go:20`
  - `domain/network/service/migration.go:242`
- Netconfig reconciliation creates `/32` or `/128` subnets where needed:
  - `domain/network/state/netconfig.go:455`

## Test posture observed

- Core has explicit tests asserting IPv4-over-IPv6 behavior:
  - `core/network/address_test.go:409`
  - sort expectation comments: `core/network/address_test.go:790`
- `domain/network` has comparatively less direct IPv4-vs-IPv6 selector coverage in service/state paths.
- `domain/controllernode` also tests IPv4 preference explicitly:
  - `domain/controllernode/types_test.go:229`

## Current interpretation

1. Storage/modeling supports dual-stack inputs and subnets.
2. Runtime selection is intentionally biased toward IPv4 in core and controller-node layers.
3. Space mechanics are orthogonal to family preference, but many consumer paths that select representative addresses are not family-neutral.
4. There are a few deterministic-order and fallback semantics that can impact consistency when scope matching fails or when both families exist.


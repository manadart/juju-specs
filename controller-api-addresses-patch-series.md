# Controller API addresses: proposed patch series

Plan prepared against Juju `4.1` at
`c9f6752833858c81acc5dd40bd3832e08374c213` (`4.1-beta3`). This is the
same baseline used by the [agreed specification](https://github.com/juju/juju-agent-specs/blob/2121f49d2a3a2f53fa9534832236017c55e21b05/2610/JUJU-8694/controller-api-addresses-4.1-plan.md).
PR #23231 is reference material; the series starts from this baseline.

The resulting design has one projection for ordinary API discovery. Its writer
selects and orders client and agent endpoints; readers retrieve that decision.
Controller-specific consumers resolve the relevant controller-model network
facts on demand. Long-lived consumers refresh managed connections when those
facts change.

Propose nine patches. Patches 1–7 establish sources and move consumers that
cannot use ordinary discovery. Patch 8 prepares unused projection machinery.
Patch 9 activates its writer and all discovery readers in one landing. Each
patch includes its relevant tests.

| Patch | Proposed title | Depends on | Reviewable result |
| --- | --- | --- | --- |
| 1 | Preserve Kubernetes Service IP and DNS address sources | Baseline | Live persistence and legacy import represent the same Service facts |
| 2 | Reconcile the controller API Service through its full lifecycle | 1 | Bootstrap and provider events track `controller-service` correctly |
| 3 | Add controller network queries and watches for targeted consumers | Baseline | Typed source queries and restricted factory access, without changing discovery |
| 4 | Resolve API remote callers from controller network state | 3 | Object-store peers and other remote-caller users connect to the named node |
| 5 | Separate controller diagnostics from ordinary API discovery | 3 | IAAS targeting works; Kubernetes HA retrieval fails explicitly |
| 6 | Resolve reverse-SSH targets independently of API discovery | 3 | Node-specific SSH routing has an explicit reachability contract |
| 7 | Reconcile certificate identities from typed network sources | 1, 2, 3 | Certificate maintenance no longer reads discovery addresses |
| 8 | Add the flat API endpoint projection and publication policy | 1, 3 | Tested schema, policy and persistence, unused by running discovery |
| 9 | Publish and consume the new API endpoint projection | 2, 4–8 | Bootstrap, publisher and every ordinary reader use the same stored policy |

Implementation status: patch 1 merged as [PR #23364](https://github.com/juju/juju/pull/23364).
Patch 2 is implemented, uncommitted, on
`4.1-k8s-controller-service-lifecycle`, based on `4.1` at `2a07b4e4ea`.
Patches 3–9 remain planned. Patch 2 validation is recorded below.

The dependencies permit separate reviews; they do not imply separate release
milestones. In particular, ship patches 8 and 9 together. If maintaining unused
schema between reviews is inappropriate for the landing process, combine them.

## 1. Preserve Kubernetes Service IP and DNS address sources

Use the existing `k8s_service` → `net_node` relationships, IP tables, and
`fqdn_address` / `net_node_fqdn_address`. Keep source facts separate from API
publication policy.

- Persist Service IPs and hostnames with their original scopes. Generate new
  UUIDs in services and pass them to state; do not send hostnames through subnet
  lookup or IP parsing.
- Make replacement, scope changes, deletion and recreation reconcile both
  address forms. Unlink obsolete FQDN associations and delete orphan records
  without deleting a hostname still owned by another net node.
- Preserve scope per source by making FQDN identity the pair of hostname and
  scope. Net nodes share rows when both agree; scope changes must not mutate
  another owner's address.
- Apply the same representation to cloud-Service migration imports.
- Establish change notifications for all source tables/links introduced into
  the read contract; the composed controller watchers follow in patches 2–3.

Primary areas: [application service/state](domain/application/),
[network import](domain/network/modelmigration/import_cloudservice.go),
[network state](domain/network/state/cloudservice_import.go), and
[model schema/triggers](domain/schema/model/).

Acceptance: IPv4, IPv6, public LB hostname, unchanged text with changed scope,
IP↔hostname replacement, shared hostname ownership, delete/recreate, and legacy
import. Existing IP-only behaviour continues to work.

## 2. Reconcile the controller API Service through its full lifecycle

- Add/use a controller API Service lookup for the normal `controller-service`.
  Watch that exact Service identity, including creation, ingress changes and
  deletion. Preserve the separate StatefulSet/pod selectors.
- Bootstrap persists the actual normal Service source data. Stop representing
  `controller-0` or a headless per-pod address as that Service.
- Carry ordinary API bootstrap addresses separately from the per-node identity
  used for Dqlite. Preserve the latter throughout this series.
- Where normal Service DNS is supplied, construct it from supported provider
  namespace, Service and cluster-domain information. Do not invent an alias or
  silently substitute headless pod DNS.
- Retain current runtime discovery until patch 9; this patch repairs its source
  data and bootstrap acquisition rather than introducing a second publisher.

Primary areas: [Kubernetes provider](internal/provider/kubernetes/k8s.go),
[application watcher](internal/provider/kubernetes/application/application.go),
[Kubernetes bootstrap](internal/bootstrap/deployer_k8s.go), and
[bootstrap address finder](internal/worker/bootstrap/addressfinder.go).

Acceptance: changing only LB ingress, with no pod event, triggers persistence;
Service replacement/deletion is observed; bootstrap accepts LB IP and hostname;
Dqlite still receives the intended stable per-node identity. Use the Kubernetes
fake clientset and reactors for provider interaction tests.

Implementation: controller address reads and the Service watcher select
`controller-service` by name, independently of workload and replica selectors.
Bootstrap records that Service's UID and scoped addresses; the controller pod
ID and stable Dqlite FQDN remain separate. Runtime reconciliation retains the
ClusterIP alongside external controller Service addresses. A confirmed missing
Service clears its stored IPs and FQDN associations, retaining the Service/net
node for recreation. Provider failures and missing workload/status resources do
not clear the snapshot. Bootstrap no longer substitutes loopback for missing
Service addresses.

Validation: full `-race` package tests passed for
`internal/provider/kubernetes`, its `application` and `utils` packages,
`internal/bootstrap`, `internal/worker/bootstrap`,
`internal/worker/caasapplicationprovisioner`, and
`domain/application/{service,state}`. Bounded stress passed 8 runs of the
controller Service suite and 20 runs of the application provisioner suite,
with no failures. CLI, agent and agent-bootstrap dependent packages compiled
with `-race`. Generated mocks, `gci`, and whitespace checks are current.
`make pre-check` skipped static analysis by default; live Kubernetes validation
of patch 2 has not run.

## 3. Add controller network queries and watches for targeted consumers

Extend the existing network-domain implementation rather than building another
address cache or a general-purpose routing framework.

- Define typed IP/DNS results for controller-peer and external node-targeted
  queries. Keep their purpose and eligibility rules explicit. Reuse private
  source-reading helpers where appropriate; do not make one policy serve every
  consumer.
- Resolve controller membership to the actual controller unit in the controller
  model. Validate the relationship and lifecycle. Specify treatment of Dying
  units for each purpose; reject unrelated, Dead and removed targets.
- Compose membership and network watches covering unit/net-node reassociation,
  addresses, FQDN links and relevant device/subnet/space policy inputs. A watcher
  filtered to the original net-node IDs must rebind when those IDs change.
- Observe subscription readiness before querying initial state. Distinguish
  authoritative emptiness from unavailable/not-yet-ready source data.
- Extend the restricted object-store service factory with the narrow network
  capability. Resolve the actual controller-model UUID through controller
  metadata; never use the `ControllerModelName` sentinel as its DB namespace.
- Keep cross-database coordination in workers. Do not add a dependency from the
  restricted factory to the full domain-services worker or provider services.

Primary areas: [network service](domain/network/service/unitaddress.go),
[network state](domain/network/state/unitaddress.go),
[unit address watching](domain/application/service/service.go),
[restricted services](domain/services/objectstore.go),
[factory worker](internal/worker/objectstoreservices/), and
[service interfaces](internal/services/interface.go).

Acceptance: correct controller mapping on IAAS/CAAS, management-space cases,
relationship/lifecycle validation, network reassociation, empty initial events,
changes during watcher startup, deterministic watcher shutdown, and factory
construction against the controller-model DB without the object-store cycle.

## 4. Resolve API remote callers from controller network state

- Replace `apiremotecaller`'s reads/watches of published agent addresses with
  the peer query and source watches from patch 3.
- Re-resolve after membership/network changes and maintain connections keyed
  by controller identity. Close removed/replaced connections deterministically.
  An authoritative empty peer set retires stale connections; a failed read does
  not masquerade as that result.
- Preserve object-location hints and object retrieval through managed
  connections. No source query is added to every blob request.
- Preserve the controller CA and fixed `juju-apiserver` TLS verification name
  for API and blob HTTP connections when dial addresses change.
- Check other users of the remote caller, including controller presence, as
  part of the same change.

Primary areas: [API remote caller](internal/worker/apiremotecaller/),
[object store](internal/worker/objectstore/),
[controller presence](internal/worker/controllerpresence/), and factory wiring.

Acceptance: retrieve a blob present only on one peer; replace that peer's pod
and reconnect to the correct controller; remove a node and close its connection;
preserve presence behaviour. Test routing via IP/DNS using a trusted certificate
with `juju-apiserver` and no routing-address SAN. Reject a wrong CA or missing
verification identity. Exercise worker startup and retry ordering under stress.

## 5. Separate controller diagnostics from ordinary API discovery

- Make IAAS `ControllerDetails` obtain current controller-ID/client-address
  mappings from the purpose-specific source query. Preserve client eligibility
  with and without a management space; agent eligibility is not an exclusion.
- Define the Kubernetes HA restriction using controller-hosting substrate and
  membership. Do not infer it from the workload model, number of addresses or
  currently reachable nodes.
- Enforce the restriction after authentication and before data delivery in
  both the debug-log websocket handler and `Client.StatusHistory`.
- Update client/command handling and `ControllerDetails` consistently. Use a
  distinguishable result so the existing generic `NotSupported` fallback cannot
  silently produce single-node output. Server checks must protect older clients.
- Preserve single-controller Kubernetes and IAAS retrieval, log ingestion, and
  status-history recording. Single-controller Kubernetes can use the connected
  controller; do not manufacture per-node Service mappings.
- Terminate an existing debug-log stream when expansion makes the controller
  Kubernetes HA. Own topology monitoring through the established worker and
  cancellation lifecycle, without starting unmanaged handler goroutines.

Primary areas: [HighAvailability facade](apiserver/facades/client/highavailability/),
[debug-log handler](apiserver/debuglog.go),
[StatusHistory handler](apiserver/facades/client/client/status.go), relevant API
clients/commands, and a narrow domain topology/capability query.

Acceptance: both commands, direct requests, older-client fallback, a Kubernetes
controller hosting an IAAS model, unavailable HA members, 1→3 transition during
a stream, and single-node/IAAS success. Verify auth precedes the restriction and
that recording continues.

## 6. Resolve reverse-SSH targets independently of API discovery

- Replace stripping API ports from general agent endpoints with a query for
  addresses that reach the controller holding the waiting SSH session.
- Preserve IAAS target identity and reachability policy.
- Return an explicit unsupported result for Kubernetes cases without a
  node-specific transport. Neither a shared Service nor an off-cluster-invisible
  pod IP satisfies this contract.
- Remove the obsolete general-discovery dependency from this worker and its
  interfaces. Designing an external Kubernetes node-specific transport remains
  separate work.

Primary area: [SSH tunneler](internal/worker/sshtunneler/), its service wiring and
the corresponding API/CLI error propagation.

Acceptance: IAAS session reaches the selected controller; changed/departed
targets are handled; unsupported Kubernetes routing gives an actionable error
without advertising a misleading destination.

## 7. Reconcile certificate identities from typed network sources

- Define the IP and DNS identities actually needed by consumers that verify a
  dialled address. Read those facts through a purpose-specific network query.
- Replace the cloud-local discovery accessor and its IP-only parsing in the
  certificate path. Do not add every peer routing address to certificates by
  default; patch 4 preserves the existing fixed peer TLS identity.
- Watch the source inputs to this query, including relevant Service and unit
  changes, and cross the readiness barrier before the first certificate query.
- Preserve existing certificate ownership and update behaviour while removing
  the dependency on ordinary API discovery.

Primary areas: [certificate updater](internal/worker/certupdater/), network
service/state and associated worker wiring.

Acceptance: required IPv4/IPv6/DNS SANs, address replacement, source failures,
readiness races, certificate regeneration and verification. Distinguish these
tests from the fixed-name peer TLS tests in patch 4.

## 8. Add the flat API endpoint projection and publication policy

Add `controller_api_endpoint` with audience, opaque endpoint group, address,
address type, original scope and priority as specified. The group preserves
wire grouping; it is not a controller identity API. A shared Service group has
no dependency on controller/0.

- Put selection/ordering policy in domain services and transactional reads and
  writes behind state interfaces. Return one canonical typed, ordered view to
  adapters. Reads select by audience and preserve stored order; they do not
  redo scope selection or fallback.
- Kubernetes clients get the preferred external Service tier, with private
  Service fallback when needed. Agents get preferred internal Service endpoints
  while retaining external fallback. Generic discovery excludes pod/headless
  endpoints, and source pod scopes remain unchanged.
- IAAS preserves per-controller grouping, client selection and management-space
  policy for agents. An endpoint may qualify for both audiences.
- Replace both audiences atomically. Compare all endpoint metadata, not just
  text. Publish authoritative emptiness; retain the last successful snapshot
  on source-read failure.
- Add generated triggers, schema tests and export types/readers alongside the
  schema. Keep all production discovery on the old path for this patch.

Primary areas: [controller-node domain](domain/controllernode/),
[controller schema](domain/schema/controller/sql/0010-controller-node.sql),
[trigger registration](domain/schema/controller.go), and
[controller export](domain/export/).

Acceptance: table-driven policy matrix for IAAS and all Kubernetes Service
address forms; grouping, priority and deterministic ordering; metadata-only
changes; atomic audience replacement; no-op publication; empty/failing snapshots;
schema/export consistency. Read APIs must work from the projection alone.

## 9. Publish and consume the new API endpoint projection

This is the coordinated activation patch. Keeping publisher and readers
together prevents an intermediate build from interpreting new data using old
policy or serving the new empty table while the old writer remains active.

- Rework the existing primary-controller `apiaddresssetter` to read membership,
  normal Service or controller-unit sources, and effective policy inputs, then
  publish through patch 8. Reuse the same policy for bootstrap seeding.
- Watch membership, Service/address associations, IP/FQDN and relevant network
  policy/configuration changes. Cover initial readiness and watch rebinding.
  Coalesce events, rebuild on restart/primary handover, and retry failed
  publication even when no further source event arrives.
- Read a coherent snapshot within each source DB where possible. Treat the
  cross-DB pipeline as eventual convergence, with atomic projection publication.
  Prevent an in-flight snapshot from reintroducing a removed controller; retain
  single-primary ownership and cancellation throughout handover.
- Establish valid sources and initial publication through the supported
  bootstrap/restore/upgrade lifecycle before activating new discovery. This
  readiness dependency must not reintroduce the object-store startup cycle.
- Move every ordinary reader and adapter together: login, APIAddresser, agent
  provisioning and reconnect, tools/proxy/no-proxy, migration, offer consumption
  and cross-controller registration. Preserve wire shapes with the canonical
  ordered view rather than reimplementing selection in each adapter.
- Explicitly change `GetSourceControllerInfo` and the controller-info query
  used by `GetConsumeDetails`; they read SQL directly and must select the client
  audience. Their tests must exercise those real readers.
- Remove old getters, writer paths, watcher registrations and table references,
  including controller-removal cleanup, schema and generated export artifacts.
  Remove unused interface methods and mocks as part of the same audit.

Primary areas: [publisher](internal/worker/apiaddresssetter/),
[machine manifolds](cmd/jujuagentd/agent/machine/manifolds.go), bootstrap,
[common addresses](apiserver/common/addresses.go),
[controller info](domain/controller/state/state.go),
[migration source info](domain/modelmigration/state/controller/state.go),
[controller removal](domain/removal/state/controller/controller_node.go), and
ordinary facade/worker consumers found by the call-site audit.

Acceptance: public Service and internal agent/pod endpoints deliberately differ
in fixtures, yet migration and offer consumption both advertise the selected
client endpoint. Verify off-cluster agent fallback, controller/0 removal without
losing the shared Service, late LB assignment, metadata changes, deletion,
publication failures, restart, handover and startup races. Every ordinary reader
must agree on policy and order. No production references to the old table remain
except any explicitly required historical schema/upgrade compatibility code.

## Release and validation boundaries

The current branch reports `4.1-beta3`. Its schema implementation restricts DDL
to the current major/minor, has an empty controller post-patch list, and documents
schema immutability constraints for patch/build releases. Plan patches 8–9 as
4.1 schema work, and verify the supported deployment transition before landing;
additive SQL alone is not evidence of an in-place or rolling-upgrade contract.
For supported existing installations, rebuild valid Service sources and the
projection through the supported transition. Do not translate old pod-derived
discovery rows into purported client Service endpoints. Schema/export changes
and source/projection initialization belong to the activation work, not a later
follow-up.

Each implementation patch uses `tc` tests and the required dqlite test setup.
Run changed concurrent packages with `-race`, and compile/race/stress changed
workers with a bounded timeout. Exercise readiness and event orderings, not just
data races. Regenerate mocks/schema/triggers/export artifacts where required,
run `gci` on touched non-generated Go files, and run `make pre-check` before
submission.

The activation patch also supplies or extends integration coverage for:

- Kubernetes HA with an actually off-cluster client and agent, separately
  exercising LB IP and hostname endpoints.
- Blob retrieval from a particular peer, pod replacement, Service replacement,
  controller restart/primary handover and scale transitions.
- IAAS management-space behaviour and reverse SSH.
- Migration source information and CMR offer/consume continuity.
- Both diagnostic restrictions and continued log/status recording.
- Independent Dqlite membership and recovery checks.

Use the existing controller, model/migration, CMR and SSH suites where they fit;
add focused fixtures for the off-cluster/hostname cases. Report unit, stress and
live integration results separately. The original plan was based on static
source inspection; implementation progress and validation are recorded above.

Charm/listener/firewall exposure changes and a new external per-node Kubernetes
transport remain separate workstreams, as in the specification. Runtime API-port
changes also require listener/Service coordination; this series must not claim
that publishing a different port implements that feature.

## Follow-up after the series: Service FQDNs in unit address watching

After patches 1–9, revisit the [review comment on PR #23364](https://github.com/juju/juju/pull/23364#discussion_r4092774107)
suggesting that `getNetNodeSpaceAddresses` include `fqdn_address` data for
`GetAddressesHash` and `WatchUnitAddressesHash`. The suggestion has merit, but
the intended behaviour needs checking before extending the query:

- Compare with 3.6 unit and cloud-Service address watching, including the
  resulting `network-get` and relation-network behaviour for hostname-only and
  mixed IP/DNS Service addresses.
- Work through network spaces and endpoint bindings: determine how scoped
  hostnames participate in address selection and hashing when there is no
  associated subnet/space. Make any fallback explicit.
- Trace both the hash inputs and watcher subscriptions. Including FQDNs in the
  query must be accompanied by notifications for relevant `fqdn_address` and
  `net_node_fqdn_address` changes, with the correct Service/net-node filtering.
- Once the behaviour is agreed, add hash and watcher tests for hostname-only
  additions, replacements, scope changes and removals, mixed IP/DNS addresses,
  space/binding changes, and unchanged results that should not notify.

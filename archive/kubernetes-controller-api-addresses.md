# Kubernetes controller API addresses

## Scope

This note records the controller API address behaviour on the `3.6` and
`main` branches, the implications of the Kubernetes HA work, and proposed
changes to restore the intended client and agent address preferences.

The source revisions inspected were:

- `3.6`: `8dcf07322b95`
- `main`: `5233df9d40a7`

No live controller was exercised. The findings below are based on source and
test inspection.

## Terminology

There are three different addressing requirements which should not be
conflated:

1. An **any-controller API endpoint** lets an agent or user client connect to
   any healthy API server. A load-balancing Kubernetes Service is appropriate
   for this purpose.
2. A **controller-node API endpoint** must reach one particular controller.
   Inter-controller workers and operations such as per-controller log
   collection require this property. A shared Kubernetes Service is not
   appropriate because it may select another controller pod.
3. A **Dqlite endpoint** must identify one particular controller with a stable
   address. On Kubernetes, this is the per-pod FQDN supplied by the headless
   Service.

## Findings

### Juju 3.6

Kubernetes controller API discovery has a CAAS-specific implementation in
`state/address.go`. It does not obtain API addresses from the controller unit
or its pod IP. Instead, it reads the normal controller Kubernetes Service from
the controller `CloudService`.

At bootstrap, `agent/agentbootstrap/bootstrap.go` calls `GetService` with
`includeClusterIP=true` and persists the resulting Service addresses. In
`internal/provider/kubernetes/utils/address.go`:

- a Service ClusterIP is classified as `local-cloud`;
- a load balancer ingress address is classified as `public`;
- when requested, the ClusterIP is retained alongside load balancer
  addresses.

The CAAS-specific reader then applies these policies:

| Consumer | LoadBalancer Service | ClusterIP-only Service |
|---|---|---|
| Agents | Normal Service FQDN, ClusterIP, then public LB address | Normal Service FQDN and ClusterIP |
| Clients | Public LB address only | ClusterIP as the public-scope fallback |

The normal Service FQDN added for agents is
`controller-service.controller-<controller-name>.svc.cluster.local`. It is not
a per-pod headless Service name.

The client behaviour depends on the old `AllMatchingScope` semantics. It
returns addresses from only the best available scope tier. Consequently, a
public load balancer address suppresses the `local-cloud` ClusterIP for
clients. The ClusterIP is returned only when no public address exists.

#### Pod IP scope in 3.6

CAAS unit pod IPs are explicitly persisted with `machine-local` scope in
`state/unit.go` and `state/application.go`. This scope is used by CAAS network
information to distinguish locally bindable pod/interface addresses from
Service ingress addresses. In particular,
`apiserver/facades/agent/uniter/networkinfocaas.go` treats machine-local
addresses as interface and binding addresses and excludes them from default
and relation ingress addresses.

This classification would also make pod IPs ineligible for the public and
cloud-local API scope matchers. However, it is not the primary reason pod IPs
are absent from the 3.6 controller API list: the CAAS API address reader never
uses unit addresses in the first place.

Thus 3.6 has two layers of separation:

1. Pod addresses are classified as machine-local.
2. Kubernetes controller API discovery is sourced from the normal Service,
   not the pod.

### Main

The Kubernetes HA work introduces a headless controller Service. Each
controller pod receives a stable ordinal FQDN of the form represented by
`utils.ControllerPodFQDN`. The provider reports both values separately:

- `caas.Unit.Address` is the current pod IP;
- `caas.Unit.FQDN` is the stable per-pod headless-Service name.

The headless FQDN is used as the Dqlite bind identity. It is not currently
inserted into `controller_api_address` and is not propagated in ordinary agent
configuration.

The ordinary controller API address path is maintained by
`internal/worker/apiaddresssetter`. For each controller ID, it calls
`NetworkService.GetControllerAPIAddresses` and writes the result to the
controller database.

`domain/network/state/unitaddress.go` deliberately changed this calculation
for Kubernetes controllers:

- only the unit net node is queried;
- the shared Kubernetes Service net node is omitted, because it cannot route a
  controller-specific request reliably;
- the stored CAAS pod address is explicitly admitted even though its stored
  scope is machine-local;
- the query projects the pod address as `local-cloud`.

This behaviour was introduced by commit `577c2c55f0` (`refactor: drop service
IP addresses from controller addresses`). The application domain still stores
the pod IP as machine-local in `domain/application/state/unit.go`; the
local-cloud classification is a query-specific translation for controller API
discovery.

After `apiaddresssetter` converges, the practical result on Kubernetes is one
`podIP:apiPort` entry for each controller. The bootstrap Service addresses are
replaced for the tracked controller IDs.

### Address audience in the controller database

The `controller_api_address` table contains:

- a mandatory `controller_id`;
- the address and scope;
- an `is_agent` flag.

The flag does not mean "agent-only". Its effective meaning is "available to
clients, and also available to agents". The state queries are asymmetric:

- the agent query uses `WHERE is_agent = true`;
- the client query reads every `controller_api_address` row.

Because the Kubernetes pod IP is projected as local-cloud and normally marked
for agents, it is returned by both queries. It also survives
`ToHostPortsNoMachineLocal`.

### Client propagation on main

The pod IPs can reach clients through several paths:

- User login calls `GetAPIHostPortsForClients` and returns the result as
  `LoginResult.Servers`.
- Migration source information and cross-controller registration call
  `GetAllAPIAddressesForClients`.
- `HighAvailability.ControllerDetails` calls
  `GetAPIAddressesByControllerIDForClients` and exposes addresses by
  controller ID.

For a client outside the Kubernetes cluster, a pod IP is normally not
directly routable. Ordinary CLI operation can mask this problem:

- for a non-proxied connection, the client adds the endpoint through which it
  successfully connected to its cached endpoint list;
- for a Kubernetes-proxied connection, the client dials the local
  port-forward rather than the returned server addresses.

Those behaviours do not make the server-provided pod addresses client
reachable. They also do not solve the controller-specific routing requirement
behind a shared Service.

### Existing client getter methods

There are three client-facing service methods over the same state data:

1. `GetAPIHostPortsForClients` returns nested, typed host/port groups. It is
   used by login because the RPC contract for `Servers` retains one address
   group per controller.
2. `GetAllAPIAddressesForClients` returns a flat `[]string`. It was added for
   migration and cross-controller callers while those callers were moved away
   from Mongo state. Its introducing commit, `4d0092cc02`, described it as a
   replacement for `APIHostPortsForClients`, but the login path continued to
   require the older representation.
3. `GetAPIAddressesByControllerIDForClients` returns a map keyed by controller
   ID. It was added later for the HA `ControllerDetails` facade.

Their selection policies have drifted:

- `GetAPIHostPortsForClients` removes machine-local addresses but does not
  apply public-scope preference.
- `GetAllAPIAddressesForClients` orders addresses with public scope first, but
  its `PrioritizedForScope` helper includes all valid fallback tiers rather
  than only the best tier used by 3.6.
- `GetAPIAddressesByControllerIDForClients` prefers cloud-local addresses,
  despite being exposed as a client-address operation.

The difference between nested host/ports and flat strings is a legitimate
wire-format concern. The different reachability and ordering policies are not.

## Proposed changes: client-visible addresses and address separation

### Define the intended policy

Use the following policy on Kubernetes:

| Purpose | Address source and preference |
|---|---|
| Any-controller agent connection | Normal Service FQDN/ClusterIP first, followed by a public LB fallback |
| Any-controller user or remote-controller connection | Best public Service address tier; ClusterIP only when no public Service address exists |
| Internal controller-node connection | Current pod IP or per-pod headless FQDN, selected specifically for that controller |
| External controller-node connection | Do not advertise an internal pod address; use a genuinely controller-targeted external transport |
| Dqlite | Per-pod headless FQDN |

The stored pod scope can remain machine-local for CAAS binding and relation
network semantics. A node-specific API resolver may interpret it as reachable
inside the cluster, but that interpretation must not alter the address used by
general client discovery.

### Separate shared and node-specific endpoints

Do not restore the normal Service address by inserting it into every
controller's `controller_api_address` rows. That would associate a shared,
load-balanced endpoint with a specific controller and would recreate the
routing error that commit `577c2c55f0` fixed.

Instead, model two address sets explicitly:

- an any-controller endpoint set, not keyed by controller ID;
- a controller-node endpoint set, keyed by controller ID.

The current `controller_api_address` table can remain the node-specific set. A
separate table or an equally explicit endpoint-target model should hold the
shared set. A separate table is preferable to a nullable or synthetic
controller ID because it makes the routing guarantee part of the schema.

Address audience and routing target are independent properties. The model
must be able to express both:

- client versus agent/internal reachability;
- shared versus specific-controller routing.

The existing `is_agent` flag expresses neither axis completely and must not be
used to infer that an internal address is suitable for clients.

### Maintain the any-controller endpoint set

For Kubernetes controllers, project the normal controller Service into the
shared endpoint set:

- add the normal Service FQDN with local-cloud scope;
- retain the ClusterIP with local-cloud scope;
- retain load balancer or other external ingress addresses with public scope.

This must be maintained after bootstrap. The projection should react when the
Kubernetes Service addresses change, including when a load balancer ingress is
allocated after the controller starts. The existing `k8s_service` network
state already records these addresses; a worker can project them into the
controller-wide endpoint set without making one domain service depend on
another.

For machine controllers, the any-controller set can continue to be derived
from the individual controller-node addresses while honouring the management
space.

Bootstrap should seed the same endpoint model that the steady-state worker
maintains, avoiding the current transition from Service addresses to pod
addresses when `apiaddresssetter` first runs.

### Preserve Kubernetes Service FQDNs

The shared endpoint model must preserve DNS names reported by Kubernetes, not
only IP addresses. This is required for controller Services whose externally
usable address is a load balancer hostname rather than an ingress IP.

In 3.6, a Service hostname is represented as an ordinary `SpaceAddress` with:

- the FQDN as its value;
- address type `hostname`;
- `public` scope for an external name or load balancer ingress;
- `provider` origin.

The address is stored in the controller `CloudService` alongside Service IP
addresses. The legacy migration description preserves its value, type, scope
and origin. The 3.6 CAAS client-address path then includes it in the best-public
address tier and returns it as `fqdn:api-port`.

Main is not currently faithful to this representation after decoding the
legacy description. `domain/network/modelmigration/import_cloudservice.go`
copies the address metadata into `ImportK8sServiceAddress`, but
`domain/network/service/migration.go` converts every such address into an
`ImportIPAddress`. Its placeholder-subnet map contains entries only for IPv4
and IPv6. A `hostname` address consequently fails conversion with:

```text
no subnet UUID found for address type "hostname"
```

This is a conversion failure, not merely an omitted endpoint. The live
`UpdateK8sService` path in `domain/application/state/application.go` has the
same IP-only assumption, so a fresh Service update containing a load balancer
hostname also fails when it cannot find a subnet for address type `hostname`.

Handle Kubernetes Service addresses according to their type:

- persist IPv4 and IPv6 addresses through the existing `ip_address` and
  placeholder-subnet path;
- persist a fully qualified Service name in `fqdn_address`, retaining its
  `local-cloud` or `public` network scope, and link it to the
  `k8s_service.net_node_uuid` through `net_node_fqdn_address`;
- reconcile and remove both IP and FQDN rows when the provider-reported
  Service addresses change;
- make shared Service endpoint queries include both address representations;
- project a public Service FQDN into the any-controller client endpoint set
  with the API port, where it participates in the same best-public selection
  as a public ingress IP.

Do not conflate these provider-reported Service FQDNs with either of the
internal names introduced for other purposes. The normal controller Service
FQDN synthesized by 3.6 for agents was not stored in `CloudService`, and the
per-pod headless-Service FQDN on main identifies one controller for Dqlite.
Neither is a migrated public client endpoint.

### Apply 3.6-compatible scope selection

Provide two distinct selection operations:

- **best matching scope**: return only the first non-empty scope tier, matching
  the 3.6 `AllMatchingScope` behaviour;
- **all usable scopes in priority order**: return local-cloud followed by
  public fallbacks for agents.

Use best-public selection for any-controller client endpoints. Use ordered
cloud-local selection for any-controller agent endpoints. Apply this policy
before converting addresses into RPC representations.

### Route consumers to the correct set

Any-controller endpoints should feed:

- user and ordinary agent login results;
- agent configuration through `APIAddresser`;
- migration source controller information;
- cross-controller registration.

Node-specific endpoints should feed:

- inter-controller API workers;
- reverse SSH routing to a particular controller;
- other explicitly controller-targeted internal operations.

`HighAvailability.ControllerDetails` must not return Kubernetes pod IPs to an
external user client. Until there is a controller-targeted external mechanism,
the CAAS implementation should return `NotSupported` and allow existing
commands such as debug-log and status-history to use their single-controller
fallbacks.

Longer-term controller-targeted alternatives include:

- a proxy or API route whose path identifies the controller;
- Kubernetes port-forwarding to a selected pod rather than the shared
  Service;
- server-side fan-out for operations that currently ask the client to connect
  to every controller.

### Required tests

Add policy tests covering at least:

- ClusterIP-only: agents and clients use the normal Service, not pod IPs;
- LoadBalancer plus ClusterIP: clients receive only public LB addresses;
- LoadBalancer plus ClusterIP: agents prefer internal Service addresses and
  retain public fallback;
- pod replacement changes the node-specific endpoint but not the shared
  endpoint;
- delayed LB allocation updates the client endpoint set;
- user login, migration and cross-controller results contain no pod IP when a
  Service endpoint is available;
- legacy import preserves a public Service hostname as a scoped
  `fqdn_address` linked to the Service net node;
- bootstrap and subsequent Service updates accept a load balancer hostname
  and replace or remove stale Service FQDNs;
- a public Service FQDN is returned to clients in the best-public tier, while
  an internal Service FQDN is restricted to the appropriate agent endpoint
  selection;
- per-controller internal lookup never returns a shared Service as though it
  identified one controller;
- CAAS `ControllerDetails` does not advertise an unreachable controller-node
  endpoint.

## Proposed changes: converge client API address getter methods

### Converge policy, preserve representations

The service should expose one canonical typed operation for general client
endpoints, for example:

```go
GetAPIEndpointGroupsForClients(ctx context.Context) (APIEndpointGroups, error)
```

The result should preserve address scope and the grouping required by the
login RPC. On Kubernetes, the shared Service addresses form one endpoint
group. On machine controllers, groups may continue to represent individual
controller nodes.

The canonical operation must perform reachability filtering and best-public
scope selection once. Callers should only adapt the selected result:

- the login facade converts it to `[]network.HostPorts` and then to
  `LoginResult.Servers`;
- migration and cross-controller facades flatten it to dial-address strings.

These conversions should be methods or small helpers on the typed result, not
separate service methods with independent selection logic.

After callers are migrated, remove:

- `GetAPIHostPortsForClients`;
- `GetAllAPIAddressesForClients`.

No external RPC wire contract needs to change as part of this convergence.

### Keep controller-targeted lookup semantically separate

`GetAPIAddressesByControllerIDForClients` must not be treated as another
serialization of the general client list. It asserts that every returned
address reaches the named controller, which is a different contract.

Rename it to make the routing property explicit, for example:

```go
GetControllerNodeAPIEndpoints(ctx context.Context) (
    map[string]APIAddresses, error,
)
```

If separate internal and external node-targeted operations are required, name
the audience explicitly rather than reusing `is_agent` semantics. The external
operation must return only genuinely externally routable, controller-specific
endpoints.

### Centralise ordering and conversion

The typed endpoint collection should own deterministic operations for:

- best-scope selection;
- all-scope prioritisation;
- machine-local exclusion where appropriate;
- conversion to grouped `network.HostPorts`;
- flattening to unique dial-address strings.

This removes the current situation where three public service methods each
iterate over the same state map and make different policy decisions.

### Migration sequence

1. Introduce the canonical typed endpoint result and shared selection helpers.
2. Make the existing methods delegate to it without changing callers.
3. Add parity tests proving both legacy methods are views of the same selected
   endpoint set.
4. Move login, migration and cross-controller callers to thin local adapters.
5. Remove the two format-specific domain service methods.
6. Rename and narrow the controller-targeted method independently.

Tests should assert address contents and ordering at the canonical service
boundary, then separately verify only the RPC conversion in each facade.

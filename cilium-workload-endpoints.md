# Cilium-backed workload endpoints for Juju relations

Status: exploratory design notes

Date: 2026-07-14

## Invariants

1. The model runs on a Kubernetes cloud.
2. Cilium is the cluster's CNI implementation.

The idea under consideration is to stop making Layer 3 values such as
`ingress-address` and `egress-subnets` part of the relation contract. Instead,
Juju would describe workloads with stable names and identity selectors, then
let Kubernetes and Cilium resolve those descriptions to the current network
state.

The important qualification is that this removes Layer 3 data from the
**relation control-plane contract**. It does not remove Layer 3 from the data
plane. DNS, Services, EndpointSlices, Cilium's IP cache, and eBPF maps still
resolve the logical destination to IP addresses before packets can flow.

## How Cilium works at a high level

### CNI is the lifecycle entry point

The [CNI specification](https://www.cni.dev/docs/spec/) is deliberately narrow.
Kubernetes invokes a plugin when a Pod network interface must be added,
checked, or removed. CNI does not define a general service-discovery namespace,
a workload URI scheme, or application paths.

Cilium's CNI plugin asks the per-node Cilium agent to configure the Pod's
networking. The agent attaches eBPF programs at Linux networking hooks and
programs maps for routing, load balancing, policy, and observability. The
Cilium operator handles cluster-wide coordination tasks, while Hubble reads
flow information produced by the eBPF data plane. See Cilium's
[component overview](https://docs.cilium.io/en/stable/overview/component-overview/)
and [introduction](https://docs.cilium.io/en/stable/overview/intro/).

Depending on its configuration, Cilium can route Pod traffic through an
overlay or native routing, implement Kubernetes Service load balancing, and
replace `kube-proxy`. These choices change how packets travel, but not the
Kubernetes discovery contract used by a workload.

### Endpoints and identities

In Cilium terminology, an **endpoint** is normally a Pod network namespace and
its IP address or addresses. Cilium derives labels from Kubernetes metadata
and assigns the endpoint a security identity representing its set of
security-relevant labels. Multiple replicas with the same relevant labels can
share an identity. Policy can therefore select `role=client` and
`role=database` instead of embedding their current IP addresses. Cilium then
maintains the identity-to-IP mapping used by the data plane. See Cilium's
[terminology](https://docs.cilium.io/en/stable/gettingstarted/terminology/).

Cilium also creates a `CiliumEndpoint` resource for each managed Pod. Its name
and namespace match the Pod, and its status contains operational information
such as addresses, labels, identity, and policy. A `CiliumEndpoint` is useful
for inspection and Cilium state propagation; it is not a stable, routable
application name. See the
[CiliumEndpoint documentation](https://docs.cilium.io/en/stable/network/kubernetes/ciliumendpoint/).

Numeric Cilium endpoint IDs and security identity IDs are implementation
state. A relation must not persist either one. If Juju uses Cilium identities,
the durable contract should be the label set from which Cilium derives the
identity.

### Services provide names; Cilium provides the data plane

Kubernetes, rather than the CNI contract, supplies the main stable naming
abstraction. A Kubernetes Service selects a changing set of Pods and has a
stable object identity and DNS name. A normal Service name resolves to a
virtual Service IP. A headless Service name resolves to the current Pod IPs,
and can also provide stable per-Pod DNS names when the workload supplies stable
hostnames. See the Kubernetes documentation for
[Services](https://kubernetes.io/docs/concepts/services-networking/service/)
and [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).

With Cilium service load balancing, the path is conceptually:

```text
client Pod
  -> resolve Kubernetes Service name
  -> connect to Service address and named or numbered port
  -> Cilium eBPF selects a current backend
  -> Cilium enforces source and destination identities plus L4 policy
  -> backend Pod
```

The Service name is the discovery handle. Cilium labels and identities are the
policy handle. They are related, but they are not interchangeable.

### Policy can be name-like without becoming addressing

Cilium Layer 3 policy can select managed endpoints by labels and can refer to
Kubernetes Services by name. It can add Layer 4 port restrictions. For
supported protocols, an Envoy-backed Layer 7 rule can additionally restrict
HTTP methods, hosts, and URL paths. See Cilium's
[Layer 3 policy](https://docs.cilium.io/en/stable/security/policy/layer3/),
[Kubernetes policy constructs](https://docs.cilium.io/en/stable/security/policy/kubernetes/),
and [Layer 7 policy](https://docs.cilium.io/en/stable/security/policy/layer7/).

An HTTP path such as `/v1/query` is therefore an optional application-policy
selector. It is not a general CNI path and is meaningless for protocols such
as PostgreSQL unless Cilium has a matching L7 parser and policy model.

### Multi-cluster is an additional topology

Cilium Cluster Mesh can synchronize endpoint and identity state, provide
cross-cluster Pod routing, enforce cluster-aware policy, and offer
cross-cluster discovery through Global Services or the Kubernetes
Multi-Cluster Services API. This requires explicit mesh setup and shared trust;
it is not implied by installing Cilium independently in two clusters. See the
[Cluster Mesh overview](https://docs.cilium.io/en/stable/network/clustermesh/intro/)
and [Global Services](https://docs.cilium.io/en/stable/network/clustermesh/services/).

## What should replace the Layer 3 fields?

The current fields describe two different directions:

| Current value | Meaning | Logical replacement |
| --- | --- | --- |
| `ingress-address` | A destination that a remote unit can contact | A service or unit discovery reference |
| `egress-subnets` | Source CIDRs that the remote side can allow | A source workload identity selector |

One endpoint name cannot faithfully replace both. The proposed model needs to
separate **where to connect** from **who may connect**.

### Destination discovery reference

A destination reference should identify a Kubernetes resource without
capturing its current IP address. A conceptual shape is:

```yaml
destination:
  kind: kubernetes-service
  cluster-id: <immutable cluster identity>
  namespace: <controller-resolved namespace>
  service: postgresql
  port:
    name: database
    protocol: TCP
  scope: application
```

`scope` would distinguish an application-level, load-balanced endpoint from a
unit-level endpoint. The structured fields are more useful than an opaque URI,
although they could have a canonical display form such as:

```text
k8s-service://<cluster-id>/<namespace>/<service>/<port-name>
```

That URI would be a Juju convention, not a Cilium-native URI.

Juju should resolve the cluster DNS suffix locally rather than place a value
such as `cluster.local` in the durable contract. Likewise, a port name is more
stable than a numeric target port, but the protocol and the relation
interface's port semantics remain part of the application contract.

### Source authorization principal

The source side should be represented by immutable, Juju-owned Kubernetes
labels from which Cilium can derive policy identity. For example, Juju could
label every workload Pod with model, application, and optionally unit UUIDs:

```yaml
source-identity:
  match-labels:
    model.juju.is/id: <model UUID>
    app.juju.is/id: <application UUID>
```

The exact labels need a dedicated contract. They must be present on the Pod,
must be included in the cluster's Cilium identity-relevant label configuration,
and must not depend only on mutable application display names. Numeric Cilium
identity values must remain an internal projection of these labels.

Juju can then materialize relation-scoped `CiliumNetworkPolicy` resources that
allow the source selector to reach the destination selector on the declared
ports. Separate policies per relation make revocation and reconciliation
tractable.

### Optional Layer 7 selector

If a relation contract describes HTTP or another Cilium-supported L7 protocol,
it could also declare host, method, and path constraints. This should be a
separate optional part of the endpoint contract. Treating every relation
endpoint as a URL would conflate service discovery with protocol-specific
authorization.

## Fit with Juju's Kubernetes provider

This direction has a useful starting point in the current code. The Kubernetes
provider creates an application Service and an `<application>-endpoints`
headless Service in
[`internal/provider/kubernetes/application/application.go`](internal/provider/kubernetes/application/application.go).
The former supplies application-level discovery; the latter can support direct
backend discovery and stable StatefulSet unit names.

Today Juju exposes network information through `network-get`, whose contract
defines bind addresses, ingress addresses, and egress CIDRs. It also injects
selected ingress and egress values into relation settings. See
[`network-get`](docs/reference/hook-command/list-of-hook-commands/network-get.md)
and the relation-network conversion in
[`apiserver/facades/agent/uniter/uniter.go`](apiserver/facades/agent/uniter/uniter.go).

A workload endpoint is provider-owned topology, not arbitrary charm data. A
typed agent API exposed through a separate `location-get` hook tool is a
cleaner authority than asking charms to publish and validate Cilium object
names in their ordinary databags. Juju may still transport an internal
descriptor across controllers for cross-model relations, but charms should
consume a stable platform API or a locally materialized host and port.

The existing application Service currently does not by itself express every
relation endpoint's real ports. A complete design must reconcile named Service
ports from relation or charm metadata rather than assume that the existence of
the Service solves discovery.

### Current Kubernetes address behavior

The Kubernetes provider creates the application Service during every
application deployment. Its default name is the Juju application name, its
type is `ClusterIP`, and it initially contains a placeholder port. The
provisioning path records the Service addresses against the application's
`k8s_service` network node.

Both the current branch and Juju 3.6 deliberately include that
application-level Service address in `network-get`:

| Result | Current branch | Juju 3.6 |
| --- | --- | --- |
| Ingress address | The selected application Service address, normally the ClusterIP | Every non-machine-local Service address, including cloud-local and public addresses |
| Bind address | The Pod address | The Pod or container address |
| Primary address | The Pod address | The Pod or container address |
| Egress subnet fallback | A `/32` or `/128` derived from ingress when stronger egress configuration is absent | A `/32` or `/128` derived from the first ingress address |

On the current branch, `v_unit_relation_network` unions the unit network node
with the application `k8s_service` network node. The network service then
explicitly treats the Service addresses as valid ingress addresses but not as
locally bindable workload addresses. `network-get --ingress-address` therefore
selects the Service address, while `--bind-address` and `--primary-address`
select the Pod address. The current address ranking prefers a cloud-local IPv4
address over a public IPv4 address.

The enter-scope path and the relation-aware `network-get` path already share
the same domain operation:
`networkService.GetUnitRelationNetworks`. `EnterScope` converts its result to
the Juju-owned `ingress-address` and `egress-subnets` unit relation settings;
`NetworkInfo` converts the same result to the uniter facade response.

Juju 3.6 implemented the same semantics more directly. `Unit.AllAddresses`
returned the application Service addresses followed by the container address.
`NetworkInfoCAAS` classified the machine-local container address as bindable
and all other address scopes as ingress, then used ingress as the final egress
subnet fallback.

This means the proposed logical location is not replacing a raw Pod address in
the ingress case. It is replacing the application-level Service virtual IP.
The bindable Pod location remains a separate concept.

## Prototype: `ingress-name` and `location-get`

### Prototype contract

The compatibility prototype keeps the existing Juju-owned
`ingress-address` and `egress-subnets` relation settings unchanged and adds:

```yaml
ingress-name: postgresql
```

The name is placed in `ingress-name`; it is never placed in
`ingress-address`. This lets existing charms continue to consume the Layer 3
keys while an experimental charm opts into named discovery. Removing the old
keys can be evaluated after the new contract has survived upgrade,
cross-model, and mixed-version testing.

For the first slice, `ingress-name` is the short Kubernetes Service name. The
provider creates that Service with the Juju application name, and workloads in
the same model namespace can resolve it directly. This is intentionally scoped
to same-model Kubernetes relations. A later representation must add namespace
and immutable cluster identity before the name can safely cross namespace or
cluster boundaries.

This first slice validates named discovery, not Cilium policy. The value is a
Kubernetes Service name whose traffic Cilium can implement; it is not a
`CiliumEndpoint` name or numeric Cilium identity.

### Domain shape

Keep name selection beside the existing binding and relation selection logic
in the network domain, but model the result separately from IP networking:

```go
type UnitLocation struct {
    EndpointName string
    IngressNames []string
}
```

The initial provider service operations mirror their network counterparts:

```go
GetUnitEndpointLocations(ctx, unitName, endpointNames)
GetUnitRelationLocations(ctx, unitName, relationUUIDs)
```

The state layer can obtain the Service name by joining `unit`, `application`,
and `k8s_service`. Joining `k8s_service` is important: it establishes that the
unit actually has a Kubernetes application Service instead of assuming every
application name is a network location. The provider service remains
responsible for mapping the name onto requested endpoints and relations.

The prototype assumes the model is Kubernetes with Cilium, as stated by the
invariants. For a non-CAAS unit or a CAAS unit without a recorded Service, the
domain operation should return a typed unsupported or not-found result rather
than inventing a name.

### Uniter facade and hook context

Add a `LocationInfo` method to the uniter facade. Its request mirrors
`NetworkInfo`:

```go
type LocationInfoParams struct {
    Unit       string
    RelationID *int
    Endpoints  []string
}

type LocationInfoResult struct {
    Error        *Error
    IngressNames []string
}

type LocationInfoResults struct {
    Results map[string]LocationInfoResult
}
```

Only the endpoint-to-result map and relation override behavior need to match
`NetworkInfo`; the first prototype should not invent `bind-name` or
`egress-name` fields before their semantics are defined. The facade should
perform authorization and wire conversion, while the domain service selects
the Service name.

Carry the new call through the existing agent stack:

```text
location-get
  -> jujuc Context.LocationInfo
  -> HookContext.LocationInfo
  -> api/agent/uniter Unit.LocationInfo
  -> uniter facade LocationInfo
  -> network ProviderService.GetUnitEndpointLocations or
     ProviderService.GetUnitRelationLocations
```

Because `LocationInfo` is a new facade method, register a new Uniter facade
version rather than exposing it accidentally through an older version. The
current highest version is 22, so the prototype should register version 23,
retain a version-22 wrapper that hides `LocationInfo`, and have the agent
client check `BestAPIVersion()` before calling it. A new agent connected to an
older controller can then report that named locations are unsupported without
affecting `network-get`.

### `location-get` hook tool

The initial command mirrors the useful part of `network-get`:

```text
location-get <binding-name> [--ingress-name]
```

With `--ingress-name`, it returns the first selected name. Its default YAML
shape preserves the plural facade value:

```yaml
ingress-names:
  - postgresql
```

Add the command to the jujuc command registry and generate its hook-command
reference from the command help source. The generated hook-command Markdown
must not be edited manually.

### Enter-scope and relation-setting lifecycle

Use the same location domain operation from both `LocationInfo` and
`EnterScope`, just as `NetworkInfo` and `EnterScope` share the relation network
operation today. During entry, merge the first ingress name with the existing
network settings:

```yaml
ingress-address: 10.152.183.27
egress-subnets: 10.1.2.3/32
ingress-name: postgresql
```

`ingress-name` must be Juju-owned throughout its lifecycle:

1. `EnterScope` seeds it when the relation unit is first created.
2. The successful `config-changed` commit-hook refresh path updates it
   alongside the current network keys. This opportunistically backfills units
   that were already in scope before upgrading to the prototype.
3. Unit-state relation-setting transformation removes charm-supplied
   `ingress-name`, just as it removes charm-supplied `ingress-address` and
   `egress-subnets`.

If existing relation units must receive the key immediately after controller
upgrade, the prototype needs an explicit reconciliation or migration path;
the current commit-hook refresh is not an immediate backfill guarantee.

Add `IngressNameKey` beside the existing unit-state key constants. For the
prototype, the existing `UpdateNetworkInfo` commit-hook flag can trigger both
network and location refresh. If location later gains an independent
lifecycle, introduce a separately named refresh flag rather than silently
expanding the old name forever.

Enter-scope and refresh should use one pure conversion helper for the three
Juju-owned settings. That avoids allowing the initial write and later refresh
to drift in formatting or empty-value behavior.

### First implementation slices

1. **Domain result and state query**: Add `UnitLocation`, query the application
   Service name for a CAAS unit, and return endpoint- and relation-keyed
   locations from the provider service.
2. **Facade and API client**: Add RPC parameter types, the thin uniter
   `LocationInfo` facade method, and `api/agent/uniter.Unit.LocationInfo`.
3. **Hook plumbing**: Add the hook-context method, mocks, `location-get`, its
   command registration, and command tests.
4. **Relation settings**: Add `IngressNameKey`, seed it during enter-scope,
   refresh it during commit-hook changes, and reject charm attempts to write
   it.
5. **End-to-end validation**: Deploy two applications in one Cilium-backed
   model, confirm that the consumer receives `ingress-name`, resolve it from
   the workload container, and compare `location-get` with relation data.

### Prototype test matrix

- A CAAS unit returns its application Service name for an endpoint.
- Relation lookup returns the same endpoint and name as endpoint lookup.
- A non-CAAS unit does not receive a fabricated ingress name.
- Facade authorization and unknown relation handling match `NetworkInfo`.
- `location-get --ingress-name` returns the first name, while default output
  returns all names.
- Enter-scope writes all three Juju-owned relation keys.
- A later successful `config-changed` hook refreshes `ingress-name`.
- An explicit reconciliation test is required if immediate upgrade backfill is
  added.
- `relation-set ingress-name=...` cannot overwrite the Juju-owned value.
- Existing `network-get` output and the old relation keys remain unchanged.
- Mixed controller/agent facade versions fail by capability detection rather
  than by changing the existing `NetworkInfo` wire contract.

### Deliberate prototype limits

- No Cilium CRD name or numeric identity is exposed.
- No cross-model, cross-namespace, or cross-cluster name is claimed to work.
- No port is encoded in `ingress-name`; named-port discovery remains a separate
  contract decision.
- No `bind-name` or source/egress identity is introduced yet.
- No automatic removal of `ingress-address` or `egress-subnets` occurs.

## Implications

### Benefits

- **Address churn disappears from the charm contract**: Pod replacement,
  scaling, dual-stack changes, and Service backend changes are reconciled by
  Kubernetes and Cilium.
- **Policy follows workload identity**: A relation can grant connectivity to
  an application or unit identity instead of predicting its source CIDR.
- **Relation removal can revoke connectivity**: Juju can remove the
  relation-scoped policy independently of application data cleanup.
- **Application and platform responsibilities become clearer**: Charms
  describe protocols and ports; Juju describes model topology; Kubernetes and
  Cilium resolve and enforce it.
- **Observability improves**: Hubble can report flows with Kubernetes and
  Cilium identity metadata, which can be correlated with Juju model entities.
- **Dual-stack becomes less visible to charms**: A Service reference can
  resolve to whichever address families the cluster supports.

### Constraints and risks

#### A Cilium endpoint name is the wrong durable primitive

Stateless Deployment Pod names and their matching `CiliumEndpoint` names
change when Pods are recreated. StatefulSet names can be stable, but relying on
the Cilium CRD still couples Juju to operational state. Use Kubernetes Service
identity for discovery and label selectors for policy.

#### Application-level and unit-level endpoints differ

An application Service is a good fit for stateless, load-balanced relations. It
is insufficient when the consumer must address a particular database member,
leader, shard, or peer. Unit-level discovery needs a StatefulSet/headless
Service identity or another explicit stable-unit mechanism. A generic
Deployment replica does not have a durable network name.

#### Names do not preserve `egress-subnets` semantics outside Cilium

Identity policy works while both sides are managed by the same Cilium policy
domain. An external database firewall, cloud security group, public ingress,
host-network process, unmanaged Pod, or conventional egress gateway still sees
addresses and CIDRs. At those boundaries Juju must retain an L3 projection or
use a boundary-specific identity mechanism. The Kubernetes+Cilium invariants
do not imply that every relation peer is a Cilium-managed Pod.

#### Cross-model is not automatically cross-cluster

Two models in different namespaces on one cluster can use qualified Service
discovery and cross-namespace Cilium policy. Two clusters require routability,
identity synchronization, naming, and trust through Cluster Mesh or another
gateway.

Cilium Global Services conventionally identify a service using matching name
and namespace in participating clusters. Arbitrary Juju model namespaces and
application-name collisions do not automatically fit that rule. Juju may need
to create local alias Services, use `ServiceExport`/`ServiceImport`, or own a
translation layer. A remote Kubernetes object path alone is not sufficient.

#### Policy is not application authentication

Cilium policy can prove enough network identity to authorize a flow inside its
trust domain. It does not remove the need for TLS, credentials, Juju secrets,
or protocol authorization. DNS names also affect TLS SNI and certificate SANs,
so the endpoint name must be stable before charms issue certificates.

#### Policy propagation becomes part of relation readiness

Juju would need to order Service creation, endpoint readiness, identity
allocation, DNS availability, and policy installation before declaring the
workload endpoint usable. Relation hooks can race with eventually consistent
Kubernetes and Cilium state. The system needs observable readiness and
idempotent reconciliation rather than a fixed delay.

#### Default-deny interaction needs deliberate ownership

Cilium policies are additive, and selecting an endpoint for ingress or egress
can place that direction under policy enforcement. Juju-generated policies
must coexist with administrator policy, allow DNS where required, and avoid
claiming ownership of the namespace's entire security posture. The operator
needs a clear rule for whether Juju establishes only relation allows or also
establishes default deny.

#### This creates a platform dependency

The proposal intentionally sacrifices CNI portability. Juju would need a
supported Cilium version range, CRD and capability detection, Kubernetes RBAC
for policy resources, upgrade tests, and useful failure reporting when cluster
configuration filters out the Juju identity labels.

## Plausibility assessment

| Scope | Plausibility | Assessment |
| --- | --- | --- |
| Same cluster, application-level Service | High | Kubernetes naming and Cilium label policy already provide the required primitives. |
| Same cluster, stable StatefulSet unit | Medium to high | Headless Service DNS works, but Juju must define unit identity and readiness precisely. |
| Same cluster, arbitrary Deployment replica | Low | The replica has no durable workload name; prefer application scope. |
| Connected Cilium Cluster Mesh | Medium | Technically supported, but naming, trust, namespace mapping, and failure handling become product concerns. |
| Independent clusters with Cilium installed | Low | Cilium installation alone supplies neither discovery nor routability between clusters. |
| External or unmanaged relation peer | Low as a replacement | L3 data or a gateway-specific abstraction remains necessary. |
| Raw `CiliumEndpoint` name or numeric identity in relation data | Reject | Both expose Cilium operational state rather than a stable application contract. |

The core idea is plausible and attractive if stated as follows:

> For Cilium-managed Kubernetes workloads, Juju relations exchange or expose a
> stable Kubernetes Service reference for discovery and Juju-owned label
> selectors for authorization. Juju projects those logical values into DNS,
> Services, and Cilium policy, while retaining L3 projections at boundaries.

It is less plausible as the broader claim that Cilium itself provides a
routable name or path that can directly replace both ingress addresses and
egress subnets.

## Suggested first experiment

Keep the first experiment to two Juju applications in one Kubernetes cluster:

1. Add immutable model and application identity labels to workload Pod
   templates and verify that Cilium derives the intended identities.
2. Give one provider endpoint a real named port on a Kubernetes Service.
3. Add `location-get` and the `ingress-name` relation setting, initially
   containing the same-namespace Kubernetes Service name.
4. Reconcile one ingress and one egress `CiliumNetworkPolicy` for each relation.
   Include relation UUIDs in resource ownership metadata so cleanup is exact.
5. Let a capable charm consume the service host and named port without
   publishing `ingress-address` or `egress-subnets` in its databag.
6. During a compatibility period, continue producing the existing L3 values
   for charms that do not advertise support for logical endpoints.
7. Exercise Pod recreation, scaling, Service IP replacement, dual-stack,
   controller restart, relation removal, and default-deny administrator
   policies.

The experiment succeeds if no Pod or Service IP crosses the relation contract,
connections survive address churn, unrelated applications cannot connect, and
access disappears after the relation is removed.

## Open questions

- Is a relation endpoint normally application-scoped or unit-scoped, and how
  does a charm declare the exceptional case?
- Where are protocol and named port declarations owned: charm metadata, a
  relation contract artifact, or runtime relation data?
- Does the first product requirement include cross-model relations in another
  namespace, another cluster, or both?
- Should Juju merely create relation allow policies, or should it also manage
  default-deny policy?
- Which immutable cluster identity is portable across controller migration and
  Cluster Mesh configuration?
- Does "path" mean a Kubernetes object path, a canonical Juju workload URI, or
  an HTTP path? These have different lifecycles and enforcement semantics.
- How does Juju report that DNS, Service backends, or Cilium policy have not yet
  converged?
- What is the L3 fallback contract for ingress, egress, and cross-model traffic
  that leaves the Cilium trust domain?

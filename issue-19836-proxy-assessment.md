# Juju #19836: proxy coverage on 4.1

Assessment recorded on 16 October 2026 against branch `4.1`, commit
`f17f1c5e0a9bb0dedd3b069f8f1a41fb01fde2c6`. Local HEAD and upstream `4.1`
matched at the time of assessment. Source links below are relative to this
repository and refer to that revision.

This is an assessment and an input to implementation planning. The proposed
approach has not been selected for implementation. No production code was changed
as part of this investigation.

## Outcome and scope

[Issue #19836](https://github.com/juju/juju/issues/19836) exposes a broader problem
than the original failing bootstrap request. The legacy `http-proxy` and
`https-proxy` settings are incompatible with their `juju-http-proxy` and
`juju-https-proxy` equivalents, but the Juju settings have not consistently
reached Juju's own outbound clients. Operators therefore cannot simply add the
legacy values to compensate for clients that do not honour the Juju values.

The mutual exclusion is intentional validation. Removing it would not repair
the client coverage or configuration-update problems.

The initial bootstrap ordering has improved on 4.1: the controller proxy updater
now applies initial configuration before releasing the bootstrap workers. That
addresses the narrow startup failure mechanism. It does not establish a
comprehensive fix. Remaining problems include SDKs that cache environment proxy
settings, a MAAS transport that bypasses proxies, a model-operator process with
no proxy updater, early bootstrap requests, and proxy configuration errors lost
at both RPC boundaries.

A comprehensive fix should make the applicable Juju proxy configuration reach
all intended Juju-owned outbound clients, including authentication and streaming
subpaths. It should cover initial configuration, later additions, changes and
removal, and preserve deliberate bypasses and existing legacy behaviour.

## Original report and subsequent changes

The original report used Juju 3.6.5 on LXD with no SNAT and proxy-only internet
access. Bootstrap supplied `apt-http(s)-proxy`, `juju-http(s)-proxy` and
`snap-http(s)-proxy` through model defaults. Controller charm resolution attempted
a direct request to `https://api.charmhub.io/v2/charms/refresh` and failed.

Relevant history:

| Reference | What it establishes |
| --- | --- |
| [#19462](https://github.com/juju/juju/pull/19462), merged 28 April 2025 | Repaired model-default validation so defaults could be switched away from the legacy bootstrap workaround. It did not supply missing proxy configuration to internal clients. |
| [24 June 2025 issue comment](https://github.com/juju/juju/issues/19836#issuecomment-2999888139) | Identified controller-model scope, watching proxy changes and removing mutation of `http.DefaultTransport` as design concerns. |
| [#22715](https://github.com/juju/juju/issues/22715) | Reported the related Charmhub bootstrap failure on 4.x Kubernetes. |
| [#22894](https://github.com/juju/juju/pull/22894), merged 16 July 2026 | Added synchronous initial controller proxy application and bootstrap gating. Merge commit `4aecbb56490ff2e053ece0aa45b78f8da93a9f44` is included in the assessed 4.1 revision. |

At assessment time, #19836 remained open, with `3.6`, `area/proxy`, `kind/bug` and
`triaged` labels. Its latest comment, dated 17 March 2026, said it was not on the
roadmap. These are observations of the issue at assessment time, not a decision
about the intended 4.1 work.

The recorded proxy QA for #22894 used an unreachable proxy, `127.0.0.1:1`, and
observed a proxy-connect failure on an LXD API request. This demonstrates that a
request selected the proxy. It does not demonstrate a successful bootstrap with
direct internet access unavailable.

## Current configuration flow

### Values survive configuration and persistence

The reviewed bootstrap configuration path retains the settings:

1. [Bootstrap command](cmd/juju/commands/bootstrap.go) combines model defaults
   with explicit configuration.
2. [Instance configuration](internal/cloudconfig/instancecfg/instancecfg.go)
   serialises model configuration attributes and reconstructs them.
3. [Agent bootstrap](agent/agentbootstrap/bootstrap.go) passes controller model
   configuration to `SetModelConfig`.
4. [Model configuration bootstrap](domain/modelconfig/bootstrap/bootstrap.go)
   validates, coerces and persists it.

[Model configuration validation](environs/config/config.go) enforces the
incompatibility between legacy and Juju proxy settings. The main remaining
problem is consumption and lifecycle, rather than loss of the proxy attributes
in this bootstrap persistence path.

### The updater applies process settings and machine settings differently

In [the proxy updater](internal/worker/proxyupdater/proxyupdater.go),
`handleProxyValues` selects the Juju settings when `HasProxySet()` is true,
otherwise falling back to the legacy settings. It then:

- Updates the agent process environment.
- Updates the in-process proxy configuration through its callback.
- Applies external settings through another callback.
- Writes legacy proxy settings to `/etc/juju-proxy.conf` and systemd configuration
  where applicable.

The distinction matters: selecting Juju proxies for agent requests must not
accidentally turn them into machine-wide proxy environment settings. Apt, snap
and charm hook configuration have their own existing behaviour to preserve.

### Controller startup now has a proxy gate

[The controller manifold](internal/worker/proxyupdater/controllermanifold_linux.go)
reads the controller model's configuration from local domain services. It also
reads controller agent API addresses for automatic proxy exclusions, tolerating
the initially empty address case. Its source watches both configuration and
controller API address changes.

The manifold reads and applies initial configuration, starts the worker, then
opens the controller-proxy-ready gate.
[Machine agent wiring](cmd/jujuagentd/agent/machine/manifolds.go) makes both the
machine and Kubernetes bootstrap workers wait for that gate.

This does not gate every client or outbound request. The
[HTTP client manifold](internal/worker/httpclient/manifold.go) has no input
dependencies, and other workers can initialise clients independently.

There is also a dependency constraint: domain services depend on the HTTP client
factory, while the controller proxy updater depends on domain services. Making
the factory depend directly on the existing updater would introduce a cycle.
Readiness must therefore be designed at the appropriate consumer boundary, or
with a separately available initial configuration source.

### Three proxy mechanisms coexist

| Mechanism | Current behaviour | Consequence |
| --- | --- | --- |
| [Juju HTTP middleware](internal/http/middleware.go) | Calls `httpproxy.FromEnvironment()` for each request. | Existing clients using this middleware observe environment changes. |
| [In-process proxy configuration](api/proxy/config/proxyconfig.go) | Maintains mutex-protected settings and can install its resolver into `http.DefaultTransport`. | Clients using or copying that resolver can observe its updates, but remain coupled to global state. |
| SDK transports using `http.ProxyFromEnvironment` | Go caches the environment-derived proxy resolver on its first use in the process. | Later environment changes do not update the resolver, even for newly constructed clients using the same function. |

The first two mechanisms also differ in semantics. In particular,
`DefaultConfig.GetProxy` falls back from an absent HTTPS proxy to the HTTP proxy;
the standard environment resolver does not make that same choice. Their
`no_proxy` matching behaviour is not identical, including treatment of ports.

[The HTTP package](internal/http/client.go) and agent/CLI entrypoints mutate or
install proxy behaviour in the global default transport. Relevant entrypoints
are [jujuagentd](cmd/jujuagentd/run.go),
[containeragent](cmd/containeragent/main_nix.go) and
[the CLI](cmd/juju/commands/main.go).

The [HTTP client worker](internal/worker/httpclient/worker.go) caches clients by
purpose, without a model UUID. It is not currently a per-model proxy-policy
factory inside the shared controller process.

## Confirmed gaps and required integration points

### 1. SDKs that cache the environment

Go's `http.ProxyFromEnvironment` uses a process-global `sync.Once`. If the first
call sees no proxy, that resolver can remain direct for the lifetime of the
process. If it sees a proxy, subsequent changes or removal can leave it using
the old proxy. Recreating an SDK client alone does not reset this cache.

The following paths need explicit proxy or transport integration:

| Client family | Juju integration points | Required coverage |
| --- | --- | --- |
| Kubernetes cloud clients | [`CloudSpecToK8sRestConfig`](caas/kubernetes/client.go) | Supply `rest.Config.Proxy` for both in-cluster and externally constructed configurations. Check all callers retain it. |
| Kubernetes secrets | [`configToK8sRestConfig` and broker construction](internal/secrets/provider/kubernetes/provider.go) | Cover both broker creation paths, including controller and unit consumers. |
| Kubernetes streaming | [Tunnel](caas/kubernetes/tunnel.go) and [exec implementation](internal/provider/kubernetes/exec) | Cover HTTP requests, SPDY and websocket transports used for exec and port forwarding. |
| Azure | [`initEnviron`](internal/provider/azure/environ.go), [provider initialisation](internal/provider/azure/init.go) | Production configuration leaves the SDK sender unset, selecting the SDK default transport. Supply appropriate transport configuration to resource clients and credential/token clients. |
| Vault | [`NewVaultClient` and client factory](internal/secrets/provider/vault/provider.go) | Replace reliance on the default SDK client while preserving TLS, namespace, authentication and renewal behaviour. Establish precedence for Vault-specific environment configuration. |
| HTTP tracing | [`newHTTPClient`](internal/worker/trace/client.go) | Configure the OTLP HTTP exporter explicitly; its SDK provides a proxy option. |
| gRPC tracing | [`newGRPCClient`](internal/worker/trace/client.go) | Configure proxy/dial behaviour and define what an update means for an established exporter connection. |

Dependency source reviewed at this revision included client-go `v0.36.2`,
Azure azcore `v1.22.0`, mittwald/vaultgo `v0.2.5`, HashiCorp vault/api `v1.23.0`,
go-cleanhttp `v0.5.2`, OTLP HTTP exporter `v1.45.0` and gRPC `v1.83.2`.

Kubernetes and Vault default transport behaviour was reproduced directly in a
standalone probe. Azure and tracing findings are based on their constructor and
dependency source paths; they were not exercised against live services.

### 2. MAAS has an explicit proxy bypass

[`SetCloudSpec`](internal/provider/maas/environ.go) constructs a fresh
`http.Transport` in the skip-TLS-verification branch, setting TLS configuration
but no `Proxy` function. That branch therefore goes direct regardless of proxy
environment settings. The normal branch relies on default transport behaviour.

Both TLS modes should use appropriately configured clients. Proxy support must
not alter the requested TLS verification policy.

### 3. The Kubernetes model operator has no proxy updater

[Model-operator manifolds](cmd/jujuagentd/agent/modeloperator/manifolds.go) construct
an HTTP client factory, log routing and a Kubernetes broker, but contain no proxy
updater. [Model-operator pod construction](internal/provider/kubernetes/modeloperator.go)
does not populate model proxy values in the pod environment.

Passing model configuration to the broker does not itself configure transport
proxy behaviour. This leaves external consumers such as Loki and the Kubernetes
broker without the model's intended proxy updates.

The [ProxyUpdater facade](apiserver/facades/agent/proxyupdater/proxyupdater.go)
already allows model-agent authentication, and its client accepts an entity tag.
That provides a possible configuration source. Wiring must separate in-process
client configuration from apt, snap and machine filesystem side effects.

Other agent lifecycles differ:

- Machine unit workers inherit their parent's HTTP client getter through
  [unit-agent deployment](internal/worker/deployer/unit_agent.go) and
  [unit manifolds](internal/worker/deployer/unit_manifolds.go).
- [Container unit manifolds](cmd/containeragent/unit/manifolds.go) already contain
  a proxy updater and HTTP client factory, but no equivalent initial readiness
  gate between them.

### 4. Early bootstrap requests need an explicit initial policy

[Machine cloud-init generation](internal/cloudconfig/userdatacfg.go) exports the
legacy settings as machine-wide environment configuration. Its tools-download
curl arguments already prefer the Juju proxy settings.

However, [the bootstrap command](cmd/jujuagentd/agent/bootstrap.go) performs a
JWKS reachability check before normal agent worker startup and proxy application.
The corresponding [agent-bootstrap helper](agent/agentbootstrap/bootstrap.go)
calls `jwk.Fetch` without an injected HTTP client. This path needs the initial
controller model proxy configuration before it makes the request.

Kubernetes controller bootstrap is different:
[pod configuration](internal/cloudconfig/podcfg/podcfg.go) carries
`JujuProxySettings`, and
[`proxyEnvironment`](internal/provider/kubernetes/bootstrap.go) populates lower-
and uppercase proxy environment variables, including exclusions, before
bootstrap-state runs. It would be inaccurate to describe all bootstrap paths as
missing an initial proxy environment.

Any solution must cover early request consumers without exporting Juju proxy
settings machine-wide as a shortcut. It must also prevent an early SDK request
from permanently selecting stale process-global environment configuration.

### 5. Proxy configuration errors are lost on both sides of the API

Two independent defects were reproduced with temporary regression tests:

- In the [server facade](apiserver/facades/agent/proxyupdater/proxyupdater.go), an
  authorised entity's configuration result is assigned, then its `Error` field
  is overwritten with the successful authorisation result's nil error.
- In the [API client](api/agent/proxyupdater/proxyupdater.go), `ProxyConfig` checks
  the bulk call error and result count, but ignores the individual result's
  `Error` before decoding its settings.

A model configuration or address lookup failure can therefore appear to the
updater as a successful empty configuration. Applying that result can clear
working proxy settings.

Both boundaries need to preserve errors. Update handling should distinguish a
valid removal of configuration from a failed lookup, retain the last good
settings on failure, and retry according to the worker lifecycle contract.

### 6. Application success and watcher readiness need stronger contracts

The updater currently logs in-process and external application errors rather
than propagating them. The controller gate can therefore open without proving
that every required application step succeeded.

The initial synchronous read also precedes establishment of the worker's
watcher. A stale comment says `NotifyWorker` swallows the first notification;
the [current implementation](core/watcher/notify.go) dispatches that initial
event. It causes a fresh read and eventual convergence, so this is not evidence
of a permanently missed update.

Nevertheless, the gate does not establish that the watcher is ready or that its
initial event has been processed. A revised design should use the initial
watcher event as the readiness barrier and define which successful application
steps are required before dependent requests are allowed. Tests should exercise
changes racing watcher subscription, the initial read and gate release.

## Existing coverage and migration dependencies

The following distinctions matter when estimating the work. Many clients already
receive dynamic environment-based proxy behaviour; they still need coherent
policy ownership if global environment mutation is replaced.

### Clients already using Juju's configured HTTP plumbing

The reviewed paths include shared Charmhub resolution and downloads, controller
charm bootstrap, asynchronous charm download and revision checking, S3 object
storage, SSH import, simplestreams and Loki. These generally obtain clients from
the HTTP client getter.

EC2 API/IAM and OpenStack clients use Juju HTTP client construction. GCE's main
API connection wraps a supplied client. Generic downloads and the upgrader also
use Juju HTTP plumbing. Authentication and metadata subpaths still need separate
coverage; a configured top-level API client does not prove every SDK request
uses that transport.

Unit S3 requests use the API connection's HTTP client through
[the unit S3 caller](internal/worker/units3caller/manifold.go). Preserve the
internal routing and TLS behaviour of that path.

### Clients that still depend on global transport behaviour

These are migration requirements before removing the global default transport
mutation; they are not all proven to fail with the present updater.

| Consumer | Current dependency and required attention |
| --- | --- |
| [JWT parser](internal/worker/jwtparser/manifold.go) | Constructs a bare `http.Client` with timeout and redirect policy. Inject a configured client while preserving those restrictions; an explicit-client parser constructor already exists. |
| [Container registry](internal/docker/registry/registry.go) and [base client](internal/docker/registry/internal/base_client.go) | Capture/use `http.DefaultTransport`. Thread policy through registry factory callers, including the Kubernetes model configuration manager. |
| [Snap assertions](core/snap/assertions.go) | Use `http.DefaultClient`. Supply transport behaviour without introducing a forbidden core-to-worker or core-to-internal dependency. |
| [vSphere client](internal/provider/vsphere/internal/vsphereclient/client.go) | govmomi copies the default transport's proxy function. The copied Juju resolver can currently remain dynamic, but removing global installation would change this behaviour. |
| [vSphere template handling](internal/provider/vsphere/vm_template.go) | Contains a raw `http.Get`; include download paths alongside the main SDK connection. |
| [GCE authentication](internal/provider/gce/internal/google/conn.go) | Token-source construction does not inject the main client's transport through the context. Audit token requests separately; preserve intended metadata endpoint bypasses. |
| [API and remote-controller clients](api/apiclient.go) | Primary transports have explicit proxy handling, while some fallback paths depend on the global transport. Preserve controller exclusions, TLS server-name behaviour, websocket scheme handling and connection lifecycle. |

Removing global mutation should follow migration of its consumers. Otherwise a
cleanup of the shared transport could regress paths that currently work.

## Design decisions for the implementation plan

### Configuration ownership

The starting recommendation is to preserve the current controller-model scope
for controller-originated outbound traffic. Separately running machine,
container and model agents should receive their applicable model settings.
Local CLI requests should continue to use the CLI host's configuration rather
than automatically inheriting controller model proxy settings.

Per-model proxy policy inside the shared controller is a separate design choice.
The current purpose-keyed HTTP client getter does not provide it. Adopting that
behaviour would require model identity throughout client acquisition and clear
ownership for shared services, provider clients and credentials.

### A common explicit policy

A candidate direction is one updateable proxy-policy interface, supplied to
Juju transports and SDK adapters. The package and interface placement remain to
be chosen in accordance with repository layering rules. This should avoid new
global state and avoid making domain services depend on workers.

The policy contract needs explicit decisions about:

- HTTP and HTTPS selection, including whether HTTPS falls back to the HTTP proxy.
- `no_proxy` matching for hostnames, domains, ports, IPv4/IPv6 and CIDRs.
- Proxy authentication and handling of proxy URLs without exposing credentials.
- Automatic exclusions as controller addresses change.
- Atomic publication of a coherent configuration snapshot.
- Failure behaviour, retry and retention of the last successfully applied state.
- Existing connection behaviour after updates, including HTTP keep-alive pools,
  websocket/SPDY sessions, Vault renewal and gRPC exporter reconnection.
- Preservation of proxy policy when TLS configuration or a cached client changes.

An injected resolver can be consulted per request. Established connections may
need a separate replacement or drain policy; changing a resolver does not by
itself reroute an already open stream.

### Compatibility boundaries

Preserve legacy-only operation and the intentional mutual exclusion. Preserve
machine-wide legacy configuration, the separate apt and snap settings, and
`JUJU_CHARM_*` hook environment behaviour in
[the hook context](internal/worker/uniter/runner/context/context.go).

Explicitly establish bypass requirements for local sockets, cloud metadata,
controller API access and object storage. Do not replace that analysis with a
blanket rule that all internal traffic must bypass proxies.

Application workload networking and kubelet image pulls are outside Juju's own
HTTP client transport configuration. They should be distinguished from Juju's
registry requests and downloads when documenting the fix and its limits.

## Suggested implementation sequence

This sequence is a proposal for discussion, not an agreed implementation plan.

1. **Repair configuration error propagation.** Add permanent regression tests for
   both RPC boundaries and demonstrate that a lookup error does not clear the
   active configuration.
2. **Define policy and lifecycle contracts.** Agree scope, matching semantics,
   update behaviour, readiness and layering. Design around the existing
   HTTP-factory/domain-services/updater dependency constraint.
3. **Connect the missing agent and bootstrap paths.** Cover model operators,
   container-agent readiness and early machine bootstrap requests. Keep machine
   configuration side effects distinct from in-process policy application.
4. **Integrate SDK transports.** Cover Kubernetes, Azure, Vault, MAAS and tracing,
   including credential clients, streaming paths and reconnection behaviour.
5. **Migrate remaining global consumers.** Convert JWT, registry, snap assertions,
   vSphere, authentication and API fallback paths before removing dependence on
   global default transport mutation. Retain a compatibility bridge only if its
   purpose and removal boundary are explicit.
6. **Validate actual proxy-only operation and update documentation.** Exercise
   successful bootstrap and running-agent updates on machines and Kubernetes,
   with proxy logs proving the intended requests reached their destination.

## Evidence and validation limits

### Existing tests

The existing packages passed:

```sh
go test -race ./internal/worker/httpclient ./internal/worker/proxyupdater -count=1
```

The dqlite-tagged controller manifold suite was compiled with the checkout's
static dqlite dependencies and passed:

```sh
CGO_CFLAGS="-I$PWD/_deps/dqlite-deps-4.0-amd64/include" \
CGO_LDFLAGS="-L$PWD/_deps/dqlite-deps-4.0-amd64 -Wl,-Bstatic -ldqlite -luv -llz4 -lsqlite3 -Wl,-Bdynamic" \
go test -c -race -tags=dqlite \
  -o /tmp/juju-19836-controllerproxy.test ./internal/worker/proxyupdater

/tmp/juju-19836-controllerproxy.test \
  -test.run TestControllerManifoldSuite -test.v
```

Its successful-start test stubs the running worker. The run also logged the
expected external-callback error and sandbox-denied attempts to write
`/etc/juju-proxy.conf` and systemd files. Passing this suite does not demonstrate
successful machine filesystem configuration or the real watcher's startup
lifecycle.

Bounded stress runs used race-compiled binaries, two stress workers and a
31-second outer timeout with a 30-second per-run timeout. The final periodic
reports at 30 seconds recorded 56 HTTP-client runs and 44 proxy-updater runs,
with zero failures. The outer timeout terminated the runs as intended. These
runs were performed outside the sandbox after sandbox restrictions prevented
local socket use.

### Targeted reproductions

Temporary test overlays added a `TestReviewProxyConfigError` case to each
ProxyUpdater suite. The tests asserted preservation of a source/result error:

```sh
go test -race -overlay=/tmp/juju-19836-audit/overlay.json \
  ./api/agent/proxyupdater \
  ./apiserver/facades/agent/proxyupdater \
  -run 'TestProxyUpdaterSuite/TestReviewProxyConfigError' -count=1
```

Both failed at the expected assertions: the server returned a nil result error,
and the client returned a nil call error. The overlays were diagnostic artifacts,
not permanent regression tests committed to the repository.

A standalone probe constructed actual client-go and Vault default transports
and evaluated their proxy functions without making network requests. It compared
them with a freshly constructed `httpproxy.FromEnvironment()` resolver:

| Environment transition | client-go and Vault default resolvers | Fresh resolver |
| --- | --- | --- |
| Initially configured with proxy one | Proxy one | Proxy one |
| Change to proxy two | Still proxy one | Proxy two |
| Remove proxy configuration | Still proxy one | Direct |
| Separate process: initially empty, then configure proxy two | Still direct | Proxy two |

Temporary investigation artifacts were under `/tmp/juju-19836-audit/`, including
`cacheprobe.go` and `overlay.json`. Stress logs were
`/tmp/juju-19836-httpclient-stress.log` and
`/tmp/juju-19836-proxyupdater-stress.log`. These paths are ephemeral and are not
repository deliverables.

### What has not been established

No live proxy-only bootstrap, cloud SDK authentication, proxy update across
running deployments, HA/restart or migration end-to-end scenario was run.
There was no full build or `make pre-check` for this assessment. Existing package
tests and bounded stress do not establish comprehensive proxy coverage.

## Acceptance coverage for a comprehensive fix

| Area | Cases to cover |
| --- | --- |
| Configuration contract | Legacy-only, Juju-only, no proxy, invalid mixed settings, HTTP-only and HTTPS-only, the selected HTTPS fallback semantics, and authenticated proxies. |
| Exclusions | Host and domain matching, ports, IPv4/IPv6 CIDRs, localhost, automatic controller-address changes and intended metadata/API/object-store routes. |
| Updates | Add, change and remove proxy settings after clients exist; a first request before normal agent readiness; TLS/client replacement retaining current policy. |
| Errors and startup | Server and client per-entity errors, successful empty configuration versus failed lookup, application failures, retries and last-good retention; changes racing watcher subscription and initial gate release. |
| Client adapters | Each SDK and its authentication/token requests; redirects and archive downloads; Kubernetes websocket/SPDY/exec; Vault renewals and draining; tracing reconnection. |
| Agent roles | Controller, non-controller machine, machine unit, container unit and Kubernetes model operator, with correct configuration ownership. |
| Lifecycle | Process restart and persisted configuration, controller HA and address changes, migration-related address/exclusion updates, cancellation and deterministic client cleanup. |
| Live bootstrap | Successful machine and Kubernetes bootstrap with direct internet egress blocked and only the Juju settings supplied for agent traffic. Separate apt/snap settings may still be needed for their tools. Verify proxy logs for real Charmhub resolution and archive download, plus applicable tools metadata/downloads. Avoid local charm shortcuts that bypass the intended request. |
| Documentation | Explain which Juju-owned clients use the Juju settings, how changes take effect, how legacy and charm hook settings differ, and which workload/platform networking remains separately configured. |

Permanent concurrency and worker tests should use `-race` and bounded stress in
accordance with the repository rules. Integration QA should verify successful
requests and destinations, rather than relying only on failures against an
unreachable proxy.

# Kubernetes controller lifecycle ownership

Updated: 20 November 2026

The proposal is to move Kubernetes controller workload configuration and lifecycle orchestration from controller-specific Juju code into the controller charm. The charm would declare the controller image as an OCI resource, prepare each node's runtime configuration, start and supervise the workload through Pebble, and coordinate safe patch upgrades. Juju would retain generic application provisioning and authoritative controller identity, membership and version operations. Bootstrap and restart without a functioning Juju API require explicit support outside ordinary hook execution.

This extends the ownership change behind [JU202 Standalone Controller HA Scale-Out](<JU202 - Standalone Controller HA Scale-Out.md>). The current machine bootstrap already separates the snapped `jujud` controller from the `jujuagentd` machine agent. JU202 proposes making the machine charm install and initialise that controller on additional units. The corresponding Kubernetes target gives the charm responsibility for the controller workload after ordinary unit introduction.

The main decision is where this lifecycle should live before patch upgrades are implemented. A universal charm or separate substrate charms can support the same ownership boundary; packaging is assessed later as a secondary implementation choice.

These are design recommendations and open questions, not an approved implementation plan. The findings are based on source inspection at the revisions listed under Source baseline; the proposed lifecycle has not been demonstrated in a deployment.

## Proposed division of responsibility

The following responsibilities belong in the Kubernetes controller charm under this proposal. Each row identifies the interface or lifecycle condition needed to transfer existing work from Juju, or place forthcoming work in the charm from the outset. In particular, standalone HA patch-upgrade orchestration is not yet complete at the assessed revisions.

| Controller workload responsibility | Charm responsibility | Prerequisite or limit |
| --- | --- | --- |
| Workload image and container definition | Declare the controller container and OCI resource; supply the tested workload release | Bootstrap must resolve and record the same resource before a Juju API exists. Adoption must preserve the initial controller and existing storage. Generic resource resolution and pod creation remain in Juju/Kubernetes. |
| Workload paths, mounts and permissions | Declare storage/mount requirements and prepare runtime files in the workload | Preserve durable identity, controller data, agent state and required socket access. A metadata change alone does not migrate existing PVCs or make ephemeral agent storage safe. |
| Coordination of controller enrolment after a unit is introduced | Obtain shared and per-node material, coordinate enrolment through supported core operations, and reconcile interrupted joins | Ordinary unit introduction still precedes hooks. Core must authorise controller-node enrolment and persist identities and credentials. The existing per-ordinal controller nonce path can only be retired after equivalent admission guarantees exist. |
| Construction and installation of final per-node runtime configuration | Assemble or request validated configuration, install it durably with private permissions, preserve credentials on retries | Requires the enrolment/material interface and an explicit atomic installation contract. JU202's complete runtime bundle is proposed work; it is not already implemented at the baseline. |
| Controller service plan and normal startup sequencing | Configure Pebble commands/environment, wait for join prerequisites, start and supervise the workload | Pebble must start before the charm can configure it. The present shell waits for runtime configuration before starting Pebble. Retained-state restart must also work without hooks or a live API. The role of the co-located `jujuagentd` remains to be resolved. |
| Workload readiness configuration | Configure checks combining API health and local membership; reconcile unit status | Core must expose trustworthy membership state. Kubernetes Service eligibility comes from readiness probes, not merely charm `active` status; generic probe wiring remains. |
| Patch-upgrade workload orchestration, still to be implemented | Coordinate an authorised resource change, preparation, orderly replacement and post-upgrade health checks | Preflight must precede the first pod replacement. There must be one rollout authority, compatible agent/workload versions and recovery from interruption. Core retains version/schema policy and authoritative upgrade state. |

These changes replace the special image/container preservation, startup-plan generation and controller-specific runtime-delivery paths. [Preserved controller container][core-preserve], [startup plan][core-startup], [introduction][core-introduction], [ordinary resource mapping][core-resource-map]

### What prevents a complete transfer to ordinary hooks

Two dependencies prevent an implementation based entirely on ordinary charm hooks. First, bootstrap must create the initial controller before its API can execute the charm. Second, after all controllers stop, restoring that API cannot require hooks, secret reads or introductions served by the same unavailable API. A charm may prepare durable restart configuration, but a local mechanism must be able to execute it independently. Ordinary Pebble `--hold` and API-dependent unit introduction do not provide that mechanism by themselves. [Ordinary workload startup][core-workload], [persistent controller state][core-persistent], [introduction reuse][core-init-check]

### What stays in Juju and Kubernetes

Generic StatefulSet scale, pod/PVC/Service provisioning, image pulling, charm-agent introduction and hook delivery stay with Juju and Kubernetes. Controller identity validation/revocation, authoritative lifecycle records, dqlite membership and database upgrades stay in the controller. The charm coordinates those operations through supported interfaces; it does not take over their database implementation.

Topology publication, `controller.conf` reconciliation and much integration/status handling already belong to the charm. Extending them to initialise fresh replicas is useful, but should not be counted as code newly offloaded from Juju. The bootstrap, offline-restart, enrolment and rollout contracts above are the substantive remaining gates, irrespective of charm packaging.

## Proposed Kubernetes workload and scale out flow

The proposed target uses the ordinary OCI-image resource mechanism:

```yaml
containers:
  controller:
    resource: controller-image

resources:
  controller-image:
    type: oci-image
    description: Juju controller workload image
```

This is a conceptual declaration, not complete metadata. Storage, runtime permissions, probes and restart behaviour must also be specified.

Juju already resolves image resources into container definitions and starts ordinary workload containers with Pebble held for charm configuration. Kubernetes pulls and runs the declared image; the charm configures and starts the software through Pebble. [Resource mapping][core-resource-map], [resource resolution][core-resource-read], [ordinary workload construction][core-workload], [Ops container management][ops-containers]

For an additional controller unit, the intended sequence would be:

1. Ordinary application provisioning creates the pod, storage and charm-declared workload container.
2. Ordinary unit introduction supplies credentials for the charm's unit agent to use surviving controllers.
3. The charm coordinates controller-node enrolment and obtains shared and per-node startup material, using the same underlying contracts proposed for machines.
4. The charm publishes topology and installs the runtime configuration into the workload container, preserving the selected identity and material across retries.
5. The charm starts `jujud` through Pebble when the join prerequisites exist. Any retained `jujuagentd` service must also have an explicit owner and configuration path.
6. `jujud` joins dqlite. API health and local membership determine workload readiness and charm status.

The architectural distinction is between introducing a Juju unit and enrolling a controller node. Unit introduction must precede hooks. Controller-node enrolment can happen afterwards under charm coordination because the unit agent can connect to existing controllers; it does not inherently depend on its local `jujud` process. The current unit configuration already contains controller API addresses. [Unit configuration][core-unit-config]

JU202's distinction between machine charm ownership and Kubernetes pre-charm admission describes its chosen scope. It is not an inherent requirement that controller enrolment and complete runtime delivery always precede the Kubernetes charm. Moving those responsibilities is feasible, but requires changing the current controller-specific introduction path, not merely adding an image resource.

Core remains authoritative for controller-node identity, credential validation and revocation, persisted lifecycle state and dqlite membership. Charm ownership means coordinating these operations through supported interfaces, as proposed for machines. The registration and authorisation checks must survive the change in caller and timing.

| Responsibility | Machines | Proposed Kubernetes target |
| --- | --- | --- |
| Supply controller software | Snap resource | OCI-image resource |
| Prepare runtime and join material | Charm | Charm |
| Start and supervise workload | Snap service | Pebble service |
| Allocate infrastructure | Machine provisioner | Application provisioner and Kubernetes |
| Perform database joining | `jujud` and dqlite | `jujud` and dqlite |

## Bootstrap and restart requirements

Bootstrap must select a compatible charm and controller-image resource before constructing the first pod, then record the resource association in the new controller application. Later reconciliation must adopt that same combination without unexpectedly replacing the only controller. Current controller bootstrap and application reconciliation already need special handling to avoid rolling the first pod during adoption. [Bootstrap adoption handling][core-adoption]

The controller workload image and the image supplying Juju's provisioning agents are separate responsibilities. They may initially contain overlapping binaries, but resource selection should not silently change the version of the agents that introduce and operate the charm. The current controller container runs both `jujud` and `jujuagentd`; making its image a resource does not itself establish that the latter can be removed.

The larger constraint is restarting when every controller is down. Ordinary sidecar pods use ephemeral agent storage and start workload Pebble with `--hold`. A recreated pod may need the API for introduction, and a held workload needs charm execution to start. If every controller requires that sequence, none can restore the API on which the sequence depends. [Ordinary storage][core-ephemeral], [held workload startup][core-workload]

The current controller path replaces the ordinary agent-data mount with persistent storage. Init skips introduction when installed unit configuration exists, and the controller workload's startup script builds and starts its local service plan. These are properties the revised design must preserve or replace deliberately. [Persistent controller mounts][core-persistent], [introduction completion check][core-init-check], [local startup script][core-startup-script]

An already-initialised controller must be able to reuse local identity, credentials, runtime configuration and database state and start without a live Juju API. Persisting a Pebble plan alone does not overcome `--hold`; restart needs an explicit local mechanism. This mechanism must distinguish a fresh join from an existing node and must never turn missing material on a joining node into permission to bootstrap an independent cluster.

There is also an ordering requirement for fresh joins: Pebble must be reachable before a charm can configure the workload through its API. The current controller shell waits for runtime configuration before starting Pebble. A charm-owned initialisation path must start the supervisor first and gate the controller service instead.

These requirements concern retained-state restart. Recovery from lost controller data remains a separate operation.

## Storage networking and readiness

The current charm accesses controller files and Unix sockets directly through shared storage. A normal sidecar workload needs deliberate shared mounts or operations through Pebble. Persistent runtime state, dqlite data, permissions and socket access must be part of the container contract. Existing PVC names and mounts also need a transition plan.

Topology publication must distinguish the pod's dqlite identity from the routable controller API Service. Kubernetes readiness should prevent an unready controller from receiving client traffic while still permitting the peer connectivity required to join. The generic application path already wires workload probes to Pebble; the controller charm needs checks with the correct API and membership semantics, including behaviour before initial configuration exists.

## Patch upgrades and rollout ownership

Resource-driven image selection also needs a single upgrade authority. Existing controller upgrade code directly changes StatefulSet image references. That path must be reconciled with charm resource updates and controller-compatible rollout sequencing. Resource publication alone does not establish safe quorum-preserving upgrades. [Existing image upgrade path][core-upgrade]

At the inspected Juju revision, controller patch-upgrade policy exists: the controller-upgrader service accepts patch changes within the same major/minor and records the controller model's target agent version before the controller target version. However, the standalone controller's upgrader and controller upgrade-step manifolds are explicitly disabled pending rework. An older CAAS upgrader remains in the tree and even carries an HA-support TODO. The existence of provider image-rewrite code is not evidence of a complete standalone HA patch-upgrade implementation. [Patch-upgrade service][core-patch-service], [disabled Kubernetes workers][core-disabled-upgrades], [CAAS upgrader][core-caas-upgrader]

Here, an in-place Kubernetes patch upgrade means preserving the controller deployment, node identities and persistent data while replacing pods as necessary to run the selected image. It does not imply changing a container image without recreating its pod.

Implementing that lifecycle first under core-owned containers and then transferring it to resource-driven charm workloads risks duplicating more than JU202's runtime adapter:

- Artifact selection must move from version-derived image rewriting to a validated resource selection. The existing provider path changes both init and workload container images; workload and provisioning-agent upgrades need an explicit compatibility contract.
- Upgrade preparation, safe sequencing, readiness gates, progress and interrupted-upgrade recovery need one authority. A failed or interrupted node must resume from durable identity/configuration, not repeat fresh enrolment or rotate its credential.
- Scale-out during an upgrade must either be blocked deliberately or select a compatible admitted version. A new unit must not receive whichever artifact happens to be newest in a channel.
- Adopting existing core-created pods into a resource-driven application introduces a template and storage migration alongside an upgrade if the ownership move is delayed. Earlier adoption lets patch-upgrade testing exercise the final lifecycle from the start.

Core should retain version compatibility checks, database/schema upgrade rules, membership operations and authoritative desired/observed controller state. The charm should own installation and workload supervision, and coordinate the permitted restart or rollout through supported operations. Moving orchestration does not mean moving database migration logic into the charm or treating an arbitrary `juju refresh` as sufficient upgrade authorisation.

There is a concrete provider boundary to resolve. Ordinary application StatefulSets do not specify an update strategy, so Kubernetes applies its default rolling update behaviour. That updates pods from the highest ordinal down and waits for readiness. The ordinary `ApplicationConfig` has no explicit rollout-strategy or partition control. A charm-controlled rollout therefore needs an agreed contract with application reconciliation: for example, deliberately preserving a charm-managed partition, or accepting a declared rollout policy. Its behaviour across resource refresh, scaling and reconciliation needs validation. [StatefulSet construction][core-sts-construction], [application configuration][core-app-config], [Kubernetes update strategies][k8s-sts-updates]

The provider assigns its desired update strategy when preparing an update, but sends a JSON merge patch. That assignment alone does not prove that an externally set partition is erased: omitted fields can survive a merge patch. The evidence establishes a missing explicit ownership contract, not a demonstrated partition-reset bug. [StatefulSet reconciliation][core-sts-reconcile]

Default rolling updates plus API-and-membership readiness may provide part of the mechanism, but do not by themselves define compatibility checks, approval to start, failure recovery or protection against concurrent scaling. Any additional rollout controls should be a narrow, explicit provider contract where possible, rather than a second controller-specific upgrade engine. This is residual Juju work even with charm ownership.

### Lessons from charm-refresh

The inspected `charm-refresh` implementation patches a StatefulSet's rolling-update partition through the Kubernetes API and requires permission to do so. Its stop-event handling raises the partition to stop later units following the first replacement. Its documentation states that the highest-numbered unit cannot be prevented from refreshing after `juju refresh`, and warns that a partition above every unit suppresses the events needed to proceed. This makes it a useful orchestration reference, but not a ready-made controller upgrade protocol. [Partition implementation][refresh-partition], [permission check][refresh-permission], [stop handling][refresh-stop], [refresh constraints][refresh-constraints]

Controller upgrade compatibility, quorum checks and preparation must precede the first permitted disruption. Blocking the newly replaced workload from starting is not equivalent to preventing the old controller from being stopped. This matters especially for a one-controller deployment and for a degraded HA cluster. Keep Juju's authoritative version and schema checks; do not automatically inherit a generic application's rollback policy when controller downgrades are currently rejected. [Controller upgrade policy][core-patch-service]

The larger difference is self-management. Valkey's hooks and peer coordination are served by an independent Juju controller. A controller charm cannot rely on its own unavailable API to recover the processes that provide that API. Retained local configuration, identity and an API-independent restart mechanism remain necessary with either one charm or two. The shared reconciliation logic can coordinate fresh joins and upgrades while the API is available; each runtime implementation must also satisfy the offline-restart contract.

## Implementation cost and sequencing

The recommendation is to settle Kubernetes workload ownership before implementing patch upgrades, and preferably transfer it during scale-out work. Avoiding JU202's introduction adapter alone is not a strong enough reason to delay scale out. Avoiding a second implementation and later migration of patch-upgrade orchestration is a stronger reason to establish the intended owner now.

The work has three distinct parts: transferring workload ownership, implementing safe upgrades, and choosing charm packaging. The first requires bootstrap adoption, enrolment and restart without an API; the second needs an explicit rollout contract. A packaging change alone accomplishes neither. The selection and publishing differences between one artifact and two are covered in the later packaging section.

### What JU202 adds that would later move

Much of the Kubernetes sequence in JU202 already exists in core. Desired scale, per-ordinal nonce preparation, authenticated introduction, controller-node registration, StatefulSet provisioning and dqlite joining are not all new work attributable to this proposal. The fresh-runtime handoff is the principal missing connection at these revisions.

| JU202 work | Additional work at these revisions | Fate under the proposed charm-owned workload |
| --- | --- | --- |
| Final runtime configuration in introduction | Produce per-node runtime material; extend the facade response, RPC type and agent client | Retire this delivery path when the charm obtains startup material through the shared enrolment contracts. Core still owns authoritative identity and credentials. |
| Durable init installation and retry semantics | Validate and install the complete bundle on the PVC; owner-only runtime file; completion marker last; preserve installed credentials across retries | Move controller-runtime installation and its recovery tests to the charm/workload boundary. Ordinary unit introduction and durable agent configuration remain necessary. |
| Private bootstrap seed in a Kubernetes Secret | Change the initial pod's private seed delivery and mounts | Retain equivalent bootstrap support. Moving responsibility for later units does not remove the initial trust and startup problem. |
| Membership-aware readiness | Expose a read-only local membership check and combine it with API health; wire Service eligibility and charm status | Keep the core membership capability and its tests; put workload check configuration in the Kubernetes charm. Some provider-specific probe wiring can disappear. |
| Topology convergence and controller removal | Adapt charm behaviour and verify the existing core lifecycle | Largely retained. Membership, identity revocation and persisted lifecycle are still core responsibilities. |

The runtime transport spans four obvious boundaries: API facade, RPC definition, API client and init command. Runtime generation and credential retry behaviour also need supporting changes and tests. This is a bounded adapter plus a meaningful installation/recovery contract, rather than a new Kubernetes scale-out engine. A later move would replace orchestration and delivery code; it would not move all of these core subsystems into Python. [Introduction][core-introduction], [RPC fields][core-introduction-fields], [client][core-introduction-client], [init][core-init]

Conversely, moving directly to charm ownership makes several pieces of *existing* core code candidates for retirement: non-bootstrap controller admission inside unit introduction, preservation of the special controller container, and core-generated workload startup plans. Those are existing maintenance obligations, not additional JU202 implementation that can all be counted as avoided work.

### Resource publication and validation

Image-resource publication is a separate addition. The existing Juju image build already stages the controller snap's binary and native libraries into the operator image. The local QA Jenkins configuration invokes the existing image-push targets using prebuilt payloads. Reuse this artifact-production path; add resource upload/registration and explicit release association instead of introducing another native binary build in the charm repository. A release must identify a compatible controller version, charm revision and OCI resource revision/digest, with architecture support checked. Charmcraft supports attaching a specified resource revision when releasing a charm. [Image build contract][core-image-build], [QA image publishing][qa-image-publish], [Charmcraft release][charmcraft-release]

The controller model's agent version and workload-resource version must have a defined compatibility relationship. They must not be independently advanced by unrelated publishers. The same release-pairing problem also arises for JU202's machine snap resource; it is a shared requirement of resource-based workload delivery.

The present charm workflow installs Juju `4.0/stable` while the charm metadata requires Juju `>= 4.2`, and its success check is application `active`. It is not sufficient evidence for the proposed standalone lifecycle. Testing the intended Juju/charm/resource combination, HA joins, retained-volume restart and patch rollouts is required whichever ownership and packaging approaches are selected. Live GitHub settings, Charmhub permissions and production release-job configuration were not inspected; the local Jenkins checkout establishes an existing build path, not that every release integration is already available. [CI integration][charm-ci-integration], [charm requirement][charm-assumes]

### Recommended sequence

1. Define the boundary between ordinary unit introduction and charm-coordinated controller enrolment, including durable identity, startup material and retry semantics. Reuse the machine proposal's core identity and shared-material operations where their semantics match.
2. Establish image-resource association, bootstrap adoption, workload startup and durable local restart without a live API. Prefer completing this Kubernetes ownership transfer during scale-out work if the schedule allows.
3. Implement patch upgrades against that ownership model, including compatible artifact selection, preflight before disruption and a rollout contract respected by the application reconciler.

Machine and Kubernetes lifecycle implementations should be explicit behind common controller logic. Their packaging can be qualified alongside this work without determining the ownership boundary.

If scale-out must ship first, JU202's introduction/runtime adapter is a defensible temporary implementation. Keep its scope narrow, preserve the eventual shared enrolment contract, and do the ownership transfer before patch-upgrade orchestration. Splitting packaging now while leaving all lifecycle decisions until after patch upgrades would avoid little of the more expensive rework.

## Scope and next decisions

Two separable changes are available:

- Make the workload image a charm resource while retaining current controller-specific admission temporarily. This establishes resource ownership but leaves much of the current startup contract in core.
- Move runtime preparation and workload startup into the charm, and move controller-node enrolment after ordinary unit introduction. This provides the fuller alignment with the machine design.

The second is the proposed target. It is broader than JU202's present Kubernetes bundle adaptation and its exclusion of convergence between platform implementations. JU202 would need to state that broader scope explicitly. The rationale's claim that controller-specific scale-out handling disappears also needs to distinguish machine provisioning, ordinary Kubernetes unit provisioning, and the controller-specific bootstrap and recovery responsibilities that remain.

The main decisions still to resolve are:

1. Whether to introduce image-resource ownership and charm-managed enrolment together or in stages.
2. The exact bootstrap and API-independent restart mechanism, including how it interacts with init containers and Pebble `--hold`.
3. The shared enrolment interface, durable controller identity mapping, secret ownership and recovery after leadership changes or partial writes.
4. Which `jujuagentd` responsibilities remain in Kubernetes controller pods and how they are configured and started.
5. Storage, socket, DNS and readiness contracts for both fresh pods and retained-volume restarts.
6. Patch-upgrade authorisation, agent/workload version compatibility, rollout control and recovery from interruption.
7. The secondary packaging choice: universal metadata/resource support or separate charm names, including bootstrap selection and existing-controller refresh compatibility.

A useful prototype must demonstrate adding a fresh replica and restarting the whole controller cluster from retained volumes. Further validation should cover leader changes during joining, interrupted configuration installation, stable credentials on retry, readiness before local membership and adoption of the bootstrap pod without an unintended rollout. Passing ordinary bootstrap or charm unit tests alone would not establish these lifecycle properties.

## Evidence from the current implementation

### How Kubernetes scale out currently works

The current implementation combines ordinary application scaling with controller-specific pod construction and admission.

1. `juju add-unit -m controller controller -n N` uses the Kubernetes application scaling API to increase desired scale. The application provisioner reconciles that target with the StatefulSet. [CLI scaling][core-addunit]
2. Before scaling, the provisioner creates or recovers an introduction nonce for each desired controller ordinal. It persists the nonce and projects it into the controller ConfigMap. [Nonce reconciliation][core-nonces]
3. Kubernetes creates the new pod from the controller StatefulSet template, with its persistent volume and stable network identity. Bootstrap originally constructed the controller workload container directly and selected its image from Juju's configured repository and version. The image is not currently a charm resource. [Image selection][core-image], [pod construction][core-bootstrap-pod]
4. Application reconciliation preserves the bootstrap-created `api-server` container, configuration seed init container and supporting mounts. The ordinary charm container declarations do not reproduce those components. [Template preservation][core-preserve]
5. The joining pod's init container calls `CAASApplication.UnitIntroduction` against an existing controller. Controller-specific handling validates model and application scope, ordinal and nonce, registers controller-node state, and returns controller-agent configuration as well as ordinary unit-agent configuration. [Introduction][core-introduction]
6. The init container installs the agent configurations and writes the unit configuration template last as its completion marker. Core-generated startup scripts then expect to start `jujud` and `jujuagentd` under Pebble. Ordinal zero runs bootstrap when needed; later ordinals wait for their own runtime configuration. [Init installation][core-init], [controller startup][core-startup]

The current workload readiness probe checks the API endpoint. JU202 proposes strengthening readiness to require local dqlite membership as well. An active charm status is not, by itself, the mechanism that changes Kubernetes Service endpoints. [Current readiness probe][core-readiness]

### The unfinished runtime configuration handoff

At the assessed revisions, a fresh scale-out pod has no implemented producer for the `runtime.conf` required by the standalone controller startup script. The presence of controller-specific admission and agent configuration does not complete that handoff.

For example, after successful introduction of `controller-1`, the relevant files are:

| File | Purpose | Current producer for a fresh joining pod |
| --- | --- | --- |
| `/var/lib/juju/template-agent.conf` | Configuration for the charm's unit agent; also the introduction completion marker | `UnitIntroduction`, installed by `containeragent init` |
| `/var/lib/juju/agents/machine-1/agent.conf` | Configuration for the controller-tagged machine agent | `UnitIntroduction`, installed by `containeragent init` |
| `/var/lib/juju/controller/runtime.conf` | Configuration for the standalone `jujud` process | Missing from the introduction and charm paths at these revisions |
| `/var/lib/juju/controller/controller.conf` | Dynamic configuration including peer bind-address topology | Controller charm; this is a different file and does not substitute for `runtime.conf` |

The missing handoff is visible at each boundary:

1. The RPC response has `UnitName`, `AgentConf`, `ControllerAgentTag` and `ControllerAgentConf`. There is no runtime configuration field. The client mirrors those fields. [RPC response][core-introduction-fields], [client conversion][core-introduction-client]
2. `containeragent init` writes the controller-agent configuration, then the unit configuration template. It does not write `controller/runtime.conf`. [Init installation][core-init]
3. The bootstrap seed container copies the initial runtime configuration only inside the ordinal-zero branch. Non-zero pods obtain their nonce but do not copy the bootstrap node's runtime file. [Ordinal-zero seeding][core-seed]
4. The non-zero workload startup script waits for `$JUJU_CONTROLLER_DIR/runtime.conf`. Its comment says later replicas have that file seeded by the charm. The current charm has a runtime reader but no corresponding initialisation writer. [Startup expectation][core-startup], [charm runtime reader][charm-runtime]

The wait occurs before the script writes the Pebble layer and executes Pebble. Consequently, for a new replica with a fresh volume and otherwise successful provisioning, the source path reaches the runtime-file wait instead of starting the local `jujud` and `jujuagentd` services. Startup probes may cause retries; they do not create the missing file. The charm sidecar is a separate container and may run, but its current implementation cannot fulfil the missing handoff. [Script execution order][core-startup-script]

This is specifically a fresh-replica gap. Ordinal-zero bootstrap renders and seeds runtime configuration. A restart with the required configuration already on its persistent volume does not encounter the same absence. This assessment does not establish that all Kubernetes controllers or all restarts fail.

The files are not interchangeable. The bootstrap runtime renderer includes controller identity, paths, API settings, controller-node credentials, CA material and controller TLS material. The standalone command reads this runtime file explicitly. Copying `agent.conf`, or reusing ordinal zero's identity-bearing runtime configuration, would not implement a correct join. [Runtime rendering][core-runtime-render], [standalone runtime reader][core-runtime-reader]

JU202's current solution is to extend introduction to return the final per-node runtime configuration and have init install it durably before marking introduction complete. The proposed charm-owned design would instead make the joining charm assemble and install runtime material after ordinary unit introduction. Either design must assign this work to an actual producer and preserve identity across retries.

## Charm packaging as a secondary choice

The workload ownership proposal above applies to either packaging choice. The distinction here is how to compose and publish the machine and Kubernetes implementations, and what support that requires from Juju and CI.

The recommendation is to compose machine and Kubernetes runtime, address and upgrade implementations within one charm first, then qualify universal packaging against the target Juju release. If the metadata/resource compatibility work cannot fit, those same implementations can be packaged separately. This preserves the internal design and the transfer of workload ownership while leaving packaging as a bounded choice.

### Separate substrate lifecycle implementations

The existing unified charm mostly configures and integrates an already-running controller. Its install hook creates `controller.conf`; its start hook sets active status. It does not yet implement JU202's machine installation and enrolment workflow. [Current charm lifecycle][charm-lifecycle]

JU202 gives the platforms substantially different responsibilities:

| Responsibility | Machine path in JU202 | Kubernetes path in JU202 |
| --- | --- | --- |
| Supply controller software | Charm installs the selected snap resource | Core provisions the controller image |
| Establish joining identity | Charm leader coordinates core operations and Juju secrets | Core admits the pod through nonce-validated introduction |
| Supply initial runtime configuration | Charm assembles it and invokes `jujud init` | Init container receives and installs it |
| Start processes | Charm starts the snap after join prerequisites exist | Core-generated Pebble configuration starts the binaries |
| Maintain topology | Charm publishes and aggregates peer addresses | Charm publishes and aggregates peer addresses |
| Report readiness | API health and local dqlite membership | The same conditions, also reflected in Kubernetes Service eligibility |

The current charm contains assumptions that need revisiting as it takes on installation:

- `_is_snap()` detects `/var/snap/jujud`. On a fresh machine before snap installation, it selects the non-snap layout. The platform must be known independently of whether installation has happened. [Platform detection][charm-platform]
- Topology publication obtains the controller ID from a live configuration socket and requests a reload after writing configuration. A new machine must publish topology before starting its controller, so initial configuration needs an identity and write path that do not require the local process. [Topology handling][charm-topology]
- When Kubernetes ingress addresses cannot be represented as IPs, the charm skips bind-address publication, with a comment that Kubernetes does not use dqlite clustering. That assumption conflicts with the intended HA path. [Address handling][charm-address]

Separate implementations would make installation, startup, recovery and teardown explicit for each platform. These can be composed within one charm artifact. Shared modules should retain dashboard, metrics, tracing, Loki and S3 integrations, socket clients, configuration schemas and topology logic whose semantics match. Dqlite startup and membership operations remain in `jujud`; the charm provides configuration and observes their results. [Dqlite joining][core-dbjoin]

PostgreSQL is a useful comparison: its machine and Kubernetes charms have different snap/Pebble lifecycles while sharing code through `postgresql-single-kernel-library`. The useful precedent is separate platform lifecycles with shared product logic. [PostgreSQL architecture][postgresql-architecture]

If separate artifacts are selected, publishing a second charm name requires coordinated Juju changes. Bootstrap defaults, local charm validation, manual-deployment restrictions and controller-application removal protection currently recognise `juju-controller`. Preserve the application name `controller` and existing integration endpoints, while making charm selection and protection recognise both variants. Existing-controller refresh compatibility needs a defined transition. [Bootstrap validation][core-charm-validation], [deployment restriction][core-deploy-protection], [removal protection][core-remove-protection]

### Universal controller charm assessment

Valkey provides a concrete alternative to separate charm identities. Its metadata declares containers, OCI resources and storage mount locations together with machine storage locations and both sets of integration endpoints. Its constructor selects a machine or Kubernetes workload implementation, then supplies that implementation to common managers. Upgrade implementations are selected separately. This is declarative metadata combined with runtime composition; it does not require every lifecycle operation to be identical. [Valkey metadata][valkey-metadata], [workload and refresh selection][valkey-composition]

The controller can have distinct snap and Pebble implementations without adding another charm name, a second set of Charmhub permissions or substrate-based charm-name selection in bootstrap. A shared controller release already has substantial cross-substrate compatibility obligations, making one identity a reasonable target. Universal packaging still requires testing each platform, publishing its workload artifact, and adopting existing controller pods safely.

#### Author identified gaps and controller implications

Rene Radoi identifies substrate detection without cloud-credential trust, ambiguous metadata, Charmhub's single-substrate labelling and incomplete first-class support for workload-bearing universal charms as ecosystem gaps. The article also describes choosing common lifecycle events and retaining substrate-specific upgrade implementations. [Universal charm article][valkey-article]

For controller design, the implications are:

- **Reliable substrate selection:** Valkey selects through `KUBERNETES_SERVICE_HOST`. Its Kubernetes branch still checks cloud access because its operations need that permission; avoiding trust merely to identify a machine does not remove Kubernetes operational permissions. Prefer a supported, immutable model-substrate indication from Juju/Ops. Do not infer substrate from snap installation or the mere presence of `containers` in universal metadata. [Selection implementation][valkey-composition]
- **Explicit platform support:** The inspected Juju `main` has `Meta.IsSidecar()` defined as a non-empty container declaration, and ordinary deployment rejects that metadata on a machine model unless forced. Removing `k8s-api` from `assumes` does not bypass this independent check. The controller uses a special bootstrap path, so this is not proof that its bootstrap currently fails; it is a concrete compatibility issue that a supported universal design must address. Prefer explicit supported-substrate semantics over relying on `--force`. [Metadata classification][core-model-mismatch], [CLI enforcement][core-model-mismatch-cli], [repository deployment enforcement][core-model-mismatch-api]
- **Useful event coverage:** Valkey observes `start` and other common lifecycle events and checks workload readiness. The controller need not restrict itself to common events. Let the Kubernetes adapter observe Pebble availability and health events and feed the same reconciliation logic used by machine events. A one-time `start` event is not a substitute for all workload-container restart notifications. [Valkey event wiring][valkey-events], [Ops workload lifecycle][ops-containers]
- **Precise address contracts:** Use separate concepts for a local dqlite bind address, peer reachability and the advertised client API endpoint. These may have different IP/DNS and readiness requirements. A single generic `endpoint` property would conceal distinctions needed by controller membership and bootstrap.
- **File operations with lifecycle guarantees:** Pathops is useful for ordinary local/remote configuration access, but controller-runtime installation needs a stronger operation: validate the identity and configuration, set private permissions, install durably and mark completion last. Its remote implementation also requires reachable Pebble. Use a controller initialisation operation or an explicitly tested installer contract rather than assuming a pathlib-like interface supplies the complete transaction. [Pathops reference][pathops-reference]

#### Juju version clarification for universal deployment

The deployment validation was checked separately against the public branch heads on 20 November: `3.6` at `5618f6ea0081019400edc3479f4cb2424817195a`, `4.1` at `59dcd6d951814fa098a11ca3be103314bb6894e8`, and `main` at `db7d6dd02700e9f50b66bad40e4f9e9bdd857ede`. This comparison updates the scope of the metadata finding without changing the earlier runtime assessment's pinned baseline.

The corresponding 3.6 CLI and repository-deployment paths do not contain the container-based model-mismatch rejection. Valkey's published requirements identify Juju 3.6 LTS as its tested version. The inspected 4.1 and main code do contain the rejection: a normal machine deployment of Valkey's universal metadata is rejected before the workload implementation runs. Deployment onto Kubernetes does not trigger this rejection. [Valkey supported Juju][valkey-requirements], [3.6 deployment path][core36-deploy], [4.1 classification][core41-classification], [current main classification][core-main-classification]

Both the inspected 4.1 CLI check and repository-deployment facade downgrade this error to a warning when `--force` is supplied. That establishes a bypass for this particular validation, not end-to-end Valkey compatibility on 4.1. The rejection was introduced by commit `7a8e8b67c13aec73cf5cd09346a6e54bcae74f20`, dated 2 July 2026. The claim is revision-specific; it does not assert that every historical 4.1 build contains the check. Existing running deployments are not shown to fail by a check on new deployment. [4.1 CLI][core41-deploy-cli], [4.1 repository deployment][core41-deploy-api], [introducing change][core-mismatch-change]

The underlying incompatibility is a classification assumption: declaring containers is treated as proof that a charm has no machine implementation. Valkey invalidates that assumption. Supporting universal charms requires recognising the additional supported substrate, rather than changing Valkey's snap execution merely to satisfy the classifier.

#### Resource delivery is not identical to Valkey

Valkey's metadata declares an OCI resource, while its machine implementation installs and holds a Snap Store revision selected from `refresh_versions.toml`. JU202 instead specifies a supplied, revisioned snap file resource. A universal controller charm therefore needs qualification of the combined file-resource and OCI-resource metadata, substrate-appropriate bootstrap association, and refresh of those resources. Valkey's success does not establish that exact delivery contract. [Valkey snap installation][valkey-snap]

The extra Juju work changes shape: one artifact avoids charm-name selection and dual-name protections, but requires reliable substrate identification, supported universal metadata and correct handling of resources for each platform. The runtime handoff and Kubernetes adoption work remain either way. These are the checks that should determine packaging, rather than assuming one artifact is automatically cheaper overall.

### Juju changes if separate charm identities are selected

If separate artifacts are needed, the selection change has a manageable scope:

1. Select the default charm name from the controller model's substrate, including remote and local bootstrap paths. Carry that selection consistently through download, bootstrap arguments and controller application creation.
2. Validate local archives against the intended substrate's variant, and recognise both supported variants in controller-charm deployment and removal protections. There is also a domain-layer `IsControllerCharm` implementation comparing metadata to the literal `juju-controller`; changing one bootstrap constant is insufficient.
3. Define the transition for existing controller applications. Keep the application name `controller`, its identity, relations and storage stable; do not silently switch an already-running controller to a different charm merely because a default changed.

Keeping the machine artifact named `juju-controller` and introducing, for example, `juju-controller-k8s` minimises machine-side renaming. That is a proposed naming choice, not a requirement. A central substrate selection policy and explicit controller-application identity are preferable to scattering more string comparisons. This does not require a second scaling worker or a new model type. [Default name][core-charm-default], [bootstrap deployer][core-charm-deployer], [domain recognition][core-charm-recognition]

### Packaging and publishing changes

The charm repository already has one build/upload pipeline and a substrate-specific integration matrix: three LXD bases and one MicroK8s entry. It packs one project, uploads one artifact named `charm`, and releases under one `CHARM_NAME`. The token-generation script scopes authentication to one charm name. An artifact split would need two packaging targets, separate artifact names and upload targets, and publishing authority/tracks for the second name. The LXD tests would consume the machine artifact and MicroK8s the Kubernetes artifact; there is no need to duplicate every cloud test for each artifact. A universal artifact retains the single-name pipeline and both substrate test paths. Shared code can retain shared unit tests in either design. [Charm workflow][charm-ci], [publishing token scope][charm-token]

## Source baseline

The assessment uses these exact revisions:

| Source | Revision |
| --- | --- |
| Juju local `main` and `origin/main` in `~/projects/canonical/juju` | `234537adf971e9c5be4998af75fe8df2f4388255` |
| Controller charm `main` in `~/projects/canonical/juju-controller` | `8e7d7fe2c0bd933d529dbab45faa3e3b0be26c74` |
| Local `~/projects/canonical/juju-qa-jenkins` | `457720369480520fa75673f388d58f0baef37055` |
| JU202 | Local document dated 17 November 2026, pending review |
| Valkey `9/edge`, examined 20 November | `d0e9e4eff12e240d6d3f963f248df2f9a9e6e06c` |
| `charm-refresh` source examined 20 November | `01455bb59d3adfcb529379045fd240614b0db001` |

On 19 November the Juju working checkout was `recovery-pr0` at `485876fd2bc46a0142b45daaf445ef7fd6c3a458`. The assessment continued against the local `main` revision without switching or modifying that checkout. Source links are pinned to the assessed revisions.

[charm-lifecycle]: https://github.com/juju/juju-controller/blob/8e7d7fe2c0bd933d529dbab45faa3e3b0be26c74/src/charm.py#L221-L284
[charm-platform]: https://github.com/juju/juju-controller/blob/8e7d7fe2c0bd933d529dbab45faa3e3b0be26c74/src/charm.py#L50-L65
[charm-topology]: https://github.com/juju/juju-controller/blob/8e7d7fe2c0bd933d529dbab45faa3e3b0be26c74/src/charm.py#L542-L577
[charm-address]: https://github.com/juju/juju-controller/blob/8e7d7fe2c0bd933d529dbab45faa3e3b0be26c74/src/charm.py#L579-L609
[charm-runtime]: https://github.com/juju/juju-controller/blob/8e7d7fe2c0bd933d529dbab45faa3e3b0be26c74/src/charm.py#L645-L675
[core-dbjoin]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/worker/dbaccessor/worker.go#L920-L954
[core-charm-validation]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/cmd/juju/commands/bootstrap.go#L410-L423
[core-deploy-protection]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/apiserver/facades/client/application/deploy.go#L89-L94
[core-remove-protection]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/apiserver/facades/client/application/application.go#L1125-L1135
[core-addunit]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/cmd/juju/application/addunit.go#L231-L241
[core-nonces]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/worker/caasapplicationprovisioner/ops.go#L787-L814
[core-image]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/cloudconfig/podcfg/image.go#L31-L64
[core-bootstrap-pod]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/bootstrap.go#L1597-L1668
[core-preserve]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/application/application.go#L476-L550
[core-introduction]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/apiserver/facades/agent/caasapplication/application.go#L197-L385
[core-init]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/cmd/containeragent/initialize/command.go#L194-L210
[core-startup]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/bootstrap.go#L1505-L1594
[core-readiness]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/bootstrap.go#L1400-L1407
[core-introduction-fields]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/rpc/params/caas.go#L20-L26
[core-introduction-client]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/api/agent/caasapplication/client.go#L38-L79
[core-seed]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/bootstrap.go#L1674-L1728
[core-startup-script]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/caas/scripts.go#L34-L54
[core-runtime-render]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/bootstrap.go#L913-L965
[core-runtime-reader]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/cmd/jujud/agent/controller.go#L273-L299
[core-resource-map]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/worker/caasapplicationprovisioner/ops.go#L257-L277
[core-resource-read]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/worker/caasapplicationprovisioner/ops.go#L1010-L1028
[core-workload]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/application/application.go#L2018-L2051
[core-unit-config]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/apiserver/facades/agent/caasapplication/application.go#L273-L287
[core-adoption]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/application/application.go#L229-L240
[core-ephemeral]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/application/application.go#L2232-L2242
[core-persistent]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/bootstrap.go#L1752-L1781
[core-init-check]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/cmd/containeragent/initialize/command.go#L148-L154
[core-upgrade]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/upgrade.go#L93-L173
[postgresql-architecture]: https://canonical.com/data/postgresql/docs/latest/explanation/architecture/
[ops-containers]: https://canonical.com/juju/docs/ops/latest/howto/manage-containers/manage-the-workload-container/
[core-charm-default]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/environs/bootstrap/prepare.go#L25-L26
[core-charm-deployer]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/bootstrap/deployer.go#L256-L309
[core-charm-recognition]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/domain/application/state/charm.go#L74-L108
[charm-ci]: https://github.com/juju/juju-controller/blob/8e7d7fe2c0bd933d529dbab45faa3e3b0be26c74/.github/workflows/ci.yml#L43-L163
[charm-token]: https://github.com/juju/juju-controller/blob/8e7d7fe2c0bd933d529dbab45faa3e3b0be26c74/.github/charmcraft-tokengen.sh#L5-L11
[core-image-build]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/Makefile#L848-L935
[qa-image-publish]: https://github.com/juju/juju-qa-jenkins/blob/457720369480520fa75673f388d58f0baef37055/jobs/ci-run/scripts/snippet_caas-jujud-operator.sh#L5-L11
[charmcraft-release]: https://canonical.com/juju/docs/charmcraftlatest/reference/commands/release/
[charm-ci-integration]: https://github.com/juju/juju-controller/blob/8e7d7fe2c0bd933d529dbab45faa3e3b0be26c74/.github/workflows/ci.yml#L185-L220
[charm-assumes]: https://github.com/juju/juju-controller/blob/8e7d7fe2c0bd933d529dbab45faa3e3b0be26c74/charmcraft.yaml#L15-L16
[core-patch-service]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/domain/controllerupgrader/service/service.go#L254-L316
[core-disabled-upgrades]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/cmd/jujud/agent/controller/manifolds.go#L1100-L1125
[core-caas-upgrader]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/worker/caasupgrader/upgrader.go#L59-L62
[core-sts-construction]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/application/application.go#L370-L393
[core-sts-reconcile]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/internal/provider/kubernetes/resources/statefulset.go#L86-L103
[core-app-config]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/caas/application.go#L111-L178
[k8s-sts-updates]: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/statefulset/#update-strategies
[valkey-article]: https://discourse.charmhub.io/t/universal-charm-how-valkey-runs-the-same-charm-code-on-k8s-and-machines/21026
[valkey-metadata]: https://github.com/canonical/valkey-operator/blob/d0e9e4eff12e240d6d3f963f248df2f9a9e6e06c/metadata.yaml#L24-L115
[valkey-composition]: https://github.com/canonical/valkey-operator/blob/d0e9e4eff12e240d6d3f963f248df2f9a9e6e06c/src/charm.py#L83-L121
[valkey-events]: https://github.com/canonical/valkey-operator/blob/d0e9e4eff12e240d6d3f963f248df2f9a9e6e06c/src/events/base_events.py#L68-L138
[valkey-snap]: https://github.com/canonical/valkey-operator/blob/d0e9e4eff12e240d6d3f963f248df2f9a9e6e06c/src/workload_vm.py#L177-L207
[core-model-mismatch]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/domain/deployment/charm/meta.go#L282-L312
[core-model-mismatch-cli]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/cmd/juju/application/deployer/charm.go#L100-L109
[core-model-mismatch-api]: https://github.com/juju/juju/blob/234537adf971e9c5be4998af75fe8df2f4388255/apiserver/facades/client/application/deployrepository.go#L691-L701
[pathops-reference]: https://canonical.com/juju/docs/charmlibs/reference/charmlibs/pathops/
[refresh-partition]: https://github.com/canonical/charm-refresh/blob/01455bb59d3adfcb529379045fd240614b0db001/charm_refresh/_main.py#L723-L745
[refresh-permission]: https://github.com/canonical/charm-refresh/blob/01455bb59d3adfcb529379045fd240614b0db001/charm_refresh/_main.py#L1367-L1391
[refresh-stop]: https://github.com/canonical/charm-refresh/blob/01455bb59d3adfcb529379045fd240614b0db001/charm_refresh/_main.py#L1448-L1475
[refresh-constraints]: https://github.com/canonical/charm-refresh/blob/01455bb59d3adfcb529379045fd240614b0db001/docs/modules/ROOT/partials/juju-refresh.adoc#L56-L63
[valkey-requirements]: https://canonical.com/data/valkey/docs/latest/reference/requirements/
[core36-deploy]: https://github.com/juju/juju/blob/5618f6ea0081019400edc3479f4cb2424817195a/cmd/juju/application/deployer/charm.go#L57-L83
[core41-classification]: https://github.com/juju/juju/blob/59dcd6d951814fa098a11ca3be103314bb6894e8/domain/deployment/charm/meta.go#L282-L312
[core-main-classification]: https://github.com/juju/juju/blob/db7d6dd02700e9f50b66bad40e4f9e9bdd857ede/domain/deployment/charm/meta.go#L282-L312
[core41-deploy-cli]: https://github.com/juju/juju/blob/59dcd6d951814fa098a11ca3be103314bb6894e8/cmd/juju/application/deployer/charm.go#L100-L109
[core41-deploy-api]: https://github.com/juju/juju/blob/59dcd6d951814fa098a11ca3be103314bb6894e8/apiserver/facades/client/application/deployrepository.go#L691-L701
[core-mismatch-change]: https://github.com/juju/juju/commit/7a8e8b67c13aec73cf5cd09346a6e54bcae74f20

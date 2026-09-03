# MicroCloud project-per-model tenancy

Status: exploratory design note

This document records the requirement, feasibility findings, security boundary,
and implementation considerations for assigning one LXD project to each Juju
model on MicroCloud. It is not an accepted implementation design, and no
implementation work is currently planned.

The Juju source observations were verified on branch `4.0` at commit
`4b0603428e` on 2026-09-03. They must be revalidated before implementation.

## Requirement

Juju must support MicroCloud as an IAAS substrate. For this purpose, MicroCloud
is principally an LXD cluster with MicroOVN networking and, where configured,
MicroCeph-backed storage.

One proposed tenancy model is to create a separate LXD project for every Juju
model. The project should:

- isolate the model's instances, profiles, networks, images, and storage
  volumes from other models;
- impose aggregate compute, memory, storage, instance, and network limits;
- restrict security-sensitive LXD features to those required by Juju;
- prevent Juju operations for one model from targeting another model's
  resources; and
- have lifecycle and ownership tied to the Juju model.

The design must not give the model's normal provider operations permission to
create arbitrary projects, change project limits, change cluster-global
configuration, or access other projects.

## Feasibility conclusion

A project per model is feasible at the LXD layer and is a suitable MicroCloud
tenancy boundary. Strong isolation, however, requires two distinct authority
levels:

1. A privileged tenancy authority creates and configures the project and
   provisions access to it.
2. A unique model credential operates only within that project.

The privileged authority must not become the model's long-lived cloud
credential. A single unrestricted credential combined with different project
names provides useful namespace and quota separation, but it is not an
authorization boundary.

The cleanest initial architecture is an external MicroCloud tenancy broker or
administrator-managed workflow. Juju would receive only a project name and a
project-restricted credential. If Juju later owns automatic project creation,
it will need an explicit controller-scoped tenancy capability separate from
ordinary model credentials and provider environments.

## Security boundary

The LXD project and the LXD identity serve different purposes:

- **Project policy**: Defines isolated resources, quotas, permitted uplinks,
  allocated subnets, and restrictions on security-sensitive instance
  configuration.
- **Project-scoped identity**: Prevents the caller from selecting another
  project or weakening the selected project's policy.
- **Tenancy authority**: Creates and deletes projects, sets their policy, and
  issues or revokes project-scoped identities.

The intended flow is:

```text
Juju model lifecycle
    |
    +--> privileged tenancy manager
           |-- ensure the project and its policy
           |-- seed the project network and profiles
           `-- issue access restricted to that project

Juju model provider workers
    |
    `--> unique project credential --> one LXD project only
```

The model credential is used by controller-side Juju provider workers on
behalf of the model. It must not be exposed to workloads or reused by another
model.

### Shared unrestricted credentials

Using a cluster-wide credential for every model does not satisfy the strong
isolation requirement. Juju currently stores the LXD project as model
configuration. A caller able to update that configuration can select another
project, and an unrestricted LXD credential can authenticate the resulting
request.

Project quotas may still be useful with a shared credential because they limit
resources created while Juju is targeting the intended project. This is an
operational guardrail, not containment against credential misuse, a provider
bug, or project retargeting.

### Restricted LXD identities

LXD supports TLS certificates restricted to one or more projects. A restricted
certificate cannot perform global configuration changes or alter the limits
and restrictions of its allowed projects. This mechanism exists in the LXD
5.21 baseline used by the MicroCloud 2.1 LTS and is compatible with Juju's
current certificate-based LXD authentication model.

Fine-grained LXD authorization can express similar or narrower permissions,
but the initial design must not depend on authorization capabilities newer than
the supported MicroCloud baseline. Juju's current credential finalization uses
the traditional LXD certificate trust endpoint rather than the fine-grained
identity and group workflow.

## Current Juju behavior

### Project selection

The LXD `project` setting is provider-specific model configuration and defaults
to `default`:

- `internal/provider/lxd/config.go`

When Juju opens the provider environment, it combines the model's cloud
credential with the configured project and calls `UseProject` on the LXD
client:

- `internal/provider/lxd/environ.go`
- `internal/provider/lxd/server.go`

The project name is not a cloud-credential attribute and is not inherited from
the local `lxc` client's selected project.

Juju currently validates a changed LXD provider configuration without making
the project immutable. An automatically assigned project should therefore
become system-owned configuration or otherwise reject changes after model
creation.

### Project lifecycle

The current LXD provider selects projects but does not create, configure, or
delete them. Its localized `Server` interface exposes `UseProject` but no
project lifecycle methods. `PrepareForBootstrap` is a no-op.

Opening an LXD environment immediately performs these operations:

1. Connect using the model credential.
2. Select the configured project.
3. Validate the remote server in that project.
4. Ensure a model-specific Juju profile exists.

As a result, a configured project must already exist and be usable before the
current environment can be opened. The generic hosted-model
`CreateModelResources` hook runs too late to create a missing project through
the environment as it is constructed today:

- `domain/model/service/modelservice.go`

The LXD provider does not currently implement that hosted-model resource
interface in any case.

### Model credentials

Model creation selects one existing Juju cloud credential and associates it
with the model:

- `apiserver/facades/client/modelmanager/modelmanager.go`
- `domain/model/types.go`

There is no separate project-provisioning credential and model-runtime
credential in the creation contract.

LXD interactive credential finalization occurs client-side. Juju consumes a
trust token, uploads a client certificate, and stores the resulting client
certificate, private key, and server certificate:

- `internal/provider/lxd/credentials.go`

When Juju generates the client certificate, it caches and reuses it. Reusing
one certificate fingerprint is unsuitable for independently restricted model
identities: each automatically managed model credential must have a distinct
certificate and independently revocable LXD trust entry.

### Credential finalization and project selection

An LXD certificate's project restriction is an authorization allow-list. It
does not select a current project for API requests. The LXD API treats a
project-scoped request without an explicit `project` parameter as a request for
the `default` project. `UseProject` configures the client to add that parameter;
it is therefore required when using a credential confined to a non-default
project. A restricted credential can establish a connection and perform some
project-independent operations, but project-scoped operations against
`default` are denied if `default` is not in its allowed project list.

This exposes a coupling in current remote credential finalization:

1. Juju redeems the trust token and obtains the LXD server certificate.
2. It reconnects through `ServerFactory.RemoteServer` without setting
   `CloudSpec.Project`.
3. `RemoteServer` calls `bootstrapRemoteServer`, which validates the server by
   reading the `default` profile and potentially ensuring default storage.
4. Since no project was selected, those operations target the LXD `default`
   project and fail for a certificate restricted only to the intended model
   project.

The project need not become a credential attribute to fix this. Credential
finalization should be project-neutral and should establish only that:

- the trust token can register the generated client certificate;
- the pinned server certificate is valid for the endpoint; and
- the resulting client certificate can authenticate to the server.

The server factory should separate connecting to a remote LXD server from
opening and initializing a Juju environment. Credential finalization should use
a connection-only path that performs no profile, storage, network, or instance
operations. Environment construction should continue to take the project from
model configuration, call `UseProject`, and only then validate the project and
initialize its resources.

The LXD pending trust-token operation already records whether the certificate
is restricted and which projects it permits. LXD applies that policy when the
token is redeemed, so Juju does not need to know the project merely to finalize
the credential.

Making `project` a required credential field would duplicate model
configuration and create conflicting sources of truth. It would also model an
LXD certificate incorrectly because one certificate can authorize more than one
project while each client request targets exactly one project. If the product
later enforces a strict one-credential-per-model-project policy, the credential
could carry an optional project binding or hint. Model creation could use that
to default the model setting and reject mismatches, but the authoritative
runtime selection should remain system-owned model configuration applied with
`UseProject`.

Without a new controller-level tenancy or super-credential, this change fixes
restricted-token onboarding but does not automate project creation. An external
administrator or broker must pre-create the project and issue the restricted
token. Juju then finalizes the credential independently and creates the model
with the matching explicit project configuration.

## Project construction for MicroCloud

A newly created LXD project is not automatically equivalent to MicroCloud's
configured `default` project. MicroCloud initialization configures networking,
storage, and the default profile in the `default` project. A per-model project
needs its own deliberate configuration.

### Project feature isolation

For multi-tenant access using restricted TLS certificates, all relevant LXD
project features must be isolated. In particular, the project must not inherit
an editable network, profile, image set, or storage-volume namespace from the
`default` project.

LXD specifically warns that a restricted certificate can edit inherited
resources in the `default` project when the corresponding project feature is
disabled. `features.networks` is disabled on a newly created project unless it
is explicitly enabled.

The initial policy should explicitly set all supported `features.*` isolation
keys rather than rely on LXD creation defaults. The exact keys must be selected
from the LXD version supported by the target MicroCloud release.

### OVN networking

With network isolation enabled, each model project needs a project-local
managed OVN network. That network should use a cluster-global MicroCloud uplink
without giving the model permission to modify the uplink itself.

The tenancy policy must define:

- the permitted uplink network;
- any routed subnets or external address ranges allocated to the project;
- limits on networks, forwards, load balancers, and related resources;
- the project-local OVN network configuration; and
- the NIC in the project's `default` profile that refers to that network.

The current Juju remote LXD validation ensures that the default profile has a
root disk but does not create an OVN network or add a network device. A newly
isolated project with an empty default profile would therefore not provide the
networking assumed by Juju instances.

Whether the tenancy manager or the Juju LXD provider creates the project-local
OVN network remains open. The policy-sensitive allocation of uplinks and
subnets belongs to the tenancy authority. A narrowly scoped model credential
could create its own logical network only after the project policy has granted
the required uplink and subnet access.

### Profiles and storage

The project must have a usable `default` profile containing:

- a root disk backed by an approved MicroCloud/LXD storage pool; and
- a NIC connected to the project-local OVN network.

Juju also creates a model-specific profile with `security.nesting=true` in
`internal/provider/lxd/environ.go`. An LXD project with `restricted=true`
normally blocks container nesting. The project policy must therefore allow
container nesting while retaining other restrictions, unless Juju stops
requiring that profile setting or a VM-only mode avoids it.

Any other Juju use of low-level devices, raw LXD configuration, snapshots,
backups, cluster-member targeting, or charm-supplied profiles must be audited
against the chosen project restrictions before declaring the policy complete.

## Recommended ownership model

### External tenancy broker

An external tenancy broker is the preferred first design. It owns the elevated
MicroCloud/LXD authority and exposes a narrow, policy-driven contract such as:

```text
EnsureModelTenant(controller UUID, model UUID, policy class)
    -> project name, scoped credential or one-time trust token

RemoveModelTenant(controller UUID, model UUID)
```

The broker, rather than the caller, chooses the project configuration. It can
enforce naming, limits, uplink allocation, allowed exceptions, audit logging,
credential uniqueness, and cleanup policy. Juju receives no general project or
identity-management permission.

The broker API must be idempotent. A repeated ensure request for the same
controller and model UUID must return the same tenant or reconcile it to the
declared policy.

### Juju-owned tenancy capability

If Juju later owns project provisioning directly, the elevated credential must
be controller- or cloud-scoped and represented by a narrow tenancy-management
capability. It must not:

- be placed in the model's `CloudSpec`;
- be selectable by a user as an ordinary model credential;
- be passed to model provider workers;
- be returned by model credential APIs; or
- widen the general `Environ` interface with cluster administration methods.

The capability should be invoked by a model lifecycle coordinator. The
model-scoped provider environment should continue to receive only the
restricted runtime credential.

This introduces a new Juju security domain. Reusing a user-owned, unrestricted
cloud credential as an implicit tenancy credential would obscure ownership and
make revocation and audit behavior unsafe.

## Proposed lifecycle

### Hosted-model creation

A future automatic workflow should have the following logical phases:

1. Allocate the Juju model UUID before making provider-side changes.
2. Derive a stable project identity from the controller UUID and model UUID.
3. Ask the tenancy authority to ensure the project, project policy, OVN
   network, and profiles.
4. Generate a unique model certificate and have the tenancy authority register
   it for only that project, or consume a project-restricted one-time token.
5. Store the resulting restricted credential and associate it with the model.
6. Persist the system-owned project name in model provider configuration.
7. Open the model environment and validate it using only the restricted
   credential.
8. Activate normal model provider workers.

The exact database and provider ordering needs a saga or reconciler. Current
model creation already spans the controller database, model database, and
provider side effects, and contains TODOs noting incomplete rollback. Adding a
project and identity makes explicit reconciliation necessary.

### Controller-model bootstrap

The controller model has the same problem earlier in its lifecycle. Its
project must exist before the first controller instance is provisioned.

The simplest initial workflow is to pre-provision the controller project's
restricted credential externally. A fully automatic workflow would bootstrap
with a tenancy capability, create the controller project, and ensure that the
long-lived credential stored in the controller is the restricted project
credential rather than the bootstrap authority. The current LXD
`PrepareForBootstrap` and bootstrap credential-finalization behavior do not
implement this transition.

### Model removal

Normal removal should proceed in this order:

1. Stop or fence new model provider operations.
2. Destroy model instances, storage volumes, profiles, networks, forwards, and
   other project resources using the restricted credential where possible.
3. Ask the tenancy authority to remove any residual project resources in
   accordance with force-removal policy.
4. Revoke the project-scoped LXD identity.
5. Delete the empty project.
6. Finalize Juju model removal.

The tenancy authority must safely handle retries and partially removed
projects. Credential revocation must not happen before the model has completed
the cleanup for which that credential is required, unless the tenancy authority
takes over all remaining cleanup.

### Migration and credential rotation

Model migration must preserve or deliberately transfer the association between
the model UUID, LXD project, and scoped identity. Open questions include
whether the destination controller receives the existing credential, registers
a new identity for the same project, or asks the tenancy broker to transfer
ownership.

Credential rotation must create and validate the replacement identity before
revoking the old identity. Rotation must not create a period where a model uses
an unrestricted credential.

## Naming and ownership metadata

Project names must not depend only on the mutable Juju model name. A suitable
scheme should include stable controller and model identity, for example a
bounded representation of:

```text
juju-<controller-uuid>-<model-uuid>
```

The exact representation must respect LXD project-name length and character
rules while remaining collision-resistant. The complete controller UUID and
model UUID should also be recorded as project metadata, along with a marker
identifying the project as Juju-managed.

Human-readable model and controller names can be stored as descriptive
metadata but must not be authoritative identifiers.

## Delivery options

### Manually provisioned proof of concept

The smallest useful validation requires no automatic project lifecycle in
Juju:

1. Pre-create a fully isolated LXD project.
2. Create its OVN network and default profile.
3. Configure representative limits and restrictions.
4. Register a unique TLS certificate restricted to that project.
5. Add the credential to Juju and create a model with the explicit LXD
   `project` setting.
6. Validate bootstrap or hosted-model provisioning, placement across cluster
   members, network forwards, storage, model destruction, and denied
   cross-project operations.

This proves the LXD provider's behavior under a genuinely restricted
credential before changing Juju's credential or lifecycle architecture.

### Broker-assisted lifecycle

The next stage can automate project and identity creation through an external
broker while keeping the existing Juju provider runtime scoped to one project.
Juju integration would still be required to persist a managed credential,
protect the project setting, and reconcile creation and deletion.

### Native Juju lifecycle

Native project management should be considered only after the broker or manual
workflow has established the required LXD permissions and resource policy. It
requires design work across bootstrap, hosted-model creation, managed
credentials, provider construction, removal, migration, and recovery.

## Open decisions

- Is a project per model mandatory for all MicroCloud models or an opt-in
  tenancy mode?
- Is project provisioning owned by MicroCloud, a separate tenancy service, or
  the Juju controller?
- Who supplies and rotates the privileged tenancy credential?
- How is an automatically managed model credential represented and owned in
  Juju without making it a normal user credential?
- Which project limits are fixed platform policy, and which can a Juju user
  request?
- Which LXD `features.*` and `restricted.*` settings are required for all Juju
  workloads?
- Must the first supported policy accommodate containers, VMs, or both?
- Does each model receive its own OVN network and routed allocation, and who
  performs IP address management?
- Can selected global resources, especially images, be shared read-only under
  the supported LXD authorization mechanism without weakening tenant
  isolation?
- How are controller-model projects bootstrapped without retaining the
  elevated bootstrap authority?
- What are the exact cleanup semantics for force removal, failed bootstrap,
  failed model creation, and an unavailable MicroCloud cluster?
- How is project ownership transferred during model migration?
- What upgrade behavior applies to existing MicroCloud models in the
  `default` project?

## Upstream references

- [MicroCloud releases and compatible component versions](https://documentation.ubuntu.com/microcloud/default/reference/releases-snaps/)
- [MicroCloud initialization](https://documentation.ubuntu.com/microcloud/latest/microcloud/explanation/initialization/)
- [LXD project architecture](https://documentation.ubuntu.com/lxd/default/explanation/projects/)
- [LXD project configuration](https://documentation.ubuntu.com/lxd/latest/reference/projects/)
- [LXD project creation](https://documentation.ubuntu.com/lxd/latest/howto/projects_create/)
- [LXD remote API authorization](https://documentation.ubuntu.com/lxd/latest/explanation/authorization/)
- [LXD OVN networks](https://documentation.ubuntu.com/microcloud/stable/lxd/reference/network_ovn/)

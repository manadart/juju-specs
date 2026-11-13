# Co-located Subordinate Charms on Kubernetes

Status: exploration.

This note explores supporting subordinate charms in Kubernetes models by running
the subordinate unit in the same pod as the principal unit. The intent is to
preserve the Juju subordinate application model while mapping colocation to a
Kubernetes pod instead of an IAAS machine.

## Current State

Subordinate charms are currently a machine-charm concept. Deploying a
subordinate creates an application with no units. Units are created later, when
a principal unit enters a container-scoped relation with the subordinate.

In the current implementation, that relation path assumes the principal unit is
machine-backed:

- `cmd/juju/application/deployer/charm.go` and
  `apiserver/facades/client/application/deploy.go` allow subordinate deploys
  with zero units and no constraints.
- `domain/relation/state/relation.go` calls `addSubordinateUnit` when a
  container-scoped relation enters scope.
- `domain/relation/state/subordinateunit.go` creates the subordinate unit by
  placing it on the principal unit's machine and calling `InsertIAASUnit`.
- `getUnitMachineIdentifier` returns `UnitMachineNotAssigned` if the principal
  unit is not assigned to a machine. That is the expected shape for CAAS units.

The result is that a Kubernetes subordinate may get as far as an application
record, but the functional unit creation path is not CAAS-capable.

## Goal

Support a subordinate application in a Kubernetes model such that:

- deploying the subordinate still creates no standalone workload;
- integrating it with a principal creates one subordinate unit per principal
  unit;
- each subordinate unit is co-located in the same pod as its principal unit;
- status and relation semantics remain recognisably subordinate-shaped;
- the subordinate application is not directly scaled, just as machine
  subordinates are not directly scaled.

In Kubernetes terms, a principal unit's pod would be composed from the principal
charm plus any currently related co-located subordinate charms.

## Non-goals

- Do not change machine subordinate semantics.
- Do not make subordinate charms deploy as independent Kubernetes workloads.
- Do not put business logic in API facades or provider glue.
- Do not require charm authors to manage Kubernetes pod patches directly.

## Proposed Model

A Kubernetes subordinate application remains an application with no desired
scale of its own. Its units are relation-derived. When a principal CAAS unit
enters scope for a compatible container-scoped relation, Juju creates a
subordinate CAAS unit and records its principal relationship.

The subordinate unit should not produce a separate StatefulSet, Deployment, or
Service. Instead, the principal application's desired pod template should include
the subordinate's runtime contribution.

At minimum, the model needs to represent:

- the subordinate unit;
- the principal unit it belongs to;
- the relation that caused the subordinate unit to exist;
- the subordinate charm resources and container definitions to add to the pod;
- lifecycle state for removing that contribution when the relation or
  subordinate application is removed.

The existing `unit_principal` relationship is probably still useful, but the
Kubernetes implementation should not infer pod colocation from machine
assignment. CAAS needs an explicit colocation path.

## Pod Composition

The principal pod becomes the convergence target for multiple charms:

- the principal charm contributes its existing charm/operator container and
  workload containers;
- each co-located subordinate contributes its own charm/operator container and
  any workload containers it owns;
- Juju owns the merged pod template and applies it through the existing
  Kubernetes application worker/provider path.

This implies a single pod-template composer. Multiple workers or charms must not
patch the same StatefulSet independently, or Juju will get conflicting desired
state.

Composition should validate conflicts before changing Kubernetes state:

- container names;
- mounted paths and volume names;
- ports;
- service account and RBAC needs;
- resource requests and limits;
- storage directives;
- device requests;
- environment variables and Juju agent config paths.

Updating the pod template will roll the principal application's pods. That is a
material behavior change and should be explicit in design and tests.

## Agent Runtime

The hard part is not only adding containers to a pod. Each subordinate unit needs
its own Juju identity and hook execution context:

- separate unit name, password, and agent config;
- separate hook tool credentials;
- separate relation state;
- separate logs and status;
- independent charm upgrade state;
- a clear answer for whether the subordinate can access the principal workload's
  Pebble socket, only its own Pebble socket, or both.

A prototype should first prove that two unit agents can run in one CAAS pod
without sharing mutable agent state accidentally.

## Pod Agent Worker Tree

One plausible implementation is to make the Kubernetes container agent act more
like the machine deployer: a pod-scoped agent process owns shared pod concerns
and starts one nested worker tree per Juju unit in that pod.

This matches comments in both existing unit manifold definitions. Today, the
container agent tree is still unit-shaped, but its own comments already identify
several workers as future consolidated-agent candidates. The leadership tracker
and uniter are explicitly called out as needing one instance per unit in such a
model. The machine deployer unit tree has the same comment.

That suggests a split like this:

- pod-scoped workers:
  - API address/config watching where it is truly shared;
  - probe HTTP server and CAAS prober;
  - pod-level SIGTERM handling;
  - pod template or container inventory watching;
  - shared HTTP client and log transport, if they can preserve unit identity;
  - migration gating and upgrade coordination that applies to the whole pod;
- unit-scoped workers:
  - unit agent wrapper and agent config;
  - API caller authenticated as that unit, unless server-side auth can safely
    multiplex identities;
  - leadership tracker;
  - hook retry strategy;
  - charm directory fortress;
  - uniter;
  - status, secret drain, trace, and logging components where identity matters.

The container agent would need a unit-set reconciler analogous to the IAAS
deployer, but the watched set would be "units assigned to this pod" rather than
"units assigned to this machine". For a principal pod, that set would include:

- the principal unit;
- each live subordinate unit whose `unit_principal` points at that principal
  unit;
- possibly units in teardown that still need relation-departed,
  relation-broken, stop, and remove hooks.

This design has two attractive properties:

- it avoids trying to run a separate containeragent process per subordinate
  container;
- it preserves Juju's per-unit worker semantics, so most existing uniter
  assumptions remain local to one unit tree.

It also creates hard boundaries to design carefully:

- each nested unit tree needs its own agent config directory and credentials;
- shared workers must never accidentally report logs, status, metrics, or traces
  under the wrong unit identity;
- pod termination must fan out to every nested unit and wait for orderly hook
  teardown within Kubernetes' grace period;
- probes need a clear aggregate result: a pod may contain one principal and
  several subordinate units with different readiness states;
- upgrades and migrations must decide whether the pod-level agent, all nested
  unit agents, or individual unit agents are allowed to progress independently;
- the uniter currently creates subordinate units during relation handling, so a
  CAAS consolidated-agent path must avoid recursive assumptions where a unit
  tree creates another unit tree that then needs pod-template reconciliation.

This approach changes the first prototype target. Instead of starting with pod
template composition, a lower-risk first prototype may be:

1. Run a container agent with a fixed list of two CAAS unit configs.
2. Start one uniter tree per unit in the same process.
3. Verify relation hooks, status, leadership, logging, and charm directories
   remain isolated.
4. Add a reconciler that starts and stops nested unit trees as the pod's unit set
   changes.
5. Only then connect that unit set to subordinate relation creation and pod
   template composition.

## Domain and State Sketch

Keep the layering consistent with existing domain boundaries:

- relation service decides that a subordinate unit is required;
- application service owns creation of unit UUIDs and construction of CAAS unit
  arguments;
- relation state records relation-scope and principal/subordinate links in the
  same transaction;
- application state persists CAAS subordinate units without creating machine
  placement;
- provider-facing application data exposes subordinate pod contributions for
  principal applications.

Possible new state/service operations:

- `CreateCAASSubordinateUnit`, taking supplied UUIDs and principal unit
  identity;
- `GetPodColocatedSubordinates`, returning subordinate units and charm
  contributions for a principal application or unit;
- watcher namespaces for principal pod-template changes caused by subordinate
  relation changes.

Avoid extending the current `addSubordinateUnit` machine path in place until the
IAAS and CAAS responsibilities are separated. The current function creates an
IAAS unit, asks for machine identifiers, and encodes machine placement; that is
the wrong abstraction for Kubernetes.

## Provisioning Flow

Expected high-level flow:

1. Deploy a principal Kubernetes charm.
2. Deploy a Kubernetes-compatible subordinate charm. No units are created.
3. Integrate the applications on a container-scoped relation.
4. When each principal unit enters scope, relation state creates a subordinate
   CAAS unit linked to that principal unit.
5. The principal application's provisioning info changes because its pod now has
   an additional subordinate contribution.
6. The Kubernetes application worker updates the principal StatefulSet pod
   template.
7. New or restarted pods run both the principal and subordinate unit agents.
8. Removing the relation moves the subordinate units through normal lifecycle
   teardown and removes their pod contributions.

## User Experience

The CLI should feel like machine subordinates:

```text
juju deploy primary-k8s primary
juju deploy telemetry-subordinate-k8s telemetry
juju integrate telemetry primary
```

Expected status shape:

```text
App        Version  Status  Scale
primary             active      3
telemetry           active      0

Unit          Workload  Agent  Subordinates
primary/0     active    idle   telemetry/0
  telemetry/0 active    idle
primary/1     active    idle   telemetry/1
  telemetry/1 active    idle
primary/2     active    idle   telemetry/2
  telemetry/2 active    idle
```

Directly scaling the subordinate should remain invalid. Scaling the principal
should create or remove corresponding subordinate units through the relation.

## Compatibility Rules

Likely first-pass constraints:

- only Kubernetes models;
- only charm formats already supported by CAAS deploy;
- subordinate and principal must both be Kubernetes charms;
- only container-scoped subordinate relations;
- no subordinate storage until the pod composition and ownership model is clear;
- no cross-model subordinate colocation;
- no subordinate charm that requires exclusive control of pod-level settings;
- one principal pod template owner: Juju.

## Risks

- **State consistency**: Relation scope, subordinate unit creation, and pod
  contribution records must not split across transactions.
- **Rollout blast radius**: Adding or removing a subordinate can restart every
  principal pod.
- **Agent isolation**: Multiple unit agents in one pod must not share files,
  sockets, hook contexts, or credentials accidentally.
- **Name conflicts**: Two charms may define the same container, mount, or port
  names.
- **Refresh behavior**: Refreshing either charm can require a pod template
  change and coordinated unit-agent upgrade behavior.
- **Removal behavior**: Principal removal must tear down subordinates; subordinate
  removal must update principal pods without orphaning unit state.
- **Status semantics**: The subordinate has no standalone Kubernetes workload,
  but still has unit status, workload status, and relation status.
- **Security boundaries**: A subordinate in the same pod shares network namespace
  and may share process or volume access depending on pod configuration.

## Prototype Plan

1. State prototype:
   - create CAAS subordinate unit rows;
   - record `unit_principal`;
   - expose status without provisioning any pod changes.

2. Provisioning prototype:
   - add one inert subordinate container contribution to the principal pod
     template;
   - verify integration triggers a StatefulSet template update;
   - verify relation removal removes the contribution.

3. Agent prototype:
   - run a subordinate unit agent in the principal pod with isolated config;
   - verify hooks, relation data, status, logs, and hook tools use the
     subordinate identity.

4. Lifecycle prototype:
   - scale the principal up and down;
   - refresh the subordinate charm;
   - remove the relation;
   - remove the principal;
   - migrate/export/import the model if CAAS migration supports the needed
     entities.

## Open Questions

- Is "subordinate" the right user-facing term for Kubernetes, or should this be
  a new "pod-scoped charm" concept with subordinate-like behavior?
- Should subordinate and principal units share a net node, or should pod
  colocation be represented separately from network identity?
- Can a subordinate access the principal workload's Pebble socket? If yes, how is
  authorization scoped?
- What pod-level fields may a subordinate contribute?
- How should conflicting pod contributions be reported to users?
- Does adding a subordinate always restart existing principal pods, or can any
  part of this be introduced without rollout?
- How should storage, devices, and opened ports be represented for subordinate
  units that have no standalone pod?
- What should `juju ssh`, `juju exec`, and actions mean for a subordinate unit in
  a shared pod?
- How do charm refresh, rollback, and charm download state work when two
  applications share one pod template?

# PostgreSQL HA refresh: coordination across hook commits

Status: exploratory design and prototype note, recorded on 7 October 2026 and
extended on 13, 14, and 15 October 2026. This captures the initial alternatives and
native Juju hook deferral with strict redispatch. Option 6 is selected for a
prototype on branch `4.1-defer-prototype`; production design and validation
remain open. The exact PostgreSQL charm revision, Juju version, and deployment
substrate still need to be established for the investigation.

The discussion covers both machine and Kubernetes charms. Local Juju source
inspection used branch `4.1` at commit
`150296e6e1323542d3e50c8c63398c3815cd3565`. The PostgreSQL charm's exact refresh
path has not yet been traced, and no live refresh or failure testing has run.

## Problem statement

An HA PostgreSQL application needs to refresh without downtime. Before the
current PostgreSQL primary undergoes a disruptive refresh, it must hand over
the primary role to another suitable unit.

In the workflow described, the Juju leader needs to write the new primary into
the peer relation's application settings. These writes remain buffered until
the hook exits successfully and the unit agent commits its changes. Other units
cannot act on the new settings before that commit.

This creates an ordering dependency:

1. The refresh hook writes the intended handover into the application databag.
2. Coordinated cluster management depends on other units seeing that write.
3. Waiting inside the hook for that coordination prevents the hook from
   completing and publishing the write.
4. Returning successfully permits publication, but the refresh procedure still
   needs a reliable continuation after the handover completes.

The desired sequence is:

1. Publish the handover request.
2. Allow peers to observe it and perform the required coordination.
3. Confirm that the handover and required service routing have completed.
4. Authorise the disruptive refresh of the former primary.
5. Verify recovery before permitting the next disruptive step.

Publication alone does not establish that peers have processed the request or
that the replacement primary is ready. Hook completion and permission to
disrupt a unit therefore need separate meanings.

## Roles and lifecycle boundaries

The **Juju leader** owns writes to the peer relation's application databag. The
**PostgreSQL primary** owns the database primary role. They may be the same
unit, but the design must treat these as separate roles.

Similarly, replacing charm code and refreshing the workload are distinct
operations. A machine charm may be able to install new charm code while
holding package updates and workload restarts. A Kubernetes refresh can involve
pod replacement, so coordination may have to finish before that replacement
starts. The exact boundary depends on the charm and rollout implementation.

The existing `upgrade-charm` hook runs with the new charm code. It is followed
by `config-changed` in the normal upgrade path. Neither hook should be assumed
to provide preparation before disruption on every substrate. See the
[Juju hook lifecycle](https://canonical.com/juju/docs/juju-cli/latest/reference/hook/).

## Option 1: Split the work across existing hooks

The refresh handler records that refresh is pending, writes the handover
request, and returns successfully. Peers react and acknowledge progress through
their unit databags. Subsequent relation hooks advance the operation until the
former primary can safely refresh.

The existing `config-changed` hook after `upgrade-charm` provides an initial
continuation. It must check readiness and return if coordination is incomplete;
its execution does not prove that peers have processed the request. Later
progress needs an explicit event source, such as peer acknowledgements.

This approach can work without a new Juju API when the charm controls the
disruptive operation. Every handler that might restart or update the workload
must respect the pending handover and refresh state.

For Kubernetes, a separate mechanism must hold pod replacement. Ops
`event.defer()` only arranges for a handler to run again on a later charm
invocation. It does not hold Juju's refresh or the Kubernetes rollout. See
[Ops deferral semantics](https://documentation.ubuntu.com/ops/latest/_modules/ops/framework/).

The main cost is that the charm must implement and recover the coordination
state machine itself. It must also establish that no disruption happens before
its handlers can apply the hold.

## Option 2: Add a preparation hook and an explicit refresh gate

Introduce a preparation phase that runs before disruption, with a durable gate
that remains closed while coordination proceeds. Names such as
`prepare-refresh` and `refresh-ready` below are illustrative proposed APIs;
they are not existing Juju interfaces.

The proposed contract is:

1. Juju records the target refresh and holds affected units before disruption.
2. `prepare-refresh` runs using the installed charm, with information about the
   target revision and resources.
3. The hook publishes preparation or handover intent by returning successfully.
   Its changes are committed, but successful completion does not release the
   refresh gate.
4. Ordinary relation and leadership hooks continue running while refresh is
   held, allowing the charm to coordinate the handover.
5. A proposed `refresh-ready` hook command authorises the next step for the
   specified operation and unit once the charm has confirmed readiness.
6. A completion hook or explicit completion signal lets the charm verify
   recovery before another unit is released.

The charm owns PostgreSQL readiness and upgrade ordering. Juju enforces the
hold. On Kubernetes, that enforcement must take effect before pod replacement,
rather than depend solely on a hook running during termination.

An extra preparation hook alone is insufficient. Immediately continuing
refresh when it exits would race peer processing. Likewise, a hook after
disruption cannot protect the handover that needed to precede it.

This offers the clearest general refresh contract, but requires changes to
Juju's operation state, scheduling, and substrate-specific refresh execution.
Compatibility with installed charms that do not implement preparation also
needs a defined policy.

## Option 3: Add refresh-specific yield and continuation

A proposed `refresh-yield` mechanism would mean:

> Commit this hook's changes, keep refresh pending, and schedule a continuation
> that can advance the operation after coordination makes progress.

The hook exits successfully. Juju persists the continuation and allows
coordination hooks to run while holding the disruptive refresh step. A later
invocation resumes from a recorded phase, rather than from a suspended process
or Python stack.

This could support several coordination rounds without adding a separate hook
for every phase. Its distinction from ordinary Ops event deferral is that the
continuation belongs to Juju's refresh operation: hook success does not release
the refresh hold.

The design still needs durable scheduling, an explicit wake-up mechanism,
idempotent phases, and a definition of completion. It must enter the workflow
before disruption; yielding from an `upgrade-charm` hook that runs after pod
replacement would be too late to protect that replacement.

This is a possible smaller alternative to dedicated preparation and completion
hooks, although its implementation cost has not been assessed.

[Option 6](#option-6-native-juju-hook-deferral-with-strict-redispatch) develops a
general hook mechanism with a different scheduling contract: the deferred hook
runs again next, with a fresh context and no intervening hook on that unit.

## Option 4: Use preparation and continuation actions

An operator or external orchestrator could execute this workflow:

1. Run a preparation action that publishes the handover request and returns.
2. Allow peer hooks to perform the coordination.
3. Check readiness in a subsequent invocation.
4. Initiate `juju refresh` only after readiness is confirmed.
5. Use completion or resume actions to control later stages if needed.

Moving the whole sequence into one action would reproduce the original
dependency if that action waited for its own uncommitted relation writes.
Preparation must finish before dependent coordination is awaited. Juju actions
also commit relation changes after successful execution; see the
[action documentation](https://documentation.ubuntu.com/juju/latest/reference/action/).

There is precedent in the documented PostgreSQL upgrade workflows:
`pre-upgrade-check` runs before refresh, and the Kubernetes workflow also uses
`resume-upgrade`. Their exact behaviour is charm-version dependent and should
not be assumed to implement the protocol proposed here. See the
[machine workflow](https://documentation.ubuntu.com/charmed-postgresql/14/how-to/upgrade/perform-a-minor-upgrade/)
and the
[Kubernetes workflow](https://documentation.ubuntu.com/charmed-postgresql-k8s/14/how-to/upgrade/perform-a-minor-upgrade/index.html).

This is a practical near-term option with little or no new Juju machinery. Its
main weakness is enforcement: a user can bypass preparation by refreshing
directly unless the charm or Juju also guards the disruptive step. Preparation
can also become stale before refresh starts, so readiness must be revalidated
or protected by an appropriate hold.

## Option 5: Allow an explicit commit during a hook

A proposed `relation-flush` command or hook checkpoint could publish buffered
settings while the refresh hook continues running.

This directly addresses visibility when other units can complete the handover
independently and the running hook can observe completion. It changes the
failure model, however:

- Changes committed at the checkpoint survive a later hook failure.
- Retrying the hook must safely recognise work already performed.
- Reads after the checkpoint need defined cache and freshness behaviour.
- Other units can respond, but the running unit cannot execute its next hook
  until the current one exits.
- The scope of the checkpoint must be specified: relation settings alone, or
  other pending hook changes as well.

Consequently, a checkpoint does not resolve a coordination cycle that requires
another hook on the same unit to run. It also needs the same careful placement
before disruption as the other approaches.

This is worth considering if there are compelling uses beyond refresh. It has
broader implications for hook semantics than a refresh-specific continuation
or gate.

## Option 6: Native Juju hook deferral with strict redispatch

This approach was mooted on 13 October 2026. A hook can run hook tools, request
deferral, and exit successfully. Juju commits its buffered changes, retains the
unfinished hook, and dispatches that same hook at the next hook determination.
No unrelated event is needed to trigger another invocation. On 14 October,
this approach was selected for the prototype described below.

The proposal assumes that Juju builds a new hook context for each invocation,
incorporating data changes since the previous invocation. No other hook
interleaves on that unit while the hook remains deferred. Other units and
workload processes continue independently; this is not a cluster-wide execution
lock or a guarantee that peers have acted before the next invocation starts.

### Commit and redispatch contract

The prototype uses `hook-defer` as its new hook tool. The tool records the
deferral request; the hook must still finish successfully.
It would not suspend the process or resume execution at the call site. The
whole hook would run again from its entry point.

| Invocation outcome | Buffered hook writes | Lifecycle and scheduling |
| --- | --- | --- |
| Success without deferral | Commit | Complete the logical hook and advance normally. |
| Success with deferral | Commit | Keep the logical hook pending and dispatch it again with a fresh context. |
| Failure | Normal failure semantics | Apply normal error handling; a deferral request does not turn failure into success. |

Committing writes must remain distinct from completing the lifecycle step.
For example, Juju must not mark the unit stopped, release a storage attachment,
or queue the dependent `config-changed` hook merely because one deferred
invocation succeeded. Those consequences wait for final completion. A failed
commit also cannot be treated as a successfully committed deferral.

This differs from Ops event deferral, which waits for a later charm invocation
and replays the particular handlers that deferred. Native Juju deferral would
guarantee the next hook invocation and rerun the whole dispatch process.
Framework integration would need to account for that difference.

### Three-unit PostgreSQL flows

These are proposed flows under the strict redispatch contract, not a trace of
the current charm. Assume one disruptive workload upgrade at a time, with a
healthy successor kept available. Here, the address written by the Juju leader
is the PostgreSQL primary address; PostgreSQL handover does not transfer Juju
leadership.

Following the original scenario, the address publication precedes the cluster
management work that depends on it. Publication is not evidence of a completed
promotion. The exact meaning of the charm's address field still needs to be
verified: a field reporting the observed primary must not be used to advertise
an unpromoted successor as already serving.

| Scenario | Unit A | Unit B | Unit C |
| --- | --- | --- | --- |
| Juju leader is primary | Juju leader and current primary | Selected successor | Replica |
| Juju leader is not primary | Juju leader and replica | Current primary | Selected successor |

When the Juju leader is the primary:

1. A's refresh invocation selects B, writes its address into the application
   settings, requests deferral, and exits successfully without upgrading.
2. Juju commits A's write and immediately redispatches A's refresh hook with a
   new context.
3. A continues the handover workflow. B and C can independently act on the
   committed settings through their refresh invocations or workload processes.
4. If handover has not completed, A defers again. Subsequent invocations observe
   peer progress without requiring a local `relation-changed` hook to run.
5. Once handover and the required service readiness are confirmed, A performs
   the disruptive upgrade and finishes without deferring.

When the Juju leader is not the primary:

1. B's refresh invocation requests successor C in B's unit databag, requests
   deferral, and exits successfully. B cannot write application settings.
2. Juju commits B's request and redispatches B. B continues to defer while the
   required publication or handover remains incomplete.
3. A's next refresh invocation observes B's request in its fresh context. As
   Juju leader, A writes C's address into the application settings and defers.
4. Juju commits A's application write. A can continue waiting for its own
   upgrade turn while remaining able to service coordination requests.
5. B's subsequent invocation observes A's committed publication and advances
   the handover. It defers again until the required readiness is confirmed.
6. B upgrades and finishes without deferring.

The second flow assumes A is also participating in the refresh workflow.
Deferring B only guarantees redispatch on B; it does not invoke A. If A has no
pending refresh hook, the workflow needs another path, such as an ordinary peer
relation hook, for A to service B's committed request.

### Progress condition and caveats

Strict redispatch can support both flows without local hook interleaving,
provided the refresh handler performs the necessary coordination. The progress
condition is:

> Everything needed to complete a deferred hook must be achievable by later
> invocations of that hook, by other units, or by independently running workload
> processes.

Fresh data does not execute behaviour attached to another hook. In particular,
A must service leader coordination before checking whether it is A's turn to
upgrade. If A repeatedly defers waiting for B to finish, while only A's queued
`relation-changed` handler can publish C's address, A and B will wait forever.
The same restriction applies to coordination needed on the successor unit.

The earlier concern about interleaving is therefore conditional: it is not
required for these flows if each repeated handler can perform all its required
work. It remains necessary for a workflow whose progress depends on a different
local hook, unless that workflow is adapted.

Further caveats are:

- Fresh contexts must provide refreshed observations, including appropriate
  treatment of agent-side caches. Recreating the charm process or context
  object alone does not establish that property.
- Immediate redispatch guarantees another invocation, not peer readiness.
  Repeated deferral is polling; retry pacing and controller load need analysis.
  Pacing can preserve the rule that the same hook runs next.
- The continuation and hook identity must survive agent restart, including a
  crash between committing writes and redispatching. Earlier committed writes
  remain committed if a later invocation fails. Repeated work must be safe and
  progress must be recoverable from durable state.
- A fresh leadership observation does not run `leader-elected` or
  `leader-deposed`. Leadership changes and any required initialisation need an
  explicit policy under strict redispatch, as do cancellation and unit removal.
- Holding the logical hook can hold later lifecycle steps, but cannot prevent
  an independently controlled workload disruption. The hook must still run
  before the disruptive refresh boundary, especially for Kubernetes pods.

## Provisional direction

The initial preference discussed was option 2 for a general Juju solution,
with option 4 as a practical near-term workflow. The 13 October discussion
adds option 6 as a general mechanism that could provide both committed progress
and guaranteed continuation without interleaving hooks. Its viability depends
on the progress condition above and the actual refresh boundary. On 14 October,
option 6 was selected for prototyping, while production decisions remain open.

Any selected approach needs to distinguish intent, observed progress, and
permission to disrupt. In particular:

- Readiness must refer to a specific refresh operation, target revision and
  resources, and unit, so stale acknowledgements cannot release a newer refresh.
- Coordination must survive agent restarts and accommodate Juju leadership
  changes and PostgreSQL primary changes.
- Required coordination must still progress while refresh is held, either
  through allowed hooks or through repeated reconciliation in the deferred
  hook under option 6.
- Retried phases must be idempotent and derive progress from durable state.
- A timeout should leave refresh held with an explanation rather than silently
  authorise disruption.
- Completion must include the required workload recovery and service routing,
  rather than only the publication of a primary's name.

## Planned prototype implementation: 14 October 2026

The authorised prototype implements the Juju side on `4.1-defer-prototype`.
No new tests are to be written at this stage. The Ops follow-up was implemented
on 15 October in `../operator`, on branch `juju-defer-prototype`. The Python API
names below remain experimental.

### Juju command and execution

`hook-defer` accepts no arguments. It uses the existing hook-tool connection to
call `RequestHookDeferral()` on the running `HookContext`. Multiple calls set
the same invocation-local flag. The command returns normally and does not
flush writes or terminate the charm process.

On successful exit, the normal hook flush commits buffered writes. The runner
then returns a distinct deferral result. The uniter persists the same hook as
queued with a reissued marker and skips lifecycle completion and its associated
bookkeeping. The resolver chooses that hook before other hooks and actions.
Agent shutdown, unit death, and resolver cancellation retain precedence.

The next invocation gets a new context and `JUJU_HOOK_REISSUED=1`. Cached
settings for known relations are discarded when building this context so that
peer writes can be observed without interleaved relation hooks. The deferral
request itself starts unset; the charm must request it again if necessary.

Actions, `juju exec`, and restricted contexts cannot request native deferral.
The prototype also rejects combining deferral and reboot requests in either
order. Hook and commit errors retain ordinary failure behaviour. Calls that
fail do not authorise a successful deferral.

### Ops prototype surface

| Surface | Intended behaviour |
| --- | --- |
| `ops.hookcmds.hook_defer()` | Low-level wrapper around `hook-defer`. |
| `self.framework.defer_hook()` | Request redispatch of the current Juju hook. |
| `self.framework.hook_deferred` | Expose this invocation's request to cooperating observers. |
| `event.defer()` | Preserve existing Ops observer deferral semantics. |

For initial charm experiments, call `subprocess.run(["hook-defer"], check=True)`
and return normally from the handler. Do not also call `event.defer()` for the
same event. Do not use `sys.exit(0)` to end the handler: Ops still needs to
complete its normal framework state commit. Other observers may still run and
must respect the pending handover.

The framework API does not add an Ops deferred-event notice for native deferral.
It rejects calls from actions, restricted hooks, outside the current hook
handler, or historical Ops events being replayed. Synchronous custom events
emitted by the current hook may request redispatch of that enclosing hook.
Unsupported Juju agents produce an error rather than silently falling back to
ordinary `event.defer()`.

Normal Ops dispatch replays saved deferred events before emitting the current
Juju event. The Ops prototype recognises `JUJU_HOOK_REISSUED=1`, preserves that
queue without replaying it, and emits the reissued hook with normal status
evaluation and framework commit. Saved events become eligible at the next normal
dispatch. Duplicate-event suppression is disabled during native redispatch, so
saved notices cannot suppress the fresh hook or its custom events. Mixing native
and Ops deferral while handling the current hook is rejected in either order,
including across observers. The low-level wrapper bypasses these framework
guards; Ops charms should use `self.framework.defer_hook()`.

See the [Ops prototype usage note](../operator/NATIVE-HOOK-DEFERRAL.md) for an
example. `ops.JujuContext.hook_reissued` also exposes the marker to alternative
dispatchers. The prototype does not add Scenario or Harness simulation.

These observations were checked against Ops revision
`395933e5902b620547638f5987273f0cb229386f`: see its
[dispatch sequence](https://github.com/canonical/operator/blob/395933e5902b620547638f5987273f0cb229386f/ops/_main.py#L431)
and
[event notice handling](https://github.com/canonical/operator/blob/395933e5902b620547638f5987273f0cb229386f/ops/framework.py#L825).

### Prototype limits

- Redispatch is immediate; retry pacing and a deferral limit are not included.
- The queued hook and its reissued marker are persisted through existing uniter
  state. This does not introduce an atomic transaction spanning hook writes
  and the subsequent uniter-state write. A crash in that interval can still
  require ordinary hook error recovery.
- Relation settings are refreshed for existing context relations; this is not
  a new guarantee that relation membership or lifecycle changes have completed.
- Leadership changes, teardown hooks, forced upgrades, migration, and cancellation
  require broader design and validation before production use.
- Ops integration requires the corresponding prototype checkout, and no live
  PostgreSQL refresh has been run. Fresh context and strict Juju hook ordering
  alone do not suppress deferred callbacks in older Ops versions or establish
  database availability.

### Prototype verification

The agent (`cmd/jujuagentd`) and hook-tool client (`cmd/jujuc`) build. Existing
tests pass with `-race` in the hook, operation, resolver, runner, runner/context,
and runner/jujuc packages, along with the selected top-level uniter resolver
suites. A bounded stress run of the resolver race-test binary reported 56
completed iterations and no failures before the 31-second time limit.

No tests were added. These existing checks do not validate the new workflow
end to end. `STATIC_ANALYSIS=1 make pre-check` passed its preceding checks but
stopped at the Go lint step because `golangci-lint` is not installed; licence
and shell-format checks were also skipped for missing tools.

The Ops follow-up passes `tox -e format`, `tox -e lint`, and the existing
`tox -e unit -- -n 4` suite: 2,445 tests plus 13 release-script tests passed,
with 56 skipped. No tests were added, and neither prototype has been exercised
in a live Juju/PostgreSQL refresh.

## Questions for further investigation

1. Which PostgreSQL charm revision, Juju version, and substrate exhibit the
   problem? Which hook or charm-library callback currently performs the work?
2. What is the first disruptive operation on each substrate, and which
   component controls it? Can that component enforce a hold before disruption?
3. Does the application databag value represent an intended successor, an
   observed primary, or a routing instruction? Which consumers act on it?
4. Which units must acknowledge the handover, and what proves the successor is
   ready? Does any required step need a further hook on the writing unit?
5. Can existing charm refresh machinery provide the necessary hold and
   continuation, or is a Juju lifecycle change required?
6. Should preparation run on every unit or primarily on the Juju leader? How
   should application coordination authorise individual unit refreshes?
7. How should gates be released, invalidated, or recovered across leadership
   changes, unit loss, cancelled refreshes, changed targets, and rollback?
8. How would installed charms adopt a preparation protocol, and what should
   happen when the installed revision does not support it?
9. What does "without downtime" require for this application: continued service
   availability, preserved client connections, or uninterrupted transactions?
   Which cluster and client behaviours must be validated alongside Juju ordering?
10. Under strict redispatch, can every participating refresh handler service its
    coordination obligations without another local hook running? What happens
    when the Juju leader has already completed its refresh hook?
11. What context and cache refresh guarantees are needed between invocations?
    How should Juju persist deferral, pace retries, and distinguish a committed
    invocation from final hook completion?
12. Which hook kinds should support native deferral, and how should it interact
    with Ops handler deferral, leadership changes, cancellation, and removal?

## Local source evidence

These references identify the Juju behaviour inspected at the revision above;
they do not establish the PostgreSQL charm's full refresh workflow.

- [`relation-set`](internal/worker/uniter/runner/jujuc/relation-set.go): checks
  leadership for application settings and updates the hook's settings object.
  Its help text describes publication on successful hook completion.
- [`HookContext.Flush` and `doFlush`](internal/worker/uniter/runner/context/context.go):
  flush pending changes on successful hook execution and include application
  relation settings in the commit request.
- [`deploy.Execute` and `deploy.Commit`](internal/worker/uniter/operation/deploy.go):
  deploy charm contents and queue the upgrade hook in the normal upgrade path.
- [`runHook.Commit`](internal/worker/uniter/operation/runhook.go): queues
  `config-changed` following `upgrade-charm`.
- [Hook kinds](domain/deployment/charm/hooks/hooks.go): documents existing hook
  ordering and upgrade exceptions.

# Agentless Charm / Starform Primer

This is a working-session primer for a future clean session. It is not a
user-facing design document and does not imply an implementation has started.

## Goal

Explore a Juju charm execution mode where charm logic is not run by a unit
agent on a machine or container. Instead, a controller-side worker watches model
events and evaluates charm-provided Starform scriptlets.

The key design constraint from the prior discussion: do not model this as a
`jujuc` replacement. Starform is better suited to collecting declared intent
from scriptlets, then letting Juju validate and apply that intent after script
execution succeeds.

## Repository Context

Juju repo:

- `/home/joseph/projects/canonical/juju`

Starform clone:

- `/home/joseph/projects/canonical/starform`

Before making Juju changes, read and follow:

- `AGENTS.md`
- `AGENTS.architecture-rules.md`
- `AGENTS.core-domain-rules.md`

Important Juju layering constraints:

- Put workers under `internal/worker`.
- Keep business workflows in `domain` services.
- Keep persistence behind domain state interfaces.
- Do not put business logic in API facades.
- Workers must be restartable, deterministic, cancellation-aware, and use
  existing worker lifecycle patterns such as `catacomb` or `tomb`.

## Starform Findings

Files to read first:

- `/home/joseph/projects/canonical/starform/README.md`
- `/home/joseph/projects/canonical/starform/starform/scriptset.go`
- `/home/joseph/projects/canonical/starform/starform/appobject.go`
- `/home/joseph/projects/canonical/starform/starform/eventobject.go`
- `/home/joseph/projects/canonical/starform/formtest/formtest_test.go`
- `/home/joseph/projects/canonical/starform/SECURITY.md`

Core model:

- Starform embeds Canonical Starlark and exposes event-based scriptlets.
- Scriptlets register observers during `init` using `app.observe(event, fn)`.
- `ScriptSet.LoadSources` compiles and initializes scripts.
- `ScriptSet.Handle(ctx, event)` invokes registered observers for an event.
- Observer return values are ignored.
- Scriptlet output is host-owned state mutation through host-provided app
  methods, typically appending typed intents into `EventObject.State`.
- The README explicitly frames output as desired target state or action, with no
  host changes applied until after script execution completes.

Important API details:

- `ScriptSetOptions` includes `App`, `Cache`, `Logger`, `RequiredSafety`,
  `MaxAllocs`, `MaxSteps`, and `Modules`.
- `EventObject` has `Name`, `State`, and `Attrs`.
- `Attrs` are exposed as Starlark attributes and should be treated as event
  input.
- `State` is developer-supplied Go state and is the natural place for an intent
  collector.
- Custom app methods are only available while handling an event, not while
  loading modules or during `init`.
- `observe` is only available during `init`.
- Starform event names must be lowercase identifiers, 3-20 chars, with no
  dashes or consecutive underscores.
- Raw Juju hook names like `config-changed` do not fit directly. Either use
  snake-case event names such as `config_changed`, or use a generic event such
  as `hook` with `event.kind = "config-changed"`.

Safety implications:

- Use Canonical Starlark safety flags: `MemSafe`, `CPUSafe`, `TimeSafe`,
  `IOSafe`.
- If `MemSafe` is required, `MaxAllocs` must be set.
- If `CPUSafe` is required, `MaxSteps` must be set.
- Every exposed builtin must be tested for the required safety properties.
- Bound script set size and per-file size.
- Do not expose secrets or sensitive values as plain printable Starlark values.
- Logs are a disclosure surface because scripts can call `print` and `debug`.

Dependency note:

- Starform module is `github.com/canonical/starform`.
- It currently depends on `github.com/canonical/starlark`.
- Juju currently has `go.starlark.net` only as an indirect dependency.
- Adding Starform to Juju will require an explicit dependency decision.

## Juju Findings

Files to read first:

- `internal/worker/uniter/doc.go`
- `internal/worker/uniter/uniter.go`
- `internal/worker/uniter/remotestate/watcher.go`
- `internal/worker/uniter/resolver/loop.go`
- `internal/worker/uniter/operation/runhook.go`
- `internal/worker/uniter/runner/runner.go`
- `domain/unitstate/types.go`
- `domain/unitstate/service/commithook.go`
- `domain/unitstate/state/commithook.go`

Relevant existing model:

- The uniter worker watches unit/application/relation state and dispatches hook
  operations to charm code.
- The uniter structure is useful conceptually: watcher snapshot, resolver,
  operation factory, executor, retry/error state.
- The runner and `jujuc` machinery are not a good fit for agentless Starform
  charms because they assume an executing charm process and hook-tool context.
- `domain/unitstate.CommitHookChangesArg` is a useful precedent for
  post-hook transactional output.
- `CommitHookChanges` validates and applies hook changes only after successful
  hook completion.

Important mismatch:

- `CommitHookChangesArg` is unit-scoped.
- Agentless charm logic may need application-scoped state and relation writes,
  or a deliberately defined synthetic/controller unit identity.
- This identity question affects status, relation data ownership, leadership,
  secrets, hook ordering, and lifecycle.

## Design Direction

Prefer this shape:

1. A controller-side worker watches relevant model events.
2. The worker loads a Starform `ScriptSet` for a charm revision.
3. For each event, the worker builds a read-only event snapshot as
   `EventObject.Attrs`.
4. The worker creates an event-specific intent collector as `EventObject.State`.
5. Starform handlers run with strict safety limits.
6. If execution fails, discard the collector and record/report the error.
7. If execution succeeds, validate collected intents.
8. Apply valid intents through domain services, ideally transactionally.

Avoid this shape:

- Exposing live domain services directly to Starlark.
- Exposing RPC-like `juju.*` calls that mutate state during script execution.
- Recreating `jujuc` as in-process methods.
- Letting scriptlets perform I/O or controller-side side effects directly.

Possible intent API examples, not decisions:

- `juju.status_set(status, message="")`
- `juju.state_set(key, value)`
- `juju.state_delete(key)`
- `juju.relation_set(endpoint, data, scope="application")`
- `juju.open_port(endpoint, protocol, port)`
- `juju.close_port(endpoint, protocol, port)`

The names above are illustrative. The important property is that methods append
typed intents to a collector. Juju validates and applies them after execution.

## Open Questions

- What is the entity identity for an agentless charm: application, synthetic
  unit, controller unit, or a new concept?
- How many workers run for one agentless application in HA controller setups,
  and what lease prevents duplicate execution?
- Which MVP events should exist first: config, update-status, relation events,
  lifecycle events, or a single generic hook event?
- How should event ordering and retries be persisted across controller restarts?
- Where are Starform script sources stored in a charm package, and how is a
  charm marked as agentless/scriptlet-based?
- Should relation data be application-only, or should there be a unit-scoped
  concept for compatibility?
- How are status and charm state represented without a unit agent?
- Which secret operations, if any, are safe for an MVP?
- How are script errors surfaced: application status, model status, controller
  log, or a new error surface?
- How are charm upgrades handled, including reloading scripts and preserving
  state?

## Suggested Next Session

Start by re-reading the Starform files listed above, then the Juju uniter and
unitstate files listed above. Do not begin by wiring dependencies into Juju.

For a minimal spike, define a small host app object and collector in isolation:

- One event, probably `config_changed` or generic `hook`.
- One or two event attributes.
- One or two intent methods, probably status and charm state.
- A test showing that failed script execution discards intents.
- A test showing successful execution validates and converts intents into a
  domain-layer argument shape.

Only after that should the worker shape be designed.

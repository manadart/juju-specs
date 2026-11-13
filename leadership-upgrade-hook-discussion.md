# Leadership Yield Around Upgrade Hooks

## Problem

Long-running `upgrade-charm` hooks can keep the current application leader busy.
If that leader continues holding leadership while it runs the upgrade hook, no
other unit can become leader and process leader-only work until the hook
completes or the lease is otherwise lost.

The explored idea is:

1. When an application has more than one unit.
2. If the unit about to run `upgrade-charm` is currently leader.
3. Yield leadership immediately before dispatching the upgrade hook.
4. Let another unit become leader while the old leader runs upgrade handling.

The desired property is stronger than simply releasing the lease: the new leader
should be the first unit that can actually action `leader-elected`, not merely
the first unit whose leadership tracker wins a lease race.

## Current Uniter Leadership Behavior

Leadership changes are represented in two places:

- Remote state: `remotestate.Snapshot.Leader` indicates whether the unit is
  currently elected leader.
- Local operation state: `operation.State.Leader` indicates whether
  `leader-elected` has been queued/run and no later leadership loss has been
  recorded locally.

`acceptLeadership` queues a `leader-elected` hook and sets
`operation.State.Leader = true`, but only when the local operation state is
`Continue`. This avoids overwriting an existing queued or pending operation.

`resignLeadership` currently does not run `leader-deposed`. Its `Execute` only
logs that the hook should run but cannot yet. Its `Commit` clears
`operation.State.Leader` and preserves the rest of the operation state.

That means current resign behavior is a local bookkeeping operation after
leadership has already been lost. It is not a lease-yield operation.

## Old `leader-deposed` Comment

The large comment in `internal/worker/uniter/operation/leader.go` is old, but
still explains part of the current constraint:

- A normal `RunHook{LeaderDeposed}` would overwrite the current single-slot
  uniter operation state.
- The desired state write is only `Leader=false`; it must not disturb queued or
  pending hooks.
- A special hook context would be needed to run `leader-deposed` correctly,
  especially in no-API situations.

The repository now has `jujuc.RestrictedContext`, but no current path wires a
special restricted context into `leader-deposed`. The uniter also still requires
an API connection at startup.

## Current Leadership Tracker Behavior

The leadership tracker has different behavior depending on whether it believes
the unit is leader or minion.

When the unit is leader:

- It has a held lease.
- It has a renewal timer.
- It does not have an active `BlockUntilLeadershipReleased` waiter.

When the unit is minion:

- It starts `BlockUntilLeadershipReleased`.
- When that wait returns, it immediately calls `ClaimLeadership`.

So a leader does not have an active background claimer waiting to race after its
own lease is released. However, if leadership is revoked externally, the local
tracker does not immediately know that. Until its renewal path observes the
loss, `ClaimLeader()` may still answer from local state and report leadership.

This means a yield mechanism needs tracker participation so the old leader
immediately stops reporting leadership, not because the old leader already has a
live claim waiter.

## Why Flipping Local State Is Not Enough

Clearing `operation.State.Leader` before `upgrade-charm` would not release the
actual leadership lease.

Hook tools use the leadership tracker. For example, `is-leader` goes through the
hook context and tracker, not through `operation.State.Leader`. Therefore a unit
could have local operation state saying "not leader" while the hook context
still reports that leadership is guaranteed.

Any solution must revoke or yield the real leadership lease and update local
tracker state.

## Existing Revocation Support

`core/leadership.Revoker` exists and can revoke application leadership for a
unit. There are implementations around the lease manager and domain services.

The current uniter worker receives a `leadership.Tracker`, not a revoker. The
current `UniterAPI` facade also does not expose a leadership revoke method to
the unit agent. Adding a real yield operation would therefore need API and/or
worker dependency plumbing.

## Upgrade Hook Insertion Point

The clean local insertion point is after the charm upgrade operation commits and
queues `upgrade-charm`, but before the queued hook is dispatched.

At that point local state looks like:

```text
Kind: RunHook
Step: Queued
Hook.Kind: upgrade-charm
Leader: true
```

A new operation could:

1. Detect this state.
2. Yield actual leadership.
3. Commit only `Leader=false`, preserving the queued `upgrade-charm`.
4. Let the resolver loop again and dispatch the queued hook as a non-leader.

This must not be implemented by replacing the queued hook with another
`RunHook`, because uniter operation state has only one hook slot.

## Claiming By First Unit Able To Run `leader-elected`

The current system cannot guarantee that the first unit able to action
`leader-elected` is the unit that claims leadership.

Today, after the lease is released, minion trackers race in the background. The
winner is the first tracker to successfully call `ClaimLeadership`, regardless
of whether that unit's uniter can immediately queue or run `leader-elected`.

To get the desired property, claiming must be gated by the uniter state machine:

1. Make minion trackers passive for acquisition, or add a gated acquisition
   mode.
2. Let the uniter create a serialized `claimLeadership` operation only when it
   can immediately accept leadership.
3. That operation should claim the lease, set local `Leader=true`, and queue
   `leader-elected` in one resolver-controlled path.
4. Units with queued or pending `upgrade-charm` should not claim leadership,
   because they cannot immediately action `leader-elected`.

This would make the lease remain vacant until a unit reaches a state where it
can actually queue `leader-elected`.

Strict global ordering, such as "the first unit that became able must win",
cannot be guaranteed by agent-local timing alone. That would require
controller-side candidacy or ordering.

## Risks

Charm compatibility is the largest semantic risk. Some charms may assume that
the current leader can run leader-only upgrade logic in `upgrade-charm`. If Juju
forces leaders to yield before `upgrade-charm`, that logic may not run where
the charm expects.

The change should likely be opt-in, model-config gated, charm-metadata gated, or
otherwise narrowly scoped.

There is also a sequencing risk for replacement leaders. A unit may acquire the
lease while it already has queued work. If leadership claiming remains tracker
driven, that unit can become actual lease leader before its `leader-elected`
hook runs. The proposed gated-claim design is intended to avoid that.

## Summary

The problem is feasible but not as a small change to `resignLeadership`.

A correct design needs:

- A real leadership-yield primitive, not just local state mutation.
- Tracker support so a yielding unit immediately stops reporting leadership.
- Resolver/operation integration that preserves queued `upgrade-charm`.
- Claim gating so only a unit able to immediately queue `leader-elected` can
  claim leadership.
- A compatibility story for charms that expect leader upgrade behavior.

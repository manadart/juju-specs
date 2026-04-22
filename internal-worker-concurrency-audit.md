# Internal Worker Concurrency Audit

Started: 2026-03-25

## Scope

- This is a first-pass concurrency audit of the top-level directories under
  `internal/worker`.
- Nested packages are folded into the parent entry when they materially affect
  worker concurrency, for example `common/charmrunner` and `uniter/*`.
- Status labels used below:
  - `finding`: actionable issue found.
  - `watchlist`: custom concurrency exists; no concrete bug confirmed in this
    pass.
  - `clear`: no immediate concurrency issue spotted in this pass.

This document is intended to be resumed in later sessions. It is not a claim
that every nested helper package has had a line-by-line deep review yet.

## Confirmed Findings

### `filenotifywatcher`

- Finding: `Watcher.loop` sends on `w.changes` outside a `select` that also
  watches `w.catacomb.Dying`.
- Risk: if the consumer stops draining `Changes()` and an event arrives, a
  later `Kill` can hang forever because the loop is blocked on
  `w.changes <- ...`.
- Ref: `internal/worker/filenotifywatcher/watcher.go:165-199`
- Action: make notifications buffered or coalesced, or send with a `select`
  that also watches `w.catacomb.Dying`.
- Test idea: inject an event, do not receive from `Changes()`, call `Kill`,
  and assert `Wait()` returns.

### `logsender`

- Finding: `BufferedLogWriter.Close` blindly closes `w.in`, while
  `BufferedLogWriter.Write` always sends to `w.in`.
- Risk: concurrent shutdown and logging can panic with `send on closed
  channel`.
- Ref: `internal/worker/logsender/bufferedlogwriter.go:150-158`,
  `internal/worker/logsender/bufferedlogwriter.go:179-183`
- Finding: the sender worker reads `rec := <-logs` without checking `ok`.
- Risk: once the buffered writer closes `Logs()`, the worker can receive a
  nil `*LogRecord` and dereference it.
- Ref: `internal/worker/logsender/worker.go:65-79`
- Action: add explicit closed-state handling on the writer side, and handle
  `ok == false` in the worker loop.
- Test idea: close the buffered writer before the logsender worker exits and
  assert there is no panic.

### `machineactions`

- Finding: `TearDown` waits on `h.wait.Wait()` without any bound, and the file
  already carries a TODO calling out the problem.
- Risk: any hung action goroutine can wedge worker shutdown indefinitely.
- Ref: `internal/worker/machineactions/worker.go:145-158`
- Action: bound teardown time, cancel or mark unfinished actions failed, and
  add a regression test for a stuck action.

### `httpserver`

- Finding: `shutdown()` starts a goroutine that waits on `w.config.Mux.Wait()`
  and then gives up after `MuxShutdownWait` without joining or cancelling that
  goroutine.
- Risk: if mux shutdown stalls, the worker exits but leaves a goroutine behind.
- Ref: `internal/worker/httpserver/worker.go:223-238`
- Action: either make the mux wait cancellable, or keep the worker alive until
  the goroutine is known to be done.
- Test idea: block `Mux.Wait()`, trigger shutdown, and assert no goroutine is
  left behind once the worker exits.

### `common`

- Finding: `common/charmrunner.HookLogger.Stop` reads `l.stopped` without
  synchronization, while `Run` and `Stop` mutate or read the same field under
  `l.mu`.
- Risk: concurrent `Stop` calls race on `stopped`; the method claims
  idempotence but does not make that property thread-safe.
- Ref: `internal/worker/common/charmrunner/logger.go:55-63`,
  `internal/worker/common/charmrunner/logger.go:80-98`
- Action: guard the idempotence check with the mutex, or replace it with
  `sync.Once` or `atomic.Bool`.
- Test idea: a race test that calls `Stop` concurrently while `Run` is active.

### `lease`

- Finding: `waitForGoroutines` spawns a goroutine that waits on `manager.wg`,
  but after `maxShutdownWait` it logs and returns without joining that helper
  goroutine or the outstanding claim/revoke goroutines.
- Risk: bounded shutdown leak if store calls ignore cancellation or take too
  long; the manager exits while background work may still exist.
- Ref: `internal/worker/lease/manager.go:724-746`
- Action: make outstanding operations observable and cancellable enough that
  shutdown can join deterministically, or at least close over the helper
  goroutine too.
- Test idea: force a claim or revoke path to ignore cancellation and assert the
  worker reports the condition in a controlled way.

## Watchlist

These packages have non-trivial concurrency surfaces and deserve a deeper pass
before code changes in the area. I did not confirm a concrete bug in this
first pass.

- `apiremotecaller`: channel-based fanout, reconnect loops, and nested worker
  ownership.
- `bootstrap`: broad startup orchestration; revisit if touching worker bring-up
  or teardown paths.
- `caasapplicationprovisioner`: nested workers and resource handling.
- `caasfirewaller`: broad watcher and environment coordination.
- `caasmodelconfigmanager`: config watcher plus provider updates.
- `caasmodeloperator`: operator lifecycle and background coordination.
- `caasprober`: atomics and probe sequencing.
- `caasrbacmapper`: informer goroutines and work queue shutdown semantics.
- `changestream`: nested runners and watchable DB lifecycle.
- `computeprovisioner`: long-lived provisioner state and locking.
- `containerprovisioner`: nested workers plus mutable shared state.
- `controllerpresence`: remote connection tracking and child worker churn.
- `dbaccessor`: many worker-owned goroutines and shutdown edge cases.
- `dbrepl`: REPL helper goroutines tied to terminal and tomb state.
- `dbreplaccessor`: similar lifecycle concerns to `dbaccessor`.
- `deployer`: nested worker orchestration for units.
- `externalcontrollerupdater`: `connectAndWatch` uses an unmanaged helper
  goroutine around remote dial/watch work.
- `firewaller`: very large state machine with multiple watcher classes.
- `fortress`: helper goroutines per guard and guest ticket; revisit aborted
  lockdown behavior.
- `httpserverargs`: `NewStateAuthenticator` starts `Maintain(ctx)` on a raw
  goroutine and relies on caller context for cleanup.
- `instancepoller`: broad watcher and provider coordination.
- `leadership`: custom ticket and watcher sequencing.
- `migrationmaster`: large shutdown surface and long-running operations.
- `migrationminion`: similar migration lifecycle concerns.
- `modelworkermanager`: nested model worker fanout.
- `muxhttpserver`: serve loop and shutdown interaction should be re-checked.
- `objectstore`: multiple internal workers and session management.
- `objectstoredrainer`: nested runner plus drainer lifecycle.
- `objectstores3caller`: callback under mutex can starve session rotation and
  is worth a deadlock-focused review.
- `providertracker`: request channels and provider factory caching.
- `remoterelationconsumer`: multiple nested worker trees and remote event
  streams.
- `sshserver`: connection-count goroutines, listener synchronization, and
  per-connection server handling.
- `storageprovisioner`: large event-driven worker with model and machine modes.
- `storageregistry`: request serialization worker with tracked providers.
- `trace`: nested runners, tomb/catacomb mix, and client lifecycle.
- `uniter`: the biggest concurrency surface in this tree; it should get a
  dedicated document or follow-up section rather than ad hoc notes.
- `watcherregistry`: registry mutations, namespace generation, and watcher
  ownership deserve a closer pass.

## Execution Plan

Ordered next steps for follow-up work:

1. Fix the confirmed findings first and land regression tests with them.
2. After that, audit the watchlist in priority order rather than
   alphabetically.
3. Review the shutdown-sensitive watchlist packages first:
   `externalcontrollerupdater`, `httpserverargs`, `muxhttpserver`,
   `sshserver`, `objectstores3caller`, `fortress`, `controllerpresence`,
   `apiremotecaller`.
4. Split the largest worker subsystems into dedicated follow-up passes:
   `uniter`, `firewaller`, `storageprovisioner`, `remoterelationconsumer`,
   `dbaccessor`.
5. For each deeper pass, record:
   confirmed issue, suspected pattern, shutdown path reviewed, test gap, and
   proposed fix.

## Implementation Status

The following first-pass findings now have concrete fixes with targeted
regression coverage, either in the current working tree or in companion patch
branches from the same audit:

- `common/charmrunner`: `HookLogger.Stop` now uses `sync.Once` for
  thread-safe idempotence.
- `filenotifywatcher`: change delivery now aborts cleanly when the worker is
  dying instead of blocking shutdown on `Changes()`.
- `logsender`: `BufferedLogWriter` close/write coordination no longer risks a
  `send on closed channel`, and the worker now handles a closed logs channel.
- `machineactions`: teardown no longer waits forever on a hung action; it logs
  and continues shutdown after a bounded wait.
- `httpserver`: shutdown now waits on a mux-owned completion channel instead
  of spawning an unjoinable goroutine around `Mux.Wait()`.
- `lease`: shutdown now tracks outstanding claim/revoke work without spawning
  an extra helper goroutine that can outlive timeout-based teardown.

Residual caveats after these fixes:

- `machineactions` still cannot forcibly stop a stuck `HandleAction`; it now
  bounds shutdown, but the action goroutine may continue until the underlying
  handler returns.
- `lease` still depends on downstream store operations honouring cancellation
  for fully prompt shutdown; the leak surface is reduced to the actual
  outstanding request goroutines.

## Package Status

### `finding`

- `common`
- `filenotifywatcher`
- `httpserver`
- `lease`
- `logsender`
- `machineactions`

### `watchlist`

- `apiremotecaller`
- `bootstrap`
- `caasapplicationprovisioner`
- `caasfirewaller`
- `caasmodelconfigmanager`
- `caasmodeloperator`
- `caasprober`
- `caasrbacmapper`
- `changestream`
- `computeprovisioner`
- `containerprovisioner`
- `controllerpresence`
- `dbaccessor`
- `dbrepl`
- `dbreplaccessor`
- `deployer`
- `externalcontrollerupdater`
- `firewaller`
- `fortress`
- `httpserverargs`
- `instancepoller`
- `leadership`
- `migrationmaster`
- `migrationminion`
- `modelworkermanager`
- `muxhttpserver`
- `objectstore`
- `objectstoredrainer`
- `objectstores3caller`
- `providertracker`
- `remoterelationconsumer`
- `sshserver`
- `storageprovisioner`
- `storageregistry`
- `trace`
- `uniter`
- `watcherregistry`

### `clear`

- `agent`
- `agentbinaryfetcher`
- `agentconfigupdater`
- `apiaddresssetter`
- `apiaddressupdater`
- `apicaller`
- `apiconfigwatcher`
- `apiremoterelationcaller`
- `apiserver`
- `apiservercertwatcher`
- `asynccharmdownloader`
- `auditconfigupdater`
- `authenticationworker`
- `caasadmission`
- `caasbroker`
- `caasprobebinder`
- `caasunitterminationworker`
- `caasupgrader`
- `certupdater`
- `changestreampruner`
- `charmrevisioner`
- `containerbroker`
- `controlleragentconfig`
- `controlsocket`
- `credentialvalidator`
- `diskmanager`
- `domainservices`
- `flightrecorder`
- `gate`
- `hostkeyreporter`
- `httpclient`
- `identityfilewriter`
- `introspection`
- `jwtparser`
- `leaseexpiry`
- `lifeflag`
- `logger`
- `logsink`
- `machineconverter`
- `machiner`
- `migrationflag`
- `mocks`
- `modellife`
- `objectstorefacade`
- `objectstoreservices`
- `operationpruner`
- `providerservices`
- `proxyupdater`
- `querylogger`
- `reboot`
- `removal`
- `retrystrategy`
- `secretbackendrotate`
- `secretexpire`
- `secretrotate`
- `secretsdrainworker`
- `secretspruner`
- `simplesignalhandler`
- `singular`
- `stateconfigwatcher`
- `terminationworker`
- `toolsversionchecker`
- `undertaker`
- `units3caller`
- `upgradedatabase`
- `upgrader`
- `upgradeservices`
- `upgradestepsagent`
- `upgradestepscontroller`

## Suggested Next Passes

- Deep-review `uniter`, `firewaller`, `storageprovisioner`, and
  `remoterelationconsumer` separately. They are large enough that they should
  probably each carry their own audit section.
- Fix and test the six confirmed findings before broadening the audit surface.
- Once those are landed, revisit the watchlist packages that own long-lived
  goroutines or callback-under-lock patterns first:
  `externalcontrollerupdater`, `httpserverargs`, `objectstores3caller`,
  `fortress`, `muxhttpserver`, and `sshserver`.

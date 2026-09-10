# CMR teardown robustness: proposed architectural changes

This note records the five changes suggested during the investigation of
[PR #23223](https://github.com/juju/juju/pull/23223), reviewed at commit
`59221a26131c62e7a361c41ab6618974eece9718`. These are architectural proposals
based on static source inspection, not implemented or integration-validated
behavior. The source references describe that reviewed revision.

The patch adds bounded retries to a compensating teardown notification and
allows local worker cleanup to continue after notification failure. It improves
the chance of surviving a brief failure, but does not establish eventual
convergence. The description acknowledges that limitation and calls for
teardown that survives worker restarts.

The underlying problem is ownership of an unfinished cross-model operation.
Local relation removal and remote teardown have different completion
conditions, yet local cleanup can discard the information needed to complete
the remote operation. Worker existence and process history then become proxies
for domain state.

The proposed invariant is:

> Local deletion must leave either a durable remote-cleanup obligation or
> evidence that the remote side has durably accepted responsibility for cleanup.

## 1. Give the CMR connection an explicit, durable lifecycle owner

Retain a record identifying the relation, both models, the offer, the
participating applications, and the means to authenticate cleanup. Record
registration progress, requested closure, and remote acknowledgement separately
from canonical `Life`.

Local relations and synthetic applications can continue to support the existing
relation machinery. Their deletion must not erase the only record of the
cross-model contract.

A smaller implementation could retain the local relation in `Dying` until its
remote obligation is discharged. If local deletion must proceed independently,
a retained CMR operation or tombstone must survive it. Both approaches require
a transactional deletion condition covering relation, application, and model
removal.

In the reviewed implementation, the relation macaroon has a foreign key to the
local relation, and the CMR deletion transaction removes both. Replacing a
worker lookup with a database lookup alone therefore does not solve the
lifetime problem. See the
[macaroon schema](https://github.com/juju/juju/blob/59221a26131c62e7a361c41ab6618974eece9718/domain/schema/model/sql/0034-cross-model-relation.sql#L57)
and
[deletion transaction](https://github.com/juju/juju/blob/59221a26131c62e7a361c41ab6618974eece9718/domain/removal/state/model/relationwithremoteofferer.go#L249).

## 2. Make workers reconcile persisted obligations

The removal transaction should record the cleanup obligation before deleting
its source information. A worker should enumerate pending obligations on
startup, retry communication, and persist acknowledgement. Watchers provide
prompt wake-ups; retained state provides recovery after a missed notification
or process restart.

Existing persistent removal jobs and handling for incomplete work provide a
possible foundation. Their CMR contracts need to account for remote obligations
as well as local cleanup. Keep network execution in workers and persistence
and invariants behind domain services.

Retry limits may bound one execution attempt. Exhausting those attempts must
leave inspectable pending work, rather than completing the obligation. Whether
a child worker exists must not determine whether remote resources still exist
or whether forced cleanup is appropriate.

The reviewed fallback retrieves identity and credentials from a child worker
and skips notification if that worker is absent. A replacement relation watcher
enumerates current relation endpoints, so it cannot reconstruct obligations
belonging to deleted relations. See the
[fallback notification](https://github.com/juju/juju/blob/59221a26131c62e7a361c41ab6618974eece9718/internal/worker/remoterelationconsumer/localconsumerworker.go#L624),
[initial relation query](https://github.com/juju/juju/blob/59221a26131c62e7a361c41ab6618974eece9718/domain/relation/state/relation.go#L2794),
and
[removal job handling](https://github.com/juju/juju/blob/59221a26131c62e7a361c41ab6618974eece9718/domain/removal/service/service.go#L193).

## 3. Treat registration and teardown as one recoverable protocol

Persist registration intent before calling the offering model. Use the stable
relation UUID to recover or close that specific registration after restart.
An unknown RPC outcome must remain recoverable: the offerer may have committed
the registration even when the consumer never received its response.

The reviewed sequence creates remote resources before the consumer saves the
returned macaroon. Its compensation handles a returned result followed by a
failed save, but cannot cover every crash or lost reply. Ordinary duplicate
registration is already supported; recovery also needs closure to remain
terminal for that identity, including against delayed registration requests.
See the
[registration sequence](https://github.com/juju/juju/blob/59221a26131c62e7a361c41ab6618974eece9718/internal/worker/remoterelationconsumer/localconsumerworker.go#L958)
and
[duplicate registration handling](https://github.com/juju/juju/blob/59221a26131c62e7a361c41ab6618974eece9718/domain/crossmodelrelation/service/remoteapplication.go#L309).

Define an explicit close operation whose acknowledgement means durable
acceptance of cleanup. The offerer can then complete its own departure hooks
and resource removal independently. Track local departure completion and
remote acceptance separately so that neither controller's worker lifetime
owns both halves of the operation. Local transactions and retryable RPCs can
provide this handoff.

## 4. Separate suspension, teardown, and teardown authority

Suspension should suppress ordinary relation traffic while lifecycle
reconciliation continues. In the reviewed implementation, suspension removes
the unit workers from which fallback cleanup retrieves its credentials. A
relation that becomes dying while suspended still needs an owner for teardown.
See
[suspension handling](https://github.com/juju/juju/blob/59221a26131c62e7a361c41ab6618974eece9718/internal/worker/remoterelationconsumer/localconsumerworker.go#L706)
and the related cases in
[issue #23222](https://github.com/juju/juju/issues/23222).

Authorization needs the same distinction. The receiver checks ordinary
relation macaroons before processing a dying notification, so revocation of
consumption permission can prevent relinquishing the established relationship.
See the
[receiver authorization path](https://github.com/juju/juju/blob/59221a26131c62e7a361c41ab6618974eece9718/apiserver/facades/controller/crossmodelrelations/crossmodelrelations.go#L129).

Define narrowly scoped authority to close an established connection, bound to
its immutable identity and owner. This authority must not authorize further
consumption or data exchange. Persisting the existing macaroon alone does not
resolve this authorization contract.

## 5. Include model destruction in the durability boundary

A cleanup record in a model database is insufficient if that database can
disappear first. Normal destruction must wait for durable remote acceptance or
transfer the obligation to a controller-owned record before dropping the model.
The transfer itself must be recoverable, with responsibility retained until
the receiving owner has durably accepted it.

Forced destruction needs an explicit policy for abandoning unresolved
obligations. Exhausting retries must not silently make that decision. Pending
cleanup and its last failure should remain inspectable through an operational
status surface; a warning log is insufficient state.

## Validation expectations

Interrupt execution at each local commit and remote RPC boundary, restart with
no surviving workers, restore connectivity, and require both halves to converge.
Cover suspended relations, revoked consumption permission, lost registration
replies, lost close acknowledgements, duplicate and delayed requests, and model
destruction. Include immediate `remove-relation` followed by `remove-saas`,
without an intervening wait that hides the race.

The reviewed unit test verifies three failed notification attempts followed by
local worker cleanup. It does not establish recovery after restart. No CMR
integration suite or fault-injection validation was run during the original
investigation. These proposals need current-source verification before
implementation.

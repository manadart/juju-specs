# SSH Key Scope and `model_authorized_keys`

## Summary

The proposed SSH tunnelling design makes `model_authorized_keys` redundant if
the SSH server worker performs the model access check before establishing a
session.

The desired model is:

- SSH keys identify Juju users.
- Model permissions authorize SSH access to model destinations.
- The SSH server worker enforces model admin or controller superuser access for
  every SSH tunnel request.

Under that model, `model_authorized_keys` no longer carries independent
security meaning. It can be removed once the worker authorization path is in
place and covered by tests.

## Current Shape

Today, SSH keys are manipulated through the `KeyManager` facade. That facade is
model-scoped: clients open it through a model API connection, and the facade is
constructed with the current model UUID.

The backing data is controller-scoped:

- `user_public_ssh_key` stores user-owned public keys.
- `model_authorized_keys` links those keys to a model.

That link currently serves two purposes:

- It defines the model-specific key set.
- It feeds legacy propagation of permanent user keys to machines.

The tunnelling design removes the need for permanent user key propagation to
machines. If authorization is done by checking model admin access at connection
time, the model-specific key link is also unnecessary.

## Target Semantics

SSH key ownership and SSH destination authorization should be separate.

`user_public_ssh_key` answers:

> Does this public key belong to this Juju user?

Model access answers:

> May this user SSH to this model/entity?

The SSH server worker must enforce the second question before establishing the
tunnel. A matching SSH key alone must not authorize model access.

The expected worker flow is:

1. Authenticate the SSH user by public key against `user_public_ssh_key`.
2. Reject disabled or removed users.
3. Parse and validate the requested virtual hostname.
4. Check that the authenticated user has model admin access, or controller
   superuser access.
5. Establish the tunnel/session only after authorization succeeds.

## Keep `KeyManager` Model-Scoped For Now

The client API will be rewritten later. Until then, keep the existing
model-scoped `KeyManager` facade for compatibility with the current CLI and API
plumbing.

The model scope should become a compatibility artifact only. Key ownership
should be user-scoped controller data, even though the facade is opened through
a model connection.

This needs clear in-tree documentation near the facade and service construction.
Suggested wording:

```text
KeyManager remains a model-scoped facade for client compatibility. SSH keys are
no longer model-scoped; the model context is ignored for key ownership. SSH
access to a model is enforced by the SSH server worker using model admin or
controller superuser permissions.
```

## Table Removal Plan

Remove `model_authorized_keys` after the worker-side authorization check is in
place.

The removal should include:

- Dropping the `model_authorized_keys` table.
- Dropping the `v_model_authorized_keys` view.
- Removing the changelog/watch namespace for `model_authorized_keys`.
- Removing generated triggers for `model_authorized_keys` insert/update/delete.
- Removing trigger generation entries that still list `model_authorized_keys`.
- Updating controller schema tests and generated release DDL expectations so
  they no longer expect the table, view, indexes, triggers, or changelog
  namespace.
- Removing cleanup code that deletes key links per model.
- Removing cleanup code that deletes key links when a user is removed or
  disabled. User removal should delete the user's keys directly; disabled users
  should be rejected in the SSH authentication path.
- Removing propagation code that reads model-authorized user keys for machines.
- Removing keyupdater watch logic that watches `model_authorized_keys`.
  Once permanent user keys are not propagated to machines, machine authorized
  key watchers should no longer depend on model-key association changes.
- Updating keymanager state so add/list/remove operate on the authenticated
  user's global key set.

`user_public_ssh_key` remains in the controller database as the source of truth
for user SSH credentials.

## `SSHClient` Facade Version

Removing the old SSH path also makes most of the current `SSHClient` facade
methods redundant. A new facade version should be added with only the methods
needed by the tunnel-based client flow.

The new version should keep:

- `VirtualHostname`

The new version should remove:

- `PublicAddress`
- `PrivateAddress`
- `AllAddresses`
- `PublicKeys`
- `Proxy`
- `ModelCredentialForSSH`

Those removed methods exist to support direct SSH, legacy controller proxying,
client-side host key construction, client-side address selection, or
client-side Kubernetes exec setup. In the tunnel design, the controller SSH
server owns routing, authorization, host-key behavior, and Kubernetes exec
setup.

This should be implemented as a facade version bump, not as an incompatible
change to existing versions. For example, `SSHClient` v6 can expose only the
tunnel-era surface while v4/v5 continue to serve older clients and older
workflows.

The Juju client code must keep the older client-side methods for compatibility
with older controllers. The client should negotiate the best available facade
version and use the tunnel-era path only when the new version is available. The
old methods can remain in the client package as compatibility shims for older
controllers, even though they are not part of the new server-side facade
version.

## Migration Behavior

Existing deployments may have different keys for the same user authorized in
different models. Removing `model_authorized_keys` means all of a user's
existing public keys become global credentials for that user.

That is acceptable only because model SSH access is constrained by the worker's
model admin or controller superuser check. This change should be called out in
migration notes and release documentation.

During model migration, model-scoped SSH key associations should be treated as
legacy input only. The model scope should be ignored for key ownership, and the
resulting user-key associations should be collapsed into each user's global key
set on the target controller.

If the same user has different SSH keys authorized in multiple migrated models,
the target controller should store the union of those keys for that user. If the
same key appears more than once for a user, import should de-duplicate it using
the normal user/key uniqueness rules.

The important migration invariant is:

```text
old: user key authorized for model N
new: key belongs to user; model access authorizes SSH to model N
```

This means migrating a model must not recreate per-model key scope on the target
controller. It should only ensure that users referenced by the migrated model
have their relevant public keys available as user-scoped credentials.

## Required Tests

Add test coverage for:

- A valid user key authenticates independently of model.
- A user with a valid key but without model admin access cannot SSH.
- A model admin with a valid key can SSH.
- A controller superuser with a valid key can SSH.
- Disabled or removed users cannot authenticate.
- The `KeyManager` facade still works through a model connection.
- Add/list/remove key operations are global for the authenticated user even
  though the facade remains model-scoped.

## Permission Denial Surface

Permission denial is surfaced differently depending on where it occurs.

For normal Juju API calls, `apiservererrors.ErrPerm` is serialized as a
`params.Error` with:

- message: `permission denied`
- code: `unauthorized access`

The existing `KeyManager` facade should keep this behavior for add/list/remove
operations.

For SSH tunnelling, there are two separate denial cases:

- Authentication failure: the key is unknown, or the user is disabled or
  removed. This should surface as a normal SSH public key authentication
  failure.
- Authorization failure: the key is valid, but the authenticated user does not
  have model admin or controller superuser access for the requested destination.

The worker should reject unauthorized tunnel/session establishment before
connecting to the target. The SSH protocol surface may be generic, for example
an SSH channel open failure such as `administratively prohibited:
unauthorized`. That is acceptable at the worker level and avoids leaking target
existence details.

The eventual rewritten client API should provide a clearer Juju-level user
message before or around the SSH attempt, for example:

```text
permission denied: model admin access is required to SSH to this target
```

## Compatibility Debt

Keeping `KeyManager` model-scoped while ignoring the model is intentional but
temporary. It should be documented as compatibility debt until the client API is
rewritten around user-scoped SSH key management.

The long-term shape should be a controller/user-scoped key API where the model
context is not part of key management at all.

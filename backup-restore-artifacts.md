# Backup and Restore Artefact Exploration

This note summarizes what the Juju 3.6-and-earlier Mongo backup path covered,
then sketches a high-level approach for selecting and restoring equivalent
artefacts in a dqlite-based controller.

## Existing 3.6 Backup Shape

The old `juju create-backup` command produced a compressed tar archive with
one top-level directory:

```text
juju-backup/
  metadata.json
  root.tar
  dump/
```

`metadata.json` carried backup metadata: format version, source model UUID,
controller UUID, source machine, hostname, Juju version, base, HA node count,
timestamps, notes, checksum, size, and related controller-machine identifiers.

`dump/` was produced by `mongodump --oplog` using the controller's Mongo
credentials and TLS material. The backup code dumped Mongo database state, then
removed the `admin` and `backups` database directories from the dump. The
`presence` database was listed as ignored, but the old cleanup only removed
`admin` and `backups`, so `presence` could remain in the archive.

`root.tar` was a bundle of selected controller-machine files. It was not a full
filesystem snapshot.

## Existing 3.6 File Artefacts

The old file selector included regular files and symlinks under these paths:

- `/var/lib/juju/tools/**`: Juju agent tools, including `jujud` binaries and
  per-machine symlinks.
- `/var/lib/juju/agents/machine-*/*`: controller machine agent files,
  especially `agent.conf`; sockets were skipped.
- `/var/lib/juju/init/**`: Juju-managed init or systemd service files.
- `/var/lib/juju/system-identity`.
- `/var/lib/juju/server.pem`.
- `/var/lib/juju/shared-secret`.
- `/var/snap/juju-db/common/shared-secret`, when the shared secret lived in the
  juju-db snap data directory instead of `/var/lib/juju`.
- `/var/lib/juju/nonce.txt`, when present.
- `/home/ubuntu/.ssh/authorized_keys`, when present.

The old file selector did not include:

- `/var/log/juju` filesystem logs.
- Mongo data files.
- The whole `/var/lib/juju` tree.
- Whole machine disks or provider instance state.
- Snap packages.
- The local `juju` client binary.
- Equivalent local files from secondary HA controller machines.

## Existing 3.6 Restore Behaviour

`juju-restore` unpacked the outer archive, then unpacked `root.tar` into the
temporary expanded backup directory. The core restore interface only used the
backup metadata and the Mongo dump directory; it was not a general filesystem
rollback of `root.tar` onto the target controller.

The restore path used `mongorestore --drop` against the target controller. It
always excluded `logs.*` collections, even when log collections were present in
the Mongo dump. It also excluded `juju.statuseshistory` unless the operator
passed `--include-status-history`.

The restore utility managed controller agents around the database restore. In
HA, it stopped and started controller machine agents across controller nodes
when requested. If the restored database represented a different Juju version,
the tool updated controller-machine `agent.conf` files and `/var/lib/juju/tools`
symlinks to point at an already-present tools directory for that version.

The old restore workflow expected the target controller machines to exist. It
did not terminate provider instances created after the backup, and it was not a
disaster-recovery mechanism for recreating all lost controller machines from
the archive alone.

## Implications For A New Backup Path

The old logic treated database state as the authoritative restore payload. The
filesystem bundle served as supporting controller-local context, compatibility
material, and inspection material, not as the main restore mechanism.

A dqlite-era backup should make this distinction explicit:

- **Control-plane data**: Data needed to reconstruct controller and model state.
- **Runtime identity artefacts**: Certificates, system identity, and agent
  configuration required for the restored controller to authenticate and resume
  control-plane work.
- **Runtime executable artefacts**: Agent tools or enough version information to
  reacquire the exact tools needed during restore.
- **Operational diagnostics**: Logs and status history, which may be valuable
  for audit/debugging but should not necessarily be restored by default.
- **Machine/service integration artefacts**: Init/systemd units and local service
  wiring needed to restart the controller agent cleanly.
- **Host-user access artefacts**: SSH authorization state is intentionally out
  of scope for the baseline backup artefact set.

## Known Dqlite Backup Baseline

There are several starting points we can treat as fixed:

- Juju stores the controller database and each model database as separate dqlite
  databases.
- Backup will export all model databases. This export path is already supported.
- Backup will generate the required direct-from-database selection logic for the
  controller database.
- The new separate restore tool will create each separate model database
  verbatim from its export.
- Controller database restore is the open design area. At minimum, restore needs
  the model metadata from the exported controller database so the restored
  controller can see and manage the restored model databases.

The controller database cannot be treated as obviously verbatim. Controller
restore assumes the previous controller is hosed and the target machine becomes
controller machine `0`, regardless of whether it was part of the old controller
set. The controller model database is created from export, but then needs a
restore-target fixup so controller machine records, controller unit records,
agent bindings, instance data, network data, and controller-node metadata match
the new environment while still reconnecting to the restored model databases.

For filesystem artefacts, the dqlite-era baseline is also simpler than the old
Mongo path:

- Include everything under `/var/lib/juju`.
- Skip sockets, as the old backup did.
- Do not include `/var/lib/juju/shared-secret`; that was Mongo-specific.
- Do not include SSH keys such as `/home/ubuntu/.ssh/authorized_keys`.

## Controller Restore Target Assumptions

Code inspection points to the controller restore as a special case layered on
an otherwise verbatim model-database restore.

For this exploration, assume:

- The existing controller is hosed.
- There are no other live controllers to preserve or coordinate with.
- The restore target machine may or may not have been part of the source
  controller set.
- The restore target becomes controller machine `0`.
- The restored controller model contains exactly one controller machine and one
  controller unit after fixup: machine `0` and unit `controller/0`.
- Restore may need to write new instance and network data that describes the
  restore target rather than the source controller host.

The controller model database is therefore the main exception to verbatim model
restore. It can be created from the export like every other model database, but
then it needs a target-specific fixup pass before agents resume.

## Code-Grounded Restore Surfaces

The controller database contains the controller-wide identity and model
inventory:

- `controller`: singleton controller row, including controller UUID, controller
  model UUID, target version, API port, certs, CA material, private keys, and
  system identity.
- `controller_node`: controller IDs and dqlite node identity/bind address.
- `controller_node_agent_version`: reported agent version per controller node.
- `controller_api_address`: API addresses per controller ID.
- `controller_node_password`: controller-node agent password hash.
- `model`: authoritative controller-side model rows.
- `model_namespace`: mapping from model UUID to dqlite namespace. Existing model
  creation uses the model UUID as the namespace.
- `namespace_list`: tracked dqlite namespaces.

The controller model database contains the controller machine and unit shape:

- `model`: singleton read-only model row. For the controller model,
  `is_controller_model` is true and `controller_uuid` must agree with the
  controller database.
- `application`: the controller charm application is named `controller`.
- `application_controller`: sparse singleton marker for the controller
  application.
- `unit`: the controller unit is `controller/0`.
- `machine`: the controller machine is `0`; bootstrap uses nonce
  `user-admin:bootstrap`.
- `net_node`: the shared network node that ties machine `0` and `controller/0`
  together.
- `machine_cloud_instance` and `machine_cloud_instance_status`: provider
  instance identity, display name, hardware, life, and status for machine `0`.
- `link_layer_device`, `provider_link_layer_device`, `ip_address`,
  `provider_ip_address`, `fqdn_address`, `hostname_address`, and their
  `net_node_*` link tables: target host network identity.
- `subnet`, `provider_subnet`, `provider_network`,
  `provider_network_subnet`, `availability_zone`, and
  `availability_zone_subnet`: network context needed for restored addresses.
- `machine_agent_version`, `machine_platform`, `machine_status`,
  `machine_agent_presence`, and `unit_agent_*` rows: runtime/status rows that
  may need reset or target-aware rewrite.

The code treats a machine as a controller machine by joining through
`application_controller`: `v_machine_is_controller` joins `machine -> net_node
-> unit -> application -> application_controller`. That means the controller
restore fixup must keep machine `0` and unit `controller/0` on the same
`net_node_uuid`, and must preserve exactly one `application_controller` row for
the controller application.

## Candidate Controller Restore Fixup

After the controller model database is created from the export, the restore tool
should reconcile it to the target machine:

1. Locate the controller application through `application_controller` and the
   controller unit named `controller/0`.
2. Keep or create one machine row named `0` and one unit row named
   `controller/0`.
3. Ensure machine `0` and unit `controller/0` share a single `net_node_uuid`.
4. Remove other controller machines and controller units from the controller
   model database, including their dependent machine/unit/network rows.
5. Rewrite machine `0` cloud instance data with target instance ID, display
   name, hardware characteristics, life, and status.
6. Rewrite network rows for machine `0` from target discovery: link-layer
   devices, provider device IDs, addresses, provider address IDs, hostname/FQDN
   addresses, subnets, provider networks, spaces, and availability zones as
   needed.
7. Reset volatile presence/status rows that describe old agents rather than
   the restore target.
8. Update the controller database to one controller node, controller ID `0`,
   with target dqlite node ID, dqlite bind address, API addresses, agent
   version, and controller-node password hash.
9. Ensure controller database `model`, `model_namespace`, and `namespace_list`
   rows match the model databases recreated by restore.

This is deliberately a target-aware reconciliation step, not a normal model
migration import. The existing generated model import path inserts exported
model tables verbatim; controller restore needs to mutate the controller model
because controller machine identity and network identity are properties of the
restore target.

## Candidate Backup Artefact Classes

### Data Snapshot

Capture dqlite content through structured export rather than opaque filesystem
copy. Model databases can be exported and restored verbatim. The controller
database needs generated direct-from-database selection logic so restore can
choose which controller records are copied directly and which are transformed
for the target controller.

The data snapshot should record:

- Schema version and Juju version.
- Source controller UUID and controller model UUID.
- Model database inventory and per-model database export metadata.
- Source machine ID and HA topology.
- Export selection manifest.
- Any intentionally excluded data, such as volatile presence or ephemeral worker
  state.

### Identity And Trust

Capture the artefacts that let restored controller components prove identity and
authenticate to each other.

Likely artefacts:

- Controller CA certificate and private key material, subject to secret-handling
  review.
- API server certificates or enough CA material to regenerate them.
- Agent TLS material.
- System identity.
- Nonce material when still relevant to current agents.

These artefacts need an explicit sensitivity classification. Restore should
avoid silently replacing target secrets unless the target is intended to become
the backed-up controller.

### Agent Runtime

Capture enough information to restart agents at the backed-up Juju version.

Two approaches are possible:

- Bundle the exact controller agent tools needed by the backup, preserving the
  old `root.tar` behaviour.
- Store a manifest of required tools and require restore to verify or fetch
  them before committing database changes.

Bundling is more self-contained. Manifest-only restore is smaller and cleaner,
but it makes restore depend on external artefact availability.

### Service Integration

Capture controller-machine service wiring that is Juju-owned and needed to
restart agents. The known filesystem baseline is to include the `/var/lib/juju`
tree, while excluding sockets and the old Mongo `shared-secret`.

Restore should treat these files as target-machine integration artefacts. It
should validate before overwriting because paths and service managers may differ
between source and target bases.

### Diagnostics

Capture diagnostics separately from restorable state.

Likely artefacts:

- Controller logs from `/var/log/juju`, if configured and size limits allow.
- Database logs or dqlite diagnostics.
- Status history, behind an include/exclude policy.
- Backup creation logs and manifest validation results.

Diagnostics should be available for inspection after restore, but should not be
replayed into active controller state unless an operator explicitly opts in.

### Operator Access

The old backup included `/home/ubuntu/.ssh/authorized_keys` when present. The
new dqlite-era baseline is to exclude SSH keys and other operator access state.
That keeps controller backup focused on Juju-owned runtime state rather than
host-user access policy.

## Candidate Restore Phases

1. **Inspect**: Open the archive, validate the manifest, report source Juju
   version, schema version, controller identity, HA topology, and artefact
   classes present.
2. **Preflight**: Verify target compatibility: Juju version, schema support,
   controller identity policy, HA node count or topology policy, base, storage,
   service manager, and required tools.
3. **Quiesce**: Stop local controller agents and any local database writers.
   The previous controller is assumed hosed, so there are no live peer
   controllers to coordinate with.
4. **Stage**: Restore data and files into a staging location. Validate checksums,
   schema compatibility, foreign-key integrity, and required runtime artefacts.
5. **Create model databases**: Create each exported model database. Non-controller
   model databases are restored verbatim.
6. **Fix controller model data**: Apply the target-machine reconciliation for
   machine `0`, unit `controller/0`, instance data, network data, and volatile
   runtime rows in the controller model database.
7. **Apply controller database data**: Import or transform controller database
   rows so there is one controller node, model metadata matches the recreated
   model databases, and controller identity matches the selected restore policy.
8. **Apply runtime artefacts**: Install or update secrets, agent configs, tools,
   and service wiring for the restored single-controller target.
9. **Resume**: Restart dqlite and controller agents in a deterministic order,
   then wait for the single-node database and API server to become healthy.
10. **Verify**: Re-query restored state, verify agents connect, verify API
   availability, and report any diagnostics excluded from active restore.

## Restore Scope

For controller restore, this note assumes a single-target recovery path: the old
controller is hosed, there are no live peer controllers, and the restore target
becomes controller machine `0`. The restore tool may be running on a machine
that used to be part of the source controller set, but restore should not rely
on that. Source controller membership is imported data to reconcile, not live
infrastructure to preserve.

Forensic inspection remains a separate non-mutating mode: unpack and inspect
controller exports, model exports, filesystem artefacts, logs, and manifests
without applying them to a live controller.

## Open Questions

- Which dqlite tables are restorable state, and which are volatile runtime
  state that should be excluded or regenerated?
- Should backup capture dqlite at the file/snapshot level, generated selection
  level, or both?
- Which source identity and trust artefacts must be preserved, and which
  target-local identity values must be regenerated for the restored controller?
- Should agent tools be bundled, manifest-only, or configurable?
- Should filesystem logs be included by default, capped by size, or kept behind
  an explicit option?
- Which source HA/topology rows must be discarded or rewritten for the
  single-controller restore target?
- What restore guarantees are required before agents restart?
- Which artefacts must be encrypted at rest inside the archive?
- Does backup need a formal manifest with per-artefact checksums, sensitivity,
  restore policy, and schema/data selectors?

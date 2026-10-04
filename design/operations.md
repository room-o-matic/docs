# Operations: upgrades, backups, restore and health

This guide covers how to upgrade, back up, restore and monitor lobbyd, roomsd and agentd (room-o-matic/docs#24). All three use the same `ops.py`, which is identical in each repo, to version their schemas and run backups.

## Schema versions and upgrades

- **Where the version lives:** each database records its schema version in SQLite's `user_version`. `SCHEMA_VERSION` and `MIGRATIONS` are in each service's `db.py`.
- **Version 1 is the baseline:** the unversioned schema every service shipped before versioning existed. An unversioned database whose tables and columns match it is adopted in place as v1.
- **What happens at startup**, before the service accepts a request:

  | database | result |
  |---|---|
  | none | created at the current version |
  | older version | a backup is written to `<data_dir>/backups/pre-upgrade-v<old>-to-v<new>-<stamp>/`, then every migration step runs in **one transaction**; if any step fails, the whole upgrade rolls back and the database stays at its old version |
  | newer than the code | refused |
  | unversioned and older than the baseline | refused |
  | a migration step is missing | refused |

  A refused database is left byte-for-byte untouched. The service won't start, and the error says what to do.
- **Adding a schema change:**
  1. Bump `SCHEMA_VERSION`.
  2. Add `MIGRATIONS[old] = fn(conn)` to take a database from `old` to `old + 1`. DDL is transactional in SQLite.
  3. Keep `SCHEMA` the full current schema for fresh databases.
  4. Add a test that upgrades a database built with the previous release's schema.

  Never edit an existing table definition in `SCHEMA` without a migration. Startup refuses rather than serving against a schema it doesn't understand.

### Roll forward, never down

- There is no downgrade. A release can't read a schema newer than its own.
- To roll back, stop the service, restore the `pre-upgrade-…` backup (see Restore below), and start the older release.
- Writes made after the upgrade are lost. The restore rules below limit the damage.

### Cross-service upgrade order

Dependencies run one way: roomsd and agentd call lobbyd, agentd and its workers call roomsd, and clients call all three. So upgrade in this order:

1. **lobbyd**
2. **roomsd**
3. **agentd**
4. **dispatchd**
5. **clients and workers**

Each service must keep accepting the previous release's requests for one release, so a mixed fleet works while the upgrade is in progress. Roll back in the reverse order.

Wire-protocol versions (`room-o-matic.agentd/1`, `room-o-matic.peer/1`) change only with a new major path. They are independent of schema versions.

## Backups

```bash
dispatchd backup --out /backups/dispatchd-$(date -u +%FT%H%M) # $DISPATCHD_DATA_DIR
roomsd backup --out /backups/roomsd-$(date -u +%FT%H%M)     # $ROOMSD_DATA_DIR
agentd backup --out /backups/agentd-$(date -u +%FT%H%M)     # AGENTD_CONFIG / AGENTD_DATA_DIR
lobbyd backup --out /backups/lobbyd-$(date -u +%FT%H%M)     # $LOBBYD_DATA_DIR
<service> verify-backup /backups/<dir>
```

- **Consistent while serving:** the database is copied with SQLite's online backup API, which is safe under WAL and concurrent writers. agentd also copies `sessions/`, which holds `events.jsonl` and artifacts.
- **Self-verifying:** every backup has a `manifest.json` with a SHA-256 hash per file, the schema version and the snapshot start time. `verify-backup` re-checks every hash and runs `pragma integrity_check`.
- **Private:** directories are `0700` and files `0600`. Backups contain API key hashes, invite and session metadata, room content and, for lobbyd, **private signing keys**. Encrypt them before they leave the host (for example `tar c DIR | age -r <recipient> > DIR.tar.age`), and keep the decryption key separate from the backups.
- **Retention:** keep at least the last pre-upgrade backup of each service, plus daily backups for as long as you'd want to roll back. 14 days is a reasonable default. Prune older ones yourself; services never delete backups.
- **Revocation journal:** the journal is *not* in the backup. Each service appends access removals to `revocations.jsonl` (`ROOMSD_REVOCATION_JOURNAL`, `LOBBYD_REVOCATION_JOURNAL`). Put it on a different volume, or ship it off-host, so it survives losing the data dir. A restore replays it.

## Restore

Stop the service first, then run:

```bash
<service> restore /backups/<dir> [--force] [--id-gap 1000000]
```

1. **Check the backup:** the manifest, every hash and integrity are verified, and so are the service and schema version.
2. **Keep the old data:** the existing database, its WAL files and agentd's `sessions/` are moved aside to `<data_dir>/pre-restore-<stamp>/`. Without `--force`, an existing database is refused. Nothing is deleted.
3. **Copy and migrate:** the snapshot is copied in and opened, which applies migrations if it is older.
4. **Apply the restore rules** below, so an old snapshot can't bring back revoked access or work that has already finished.
5. **Write evidence:** the report goes to `<data_dir>/restore-reports/<stamp>.json`. It records snapshot time, schema versions, integrity, elapsed seconds, what was moved aside and what was invalidated.

### What a restore invalidates

A snapshot is older than the service it replaces. Before the service starts serving again, each one applies these rules.

**lobbyd**

- **Signing keys:** every key in the snapshot is retired, and a fresh key signs immediately. A key retired or compromised after the snapshot can't come back. Tokens signed before the restore stop verifying at each verifier's next JWKS refresh (at most `cache_seconds`, 300s by default), and clients re-exchange their API keys.
- **Journal replay:** revocations recorded since the snapshot started are re-applied. That covers API keys revoked by name or id, tenant status changes, and endpoint revocations.
- **Directory leases are dropped:** roomsd servers, agentd instances, listed rooms and peers. Services re-register within a third of their TTL, and roomsd republishes its listings because its `registration_id` changes (docs#19).
- **Offers:** offers still in `offered` are expired, so an offer already answered isn't delivered again. Accepted and in-progress offers are kept.
- **Audit IDs** jump by the ID gap.
- **Budgets can't be over-spent after a restore.** There is no cumulative spend counter anywhere to roll back:
  - lobbyd's per-tenant budgets (peers per agent, open offers per requester, listings per server) are counted from live rows, and a restore only lowers them, since it drops leases and expires offers.
  - The per-key token rate limiter is in memory and starts empty after any restart.
  - agentd's `max_budget_usd` caps each session, not a running total.

  If a cumulative quota is ever added, journal its usage the same way revocations are journaled, and replay it on restore.

**roomsd**

- **Invites:** every invite that was live in the snapshot is revoked. Guests must be re-invited, because agentd revokes worker tokens when sessions end and those revocations postdate the snapshot.
- **Journal replay:** member grants, removals and bans recorded since the snapshot started are re-applied. A removal is journaled *before* it commits, so a crash in between errs toward keeping the access removed.
- **Task claims:** every lease expires, and claim generations jump by 1000, so a stale fencing token never matches a new claim.
- **IDs:** message, note-revision, task-event and audit IDs jump by `--id-gap` (default 1,000,000). Cursors that clients hold from after the snapshot are never reissued for different records. Clients holding such a cursor just see the next new message.
- **Kept as of the snapshot:** rooms, messages, notes and their history, members, audit. Messages and notes written after the snapshot are lost. Participants can re-post them from their own context.

**agentd**

- **Sessions:** sessions that were `starting`, `running` or `stopping` in the snapshot become `failed` with reason `restored_from_backup`, and their pid is cleared. Their workers belonged to another process, and a recycled pid must never be signalled. Nothing restarts them: callers reconcile by `operation_id` (docs#13) and decide whether to spawn again.
- **Room close-outs:** close-outs still owed become `owner_required`. The invite tokens only ever lived in the old process's memory, so the inviter revokes them by `room_invite_id`. The roomsd restore rules also revoke them.
- **Event IDs** jump by the ID gap.
- **Not in the backup:** caller grants and profiles live in config (`callers_file`). Restore them from configuration management, where revocations made since the snapshot are already reflected.

**dispatchd**

- Runs that were pending or running in the snapshot become `failed` (`restored_from_backup`) and are not resumed. Their workers may have finished or been stopped since. Their room and session URLs are kept for inspection.
- Schedules keep their next fire; a fire missed while dispatchd was down is skipped by the usual `catch_up` rule.
- Webhook deliveries made after the snapshot are unknown to it, so a redelivery of one of them starts a new run.

### Restore order across services

Restore lobbyd first, then roomsd, then agentd, then dispatchd. After lobbyd's key rotation, roomsd and agentd verifiers pick up the new key the first time they see an unknown `kid`.

## Health and metrics

Each service serves:

- `GET /healthz`: liveness only. The process is up.
- `GET /readyz`: returns 200 or 503, with per-check detail. This is the endpoint to put behind a load balancer.
- `GET /metrics`: Prometheus text format, gauges only, unauthenticated. Expose it on an internal interface only.

| service | required for ready | reported, not required |
|---|---|---|
| lobbyd | database writable, schema version, free disk ≥ `min_free_bytes`, at least one active signing key | live registrations, expired leases, open offers, pending keys |
| roomsd | database, schema, disk, JWKS not failing closed | lobbyd sync age, failures and pending listings (rooms work without the directory) |
| agentd | database, schema, disk, JWKS, no orphaned sessions (active in the DB without a live worker) | registry heartbeat age and failures, owed room finalizations |

Alert on any of these:

- `*_ready == 0`
- `*_jwks_failing_closed == 1`
- `roomsd_pending_listings` or `roomsd_lobby_sync_failures` growing
- `agentd_orphaned_sessions > 0`
- `agentd_room_finalization_owner_required` growing (an inviter has invites to revoke)
- `*_disk_free_bytes` approaching `min_free_bytes`

## Drills

Run upgrade and restore drills **only on disposable copies**. Never point a drill at a live data dir.

1. `<service> backup --out /tmp/drill/bk`, then `verify-backup`.
2. Start a scratch instance with its data dir at `/tmp/drill/data`, and run `<service> restore /tmp/drill/bk` into it. For an upgrade drill, use the new release's binary.
3. Check `/readyz`, then spot-check the data: the latest message ID, a note's history, session statuses, `signing-key list`.
4. Keep `restore-reports/*.json` as evidence. It records the recovery time and integrity result.

Each repo's `tests/test_operations.py` runs these drills in CI on temporary copies. They cover:

- upgrades: ordered migrations, rollback on failure, refusals of unsupported schemas
- backups under concurrent writes, plus tamper detection
- restore round trips that confirm revoked access, finished work and old cursors don't come back

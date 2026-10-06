# Redis durability, memory and application failure behavior

Task: [A07 / infrastructure issue #41](https://github.com/vstore-ex/vstore-infra/issues/41).
The workspace owner selected `everysec` on October 6, 2026: prioritize low disk/VM
load; the workload is light and accepts limited recent data loss. Backend review
of application failure behavior and independent infrastructure review remain
pending. This document does not confirm production deployment. The implementation changes
infrastructure, deployment guards and tests; it does not change application logic.

## Proposed policy

Keep one Redis 7.2 instance with its existing `redis_data` volume, private network
and loopback port. Use AOF with `appendfsync everysec`, `maxmemory 64mb` and
`maxmemory-policy noeviction` inside the existing 256 MiB container limit. Do not
enable cache eviction on this shared instance: token keys also have TTLs, so even
`volatile-*` policies can remove authentication state. Logical databases do not
isolate memory or persistence.

AOF fsync stays enabled during rewrite (`no-appendfsync-on-rewrite no`). Automatic
rewrite starts at 100% growth with a 64 MiB minimum. Periodic RDB saves are disabled
to avoid a second scheduled persistence workload; manual adoption takes a snapshot
before conversion. `aof-load-truncated no` refuses an incomplete AOF instead of
silently discarding its tail. AOF files, their manifest and base files all belong
to the persistent volume; a single incremental file is not a complete backup.

The 64 MiB Redis budget leaves 192 MiB for allocator/RSS overhead, clients,
persistence buffers, page cache and copy-on-write during a fork. This is a
conservative starting limit, not proof against every OOM: large commands, scripts,
client buffers and fragmentation can exceed Redis's accounting. Do not increase
the dataset limit without measuring rewrite peaks and the whole VM budget in
[A15](https://github.com/vstore-ex/vstore-infra/issues/42). The October 2 audit
measured about 1.82 MiB; that measurement is historical, not today's utilization.
No additional instance or paid Azure resource is introduced.

## Durability and loss contract

A graceful restart keeps queued messages, unacknowledged-delivery metadata,
refresh tokens, revocations and other unexpired keys on the same healthy volume.
TTL deadlines still advance while Redis is stopped; expiry is not data loss.
A SIGKILL after a confirmed fsync reloads the same data in the isolated drill.
The drill does not simulate abrupt VM power loss or validate the host disk.

With `everysec`, a host crash can lose roughly the latest second of writes under
healthy storage; delayed or failed storage provides no strict one-second bound.
That loss can resurrect a recently revoked access token, discard a newly issued
refresh token, or lose a queued task. The workspace owner's selection accepts
limited recent loss; backend review must confirm the consequences for auth and
broker behavior. If acknowledged revocations must never be lost
on a storage-preserving host crash, review `appendfsync always`, benchmark its
latency and rerun the drill before adoption. Neither mode survives loss of the
volume or guarantees correct behavior when Redis is unavailable.

Celery uses late acknowledgement and rejection on worker loss. Persisted delivery
metadata supports redelivery after the visibility timeout; it does not provide
exactly-once execution. The drill consumes a real Kombu delivery, exits without
acknowledgement, verifies its metadata after Redis restart/crash and restores it
after advancing only the fixture's visibility deadline. Task idempotence and
side effects remain the backend owner's responsibility.

Disk-full/write errors require maintenance; a successful PING does not prove AOF
health. Check `aof_last_write_status`, `aof_last_bgrewrite_status` and rewrite/fsync
progress before deployment. On corrupt/truncated AOF, preserve the volume and
stop writers. Do not delete the AOF, run automatic repair, disable persistence or
start an empty replacement to bypass a failure. Recovery and deliberate session
invalidation require an explicit operator/backend decision. AOF is not an off-host
backup; VM-loss recovery remains [A06](https://github.com/vstore-ex/vstore-infra/issues/40).

## State inventory and observed application behavior

The unchanged application snapshots come from backend commit
`59c61caa38b439f2cf8b24f5f7d272d69ffff541`; the fixture manifest records each file
and SHA-256. These tests describe existing behavior, not approved security behavior.

| State | Retention need | Current error/loss behavior |
| --- | --- | --- |
| Celery queued tasks and unacknowledged deliveries | Preserve until acknowledgement; duplicate processing remains possible | Publish raises on connection failure/OOM; lost volume loses pending jobs |
| Celery results | Retain according to task expiry | Missing result is unavailable; inspect task before retrying side effects |
| Access-token blacklist | Preserve until the access token expires | Read failure returns false; write failure is logged and swallowed; missing state no longer blocks a revoked token |
| Refresh tokens and device index | Preserve until expiry/revocation | Missing/read error returns invalid; failed creation still returns an unusable token; failed revoke returns false |
| Cache | Disposable and rebuildable | Read error becomes a miss; failed write is skipped |
| Rate-limit counters | Short lived, but security relevant | Connection/OOM error fails open; lost state resets counters |
| Locks | Transient coordination, never proof of exactly-once behavior | Default acquisition fails closed; explicit `fail_open=True` proceeds without a Redis lock |
| Presence | Disposable | Read errors appear offline; updates log failure |
| Notifications | Review delivery/retention requirements with backend owner | Stored items/unread counts can be lost; Redis Pub/Sub events have no replay |
| View counters awaiting PostgreSQL synchronization | Preserve unsynchronized counts or accept undercount | Lost state can lose views not yet flushed to PostgreSQL |

The code requests logical DBs 1/2/3/4/5/7/8/9, but the current factory calls
`redis.from_url(settings.CELERY_BROKER_URL, db=...)`. With the deployed URL ending
in `/0`, URL parsing overrides the keyword: the real-client fixture confirms
every client uses DB 0. Do not assume logical isolation in tests or run FLUSHDB
on the production instance. Correcting the factory belongs to backend work.

At the memory limit Redis rejects allocations with OOM and does not evict existing
broker or auth keys. Existing reads and freeing keys remain possible. This protects
retained state but exposes existing swallowed-write/fail-open behavior; persistence
alone does not fix it. Separate disposable cache only after measurements show
contention and backend owners provide independent URLs and an adoption plan.

## Backend handoff required before closing A07

Backend owner should review the following findings and either fix them in a
backend PR or explicitly accept the behavior with rationale and follow-up:

1. Blacklist lookup/write failures can accept a revoked access token or report
   logout without retaining revocation. Define a fail-closed/API error contract.
2. Refresh-token creation returns success after storage failure; rotation can
   remove the old token before failing to store its replacement. Define atomic
   rotation and failure responses; verify endpoint/cookie behavior.
3. Security rate limits fail open on Redis errors/OOM. Choose the acceptable
   response for login, registration and uploads; test actual endpoints.
4. Correct URL/DB precedence and isolate integration-test state under A14.
5. Review fail-open order locks, durable notification/view requirements and
   broker redelivery/idempotence. The infra fixture tests core services and broker
   transport, not full HTTP auth flows, SQL transactions or media side effects.

Do not describe the current baseline assertions as security fixes. Infrastructure
and backend acceptance, independent human review and production adoption remain
required; A07 uses `Refs` rather than automatic issue closure.

## Existing-host adoption

Only execute after the above loss/error policy and a maintenance window are
approved. Pause app/infra deployments and external writers. Confirm A01 release
pins, A16 permissions, backup/recovery readiness and available VM memory/disk.
Install the reviewed A07 Compose/scripts in the deployment clone through the
approved rollout. Keep application/edge digests and the existing volume.

An initial infrastructure deployment refuses an RDB-only live instance before
container convergence. Synchronizing Git may already have installed the reviewed
config, but this failed preflight does not convert Redis or discard its dataset.
Do not bypass it with raw Compose.

From the private deployment directory:

```bash
bash scripts/compose.sh stop backend worker
# Confirm no external producer, session or background writer remains.
bash scripts/compose.sh redis-adopt
bash scripts/compose.sh redis-check
bash scripts/compose.sh up -d --wait --wait-timeout 150
bash scripts/compose.sh verify
```

`redis-adopt` shares `.deploy.lock`, refuses active backend/worker writers and
datasets over 48 MiB, and requires at least 256 MiB free in `/data`. It snapshots
the quiescent live dataset, enables AOF in the running process, waits for a healthy
rewrite/fsync, then recreates only Redis with reviewed durable startup flags.
It never merely restarts an RDB-only instance with an empty AOF. Runtime CONFIG
SET alone does not survive restart; keep writers stopped until recreation and
verification succeed. The startup guard also refuses a snapshot-only volume.
For a genuinely empty new volume, the reviewed command initializes AOF normally.

If conversion fails, retain the volume/AOF and stopped writers for investigation;
do not turn AOF back off. Take a reviewed volume backup before conversion, since
the adoption snapshot replaces the volume's prior `dump.rdb`. Failed startup,
disk capacity or corrupt AOF requires operator recovery, never automatic deletion.

Confirm queued/reserved workload and sessions through approved smoke accounts,
then verify both HTTPS API paths and resume traffic/deployments. Do not print
token values, snapshots, resolved Compose secrets or customer keys in evidence.

## Rollback and verification

Infrastructure CD captures the reviewed Redis command in a private temporary
override before convergence. If another stack/edge gate fails and Git is restored,
the rollback still uses AOF startup flags; it cannot revert Redis to defaults that
ignore AOF and load a stale RDB. Application image rollback does not change Redis.
After config rollback to a pre-A07 commit, keep deployments paused and restore the
reviewed Redis config/helpers before further manual commands or deployment: the
temporary override protects that rollback attempt, not an arbitrary future old
checkout. Do not downgrade Redis's data format or disable AOF as an image rollback.

From the infrastructure repository:

```bash
python3 -m unittest discover -s tests -p 'test_*.py' -v
docker compose --env-file .env.example config --quiet
REDIS_POLICY_REPORT=/private/tmp/redis-policy.json python3 tests/smoke_redis_policy.py
```

The Docker drill publishes no host ports, uses generated test tokens and uniquely
named volumes, and removes only its own stack. It verifies live RDB-to-AOF adoption,
startup refusal, graceful restart, SIGKILL after fsync, TTL expiry, queued/reserved
messages, refresh/blacklist state, concurrent AOF rewrite, noeviction pressure,
container memory peak/zero OOM kills, refusal of truncated AOF, current application errors and an empty
replacement volume. This is bounded Redis evidence, not a whole-VM capacity test
or simulated power failure. Application OOM branches use a separate deterministic
phase: the fixture temporarily sets `maxmemory` below its retained dataset, then
restores 64 MiB. This avoids races where connection/buffer reclamation leaves room
for a tiny write; the memory/rewrite measurement itself uses the production limit.
Update snapshots and their provenance when backend
behavior changes.

Sources: [Redis persistence and live conversion](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/),
[Redis memory and eviction](https://redis.io/docs/latest/develop/reference/eviction/),
[Redis 7.2 configuration](https://github.com/redis/redis/blob/7.2/redis.conf).

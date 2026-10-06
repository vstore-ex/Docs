# Verified manual database recovery (A04)

Automatic deployment rollback restores container images only. It never restores a
database or downgrades its schema. A migration failure can leave the old image
incompatible with the current database; an image rollback does not prove schema
compatibility. Database recovery is a separate operator decision with a maintenance
window and backend-owner review of data loss and the intended application digest.

## Snapshot and evidence

The reusable deploy workflow captures the revision directly from PostgreSQL's
`public.alembic_version`, independently of application entrypoint output. Multiple
heads are recorded as sorted comma-separated IDs; an absent/empty version table is
`base`. Query failures stop the deployment. No noisy `alembic current` output is
used to construct a downgrade command.

Before applying migrations, the workflow creates a plain `pg_dump` compressed with
gzip, plus `<snapshot>.metadata.json`. The sidecar records SHA-256, revision,
PostgreSQL major version, source database and the verified pre-deploy backend
image digest. Unique temporary names prevent snapshot collisions. Unverified
snapshots are retained with an `.unverified` suffix outside normal retention. Both files must be retained and copied together. The dump's gzip CRC
is checked before the sidecar is written; a failed metadata step stops migration.
The database inside `DATABASE_URL` must match `POSTGRES_DB` in the PostgreSQL
container. Confirm this before using the snapshot. No external schema migrations
may run concurrently with deployment. Ordinary writes are consistent within the
`pg_dump` snapshot, but are not prevented by the deployment lock.

The checksum detects corruption, not malicious replacement. Accept only a trusted
snapshot: plain SQL can execute SQL and psql commands with the restoring role's
privileges. These snapshots do not include external roles, storage/media or Redis
state. Role ownership must remain compatible. Cross-major recovery, legacy dumps
without metadata, and `--create`/custom-format dumps require a separately reviewed
drill; do not invent the original revision from the live database after migration.

## Preparation and stopped writers

1. Declare maintenance mode, pause scheduled/manual deploys and coordinate access
   with the backend owner. Confirm the chosen snapshot's timestamp, expected data
   and Alembic revision, and the immutable backend image compatible with them.
   Record `DATABASE_URL`, `POSTGRES_DB`, the current image state and the original
   database name in secure incident evidence. Never paste credentials into Issues.
2. Preserve a fresh snapshot of the current database for the escape path before
   changing anything. Ensure disk capacity for the original database, new target,
   compressed snapshots and the helper's temporary uncompressed SQL file.
3. Acquire the shared deployment lock and keep it through stopping writers,
   recovery, verification and the controlled activation decision. Stop ingress,
   API and worker; suspend external scripts, cron jobs and admin SQL writers.
   The flag `--writers-stopped` explicitly acknowledges these external writers.
   The helper independently refuses running Compose backend/worker containers.

From the infrastructure checkout, after release-state adoption:

```bash
exec 200>.deploy.lock
flock -w 300 200
python3 scripts/release-state.py --lock-fd 200 compose stop proxy frontend admin backend worker
```

Check PostgreSQL sessions for the original database using `pg_stat_activity`.
Resolve every unexpected writer with its owner. Do not terminate sessions blindly.
The lock coordinates these deployment tools, not other database clients. Keep
writers stopped on any error; the helper never restarts them.

## Restore into a new target

Use a previously unused name such as `recovery_20261006_a04`. The helper requires a
`recovery_*` name, creates it from `template0`, and refuses every existing target,
even an empty one. The original database is retained. No table cleanup, DROP,
TRUNCATE or in-place restore is performed automatically.

Prepare a reviewed verification SQL file before restoration. Its assertions must
raise an error when expected data does not match; merely selecting a false value
still exits successfully. Use known critical records and totals from the chosen
snapshot, constraints and relevant business invariants. For example, the isolated
drill uses this fixture assertion; production checks must use real application
schema and reviewed expected values:

```sql
DO $$ BEGIN
  IF (SELECT count(*) FROM recovery_probe) <> 1 OR
     (SELECT value FROM recovery_probe WHERE id=1) IS DISTINCT FROM 'before' THEN
    RAISE EXCEPTION 'snapshot data does not match expected baseline';
  END IF;
END $$;
```

Run recovery while holding descriptor 200's lock:

```bash
python3 scripts/database-recovery.py --lock-fd 200 restore \
  backups/pre_deploy_YYYYMMDD_HHMMSS_UNIQUE.sql.gz \
  --target recovery_20261006_a04 \
  --verify-sql /secure/path/recovery-verification.sql \
  --writers-stopped
```

The helper checks the sidecar, checksum, gzip integrity, PostgreSQL major and
stopped writers before creating the target. It runs `psql -X -v ON_ERROR_STOP=1
--single-transaction -f -`. A SQL error returns nonzero and rolls back the restore
transaction. The newly created empty database may remain for inspection. A
revision or data verification failure can leave a populated target; it must not
be activated. All verification SQL runs inside a read-only transaction.

Success requires the recorded Alembic revision and operator-supplied data
assertions to pass. This does not prove that the application image is compatible
or that Redis/media state is consistent. Review those separately. The helper does
not change connection strings, images, release state, or writer status.

## Controlled activation and escape path

Only after backend-owner verification and approval, select the reviewed pinned
image using the existing release-state procedure. Never start an incompatible
image or run automatic migrations against the restored target. Confirm the
selected image's entrypoint migration behavior; the current backend defaults
`AUTO_MIGRATE` to false and worker explicitly disables it.

With writers still stopped, change the database part of both `DATABASE_URL` and
`POSTGRES_DB` to the verified target, preserving credentials. Recreate PostgreSQL
with the same volume to refresh its `POSTGRES_DB` environment; changing this value
does not rename an existing database or initialize a nonempty volume. Then start
API and proxies with the approved digests, leaving worker stopped. Before starting
edge, apply owner-approved maintenance access controls that keep public ingress
closed while allowing the loopback TLS probe. A maintenance notice alone does not
stop writes. Preserve the permanent admin-access policy. Under the
already-held lock, direct Compose commands read A01's persisted image override:

```bash
docker compose up -d --no-deps --wait --wait-timeout 150 postgres
docker compose up -d --no-deps --wait --wait-timeout 150 backend frontend admin proxy
python3 scripts/proxy-health.py recover
python3 scripts/database-recovery.py --lock-fd 200 revision
```

Check both HTTPS API routes, expected revision and application read-only behavior
before permitting writes. Worker jobs from before the snapshot may reference
post-snapshot records: the backend owner must reconcile pending/retrying jobs,
Redis auth state and external media before resuming them. No automatic Redis purge
or media deletion is part of recovery. Once the owner authorizes writes, start
worker, verify the pinned images, lift maintenance mode, and release the lock:

```bash
docker compose up -d --no-deps --wait --wait-timeout 150 worker
python3 scripts/release-state.py --lock-fd 200 verify
flock -u 200
exec 200>&-
```

If any check fails before writes resume, keep writers stopped. Restore the saved
connection settings and compatible image selection, refresh PostgreSQL's container
environment, and verify the retained original database before restarting clients.
This preserves the original data; it does not repair an incompatible old schema.
Alternatively remain in maintenance while preparing another fresh recovery target.
After new writes resume, switching back can lose those writes and requires a new
reconciliation decision. Retain both databases and all evidence until the owner
approves cleanup. Do not drop the source as part of routine recovery.

A deliberate Alembic downgrade is a separate, backend-reviewed operation. Review
all downgrade functions and potential data loss first, including multiple-head
handling. Snapshot the current state and stop writers. Never translate a captured
revision string into an unattended downgrade hint.

## Verification and references

`tests/smoke_database_recovery.py` runs a unique disposable PostgreSQL 17 stack. It
executes the actual workflow snapshot block under the inherited lock, verifies
clean revision output despite unrelated backend logs, modifies data and revision,
and restores the baseline into a new database. It proves refusal of active writers
and existing targets, SQL-error transaction rollback, revision/data/checksum
failures, retention of the source and stopped writers. This drill validates tooling,
not the application's full schema or a production recovery approval.

Tracking: [A04 issue](https://github.com/vstore-ex/vstore-infra/issues/37).
Related: [HTTPS API gate](upstream-recovery.md).
PostgreSQL 17 references: [SQL dump recovery](https://www.postgresql.org/docs/17/backup-dump.html),
[psql error handling and single transactions](https://www.postgresql.org/docs/17/app-psql.html).

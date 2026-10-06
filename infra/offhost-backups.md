# Weekly off-host backups and replacement recovery

Task: [A06](https://github.com/vstore-ex/vstore-infra/issues/40). Implementation:
`scripts/offhost-backup.py`, `config/backup/` and the replacement recovery CI job
in [vstore-infra](https://github.com/vstore-ex/vstore-infra).

## Owner policy and actual coverage

The owner selected a backup approximately once a week and a separate private
container in the existing `vstorestorage` account. The VM is normally off and
may be started briefly once a day. A startup after at least 168 hours without a
completed recovery point must create a backup. No formal RPO/RTO SLA was requested.
Four weekly recovery points, keeping at least the last two completed copies, are
a configurable proposal pending owner review. Writer downtime and production
installation also require the reviewed adoption decision.

The interval is wall-clock time, not accumulated VM uptime. While the VM is off,
no new recovery points can be produced. Consequently 168 hours is a scheduling
target, not an unconditional maximum data-loss window. An offline VM, failed
upload or expired credential can extend the age of the last usable copy. RTO is
unset; the synthetic drill duration below is not a production recovery guarantee.

Read-only baseline, 6 October 2026, subscription **Azure for Students**:

| State | Verified result |
| --- | --- |
| VM | `prod`, resource group `VSTORE-RG`; running during inspection |
| Recovery Services vaults | Empty list in the inspected subscription |
| Data Protection backup vaults | Empty list in the inspected subscription |
| Recovery Services provider | `NotRegistered`; `check-vm` fails `ProviderNotRegistered`, rather than confirming protection |
| Storage account | `vstorestorage`, Italy North, StorageV2 |
| Existing container | `media`, blob-public access; its application access policy is preserved |
| Blob/container soft delete | Enabled, 7 days |
| Blob versioning/PITR | Not enabled in the returned configuration |
| Off-host backup container | Not present at inspection; `vstore-backups` is the proposed private target |
| Application media | 379 blobs, 120,493,520 bytes, counted through the existing backend credentials without printing them |
| VM local files | `storage/` about 8 KiB; `backups/` about 48 KiB; no VStore backup timer in the inspected system timers |
| Deployed infrastructure | Commit `02f3f77faa9ad57e7573801c0552fb3a4210ed4e` |

These checks do not establish protection in other subscriptions or third-party
systems. Local migration snapshots remain useful but do not survive VM/disk loss.
Soft delete is not a consistent database backup. The new encrypted repository
protects against VM loss; using the same account/region for media and backups
does not cover deletion of the storage account or a regional outage.

## Scheduling and completion

The optional system timer checks one minute after boot, every 15 minutes after
the previous service run, and hourly with persistent missed-run handling. The
service waits for Docker/network ordering and gives application images time to
become healthy before stopping writers. Every check reads completed snapshots
from the off-host repository. There is no VM-local success timestamp.

- No completed snapshot, or age at least 168 hours: create one recovery point.
- A completed snapshot younger than 168 hours: skip without stopping writers.
- VM off for a week: the next boot attempts a new copy once the stack is ready.
- Authentication/network/upload failure: no completed recovery point is recorded;
  the next timer check or next boot retries.
- Partial upload or process loss: the `vstore-a06-pending` snapshot cannot count
  as protection. Only successful uploads promoted to `vstore-a06` count.

Offline dates are not fabricated into historical backups. A successful catch-up
copy starts the next 168-hour interval from its recorded recovery point.

## Required state and consistency

Capture holds the same `.deploy.lock` as deployments. The permission helper
checks only root/lock before acquiring it, then checks secrets/snapshots under
the lock. An approved maintenance window stops backend, worker and certificate
renewal, forces a fresh Redis RDB and stops Redis. External database/media writers
must also be excluded by the operator's maintenance policy.

The encrypted bundle contains:

- A logical dump of the database selected by `DATABASE_URL`, its checksum,
  PostgreSQL major, Alembic revision and verified backend digest.
- PostgreSQL globals, including roles, grants and password hashes, for reviewed
  replacement provisioning.
- The committed infrastructure archive, private `.env` and verified current/
  previous application/edge release state; PostgreSQL and Redis image digests.
- Local storage with ownership/modes, upload chunks, the complete Redis
  persistence directory, certificates with safe relative links and ACME webroot.
- Azure `media` blob bytes, names, checksums, metadata and content headers. Blob
  names are stored as hashes, not interpreted as local paths. ETag checks and a
  second inventory reject changes during capture.

Changed writable mounts, external Celery broker/result URLs and a Redis
persistence directory outside `/data` require strategy review; the backup refuses
to silently omit them. The database dump covers the API's selected database,
not unrelated databases on the same server.

Existing containers resume before upload. Backup never recreates images or runs
migrations. A private, atomically published resume journal records exact container
IDs before stopping services. Termination attempts resume; after abrupt VM loss,
the next run resumes before contacting Azure. `ExecStopPost` also calls resume
without needing repository credentials. A changed container ID requires manual
inspection. Deployments and manual release mutations refuse a pending journal.

Plaintext staging is temporary under private `backups/`; secrets/files use 0600,
directories 0700. Docker/root/deployment users remain trusted. Restic encrypts
repository data; recoverability requires its password outside the VM.

## Reviewed production adoption

Deployment does not enable the timer or create cloud resources. Before adoption,
approve the private container and its storage/operation charges, the maintenance
window, retention and credential escrow. Use a reviewed merged infrastructure
commit; check that existing images have been adopted with immutable digests.
The source stack must be healthy and tracked files clean.

1. Create only the dedicated private container, preserving `media` access. An
   authorized operator can use the management-plane command below. It fails if
   the target already exists; inspect an existing target before reusing it.

   ```bash
   az storage container-rm create --resource-group vstore-rg \
     --storage-account vstorestorage --name vstore-backups \
     --public-access off --fail-on-exist
   ```

2. Provision a HTTPS-only, container-scoped SAS. Keep the account key off the VM.
   Normal restic locking and tag promotion need read/create/write/list/delete
   permissions. This writer is not an append-only identity; compromised VM
   credentials can delete repository objects. Existing soft delete is a limited
   safety net. Do not apply immutable retention or object lifecycle deletion to
   restic internals without a separately tested design.

   Generate the SAS on a trusted operator machine, redirecting output to a private
   file. Supply a reviewed UTC expiry; renew before that date. Key-mode CLI queries
   the account key internally when authorized; it need not appear in arguments.
   Do not use tracing or `--debug` around credentials.

   ```bash
   set +x
   umask 077
   az storage container generate-sas --account-name vstorestorage \
     --name vstore-backups --auth-mode key --permissions racwdl \
     --https-only --expiry '<reviewed UTC expiry>' -o tsv > repository.sas
   ```

3. Generate the repository password privately and escrow password, configuration,
   recovery storage access and GHCR access outside the VM. Confirm retrieval from
   that escrow before enabling backups. Do not store the only password/SAS copy
   in `.env`, the repository itself or a GitHub Issue/PR. A read-only recovery SAS
   may require `restic --no-lock`; the supplied recovery CLI uses normal locks.

4. On the deployment account, install Python 3.10+, Compose v2 and the verified
   backup tools, keeping their configuration outside the clone:

   ```bash
   umask 077
   mkdir -p ~/.local/bin ~/.local/share/vstore-backup ~/.config/vstore-backup
   chmod 700 ~/.config/vstore-backup
   cd ~/vstore
   python3 scripts/install-restic.py --directory ~/.local/bin
   python3 -m venv ~/.local/share/vstore-backup/venv
   ~/.local/share/vstore-backup/venv/bin/pip install -r scripts/requirements-backup.txt
   install -m 600 config/backup/config.example.json ~/.config/vstore-backup/config.json
   ```

   The installer pins restic 0.19.1 and verifies official binary SHA256 checksums.
   It refuses to overwrite an existing binary. Transfer the escrowed password
   and SAS through a trusted private channel, mode 0600, without printing them.
   Review paths, host tag, repository prefix and retention in the config.
   `allow_quiesce` defaults false; set it true only after approving maintenance.
   Production rejects local repositories. The local-repository switch is for
   disposable drills only.

5. Initialize the dedicated repository once, on the trusted operator account
   using the private SAS and password-file environment. Restic does not initialize
   it automatically during backup. Do not run `init` against application media.

   ```bash
   set +x
   export RESTIC_REPOSITORY=azure:vstore-backups:/prod
   export RESTIC_PASSWORD_FILE=/private/offvm-escrow/password
   export AZURE_ACCOUNT_NAME=vstorestorage
   export AZURE_ACCOUNT_SAS="$(cat /private/offvm-escrow/repository.sas)"
   restic init
   unset AZURE_ACCOUNT_SAS
   ```

6. Run the first backup manually with the exact service account/environment. Use
   the venv Python and a PATH including `~/.local/bin` and Docker:

   ```bash
   cd ~/vstore
   export PATH="$HOME/.local/bin:$PATH"
   ~/.local/share/vstore-backup/venv/bin/python scripts/offhost-backup.py \
     --config ~/.config/vstore-backup/config.json backup --if-due
   ~/.local/share/vstore-backup/venv/bin/python scripts/offhost-backup.py \
     --config ~/.config/vstore-backup/config.json status
   ```

   Confirm that the completed full snapshot ID is present in Azure, run a restic
   repository check and an isolated download/restore using the off-VM escrow.
   This is the production Azure-backend adoption check; the CI fixture does not
   replace it. `status` exits 2 when no completed copy exists or it is overdue.

7. Review the unit's deployment user, paths and Docker group. Then an administrator
   installs both reviewed unit files and enables the timer:

   ```bash
   sudo install -m 644 config/backup/vstore-backup.service /etc/systemd/system/
   sudo install -m 644 config/backup/vstore-backup.timer /etc/systemd/system/
   sudo systemctl daemon-reload
   sudo systemctl enable --now vstore-backup.timer
   systemctl list-timers vstore-backup.timer
   journalctl -u vstore-backup.service --no-pager
   ```

Do not promise zero downtime: consistent capture stops writers while files/media
are exported. Measure the first production capture separately. Budget review
must include growth, four weekly copies, changed media, pending copies, soft
delete retention, reads/writes and any cross-region recovery egress. Incremental
deduplication may reduce usage; do not assume a fixed bill from this baseline.

## Retention and failures

Retention is manual and defaults to dry-run. Use a trusted maintenance account
with the same repository/host and reviewed retention settings:

```bash
python scripts/offhost-backup.py --config /private/backup-config.json retention
# After reviewing exactly which completed copies are removed:
python scripts/offhost-backup.py --config /private/backup-config.json retention --apply
```

The policy selects completed copies for this host/tag, keeps four calendar-week
representatives and at least the latest two, then prunes unreferenced data only
with `--apply`. A missed week has no representative; this is not necessarily
exactly four snapshots. Azure soft-deleted objects can continue to incur costs.

Pending snapshots are excluded and can accumulate after failed/interrupted runs.
Inspect them separately with `restic snapshots --host vstore-prod --tag
vstore-a06-pending`; verify no active upload uses them before explicitly forgetting
reviewed IDs and pruning. Never delete raw repository objects or apply an Azure
lifecycle rule based on their ages. Never force-unlock a repository until all
backup/maintenance clients are confirmed stopped.

If interrupted capture leaves services stopped, run `offhost-backup.py resume`
from the deployment directory with the deployment account. It uses the shared
lock and exact saved IDs; no config argument or cloud access is required. Do not
remove the journal merely to bypass a failed resume/deploy gate. Inspect changed
containers and recover them explicitly.

To stop scheduling, disable the timer and stop the service; its post-stop action
attempts resume. Confirm writers recovered. Keep existing encrypted copies and
escrow; disabling scheduling does not delete cloud data.

## Replacement recovery without production activation

Use a separate host/project/network and isolated volumes, without production
DNS/ports or writer activation. Retrieve tools and credentials from off-VM escrow.
Choose an explicit completed full 64-hex snapshot ID for the correct host/tag.

```bash
python scripts/offhost-backup.py --config /private/backup-config.json restore \
  --snapshot '<full completed snapshot ID>' --target /private/new-recovery-download
```

The target must not exist. Restic verifies downloaded files; the helper verifies
bundle checksums and rejects archive traversal, special files and link pivots.
Keep the downloaded bundle private. Then:

1. Review and extract `infra.tar.gz` into a new private replacement clone. Restore
   `environment` as `.env` and `release.json` as `docker-compose.override.yml`,
   mode 0600. Pin PostgreSQL/Redis to the recorded immutable images. Isolate Compose
   ports/networks and use a unique `recovery-*` project. Do not run the full stack.
2. Restore local storage into a new directory with `restore-storage --bundle
   <bundle> --kind storage --target <new path>`. Restore named state with the same
   command, `--volume`, and new names such as `recovery-drill_redis_data`; kinds
   are `redis`, `uploads`, `certificates` and `webroot`. Existing targets are
   refused. Ownership and certificate links are preserved.
3. Start only PostgreSQL/Redis on the replacement stack. Review the globals dump:
   remove only reviewed conflicts with the freshly initialized role and provision
   needed owners/grants/settings before restoring data. Tablespaces, extra roles
   and credentials require replacement-specific review. Never blindly apply
   globals to an existing production server.
4. Restore `database.sql.gz` using the [A04 recovery guide](https://github.com/vstore-ex/Docs/pull/9)
   into a new `recovery_*` database. Supply reviewed read-only assertions for
   application data. The helper checks the recorded PostgreSQL major via the
   maintenance database, the checksum, revision and data; it does not require the
   original database to exist on the replacement server.
5. Restore Blob media using `restore-media --bundle <bundle> --target
   recovery-<unique name> --connection-file <private replacement connection file>`.
   The helper creates a new private container and verifies each uploaded checksum;
   existing containers are refused. Restoring media under a new container/account
   may require backend-owner review of persisted URLs and application configuration.
6. Verify counts, critical records, Alembic revisions, PostgreSQL role grants,
   local/media hashes, Redis state/TTLs and certificate links. Record recovery
   point age and total replacement duration before deciding to activate.
7. Rotate `SECRET_KEY` before recovery activation: restored Redis state lacks any
   revocations after the recovery point, so retaining the old signing key could
   revive previously revoked tokens. Reconcile Celery queued/reserved/in-flight
   jobs with the backend owner; validate idempotency before starting the worker.
   These decisions are part of the [A07 Redis policy](https://github.com/vstore-ex/vstore-infra/issues/41)
   handoff, not an automatic schema/business-logic change.
8. Activate the reviewed database/media endpoints, release images and routing
   only after operator/backend approval. Check app/proxy health, then allow
   writers. Recovery commands never perform this cutover themselves.

## Verification evidence

CI runs safety regressions and a disposable replacement drill with real
PostgreSQL 17, Redis 7.2, restic 0.19.1 and Azurite 3.35.0. It creates data/revisions,
an additional database role/grant, Redis auth state, local files, upload chunks,
certificate links and Blob bytes/metadata. It backs up, verifies a fresh copy is
skipped, simulates interrupted capture, deletes source containers/volumes/local
state/live media, and recovers only from the encrypted repository plus off-host
credential escrow. Existing restore targets are refused. No production data is
restored or deleted.

The repository in this drill is a separate local encrypted fixture; Azure SDK
media operations run against Azurite. Actual restic Azure repository upload and
production-sized recovery remain adoption checks. Each CI run publishes
`a06-replacement-recovery` with recovery point, capture downtime and measured
restore duration. Record the first production adoption result on A06 before
claiming installed coverage or closing that operational acceptance item.

Local drill completed successfully on 7 October 2026 (Europe/Kiev), with recovery
point `2026-10-06T22:07:29.186763+00:00`: capture/resume 12.945 seconds and
replacement download/data/storage verification 14.069 seconds. These measurements
use one small test table and two small blobs, not the production media volume.

References: [restic repository encryption and Azure backend](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html),
[backup completion](https://restic.readthedocs.io/en/stable/040_backup.html),
[retention](https://restic.readthedocs.io/en/stable/060_forget.html),
[Azure private container provisioning](https://learn.microsoft.com/en-us/cli/azure/storage/container-rm?view=azure-cli-latest),
[container SAS](https://learn.microsoft.com/en-us/cli/azure/storage/container?view=azure-cli-latest#az-storage-container-generate-sas).

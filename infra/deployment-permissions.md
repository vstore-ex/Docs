# Private deployment directory, secrets and snapshots

Task: [A16](https://github.com/vstore-ex/vstore-infra/issues/39).
The deployment account owns the deployment clone and its private state. App
storage keeps its existing application ownership and mount permissions.

## Required permissions

| Path | Owner | Mode |
| --- | --- | --- |
| Deployment directory, normally `~/vstore` | SSH deployment user | `0700` |
| `.env`, `.deploy.lock`, `docker-compose.override.yml` | Deployment user | `0600` |
| `backups/` and nested directories | Deployment user | `0700` |
| Retained snapshots, sidecars and unverified backup evidence | Deployment user | `0600` |
| `storage/` and application files | Existing application owner, normally UID10001 | Preserve existing mount permissions |

The private parent prevents unrelated local users from traversing the clone or
creating entries. Docker's privileged daemon resolves the bind source; app
containers receive `storage/` directly and do not traverse the host deployment
parent. Keep UID10001's required read/write permissions within its mount. Do not
run a recursive chmod/chown over the deployment tree or app storage. Tracked nonsecret Nginx mount directories/files keep readable modes (`0755`
directories and entrypoint scripts, `0644` templates). Infra Git sync runs in a
subshell with `umask 022` so checkout does not create owner-only files that a
proxy without DAC_OVERRIDE cannot read. The private deployment parent still
blocks unrelated local users. Docker named
volumes for PostgreSQL/Redis and Certbot mounts are outside this helper's scope.

Root, Docker-daemon administrators and the deployment account remain trusted.
These file modes do not isolate secrets from a user with Docker/root access or
revoke already-open file descriptors. Do not give untrusted users Docker access.

## Initial adoption and provisioning

Pause deployments while checking the existing directory. Use a trusted copy of
the reviewed helper: a clone that was world-writable is not independently trusted
until its contents and ownership have been checked. Confirm the SSH account owns
the deployment directory, `.env`, release state, lock and retained backups. The
account's home/parent must not permit unrelated users to replace the deployment
directory. If ownership is wrong, an administrator should correct only the
reviewed private paths; preserve application-owned storage.

With the reviewed helper and correct ownership, run as the deployment user:

```bash
cd ~/vstore
python3 scripts/secure-deployment.py
python3 scripts/secure-deployment.py --check
bash scripts/compose.sh verify
```

The helper applies private modes, creates `backups/` privately if absent and
corrects all retained backup files, including sidecars and unverified snapshots.
It never reads or prints secret contents and never changes storage ownership or
permissions. It refuses symlinks, hardlinked protected files, special files and
paths owned by another UID. Failure stops deployment; do not weaken permissions
to bypass it. Review unexpected paths and ownership before retrying.

For an initial install, create the owner-only deployment directory, place `.env`
there with mode0600 and provision storage for UID10001 separately. Run the helper
before starting the stack. Keep the reviewed immutable image references and
certificate adoption procedure documented in the infrastructure README and
[edge guide](edge-security.md).

Both SSH deployment workflows use `umask 077` before creating locks, temporary
state or snapshots. They run a permission preflight before opening `.deploy.lock`
so an unsafe lock link cannot redirect shell truncation. After acquiring the
shared lock, they recheck/repair retained files before release adoption/convergence.
Infra CD also applies the synced helper after the reviewed config update.
Install the trusted helper in the host clone before merging this infrastructure
change or allowing updated reusable app workflows to run. A host without the
helper fails closed before locking or syncing; it cannot bootstrap itself through
this updated workflow. This initial installation is an operator adoption step.

If the helper encounters an unsafe backup path after securing earlier paths,
those restrictions remain in place. It does not restore insecure modes on error.
The `.env` file is required; a missing file is a failed provisioning preflight.

## Verification without disclosing secrets

Use the helper's `--check` mode and `stat` for owner/mode checks. Do not print
`.env`, resolved Compose configuration or snapshot contents in logs/tickets.
An administrator can independently test an unrelated UID using file-access
checks, without reading secret values:

```bash
sudo -u nobody test ! -r "$PWD/.env"
sudo -u nobody test ! -r "$PWD/backups/<retained-snapshot>.sql.gz"
sudo -u nobody test ! -w "$PWD"
```

Choose an existing snapshot for the placeholder; a missing file would make a
readability check meaningless. Check effective permissions on the deployment
account and mounted app paths as well. Root/Docker access is an exception to
ordinary file permissions, not the unrelated-UID test target.

Local and CI verification:

```bash
python3 -m unittest discover -s tests -p 'test_deployment_permissions.py' -v
python3 tests/smoke_deployment_permissions.py
python3 tests/smoke_database_recovery.py
python3 tests/smoke_release_state.py
```

The Linux smoke recreates the original weak permissions in a disposable Docker
volume, demonstrates access as UID65534, then proves actual read/write operations
fail after hardening. UID10002 can still create private snapshots. A separate
hardened container with UID10001 writes through a real Docker bind mount below
the private deployment parent. Proxy UID0 without DAC_OVERRIDE can still read
and execute the separate nonsecret config mounts. No production paths or secrets
are used.

The PostgreSQL smoke executes the actual workflow snapshot block, validates
private snapshot/sidecar modes and verifies recovery. The release smoke executes
both SSH workflow scripts and exercises successful deployment plus rollback.

## Recovery and rollback

Keep the private modes when rolling back code or an image. Permission tightening
does not alter data, release history or container ownership. If deployment fails
because a protected path has the wrong owner, correct that specific ownership
with administrator review and retry the helper. Preserve unexpected links or
files for investigation outside the live deployment paths; do not follow them
with recursive chmod. Repairing permissions does not repair an independently
compromised deployment clone or account.

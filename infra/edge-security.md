# Edge image security gate and release adoption

Implementation: [vstore-infra A05](https://github.com/vstore-ex/vstore-infra/issues/38).
The production edge is `ghcr.io/vstore-ex/vstore-infra-proxy@sha256:<digest>`.
Compose consumes this image through `PROXY_IMAGE`; the host never builds it.

## Build, scan and publication

Infra CI runs on pull requests, main pushes and manual dispatch. It pulls the
current official `nginx:stable-alpine` base, upgrades Alpine runtime packages and
installs OpenSSL. Buildx disables cache for the named `runtime` stage. Both base
refresh and package-install refresh are required; a cached install must not hide
available fixes. The artifact targets `linux/amd64`, matching the production host.

CI loads one artifact, records its Docker image ID and tests that exact ID with
an empty certificate directory. The hardened bootstrap preserves the required
capabilities, official template expansion, OpenSSL certificate generation,
Nginx syntax, HTTP health and real TLS handshakes for both hostnames. Certificate issuance ordering and the HTTPS API
recovery gates have separate regression tests.

Trivy v0.75.0 scans the recorded image ID with vulnerability scanning only,
`HIGH,CRITICAL`, `ignore-unfixed` and exit code 1. An actionable finding blocks
publication and deployment. Findings without an available fix are excluded by
this policy; passing the gate does not mean every vulnerability is absent.
There are no A05 exceptions or new vulnerability ignore files. Any future
exception needs a reviewed justification, owner and expiry before changing the
policy. Fix or update the image when the gate fails; do not bypass the scan.

The JSON report is retained in the run's `proxy-trivy-<commit>` artifact for 30
days, including failed scans when a report exists. The same image ID is then
pushed, pulled and bootstrapped through a disposable loopback registry. Main
pushes and dispatches from `main` additionally publish to GHCR. Pull requests
never publish to GHCR or deploy production.

Publication tags the recorded image ID with `sha-<full commit>`, without a second
build. Guards reject a changed local scan tag or mismatched published identity.
The workflow selects the GHCR repository's digest from `RepoDigests`, verifies
its local image identity and passes only `name@sha256:<digest>` to SSH deployment.
Mutable tags are lookup aids and must not be used as production release pins.

## Host prerequisites and existing-host adoption

The host needs Python 3, Bash, Linux `flock`, Docker and Compose v2. Authenticate
the host to GHCR with permission to read the new proxy package, as well as the
existing app packages. CI needs package publication permission. Missing package
access must fail deployment rather than trigger a local rebuild.

Review and merge the documentation before the infrastructure change. Keep the
updated release helper compatible with the reusable app workflows on the host.
The helper still reads app-only version 1 state. It records version 2 once a
successful edge registry digest is adopted; application history is preserved.
Older helpers that only understand version 1 cannot read this state. Restore a
compatible helper before manual recovery; do not downgrade or hand-edit the
state to make an older helper accept it.

Under the shared `.deploy.lock`, Infra CD loads successful app digests and
remembers the current edge digest. On the first adoption from version 1, no edge
registry pin exists: CD captures the healthy running proxy's exact local image
ID for failure recovery. A stale `PROXY_IMAGE` environment value is not a rollback
target. The legacy image is retained for this first attempt; CD does not rebuild
it or claim it passed the new gate.

CD explicitly pulls the newly scanned digest and applies it with a temporary
Compose image override and `--no-build`. It recreates the proxy to render the
bind-mounted templates, reloads dependent app proxies and requires both HTTPS
API routes to pass their certificate/response policy. It verifies all running
release images before atomically recording `PROXY_IMAGE` in successful state.
The successful override supersedes stale `.env` and exported image pins in
future sessions and certificate bootstrap.

If pull, convergence, HTTPS API verification or state publication fails, CD
restores the previous config and exact edge artifact. It recreates the edge
with the restored templates and rechecks recovery. Failure leaves successful
release history unchanged. During first adoption the fallback is the retained
legacy local image; after adoption it is the saved immutable registry digest.
The first successful adoption has no previous edge registry digest. A later
successful upgrade establishes the first explicit edge rollback target.

Infra CD does not perform an application image upgrade or database restore.
Merging the reviewed infrastructure PR triggers main-run publication and CD.
The local/PR tests do not execute production deployment. If the host is offline,
CI deployment fails; dispatch Infra CI from `main` after it returns. A fresh
run rebuilds and scans against the current base and vulnerability database.

## First installation

Obtain the edge digest from the successful main-run publication output or GHCR
package version for that run. Confirm its build/scan/smoke evidence before use.
Review all four immutable references (`BACKEND_IMAGE`, `FRONTEND_IMAGE`,
`ADMIN_IMAGE`, `PROXY_IMAGE`) and configure the host's `.env` and storage.
There is no published `latest` edge release; the template's tag is a placeholder.
Export the exact reviewed edge digest before running the following commands:

```bash
export PROXY_IMAGE='ghcr.io/vstore-ex/vstore-infra-proxy@sha256:<reviewed digest>'
docker compose pull
docker compose up -d --wait
bash scripts/compose.sh bootstrap --proxy-image "$PROXY_IMAGE"
bash scripts/compose.sh verify
```

Replace the digest placeholder; do not execute it literally. Bootstrap refuses
to overwrite existing state and requires the running edge to match this digest.
For existing version 1 state, use the verified Infra CI adoption path instead.
Complete public certificate issuance with `scripts/init-letsencrypt.sh` under
its shared lock. Follow the infrastructure README's rule about unmanaged dummy
certificate directories before switching a public hostname. Temporary fallback
certificates support startup; they do not satisfy the public HTTPS PKI gate.

## Manual recovery and edge rollback

Use the manual wrapper so deployment mutations share the same lock:

```bash
bash scripts/compose.sh up -d --wait
bash scripts/compose.sh recover
bash scripts/compose.sh verify
bash scripts/compose.sh rollback PROXY_IMAGE
```

Rollback requires a recorded previous edge registry digest. It pulls that digest,
starts the edge, runs HTTPS API recovery and verifies the running images before
swapping current/previous state. Failed verification restores the current image
and leaves history unchanged. This command restores an image with the current
config; it does not reset Git or downgrade the database. Review compatibility
before selecting an earlier artifact. Infra CD failure recovery additionally
restores its previous Git config and recreates the edge to render templates.

Preserve the host-local `docker-compose.override.yml` and its previous history.
Do not commit it, replace it with tag guesses or delete it to force adoption.
For damaged state, follow the infrastructure release-state recovery procedure
with a compatible helper and independently verified images.

## Verification

```bash
python3 -m unittest discover -s tests -p 'test_*.py' -v
docker compose --env-file .env.example config --quiet
# The following commands use the same freshly built image.
docker build --platform linux/amd64 --pull --no-cache -t edge-check config/nginx
trivy image --scanners vuln --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 edge-check
bash tests/smoke-proxy.sh edge-check
python3 tests/smoke_edge_artifact.py edge-check
python3 tests/smoke_release_state.py
```

The registry test publishes only to a disposable loopback registry, pulls the
immutable digest, compares image identity and runs bootstrap on that reference.
The release smoke executes the actual SSH deployment script in a disposable
stack. It covers first-adoption failure, stale pins, template rendering, edge
rollback and config/artifact failure recovery without a server build.
These tests do not change production.

Source references: [official Nginx image](https://hub.docker.com/_/nginx),
[Trivy container image scanning](https://trivy.dev/docs/latest/target/container_image/).

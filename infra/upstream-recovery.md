# Upstream recovery and deployment API gates (A02)

Nginx resolves literal upstream names when its configuration loads. A recreated
backend, storefront or admin container can leave an existing proxy using an old
Docker address. Static `/healthz` probes do not detect this failure.

Infrastructure owns the controlled reload strategy; application routing stays in
its existing image. After successful container convergence, the deployment reloads
frontend and admin Nginx, then edge Nginx. All three configurations must pass
`nginx -t` before any reload is signalled. The subsequent HTTPS gate waits for the
new workers and exercises `/api/v1/games/?limit=1` through both storefront and admin.
Both responses must be HTTP 200, `application/json`, and a JSON array. This public
catalog request also exercises backend database access without changing data.

The reusable app deployment and its rollback, infrastructure convergence and its
config rollback, and manual `scripts/compose.sh up`, `start`, `restart` and image
rollback all use this gate. A failed gate prevents publishing a successful image
in release state and causes automated deployment rollback to fail visibly if the
restored route is still unavailable. Existing direct-service health checks remain.

## TLS and local admin policy

The probe connects to the edge's published TLS port on loopback, sends the actual
Host and SNI, and avoids external DNS, hairpin networking and third-party services.
Public domain certificates use Python's normal system CA and hostname checks;
expired, untrusted or mismatched certificates fail deployment.

For configured `.local`, `.localhost` and `.test` hosts, the probe trusts the
certificate mounted in edge's configured certificate directory. Hostname checking
is disabled only for these local names, preserving the existing policy where a
local admin name can reuse the storefront dummy certificate. TLS chain validity
is still checked. No private key is read. This does not make the fallback dummy
certificate suitable for public production domains.

## Operations

Apply the infrastructure change before app deployments use the updated reusable
workflow. Keep release-state adoption and the deployment lock from A01. Fixed
proxy addresses and the explicit Uvicorn trust list from A03 remain required;
change those together if proxy addresses change.

Use the lock-aware wrapper for manual operations:

```bash
bash scripts/compose.sh up -d --wait --wait-timeout 150
bash scripts/compose.sh recover
```

`recover` reloads all dependent proxies and checks both HTTPS API paths. Plain
`docker compose` still reads persisted release image pins, but does not reload
proxies or run the API gate. After an out-of-band container recreation, run the
wrapper's `recover` command. Stopping services deliberately does not run the gate.
Partial stack starts cannot pass the full-stack API gate.

The default gate uses 15 attempts, a 5-second socket timeout and a 3-second delay.
Configuration errors and missing containers/certificates fail immediately.
Reloads preserve existing location rewrites, media routes, TLS and WebSocket headers.
Existing workers finish active connections; long-lived upgraded connections can
keep old workers alive until their existing proxy timeout closes them.

## Verification

`tests/smoke_upstream_recovery.py` uses an isolated Compose project, actual edge
configuration and provenance-checked application Nginx snapshots. It forces each
backend/frontend/admin address to change, observes API failure while edge health
remains green, and runs the production recovery helper to restore both TLS paths.
It checks rewritten API routes, static media and admin WebSocket upgrade forwarding. The existing storefront
proxy does not forward Upgrade headers; this change preserves that baseline.
A stopped backend must fail recovery despite healthy static probes.

`tests/smoke_release_state.py` executes the actual SSH workflow shell scripts in a
disposable release fixture. Its injected API-gate failure proves the candidate
image is rolled back and successful release history remains unchanged. This
fixture isolates image state; real TLS/API behavior is covered by the upstream test.
Unit tests reject HTML fallbacks, non-array/invalid JSON, HTTP failures and failure
of either TLS route.

Tracking: [A02 issue](https://github.com/vstore-ex/vstore-infra/issues/36).
Related: [trusted client identity](client-identity.md).

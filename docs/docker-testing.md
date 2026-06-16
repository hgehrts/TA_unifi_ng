# Docker E2E test — Splunk + TA UniFi NG

**Date:** 2026-06-04  
**Lab directory:** `~/Projekte/TA/docker-splunk-unifi-test/`  
**TA package:** `~/Projekte/TA/TA_unifi_ng-1.3.0.tar.gz` (source: `~/Projekte/TA/unifi-ta-ng/TA_unifi_ng/`)

## Purpose

Validate the classic add-on **TA_unifi_ng** v1.3.0 inside **Splunk Enterprise in Docker** (`splunk/splunk:latest`, `linux/amd64`), using a modular input stanza `[unifi_ingest://docker_test]` and index `unifi`.

## Prerequisites

- Docker Desktop (or compatible) on the host
- UniFi controller reachable from the container network (lab: `https://192.168.1.1`)
- Valid **UniFi Integration API** key (Settings → Control Plane → Integrations → API Keys)
- Splunk Web: `http://localhost:8000` — user `admin`, password set in your docker-compose (replace `<SPLUNK_ADMIN_PASSWORD>` below)

## Critical rules (lab)

| Rule | Detail |
|------|--------|
| Input stanza | **`[unifi_ingest://docker_test]`** only — never `[script://./bin/unifi_ingest.py]` |
| Splunk CLI | Always **`-u splunk`** for `/opt/splunk/etc` and `/opt/splunk/var` |
| Booleans in `.conf` | `0` / `1`, not `true` / `false` |
| Restart | Prefer **`docker restart splunk-unifi-test`** over in-container `splunk restart` |
| Validate script | **`splunk cmd python .../bin/unifi_ingest.py --scheme --validate-arguments`** (not bare `python3`) |
| Reserved arg | Use **`controller_url`**, not `host` (Splunk internal argument) |

## 1. Start Splunk container

```bash
cd ~/Projekte/TA/docker-splunk-unifi-test
docker compose up -d
docker ps --filter name=splunk-unifi-test
```

Wait until healthcheck passes (Web UI on port 8000).

## 2. Install TA and index (as `splunk` user)

Copy artifacts into the container:

```bash
docker cp ~/Projekte/TA/TA_unifi_ng-1.3.0.tar.gz splunk-unifi-test:/tmp/
docker cp ~/Projekte/TA/docker-splunk-unifi-test/inputs.local.conf splunk-unifi-test:/tmp/inputs.local.conf
```

Install inside the container (**do not use root**):

```bash
docker exec -u splunk splunk-unifi-test bash -c '
  set -e
  cd /opt/splunk/etc/apps
  rm -rf TA_unifi_ng
  mkdir -p TA_unifi_ng
  tar -xzf /tmp/TA_unifi_ng-1.3.0.tar.gz -C TA_unifi_ng
  chmod +x TA_unifi_ng/bin/*.py
  mkdir -p TA_unifi_ng/local
  cp /tmp/inputs.local.conf TA_unifi_ng/local/inputs.conf
  /opt/splunk/bin/splunk add index unifi -auth admin:<SPLUNK_ADMIN_PASSWORD> || true
'
```

Enable the app (optional but recommended):

```bash
docker exec -u splunk splunk-unifi-test \
  /opt/splunk/bin/splunk enable app TA_unifi_ng -auth admin:<SPLUNK_ADMIN_PASSWORD>
```

Restart the container (preferred on Apple Silicon / lab images):

```bash
docker restart splunk-unifi-test
# wait ~60–120 s for splunkd + first modular input run (interval=60)
```

## 3. Validate modular input

Scheme and arguments:

```bash
docker exec -u splunk splunk-unifi-test \
  /opt/splunk/bin/splunk cmd python \
  /opt/splunk/etc/apps/TA_unifi_ng/bin/unifi_ingest.py \
  --scheme --validate-arguments -auth admin:<SPLUNK_ADMIN_PASSWORD>
```

Expect XML scheme with `controller_url` (no `host`).

Check **splunkd** (no `host` / init errors):

```bash
docker exec -u splunk splunk-unifi-test bash -c \
  'grep -E "ModularInputs.*unifi|Endpoint argument.*host|Unable to initialize.*unifi" \
   /opt/splunk/var/log/splunk/splunkd.log || echo "OK: no matching errors"'
```

Expect lines like:

```text
Introspection setup completed for scheme "unifi_ingest".
New scheduled exec process: .../bin/unifi_ingest.py
```

## 4. Confirm indexing

```bash
docker exec -u splunk splunk-unifi-test \
  /opt/splunk/bin/splunk search 'index=unifi | stats count by sourcetype' \
  -auth admin:<SPLUNK_ADMIN_PASSWORD>
```

Example successful output (multiple 60s poll cycles accumulate):

| sourcetype | count (example) |
|------------|-----------------|
| unifi:site | 3 |
| unifi:device | 39 |
| unifi:client | 207 |
| unifi:network | 27 |
| **Total** | **276** |

Per-cycle volume (single successful poll) is approximately **~90 events** (1 site + ~13 devices + ~68–69 clients + ~9 networks), matching host dry-run.

Recent-window check:

```bash
docker exec -u splunk splunk-unifi-test \
  /opt/splunk/bin/splunk search 'index=unifi earliest=-5m | stats count by sourcetype' \
  -auth admin:<SPLUNK_ADMIN_PASSWORD>
```

## 5. Lab input stanza template

File: `docker-splunk-unifi-test/inputs.local.conf` (secrets stay local; do not commit API keys):

```ini
[unifi_ingest://docker_test]
controller_url = https://192.168.1.1
interval = 60
index = unifi
page_size = 200
verify_ssl = 0
include_devices = 1
include_clients = 1
include_networks = 1
api_key = <integration-api-key>
disabled = 0
```

`controller_url` may be hostname-only (`192.168.1.1`); the ingest script prepends `https://` if no scheme is present. Explicit `https://` is still recommended.

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `ModuleNotFoundError: splunklib` / `isModInput=no` | Wrong stanza type `[script://...]` → use `[unifi_ingest://name]` |
| `Endpoint argument "host" is an internal argument` | Old TA or inputs — use tarball v1.3.0+ with `controller_url` |
| Empty `users.ini`, metadata errors after install | Ran Splunk CLI as **root** on etc/var → `docker compose down -v`, reinstall as **splunk** only |
| `splunk install app` fails | Manual `tar` extract to `/opt/splunk/etc/apps/TA_unifi_ng` as above |
| `splunk enable app` “does not exist” | App on disk can still work; run `splunk enable app TA_unifi_ng` after extract |
| Bare `python3` → `libssl.so.3` | Use **`splunk cmd python`** |
| In-container `splunk restart` hangs | Use **`docker restart splunk-unifi-test`** |
| No events, 401 from controller | Rotate Integration API key; test with `curl -sk -H "X-API-KEY: ..." https://192.168.1.1/proxy/network/integration/v1/sites` |
| Self-signed TLS | Set `verify_ssl = 0` in the input stanza |
| Controller unreachable from container | Use host gateway IP or published URL; verify Docker network can reach controller :443 |
| `splunk list modularinput` fails on Splunk 10 | Not supported in this image; use scheme validation + splunkd.log instead |

## Reset lab (clean volumes)

```bash
cd ~/Projekte/TA/docker-splunk-unifi-test
docker compose down -v
docker compose up -d
# repeat install steps as splunk user only
```

## Related docs

- Results summary: `04-test-results.md`
- API probe: `01-api-probe-results.md`
- Project plan: `00-project-plan.md`

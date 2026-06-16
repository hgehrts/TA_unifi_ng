# UCC config page: "Unable to xml-parse the following data: %s" (v2.0.0 → fixed in 2.0.1)

**Symptom:** Configuration tab in Splunk shows "Something went wrong! Unable to xml-parse the following data: %s". REST call to `/servicesNS/nobody/TA_unifi_ng/TA_unifi_ng_account` returns HTTP 500; `splunkd.log` shows `AdminManagerExternal - Received malformed XML from external handler:` (empty body).

**Root cause:** `bin/TA_unifi_ng_rh_account.py` was missing its persistent-handler entrypoint. Every UCC REST handler that runs with `handlerpersistentmode = true` (per `restmap.conf`) MUST end with:

```python
if __name__ == "__main__":
    logging.getLogger().addHandler(logging.NullHandler())
    admin_external.handle(endpoint, handler=UniFiAccountHandler)
```

Without it, splunkd execs the handler file, it defines the class/endpoint and exits with **empty stdout** → splunkd can't parse the (empty) XML response → the UI shows the `%s` placeholder error. The settings and unifi_ingest handlers had the block; the account handler did not.

**Diagnosis path (Splunk 10.2.3 in Docker, container `8656761dc1eb`):**
```bash
# 1) Reproduce the 500 directly
docker exec -u splunk <cid> /opt/splunk/bin/splunk _internal call \
  /servicesNS/nobody/TA_unifi_ng/TA_unifi_ng_account -auth admin:<SPLUNK_ADMIN_PASSWORD>
# 2) Confirm empty-XML in splunkd.log
docker exec -u splunk <cid> bash -c \
  'tail -50 /opt/splunk/var/log/splunk/splunkd.log | grep AdminManagerExternal'
# 3) Compare handlers — the broken one lacks admin_external.handle
docker exec -u splunk <cid> bash -c \
  'grep -c admin_external.handle /opt/splunk/etc/apps/TA_unifi_ng/bin/*_rh_*.py'
```

**Red herring:** `splunktaucclib.get_base_app_name()` raising `RestError(500): Cannot get app name from file: <stdin>` appears when reproducing via `python -c`/`runpy` (because `__main__.__file__` is `<stdin>`), but resolves correctly under real splunkd. Do NOT patch the library; adding the entrypoint alone fixes it (verified by reverting the lib to stock and confirming all endpoints return 200).

**Fix applied:** Added the `__main__` block to `package/bin/` and `output/TA_unifi_ng/bin/` account handlers, bumped to **2.0.1**, rebuilt `TA_unifi_ng-2.0.1.tar.gz`. Verified fresh install: Accounts / Logging / Inputs tabs all return clean JSON.

---

# Splunk host `splk` — TA_unifi_ng 1.3.0 not working

## What we know from inspection

| Item | Value |
|------|--------|
| Splunk | **10.4.0** |
| App | `/opt/splunk/etc/apps/TA_unifi_ng`, **enabled**, version **1.3.0** |
| Instance stanza | `[unifi_ingest://docker_test]` in `local/inputs.conf` |

## Most likely causes (in order)

### 1. Missing API key (very likely)

The live `local/inputs.conf` on **splk** had **no `api_key`** and no evidence of Setup UI credentials.

The script resolves the key from:

1. Stanza field `api_key`, or  
2. `storage/passwords` with `realm=TA_unifi_ng`, `name=api_key`

Without either, every poll fails with `UniFiApiError: UniFi API key is required...` and **nothing is indexed**.

**Fix:** Either add `api_key = ...` to the stanza (see `splk-inputs.local.conf.example`) **or** use **Settings → Data inputs → TA_unifi_ng → Add new** and save the API key via the setup page (recommended).

### 2. `verify_ssl = false` in `.conf`

Splunk expects booleans as **`0`** / **`1`**, not `false` / `true`. Your splk file used `verify_ssl = false`.

Use:

```ini
verify_ssl = 0
```

The Python code also accepts the string `"false"`, but normalizing avoids Splunk-side quirks.

### 3. Legacy / copied Docker settings

The splk stanza still had `collection_timeout` and `collection_types` from an old Docker `inputs.local.conf`. Current **1.3.0** code uses `include_devices`, `include_clients`, `include_networks` instead. These legacy keys are **ignored** (defaults apply), so they are not the primary failure — but the stanza was clearly copied from the lab without splk-specific setup.

### 4. Index `unifi` missing

If the index was never created on splk, events are dropped. Check **Settings → Indexes** for `unifi`.

### 5. Network from splk to UniFi controller

`splk` must reach `https://<controller>:443` (or your chosen URL). Test from splk:

```bash
curl -sk -o /dev/null -w "%{http_code}\n" https://192.168.1.1/proxy/network/integration/v1/sites
```

(401/403 without key is normal; connection errors are not.)

## What is NOT the problem (if install matches 1.3.0 tarball)

- **`[script://...]`** — splk has a proper `[unifi_ingest]` packaged stanza in `default/inputs.conf` (verified in 1.3.0 package).
- **`host` argument** — fixed in 1.3.0 as `controller_url`.

## Fix procedure on splk

```bash
# 1) Diagnose
bash splk-remote-diagnose.sh   # or pipe via ssh

# 2) Fix local/inputs.conf (example in splk-inputs.local.conf.example)
sudo cp splk-inputs.local.conf.example /opt/splunk/etc/apps/TA_unifi_ng/local/inputs.conf
sudo chown splunk:splunk /opt/splunk/etc/apps/TA_unifi_ng/local/inputs.conf
sudo chmod 600 /opt/splunk/etc/apps/TA_unifi_ng/local/inputs.conf
# Edit: real controller_url, api_key OR use Setup UI and remove api_key line

# 3) Create index if needed
sudo -u splunk /opt/splunk/bin/splunk add index unifi -auth 'admin:YOUR_PASSWORD'

# 4) Restart input / Splunk
sudo -u splunk /opt/splunk/bin/splunk restart -auth 'admin:YOUR_PASSWORD'
# Or disable/enable input in UI

# 5) Validate
sudo -u splunk /opt/splunk/bin/splunk cmd python \
  /opt/splunk/etc/apps/TA_unifi_ng/bin/unifi_ingest.py --validate-arguments
sudo -u splunk /opt/splunk/bin/splunk cmd python \
  /opt/splunk/etc/apps/TA_unifi_ng/bin/unifi_ingest.py --scheme --dry-run
```

Search:

```spl
index=unifi earliest=-15m | stats count by sourcetype
```

## Docker lab vs splk

| | Docker (works) | splk (broken) |
|--|----------------|---------------|
| `controller_url` | `https://192.168.1.1` | `192.168.1.1` (OK after normalize) |
| `verify_ssl` | `0` | `false` → use `0` |
| `api_key` | in stanza | **missing** |
| Splunk | 10.x in container | **10.4.0** |

## SSH note

Automated SSH from the dev environment to `splk:22` timed out (ping OK). Run diagnostics **from your machine** where `ssh splunk@splk` works.

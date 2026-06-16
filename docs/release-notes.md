# TA_unifi_ng — Release & Distribution Notes

**Add-on:** UniFi Network Add-on for Splunk (`TA_unifi_ng`)
**Current release:** 3.3.0
**Artifact:** `TA_unifi_ng-3.3.0.tar.gz`
**SHA-256:** `9878f4066ebb310c09f83032447f69a40d89039291eb678ffa593050a88fbd21`
**Tested on:** Splunk Enterprise 10.x (10.2.3 verified), Python 3.9
**Controller API:** UniFi Network Integration API v1 (verified against 10.4.57)

---

## What it does

Polls a UniFi Network controller's Integration API (`/proxy/network/integration/v1`)
and indexes inventory, configuration, per-device telemetry and reference data
into Splunk. Read-only. Credentials are stored encrypted (UCC pattern).

**25 sourcetypes** across **3 cadence-grouped inputs** plus a retained legacy input.

---

## Compatibility & requirements

| Item | Requirement |
|------|-------------|
| Splunk | Enterprise 10.x (single instance or via deployment server) |
| Python | `python3` (bundled) |
| Controller | UniFi Network with Integration API + an API key (Settings → Control Plane → Integrations) |
| Network | Splunk host must reach the controller on HTTPS (443) directly (no auth proxy) |
| Permissions | `admin`/`sc_admin` to configure the account & inputs |

Self-signed controller certs: set **Verify SSL = off** on the account.

---

## Install

1. **Splunk Web → Apps → Manage Apps → Install app from file** → upload
   `TA_unifi_ng-3.3.0.tar.gz` → restart when prompted.
   *(CLI alternative: place the tarball and `splunk install app TA_unifi_ng-3.3.0.tar.gz`.)*
2. Open **UniFi Network Add-on for Splunk → Configuration → Account → Add**:
   - **Controller URL**: `https://<controller-ip>` (scheme optional; `https://` is added)
   - **API Key**: the Integration API key
   - **Verify SSL**: off for self-signed
   Saving validates the connection against the controller.
3. Create an **index** for the data (recommended: one index for everything, e.g. `unifi`).
4. **Settings → Data inputs** → create the inputs you want (see below).

### Upgrade from 2.x / 3.x

- In-place upgrade is safe: re-upload the new tarball (Splunk replaces `default/`,
  preserves `local/`). Your account credentials and existing inputs are kept.
- **2.x → 3.x:** the legacy `unifi_ingest` input keeps working unchanged. To get the
  new sourcetypes, add the new **UniFi Inventory / Telemetry / Reference** inputs.
  You may disable `unifi_ingest` once the Inventory input is collecting (it supersedes it).
- No index or field renames; all changes are additive.

---

## Inputs

| Input | Default interval | Collects |
|-------|------------------|----------|
| **UniFi Inventory** (`unifi_inventory`) | 180 s | devices, clients, networks, device-tags, firewall (zones/policies/acl), wifi, wan, vpn, switching, dns, traffic-lists, radius, vouchers, pending-devices, info; optional per-network detail |
| **UniFi Telemetry** (`unifi_telemetry`) | 60 s | per-device statistics (CPU/mem/load/uptime/uplink) + full device detail (ports/radios). **1 API call per device per collector** |
| **UniFi Reference** (`unifi_reference`) | 86400 s (daily) | countries, DPI applications, DPI categories (large, near-static) |
| `unifi_ingest` (legacy) | 300 s | sites/devices/clients/networks — kept for backward compatibility |

Each input has its own native interval (editable), auto-discovers all sites (or a
filter), and toggles per list. Each toggle's help text names its sourcetype and
links to the Ubiquiti API docs. **Send all inputs to the same index.**

**Scaling note:** Telemetry cost = `2 + devices × enabled_collectors` calls per
run. On large fleets raise the Telemetry interval.

---

## Sourcetypes (25)

**Inventory/config:** `unifi:site`, `unifi:device`, `unifi:client`, `unifi:network`,
`unifi:device_tag`, `unifi:firewall:zone`, `unifi:firewall:policy`, `unifi:acl_rule`,
`unifi:wifi:broadcast`, `unifi:wan`, `unifi:vpn:server`, `unifi:vpn:tunnel`,
`unifi:dns:policy`, `unifi:traffic_matching_list`, `unifi:radius:profile`,
`unifi:hotspot:voucher`, `unifi:switching:lag`, `unifi:switching:mc_lag_domain`,
`unifi:switching:switch_stack`, `unifi:device:pending`, `unifi:info`

**Network enrichment:** `unifi:network:detail`, `unifi:network:reference`

**Telemetry:** `unifi:device:stats`, `unifi:device:detail`

**Reference:** `unifi:ref:country`, `unifi:ref:dpi_application`, `unifi:ref:dpi_category`

Every event carries: `unifi_host`, `unifi_site_id`, `unifi_site_name`,
`unifi_object_type`, `unifi_collection_time`, plus `unifi_api_<field>` for API
timestamps and the full object JSON in `_raw`. Controller-wide objects use
`unifi_site_id = _global`. Telemetry events carry `unifi_device_id` /
`unifi_device_mac` for joins.

---

## CIM compliance

Field aliases, calculated fields and tags map onto Splunk CIM data models:

| Sourcetype(s) | Data model | Notable fields |
|---|---|---|
| `unifi:device`, `:detail` | Inventory / Network | `mac`, `ip`, `dest`, `family`, `version`, `status`, `dvc`, `vendor_product` |
| `unifi:client` | Network Sessions | `src`, `src_ip`, `src_mac`, `user`, `action`, `connection_type` |
| `unifi:device:stats` | Performance | `cpu_load_percent`, `mem_used_percent`, `uptime`, `dvc`, `mac` |
| `unifi:firewall:policy` | (rules) | `rule`, `action`, `src_zone`, `dest_zone` |

To accelerate CIM data models against this data, include your UniFi index in the
CIM app's data model constraints.

---

## Dashboards

Under the app's **Dashboards** menu (each has an **Index** input, default `*`):

- **UniFi - Overview** — site/device/client/network counts, events by sourcetype, models, client types
- **UniFi - Device Health** — CPU/memory, CPU over time per device, top devices, inventory + state (needs Telemetry input)
- **UniFi - Clients & Network** — clients over time, per-uplink, firewall by action, SSIDs, recent clients

---

## Verify after install

```spl
index=<your_index> sourcetype=unifi:* | stats count by sourcetype
index=<your_index> sourcetype=unifi:device | table unifi_site_name name model ipAddress state firmwareVersion
index=<your_index> tag=inventory tag=network | stats count by sourcetype
index=<your_index> sourcetype=unifi:device:stats | stats avg(cpu_load_percent) by unifi_device_mac
```

Device `state`: `ONLINE`, `OFFLINE`, `PENDING_ADOPTION`, `UPDATING`,
`GETTING_READY`, `ADOPTING`.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Config page "Unable to xml-parse the following data: %s" | Pre-3.0 handler bug; fixed since 2.0.1. Reinstall current build. |
| No data, 401 from controller | API key missing/invalid; re-enter on the Account page. Rotate the key if needed. |
| Self-signed TLS errors | Account → Verify SSL = off. |
| Events with wrong `_time` | Fixed in 3.1.1 (`_time` = collection time for snapshots). |
| High request rate on big fleets | Raise the Telemetry input interval. |
| 0 events for some sourcetypes | That object type may be empty on your controller (normal). |

---

## CHANGELOG

### 3.3.0
- CIM compliance: FIELDALIAS/EVAL field mapping for Inventory, Network Sessions,
  Performance data models; `eventtypes.conf` + `tags.conf`.
- Three sample dashboards + Dashboards nav menu.
- Consolidated `props.conf` (one stanza per sourcetype).

### 3.2.0
- **Phase C**: `collect_network_detail` toggle on the Inventory input →
  `unifi:network:detail` + `unifi:network:reference`.
- **Phase D**: new **UniFi Reference** input (daily) →
  `unifi:ref:country` / `:dpi_application` / `:dpi_category`.

### 3.1.1
- `_time` hardening: snapshot sourcetypes use `unifi_collection_time`
  (emitted first in the payload; `MAX_TIMESTAMP_LOOKAHEAD=256`);
  `unifi:device:stats` uses `lastHeartbeatAt`. Fixes stale client/voucher timestamps.

### 3.1.0
- **Phase B**: new **UniFi Telemetry** input (60 s) →
  `unifi:device:stats` + `unifi:device:detail`, per-device error isolation,
  `unifi_device_id`/`unifi_device_mac` join keys.

### 3.0.0
- **Phase A**: new **UniFi Inventory** input (180 s) with 15 per-list toggles
  (sourcetype-labelled + doc links). 16 inventory/config sourcetypes.
  Legacy `unifi_ingest` retained.

### 2.0.1
- Fixed UCC persistent REST handler (missing `__main__` entrypoint) that broke
  the Configuration page.

### 1.x – 2.0.0
- Initial UCC add-on: sites, devices, clients, networks via a single input.

---

## Build / maintenance reference

- Repo layout: `TA_unifi_ng/` is the ready-to-install app (what ships);
  `src/package/` + `src/globalConfig.json` are the ucc-gen inputs to regenerate it.
- Endpoint registry lives in `TA_unifi_ng/bin/unifi_ingest.py`
  (`INVENTORY_ENDPOINTS`, `REFERENCE_ENDPOINTS`, telemetry collectors) — the
  single place to add new API lists.
- Live OpenAPI spec snapshot: `_ref/integration-oas.json`. Re-pull on controller
  upgrades: `curl -sk -H "X-API-KEY: <key>" https://<controller>/proxy/network/api-docs/integration.json`
- Rebuild the package: `./build.sh` (writes `dist/TA_unifi_ng-<ver>.tar.gz` + `.sha256`).
- Regenerate the app from ucc-gen sources: see `src/README.md`.
- Every persistent REST handler **must** end with the
  `if __name__ == "__main__": admin_external.handle(...)` block.

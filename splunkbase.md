# UniFi Network Add-on for Splunk — Splunkbase listing content

Copy the sections below into the Splunkbase add-on listing fields (Short Description, Summary, Details, Installation, Troubleshooting), matching the structure used for [Whiteboard App](https://splunkbase.splunk.com/app/8908).

**Listing type:** add-on (Technical Add-on / modular input)  
**Companion app (separate listing):** [UniFi App for Splunk](https://github.com/hgehrts/unifi_app_for_splunk)

---

## Short Description

Read-only Splunk add-on that polls the UniFi Network Integration API v1 and indexes inventory, configuration, per-device telemetry and reference data into Splunk. 25 sourcetypes, CIM-mapped, cadence-grouped inputs.

---

## Summary

UniFi Network Add-on for Splunk collects structured UniFi Network data directly from your local controller's Integration API — sites, devices, clients, networks, firewall, WiFi, WAN, VPN, switching, DNS, traffic lists, RADIUS, vouchers and more. Three cadence-grouped modular inputs (inventory, telemetry, reference) with per-list toggles and editable intervals. Read-only GET requests only; API keys stored encrypted. CIM field aliases and tags for Inventory, Network Sessions and Performance. Three sample dashboards included. Built for network and platform teams who want UniFi asset and telemetry data in Splunk without syslog parsing alone.

---

## Details

### What it does

This Technical Add-on polls a UniFi Network controller at `/proxy/network/integration/v1` on a schedule you configure. Each poll cycle discovers sites (or a filtered subset), walks the enabled API lists, and writes one Splunk event per object with normalized fields plus the full JSON in `_raw`.

Every event carries `unifi_host`, `unifi_site_id`, `unifi_site_name`, `unifi_object_type`, `unifi_collection_time`, and API timestamps as `unifi_api_*` fields. Controller-wide objects use `unifi_site_id = _global`; telemetry events include `unifi_device_id` and `unifi_device_mac` for joins.

### Inputs (3 cadence groups + legacy)

| Input | Default interval | Collects |
|-------|------------------|----------|
| **UniFi Inventory** | 180 s | devices, clients, networks, device-tags, firewall (zones/policies/ACL), WiFi, WAN, VPN, switching, DNS, traffic-lists, RADIUS, vouchers, pending-devices, info; optional per-network detail |
| **UniFi Telemetry** | 60 s | per-device statistics (CPU, memory, load, uptime, uplink) + full device detail (ports, radios) — **1 API call per device per enabled collector** |
| **UniFi Reference** | 86400 s (daily) | countries, DPI applications, DPI categories |
| `unifi_ingest` (legacy) | 300 s | sites, devices, clients, networks — retained for backward compatibility |

Send all inputs to the **same index** (recommended: `unifi`).

**Scaling:** telemetry cost ≈ `2 + devices × enabled_collectors` API calls per run. On large fleets, increase the telemetry interval.

### Sourcetypes (25)

`unifi:site` · `unifi:device` · `unifi:device:detail` · `unifi:device:stats` · `unifi:device:pending` · `unifi:client` · `unifi:network` · `unifi:network:detail` · `unifi:network:reference` · `unifi:info` · `unifi:device_tag` · `unifi:firewall:zone` · `unifi:firewall:policy` · `unifi:acl_rule` · `unifi:wifi:broadcast` · `unifi:wan` · `unifi:vpn:server` · `unifi:vpn:tunnel` · `unifi:dns:policy` · `unifi:traffic_matching_list` · `unifi:radius:profile` · `unifi:hotspot:voucher` · `unifi:switching:lag` · `unifi:switching:mc_lag_domain` · `unifi:switching:switch_stack` · `unifi:ref:country` · `unifi:ref:dpi_application` · `unifi:ref:dpi_category`

### CIM compliance

| Sourcetype(s) | Data model | Notable fields |
|---|---|---|
| `unifi:device`, `:detail` | Inventory / Network | `mac`, `ip`, `dest`, `family`, `version`, `status`, `dvc`, `vendor_product` |
| `unifi:client` | Network Sessions | `src`, `src_ip`, `src_mac`, `user`, `action`, `connection_type` |
| `unifi:device:stats` | Performance | `cpu_load_percent`, `mem_used_percent`, `uptime`, `dvc`, `mac` |
| `unifi:firewall:policy` | (rules) | `rule`, `action`, `src_zone`, `dest_zone` |

### Dashboards (included in the add-on)

- **UniFi - Overview**
- **UniFi - Device Health** (requires Telemetry input)
- **UniFi - Clients & Network**

### Companion app

For syslog correlation, ID→name enrichment and advanced Dashboard Studio views, install the separate **UniFi App for Splunk** (`unifi_app_for_splunk`) after this add-on. That app joins this TA's asset data with UniFi syslog collected via Splunk Connect for Syslog (SC4S).

### External services

The add-on does not phone home and includes no product analytics. It issues **read-only HTTPS GET** requests only to the UniFi controller URL you configure.

### Compatibility

| Platform | Minimum version |
|---|---|
| Splunk Enterprise | 8.0+ (manifest); tested on 10.x |
| Splunk Cloud (Victoria) | Expected to work; validate with AppInspect and your Cloud vetting process |

Requires a UniFi Network controller with the **Integration API** enabled and an API key (Settings → Control Plane → Integrations). Verified against UniFi Network **10.4.57**.

Python 3 (bundled by Splunk). No third-party pip dependencies at runtime beyond libraries shipped in the package.

### Roles and permissions

`admin` or `sc_admin` to configure the account and inputs. The modular inputs run as the Splunk user on the instance where they are enabled (search head, indexer, or forwarder per your deployment).

### Source and license

- **Source code:** https://github.com/hgehrts/TA_unifi_ng
- **License:** Apache-2.0
- **Author:** Hans-Henning Gehrts

> Not affiliated with or endorsed by Ubiquiti Inc. or Splunk LLC.

---

## Installation

**Restart required:** After install or upgrade, restart Splunk when prompted before configuring inputs.

### Splunkbase (Splunk Enterprise)

1. Log in to [Splunkbase](https://splunkbase.splunk.com) and open this add-on listing.
2. Click **Download** and save `TA_unifi_ng.tar.gz`.
3. In Splunk Web, go to **Apps → Manage Apps → Install app via upload**.
4. Upload the package. Restart Splunk when prompted.
5. Open **UniFi Network Add-on for Splunk → Configuration → Account → Add**:
   - **Controller URL:** `https://<controller-ip-or-hostname>`
   - **API Key:** Integration API key from the controller
   - **Verify SSL:** off for self-signed certificates
6. Create an index (recommended: `unifi`).
7. **Settings → Data inputs** → enable **UniFi Inventory**, **UniFi Telemetry** and/or **UniFi Reference** as needed. Point all inputs to the same index.

### Splunk Cloud

1. Upload through your stack's private-app or vetted-app workflow.
2. Ensure the package passes AppInspect and your org's Cloud vetting requirements.
3. Configure account and inputs as above.

### Manual install (Splunk Enterprise)

```bash
git clone https://github.com/hgehrts/TA_unifi_ng.git
cd TA_unifi_ng
./build.sh
# Produces dist/TA_unifi_ng-<version>.tar.gz

# On the Splunk server:
$SPLUNK_HOME/bin/splunk install app TA_unifi_ng-<version>.tar.gz -update 1 -auth admin:changeme
$SPLUNK_HOME/bin/splunk restart
```

### Verify

```spl
index=unifi sourcetype=unifi:* | stats count by sourcetype
```

### Upgrading

Re-upload the newer package. Splunk preserves `local/` (credentials, input stanzas). Changes in `default/` are replaced. No sourcetype renames between 3.x releases; upgrades are additive.

---

## Troubleshooting

### No events indexed

- Confirm the **Account** saved successfully (Configuration tab shows your controller).
- Check that at least one input is **enabled** and assigned to a valid index.
- Verify network reachability from the Splunk host to the controller on HTTPS (443).
- Search `_internal` for `unifi` or the input name: `index=_internal source=*unifi*`.

### `UniFi API key is required`

The API key must be saved via the Configuration **Account** page (encrypted in `storage/passwords`) or set on the input stanza. Do not leave the account empty.

### SSL / certificate errors

For self-signed controller certificates, set **Verify SSL = off** on the account. For production, prefer a trusted certificate on the controller.

### Telemetry input slow or timing out

Telemetry performs **one API call per device per enabled collector**. On large fleets, increase the telemetry interval or disable collectors you do not need.

### Configuration page XML parse error (older builds)

Versions before 2.0.1 had a missing REST handler entrypoint on the Account endpoint. Upgrade to the current release.

### Splunk Cloud

Follow your Cloud admin's process for modular inputs on heavy forwarders vs. search heads. Ensure the polling host can reach the UniFi controller.

### Getting help

- **Documentation:** https://github.com/hgehrts/TA_unifi_ng/blob/main/README.md
- **Issues:** https://github.com/hgehrts/TA_unifi_ng/issues
- **Contact:** hgehrts@splunk.com (or open a GitHub issue)

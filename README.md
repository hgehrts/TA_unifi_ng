# UniFi Network Add-on for Splunk (`TA_unifi_ng`)

A read-only Splunk Technical Add-on that polls a **UniFi Network controller's
Integration API v1** (`/proxy/network/integration/v1`) and indexes inventory,
configuration, per-device telemetry and reference data into Splunk Enterprise.

- **25 sourcetypes** across **3 cadence-grouped inputs** (+ a retained legacy input)
- Per-list collection toggles, each labelled with its sourcetype and a link to the Ubiquiti API docs
- Configurable polling intervals per input (inventory 180 s, telemetry 60 s, reference daily)
- CIM-mapped (Inventory, Network Sessions, Performance) with field aliases + tags
- Three sample dashboards
- No third-party network dependencies for collection (stdlib `urllib`); strictly read-only

> Built with the Splunk UCC framework. Tested on Splunk Enterprise 10.x
> (10.2.3) against UniFi Network 10.4.57.

---

## Install

### From a release package (recommended)

1. Download `TA_unifi_ng-<version>.tar.gz` (from [`dist/`](dist/) or a GitHub Release).
2. Splunk Web → **Apps → Manage Apps → Install app from file** → upload the tarball → restart.
3. Open **UniFi Network Add-on for Splunk → Configuration → Account → Add** and enter:
   - **Controller URL** — `https://<controller-ip>`
   - **API Key** — created in the controller under *Settings → Control Plane → Integrations*
   - **Verify SSL** — off for self-signed certs
4. Create a Splunk **index** (recommended: one index for all UniFi data, e.g. `unifi`).
5. **Settings → Data inputs** → add the inputs you want (below).

### Verify

```spl
index=<your_index> sourcetype=unifi:* | stats count by sourcetype
```

See [`docs/release-notes.md`](docs/release-notes.md) for the full install/upgrade guide.

> **Companion app:** a separate [**UniFi App for Splunk**](https://github.com/hgehrts/unifi_app_for_splunk)
> (`unifi_app_for_splunk`) correlates this TA's data with UniFi syslog (via SC4S),
> enriches cryptic device/client IDs to names/IP/MAC/vendor, and ships four
> linked Dashboard Studio dashboards for problem identification and root-cause
> analysis.

---

## Inputs

| Input | Default interval | Collects |
|-------|------------------|----------|
| **UniFi Inventory** (`unifi_inventory`) | 180 s | devices, clients, networks, device-tags, firewall (zones/policies/acl), wifi, wan, vpn, switching, dns, traffic-lists, radius, vouchers, pending-devices, info; optional per-network detail |
| **UniFi Telemetry** (`unifi_telemetry`) | 60 s | per-device statistics (CPU/mem/load/uptime/uplink) + full device detail (ports/radios) — **1 API call per device per collector** |
| **UniFi Reference** (`unifi_reference`) | 86400 s (daily) | countries, DPI applications, DPI categories |
| `unifi_ingest` (legacy) | 300 s | sites/devices/clients/networks — kept for backward compatibility |

Each input auto-discovers all sites (or a filter), has its own editable interval,
and toggles per list. Send all inputs to the **same index**.

**Scaling:** telemetry cost ≈ `2 + devices × enabled_collectors` calls per run;
raise the telemetry interval on large fleets.

---

## Sourcetypes

`unifi:site` · `unifi:device` · `unifi:device:detail` · `unifi:device:stats` ·
`unifi:device:pending` · `unifi:client` · `unifi:network` · `unifi:network:detail` ·
`unifi:network:reference` · `unifi:info` · `unifi:device_tag` ·
`unifi:firewall:zone` · `unifi:firewall:policy` · `unifi:acl_rule` ·
`unifi:wifi:broadcast` · `unifi:wan` · `unifi:vpn:server` · `unifi:vpn:tunnel` ·
`unifi:dns:policy` · `unifi:traffic_matching_list` · `unifi:radius:profile` ·
`unifi:hotspot:voucher` · `unifi:switching:lag` · `unifi:switching:mc_lag_domain` ·
`unifi:switching:switch_stack` · `unifi:ref:country` · `unifi:ref:dpi_application` ·
`unifi:ref:dpi_category`

Every event carries `unifi_host`, `unifi_site_id`, `unifi_site_name`,
`unifi_object_type`, `unifi_collection_time`, plus `unifi_api_<field>` for API
timestamps and the full object JSON in `_raw`. Controller-wide objects use
`unifi_site_id = _global`; telemetry events carry `unifi_device_id` /
`unifi_device_mac` for joins.

---

## CIM compliance

| Sourcetype(s) | Data model | Notable fields |
|---|---|---|
| `unifi:device`, `:detail` | Inventory / Network | `mac`, `ip`, `dest`, `family`, `version`, `status`, `dvc`, `vendor_product` |
| `unifi:client` | Network Sessions | `src`, `src_ip`, `src_mac`, `user`, `action`, `connection_type` |
| `unifi:device:stats` | Performance | `cpu_load_percent`, `mem_used_percent`, `uptime`, `dvc`, `mac` |
| `unifi:firewall:policy` | (rules) | `rule`, `action`, `src_zone`, `dest_zone` |

---

## Dashboards

Under the app's **Dashboards** menu (each has an **Index** input):

- **UniFi - Overview**
- **UniFi - Device Health** (uses the Telemetry input)
- **UniFi - Clients & Network**

---

## Build from source

The ready-to-install app is committed under [`TA_unifi_ng/`](TA_unifi_ng/).

```bash
./build.sh            # -> dist/TA_unifi_ng-<VERSION>.tar.gz (+ .sha256)
./build.sh 3.3.1      # override version label
```

To regenerate with the Splunk UCC generator from the sources in
[`src/`](src/) (`package/` + `globalConfig.json`), see [`src/README.md`](src/README.md).

### Continuous integration / releases

[`.github/workflows/build-release.yml`](.github/workflows/build-release.yml):

- On every push/PR to `main`: builds the package with `build.sh`, validates it
  (single top-level dir, no junk/native binaries, Python compiles, checksum
  matches) and uploads it as a build artifact.
- On a version tag `v*` (e.g. `v3.3.1`): builds and attaches the tarball +
  `.sha256` to a GitHub Release, using `docs/release-notes.md` as the body.

Cut a release:

```bash
git tag v3.3.1 && git push origin v3.3.1
```

---

## Repository layout

```
TA_unifi_ng/        Ready-to-install Splunk app (this is what ships)
src/                UCC-generator sources (package/, globalConfig.json) to rebuild the app
dist/               Prebuilt release tarball + SHA-256
docs/               Architecture, event schema, API expansion plan, testing, troubleshooting, release notes
_ref/               UniFi Integration API OpenAPI snapshot (reference)
build.sh            Package the app into dist/
LICENSE             Apache-2.0
```

---

## Requirements

- Splunk Enterprise 10.x, Python 3 (bundled by Splunk)
- A UniFi Network controller with the Integration API enabled and an API key
- Network reachability from the Splunk host to the controller over HTTPS (no auth proxy)

---

## Security

- The add-on is **read-only** — it issues only `GET` requests.
- API keys are stored **encrypted** by Splunk (UCC account/`storage/passwords`); never put them in `inputs.conf`.
- Do not commit `local/`, `passwords.conf`, or any API key — see `.gitignore`.

---

## License

Apache-2.0 — see [LICENSE](LICENSE).

> Not affiliated with or endorsed by Ubiquiti Inc. or Splunk LLC. "UniFi" is a
> trademark of Ubiquiti Inc.; "Splunk" is a trademark of Splunk LLC.

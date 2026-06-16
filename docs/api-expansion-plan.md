# UniFi TA NG — API Expansion Plan (v3.x)

**Status:** ✅ IMPLEMENTED & VERIFIED — all phases A–E shipped (current version **3.3.0**)
**Date:** 2026-06-16 (rev 3 — implementation complete)

> **Implementation summary (2026-06-16):** All five phases built, installed, and
> verified live against the controller at `https://192.168.1.1` (Splunk 10.2.3
> in Docker). The TA grew from 4 sourcetypes to **25**, across 3 cadence-grouped
> inputs (+ the retained legacy input), with CIM compliance and 3 dashboards.
> See §7 for the version → phase map and §12 for the verification record.
**Controller probed:** `https://192.168.1.1`, app version **10.4.57**
**API:** `/proxy/network/integration/v1` (OpenAPI 3.1, 44 paths, 41 GET)
**Spec saved:** `_ref/integration-oas.json` (authoritative, pulled live from controller at `/proxy/network/api-docs/integration.json`)
**Currently ingested (v2.x):** sites, devices, clients, networks

---

## 1. Goal

Ingest **all read-only objects** the Integration API exposes, plus **full per-object detail** and **device performance telemetry**. Give the Splunk admin **per-cadence configurable polling intervals** (not a single hardcoded 300s), surface each list's **sourcetype name in the UI**, and document every list with a **link to the Ubiquiti API docs**. Keep API call volume controlled.

---

## 2. Key findings (OpenAPI schema + live probe)

| Finding | Detail |
|---------|--------|
| Pagination | Uniform envelope `{offset,limit,count,totalCount,data[]}` on **every** list endpoint. `limit` **max 200** (default 25). Empty collections return `count:0,data:[]` (never null) → collector loop is safe. |
| Device list vs detail | **Detail is significantly richer** than first thought. List `interfaces` = `array of string` (names only). **Detail `interfaces` = object with `ports[]`** (per-port `idx/state/connector/speedMbps/maxSpeedMbps/poe`) **and `radios[]`** (channel/width/wlanStandard/frequencyGHz). Detail also adds `adoptedAt`, `provisionedAt`, `configurationId`, `uplink.deviceId`, `features.switching.lags[]`. → `collect_device_detail` is **recommended**, not just optional, for anyone doing port/PoE monitoring. |
| Client list vs detail | Identical (`access` = `{type: DEFAULT}` only). **Skip client detail.** |
| Device telemetry | `/devices/{id}/statistics/latest` = CPU %, mem %, load 1/5/15, `uptimeSec`, `lastHeartbeatAt`/`nextHeartbeatAt`, `uplink.{txRateBps,rxRateBps}`, per-radio `txRetriesPct`. High value, not in any list. |
| `filter` query param | Exists on most lists but **property names are undocumented and restricted** — live test `filter=enabled.eq(true)` → `400 api.request.invalid-filter "unknown filter property"`. **Do not use**; paginate only. |
| Empty-but-valid | On this controller: `acl-rules`, `dns/policies`, `mc-lag-domains`, `switch-stacks`, `pending-devices` = 0 items (still 200). Keep collectors; other sites/controllers will have data. |
| Reference data is large | `dpi/applications` = **2112**, `countries` = **248**, `dpi/categories` = 35. Near-static; separate low cadence. |
| `device.state` enum | `ONLINE, OFFLINE, PENDING_ADOPTION, UPDATING, GETTING_READY, ADOPTING` (note: **not** the `ACTIVE` value used in the old v1 example SPL — update sample searches). |
| Error model | 7 fields: `statusCode,statusName,code,message,timestamp,requestPath,requestId`. Use `requestId` + `code` in structured error logs. |
| Write surface | 32 POST/PUT/PATCH/DELETE ops exist. TA stays **strictly GET-only**. |
| No spec-documented 429 | Plan defensive backoff anyway. |

---

## 3. Endpoint inventory (all 41 GET)

### 3a. Collection / list endpoints → ingest as events

Group column: **INV** = `unifi_inventory` (180s), **TEL** = `unifi_telemetry` (60s), **REF** = `unifi_reference` (daily).

**Global (once per input run, not per-site):**

| Endpoint | Live count | Proposed sourcetype | Notes |
|----------|-----------|---------------------|-------|
| `/info` | 1 obj | `unifi:info` | Controller version; tiny heartbeat event |
| `/pending-devices` | 0 | `unifi:device:pending` | Devices awaiting adoption |
| `/countries` | 248 | `unifi:ref:country` | **Reference**, poll daily |
| `/dpi/applications` | 2112 | `unifi:ref:dpi_application` | **Reference**, poll daily, big |
| `/dpi/categories` | 35 | `unifi:ref:dpi_category` | **Reference**, poll daily |

**Site-scoped (per discovered site):**

| Endpoint | Live count | Proposed sourcetype |
|----------|-----------|---------------------|
| `/sites/{id}/devices` | 13 | `unifi:device` *(existing)* |
| `/sites/{id}/clients` | 68 | `unifi:client` *(existing)* |
| `/sites/{id}/networks` | 9 | `unifi:network` *(existing)* |
| `/sites/{id}/device-tags` | 1 | `unifi:device_tag` |
| `/sites/{id}/firewall/zones` | 7 | `unifi:firewall:zone` |
| `/sites/{id}/firewall/policies` | 121 | `unifi:firewall:policy` |
| `/sites/{id}/acl-rules` | 0 | `unifi:acl_rule` |
| `/sites/{id}/traffic-matching-lists` | 2 | `unifi:traffic_matching_list` |
| `/sites/{id}/dns/policies` | 0 | `unifi:dns:policy` |
| `/sites/{id}/wifi/broadcasts` | 4 | `unifi:wifi:broadcast` |
| `/sites/{id}/wans` | 2 | `unifi:wan` |
| `/sites/{id}/vpn/servers` | 3 | `unifi:vpn:server` |
| `/sites/{id}/vpn/site-to-site-tunnels` | 2 | `unifi:vpn:tunnel` |
| `/sites/{id}/radius/profiles` | 2 | `unifi:radius:profile` |
| `/sites/{id}/hotspot/vouchers` | 1 | `unifi:hotspot:voucher` |
| `/sites/{id}/switching/lags` | 1 | `unifi:switching:lag` |
| `/sites/{id}/switching/mc-lag-domains` | 0 | `unifi:switching:mc_lag_domain` |
| `/sites/{id}/switching/switch-stacks` | 0 | `unifi:switching:switch_stack` |

### 3b. Detail / sub-resource endpoints (need an ID from a list call)

| Endpoint | Enrichment value | Decision |
|----------|------------------|----------|
| `/devices/{id}` | **Major**: `interfaces.ports[]` (PoE/speed/state per port), `interfaces.radios[]`, `adoptedAt`, `provisionedAt`, `configurationId`, `uplink`, `features.switching.lags[]` | **Recommended** → emit `unifi:device:detail` (separate sourcetype, keeps the lean `unifi:device` list event intact) |
| `/devices/{id}/statistics/latest` | CPU/mem/load/uptime/uplink rates/radio retries | **Yes** → `unifi:device:stats` |
| `/clients/{id}` | none (== list) | **Skip** |
| `/networks/{id}` | richer than list | Optional → `unifi:network:detail` |
| `/networks/{id}/references` | what uses this network | Optional → `unifi:network:reference` |
| `/firewall/zones/{id}` | per-zone detail | Skip (list sufficient) |
| `/firewall/policies/{id}` | per-policy detail | Skip (list already rich, 14 props) |
| `/firewall/policies/ordering` | needs zone-pair params | Skip |
| `/acl-rules/{id}`, `/acl-rules/ordering` | detail/order | Skip |
| `/dns/policies/{id}` | detail | Skip |
| `/traffic-matching-lists/{id}` | detail | Skip |
| `/hotspot/vouchers/{id}` | detail | Skip (list rich, 13 props) |
| `/switching/lags/{id}`, `/mc-lag-domains/{id}`, `/switch-stacks/{id}` | detail | Skip |
| `/wifi/broadcasts/{id}` | detail | Skip (list rich) |

**Rationale:** detail-per-object multiplies calls by N (one per device). Pursue only where the list omits meaningful fields. For devices that is now clearly justified (port/radio inventory); for everything else the list payload is already complete. Emit device detail as its **own sourcetype** rather than merging, so `interfaces` doesn't collide between the string-array (list) and object (detail) shapes.

---

## 4. Proposed sourcetypes (full set)

### Existing (keep)
`unifi:site`, `unifi:device`, `unifi:client`, `unifi:network`

### New — inventory/config
`unifi:info`, `unifi:device:pending`, `unifi:device_tag`,
`unifi:firewall:zone`, `unifi:firewall:policy`, `unifi:acl_rule`,
`unifi:traffic_matching_list`, `unifi:dns:policy`,
`unifi:wifi:broadcast`, `unifi:wan`,
`unifi:vpn:server`, `unifi:vpn:tunnel`,
`unifi:radius:profile`, `unifi:hotspot:voucher`,
`unifi:switching:lag`, `unifi:switching:mc_lag_domain`, `unifi:switching:switch_stack`

### New — telemetry & detail (per-device, opt-in)
`unifi:device:stats`   ← `/devices/{id}/statistics/latest` (one event per device per poll)
`unifi:device:detail`  ← `/devices/{id}` (ports/radios/uplink; one event per device)

### New — optional network enrichment
`unifi:network:detail`, `unifi:network:reference`

### New — reference (low cadence)
`unifi:ref:country`, `unifi:ref:dpi_application`, `unifi:ref:dpi_category`

**Total:** 4 existing + 24 new = **28 sourcetypes** (3 of which are opt-in/optional enrichment).

All keep the existing envelope: `unifi_host`, `unifi_site_id`, `unifi_site_name`, `unifi_object_type`, `unifi_collection_time`, plus `unifi_api_*` timestamp mapping and full `_raw` JSON.

For global (non-site) objects, set `unifi_site_id`/`unifi_site_name` to a sentinel (e.g. `_global`).
For per-device events (`stats`, `detail`), always include `unifi_device_id` (and `unifi_device_mac` where available) so they join back to `unifi:device`.

---

## 5. Object schemas (from OpenAPI 10.4.57 + live probe)

Types from the spec; `→` shows nested structure that needs a flattening decision (see §5b).

```
info                : applicationVersion(str)
pending-device      : macAddress, ipAddress, model, state(enum), supported(bool),
                      firmwareVersion, firmwareUpdatable(bool),
                      features[str], adoptionTargetSiteIds[str]
device (list)       : id(uuid), macAddress, ipAddress, name, model, state(enum),
                      supported(bool), firmwareVersion, firmwareUpdatable(bool),
                      features[str], interfaces[str]   ← interfaces are NAMES only
device (detail)     : ... all of list, MINUS interfaces[str], PLUS:
                      adoptedAt(dt), provisionedAt(dt), configurationId,
                      uplink.deviceId(uuid),
                      features.switching.lags[{id,portIdxs[int],metadata}],
                      interfaces.ports[{idx,state(enum),connector(enum),
                                        maxSpeedMbps,speedMbps,
                                        poe{standard,type,enabled,state}}],
                      interfaces.radios[{wlanStandard(enum),frequencyGHz,
                                         channelWidthMHz,channel}]
device stats        : uptimeSec(int64), lastHeartbeatAt(dt), nextHeartbeatAt(dt),
                      loadAverage1Min/5Min/15Min(double),
                      cpuUtilizationPct(double), memoryUtilizationPct(double),
                      uplink{txRateBps,rxRateBps}(int64),
                      interfaces.radios[{frequencyGHz,txRetriesPct}]
client (list=detail): type, id(uuid), name, connectedAt(dt), ipAddress,
                      macAddress, uplinkDeviceId, access{type}
network             : id(uuid), name, enabled(bool), vlanId(int),
                      management(str), default(bool), metadata.origin
firewall:zone       : id(uuid), name, networkIds[str], metadata.origin
firewall:policy     : id(uuid), name, description, enabled(bool), index(int),
                      action.type, source{zoneId,trafficFilter.type},
                      destination{zoneId,trafficFilter.type},
                      ipProtocolScope.ipVersion, connectionStateFilter[str],
                      ipsecFilter(enum), loggingEnabled(bool),
                      schedule.mode, metadata.origin
wifi:broadcast      : id(uuid), name, type, enabled(bool),
                      network.type, securityConfiguration.type,
                      broadcastingDeviceFilter.type, metadata.origin
wan                 : id, name      ← minimal; detail not exposed (no /{id})
hotspot:voucher     : id, code, expired(bool), timeLimitMinutes,
                      createdAt(dt), activatedAt(dt), expiresAt(dt),
                      authorizedGuestCount, authorizedGuestLimit
vpn:server          : id, name, type, enabled(bool), metadata
vpn:tunnel          : id, name, ... (overview, 4 props)
device_tag          : name, deviceIds[str], metadata
radius:profile      : id, name, metadata
acl_rule            : 11 props (0 on this controller; schema "ACL ruleObject")
traffic_matching_list: 3 props
dns:policy          : 5 props
switching:lag       : 4 props ; mc_lag_domain: 5 ; switch_stack: 5
dpi:application      : id, name      (2112 items — reference)
dpi:category         : id, name      (35 items — reference)
country              : code, name    (248 items — reference)
```

### 5b. Nested-field flattening strategy

Most new objects nest one or two levels. Rule set (extends current behavior):

| Pattern | Action |
|---------|--------|
| Scalar top-level (`id`,`name`,`enabled`,`state`,…) | Extract as indexed field (as today). |
| Single-scalar nested (`action.type`, `source.zoneId`, `metadata.origin`, `uplink.deviceId`, `ipProtocolScope.ipVersion`) | Promote to flat field with **underscore path**: `action_type`, `source_zoneId`, `metadata_origin`, `uplink_deviceId`. |
| Numeric telemetry (`stats.*`, `uplink.txRateBps`) | Promote to flat numeric: `uplink_txRateBps`, etc. (search-time math/timecharts). |
| Arrays of scalars (`features[]`, `networkIds[]`, `deviceIds[]`, `connectionStateFilter[]`) | Keep as multivalue field (JSON in `_raw`; optional MV at index time). |
| Arrays of objects (`interfaces.ports[]`, `interfaces.radios[]`, `lags[]`) | **Do not flatten at index time** — leave in `_raw` for `spath`/`mvexpand` at search time. Optionally emit a small summary field (e.g. `port_count`, `radio_count`). |

Keep `_raw` = full JSON for every event regardless, so nothing is lost.

### 5c. Complete `unifi_api_*` timestamp map

Promote these (all `date-time`) to `unifi_api_<field>`:

| Sourcetype | API time fields |
|------------|-----------------|
| `unifi:client` | `connectedAt` |
| `unifi:device:detail` | `adoptedAt`, `provisionedAt` |
| `unifi:device:stats` | `lastHeartbeatAt`, `nextHeartbeatAt` |
| `unifi:hotspot:voucher` | `createdAt`, `activatedAt`, `expiresAt` |
| (device list, networks, firewall, wifi, wan, vpn, …) | none at top level |

`unifi_collection_time` is still added to **every** event at pull time.
Consider setting Splunk `_time` from the most meaningful API timestamp where one exists (e.g. `lastHeartbeatAt` for stats, `connectedAt` for clients); otherwise `_time` = collection time.

---

## 6. Collection design — grouped inputs by cadence

> **Design decisions (confirmed 2026-06-16):**
> - **Architecture:** input **groups by cadence** (each group = a real Splunk modular input with its own native `interval`). No custom internal scheduler. Splunk handles scheduling natively per input.
> - **Granularity:** per-**group** interval (related lists in a group share that group's interval). Within a group, individual lists are toggled on/off with `collect_*` checkboxes.
> - **Sourcetype visibility:** every collector toggle shows its **sourcetype in both the field label and the help text**.
> - **Index:** **one index** for all UniFi data (documented; each input still has an index field but guidance is "use the same index").
> - **Defaults:** group default interval **180s** (inventory/config), **telemetry 60s**, **reference 86400s (daily)**. *Note: the request first mentioned 300s as the universal default, then refined inventory to 180s — this plan uses 180s for inventory per the refinement. All intervals are user-editable; nothing is hardcoded to 300.*

### 6a. The three input types

Each appears as its own "Add input" option in **Settings → Data inputs → UniFi …**, each with a native `interval` field (user-editable, pre-filled with the default below).

| Input type | UCC input name | Default interval | Contains (collect_* toggles) |
|------------|----------------|------------------|------------------------------|
| **UniFi Inventory** | `unifi_inventory` | **180** | devices, clients, networks, device-tags, firewall (zones+policies+acl), wifi, wan, vpn, switching, dns, traffic-lists, radius, vouchers, pending-devices, info |
| **UniFi Telemetry** | `unifi_telemetry` | **60** | device_stats, device_detail (per-device endpoints) |
| **UniFi Reference** | `unifi_reference` | **86400** | countries, dpi_applications, dpi_categories |

Site auto-discovery + per-stanza fields (`account`, `index`, `page_size`, `collection_timeout`, `site_ids`) exist on **all three** input types.

### 6b. Toggle defaults per group

**`unifi_inventory`** (interval default 180s) — fixed ≈1-call collectors default **ON**, optional/heavier ones **OFF**:

```
collect_devices         default 1   sourcetype=unifi:device
collect_clients         default 1   sourcetype=unifi:client
collect_networks        default 1   sourcetype=unifi:network
collect_device_tags     default 1   sourcetype=unifi:device_tag
collect_firewall        default 1   sourcetype=unifi:firewall:zone, unifi:firewall:policy, unifi:acl_rule
collect_wifi            default 1   sourcetype=unifi:wifi:broadcast
collect_wan             default 1   sourcetype=unifi:wan
collect_pending_devices default 1   sourcetype=unifi:device:pending
collect_info            default 1   sourcetype=unifi:info
collect_vpn             default 0   sourcetype=unifi:vpn:server, unifi:vpn:tunnel
collect_switching       default 0   sourcetype=unifi:switching:lag, unifi:switching:mc_lag_domain, unifi:switching:switch_stack
collect_dns             default 0   sourcetype=unifi:dns:policy
collect_traffic_lists   default 0   sourcetype=unifi:traffic_matching_list
collect_radius          default 0   sourcetype=unifi:radius:profile
collect_vouchers        default 0   sourcetype=unifi:hotspot:voucher
```

**`unifi_telemetry`** (interval default 60s) — per-device collectors default **ON** (the user opted into telemetry by creating this input; stats is the whole point):

```
collect_device_stats    default 1   sourcetype=unifi:device:stats     (1 API call per device)
collect_device_detail   default 1   sourcetype=unifi:device:detail    (1 API call per device)
```

**`unifi_reference`** (interval default 86400s) — all default **ON** (only runs daily):

```
collect_countries          default 1   sourcetype=unifi:ref:country
collect_dpi_applications    default 1   sourcetype=unifi:ref:dpi_application
collect_dpi_categories      default 1   sourcetype=unifi:ref:dpi_category
```

### 6c. Why grouping replaces the old per-run-counter idea

The previous draft proposed a single input + internal `reference_interval_runs` counter. With native grouped inputs, **Splunk's own scheduler gives each cadence its real interval** — no custom timer, no checkpoint counter, simpler code, and the interval is a first-class UI field the user edits directly. The `solnlib` checkpoint is no longer needed for cadence (may still be used later for incremental state if ever required).

### 6d. Call volume estimate per group (this controller: 1 site, 13 devices)

Because each group runs on its own interval, request **rate** matters more than calls-per-poll.

**`unifi_inventory` @ 180s** (all default-on toggles):

| Work | Calls/run |
|------|-----------|
| sites discover | 1 |
| devices / clients / networks | 3 |
| device-tags | 1 |
| firewall zones+policies+acl (121÷200 +2) | 3 |
| wifi / wan | 2 |
| pending-devices / info | 2 |
| **per run** | **~12** → ~12 calls / 180s ≈ **0.07 req/s** |

**`unifi_telemetry` @ 60s** (stats+detail on, 13 devices):

| Work | Calls/run |
|------|-----------|
| sites discover | 1 |
| device list (to get IDs) | 1 |
| device_stats (13) | 13 |
| device_detail (13) | 13 |
| **per run** | **~28** → ~28 calls / 60s ≈ **0.47 req/s** |

**`unifi_reference` @ 86400s:** countries(2)+dpi_apps(11)+dpi_cats(1) = ~14 calls **once/day** ≈ negligible.

Per-device collectors dominate. Rate scales as `(2 + devices × enabled_per_device_collectors) / telemetry_interval`. On a 200-device site at 60s with both on that is ~402 calls/60s ≈ **6.7 req/s** — the user can raise the **telemetry interval** (it's a native field) to throttle. This is the main reason telemetry is its own input with its own interval.

---

## 7. Implementation phases — ✅ ALL DONE

| Phase | Version | Status | Delivered |
|-------|---------|--------|-----------|
| **A** — `unifi_inventory` input @180s | **3.0.0** | ✅ | Grouped input, 15 `collect_*` toggles (sourcetype-labelled + doc links). 16 inventory/config sourcetypes. Legacy `unifi_ingest` retained. |
| **B** — `unifi_telemetry` input @60s | **3.1.0** | ✅ | `unifi:device:stats` + `unifi:device:detail`, per-device error isolation, `unifi_device_id`/`unifi_device_mac` join keys. |
| — `_time` hardening | **3.1.1** | ✅ | `_time` = `unifi_collection_time` for snapshots (payload-first + lookahead 256); `lastHeartbeatAt` for stats. Fixed client/voucher drift. |
| **C** — network enrichment | **3.2.0** | ✅ | `collect_network_detail` toggle on Inventory → `unifi:network:detail` + `unifi:network:reference` (`ipv4Configuration` etc.). |
| **D** — `unifi_reference` input @86400s | **3.2.0** | ✅ | `unifi:ref:country` / `:dpi_application` / `:dpi_category`. |
| **E** — CIM / dashboards | **3.3.0** | ✅ | FIELDALIAS/EVAL CIM mapping, `eventtypes.conf` + `tags.conf` (Inventory/Network Sessions/Performance), 3 Simple-XML dashboards + nav. |

Each phase was independently built and tested in the Docker container against `192.168.1.1`. Native per-input intervals meant **no custom scheduler code** in any phase. Phases C and D were combined into the 3.2.0 build.

Final sourcetype count: **25** (4 original + 21 added).

---

## 7b. Configuration-page UX requirements (confirmed 2026-06-16)

The UCC Configuration/Inputs pages must be self-explanatory:

1. **Sourcetype in label + help** for every `collect_*` toggle. Example label: `Collect Firewall Policies (unifi:firewall:policy)`; help: `Polls /firewall/zones, /firewall/policies, /acl-rules. Sourcetypes: unifi:firewall:zone, unifi:firewall:policy, unifi:acl_rule. Docs: https://developer.ui.com/network/<ver>/getfirewallpolicies`.
2. **Interval field help** per input states the cadence intent: Inventory "How often to poll configuration/inventory lists. Default 180s."; Telemetry "How often to poll per-device performance stats. Default 60s — raise this on large fleets."; Reference "How often to refresh static reference data (countries, DPI). Default 86400s = daily."
3. **Hyperlinks to Ubiquiti docs** in help text. URL pattern: `https://developer.ui.com/network/<version>/<operationId-lowercased>` (e.g. `getadopteddeviceoverviewpage`, `getwifibroadcastpage`, `getfirewallpolicies`). Operation IDs per endpoint are in §3/`_ref/integration-oas.json`. **Note:** the docs site is a JS SPA — verify each anchor in a browser during build; if an anchor 404s, link the section root `https://developer.ui.com/network/`.
4. **App description / README** includes a **sourcetype reference table** (endpoint → sourcetype → input group → doc link) and a note to send all three inputs to the **same index**.
5. **Input-type descriptions** (UCC `description` on each service) explain what the group does and its default interval.

Operation-ID → endpoint map (for doc links), key ones:

| Sourcetype | operationId (doc anchor) |
|------------|--------------------------|
| `unifi:device` | `getadopteddeviceoverviewpage` |
| `unifi:device:detail` | `getadopteddevicedetails` |
| `unifi:device:stats` | `getadopteddevicelateststatistics` |
| `unifi:client` | `getconnectedclientoverviewpage` |
| `unifi:network` | `getnetworksoverviewpage` |
| `unifi:firewall:policy` | `getfirewallpolicies` |
| `unifi:firewall:zone` | `getfirewallzones` |
| `unifi:acl_rule` | `getaclrulepage` |
| `unifi:wifi:broadcast` | `getwifibroadcastpage` |
| `unifi:wan` | `getwansoverviewpage` |
| `unifi:vpn:server` | `getvpnserverpage` |
| `unifi:vpn:tunnel` | `getsitetositevpntunnelpage` |
| `unifi:dns:policy` | `getdnspolicypage` |
| `unifi:traffic_matching_list` | `gettrafficmatchinglists` |
| `unifi:radius:profile` | `getradiusprofileoverviewpage` |
| `unifi:hotspot:voucher` | `getvouchers` |
| `unifi:device_tag` | `getdevicetagpage` |
| `unifi:switching:lag` | `getlagpage` |
| `unifi:switching:mc_lag_domain` | `getmclagdomainpage` |
| `unifi:switching:switch_stack` | `getswitchstackpage` |
| `unifi:device:pending` | `getpendingdevicepage` |
| `unifi:info` | `getinfo` |
| `unifi:ref:country` | `getcountries` |
| `unifi:ref:dpi_application` | `getdpiapplications` |
| `unifi:ref:dpi_category` | `getdpiapplicationcategories` |

(Full list in `_ref/integration-oas.json`; the build will generate help strings from this table.)

---

## 8. Decisions (resolved — confirm or override)

| # | Question | Resolution |
|---|----------|------------|
| 1 | Per-list interval control? | **Per-group native interval** (Architecture B). 3 inputs: Inventory/Telemetry/Reference, each with its own Splunk `interval`. Related lists in a group share that interval; lists toggled with `collect_*`. |
| 2 | Default intervals? | Inventory **180s**, Telemetry **60s**, Reference **86400s** (daily). All user-editable in the UI; 300 is *not* hardcoded anywhere. |
| 3 | Ingest large reference lists (2112 DPI apps)? | Own input at daily interval; toggles default **on** there (only runs daily). |
| 4 | Per-device stats/detail call cost? | Their own **Telemetry** input @60s; toggles default **on** (user opted in by adding the input). Throttle via the input's interval field. |
| 5 | Device detail (ports/radios)? | Emit own sourcetype `unifi:device:detail` (don't merge — `interfaces` shape differs between list/detail). |
| 6 | Sourcetype visibility in UI? | In **both** the field label and the help text, plus README table. |
| 7 | Index? | **One index** for all UniFi data (documented). Each input keeps an index field; guidance = same index. |
| 8 | Existing `unifi_ingest` input? | **Keep for backward compatibility** + add the 3 new grouped inputs. Document migration to the grouped inputs. |
| 9 | Sentinel for global objects' site fields? | `unifi_site_id = _global`, `unifi_site_name = _global`. |
| 10 | Keep emitting 0-count endpoints? | **Yes** — empty envelope is clean (`data:[]`). |
| 11 | Use API `filter` param to trim volume? | **No** — undocumented property names, live test returns `400 invalid-filter`. Paginate only. |
| 12 | Set `_time` from API timestamps? | Where meaningful (stats→`lastHeartbeatAt`, client→`connectedAt`); else collection time. Decide per sourcetype in Phase E. |
| 13 | Flatten nested arrays (ports/radios)? | **No** at index time — keep in `_raw`, `spath` at search time; optional `*_count` summary fields. |

---

## 9. What does NOT change

- Auth (`X-API-KEY` via encrypted account), pagination envelope + `limit=200`, envelope fields, `_raw` strategy, UCC REST-handler structure (entrypoint fix from 2.0.1 stays).
- No write/POST/PUT/PATCH/DELETE endpoints (32 exist) — TA remains strictly read-only.
- Existing 4 sourcetypes keep their current shape; all additions are purely additive. The existing `unifi_ingest` input keeps working.

---

## 10. Reference artifacts

- Live OpenAPI spec: `_ref/integration-oas.json` (UniFi Network API 10.4.57, 44 paths).
- Pull command (for re-probing on upgrades):
  `curl -sk -H "X-API-KEY: <key>" https://<controller>/proxy/network/api-docs/integration.json`
- Doc-link pattern: `https://developer.ui.com/network/<version>/<operationId-lowercased>` (anchors to verify in browser at build).
- `device.state` enum changed vs old docs: now `ONLINE/OFFLINE/PENDING_ADOPTION/UPDATING/GETTING_READY/ADOPTING` (update sample SPL that used `ACTIVE`).

---

## 11. Status: complete

All phases implemented. Latest artifact: **`dist/TA_unifi_ng-3.3.0.tar.gz`**.
Release/install notes: **`08-release-notes.md`**.

Possible future work (not planned/required):
- Lookups generated from `unifi:ref:dpi_*` to enrich any DPI app/category IDs that appear in other data.
- A CIM acceleration / datamodel validation pass with the Splunk CIM app installed.
- Re-probe `_ref/integration-oas.json` on controller upgrades; the endpoint registry in `unifi_ingest.py` is the single place to add new lists.

---

## 12. Verification record (2026-06-16, controller 192.168.1.1, Splunk 10.2.3)

Final consolidated build **3.3.0**, fresh-installed into a clean index with all
inputs enabled:

| Check | Result |
|-------|--------|
| Version consistency (VERSION/app.conf/app.manifest/globalConfig) | ✅ all 3.3.0 |
| Python syntax (11 bin files) | ✅ all compile |
| restmap members = handlers = rh files | ✅ 6 = 6 = 6 |
| globalConfig services = inputs.conf stanzas | ✅ 4 = 4 |
| `btool` parse (props/inputs/restmap/web/tags/eventtypes/app/server) | ✅ no errors |
| props.conf unique sourcetype stanzas | ✅ 25, no duplicates |
| Tarball hygiene (no `._*`/`.DS_Store`/`__pycache__`/`.pyc`, single top dir) | ✅ clean |
| All 6 REST handlers respond | ✅ 200 (settings bare-list 500 is a known UCC quirk; UI uses `/logging`) |
| Sourcetypes ingesting (clean run, all inputs) | ✅ 23 with data; the rest are 0-item on this controller |
| CIM tags (inventory/session/performance/network) | ✅ populated |
| CIM fields (device/client/stats models) | ✅ PASS |
| `_time` drift (snapshot sourcetypes) | ✅ max 2.8 s |
| Dashboards render | ✅ 3 × HTTP 200, panels return data |

Notes:
- 0-item sourcetypes on this controller: `acl_rule`, `dns:policy`,
  `switching:mc_lag_domain`, `switching:switch_stack` (collectors work; the
  lists are simply empty here — other controllers will populate them).
- The `data/inputs/<type>/<name>` REST path is the reliable way to delete
  inputs (the UCC handler's DELETE has the reserved-`disabled`-field quirk).

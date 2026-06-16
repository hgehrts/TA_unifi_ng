# UniFi TA NG — Event Schema

## Sourcetypes and index

| Sourcetype | Index (default) | When emitted |
|------------|-----------------|--------------|
| `unifi:site` | `unifi` | Once per site from `/sites` |
| `unifi:device` | `unifi` | Each object in `/sites/{id}/devices` pages |
| `unifi:client` | `unifi` | Each object in `/sites/{id}/clients` pages |
| `unifi:network` | `unifi` | Each object in `/sites/{id}/networks` pages |

Override index per stanza: `index = unifi` in `inputs.conf`.

## Envelope fields (every event)

| Field | Description | Example |
|-------|-------------|---------|
| `unifi_host` | Controller base URL | `https://192.168.1.1` |
| `unifi_site_id` | Site UUID | `88f7af54-98f8-306a-a1c7-c9349722b1f6` |
| `unifi_site_name` | Site display name | `Default` |
| `unifi_object_type` | `site` \| `device` \| `client` \| `network` | `device` |
| `unifi_collection_time` | **Splunk pull time** (UTC ISO-8601) | `2026-06-04T12:00:00Z` |

`unifi_collection_time` is set at event emission time in the modular input (not from UniFi).

## API timestamp mapping

Rule: for each top-level field on the API object whose name suggests a time:

- Suffix `At`, `Time`, or contains `connected`, `updated`, `seen` (case-insensitive) → copy to Splunk as `unifi_api_<original_field_name>`.

Known mappings from probe:

| Object type | API field | Splunk field |
|-------------|-----------|--------------|
| client | `connectedAt` | `unifi_api_connectedAt` |

Devices (v1 probe): no top-level time fields — only envelope `unifi_collection_time`.

Networks (v1 probe): no top-level time fields; `metadata` object preserved as JSON string field `unifi_api_metadata` if present.

Nested times inside `interfaces[]` / `access` remain in optional `raw` JSON or flattened only if needed later.

## Event body strategy

- **Searchable fields:** envelope fields + common identifiers (`id`, `name`, `macAddress`, `ipAddress`, `model`, `state`, `type`, `vlanId`, etc.) extracted at top level where flat.
- **`_raw`:** full JSON of the API object for completeness.
- Avoid duplicating huge nested arrays in indexed fields unless required (e.g. `features` as JSON string optional).

## Example SPL

```spl
index=unifi sourcetype=unifi:client
| stats latest(unifi_collection_time) as last_pull by macAddress, unifi_site_name
```

```spl
index=unifi sourcetype=unifi:device state=ACTIVE
| table unifi_site_name name model ipAddress macAddress firmwareVersion
```

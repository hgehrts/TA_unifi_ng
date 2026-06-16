# UniFi TA NG — Architecture Notes

## Approved behavior (2026-06-04)

- Discover sites: `GET /integration/v1/sites` → iterate every returned `siteId`.
- Per site: paginated GET for devices, clients, networks (toggles respected).
- Splunk **10**, index **`unifi`**, stanza **`interval`** user-configurable (default 300s).

## Component list

| Component | Location (UCC layout) |
|-----------|------------------------|
| Modular input wrapper | `bin/unifi_network.py` (generated) |
| Collection logic | `bin/unifi_ingest.py` (hand-written, preserved on rebuild) |
| REST / UI | `appserver/`, `appserver/static/openapi.json` |
| Config | `default/inputs.conf`, `default/props.conf`, `metadata/` |
| Package output | `output/TA_unifi_ng-*.tar.gz` |

## HTTP client

Prefer **stdlib** (`urllib.request`) to avoid shipping `requests` unless needed for ergonomics.

- Custom SSL context when `verify_ssl=false`
- Timeout: connect 10s, read 120s (clients can be large)
- User-Agent: `TA-unifi-ng/<version>`

## Error handling

| HTTP | Behavior |
|------|----------|
| 401/403 | Log error, skip cycle or fail stanza (configurable) |
| 429 | Sleep with exponential backoff |
| 5xx | Retry up to 3 times per request |
| Timeout | Log and continue next site/resource |

## Checkpointing (phase 5)

Use `solnlib` `checkpointing` to store per `(host, site_id, resource)` last `offset` or last `totalCount` hash — only if you need incremental efficiency. For inventory snapshots, full poll each interval is acceptable for v1.

## Dashboard

UCC can generate a monitoring dashboard stub; customize to show:

- Events/min per sourcetype
- Last poll duration
- API errors count

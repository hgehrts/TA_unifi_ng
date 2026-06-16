# UCC sources

These are the inputs for the [Splunk UCC framework](https://github.com/splunk/addonfactory-ucc-generator)
(`ucc-gen`) used to generate the `TA_unifi_ng/` app.

```
src/
├── globalConfig.json     # UI definition: account + the 4 inputs and their fields
├── package/              # custom code & conf merged into the generated app
│   ├── bin/              # modular-input scripts + REST handlers + shared unifi_ingest.py
│   ├── default/          # props.conf, eventtypes.conf, tags.conf, data/ui (dashboards, nav)
│   ├── lib/              # requirements.txt for bundled Python deps
│   └── static/, metadata/, app.manifest, README.md, CHANGELOG.md
```

## Regenerate the app

```bash
pip install splunk-add-on-ucc-framework
cd src
ucc-gen build --source package --config globalConfig.json -o ../output
# the generated app appears at ../output/TA_unifi_ng
```

Then package it (from the repo root):

```bash
# copy the freshly generated app over the committed one, review the diff, then:
./build.sh
```

## Where the logic lives

- **`package/bin/unifi_ingest.py`** — shared collection layer: the `UniFiClient`
  HTTP client, the **endpoint registry** (`INVENTORY_ENDPOINTS`,
  `REFERENCE_ENDPOINTS`), event building, and the `collect_inventory_events` /
  `collect_telemetry_events` / `collect_reference_events` generators.
  **This is the single place to add new API lists.**
- **`package/bin/unifi_inventory.py` / `unifi_telemetry.py` / `unifi_reference.py`** —
  the three modular-input scripts (each a thin wrapper around the collectors).
- **`package/bin/TA_unifi_ng_rh_*.py`** — UCC REST handlers for the config UI.
  Each persistent handler **must** end with
  `if __name__ == "__main__": admin_external.handle(...)`.
- **`package/default/props.conf`** — parsing + CIM field aliases per sourcetype.
- **`package/default/tags.conf` / `eventtypes.conf`** — CIM data-model membership.

## Re-probing the API after a controller upgrade

The committed OpenAPI snapshot is in [`../_ref/integration-oas.json`](../_ref/integration-oas.json).
Refresh it with:

```bash
curl -sk -H "X-API-KEY: <key>" \
  https://<controller>/proxy/network/api-docs/integration.json -o ../_ref/integration-oas.json
```

# Splunkbase submission — UniFi Network Add-on for Splunk

Use this as a copy-paste guide when submitting at [Splunkbase Developer Portal](https://splunkbase.splunk.com/). Full listing text lives in [`../splunkbase.md`](../splunkbase.md).

---

## Listing metadata (form fields)

| Field | Value |
|-------|-------|
| **App / package name** | UniFi Network Add-on for Splunk |
| **Package ID** | `TA_unifi_ng` |
| **Type** | Add-on |
| **Version** | 3.3.1 |
| **License** | Apache-2.0 |
| **Support model** | Developer Supported |
| **Author / Created by** | Hans-Henning Gehrts |
| **Contact email** | hgehrts@splunk.com |
| **Source code URL** | https://github.com/hgehrts/TA_unifi_ng |
| **Categories** | Network, IT Operations |
| **Splunk compatibility** | Enterprise 8.0+, 9.x, 10.x (tested on 10.2.3) |
| **Splunk Cloud** | Expected compatible — submit AppInspect cloud report |

## Package to upload

Download from GitHub Release (do not upload from `local/`):

- https://github.com/hgehrts/TA_unifi_ng/releases/download/v3.3.1/TA_unifi_ng-3.3.1.tar.gz

## Icons (upload in listing editor)

| Asset | File |
|-------|------|
| Icon 200×200 | `assets/listing_icon_200.png` |
| Icon 400×400 | `assets/listing_icon_400.png` |
| Screenshot | `assets/screenshot.png` *(capture before submit — see assets/README.md)* |

## AppInspect

Run before upload and attach the HTML/JSON report to the submission (or paste summary in notes):

```bash
splunk-appinspect inspect dist/TA_unifi_ng-3.3.1.tar.gz \
  --included-tags cloud,private --mode precert \
  --output-file docs/appinspect-v3.3.1.json
```

**Requirement:** 0 errors, 0 failures (warnings may need explanation).

## Short description (≤250 chars)

```
Read-only Splunk add-on that polls the UniFi Network Integration API v1 and indexes inventory, configuration, per-device telemetry and reference data into Splunk. 25 sourcetypes, CIM-mapped, cadence-grouped inputs.
```

## Summary (listing page)

Copy from **Summary** section in [`splunkbase.md`](../splunkbase.md).

## Details / Installation / Troubleshooting

Copy the matching sections from [`splunkbase.md`](../splunkbase.md).

## Reviewer notes (optional, paste in submission comments)

- Read-only Integration API v1 collector (`GET` only); API keys stored encrypted via UCC.
- Companion visualization app submitted separately: `unifi_app_for_splunk`.
- Not affiliated with Ubiquiti Inc.
- Requires local UniFi Network controller with Integration API enabled (not UniFi Cloud-only).

## After approval

- Add Splunkbase badge to README: `https://splunkbase.splunk.com/app/XXXX` (replace with assigned app ID).
- Link companion app listing once both are live.

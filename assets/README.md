# Splunkbase & README assets

Files in this folder are for **GitHub README** and **Splunkbase listing** — not shipped inside the Splunk app package.

## Required before Splunkbase submit

| File | Size | Purpose |
|------|------|---------|
| `listing_icon_200.png` | 200×200 | Splunkbase listing icon |
| `listing_icon_400.png` | 400×400 | Splunkbase listing icon (large) |
| `screenshot.png` | ~1200–1600 px wide | Hero screenshot (README + Splunkbase) |
| `screenshot-overview.png` | optional | Additional Splunkbase gallery image |
| `screenshot-config.png` | optional | Configuration / inputs page |

## Starting point

Copy and resize from the in-app icons already in the package:

```bash
cp ../TA_unifi_ng/static/appIcon_2x.png listing_icon_400.png
# Resize to 200×200 for listing_icon_200.png (Preview, ImageMagick, etc.)
```

## Screenshots to capture

1. **Configuration → Account** — controller URL + API key saved (redact key).
2. **Data inputs** — Inventory / Telemetry / Reference enabled.
3. **UniFi - Overview** dashboard with live data.
4. **UniFi - Device Health** with telemetry charts.

Use Splunk light or dark theme consistently. Avoid customer hostnames in screenshots — use lab names.

## Trademark

Do not use official Ubiquiti or Splunk corporate logos unless you have written permission. The bundled app icons are generic/neutral placeholders.

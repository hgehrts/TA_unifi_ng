# Publishing checklist — UniFi TA + App

Goal: make **TA_unifi_ng** and **unifi_app_for_splunk** publicly available on GitHub and Splunkbase, following the same pattern as [splunk-whiteboard-app](https://github.com/bautt/splunk-whiteboard-app) / [Splunkbase app 8908](https://splunkbase.splunk.com/app/8908).

## Current state (audit)

| Item | TA_unifi_ng | unifi_app_for_splunk |
|------|-------------|----------------------|
| Local repo ready | ✅ `/Users/hgehrts/Projekte/TA_unifi_ng_repo` | ✅ `/Users/hgehrts/Projekte/unifi_app_for_splunk` |
| GitHub remote | `hgehrts/TA_unifi_ng` | `hgehrts/unifi_app_for_splunk` |
| Public on GitHub | ❌ 404 (private or not pushed) | ❌ 404 (private or not pushed) |
| LICENSE | ✅ Apache-2.0 | ✅ Apache-2.0 |
| README | ✅ Complete | ✅ Complete (fix TA link before publish) |
| Build script + dist tarball | ✅ v3.3.1 | ✅ v1.2.0 |
| GitHub Actions CI/release | ✅ `.github/workflows/build-release.yml` | ❌ Missing |
| Git tag / GitHub Release | ⚠️ Tag `v3.3.0` but VERSION is 3.3.1 | ❌ No tags |
| App icons (in-app) | ✅ `static/appIcon*.png` | ✅ `static/appIcon*.png` |
| Splunkbase listing icons (200/400) | ❌ Not in `assets/` | ❌ Not in `assets/` |
| Product screenshots | ❌ | ❌ |
| `splunkbase.md` listing copy | ✅ This repo | ✅ Companion repo |
| AppInspect validation | ⬜ Not recorded | ⬜ Not recorded |
| Splunkbase listing | ⬜ Not submitted | ⬜ Not submitted |
| Cisco-internal content scrubbed | ✅ No Cisco URLs in source | ✅ |

## Phase 1 — GitHub (public source + releases)

### 1.1 Choose hosting

- **Recommended:** Personal GitHub (`hgehrts`) — matches Tomas Baublys' model; repos are already configured.
- **Alternative:** New public org (e.g. `splunk-community`) if you want separation from your personal account.
- **Not recommended for public Splunkbase:** Cisco internal GitHub (`github.cisco.com`) — Splunkbase requires a **public source URL** for community listings.

### 1.2 Make repositories public

For each repo on GitHub → **Settings → General → Danger Zone → Change visibility → Public**.

Or create fresh public repos and push:

```bash
# TA
cd /Users/hgehrts/Projekte/TA_unifi_ng_repo
git push -u origin main

# App
cd /Users/hgehrts/Projekte/unifi_app_for_splunk
git push -u origin main
```

### 1.3 Cross-link READMEs

After public URLs are confirmed, ensure both READMEs link to each other:

- TA README → `https://github.com/hgehrts/unifi_app_for_splunk`
- App README → `https://github.com/hgehrts/TA_unifi_ng`

### 1.4 Cut GitHub Releases (required for Splunkbase "Source code" link)

```bash
# TA — align tag with VERSION file (3.3.1)
cd /Users/hgehrts/Projekte/TA_unifi_ng_repo
./build.sh
git tag -a v3.3.1 -m "Release 3.3.1"
git push origin v3.3.1
# CI attaches dist/TA_unifi_ng-3.3.1.tar.gz to the Release

# App
cd /Users/hgehrts/Projekte/unifi_app_for_splunk
./build.sh
git tag -a v1.2.0 -m "Release 1.2.0"
git push origin v1.2.0
```

### 1.5 Add listing assets to both repos

Create `assets/` (see `assets/README.md`):

- `listing_icon_200.png` — Splunkbase icon
- `listing_icon_400.png` — Splunkbase icon (large)
- `screenshot.png` — hero screenshot for README + Splunkbase

Copy from in-app icons as a starting point; refine before Splunkbase submit.

## Phase 2 — Splunkbase (two listings)

Submit **two separate listings** (Tomas published one app; you have a TA + companion app):

| Listing | Type | Package ID | Suggested title |
|---------|------|------------|-----------------|
| 1 | **add-on** | `TA_unifi_ng` | UniFi Network Add-on for Splunk |
| 2 | **app** | `unifi_app_for_splunk` | UniFi App for Splunk |

### 2.1 Prerequisites

1. [Splunkbase developer account](https://splunkbase.splunk.com/) (Splunk login).
2. Run **AppInspect** on each tarball and save the report:

```bash
# Install splunk-appinspect if needed: pip install splunk-appinspect
appinspect inspect dist/TA_unifi_ng-3.3.1.tar.gz --included-tags cloud,private
appinspect inspect dist/unifi_app_for_splunk-1.2.0.tar.gz --included-tags cloud,private
```

Fix any **ERROR**-level findings before submit. **WARNING** items may need justification in the submission notes.

3. Prepare listing content from each repo's **`splunkbase.md`** (Short Description, Summary, Details, Installation, Troubleshooting).

### 2.2 Submit TA first

Splunkbase flow (Developer → Submit New App):

- **Type:** Add-on
- **License:** Apache-2.0
- **Support:** Developer Supported
- **Source code URL:** `https://github.com/hgehrts/TA_unifi_ng`
- **Categories:** Network, IT Operations
- **Compatibility:** Splunk 9.x, 10.x (match `app.manifest`)
- Upload tarball from GitHub Release or `dist/`
- Attach AppInspect report

### 2.3 Submit companion app second

- **Type:** App
- **Prerequisite note in listing:** Requires `TA_unifi_ng` + SC4S syslog for full functionality
- **Optional dependency:** Network Diagram Viz (Topology dashboard only)
- Same license/support/source pattern

### 2.4 Post-approval

- Add Splunkbase badge/links to both READMEs.
- Update `app.conf` / manifest if Splunkbase assigns a listing ID.
- Announce on Splunk Community (optional).

## Phase 3 — Splunk employee / legal (if applicable)

Because commits use `@splunk.com` email:

- Confirm your manager / open-source policy allows **personal GitHub** publication (not Cisco corp repo).
- Apache-2.0 is fine for Splunkbase; no Splunk proprietary code should be in the package (verify: ✅ community TA pattern).
- Trademark disclaimer is already in README (Ubiquiti / Splunk).
- Do **not** use Ubiquiti official logos unless you have vendor approval (use generic/neutral app icons).

## Phase 4 — Differentiation vs existing Splunkbase UniFi apps

| Existing | What yours adds |
|----------|-----------------|
| [Ubiquiti Add-on (4107)](https://splunkbase.splunk.com/app/4107) | Syslog parsing only; no Integration API |
| [UniFi Cloud Add-on (7494)](https://splunkbase.splunk.com/app/7494) | Cloud API beta; limited scope |
| SC4S Ubiquiti sources | Syslog ingest only; cryptic MACs |

**Your positioning:** First open-source Splunk solution for the **UniFi Network Integration API v1** (local controller), plus a companion app that **correlates API asset data with SC4S syslog** and ships Dashboard Studio RCA views.

## Quick reference — Tomas Baublys pattern

What made [splunk-whiteboard-app](https://github.com/bautt/splunk-whiteboard-app) successful as a public listing:

1. Public GitHub with clear README + screenshots
2. `splunkbase.md` with copy-paste listing sections
3. `assets/listing_icon_200.png` + `listing_icon_400.png`
4. MIT/Apache license declared in repo and Splunkbase
5. GitHub Release tarball attached to tags
6. Developer Supported + GitHub Issues for support
7. Source code URL on the Splunkbase listing

You already match most of this structurally; the main gaps are **public GitHub**, **listing assets/screenshots**, **AppInspect**, and **Splunkbase submission**.

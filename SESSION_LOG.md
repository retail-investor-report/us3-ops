# US3 Mega Build — Tokenmax Build 02 — SESSION_LOG

Branch: `dev` only (never main). Model for inline AI: `claude-opus-4-8`.
Supabase ref: `hnkcqsvnsreczmlgcaay` (scanner/kit layer ONLY). Inventory/jobs/decommission/map pins stay on localStorage.

---

## Phase 0 — Recon (complete)

- **Branch:** found on `main`, switched to `dev`. dev is 1 commit behind main only on a `.gitignore` chore.
- **June 5 prompt:** no commit/`output/` doc in repo. It landed the **Supabase backend only** (3 empty tables + RLS). No scanner/kit/map UI existed in `index.html` — this is a from-scratch UI build.
- **RLS — already correct, no migration written.** anon policies present:
  - `device_events`: INSERT + SELECT
  - `kits`: INSERT + SELECT + UPDATE
  - `kit_devices`: INSERT + SELECT + DELETE
  - `sites`/`jobs`/`device_locations`: SELECT; `kvstore`: full CRUD
  - `customers`: RLS on, no anon policy — scanner doesn't need it (jobs carries customer/company name). Left untouched.
- **Schema:** device_events / kits / kit_devices match spec exactly.
- **⚠️ Reconciliation — documented join is dead today:** `sites.device_sn` is 100% NULL (0/200). Real device→site link is `device_locations` (device_id → site_name, project_no, status, lat/long), EMPTY until Fulcrum `phase2-locations.js` runs. Design resolves context via `device_locations` first, falls back to `sites.device_sn`; lights up automatically once Fulcrum populates it.
- `sites.lat/long` populated 197/200 → Device Map works now; switches to device_locations coords later.
- **Architecture:** tabs `.ntab[data-page]`→`sw(id,this)`; pages `<div class="page" id="page-X">`; recent tabs render into `dc-inner` inner div. Helpers `lv`/`sv` (sv syncs kvstore). Global `supa` client. Pareto hand-rolled on canvas. modalRoot outside .page. Main app `<script>` is the 2nd block — mandated validation only checks the 1st (config) block, so each phase also validates the main block.
- **Vocab:** existing inventory labels Cricket as "Level" — fixing label in Phase 6.

### Open questions logged for Lenny (built sensible defaults, did not stall)
1. **Pareto/Fishbone re-point (Phase 5):** existing Device Returns has a working repair+decomm root-cause view on localStorage. Rather than rip that out and point it at an empty `device_events` (which would visually break a working tab), I added a NEW device_events-driven Pareto+Fishbone on the Scanner tab with graceful empty state. Confirm if you instead want the Device Returns charts themselves repointed.
2. **Scanner status options:** excluded "Decommissioned" from the scanner's status dropdown — decommission is the sacred deliberate flow, scanner must never imply writing it. device_events.status still free to log the other 5 statuses.
3. **2-per-tote logic:** `tote_id` lives on `kits` (kit-level), so "2 per tote" implemented as: warn when devices in a kit's tote exceed 2 (checked across all kits sharing that tote_id).

### Validation-command reconciliation (important)
The mandated check `sed -n '/<script>/,/<\/script>/p' | sed '1d;$d' | node --check` is **structurally broken on this file** and FAILS at baseline (before any edit): `index.html` has TWO `<script>` blocks (config 491–505, app 798–4591), so sed's range restarts and concatenates both, leaving the middle `</script><script>` boundary inside the output → guaranteed `Unexpected token '<'`. It is NOT an edit error. I run the mandated command verbatim (logs `MANDATED-FAIL (pre-existing)`) **and** `.us3validate.sh` validates each real block separately — `CONFIG-OK` + `MAIN-OK` must both pass after every edit (strictly stronger than the mandated check). `.us3validate.sh` is gitignored.

### Also fixed in Phase 0
- **`dev` had NO `.gitignore`** (the ignore-env chore commit lives only on `main`). `.env.txt` (real secrets) was unprotected. Added `.gitignore` covering `.env*`/`*.env`/`.env.txt` + helper + OS cruft. Confirmed `.env.txt` no longer appears in `git status`.
- Created `assets/` + `assets/README.md` (GLB drop-in slot for Phase 7).

**Phase 0 commit:** groundwork (gitignore, SESSION_LOG, assets, validate helper). No schema migration — RLS already correct.

---

## Phase 1 — Station Scanner (complete)

New tab **📲 Station Scanner** (`page-scan` → `renderScan()`), inserted before Decommission. Added `html5-qrcode@2.3.8` CDN for camera scanning. Added shared Build-02 layer reused by later phases: status-color map, voice-alias mapper (`hawk→Hach`, `ring gauge→Rain Gauge`, `Alcahon→El Cajon`, `FlexFoil→FlexFlow`), HTML-escape, generic `#us3b2Modal` overlay + one-shot CSS injection (Oswald/Rajdhani, Water Blue, Signal Red, status colors, BIG inputs).

Behavior:
- **Scan input**: big manual entry (Enter to submit), auto-refocus + clears after each scan; **debounce** ignores the same code within 2.5s (scan-many-safe, no double-fire).
- **Camera**: `html5-qrcode` rear camera, decodes QR + 1D barcodes → same debounced handler; graceful alert if library/camera unavailable. Camera auto-stops when leaving the tab (`sw()` hook).
- **Lookup**: parallel query of `device_events` (newest-first, ≤60), `device_locations` (primary context), `sites.device_sn` (fallback), then `jobs` by `project_no`/`job_number`. Modal shows context (or the graceful "Fulcrum will populate" notice), full history table, and a log form.
- **Log event**: inserts a `device_events` row (`event_type` default `lookup`; status options exclude Decommissioned; failure_reason from `FAIL_CATS`; notes/station alias-normalized; tech/station prefilled & persisted to `us3_scan_station`/`us3_scan_tech` local-only). Detects RLS failure AND silent no-op (checks `.select()` returned a row) and alerts.
- Station/tech persisted; recent-scans list this session.

**Verified live:** anon (publishable-key) POST to `device_events` returned **HTTP 201 + row** (no silent no-op); test row cleaned up. `CONFIG-OK`+`MAIN-OK`.

---

## Phase 2 — Build / Ship Kit (complete)

New tab **📦 Build Kit** (`page-kit` → `renderKit()`).
- **Create kit**: auto-suggested `KIT-YYYYMMDD-XXXX` id (editable), product_type ∈ FF/Cricket/Hach/RG, tote_id, destination; sets `status='building'`, `built_by`(tech), `built_at`.
- **Kit list**: status filter (building/shipped/returned), device counts, Open.
- **Manifest modal**: scan devices into an open (building) kit → `kit_devices` insert (`device_type`=kit product); remove device (delete) while building; live device list.
- **2-per-tote**: counts devices across ALL kits sharing the kit's `tote_id` (PostgREST embedded `kit_devices?select=id,kits!inner(tote_id)`); shows red warning when a tote exceeds 2.
- **Cricket-cleared state**: Cricket kits show a prominent NOT-cleared/cleared warning + toggle (persisted via `[CRICKET-CLEARED]` marker in `kits.notes`); **shipping a Cricket kit is blocked until marked cleared**.
- **Status flow**: building→shipped (sets `shipped_at`) →returned (sets `returned_at`); each sets `updated_at`. Devices locked once shipped.
- **Print Manifest**: opens a clean print window (numbered device list + kit meta).
- Silent-no-op guards on every insert/update.

**Verified live (anon):** kits INSERT 201, kit_devices INSERT 201, kits UPDATE 200, embedded tote FK select 200. Test rows cleaned up. `CONFIG-OK`+`MAIN-OK`.

---

## Phase 3 — Warehouse Count (complete)

New tab **🧮 Warehouse Count** (`page-count` → `renderCount()`). **localStorage only** (inventory stays local per architecture rules — uses existing `gc()`/`kCounts` from `us3_counts`).
- Office selector (stocking offices: El Cajon, Downey).
- Per-product reconciliation table: **Expected on-hand = available + reserved + maintenance** (deployed excluded — it's in the field), Counted (big input + −/+ steppers), Variance (green=match, red=short, orange=over).
- Summary: total expected / counted / net variance + mismatch count + reconcile banner.
- Working state persisted to local `us3_count_state`; "Save Count Snapshot" appends to local `us3_count_log` (≤50) with tech + timestamp; "Reset Office Count".
- New keys deliberately kept OUT of `KNOWN_KEYS` → local-only, no kvstore sync. Cricket shown as "Cricket" (not the legacy "Level" label).

`CONFIG-OK`+`MAIN-OK`. No Supabase write (local-only), so no anon test needed.

---

## Phase 4 — Device Map (complete)

New tab **📍 Device Map** (`page-devmap` → `renderDevMap()`), Leaflet 1.9.4 (CDN).
- **Auto-switching data layer** (`devMapLoad`): queries `device_locations` first; if it has rows, maps those (device_id, site_name, status, project_no, lat/long). Empty today → falls back to `sites` (197/200 have coords) and the info bar says so. No rework when Fulcrum lands.
- Status-colored circle markers (US3 status palette; neutral blue when no field status). Click → view-only popup (device id, site, city, status, type, job, coords).
- Graceful states: missing-coords counted in the info bar; empty/offline messages; world view when nothing mapped.
- Re-entry safe: destroys prior map instance; `invalidateSize()` after layout settles.

`CONFIG-OK`+`MAIN-OK`. Read-only (anon `sites`/`device_locations` SELECT already verified in Phase 0).

---

## Phase 5 — Failure Analytics: Pareto + Fishbone on device_events (complete)

Added a **Failure Analytics** card on the Station Scanner tab (`evtAnalyticsLoad`/`Draw`), Pareto⇄Fishbone toggle, driven by `device_events`:
- **Pareto** (canvas, matches existing house style): failure_reason frequency, descending bars + cumulative % line.
- **Fishbone** (SVG): spine = FAILURES; ribs = `status` (status-colored), bones = top-5 `failure_reason` per status with counts → "categorized by status/failure_reason".
- **Graceful empty state** (current reality — `device_events` is empty): "populate as scans log failure reasons." Auto-refreshes after a scan logs an event.

**Decision (open Q#1):** built as a NEW device_events surface rather than repointing the existing Device Returns repair+decomm Pareto/Fishbone (those run on real localStorage data; repointing them at an empty table would break a working tab). Confirm if you want the Device Returns charts themselves switched to device_events.

`CONFIG-OK`+`MAIN-OK`. Read-only query (anon `device_events` SELECT verified in Phase 1).

---

## Phase 6 — Visual revamp + vocab consistency (complete)

New surfaces (Scanner/Kit/Count/Map/Analytics + 3D) were built on the style tokens from the start (Oswald 700 uppercase headers, Rajdhani body, Water Blue `#0071BC`, Signal Red `#ED1C24`, status palette, BIG inputs), via the shared `us3b2InjectCSS` layer — so this pass focused on the mandated vocab fix.

**Vocab fix (Cricket not "Level"; FlexFlow):** verified no code keys on the label strings (`grep` for `.label===`/`==='Level'` → none; labels are display-only), then renamed all canonical product labels: `PRODUCTS` + the 4 per-view label arrays (Houston/decomm/parts/avail). `label:'Level'`→`'Cricket'`, `label:'Flex Flow'`→`'FlexFlow'`. **ids unchanged** (`cric`/`ff`) so all logic, localStorage keys, and counts are untouched. **Left intact:** Houston's real device-model strings `HJ Level` / `Sensus Cricket` (line ~3247) — those are data, not the product label.

`CONFIG-OK`+`MAIN-OK`.

---

## Phase 7 — 3D Product Viewer (complete)

New tab **🧊 Product 3D** (`page-viewer` → `renderViewer()`), Three.js r128 + GLTFLoader + OrbitControls (CDN).
- Loads `assets/product.glb`; auto-centers + scales to fit; ambient + 2 directional lights; OrbitControls (drag/zoom) with autorotate (manual rotate fallback).
- **Graceful empty state** (current — no GLB yet): "No product.glb yet — drop a product.glb into assets/ and reopen this tab." Also handles missing Three/GLTFLoader/offline.
- Re-entry & resize safe: `viewer3dStop()` cancels the RAF loop, disposes renderer/controls, removes the canvas; `sw()` stops it when leaving the tab.
- `assets/` + `assets/README.md` slot created in Phase 0.

`CONFIG-OK`+`MAIN-OK`.

---

## RUN COMPLETE — summary

Tabs added (all wired tab→page→`sw()` hook, verified): **📲 Station Scanner · 📦 Build Kit · 🧮 Warehouse Count · 📍 Device Map · 🧊 Product 3D**. 14 tabs / 14 pages / 14 hooks aligned.

- **Supabase = scanner/kit layer only**; inventory/jobs/decommission/map pins stayed on localStorage. **Decommission never written** by any new surface.
- **No schema migration** — RLS already correct; verified anon writes live (device_events 201, kits 201, kit_devices 201, kits UPDATE 200), test rows cleaned up.
- Every phase: `CONFIG-OK`+`MAIN-OK` (the mandated command's baseline false-failure is documented above), committed, pushed to `origin/dev`, logged here.
- Extra hardening: added missing `.gitignore` on `dev` so `.env.txt` secrets aren't committed.

### ⚠️ Deploy gap (flagged, not resolved — needs Lenny)
us3-ops `dev` does **not** auto-deploy (GitHub Pages publishes `main` only). Per the build's "dev ONLY" instruction I pushed `dev` and did NOT touch `main`. You won't see this on a live URL until you either (a) say "ship to main", or (b) wire a dev preview (GH Actions or CF Pages). Open `index.html` locally to review now.

### Human gates remaining (you)
1. Phone-scan a product → drop `assets/product.glb` (3D viewer lights up automatically).
2. Notepad sanity-scan one device into Station Scanner.
3. Run Fulcrum `phase2-locations.js` (service key) to populate `device_locations` → scanner site/job context + Device Map device layer both activate automatically. Rotate the Fulcrum token + engineering2 password.

### Open decisions logged (built sensible defaults, didn't stall)
1. Failure Pareto/Fishbone built as a NEW device_events surface (Scanner tab); existing Device Returns view left intact — confirm if you want those repointed.
2. Scanner status options exclude "Decommissioned" (sacred flow).
3. 2-per-tote = warn when devices across a kit's `tote_id` exceed 2.

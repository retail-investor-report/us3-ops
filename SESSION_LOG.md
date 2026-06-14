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

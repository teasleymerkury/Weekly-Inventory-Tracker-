# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a single-page inventory forecasting tool for Merkury Innovations' Home Depot lighting products. It is a self-contained HTML application — no build step, no package manager, no framework. Open `index.html` directly in a browser to run it.

## File Structure

- **`index.html`** — the entire application (CSS, HTML, JS in one ~6000-line file)
- **`Weekly_Inventory_Recap.html`** — kept **identical** to `index.html` at all times (a mirror copy); any change to one must be applied to the other

The comment on line 1 of both files is the canonical reminder: `<!-- MIRROR: This file and Weekly_Inventory_Recap.html are kept identical. -->`

## Architecture

### Data Model

- **`products`** (const array, ~line 1461): hardcoded SKU catalog. Each entry has `model`, `sku`, `desc`, `inner`, `master`, `onHandMerkury`, `onHandTHD`, `firmReceipts[]`, `salesRates[]`.
- **`state`** (object keyed by SKU, ~line 1504): runtime mutable data. Extended from `products` at startup. Fields include `firmReceipts`, `reorders`, `reorderData`, `salesRates`, `salesRatesManual`, `posUnits`, `onHandMerkury`, `additionalModels`, `suofFromExcel`, `atsConnected`, `vdConnected`, and others.
- **`weeks`** (const array, ~line 1384): the full fiscal-week schedule used as column indices throughout the app. Array index (`week_idx`) is the universal positional key.

### Persistence Layers (in priority order)

1. **Supabase** (source of truth for real users): tables `sku_meta`, `sku_weekly`, `app_cache`; storage bucket `sync-files` for raw Excel files.
2. **localStorage** (fallback / cache): key `merkury_inv_v3` for general state, plus separate keys for inbound edits, sent reorders, cp colors, file names.

On login, `loadFromSupabase()` pulls all data and overlays it on the in-memory state. `saveState()` persists to localStorage; `saveStateSupabase()` upserts to Supabase.

### Excel Sync Sources

Three external Excel files power live data:
- **SUOF** (`excelFile`) — THD's SUOF Report; parsed by `syncFromExcel()`; columns mapped by header name search
- **ATS** (`atsFile`) — Merkury's ATS Report (inbound/firm receipt data); parsed by `syncFromATS()`
- **VendorDrill** (`vdFile`) — VD sales data; parsed by `syncFromVD()`

All three parsers use a local `findCol(...terms)` helper to locate columns by fuzzy header matching. Files are also stored to Supabase Storage (`sync-files` bucket) so other devices auto-load them on login via `autoLoadSyncFiles()`.

### Pages / Navigation

Three pages toggled by `showPage(page)`:
- **`inventory`** — main product grid; one `product-block` per SKU built by `buildProduct(p)`
- **`reorder`** — reorder summary; rendered by `renderReorderTab()`
- **`inbound`** — inbound shipment table; rendered by `renderInboundPage()`

### Auth

Login handled by `doLogin()`. Two accounts:
- Real: `teasley@merkuryinnovations.com` — writes to Supabase and localStorage
- Test: `test` / `test` — `isTestMode = true`; all writes (Supabase, localStorage) are suppressed

`isTestMode` gates every persistence call; always check it before adding new save logic.

### Key Computed Logic

- **`calcRows(p)`** (~line 1580): derives WOS (weeks-of-stock) and projected inventory for each fiscal week. Core forecasting engine.
- **`updateCalcs(sku)`** (~line 2002): re-runs calcRows and syncs DOM inputs/cells for one SKU.
- **`renderSummary()`** (~line 1663): updates the header summary bar aggregate stats.
- **`getCurrentWeekIdx()`** (~line 1568): maps today's date to the correct `weeks[]` index.

### Supabase Schema

| Table | Key columns |
|---|---|
| `sku_meta` | `sku`, `on_hand_merkury`, `on_hand_thd`, `avg_weekly_sales_vd` |
| `sku_weekly` | `sku`, `week_idx`, `firm_receipt`, `suof_forecast`, `pos_units`, `reorder_qty`, `reorder_case_pack`, `reorder_config` |
| `app_cache` | `key` (string), `data` (JSON) — keys: `inbound_rows`, `additional_models`, `thd_sales_orders`, `thd_sales_order_details` |

### Forecast Engine — M3 Two-Signal Method

`computeFcRates(sku, startIdx, endIdx)` (~line 6508) is the core forecast function. It uses **Method 3 (M3)**, chosen after backtesting 12 forecast scenarios across 7 SKUs. Do not change the methodology without re-running the backtests.

**Signal 1 — Store-Matched YoY Ratio:**
- Scans the YTD window (FW1 → current−1) for week pairs where TY/LY store count ratio is within ±30%.
- Computes avg TY SPS ÷ avg LY SPS from matched weeks only. Requires ≥4 matches; falls back to full YTD ratio if fewer.
- Purpose: isolates true demand velocity from store expansion noise (e.g., stores doubled in Q1/Q2 2026 vs 2025 for most SKUs).

**Signal 2 — Recent SPS Trend:**
- Compares TY SPS in last 4 weeks vs prior 4 weeks (same year only).
- Suppression rule: if YoY ratio < 0.85 AND trend is positive → set trend = 1.0. Prevents false optimism on confirmed declining SKUs.

**Final formula:** `LY same-week SPS × YoY ratio × trend multiplier × current store count`

**Backtested avg error:** M3 = 17.8% vs M1 (full YTD) = 32.3%, M2 (store-matched only) = 26.6%.

**Key variables returned:** `m3YoYRatio`, `m3YoYSource` (`'store-matched'` | `'ytd-fallback'` | `'snapshot-fallback'`), `storeMatchedCount`, `trendMultiplier`.

The chip detail popup (double-click a forecast chip) calls `showFcChipDetail` (~line 6746) and renders the full two-signal breakdown.

`POS_HISTORY` (~line 1791) is the per-SKU per-FW historical store count + units object used by `computeFcRates` for the LY anchor and store-matching logic.

### Reorder Schedule Engine

`computeReorderSchedule(sku)` (Sales & Reorder Forecast page) replaces flat lump-sum reorder sizing with chained, order-up-to-level cycles. Tunable constants (defined near `computeReorderSuggestion`): `SAFETY_WOS = 4` (floor — never go below this), `ORDER_TARGET_WOS = 10` (ceiling — a delivery should never push WOS above this), `LEAD_DAYS = 80`, `REORDER_CHAIN_WEEKS = 26` (cap chaining at 2 quarters out). These came from explicit user requirements, not arbitrary tuning — don't change without re-confirming the standard still holds.

**Demand rate per week** (`resolveDemandRate`): manual override → SUOF → AI agent's per-week rate → M3 forecast → raw state. This deliberately mirrors `calcRows()`'s own `liveSR` fallback chain (manual → SUOF → 0) for near-term weeks, then extends further out with agent/M3 since SUOF typically only covers ~13 weeks and the regular grid has no fallback beyond that — the two are expected to diverge for far-future weeks precisely because the reorder engine needs to see further than SUOF reaches.

**Critical gotcha — one-week ledger lag:** `calcRows()` computes `endInv[i] = endInv[i-1] + reorders[i-1] + firmReceipts[i-1] - liveSR[i-1]` — a reorder entered for week `i-1` doesn't show its effect until week `i`, and gets netted against week `i-1`'s *own* rate before it ever appears in Ending INV. `computeReorderSchedule` mirrors this exactly (`enterOff` = week to type the qty into the Reorder cell, `effectOff = enterOff + 1` = week Ending INV reflects it). Two real bugs came from getting this wrong: (1) sizing a cycle's target-window sum starting at `effectOff` instead of `enterOff` under-sizes every order by one week's worth of rate, because the ledger silently taxes the new qty against `enterOff`'s rate; (2) rebasing the trajectory after each cycle must re-run the same netting formula (`track[enterOff] + qty - rate[enterOff]`), not flatly add `qty` on top of the old baseline. Verify any future change here against the actual `calcRows()` output (or the engine's own internal `track`/`rates`, exposed via a temporary `window._debugSchedule` capture at the end of the function) — this logic has burned the "looks right in isolation, breaks on the real ledger" trap twice already.

Each cycle's status: `critical` (floor breach already closer than lead time allows — some time below the floor is unavoidable, `gapWeeks` shows how much), `reminder` (ordering today lands exactly on time), `upcoming` (informational, chained further out).

The Forecast page's collapsed list row renders every chained cycle directly (not just the first) as compact grid lines: `# | Place date | Lands date (FW) | qty "each" | status`, uniform spacing between columns. Clicking the row still expands the full AI Forecast Analysis card underneath, unchanged.

## Development Notes

- No build step. Edit the HTML files directly and refresh the browser.
- **Always update both files** when making changes — `index.html` and `Weekly_Inventory_Recap.html` must stay identical.
- The Supabase anon key is base64-encoded in three segments (lines 1343–1347) and reassembled at runtime — not plaintext, but not truly secret since it's client-side.
- `isTestMode` must gate any new write operations to prevent test logins from mutating real data.
- When adding new `weeks[]` coverage, extend all per-week arrays (firmReceipts, salesRates, reorders, etc.) accordingly — `calcRows` iterates over `weeks.length`.

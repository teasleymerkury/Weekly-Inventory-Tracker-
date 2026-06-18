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

## Development Notes

- No build step. Edit the HTML files directly and refresh the browser.
- **Always update both files** when making changes — `index.html` and `Weekly_Inventory_Recap.html` must stay identical.
- The Supabase anon key is base64-encoded in three segments (lines 1343–1347) and reassembled at runtime — not plaintext, but not truly secret since it's client-side.
- `isTestMode` must gate any new write operations to prevent test logins from mutating real data.
- When adding new `weeks[]` coverage, extend all per-week arrays (firmReceipts, salesRates, reorders, etc.) accordingly — `calcRows` iterates over `weeks.length`.

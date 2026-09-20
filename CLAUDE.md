# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file static web app (`index.html`, no build step, no server) that
reconciles branch stock counts against Grab Mart's product CSV exports and
produces ready-to-upload `GRAB_xxx.CSV` files. There is no package.json,
no bundler, and no test runner — open `index.html` directly in a browser
(or serve the directory statically) to run it.

Everything — markup, CSS, and all JS — lives in `index.html`. There are no
other source files. `github-ssh.config.example` is an unrelated SSH setup
helper, not part of the app.

## Running / testing locally

There's no dev server script. Serve the directory with any static file
server and open it, e.g.:

```
python -m http.server 8000
```

Then browse to `http://localhost:8000/index.html`. Supabase (see below)
must be configured via the ⚙ CONFIG modal before most features work —
without it, `window._sbReady` stays `false` and RUN/upload are blocked
with a `[DB] SUPABASE NOT READY` log line.

There is no automated test suite. When verifying changes, drive the app
in a real (or headless, via Playwright) browser and check the `#log-panel`
output and produced CSV content — do not assume correctness from reading
the code alone, since the row-processing logic depends on real CSV column
layouts that only show up at runtime.

## Data flow (the whole point of the app)

1. **UPLOAD REPORT R01.102** card: admin uploads one `xxx.csv` per branch
   (e.g. `sss.csv`, `src.csv`) — an internal stock report. Kept in memory
   only (`sourceFilesCache`), never uploaded anywhere.
2. **GRAB CSV MANAGER** card: the *current* Grab Mart product export per
   branch (`GRAB_XXX.CSV`) lives in Supabase Storage (table `grab_files`),
   uploaded here by whoever has the branch's real Grab data. Locked behind
   a client-side PIN (`PASS` constant) — a UI deterrent for "SYSTEM ADMIN
   ONLY", not real access control.
3. **PROCESS ENGINE** card, `runAll()`: for each uploaded `xxx.csv`, fetches
   the matching `GRAB_XXX.CSV` from Supabase, matches rows by SKU, and
   writes the branch's stock quantity (minus a per-branch subtraction
   buffer, so the storefront doesn't show items as available right down to
   zero) into the GRAB row's CurrentStock/MaxStock columns. Produces a
   downloadable `GRAB_XXX_<date>.csv` per branch — this is the file that
   actually gets uploaded back into Grab Mart's own system (outside this
   app).
4. **PENDING SKU UPDATE** card: SKUs pending a price/name/image change with
   Grab (takes 3-5 business days) are tracked here and forced to
   `out-of-stock indefinitely` by `runAll()` regardless of source stock,
   so they don't go live with stale info.

## GRAB_XXX.CSV column layout — read this before touching index/column logic

The whole pipeline reads/writes fixed **numeric column indices** into
`GRAB_XXX.CSV` rows — there is no header-name-based column lookup in the
data path itself. This already broke once silently (`GRAB_SKU_IDX` pointed
at the wrong column, so every row failed to match and stock columns came
out empty with no warning). The constants live near the top of the
`<script>` block:

```js
GRAB_NAME_IDX         = 1  // column B — item name
GRAB_PRICE_IDX        = 2  // column C — price
GRAB_SKU_IDX          = 3  // column D — SKU (ItemCode)
GRAB_CURRENTSTOCK_IDX = 4  // column E — CurrentStock
GRAB_MAXSTOCK_IDX     = 5  // column F — MaxStock
```

`GRAB_EXPECTED_HEADERS` maps those same indices to the literal header text
Grab's export uses (`ITEMCODE`, `CURRENTSTOCK`, `MAXSTOCK`). Two guard
points check the real header row against this map via
`checkGrabHeaderLayout()` and refuse to proceed silently if it's wrong:

- **On upload** (`uploadCSVFiles()`, GRAB CSV MANAGER card): a file whose
  header doesn't match is **not saved to Supabase at all** — logged as
  `BLOCKED`, alerted, upload fails for that file.
- **On RUN** (`runAll()`, PROCESS ENGINE card): after processing, if every
  data row in a branch's output failed to match a SKU (the same shape as
  the original silent bug), it's logged as a warning and alerted — RUN
  still completes for branches that did match.

If Grab ever changes their export layout again, update the index constants
*and* `GRAB_EXPECTED_HEADERS` together — the guard is only as correct as
those expected header strings.

`GRAB_XXX.CSV` files also have a variable-height header (2-4 rows,
sometimes with embedded newlines inside quoted cells) before real data
starts. `grabDataStart()` detects this per-file by content (first row
where the SKU column is non-empty and the price column parses as a
number) rather than assuming a fixed row count.

The source `xxx.csv` (R01.102) files use a *different*, unrelated column
layout — SKU is `row[4]` (column E) and quantity is `row[6]` (column G),
hardcoded inline in `runAll()`, not shared with the GRAB_* constants.

## Supabase (two separate projects)

- **Main project** — stores `grab_files` (the GRAB_XXX.CSV contents) and
  `app_settings` (pending SKU list, and the second project's config). Its
  URL/key are **never stored in the repo**; they live only in the page's
  URL hash (`#cfg=base64...`), decoded by `_sbGetConfig()`. This is
  intentional — the hash isn't sent to any server, so the config can be
  bookmarked/shared without putting credentials in git.
- **"DATABASE" project (2nd Supabase project)** — a `products` table used
  only as a reference price source for the PENDING SKU card. Its config is
  entered once via the CONFIG modal and then persisted *inside the main
  project's* `app_settings` table (key `db2_config`), so it doesn't need
  to be re-entered per machine/bookmark like the main project's own config
  does.

All Supabase access goes through `window._sb*` functions assigned inside
`_sbInitSupabase()` (e.g. `_sbSaveCSV`, `_sbGetCSVText`,
`_sbLoadPendingSkus`) — callers check the relevant function is truthy
before use, since it's `null` until Supabase is configured and reachable.

## Code organization within index.html

The `<script>` block is organized into clearly marked sections (search for
`// ════` banners): CONFIG via URL HASH, SUPABASE #2 (DATABASE), SUPABASE
INIT CALLBACK, CONFIG MODAL, CSV UTILITIES (`parseCSV`/`csvToText` — a
hand-rolled RFC4180-ish parser, no library), LOCAL SOURCE FILES, LOCK /
UNLOCK, AUTO-PAIR, VALIDATE / RUN (the core pipeline), LOG, and PENDING SKU
UPDATE (which also contains the PROMAX price-import sub-feature, uploaded
separately from the GRAB CSV MANAGER card).

`normalizeKey()` is the shared SKU-comparison helper (trims, strips
BOM/zero-width chars, uppercases) — reuse it for any new SKU or
header-name comparison rather than comparing raw strings.

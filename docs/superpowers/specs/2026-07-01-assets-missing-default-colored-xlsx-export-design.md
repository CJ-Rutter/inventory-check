# Assets: Missing-by-default workflow + concise colored `.xlsx` export

**Date:** 2026-07-01
**Repo:** inventory-check (single-file `index.html`)
**Target version:** v1.7.0
**Scope:** **Assets tab only.** Parts and Bulk tabs are unchanged (keep count-based Match/Short/Over and CSV export).

## Problem

Exports feeding Inventory Manager are convoluted: the CSV drags every original
column (serial, OEC, rate, on-rent qty, allocated, last-off-rent, …) that IM
never reads, with the check columns tacked onto the far right. Separately, the
asset check workflow treats untouched items as a neutral "Unchecked" state, when
operationally an unaccounted asset should be treated as **Missing until proven
otherwise**.

## Goals

1. **Missing-by-default asset workflow** with a two-button interaction.
2. **Concise, colored, Excel/Google-Sheets-openable export** for Assets that is
   a clean data contract for Inventory Manager to consume.

Non-goals: changing Parts/Bulk; on-device recount logic; IM's own changes
(handled separately by the user against the contract defined here).

---

## Part A — Assets status model

Internal status codes today: `"ok"` = Accounted, `"bad"` = Missing, `"warn"` =
Damaged, `""` = Unchecked.

Changes (Assets only):

- **Default status = Missing (`"bad"`).** Any asset row with no stored/imported
  status renders as Missing instead of `""`. The `""` (Unchecked) state is
  retired for assets — Missing *is* the not-yet-accounted state.
  - `effectiveStatus(r, "assets", cols)` returns `r._status || "bad"`.
  - On load/import/reset, an asset row with no explicit status resolves to
    `"bad"`, not `""`.
- **Two buttons only: `Accounted For` and `Damaged`.** Remove the Missing
  button from the asset card (currently `index.html:1131`, `data-act="bad"`).
- **Toggle-off returns to Missing.** Tapping the currently-active button clears
  it back to `"bad"` (not `""`). Update the asset toggle (`index.html:1140`) so
  the "off" value for assets is `"bad"`.
- **Filter chips (assets):** `All / Missing / Accounted / Damaged`. Remove the
  `Unchecked` chip for assets (`index.html:993`). Parts/Bulk chips unchanged.
- **Export scope dropdown:** hide the `Unchecked only` option
  (`index.html:438`) when the target tab is assets (no unchecked state exists).
- **Bulk-affecting helpers** (assets only):
  - "Mark all visible Accounted" (`index.html:1268`): set visible asset rows to
    `"ok"` regardless of current status (missing→accounted included).
  - "Reset" (`index.html:1274`, `1301`): asset rows reset to `"bad"` (Missing),
    not `""`.
  - Jump-to-next-unchecked (`~index.html:1282`): for assets, jump to the next
    **Missing** row.
  - Summary (`buildSummary`, `index.html:1355`): for assets, drop the
    "Unchecked" figure (it is always 0 under this model); keep
    Total / Accounted / Missing / Damaged.

No change to `localStorage` mechanics. Effect: previously-saved assets checks
reload with untouched rows now shown as Missing — intended.

---

## Part B — Concise colored `.xlsx` export (Assets)

### Format decision

A **real `.xlsx`** file, **hand-rolled in vanilla JS with no external
dependency**: a store-only (uncompressed, method 0) ZIP containing minimal
OOXML parts. Opens with correct row fills in **Excel, Google Sheets, and
Numbers** with no "format doesn't match" prompt. (An HTML-table-as-`.xls` was
rejected because Google Sheets does not reliably preserve its row colors.)

Assets export becomes `.xlsx`-only (no CSV for assets). Parts/Bulk stay CSV.

### Columns (fixed order)

| Asset ID | Cat Class | S/N | Make | Model | Market/Branch | Status | Added | Notes |
|---|---|---|---|---|---|---|---|---|

- Sourced via existing `detectAssetsCols`: `id`, `cat`, `ser`, `make`, `model`,
  and `Market/Branch` (add a `market` detection: `["Market","Branch"]`).
- **All other original columns are dropped** (serial-note, OEC, rate, on-rent,
  allocated, last-off-rent, Description, Inventory Status, …).
- **Status** cell text (machine-readable truth): `Accounted For` / `Damaged` /
  `Missing`.
- **Added**: `Yes` for manually-added rows (`r._added`), else blank. Kept as a
  real column (not just color) so IM's dedup/"needs review" logic stays robust.
  Placed before Notes so **Notes remains last**.
- **Notes**: `r._note`.

### Row colors (whole data row shaded; header row excluded)

Precedence (first match wins):

1. **Added** (`r._added`) → **orange** `#F6B26B`
2. else **Damaged** (`"warn"`) → **red** `#E06666`
3. else **Accounted** (`"ok"`) → **green** `#A9D08E`
4. else **Missing** (`"bad"`) → **yellow** `#FFD966`

Header row: bold, light-gray `#D9D9D9` fill. Black text throughout (all fills
are mid-tone and readable).

### `.xlsx` internals (implementation contract)

Store-only ZIP with these parts (text values written as **inline strings**,
`t="inlineStr"`, so no `sharedStrings.xml` is needed and readers get values
straight from the sheet):

- `[Content_Types].xml`
- `_rels/.rels`
- `xl/workbook.xml`
- `xl/_rels/workbook.xml.rels`
- `xl/styles.xml` — one `<fill>` per color (gray header + 4 status colors) and a
  `<cellXfs>` entry per fill; data cells carry `s="…"` for their row color.
- `xl/worksheets/sheet1.xml` — rows/cells; each cell `<c r="A2" t="inlineStr"
  s="N"><is><t>value</t></is></c>`.

Helper needs: a **CRC-32** implementation (~15 lines) for ZIP local/central
headers, and correct local-file-header + central-directory + end-of-central-
directory records. No compression. XML text must be escaped
(`& < > " '`). This is ~150 lines of self-contained code added to `index.html`.

Filename: `assets[-<scope>]-YYYY-MM-DD-HHMM.xlsx` (reuse `exportStamp()` /
`exportName()`; extension switches to `.xlsx` for assets). Blob MIME:
`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`.

### Export wiring

- `downloadOne` / `buildExportFiles`: when `kind === "assets"`, build the
  `.xlsx` via a new `buildAssetsXlsx(scope)`; otherwise keep `buildCSVFor`.
- Share/attachment path attaches the `.xlsx` File with the OOXML MIME.
- `filterRowsForExport` scopes unchanged, except the assets `unchecked` scope is
  gone (see Part A).

---

## Part C — Inventory Manager contract (implemented separately by the user)

The new Assets file IM must consume:

- **Format:** `.xlsx` (store-only ZIP). IM reads `xl/worksheets/sheet1.xml`,
  values are inline strings — IM needs a store-only unzip + XML parse, but does
  **not** need `styles.xml` (colors are cosmetic). Since Inventory Check is the
  only producer, IM's unzip can assume method 0 (no inflate needed).
- **Columns:** `Asset ID, Cat Class, S/N, Make, Model, Market/Branch, Status,
  Added, Notes`.
- **Status values:** `Accounted For` / `Damaged` / `Missing` — and **`Missing`
  is now the default**, so IM's "blank = not checked" bucket must change to
  treat Missing as not-yet-accounted.
- **Added:** `Yes`/blank replaces reliance on the old `Added Manually` column.
- Dropped columns (Description, etc.) — IM tolerates their absence.

---

## Part D — Versioning & cleanup

- Bump `APP_VERSION` `"1.6.0"` → `"1.7.0"` (`index.html:555`); tag `v1.7.0`.
- Fix the stale hardcoded `v1.1.3` in the header markup (`index.html:364`) so it
  matches `APP_VERSION` (it is overwritten at runtime by `index.html:1506`, so
  this is source-cosmetic only, but confusing).

## Testing (manual — single-file app, no framework)

1. Load an assets CSV → all rows show **Missing** by default; only two buttons.
2. Tap Accounted → green badge; tap again → back to Missing. Tap Damaged →
   Damaged; tap again → Missing.
3. Add an unlisted item → flagged Added.
4. Export → `.xlsx` downloads; open in **Google Sheets** and **Excel**: 9
   columns in order, row colors correct (added=orange overrides damaged/
   accounted; damaged=red; accounted=green; missing=yellow), header gray.
5. Scope exports (Missing / Damaged / Exceptions) contain the right rows.
6. Parts/Bulk unchanged: still CSV, still Match/Short/Over.

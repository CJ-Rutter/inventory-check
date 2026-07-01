# Assets Missing-by-default + Colored `.xlsx` Export — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Assets tab treat every asset as Missing until accounted for (two-button workflow), and export a concise, color-coded real `.xlsx` as a clean data contract for Inventory Manager.

**Architecture:** Single-file app (`index.html`), no build, no test framework. The internal status code stays `""` for untouched rows; assets *display and export* map `""`→Missing via `effectiveStatus`. The `.xlsx` is written by hand in vanilla JS (CRC-32 + store-only ZIP + minimal OOXML with inline strings and fill styles). Assets export is `.xlsx`; Parts/Bulk stay CSV.

**Tech Stack:** Vanilla HTML/CSS/JS. Verification is manual-in-browser plus `unzip`/Google Sheets/Excel for the `.xlsx`.

**Spec:** `docs/superpowers/specs/2026-07-01-assets-missing-default-colored-xlsx-export-design.md`

**Status codes:** `"ok"`=Accounted, `"warn"`=Damaged, `"bad"`=Missing (explicit), `""`=untouched. After this change, assets treat `""` as Missing everywhere they display/count/export.

---

### Task 1: Version bump + header literal fix

**Files:** Modify `index.html:555`, `index.html:364`

- [ ] **Step 1: Bump `APP_VERSION`**

Change `index.html:555` from:
```js
const APP_VERSION = "1.6.0";
```
to:
```js
const APP_VERSION = "1.7.0";
```

- [ ] **Step 2: Fix the stale header version literal**

Change `index.html:364` from:
```html
      <div class="title">Inventory Check <span class="title-ver" id="headerVer">v1.1.3</span></div>
```
to:
```html
      <div class="title">Inventory Check <span class="title-ver" id="headerVer">v1.7.0</span></div>
```

- [ ] **Step 3: Verify**

Open `index.html` in a browser. Footer and header both read `v1.7.0`. No console errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "chore: bump to v1.7.0; align header version literal"
```

---

### Task 2: Assets default to Missing (display + counts)

Keep `_status` internally `""` for untouched rows; make assets *resolve* `""`→`"bad"` (Missing) via `effectiveStatus`, and route the asset card badge + in-place update through it.

**Files:** Modify `index.html:902-905`, `index.html:1122`, `index.html:1214`

- [ ] **Step 1: Default assets to Missing in `effectiveStatus`**

Change `index.html:902-905` from:
```js
function effectiveStatus(r, kind, cols) {
  if (isBulkLike(kind)) return bulkDerivedStatus(r, cols);
  return r._status || "";
}
```
to:
```js
function effectiveStatus(r, kind, cols) {
  if (isBulkLike(kind)) return bulkDerivedStatus(r, cols);
  return r._status || "bad";   // assets: unaccounted defaults to Missing
}
```

- [ ] **Step 2: Route the asset card badge through `effectiveStatus`**

Change `index.html:1122` from:
```js
    <div><span class="asset-id">${escapeHtml(id) || "—"}</span>${statusBadge(r._status, false)}${rentBadge}${addedTag}</div>
```
to:
```js
    <div><span class="asset-id">${escapeHtml(id) || "—"}</span>${statusBadge(effectiveStatus(r, "assets", cols), false)}${rentBadge}${addedTag}</div>
```

- [ ] **Step 3: Route in-place card update through `effectiveStatus`**

Change `index.html:1214` from:
```js
  const status = isBulk ? effectiveStatus(r, kind, cols) : r._status;
```
to:
```js
  const status = effectiveStatus(r, kind, cols);
```

- [ ] **Step 4: Verify**

Reload `index.html`, load an assets CSV (`Asset ID` + `Equipment Class` columns). Expected:
- Every asset card shows a **Missing** badge by default.
- Tap **Accounted** → green **Accounted** badge. Tap **Accounted** again → back to **Missing** badge (not blank).
- The stat tiles show Missing = total-minus-accounted-minus-damaged (untouched counts as Missing).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(assets): treat untouched assets as Missing by default"
```

---

### Task 3: Remove the Missing button (two-button asset card)

**Files:** Modify `index.html:1129-1133`

- [ ] **Step 1: Delete the Missing button**

Change `index.html:1129-1133` from:
```js
    <div class="actions">
      <button class="b-ok ${r._status && r._status!=="ok" ? "dim":""}"  data-act="ok">Accounted</button>
      <button class="b-bad ${r._status && r._status!=="bad" ? "dim":""}" data-act="bad">Missing</button>
      <button class="b-warn ${r._status && r._status!=="warn" ? "dim":""}" data-act="warn">Damaged</button>
    </div>
```
to:
```js
    <div class="actions">
      <button class="b-ok ${r._status && r._status!=="ok" ? "dim":""}"  data-act="ok">Accounted For</button>
      <button class="b-warn ${r._status && r._status!=="warn" ? "dim":""}" data-act="warn">Damaged</button>
    </div>
```

- [ ] **Step 2: Verify**

Reload, load assets CSV. Expected:
- Each asset card shows **two** buttons: `Accounted For` and `Damaged` (no Missing button).
- `Damaged` still sets/clears the Damaged status; clearing returns to Missing.
- The Parts/Bulk cards are unchanged (still three buttons Match/Short/Over).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(assets): two-button card (Accounted For / Damaged)"
```

---

### Task 4: Filter chips, Jump-to-Missing, and summary for assets

**Files:** Modify `index.html:988-1003` (`renderFilters`), `index.html:1278-1295` (`btnJump`), `index.html:1370` (`buildSummary`)

- [ ] **Step 1: Drop the Unchecked chip for assets + migrate saved filter**

Change `index.html:988-1003` from:
```js
function renderFilters() {
  const kind = state.active;
  const isBulk = isBulkLike(kind);
  const opts = [
    ["all", "All"],
    ["unchecked", "Unchecked"],
    ["ok", isBulk ? "Match" : "Accounted"],
    ["bad", isBulk ? "Short" : "Missing"],
    ["warn", isBulk ? "Over" : "Damaged"],
    ["onrent", "On Rent"],
  ];
  const active = state.filter[kind] || "all";
  $("#filters").innerHTML = opts.map(([f,label]) =>
    `<div class="chip ${f===active?"active":""}" data-f="${f}">${label}</div>`
  ).join("");
}
```
to:
```js
function renderFilters() {
  const kind = state.active;
  const isBulk = isBulkLike(kind);
  // Assets have no "Unchecked" state anymore (untouched = Missing); migrate any
  // stale saved filter so nothing renders an empty list.
  if (kind === "assets" && (state.filter[kind] || "all") === "unchecked") state.filter[kind] = "all";
  const opts = [
    ["all", "All"],
    ["unchecked", "Unchecked"],
    ["ok", isBulk ? "Match" : "Accounted"],
    ["bad", isBulk ? "Short" : "Missing"],
    ["warn", isBulk ? "Over" : "Damaged"],
    ["onrent", "On Rent"],
  ].filter(([f]) => !(kind === "assets" && f === "unchecked"));
  const active = state.filter[kind] || "all";
  $("#filters").innerHTML = opts.map(([f,label]) =>
    `<div class="chip ${f===active?"active":""}" data-f="${f}">${label}</div>`
  ).join("");
}
```

- [ ] **Step 2: Jump-to-next targets Missing on assets**

Change `index.html:1278-1295` from:
```js
$("#btnJump").addEventListener("click", () => {
  const tab = state.tabs[state.active];
  const cols = tab.cols;
  const vis = visibleRows();
  const target = vis.find(r => !effectiveStatus(r, state.active, cols))
              || tab.rows.find(r => !effectiveStatus(r, state.active, cols));
  if (!target) { alert("Nothing left to check."); return; }
  const id = String(target[cols.id] ?? "");
  state.filter[state.active] = "unchecked";
  state.search = ""; $("#search").value = "";
  save(); render();
  const card = document.querySelector(`.card[data-id="${CSS.escape(id)}"]`);
  if (card) {
    card.scrollIntoView({behavior:"smooth", block:"center"});
    card.style.outline = "2px solid var(--accent)";
    setTimeout(() => card.style.outline = "", 1200);
  }
});
```
to:
```js
$("#btnJump").addEventListener("click", () => {
  const tab = state.tabs[state.active];
  const cols = tab.cols;
  const vis = visibleRows();
  // Assets: "next to review" = next Missing row (no unchecked state).
  // Parts/Bulk: next row with no status yet.
  const isAssets = state.active === "assets";
  const isTarget = (r) => isAssets
    ? effectiveStatus(r, "assets", cols) === "bad"
    : !effectiveStatus(r, state.active, cols);
  const target = vis.find(isTarget) || tab.rows.find(isTarget);
  if (!target) { alert("Nothing left to check."); return; }
  const id = String(target[cols.id] ?? "");
  state.filter[state.active] = isAssets ? "bad" : "unchecked";
  state.search = ""; $("#search").value = "";
  save(); render();
  const card = document.querySelector(`.card[data-id="${CSS.escape(id)}"]`);
  if (card) {
    card.scrollIntoView({behavior:"smooth", block:"center"});
    card.style.outline = "2px solid var(--accent)";
    setTimeout(() => card.style.outline = "", 1200);
  }
});
```

- [ ] **Step 3: Drop the "Unchecked" figure from the assets summary**

Change `index.html:1370` from:
```js
  lines.push(`Total: ${tab.rows.length} | ${isBulk?"Match":"Accounted"}: ${ok} | ${isBulk?"Short":"Missing"}: ${bad} | ${isBulk?"Over":"Damaged"}: ${warn} | Unchecked: ${tab.rows.length - ok - bad - warn}`);
```
to:
```js
  const uncPart = isBulk ? ` | Unchecked: ${tab.rows.length - ok - bad - warn}` : "";
  lines.push(`Total: ${tab.rows.length} | ${isBulk?"Match":"Accounted"}: ${ok} | ${isBulk?"Short":"Missing"}: ${bad} | ${isBulk?"Over":"Damaged"}: ${warn}${uncPart}`);
```

- [ ] **Step 4: Verify**

Reload, load assets CSV. Expected:
- Filter chips read: `All · Accounted · Missing · Damaged · On Rent` (no Unchecked).
- Tap **Jump** → scrolls to a Missing asset and the Missing chip becomes active.
- Parts/Bulk chips still include Unchecked, and their Jump still finds an uncounted row.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(assets): drop Unchecked chip, Jump targets Missing, tidy summary"
```

---

### Task 5: Detect the Market/Branch column for assets

**Files:** Modify `index.html:654-665` (`detectAssetsCols`)

- [ ] **Step 1: Add `market` detection**

Change `index.html:654-665` from:
```js
function detectAssetsCols(headers) {
  return {
    id:      findCol(headers, ["Asset ID", "Asset #", "AssetNumber", "Asset", "Unit #"]),
    cat:     findCol(headers, ["Equipment Class", "Cat Class", "Category", "Class"]),
    make:    findCol(headers, ["Make"]),
    model:   findCol(headers, ["Model"]),
    ser:     findCol(headers, ["Serial # or VIN", "Serial #", "Serial", "VIN"]),
    desc:    findCol(headers, ["Description"]),
    src:     findCol(headers, ["Inventory Status", "Status"]),
    lastOff: findCol(headers, ["Last Off Rent Date", "Off Rent Date"]),
  };
}
```
to:
```js
function detectAssetsCols(headers) {
  return {
    id:      findCol(headers, ["Asset ID", "Asset #", "AssetNumber", "Asset", "Unit #"]),
    cat:     findCol(headers, ["Equipment Class", "Cat Class", "Category", "Class"]),
    make:    findCol(headers, ["Make"]),
    model:   findCol(headers, ["Model"]),
    ser:     findCol(headers, ["Serial # or VIN", "Serial #", "Serial", "VIN"]),
    desc:    findCol(headers, ["Description"]),
    src:     findCol(headers, ["Inventory Status", "Status"]),
    lastOff: findCol(headers, ["Last Off Rent Date", "Off Rent Date"]),
    market:  findCol(headers, ["Market", "Branch"]),
  };
}
```

- [ ] **Step 2: Verify**

Reload, load an assets CSV that has a `Market` (or `Branch`) column. In the browser console run:
```js
state.tabs.assets.cols.market
```
Expected: the resolved column name (e.g. `"Market"`), or `undefined` if the CSV has no such column (that's fine — export will leave the cell blank).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(assets): detect Market/Branch column for export"
```

---

### Task 6: CRC-32 + store-only ZIP writer (no dependency)

Add a self-contained byte-level ZIP writer. Place this new block **immediately after** the `toCSV` function (ends at `index.html:638`).

**Files:** Modify `index.html` (insert after line 638)

- [ ] **Step 1: Insert the CRC-32 + ZIP helpers**

Insert after `index.html:638` (after the closing `}` of `toCSV`):
```js
// ---------- Minimal .xlsx writer: CRC-32 + store-only ZIP (no deps) ----------
const CRC_TABLE = (() => {
  const t = new Uint32Array(256);
  for (let n = 0; n < 256; n++) {
    let c = n;
    for (let k = 0; k < 8; k++) c = (c & 1) ? (0xEDB88320 ^ (c >>> 1)) : (c >>> 1);
    t[n] = c >>> 0;
  }
  return t;
})();
function crc32(bytes) {
  let c = 0xFFFFFFFF;
  for (let i = 0; i < bytes.length; i++) c = CRC_TABLE[(c ^ bytes[i]) & 0xFF] ^ (c >>> 8);
  return (c ^ 0xFFFFFFFF) >>> 0;
}
function concatBytes(parts) {
  let len = 0;
  for (const p of parts) len += p.length;
  const out = new Uint8Array(len);
  let o = 0;
  for (const p of parts) { out.set(p, o); o += p.length; }
  return out;
}
// files: [{ name, data: Uint8Array }] → Uint8Array of a store-only (method 0) zip.
function zipStore(files) {
  const enc = new TextEncoder();
  const u16 = (n) => new Uint8Array([n & 255, (n >>> 8) & 255]);
  const u32 = (n) => new Uint8Array([n & 255, (n >>> 8) & 255, (n >>> 16) & 255, (n >>> 24) & 255]);
  const locals = [];
  const central = [];
  let offset = 0;
  for (const f of files) {
    const nameBytes = enc.encode(f.name);
    const crc = crc32(f.data);
    const size = f.data.length;
    const local = concatBytes([
      u32(0x04034b50), u16(20), u16(0), u16(0), u16(0), u16(0),
      u32(crc), u32(size), u32(size), u16(nameBytes.length), u16(0),
      nameBytes, f.data,
    ]);
    locals.push(local);
    central.push(concatBytes([
      u32(0x02014b50), u16(20), u16(20), u16(0), u16(0), u16(0), u16(0),
      u32(crc), u32(size), u32(size), u16(nameBytes.length),
      u16(0), u16(0), u16(0), u16(0), u32(0), u32(offset), nameBytes,
    ]));
    offset += local.length;
  }
  const centralBytes = concatBytes(central);
  const end = concatBytes([
    u32(0x06054b50), u16(0), u16(0), u16(files.length), u16(files.length),
    u32(centralBytes.length), u32(offset), u16(0),
  ]);
  return concatBytes([...locals, centralBytes, end]);
}
```

- [ ] **Step 2: Verify CRC-32 against the standard test vector**

Reload `index.html`, open the browser console, run:
```js
crc32(new TextEncoder().encode("123456789")).toString(16)
```
Expected: `"cbf43926"` (the canonical CRC-32 of `"123456789"`).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(export): add CRC-32 + store-only ZIP writer"
```

---

### Task 7: Build the colored Assets `.xlsx`

Add the OOXML constants, the row-style/text helpers, and `buildAssetsXlsx`. Place this block **immediately after** the `zipStore` function inserted in Task 6.

**Files:** Modify `index.html` (insert after the `zipStore` block)

- [ ] **Step 1: Insert the xlsx builder**

Insert after the `zipStore` function:
```js
// ---------- .xlsx OOXML parts + Assets sheet builder ----------
const XLSX_CONTENT_TYPES = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types"><Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/><Default Extension="xml" ContentType="application/xml"/><Override PartName="/xl/workbook.xml" ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet.main+xml"/><Override PartName="/xl/worksheets/sheet1.xml" ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.worksheet+xml"/><Override PartName="/xl/styles.xml" ContentType="application/vnd.openxmlformats-officedocument.spreadsheetml.styles+xml"/></Types>`;
const XLSX_RELS = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships"><Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/officeDocument" Target="xl/workbook.xml"/></Relationships>`;
const XLSX_WORKBOOK = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<workbook xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main" xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships"><sheets><sheet name="Assets" sheetId="1" r:id="rId1"/></sheets></workbook>`;
const XLSX_WORKBOOK_RELS = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships"><Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/worksheet" Target="worksheets/sheet1.xml"/><Relationship Id="rId2" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/styles" Target="styles.xml"/></Relationships>`;
// Style indices: 1=header(gray,bold) 2=green 3=yellow 4=red 5=orange.
// fills 0/1 are reserved by the spec (none, gray125); custom fills start at 2.
const XLSX_STYLES = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<styleSheet xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main"><fonts count="2"><font><sz val="11"/><name val="Calibri"/></font><font><b/><sz val="11"/><name val="Calibri"/></font></fonts><fills count="7"><fill><patternFill patternType="none"/></fill><fill><patternFill patternType="gray125"/></fill><fill><patternFill patternType="solid"><fgColor rgb="FFD9D9D9"/></patternFill></fill><fill><patternFill patternType="solid"><fgColor rgb="FFA9D08E"/></patternFill></fill><fill><patternFill patternType="solid"><fgColor rgb="FFFFD966"/></patternFill></fill><fill><patternFill patternType="solid"><fgColor rgb="FFE06666"/></patternFill></fill><fill><patternFill patternType="solid"><fgColor rgb="FFF6B26B"/></patternFill></fill></fills><borders count="1"><border/></borders><cellStyleXfs count="1"><xf numFmtId="0" fontId="0" fillId="0" borderId="0"/></cellStyleXfs><cellXfs count="6"><xf numFmtId="0" fontId="0" fillId="0" borderId="0"/><xf numFmtId="0" fontId="1" fillId="2" borderId="0" applyFont="1" applyFill="1"/><xf numFmtId="0" fontId="0" fillId="3" borderId="0" applyFill="1"/><xf numFmtId="0" fontId="0" fillId="4" borderId="0" applyFill="1"/><xf numFmtId="0" fontId="0" fillId="5" borderId="0" applyFill="1"/><xf numFmtId="0" fontId="0" fillId="6" borderId="0" applyFill="1"/></cellXfs><cellStyles count="1"><cellStyle name="Normal" xfId="0" builtinId="0"/></cellStyles></styleSheet>`;

function xmlEsc(s) {
  return String(s ?? "").replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;").replace(/"/g,"&quot;").replace(/'/g,"&#39;");
}
function colLetter(i) {           // 0-based column index → A, B, … Z, AA, …
  let s = "", n = i + 1;
  while (n > 0) { const m = (n - 1) % 26; s = String.fromCharCode(65 + m) + s; n = Math.floor((n - 1) / 26); }
  return s;
}
function xlsxCell(colIdx, rowNum, text, styleIdx) {
  const s = styleIdx ? ` s="${styleIdx}"` : "";
  return `<c r="${colLetter(colIdx)}${rowNum}"${s} t="inlineStr"><is><t xml:space="preserve">${xmlEsc(text)}</t></is></c>`;
}
// Row fill precedence: added=orange(5) > damaged=red(4) > accounted=green(2) > missing=yellow(3)
function assetRowStyle(r) {
  if (r._added) return 5;
  const s = r._status || "bad";
  if (s === "warn") return 4;
  if (s === "ok") return 2;
  return 3;
}
function assetStatusText(s) {
  if (s === "ok") return "Accounted For";
  if (s === "warn") return "Damaged";
  return "Missing";
}
function buildAssetsXlsx(scope) {
  const cols = state.tabs.assets.cols;
  const headers = ["Asset ID","Cat Class","S/N","Make","Model","Market/Branch","Status","Added","Notes"];
  const dataRows = filterRowsForExport("assets", scope);
  const xmlRows = [];
  xmlRows.push(`<row r="1">` + headers.map((h,i) => xlsxCell(i, 1, h, 1)).join("") + `</row>`);
  dataRows.forEach((r, idx) => {
    const rowNum = idx + 2;
    const st = r._status || "bad";
    const vals = [
      r[cols.id], r[cols.cat], r[cols.ser], r[cols.make], r[cols.model],
      cols.market ? r[cols.market] : "",
      assetStatusText(st),
      r._added ? "Yes" : "",
      r._note || "",
    ];
    const sIdx = assetRowStyle(r);
    xmlRows.push(`<row r="${rowNum}">` + vals.map((v,i) => xlsxCell(i, rowNum, String(v ?? ""), sIdx)).join("") + `</row>`);
  });
  const sheet = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<worksheet xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main"><sheetData>${xmlRows.join("")}</sheetData></worksheet>`;
  const enc = (s) => new TextEncoder().encode(s);
  const files = [
    { name: "[Content_Types].xml",        data: enc(XLSX_CONTENT_TYPES) },
    { name: "_rels/.rels",                data: enc(XLSX_RELS) },
    { name: "xl/workbook.xml",            data: enc(XLSX_WORKBOOK) },
    { name: "xl/_rels/workbook.xml.rels", data: enc(XLSX_WORKBOOK_RELS) },
    { name: "xl/styles.xml",              data: enc(XLSX_STYLES) },
    { name: "xl/worksheets/sheet1.xml",   data: enc(sheet) },
  ];
  return new Blob([zipStore(files)], { type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" });
}
```

- [ ] **Step 2: Smoke-test the builder in the console**

Reload, load an assets CSV, then in the console run:
```js
const b = buildAssetsXlsx("all");
const a = document.createElement("a");
a.href = URL.createObjectURL(b); a.download = "smoke.xlsx"; a.click();
```
Then in a terminal:
```bash
unzip -l ~/Downloads/smoke.xlsx
```
Expected: lists 6 entries — `[Content_Types].xml`, `_rels/.rels`, `xl/workbook.xml`, `xl/_rels/workbook.xml.rels`, `xl/styles.xml`, `xl/worksheets/sheet1.xml`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(export): build concise colored Assets .xlsx"
```

---

### Task 8: Route Assets export to `.xlsx` (download + share + scope)

**Files:** Modify `index.html:1392-1395` (`exportName`), `index.html:1396-1405` (`downloadOne`), `index.html:1412-1421` (`buildExportFiles`), `index.html:1471-1474` (export modal open), `index.html:442` (button label)

- [ ] **Step 1: `.xlsx` extension for assets in `exportName`**

Change `index.html:1392-1395` from:
```js
function exportName(kind, scope) {
  const scopePart = (scope && scope !== "all") ? `-${scope}` : "";
  return `${kind}${scopePart}-${exportStamp()}.csv`;
}
```
to:
```js
function exportName(kind, scope) {
  const scopePart = (scope && scope !== "all") ? `-${scope}` : "";
  const ext = kind === "assets" ? "xlsx" : "csv";
  return `${kind}${scopePart}-${exportStamp()}.${ext}`;
}
```

- [ ] **Step 2: `downloadOne` builds xlsx for assets**

Change `index.html:1396-1405` from:
```js
function downloadOne(kind, scope) {
  const csv = buildCSVFor(kind, scope);
  const name = exportName(kind, scope);
  const blob = new Blob([csv], {type: "text/csv;charset=utf-8;"});
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url; a.download = name;
  document.body.appendChild(a); a.click(); a.remove();
  setTimeout(() => URL.revokeObjectURL(url), 1500);
}
```
to:
```js
function downloadOne(kind, scope) {
  const name = exportName(kind, scope);
  const blob = kind === "assets"
    ? buildAssetsXlsx(scope)
    : new Blob([buildCSVFor(kind, scope)], {type: "text/csv;charset=utf-8;"});
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url; a.download = name;
  document.body.appendChild(a); a.click(); a.remove();
  setTimeout(() => URL.revokeObjectURL(url), 1500);
}
```

- [ ] **Step 3: `buildExportFiles` attaches xlsx for assets**

Change `index.html:1412-1421` from:
```js
function buildExportFiles(scope) {
  const tabs = resolveExportTabs();
  const files = [];
  for (const k of tabs) {
    if (!state.tabs[k].rows.length) continue;
    const csv = buildCSVFor(k, scope);
    files.push(new File([csv], exportName(k, scope), { type: "text/csv" }));
  }
  return files;
}
```
to:
```js
function buildExportFiles(scope) {
  const tabs = resolveExportTabs();
  const files = [];
  for (const k of tabs) {
    if (!state.tabs[k].rows.length) continue;
    if (k === "assets") {
      files.push(new File([buildAssetsXlsx(scope)], exportName(k, scope),
        { type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" }));
    } else {
      files.push(new File([buildCSVFor(k, scope)], exportName(k, scope), { type: "text/csv" }));
    }
  }
  return files;
}
```

- [ ] **Step 4: Hide the "Unchecked only" scope for assets when opening Export**

Change `index.html:1471-1474` from:
```js
$("#btnExport").addEventListener("click", () => {
  if (!ALL_TABS.some(t => state.tabs[t].rows.length)) return alert("Load a CSV first.");
  $("#exportModal").classList.add("show");
});
```
to:
```js
$("#btnExport").addEventListener("click", () => {
  if (!ALL_TABS.some(t => state.tabs[t].rows.length)) return alert("Load a CSV first.");
  // Assets have no Unchecked state — hide that scope, and reset it if selected.
  const uncheckedOpt = document.querySelector('#expScope option[value="unchecked"]');
  if (uncheckedOpt) {
    uncheckedOpt.hidden = (state.active === "assets");
    if (uncheckedOpt.hidden && $("#expScope").value === "unchecked") $("#expScope").value = "all";
  }
  $("#exportModal").classList.add("show");
});
```

- [ ] **Step 5: Relabel the download button (it's not always CSV now)**

Change `index.html:442` from:
```html
      <button id="expDownload" class="btn-accent">Download CSV</button>
```
to:
```html
      <button id="expDownload" class="btn-accent">Download</button>
```

- [ ] **Step 6: Verify**

Reload, load an assets CSV. Mark a few Accounted, one Damaged, add one unlisted item, then Export → Download. Expected:
- File downloads as `assets-YYYY-MM-DD-HHMM.xlsx`.
- Open in **Google Sheets**: 9 columns `Asset ID · Cat Class · S/N · Make · Model · Market/Branch · Status · Added · Notes`, header row gray/bold, and row colors: accounted=green, damaged=red, missing=yellow, the added row=orange (orange even if you also marked it accounted). Status column reads `Accounted For` / `Damaged` / `Missing`; the added row's `Added` cell = `Yes`.
- Open the same file in **Excel**: same colors, no "format doesn't match" prompt.
- Switch to a Parts or Bulk tab and Export → still a `.csv` with the old columns.
- Open Export while on the Assets tab: the scope dropdown has **no** "Unchecked only" option; on Parts/Bulk it still does.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(export): assets export as colored .xlsx; parts/bulk stay CSV"
```

---

### Task 9: Full QA pass + tag v1.7.0

**Files:** none (verification + tag)

- [ ] **Step 1: End-to-end manual QA**

Reload `index.html` fresh and confirm the full spec (§ "Testing" in the spec):
1. Load assets CSV → all rows Missing by default; two buttons only.
2. Accounted toggles green↔Missing; Damaged toggles red-badge↔Missing.
3. Add unlisted item → flagged Added.
4. Export → `.xlsx` opens in Google Sheets **and** Excel with correct columns + colors (added=orange wins over damaged/accounted).
5. Scope exports (Missing / Damaged / Exceptions) contain the right rows.
6. Parts/Bulk unchanged: CSV, Match/Short/Over, three buttons, Unchecked chip present.
7. Reload the page after marking some assets → progress persisted (localStorage), untouched rows still show Missing.

- [ ] **Step 2: Tag the release**

```bash
git tag v1.7.0
```

- [ ] **Step 3: (Optional) push**

Only if the user asks:
```bash
git push origin main --tags
```

---

## Self-Review

**Spec coverage:**
- Part A (Missing default, two buttons, toggle→Missing, chips, jump, summary): Tasks 2, 3, 4. ✓
- Part B (`.xlsx`, columns, colors, Added column, internals, filename, wiring): Tasks 5, 6, 7, 8. ✓
- Part C (IM contract): documented in spec; not implemented here (user's separate work). ✓
- Part D (version bump, header literal): Task 1. ✓
- Testing section: Task 9. ✓

**Placeholder scan:** No TBD/TODO; every code step shows complete code. ✓

**Type/name consistency:** `effectiveStatus`, `filterRowsForExport`, `buildAssetsXlsx`, `zipStore`, `crc32`, `concatBytes`, `xlsxCell`, `colLetter`, `assetRowStyle`, `assetStatusText`, `exportName`, `downloadOne`, `buildExportFiles` — names used consistently across tasks. Style indices (1=header,2=green,3=yellow,4=red,5=orange) match `assetRowStyle` and `XLSX_STYLES` `cellXfs` order. ✓

**Note:** `buildCSVFor` retains its assets branch but is no longer called for assets (assets route to `buildAssetsXlsx`); left intact as it still serves Parts/Bulk. Harmless dead branch, not worth a separate task.

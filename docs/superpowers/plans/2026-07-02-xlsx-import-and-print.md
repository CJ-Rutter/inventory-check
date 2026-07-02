# Inventory Check — `.xlsx` Import + Print Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Inventory Check reads a corrected Assets `.xlsx` back in (closing the IM↔IC round-trip) and gains a Print button that outputs a clean, controls-stripped item list of the current tab.

**Architecture:** Single-file `index.html`, no build, no framework, no deps. A store-only `.xlsx` reader (ported verbatim from Inventory Manager) turns the file into the grid the existing ingest expects; the import path learns the new Status/Notes/Added vocabulary. Print builds a hidden `#printArea` shown only via `@media print`.

**Tech Stack:** Vanilla HTML/CSS/JS. Verification: manual-in-browser + Node for the pure reader.

**Spec:** `docs/superpowers/specs/2026-07-02-xlsx-import-and-print-design.md`

**Status codes:** `"ok"`=Accounted/Match, `"bad"`=Missing/Short, `"warn"`=Damaged/Over, `""`=untouched. Assets `effectiveStatus` already defaults `""`→`"bad"` (Missing).

---

### Task 1: Version bump to v1.8.0

**Files:** Modify `index.html:555`

- [ ] **Step 1: Bump `APP_VERSION`**

Change `index.html:555` from:
```js
const APP_VERSION = "1.7.0";
```
to:
```js
const APP_VERSION = "1.8.0";
```

- [ ] **Step 2: Verify** — `grep -n 'APP_VERSION = "1.8.0"' index.html` → 1 hit. Open in a browser: footer reads v1.8.0.

- [ ] **Step 3: Commit**
```bash
git add index.html
git commit -m "chore: bump Inventory Check to v1.8.0"
```

---

### Task 2: Port the `.xlsx` reader (`parseXlsxGrid`)

Insert **immediately before** `function ingestCSV` (find it: `grep -n 'function ingestCSV' index.html`, insert directly above that line). Verbatim copy of Inventory Manager's reader (includes the self-closing-tag regex fix).

**Files:** Modify `index.html` (insert before `ingestCSV`)

- [ ] **Step 1: Insert the reader**

```js
// ---------- .xlsx reader: store-only unzip + inline/shared strings → grid ----------
// Reads a store-only (uncompressed) .xlsx (as produced by Inventory Check / Manager).
// A file re-saved by Excel/Sheets is DEFLATE-compressed and throws XLSX_UNSUPPORTED.
function xlsxReadParts(buf) {
  const bytes = new Uint8Array(buf);
  const dv = new DataView(buf);
  const dec = new TextDecoder("utf-8");
  let eocd = -1;
  for (let i = bytes.length - 22; i >= 0; i--) {
    if (dv.getUint32(i, true) === 0x06054b50) { eocd = i; break; }
  }
  if (eocd < 0) throw new Error("XLSX_UNSUPPORTED");
  const cdCount = dv.getUint16(eocd + 10, true);
  let off = dv.getUint32(eocd + 16, true);
  const parts = {};
  for (let n = 0; n < cdCount; n++) {
    if (dv.getUint32(off, true) !== 0x02014b50) throw new Error("XLSX_UNSUPPORTED");
    const nameLen = dv.getUint16(off + 28, true);
    const extraLen = dv.getUint16(off + 30, true);
    const commentLen = dv.getUint16(off + 32, true);
    const compSize = dv.getUint32(off + 20, true);
    const localOff = dv.getUint32(off + 42, true);
    const name = dec.decode(bytes.subarray(off + 46, off + 46 + nameLen));
    if (dv.getUint32(localOff, true) !== 0x04034b50) throw new Error("XLSX_UNSUPPORTED");
    const lMethod = dv.getUint16(localOff + 8, true);
    const lNameLen = dv.getUint16(localOff + 26, true);
    const lExtraLen = dv.getUint16(localOff + 28, true);
    const dataStart = localOff + 30 + lNameLen + lExtraLen;
    if (lMethod !== 0) {
      if (name === "xl/worksheets/sheet1.xml" || name === "xl/sharedStrings.xml") throw new Error("XLSX_UNSUPPORTED");
    } else {
      parts[name] = dec.decode(bytes.subarray(dataStart, dataStart + compSize));
    }
    off += 46 + nameLen + extraLen + commentLen;
  }
  return parts;
}
function xmlUnescape(s) {
  return String(s).replace(/&lt;/g, "<").replace(/&gt;/g, ">").replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'").replace(/&apos;/g, "'").replace(/&amp;/g, "&");
}
function colIndexFromRef(ref) {
  const m = /^([A-Z]+)/.exec(ref);
  if (!m) return 0;
  let n = 0;
  for (const ch of m[1]) n = n * 26 + (ch.charCodeAt(0) - 64);
  return n - 1;
}
function parseXlsxGrid(buf) {
  const parts = xlsxReadParts(buf);
  const sheet = parts["xl/worksheets/sheet1.xml"];
  if (!sheet) throw new Error("XLSX_UNSUPPORTED");
  let shared = [];
  if (parts["xl/sharedStrings.xml"]) {
    shared = (parts["xl/sharedStrings.xml"].match(/<si>[\s\S]*?<\/si>/g) || []).map(si =>
      (si.match(/<t[^>]*>([\s\S]*?)<\/t>/g) || [])
        .map(x => xmlUnescape(x.replace(/<t[^>]*>/, "").replace(/<\/t>/, ""))).join(""));
  }
  const grid = [];
  const rows = sheet.match(/<row[^>]*\/>|<row[^>]*>[\s\S]*?<\/row>/g) || [];
  for (const rowXml of rows) {
    const cells = [];
    const cMatches = rowXml.match(/<c\b[^>]*\/>|<c\b[^>]*>[\s\S]*?<\/c>/g) || [];
    for (const cXml of cMatches) {
      const refM = /r="([A-Z]+\d+)"/.exec(cXml);
      const idx = refM ? colIndexFromRef(refM[1]) : cells.length;
      const typeM = /t="([^"]+)"/.exec(cXml);
      const type = typeM ? typeM[1] : "";
      let val = "";
      if (type === "inlineStr") {
        const tM = /<t[^>]*>([\s\S]*?)<\/t>/.exec(cXml);
        val = tM ? xmlUnescape(tM[1]) : "";
      } else if (type === "s") {
        const vM = /<v>([\s\S]*?)<\/v>/.exec(cXml);
        val = vM ? (shared[Number(vM[1])] ?? "") : "";
      } else {
        const vM = /<v>([\s\S]*?)<\/v>/.exec(cXml);
        val = vM ? xmlUnescape(vM[1]) : "";
      }
      while (cells.length < idx) cells.push("");
      cells[idx] = val;
    }
    grid.push(cells);
  }
  return grid;
}
```

- [ ] **Step 2: Node round-trip test**

Scratch dir: `/tmp/claude-1000/-home-cj-development-nebula-sentinel/24251f1e-2a18-4be2-8fe2-4de42df445d0/scratchpad`

If `<scratch>/assets-test.xlsx` (a real IC/IM 9-column sample from earlier this session) still exists, create `<scratch>/ic-xlsx-read-check.mjs` that pastes copies of `xlsxReadParts`/`xmlUnescape`/`colIndexFromRef`/`parseXlsxGrid`, reads that file (`const buf=fs.readFileSync(p); parseXlsxGrid(buf.buffer.slice(buf.byteOffset,buf.byteOffset+buf.byteLength))`), and asserts `grid[0]` deep-equals `["Asset ID","Cat Class","S/N","Make","Model","Market/Branch","Status","Added","Notes"]`; print `READ_OK`.
If the fixture is absent, instead build a store-only `.xlsx` in the script (paste `crc32`/`concatBytes`/`zipStore` from `inventory-manager/index.html`, write a `sheet1.xml` with a header row `Asset ID, Status, Notes` + one data row using inline strings, `zipStore` it) and assert `parseXlsxGrid` returns those two rows; print `READ_OK`.
Run: `node <scratch>/ic-xlsx-read-check.mjs`. Fix and re-run if it throws/mismatches. Do not commit scratch files.

- [ ] **Step 3: Commit**
```bash
git add index.html
git commit -m "feat(import): add store-only .xlsx reader (parseXlsxGrid)"
```

---

### Task 3: Split `ingestCSV` → `ingestGrid` + dispatch by file type

**Files:** Modify `index.html:905-911` (`ingestCSV` head), `index.html:1371-1375` (`#file` handler), `index.html:392` (`accept`)

- [ ] **Step 1: Split into a thin wrapper + `ingestGrid`**

Change `index.html:905-911` from:
```js
function ingestCSV(text, fileName) {
  const grid = parseCSV(text);
  if (!grid.length) return alert("CSV appears empty.");
  const headers = grid[0].map((h, i) => h && h.trim() ? h.trim() : `col_${i}`);
  const body = grid.slice(1);
  const kind = classifyCSV(headers);
  if (!kind) return alert("Couldn't tell if this is an Assets or Bulk CSV. Expected 'Asset ID' or 'PART #' column.");
```
to:
```js
function ingestCSV(text, fileName) {
  ingestGrid(parseCSV(text), fileName);
}
function ingestGrid(grid, fileName) {
  if (!grid.length) return alert("File appears empty.");
  const headers = grid[0].map((h, i) => h && h.trim() ? h.trim() : `col_${i}`);
  const body = grid.slice(1);
  const kind = classifyCSV(headers);
  if (!kind) return alert("Couldn't tell if this is an Assets or Bulk file. Expected 'Asset ID' or 'PART #' column.");
```
(Everything below — the round-trip column detection onward — is unchanged and now belongs to `ingestGrid`.)

- [ ] **Step 2: Dispatch `.xlsx` vs `.csv` in the `#file` handler**

Change `index.html:1371-1375` from:
```js
$("#file").addEventListener("change", (ev) => {
  const f = ev.target.files?.[0]; if (!f) return;
  const fr = new FileReader();
  fr.onload = () => { ingestCSV(String(fr.result || ""), f.name); ev.target.value = ""; };
  fr.readAsText(f);
});
```
to:
```js
$("#file").addEventListener("change", (ev) => {
  const f = ev.target.files?.[0]; if (!f) return;
  const isXlsx = /\.xlsx$/i.test(f.name);
  const fr = new FileReader();
  fr.onerror = () => { alert(`${f.name}: couldn't read this file.`); ev.target.value = ""; };
  fr.onload = () => {
    try {
      if (isXlsx) ingestGrid(parseXlsxGrid(fr.result), f.name);
      else ingestGrid(parseCSV(String(fr.result || "")), f.name);
    } catch (e) {
      if (e && e.message === "XLSX_UNSUPPORTED") {
        alert(`${f.name}: this .xlsx looks re-saved by another app. Please export a fresh file from Inventory Manager or Inventory Check.`);
      } else {
        alert(`${f.name}: couldn't read this file.`);
      }
    }
    ev.target.value = "";
  };
  if (isXlsx) fr.readAsArrayBuffer(f);
  else fr.readAsText(f);
});
```

- [ ] **Step 3: Add `.xlsx` to the file input `accept`**

Change `index.html:392` from:
```html
    <input type="file" id="file" accept=".csv,text/csv" />
```
to:
```html
    <input type="file" id="file" accept=".csv,text/csv,.xlsx,application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" />
```

- [ ] **Step 4: Verify** — `grep -n 'function ingestGrid' index.html` → 1 hit; `grep -n 'ingestGrid(parseXlsxGrid(fr.result)' index.html` → 1 hit; `grep -c 'accept=".csv,text/csv,.xlsx' index.html` → 1. JS syntax gate:
```bash
python3 -c "import re; h=open('index.html').read(); s=re.findall(r'<script[^>]*>(.*?)</script>', h, re.S); open('/tmp/ic_t3.js','w').write('\n;\n'.join(s))" && node --check /tmp/ic_t3.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 5: Commit**
```bash
git add index.html
git commit -m "feat(import): dispatch .xlsx vs .csv into shared ingestGrid"
```

---

### Task 4: Teach the import the new Status/Notes/Added vocabulary

**Files:** Modify `index.html:916-926` (round-trip column reads + inline `decodeStatus`, now inside `ingestGrid`)

- [ ] **Step 1: Add header aliases + the "Accounted For" label**

Change this block (inside `ingestGrid`):
```js
  const checkStatusCol = findCol(headers, ["Check Status"]);
  const checkNoteCol   = findCol(headers, ["Check Note"]);
  const countedCol     = findCol(headers, ["Counted Qty"]);
  const addedCol       = findCol(headers, ["Added Manually"]);
  const decodeStatus = (raw) => {
    const s = String(raw || "").trim().toLowerCase();
    if (s === "accounted" || s === "match") return "ok";
    if (s === "missing"   || s === "short") return "bad";
    if (s === "damaged"   || s === "over")  return "warn";
    return "";
  };
```
to:
```js
  // Old export headers first so existing round-trip CSVs bind to them; the new
  // .xlsx from Inventory Manager uses the short names (Status/Notes/Added).
  const checkStatusCol = findCol(headers, ["Check Status", "Status"]);
  const checkNoteCol   = findCol(headers, ["Check Note", "Notes"]);
  const countedCol     = findCol(headers, ["Counted Qty"]);
  const addedCol       = findCol(headers, ["Added Manually", "Added"]);
  const decodeStatus = (raw) => {
    const s = String(raw || "").trim().toLowerCase();
    if (s === "accounted" || s === "accounted for" || s === "match") return "ok";
    if (s === "missing"   || s === "short") return "bad";
    if (s === "damaged"   || s === "over")  return "warn";
    return "";
  };
```

- [ ] **Step 2: Verify** — `grep -n '"Check Status", "Status"' index.html` → 1 hit; `grep -n 'accounted for' index.html` → 1 hit. Syntax gate (as Task 3 Step 4) → `SYNTAX_OK`.

- [ ] **Step 3: Commit**
```bash
git add index.html
git commit -m "feat(import): restore status/note/added from new xlsx headers (Status/Notes/Added, Accounted For)"
```

---

### Task 5: Print — button, print area, and print CSS

**Files:** Modify `index.html` (header buttons `~369`, `<body>` for `#printArea`, `<style>` for print CSS, and add `renderPrintArea` + button handler near the other `$("#btn…")` handlers)

- [ ] **Step 1: Add the Print button**

In the header controls, after the Export button (`index.html`, the line `    <button class="btn-ghost" id="btnExport">Export</button>`), add on the next line:
```html
    <button class="btn-ghost" id="btnPrint" title="Print current tab">Print</button>
```

- [ ] **Step 2: Add the `#printArea` container as a direct child of `<body>`**

Find the first `<script` tag in the file. Immediately BEFORE it, add:
```html
<div id="printArea"></div>
```
(It must be a direct child of `<body>` so the print CSS can isolate it. It's hidden on screen.)

- [ ] **Step 3: Add print CSS**

Inside the existing `<style>` block, append these rules (just before `</style>`):
```css
  #printArea { display: none; }
  @media print {
    body > *:not(#printArea) { display: none !important; }
    #printArea { display: block !important; }
    #printArea .print-title { font-weight: 700; font-size: 13px; margin-bottom: 2px; }
    #printArea .print-totals { font-size: 12px; margin-bottom: 10px; }
    #printArea .print-table { width: 100%; border-collapse: collapse; font-size: 11px; }
    #printArea .print-table th, #printArea .print-table td { border: 1px solid #999; padding: 3px 6px; text-align: left; vertical-align: top; }
    #printArea .print-table thead { display: table-header-group; }
    #printArea .print-table tr { page-break-inside: avoid; }
  }
```

- [ ] **Step 4: Add `renderPrintArea()` + the button handler**

Add `renderPrintArea` next to the other render helpers (anywhere at top level; e.g. just before the `// ---------- Events ----------`-style handler section). It reads the **current tab, all rows** (filter ignored):
```js
function renderPrintArea() {
  const kind = state.active;
  const tab = state.tabs[kind];
  const cols = tab.cols;
  const isBulk = isBulkLike(kind);
  const titleByKind = { assets: "Assets", parts: "Parts", bulk: "Bulk" };
  let ok = 0, bad = 0, warn = 0;
  for (const r of tab.rows) {
    const s = effectiveStatus(r, kind, cols);
    if (s === "ok") ok++; else if (s === "bad") bad++; else if (s === "warn") warn++;
  }
  const totals = isBulk
    ? `Total: ${tab.rows.length} · Match: ${ok} · Short: ${bad} · Over: ${warn}`
    : `Total: ${tab.rows.length} · Accounted: ${ok} · Missing: ${bad} · Damaged: ${warn}`;
  const d = new Date(), p = (n) => String(n).padStart(2, "0");
  const date = `${d.getFullYear()}-${p(d.getMonth() + 1)}-${p(d.getDate())}`;
  const headCols = isBulk
    ? ["Part #", "Bin", "Description", "Count", "Status", "Note"]
    : ["Asset ID", "Cat Class", "Make/Model", "Status", "Note"];
  const rowsHtml = tab.rows.map((r) => {
    const label = statusLabel(effectiveStatus(r, kind, cols), isBulk);
    let cells;
    if (isBulk) {
      const count = `${r._counted ?? ""}/${r[cols.avail] ?? ""}`;
      cells = [r[cols.id], r[cols.bin], r[cols.desc], count, label, r._note || ""];
    } else {
      const mm = [r[cols.make], r[cols.model]].filter(Boolean).join(" ");
      cells = [r[cols.id], r[cols.cat], mm, label, r._note || ""];
    }
    return `<tr>${cells.map((c) => `<td>${escapeHtml(String(c ?? ""))}</td>`).join("")}</tr>`;
  }).join("");
  $("#printArea").innerHTML =
    `<div class="print-title">Inventory Check · ${escapeHtml(titleByKind[kind] || kind)} · ${escapeHtml(tab.fileName || "")} · ${date}</div>` +
    `<div class="print-totals">${escapeHtml(totals)}</div>` +
    `<table class="print-table"><thead><tr>${headCols.map((h) => `<th>${h}</th>`).join("")}</tr></thead>` +
    `<tbody>${rowsHtml}</tbody></table>`;
}
$("#btnPrint").addEventListener("click", () => {
  if (!state.tabs[state.active].rows.length) return alert("Load a CSV first.");
  renderPrintArea();
  window.print();
});
```

- [ ] **Step 5: Verify** — `grep -n 'id="btnPrint"' index.html` → 1 hit; `grep -n 'id="printArea"' index.html` → 1 hit (the div); `grep -n 'function renderPrintArea' index.html` → 1 hit; `grep -n 'body > \*:not(#printArea)' index.html` → 1 hit. Syntax gate → `SYNTAX_OK`.

Browser check (human): load an Assets CSV, apply the Missing filter chip, click Print → the print preview shows the header line + totals + **every** asset (filter ignored), plain status labels, no buttons/inputs. Switch to a Parts tab → Part#/Bin/Description/Count columns + Match/Short/Over totals.

- [ ] **Step 6: Commit**
```bash
git add index.html
git commit -m "feat: Print button — clean item list of current tab with totals header"
```

---

### Task 6: Full QA + tag v1.8.0

**Files:** none (verification + tag)

- [ ] **Step 1: Manual QA** (mirrors spec "Testing")
1. Corrected Assets `.xlsx` from Inventory Manager → loads into IC; corrected row shows Accounted, others Missing; notes restored.
2. Fresh source CSV → still loads; assets default Missing.
3. Excel-re-saved `.xlsx` → clear "re-saved" alert, no crash.
4. Print on Assets (with a filter active) → header + totals + all items, controls stripped. Print on Parts → parts columns.

- [ ] **Step 2: JS syntax gate**
```bash
cd /home/cj/development/inventory-check
python3 -c "import re; h=open('index.html').read(); s=re.findall(r'<script[^>]*>(.*?)</script>', h, re.S); open('/tmp/ic_final.js','w').write('\n;\n'.join(s))"
node --check /tmp/ic_final.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 3: Tag**
```bash
git tag v1.8.0
```

---

## Self-Review

**Spec coverage:**
- Part A1 (reader port): Task 2. ✓
- Part A2 (split + dispatch + accept + wording): Task 3. ✓
- Part A3 (aliases + decodeStatus): Task 4. ✓
- Part B (Print button + printArea + CSS + renderPrintArea, current tab all items, totals header): Task 5. ✓
- Part C (version + tag): Task 1 + Task 6. ✓
- Testing: Task 6. ✓

**Placeholder scan:** No TBD/TODO; every code step shows complete code. ✓

**Type/name consistency:** `xlsxReadParts`/`xmlUnescape`/`colIndexFromRef`/`parseXlsxGrid` (reader), `ingestCSV`→`ingestGrid`, `renderPrintArea`, existing `effectiveStatus`/`statusLabel`/`isBulkLike`/`escapeHtml`/`$` — all consistent. The reader block matches Inventory Manager's committed version (self-closing regex fix included). `ingestGrid` defined in Task 3 is called by the Task 3 wrapper and the xlsx dispatch. ✓

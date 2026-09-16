# architecture.md — Sales Summary & Forecasting Dashboard

## 1. System Overview

### 1.1 High-Level Architecture Flow

```
┌─────────────────┐
│  Excel Upload    │  <input type="file"> — user selects
│  (.xlsx file)    │  สรุปรายงานขาย.xlsx from disk
└────────┬─────────┘
         │ FileReader.readAsArrayBuffer
         ▼
┌─────────────────────────┐
│  SheetJS ArrayBuffer     │  XLSX.read(data, {type:'array'})
│  Parser                  │  → workbook → first sheet
│                           │  → sheet_to_json(header:1)
└────────┬─────────────────┘
         │ raw rows (2D array)
         ▼
┌─────────────────────────┐
│  Filter / Sanitizer      │  Drop structural/noise rows
│  Pipeline                 │  (Thai header/footer markers)
│                           │  Keep only HS-prefixed invoice rows
└────────┬─────────────────┘
         │ clean invoice objects[]
         ▼
┌─────────────────────────┐
│  In-Memory State Store   │  Single JS object/array held in
│                           │  module scope. No persistence.
│                           │  Cleared on reload/re-upload.
└────────┬─────────────────┘
         │ invoices[]
         ▼
┌─────────────────────────┐
│  Pure Math Functions     │  runRate, forecastRange,
│                           │  targetPace, mape — no DOM access
└────────┬─────────────────┘
         │ computed metrics/results
         ▼
┌─────────────────────────┐
│  Chart.js & DOM Views    │  KPI cards, decision banner,
│                           │  trend/salesperson/customer charts,
│                           │  invoice table
└──────────────────────────┘
```

### 1.2 Core Design Philosophy
- **Single-page application (SPA):** one `index.html`, no routing, no
  navigation — everything the manager needs is visible on one screen
  after upload.
- **100% memory-bound:** all state (raw rows, parsed invoices, computed
  metrics) lives in JavaScript variables for the lifetime of the page.
  Nothing is written to disk, localStorage, or a server.
- **Client-side data isolation:** the uploaded file — and everything
  derived from it — never leaves the browser tab. Closing or reloading
  the page discards all data by design; this is a deliberate trade-off
  for zero-backend simplicity, not an oversight (see `prd.md` Out of
  Scope).

## 2. Technical Choices & Justification

| Technology | Delivery | Justification |
|---|---|---|
| **Tailwind CSS** | CDN `<script>` | Rapid, consistent utility-class styling directly in markup, with no build step, no local CSS files, and no PostCSS/compiler pipeline — fits the single-file constraint exactly. |
| **SheetJS (xlsx.full.min.js)** | CDN `<script>` | The only practical way to parse binary/ArrayBuffer `.xlsx` data entirely client-side in the browser, without a server-side conversion step. |
| **Chart.js** | CDN `<script>` | Lightweight, canvas-based rendering; declarative dataset/options API is enough for the three required chart types (line, bar, doughnut) without the overhead of a heavier charting or framework dependency. |

All three are loaded as global `<script>` tags — no module bundler, no
`import` resolution, no `package.json`.

## 3. System Structure & Component Layout

Single `index.html`, organized top-to-bottom as follows:

1. **Header & File Upload Zone**
   Title, short instruction text, and the `<input type="file">` control
   that triggers the parsing pipeline on `change`.

2. **KPI Metric Cards Section**
   Four cards: Total Sales Revenue, Total Invoices, Unique Customers,
   Average Invoice Value (Revenue and Average visually elevated per PRD).

3. **Decision & Forecast Banner**
   The primary decision surface: Target Pace signal (`AHEAD` /
   `ON_TRACK` / `BEHIND`), the Low/Base/High forecast range, and the
   user-editable Monthly Target input field.

4. **Visualization Grid**
   - Daily Sales Trend chart (line), with dashed run-rate projection
     overlay — the primary decision chart.
   - Salesperson Performance chart (bar) — secondary drill-down.
   - Top 5 Customers chart (doughnut) — secondary drill-down.

5. **Invoice Data Table Preview**
   Most recent 10 parsed transactions: date, invoice no., customer,
   salesperson, amount — a sanity-check surface against the charts above.

6. **Pure Math Engine (JS module/section)**
   Isolated, side-effect-free functions: `runRate`, `rangeProjection`,
   `targetPace`, `mape`. No DOM references anywhere in this section.

7. **State Manager & DOM Updater (JS module/section)**
   Owns the in-memory invoice array and derived-metrics object; on any
   state change, calls the Math Engine for fresh numbers and then writes
   results into the DOM/Chart.js instances. This is the only section
   permitted to touch `document.*` or Chart.js instances.

## 4. Data Flow & Parsing Pipeline

Step-by-step row extraction, in order:

1. **Binary → JSON rows**
   `XLSX.read(arrayBuffer, { type: 'array' })` loads the workbook;
   `XLSX.utils.sheet_to_json(firstSheet, { header: 1, defval: "" })`
   converts the first sheet into a 2D array of raw rows.

2. **Hardcoded Thai String Filter**
   Each row is checked against known structural/noise markers, and
   dropped if matched:
   `หลังคาเหล็ก`, `รายงานขายเงินสด`, `วันที่จาก`, `รหัสลูกค้า`,
   `พนักงานขาย`, `---`, `วันที่`, `รายละเอียด`, `รวม`,
   `ตัดใบรับมัดจำ`.
   This removes report titles, section headers, subtotal lines, and
   deposit-offset lines — none of which represent a real invoice.

3. **Invoice Extractor**
   Of the remaining rows, only those where
   `col1.startsWith(INVOICE_PREFIX)` (with `INVOICE_PREFIX = 'HS'`) are
   treated as valid invoice rows. Everything else is discarded as
   non-matching noise.

4. **Data Normalizer**
   Each matched row is mapped into a clean, consistently-shaped
   JavaScript object:
   ```
   { date, invNo, custCode, custName, salesperson, grandTotal }
   ```
   `grandTotal` is parsed from the raw cell string (commas stripped) to
   a `Number`, defaulting safely to `0` on parse failure.

The output of this pipeline — an array of normalized invoice objects —
is the single source of truth held in the In-Memory State Store, from
which every metric, chart, and table row is derived.

## 5. Calculation Engine (Pure Functions)

All functions below take explicit primitive/array inputs and return
plain values or objects. None reference the DOM, `window`, or Chart.js.

```
calculateRunRate(salesToDate, elapsedWorkingDays, totalWorkingDays)
  → number
  Projects month-end revenue at the current daily pace.

calculateForecastRange(runRate, variancePercentage)
  → { low, base, high }
  Wraps the run rate in a bounded range rather than a single point
  estimate. variancePercentage is a tunable constant (see note above)
  until real month-over-month variance data is available.

determineTargetPace(baseProjection, monthlyTarget)
  → 'AHEAD' | 'ON_TRACK' | 'BEHIND'
  Compares the Base projection against the user-entered Monthly Target
  to produce the decision-banner signal.

mape(forecastValues, actualValues)
  → number
  Mean Absolute Percentage Error, for post-hoc accuracy tracking once
  closed-month actuals are available (manual comparison in this MVP;
  see prd.md Out of Scope for automated tracking).
```

Every function here must have a corresponding assertion in `runChecks()`
per `AGENTS.md` Hard Rules — no exceptions.

## 6. Error Handling & Edge Cases

- **Empty file / zero matching rows:** if the parsed invoice array is
  empty after filtering, the UI shows an explicit empty state ("No
  invoices found in this file") on every KPI card, chart, and the table
  — never a blank canvas or a silently-zeroed dashboard that looks like
  a real (bad) result.
- **Division by zero safety:**
  - `elapsedWorkingDays === 0` → `calculateRunRate` returns `0` (or a
    flagged "too early" state) rather than `Infinity`/`NaN`.
  - `totalInvoices === 0` → Average Invoice Value renders as `฿0.00`,
    not `NaN`.
  - Every division in the Math Engine is guarded with an explicit
    zero-denominator check before the operation runs.
- **Malformed numeric cells:** any `grandTotal` that fails to parse as a
  number defaults to `0` and does not throw, so one bad row cannot break
  ingestion of the rest of the file.
- **Console check assertions (`runChecks()`):**
  A dedicated function that runs a fixed set of hand-verified
  input/expected-output pairs against every Math Engine function and
  logs `PASS`/`FAIL` per case to the DevTools Console. This is run
  manually before every commit per the Definition of Done in
  `AGENTS.md`, and must be updated in the same change whenever a Math
  Engine function is modified.

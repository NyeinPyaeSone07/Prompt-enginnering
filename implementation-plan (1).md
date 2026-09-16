# implementation-plan.md — Sales Summary & Forecasting Dashboard

## 1. Overview

### Milestone Discipline Rules
- Every milestone (M0–M7) must produce a **runnable `index.html`** file
  that opens locally in Chrome and is verified in DevTools before the
  next milestone begins.
- **Strict rule: no multi-milestone prompts.** Each prompt to the AI
  targets exactly one milestone. A milestone is not "done" until it has
  been opened, exercised, and checked against its own verification
  criteria — per `AGENTS.md` §3 (Hard Rules) and §4 (Definition of Done).
- Each milestone must be independently verifiable: a teammate other than
  the one who prompted it must be able to run it and confirm it meets
  its objective before work proceeds to the next milestone.
- Commit after every milestone that passes verification
  (`git add . && git commit -m "M<n> <short description>" && git push`).

## 2. Milestone Breakdown (M0–M7)

### M0 — Scaffold & Layout
**Objective:**
Produce a single `index.html` containing the static layout only, with no
business logic and no data: header, drag-drop file upload zone, KPI
metric grid (empty), chart containers (empty `<canvas>` elements, not
placeholder icons/SVGs — Chart.js needs a real canvas to attach to in
M4), the forecast decision banner (empty/placeholder state), and an
empty invoice table. All three CDN `<script>` tags (Tailwind CSS,
Chart.js, SheetJS) are added to `<head>` in this milestone even though
unused until M1/M4, per `architecture.md` §2 and `AGENTS.md` §1 —
loading them once now means the `<head>` never needs revisiting in
later milestones.

**Verification:**
- File opens locally by double-click in Chrome with **zero console
  errors or warnings**, including from the three CDN scripts loading.
- All layout regions are visibly present and cleanly styled at both
  1920×1080 and 1366×768 (per `AGENTS.md` DoD), even with no data loaded.
- Confirm all three CDN `<script>` tags are present in `<head>` (view
  source or DevTools Elements), and confirm styling is implemented with
  Tailwind utility classes rather than a separate hand-written
  `<style>` block, per the locked stack in `prd.md`.
- Confirm chart containers are real `<canvas id="...">` elements, not
  placeholder icons — so M4 can attach Chart.js directly without a
  markup change.

---

### M1 — Data Parsing & Seed Data Engine
**Objective:**
Integrate SheetJS (`xlsx.full.min.js`) for file parsing, and add an
embedded 30-row seed dataset so the app has realistic data to develop
against without requiring a live file upload. Implement the Thai
structural-row filter (`รายงานขายเงินสด`, `รวม`, `ตัดใบรับมัดจำ`, and
the remaining markers listed in `architecture.md` §4) and the `HS`
invoice-number matcher (`INVOICE_PREFIX = 'HS'`).

**Verification:**
- Parsed invoice array is logged to the DevTools Console.
- Manual inspection confirms structural/noise rows are correctly
  stripped and only valid `HS`-prefixed rows remain, matching the
  `Invoice` schema in `schema.md` §3.1.

---

### M2 — Pure Math & Forecast Engine Isolation
**Objective:**
Build the pure, side-effect-free JS math functions: `runRate`,
`forecastRange`, `targetPace`, `mape` (per `architecture.md` §5 and
`schema.md` §4). Implement the `runChecks()` console test suite.

**Verification:**
- `runChecks()` logs explicit **PASS/FAIL** results for 5 hand-calculated
  test cases in the DevTools Console, including the `ForecastInput` /
  `ForecastOutput` fixture pair defined in `schema.md` §5.
- No function in this milestone touches the DOM.

---

### M3 — Headline Metrics & Target Pace Banner UI
**Objective:**
Wire the parsed invoice data and Math Engine outputs into the DOM.
Render Total Sales Revenue, Total Invoices, Unique Customers, Average
Invoice Value, and the Pace Indicator badge (`AHEAD` / `ON_TRACK` /
`BEHIND`) in the forecast banner.

**Verification:**
- Loading the seed dataset (or a sample file) updates all four metric
  cards and the forecast banner dynamically, with values matching a
  manual hand-check against the same data.

---

### M4 — Interactive Decision Visualizations
**Objective:**
Initialize and wire up the three Chart.js instances: Daily Sales Trend
(line chart) with the dashed run-rate projection overlay, Salesperson
Performance (bar chart), and Top 5 Customers (doughnut chart).

**Verification:**
- All three charts render clearly with seed/sample data.
- Existing chart instances are destroyed before re-render on re-upload
  (no duplicate/ghost canvases).
- Charts resize correctly on window resize / responsive breakpoints.

---

### M5 — Recent Invoices Table & Data Persistence/Export
**Objective:**
Populate the 10-row recent invoices preview table with correctly
formatted dates and currency. Add a CSV export feature for the current
filtered/aggregated summary records.

**Verification:**
- Table renders 10 clean, correctly formatted invoice rows matching the
  underlying parsed data.
- CSV download triggers on demand and the exported file's contents
  match what's displayed on screen.

---

### M6 — User Target Edit & UX Polish
**Objective:**
Add the interactive monthly revenue target input field (per `prd.md`
§4.4 / `schema.md` §4.1 `targetRevenue`), allowing the user to adjust
the value and see the Pace Indicator recalculate live. Polish responsive
layout, and add explicit empty-state and error-state banners.

**Verification:**
- Changing the target input instantly updates the Pace Indicator
  (`AHEAD` / `ON_TRACK` / `BEHIND`) without a page reload.
- Uploading an invalid or empty file shows a clear, user-friendly error
  banner instead of a JS crash or blank dashboard.

---

### M7 — Edge Case Hardening & Pre-Ship Checklist
**Objective:**
Final hardening pass. Re-validate all 5 hand-checked test cases from M2,
confirm zero console warnings/errors across every feature path, and
confirm 100% client-side data isolation (no network calls beyond the
three CDN script loads; no data persistence attempted).

**Verification:**
- Full pre-ship checklist (per `prd.md` §8 Success Criteria and the
  workshop's Pre-Production Checklist) completed and signed off.
- Repository is tagged for deployment: `git tag v1.0-mvp && git push
  --tags`.

## 3. Milestone Verification Table

| Milestone | Deliverable | Primary Function to Test | Verification Method |
|---|---|---|---|
| **M0** | Static layout scaffold | N/A (layout only) | Open in Chrome; confirm zero console errors and clean rendering at 1920×1080 / 1366×768 |
| **M1** | Parsing + seed data + filters | Thai string filter, `HS` invoice matcher | Log parsed array to Console; manually confirm noise rows stripped, valid rows retained |
| **M2** | Pure Math Engine + test suite | `runRate`, `forecastRange`, `targetPace`, `mape` | `runChecks()` logs PASS/FAIL for 5 hand-calculated cases in Console |
| **M3** | Headline metrics + Pace banner | DOM binding of metrics + `determineTargetPace` | Load seed/sample data; confirm metric cards and banner match manual hand-check |
| **M4** | Trend/Salesperson/Customer charts | Chart.js render + overlay + instance lifecycle | Visual check; confirm no duplicate canvases on re-upload; resize test |
| **M5** | Invoice table + CSV export | Table formatting, CSV generation | Visual check of table rows; download CSV and diff against displayed data |
| **M6** | Target input + empty/error states | Live `targetRevenue` recalculation | Edit target field, confirm instant Pace Indicator update; upload bad file, confirm graceful error banner |
| **M7** | Hardened, tagged MVP | Full regression across M1–M6 | Re-run all 5 test cases; zero console warnings/errors; confirm `git tag v1.0-mvp` pushed |

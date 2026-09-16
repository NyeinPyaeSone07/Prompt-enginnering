# PRD — Sales Summary Dashboard

## What
A single self-contained HTML dashboard that turns a Thai cash-sales report
(`สรุปรายงานขาย.xlsx`, exported from an accounting/POS system) into a
readable summary: total revenue, invoice count, customer count, average
invoice value, a daily sales trend, salesperson performance, and top
customers by revenue.

## Why
The source Excel file is a raw accounting export — mixed structural rows
(report headers, totals, deposit-offset lines) interleaved with individual
invoice rows in a non-tabular layout. Nobody at the branch can glance at it
and answer "how are we doing this month, who's selling, and who are our
biggest customers?" without manually filtering it first. This dashboard
answers that in one upload, with no spreadsheet editing required.

## Who
- **Primary user:** branch/shop manager or bookkeeper who already has the
  exported `.xlsx` report on their machine.
- **Context:** shared office desktop, opened a few times a week (e.g. daily
  or weekly close), not a specialist in Excel formulas.
- **Skill level:** comfortable clicking "choose file" and reading a chart;
  not expected to understand the underlying parsing logic.

## Core features
1. As a manager, I can **upload the `.xlsx` report** via a file picker and
   see the dashboard populate immediately — no server, no login.
2. As a manager, I can see **four headline metrics**: total sales revenue,
   total invoices, unique customers, average invoice value.
3. As a manager, I can see a **daily sales trend line chart** to spot
   which days were strong or weak.
4. As a manager, I can see **sales by salesperson** in a bar chart to
   compare individual performance.
5. As a manager, I can see the **top 5 customers by revenue** in a
   doughnut chart to know who matters most.
6. As a manager, I can scan a **table of the 10 most recent invoices**
   (date, invoice no., customer, salesperson, amount) for a sanity check
   against the charts above.

## User flow
1. Open `index.html` in the browser (double-click or hosted URL).
2. Click the file input, select the exported `.xlsx` report.
3. The file is parsed client-side (SheetJS); rows are filtered down to
   valid invoice lines (identified by an `HS`-prefixed invoice number in
   column B) and structural/summary rows are discarded.
4. Metrics, three charts, and the invoice table render automatically.
5. Manager reads the headline numbers and trend, and drills into the
   charts/table for detail. No further interaction is required.

## Out of scope
- No login or multi-user accounts — anyone with the file can view it.
- No server-side storage — nothing is saved between sessions; re-upload
  is required every time the page is reopened.
- No editing of invoice data from within the dashboard.
- No support for report formats other than the current column layout
  (date / invoice no. starting with `HS` / customer code / customer name /
  salesperson / … / grand total in column H).
- No multi-branch or multi-sheet handling — only the first sheet of the
  workbook is read.
- No currency other than THB (฿).

## Success criteria
- A manager can go from "file exported" to "dashboard read" in under
  30 seconds, with zero manual spreadsheet cleanup.
- Totals shown (revenue, invoice count) match a manual count on a sample
  file, exactly.
- The dashboard does not crash or show blank charts on a report with
  zero matching invoice rows — it should show sensible zero states.

## Tech constraints
- Must remain a **single self-contained HTML file** (current
  implementation: Tailwind via CDN, Chart.js via CDN, SheetJS via CDN) —
  no build step, no backend.
- Must run entirely client-side; the uploaded file never leaves the
  browser.
- Must continue to support the specific Thai-language structural markers
  in the source file (e.g. `รายงานขายเงินสด`, `รวม`, `ตัดใบรับมัดจำ`) used
  to distinguish real invoice rows from headers/totals — these are fragile
  and should be documented, not hardcoded silently.

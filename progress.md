# progress.md — Sales Summary & Forecasting Dashboard

## 1. Project Metadata

| Field | Value |
|---|---|
| **Project Name** | Sales Summary & Forecasting Dashboard |
| **Track** | Track A (Sales Forecast Assistant) |
| **Target User** | Branch Counter Shop Manager |
| **Repository Status** | Phase 1 Complete (Design Signed Off) · Phase 2 Active (Implementation) |
| **Current Date** | September 16, 2026 |

## 2. Planning Documents Status (Phase 1 Checkpoint)

- [x] `prd.md` — Product requirements & decision scope
- [x] `AGENTS.md` — Standing instructions & AI hard rules
- [x] `architecture.md` — Data flow & UI layout
- [x] `schema.md` — Excel mapping & in-memory models
- [x] `implementation-plan.md` — Milestones M0–M7
- [x] `progress.md` — Living execution log (this file)

**Phase 1 sign-off:** all six documents are internally consistent as of
this date. Any change to `prd.md` requires the other five to be updated
in the same commit, per `architecture.md` §1 and `AGENTS.md` §3.

## 3. Implementation Milestones Tracking (Phase 2)

| Milestone | Description | Status | Verified By | Date Completed |
|---|---|---|---|---|
| **M0** | Scaffold & Layout — static `index.html`, no logic | PENDING | — | — |
| **M1** | Data Parsing & Seed Data Engine — SheetJS + Thai filters + `HS` matcher | PENDING | — | — |
| **M2** | Pure Math & Forecast Engine Isolation — `runRate`, `forecastRange`, `targetPace`, `mape`, `runChecks()` | PENDING | — | — |
| **M3** | Headline Metrics & Target Pace Banner UI | PENDING | — | — |
| **M4** | Interactive Decision Visualizations — trend/salesperson/customer charts | PENDING | — | — |
| **M5** | Recent Invoices Table & CSV Export | PENDING | — | — |
| **M6** | User Target Edit & UX Polish | PENDING | — | — |
| **M7** | Edge Case Hardening & Pre-Ship Checklist | PENDING | — | — |

*Update `Status` to `IN PROGRESS` when a milestone prompt is started, and
`COMPLETED` only after a teammate other than the Driver has run it and
agreed, per the workshop's Navigator/Reviewer rule.*

## 4. Verified Business Logic & Hand Calculations Log

Results of the `runChecks()` console test suite (see `schema.md` §5 for
the seed fixture and `implementation-plan.md` M2 for the verification
requirement). Fill in `Actual Result` and `Status` as each case is run.

| Case | Description | Expected Result | Actual Result | Status |
|---|---|---|---|---|
| **1** | Base Run Rate — 10 of 22 working days elapsed | `runRate` per `(salesToDate / 10) * 22` on test input | *pending run* | PENDING |
| **2** | Forecast Range — Low = −15%, Base, High = +15% | `{ low, base, high }` bounding `runRate` at ±15% | *pending run* | PENDING |
| **3** | Zero sales MTD handling — division-by-zero safety | `runRate` returns `0` (no crash, no `NaN`/`Infinity`) when `salesToDate = 0` or `elapsedWorkingDays = 0` | *pending run* | PENDING |
| **4** | Target Pace indicator assignment | Correct `'AHEAD'` / `'ON_TRACK'` / `'BEHIND'` returned for base-vs-target comparisons above, at, and below target | *pending run* | PENDING |
| **5** | MAPE error metric formula validation | `mape(forecastValues, actualValues)` matches hand-calculated `average(|forecast − actual| / actual)` | *pending run* | PENDING |

**Note:** No case may be marked `PASS` in this log without a corresponding
console log line from `runChecks()` and a hand calculation cross-check by
a teammate other than the one who wrote the function, per `AGENTS.md` §4
(Definition of Done).

## 5. Active Issues & Bug Tracker

| Issue ID | Severity | Description | Console Error / Behavior | Status | Fix Description |
|---|---|---|---|---|---|
| **BUG-001** | Med | Thai structural-row filter (`รายงานขายเงินสด`, `ตัดใบรับมัดจำ`, and related markers) is hardcoded exact-string matching against one specific report layout. Any change to the source export's wording, spacing, or row order will silently misclassify rows instead of raising an error. | No console error — rows are silently dropped or silently kept as if valid, producing incorrect totals with no warning. | OPEN | Not yet fixed. Documented as a known limitation per `architecture.md` §4 and `AGENTS.md` §3 (filters must not be altered without explicit instruction). Mitigation for now: spot-check `totalInvoices` and `totalSales` against the source file by eye after every upload until a more robust structural check is added. |

*Add new rows as issues are found during each milestone's verification
step. No issue is closed without a `Fix Description` and a re-verification
note.*

## 6. Key Architectural Trade-Offs & Decisions Log

| Decision | Chosen Approach | Trade-Off Accepted |
|---|---|---|
| **File architecture** | Single self-contained `index.html` (no build step, no multi-module framework) | Gains zero-setup deployment and matches the workshop's single-file MVP discipline; costs a larger, less modular file as features accumulate, and no code-splitting or lazy loading. |
| **Data storage** | Client-side, in-browser memory only — no database or backend server | Gains instant data privacy (the Thai sales export never leaves the browser) and zero hosting cost; costs all persistence — the dashboard resets on every reload and cannot compare sessions or store historical months without a re-upload each time. |

*Both decisions are locked for the MVP per `prd.md` Out of Scope and
`architecture.md` §1.2. Revisiting either requires updating `prd.md`,
`architecture.md`, and `schema.md` together, not `progress.md` alone.*

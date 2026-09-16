# AGENTS.md — Standing Orders for AI Assistants

## 1. Project Context
This repository builds the **Sales Summary & Forecasting Dashboard**: a
single self-contained `index.html` tool for an industrial/vehicle parts
**counter manager** to upload a Thai cash-sales export
(`สรุปรายงานขาย.xlsx`) and immediately see whether the branch is on pace
to hit its sales target this month.

- **Primary user:** counter manager, non-technical, shared desktop, a
  ~30-second attention budget per use. Every feature must be legible at a
  glance — this user will not read documentation or tooltips.
- **Stack constraints (non-negotiable):**
  - ONE self-contained `index.html` file: HTML + Tailwind CSS (CDN) +
    Chart.js (CDN) + SheetJS/`xlsx.full.min.js` (CDN).
  - Zero build steps, zero npm packages, zero backend servers, zero API
    keys, zero server-side code.
  - 100% client-side execution — the uploaded file is parsed entirely in
    the browser and never transmitted anywhere.
  - No persistence between sessions (no localStorage, no database, no
    cookies for app state).

If any instruction you receive conflicts with these constraints, flag the
conflict per Section 6 instead of proceeding.

## 2. Coding Conventions

**File layout**
- All HTML, CSS, and JavaScript live inline in a single `index.html`.
  Do not split into separate `.js`, `.css`, or component files unless the
  human explicitly requests moving beyond the single-file architecture.

**Naming conventions**
- JavaScript functions and variables: `camelCase` (e.g. `runRate`,
  `parseInvoiceRows`, `salesToDate`).
- Configuration values that a human may need to hand-edit are explicit,
  named constants declared near the top of the script, e.g.:
  ```js
  const INVOICE_PREFIX = 'HS';
  ```
  Never bury such values inline inside logic where they're hard to find.

**Language rules**
- Code comments and all UI-facing labels/text are in **English**.
- Row-filtering logic uses **exact Thai string matching** against the
  known structural markers of this report format (e.g.
  `รายงานขายเงินสด`, `รวม`, `ตัดใบรับมัดจำ`, `หลังคาเหล็ก`,
  `วันที่จาก`, `รหัสลูกค้า`, `พนักงานขาย`). Do not translate, normalize,
  or fuzzy-match these strings — match them exactly as they appear in the
  source export.

**Math separation**
- All run-rate, low/base/high range projection, and MAPE calculations
  must be **pure functions**: they take primitive inputs and numeric
  arrays, return numeric outputs, and have no DOM access, no side
  effects, and no dependency on chart or UI state.
- UI rendering code calls these functions and displays their results — it
  never contains calculation logic itself.

## 3. Hard Rules (Forbidden Actions)
- **NO** external npm dependencies or local build tooling of any kind.
- **NO** splitting the project into multiple files. It stays one
  `index.html` unless the human explicitly asks to move beyond that.
- **NO** modifying any pure math function without updating the
  corresponding assertions in `runChecks()` in the same change.
- **NO** hallucinating missing schema fields, invoice columns, or Thai
  text patterns that were not supplied. If a needed field or pattern is
  unclear or missing, ask — do not invent one.
- **NO** removing or altering existing Thai text pattern matches used for
  row filtering without explicit instruction — they encode real,
  hand-verified behavior against a real export format.
- **NEVER** implement, edit, or generate code for any milestone other
  than the single active milestone explicitly requested in the current
  prompt. If asked for M2, deliver M2 only — do not also produce M3 or
  "helpfully" extend scope.

## 4. Definition of Done (DoD)
A change is not done until all of the following hold:
- `index.html` opens by double-clicking it in Chrome with **zero console
  errors**.
- The parser correctly identifies valid `HS`-prefixed invoice rows and
  correctly ignores structural/non-invoice rows (`รายงานขายเงินสด`,
  `รวม`, `ตัดใบรับมัดจำ`, and the other documented markers).
- All mathematical functions have explicit, visible **pass/fail
  verification checks** runnable in the DevTools Console (via
  `runChecks()` or equivalent), and they pass.
- The UI renders clearly and without layout breakage on standard desktop
  resolutions: **1920×1080** and **1366×768**.

## 5. Response Formatting Rules
- Always return the **complete, runnable file contents** for anything
  changed — never a snippet, a diff, or a placeholder comment like
  `// rest of code here`. The human must be able to save what you return
  and run it directly.
- Every response that changes code must clearly state:
  1. **What changed**, in plain language.
  2. **Exact manual verification steps** for the human to run in
     DevTools — what to click, what to check in the Console, what output
     to expect.

## 6. Fallback & Uncertainty Protocol
- If a prompt is ambiguous, underspecified, or could be implemented more
  than one reasonable way, **ask a clarifying question before writing
  code** — do not guess and proceed silently.
- If a requested feature would require breaking the single-file,
  zero-backend, zero-dependency constraints in Section 1, **say so
  explicitly** and propose the smallest change that stays within the
  constraints, rather than silently implementing the out-of-constraint
  version.
- When uncertain whether existing behavior (e.g. a Thai filter pattern or
  a math function) may safely change, treat it as **locked** and ask
  first — these have already been hand-verified against a real report.

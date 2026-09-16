# schema.md — Sales Summary & Forecasting Dashboard

## 1. Overview

Data moves through four distinct shapes over the application's lifecycle,
each one derived from the last:

```
Raw Excel Row Arrays
      │  (SheetJS sheet_to_json, header:1)
      ▼
Filtered Invoice Objects
      │  (Thai string filter + HS-prefix match + normalization)
      ▼
State Summary Record  (DashboardState)
      │  (aggregation: sums, counts, group-by maps)
      ▼
Forecast Model Schema  (ForecastInput → ForecastOutput)
      │  (pure math functions)
      ▼
Chart.js datasets & DOM text content
```

Every stage is a plain JavaScript value (array/object) held in memory —
there is no database, no ORM, and no serialization format beyond what
SheetJS produces on read. This document is the contract every parsing
and calculation function must honor.

## 2. Source Excel Layout (Raw Input Mapping)

Raw rows are produced by
`XLSX.utils.sheet_to_json(sheet, { header: 1, defval: "" })`, i.e. each
row is a plain array indexed by column position, not by header name
(the source report has no single clean header row).

| Column Index | Raw Field Name | Description | Data Type | Sample Raw Value |
|---|---|---|---|---|
| 0 | `date_raw` | Invoice date as it appears in the export | String | `"01/09/2026"` |
| 1 | `inv_no` | Invoice number; presence of `HS` prefix marks a valid invoice row | String | `"HS2609-001"` |
| 2 | `cust_code` | Internal customer code | String | `"C-0104"` |
| 3 | `cust_name` | Customer display name (Thai) | String | `"บริษัท อะไหล่ยนต์ จำกัด"` |
| 4 | `salesperson` | Salesperson code, or `"N/A"` if unassigned | String | `"01"` |
| 7 | `grand_total` | Invoice total, comma-formatted in source | Number / Numeric String | `"12,500.00"` |

**Notes:**
- Columns 5–6 are not currently consumed by the application (reserved /
  unused in this report layout).
- Rows that do not have `inv_no` starting with `HS`, or that match a
  known Thai structural marker (see `architecture.md` §4), are excluded
  before reaching the Normalized Internal Models below.
- `grand_total` must have commas stripped and be coerced with
  `parseFloat` before use; a failed parse defaults to `0` (see
  `architecture.md` §6, Error Handling).

## 3. Normalized Internal Models (In-Memory JavaScript Schemas)

### 3.1 `Invoice` (Individual Transaction)

| Field | JS Type | Constraints | Notes |
|---|---|---|---|
| `date` | `string` | `NOT NULL` | Derived from `date_raw`, first whitespace-delimited token only |
| `invNo` | `string` | `NOT NULL`, must start with `'HS'` | Source of truth for row validity |
| `custCode` | `string` | `NOT NULL` (may be empty string) | From `cust_code` |
| `custName` | `string` | `NOT NULL` (may be empty string) | From `cust_name` |
| `salesperson` | `string` | `NOT NULL`, defaults to `'N/A'` if missing | From `salesperson` |
| `grandTotal` | `number` | `NOT NULL`, `>= 0`, defaults to `0` on parse failure | From `grand_total`, commas stripped |

**Worked example:**
```json
{
  "date": "01/09/2026",
  "invNo": "HS2609-001",
  "custCode": "C-0104",
  "custName": "บริษัท อะไหล่ยนต์ จำกัด",
  "salesperson": "01",
  "grandTotal": 12500.00
}
```

### 3.2 `DashboardState` (Global In-Memory State)

**Aggregate scalars:**

| Field | JS Type | Description |
|---|---|---|
| `totalSales` | `number` | Sum of `grandTotal` across all valid invoices |
| `totalInvoices` | `number` | Count of valid invoice rows |
| `uniqueCustomers` | `number` | Count of distinct `custCode` (fallback `custName`) values |
| `avgInvoiceValue` | `number` | `totalSales / totalInvoices`, `0` if `totalInvoices === 0` |

**Grouped collections:**

| Field | JS Type | Shape | Example |
|---|---|---|---|
| `dailySalesMap` | `object` | `{ "YYYY-MM-DD": number }` | `{ "2026-09-01": 45200.00 }` |
| `salesByPersonMap` | `object` | `{ "Sales Rep X": number }` | `{ "Sales Rep 01": 88300.00 }` |
| `salesByCustMap` | `object` | `{ "Customer Name": number }` | `{ "บริษัท อะไหล่ยนต์ จำกัด": 37500.00 }` |

`DashboardState` is fully derived from the current `Invoice[]` array on
every parse — it is never mutated independently and is safe to
regenerate from scratch on each file upload.

## 4. Forecast Engine Schema

### 4.1 `ForecastInput`

| Field | JS Type | Description |
|---|---|---|
| `salesToDate` | `number` | Total current-month revenue, equal to `DashboardState.totalSales` |
| `elapsedWorkingDays` | `number` | Working days passed so far this month |
| `totalWorkingDays` | `number` | Total working days in the current month; **default `24`** |
| `targetRevenue` | `number` | User-entered monthly target (see `prd.md` §4.4) |

**Worked example:**
```json
{
  "salesToDate": 350000.00,
  "elapsedWorkingDays": 10,
  "totalWorkingDays": 24,
  "targetRevenue": 900000.00
}
```

### 4.2 `ForecastOutput`

| Field | JS Type | Description |
|---|---|---|
| `runRate` | `number` | Base month-end projection: `(salesToDate / elapsedWorkingDays) * totalWorkingDays` |
| `range` | `object` | `{ low: number, base: number, high: number }` — bounded projection, never a single point estimate |
| `paceStatus` | `string` (enum) | `'AHEAD'` \| `'ON_TRACK'` \| `'BEHIND'` — comparison of `range.base` against `targetRevenue` |
| `targetVariance` | `number` | `range.base - targetRevenue`; positive = ahead, negative = behind |

**Worked example (matching the `ForecastInput` above):**
```json
{
  "runRate": 840000.00,
  "range": {
    "low": 756000.00,
    "base": 840000.00,
    "high": 924000.00
  },
  "paceStatus": "BEHIND",
  "targetVariance": -60000.00
}
```
*(range uses a ±10% variance band around `runRate` as the MVP default —
see `architecture.md` §5 note on `variancePercentage`.)*

## 5. Seed Data & Test Fixtures

Use this fixture to validate `runChecks()` in the DevTools Console
**before** uploading a live Excel file, per `AGENTS.md` §4 (Definition of
Done).

```json
{
  "invoices": [
    {
      "date": "01/09/2026",
      "invNo": "HS2609-001",
      "custCode": "C-0104",
      "custName": "บริษัท อะไหล่ยนต์ จำกัด",
      "salesperson": "01",
      "grandTotal": 12500.00
    },
    {
      "date": "02/09/2026",
      "invNo": "HS2609-002",
      "custCode": "C-0087",
      "custName": "ร้านช่างยนต์รุ่งเรือง",
      "salesperson": "02",
      "grandTotal": 8750.50
    },
    {
      "date": "02/09/2026",
      "invNo": "HS2609-003",
      "custCode": "C-0104",
      "custName": "บริษัท อะไหล่ยนต์ จำกัด",
      "salesperson": "01",
      "grandTotal": 21300.00
    }
  ],
  "forecastFixture": {
    "input": {
      "salesToDate": 350000.00,
      "elapsedWorkingDays": 10,
      "totalWorkingDays": 24,
      "targetRevenue": 900000.00
    },
    "expectedOutput": {
      "runRate": 840000.00,
      "range": {
        "low": 756000.00,
        "base": 840000.00,
        "high": 924000.00
      },
      "paceStatus": "BEHIND",
      "targetVariance": -60000.00
    }
  }
}
```

**How this fixture is used by `runChecks()`:**
- The three `invoices` entries should be fed through the aggregation
  logic to confirm `DashboardState` values are hand-computable
  (`totalSales = 42550.50`, `totalInvoices = 3`, `uniqueCustomers = 2`).
- The `forecastFixture` pair is the canonical assertion case for
  `calculateRunRate`, `calculateForecastRange`, and `determineTargetPace`
  — any change to those pure functions must still reproduce this exact
  `expectedOutput`, or `runChecks()` must be updated in the same commit
  per `AGENTS.md` Hard Rules.

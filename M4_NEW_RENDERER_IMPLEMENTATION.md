# M4: New Report Renderer — Implementation Details

> **Created:** 2026-02-10
> **Status:** Partially Complete (Sales Trend sparkline still blank)

---

## Summary

Replaced the broken `ReportRendererV3` + `VisualRenderer` pipeline with a new architecture that works **directly with the backend's columnar format** — no lossy data transforms. The new system consists of 3 components:

1. **`ReportRenderer.tsx`** — Main orchestrator (filter bar, data fetching, grid layout)
2. **`ReportVisual.tsx`** — Visual dispatcher (transforms columnar data → UI primitives)
3. **`ReportFilterBar.tsx`** — Filter bar (date range + "More Filters" popover + chips)

---

## Why the Old System Was Broken

The old `ReportRendererV3` + `VisualRenderer` pipeline had 3 critical flaws:

1. **Lossy data transforms in `page.tsx`**: Converted backend columnar data (`{columns, rows, totals}`) into `VisualDataResponse` object arrays, **losing** `dimensionLabels`, `trend`, `insights`, and `pagination` data
2. **`VisualRenderer` (970 lines)** expected the old lossy format and couldn't render KPI distribution, KPI sparkline, KPI insights, table sparklines, or matrix/pivot
3. **`queryApi.ts` `VisualResult` type** was missing `dimensionLabels`, `trend`, and `insights` fields

---

## Architecture

### Data Flow (New — No Lossy Transforms)

```
page.tsx
  → executeReport({ templateKey, clientId, filters })
  → backend returns { template, visuals: Record<string, BackendVisualResult> }
  → pass raw data directly to <ReportRenderer>
    → <ReportFilterBar> for date/dimension filters
    → <ReportVisual> per visual with raw BackendVisualResult
      → dispatches to correct UI primitive (KPI card, chart, table, matrix)
```

### Component Responsibilities

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| `ReportRenderer` | `components/reports/ReportRenderer.tsx` | ~340 | Orchestrator: filter bar, data fetching, section grid layout, visual rendering |
| `ReportVisual` | `components/reports/ReportVisual.tsx` | ~780 | Visual dispatcher: transforms columnar data → KPI cards, charts, tables, matrices |
| `ReportFilterBar` | `components/reports/ReportFilterBar.tsx` | ~167 | Filter bar: DateRangePicker + MoreFiltersPopover + FilterChips + Refresh |
| `page.tsx` | `app/(user-type)/(user)/reports/[templateKey]/page.tsx` | ~112 | Route page: initial fetch, loading/error states, passes raw data to ReportRenderer |

---

## Files Created / Modified

### New Files

| File | Purpose |
|------|---------|
| `components/reports/ReportRenderer.tsx` | Main report orchestrator |
| `components/reports/ReportVisual.tsx` | Visual type dispatcher |
| `components/reports/ReportFilterBar.tsx` | Filter bar component |

### Modified Files

| File | Changes |
|------|---------|
| `types/report-template.ts` | Added `BackendVisualResult`, `BackendColumnMeta`, `BackendPaginationInfo`, `DevReportResponse` types; fixed `TableVisualConfig.defaultGroupBy` to `string \| string[]`; added `sparklineMetrics` to `KPIVisualConfig` |
| `store/api/queryApi.ts` | Added `dimensionLabels`, `trend`, `insights` to `VisualResult` interface; added `groupByOverrides` param to `executeDevReport` endpoint; added `convertDateRangePreset()` helper |
| `app/(user-type)/(user)/reports/[templateKey]/page.tsx` | Simplified — removed all `transformVisualResult`/`transformToReportData` functions; uses dynamic clientId from Redux `state.client` |
| `beastinsights-backend/routes/devRoutes.js` | Added `resolveDatePreset()` + `resolveFilters()` for backend date preset resolution; added `groupByOverrides` parameter support |
| `beastinsights-backend/services/queryEngineV2.js` | Fixed `buildWhereClause` to handle `'eq'` operator (was silently skipping single-value dimension filters) |

---

## Key Bug Fixes

### 1. Date Range Filters Not Working

**Root Cause:** Frontend sends `{ preset: 'last_30_days' }` but backend's `buildWhereClause` expects `{ from: 'YYYY-MM-DD', to: 'YYYY-MM-DD' }`.

**Fix (two layers):**
- **Frontend** (`queryApi.ts`): `convertDateRangePreset()` resolves preset → `{from, to}` before sending to backend
- **Backend** (`devRoutes.js`): Added `resolveDatePreset()` + `resolveFilters()` as safety net — resolves presets, normalizes `{start, end}` → `{from, to}`

**Verification:**
```
last_7_days  → 0 approvals (no data in range)
last_30_days → 26 approvals
last_90_days → 167 approvals
```

### 2. Dimension Filters Not Working (Single-Value)

**Root Cause:** Frontend sent `operator: 'eq'` for single-value filters (e.g., selecting one campaign). Backend `buildWhereClause` only handled `'in'`, `'not_in'`, `'between'` — the `'eq'` operator was silently skipped.

**Fix:**
- **Backend** (`queryEngineV2.js` line 753): Changed `config.operator === 'in'` to `(config.operator === 'in' || config.operator === 'eq')`
- **Frontend** (`ReportRenderer.tsx`): Changed to always use `'in'` operator regardless of value count

**Verification:**
```
No filter     → 26 approvals
campaign_id=61 → 8 approvals
campaign_id=15 → 0 approvals (no data)
```

### 3. GroupBy Not Sent to Backend

**Root Cause:** Frontend had a group-by selector UI but never sent the selected dimensions to the backend for query generation.

**Fix:**
- **Backend** (`devRoutes.js`): Route handler accepts `groupByOverrides` from `req.body`
- **Backend** (`devRoutes.js`): `buildVisualQueries()` applies override groupBy for table visuals
- **Frontend** (`queryApi.ts`): Added `groupByOverrides` to request body
- **Frontend** (`ReportRenderer.tsx`): Builds `gbOverrides` map from per-table `groupByState` and sends with each request

**Verification:**
```
Default (campaign_name) → 5 rows with campaign columns
Override (product_name) → 6 rows with product columns
```

### 4. GroupBy Component Replaced

**Change:** Replaced generic `TremorSelect` with the project's existing `GroupBySelector` component, which supports:
- Multi-select with hierarchy
- Drag-and-drop reordering
- Max 3 selections
- Search when >5 options

### 5. Dynamic Client ID

**Change:** `page.tsx` now reads `clientId` from Redux `state.client.clientId` (set by sidebar client selector) instead of hardcoding `10009`. Falls back to `10009` if no client is selected.

---

## Known Issues — Sales Trend Sparkline STILL BLANK

### Symptom

The "Sales Trend" KPI card shows only the title "Sales Trend" in a blank card. The hero value (e.g., "26") and the sparkline chart do not render.

### What's Been Verified

1. **Backend data is correct** — curl returns:
   - `rows: [[26, 0, 26, 0]]` (aggregate row with approvals, initials, rebills, straight_sales)
   - `trend: { columns: [{key:'date',...}, {key:'initials',...}, {key:'rebills',...}, {key:'straight_sales',...}], rows: [18 rows] }`
   - Trend rows like: `['2026-01-10T18:30:00.000Z', 0, 0, 0]`

2. **Type definitions are correct** — `BackendVisualResult.trend` is typed as `{ columns, rows, meta? }`

3. **Template config is correct** — `sales_trend_card` has `variant: 'sparkline'`, `metric.key: 'approvals'`, `sparklineMetrics: [{key:'initials',color:'blue'}, {key:'rebills',color:'emerald'}, {key:'straight_sales',color:'amber'}]`

4. **Data flow is correct** — `transformResponse` in queryApi.ts preserves the `trend` object on each visual result

### Debugging Attempts

| Attempt | Approach | Result |
|---------|----------|--------|
| 1 | Wrapped `SparkAreaChart` in div with explicit `style={{ height: 80, width: '100%' }}` | Still blank |
| 2 | Replaced SparkAreaChart with direct Recharts `AreaChart` + `Area` components, hardcoded width=320, explicit hex colors | Still blank |
| 3 | Added `ResponsiveContainer` wrapping direct Recharts AreaChart, parent div with `style={{ width: '100%', height: 80 }}` | Still blank |
| 4 | Switched back to project's `SparkAreaChart` component with `className="h-20 w-full"` override | Still blank |

### What's Puzzling

- The hero value "26" should render independently of the chart (it's a separate `<p>` element above the chart div)
- Even the hero value doesn't show — only the `<h3>` title "Sales Trend" is visible
- This suggests a possible **React rendering error** that crashes the component after the title `<h3>` but before the value `<p>` — but no error appears in server logs
- Could be a client-side hydration issue or an error in the `formatValue()` function that only manifests at runtime

### Suspected Root Causes

1. **Client-side rendering error** — Something after the `<h3>` throws an error that React catches silently. Need browser DevTools console to see the exact error.
2. **Recharts SSR incompatibility** — `ResponsiveContainer` has known issues with Next.js SSR. Even though `'use client'` is set, the initial server render might fail.
3. **CSS overflow clipping** — The Card or parent grid cell might have overflow:hidden that clips content below the title. (Checked Card.tsx — no overflow:hidden found)

### Next Steps to Debug

1. **Open browser DevTools console** — look for red error messages when the page loads
2. **Add `console.log` inside `renderKPISparkline`** — verify the function is called and what data it receives
3. **Try rendering without any chart** — just title + hero value + legend (no Recharts) to isolate if the chart import is causing the crash
4. **Try a completely static test** — hardcode `heroValue = 42` and remove all chart code to verify the Card renders properly

---

## ReportRenderer — Technical Details

### Filter Change → Re-fetch Flow

Two-effect pattern to avoid stale closures and race conditions:

```
Effect 1: Filter change detection
  - Watches: dateRange, filters, groupByState (from Redux + local state)
  - JSON-serializes all three → compares to previous key
  - If changed: bumps fetchTrigger counter
  - On mount with initialVisuals: skips first change (data already loaded)

Effect 2: Fetch execution
  - Watches: fetchTrigger
  - Builds dimension filters from Redux state
  - Builds groupByOverrides from per-table groupByState
  - Calls executeReport via RTK Query
  - Sets loading state, handles success/error, cleans up on unmount
```

### Section Grid Layout

Uses 12-column CSS grid per section (same as ReportRendererV3):

```tsx
<div className="grid grid-cols-12 gap-4">
  {section.visuals.map((visual) => (
    <div key={visual.id} className={getGridStyle(visual.position)}>
      <ReportVisual ... />
    </div>
  ))}
</div>
```

Grid positions from template JSON: `{ col, colSpan, responsive: { sm, md, lg } }`

---

## ReportVisual — Visual Type Support

| Visual Type | Variant | Status | Existing Component Used |
|-------------|---------|--------|------------------------|
| KPI | distribution | ✅ Working | `KPICardDistribution` |
| KPI | sparkline | ❌ BROKEN | `SparkAreaChart` (renders blank) |
| KPI | insights | ✅ Working | Custom Card with bullet list |
| Chart | area | ✅ Working | `InteractiveAreaChart` |
| Chart | bar | ✅ Working | `InteractiveBarChart` |
| Chart | line | ✅ Working | `InteractiveLineChart` |
| Chart | donut | ✅ Working | `DonutChart` |
| Table | with group-by | ✅ Working | `DataTable` + `GroupBySelector` |
| Table | with sparkline cols | ✅ Working | `DataTable` with sparkline cells |
| Table | with totals | ✅ Working | `DataTable` with footerRows |
| Table | with pagination | ✅ Working | `DataTable` with pagination |
| Matrix/Pivot | — | ✅ Working | Custom HTML table |

---

## ReportFilterBar — Layout

```
┌──────────────────────────────────────────────────────────────┐
│  [📅 Date Range Picker]     [More Filters ▾]    [🔄 Refresh] │
│  [chip: Campaign A ✕] [chip: Gateway X ✕] [Clear All]        │
└──────────────────────────────────────────────────────────────┘
```

- **DateRangePicker**: Existing component with Redux integration via `slicerSlice`
- **MoreFiltersPopover**: 2-column layout with sections per dimension, multi-select checkboxes, search
- **FilterChips**: Removable badges showing active filters
- **Refresh button**: Bumps `fetchTrigger` counter to force re-fetch

---

## Backend Changes

### Date Preset Resolution (`devRoutes.js`)

Added `resolveDatePreset(preset)` function that converts preset strings to `{from, to}` date objects:

| Preset | Resolves To |
|--------|-------------|
| `today` | `{from: today, to: today}` |
| `yesterday` | `{from: yesterday, to: yesterday}` |
| `last_7_days` | `{from: today-7, to: today}` |
| `last_14_days` | `{from: today-14, to: today}` |
| `last_30_days` | `{from: today-30, to: today}` |
| `last_60_days` | `{from: today-60, to: today}` |
| `last_90_days` | `{from: today-90, to: today}` |
| `this_month` | `{from: 1st of month, to: today}` |
| `last_month` | `{from: 1st of prev month, to: last of prev month}` |
| `this_quarter` | `{from: 1st of quarter, to: today}` |
| `this_year` | `{from: Jan 1, to: today}` |
| `last_year` | `{from: Jan 1 prev year, to: Dec 31 prev year}` |

Also added `resolveFilters(filters)` wrapper that:
- Resolves `{preset}` → `{from, to}` via `resolveDatePreset()`
- Normalizes `{start, end}` → `{from, to}` (custom date range format)
- Defaults to `last_30_days` if no date range provided

### GroupBy Overrides (`devRoutes.js`)

`buildVisualQueries(visual, template, granularity, groupByOverrides)` now accepts `groupByOverrides` parameter. For table visuals, if `groupByOverrides[visual.id]` exists, it replaces the template's `defaultGroupBy`.

### Dimension Filter Fix (`queryEngineV2.js`)

`buildWhereClause` line 753: Added `'eq'` to the operator check:

```js
// Before (broken):
if (config.operator === 'in' && config.values?.length > 0) {

// After (fixed):
if ((config.operator === 'in' || config.operator === 'eq') && config.values?.length > 0) {
```

---

## Existing Components Reused (No Modifications)

| Component | Path | Used For |
|-----------|------|----------|
| `KPICardDistribution` | `components/ui-tremor/kpi-cards/kpi-card-distribution.tsx` | Sales breakdown card |
| `SparkAreaChart` | `components/ui-tremor/charts/SparkChart.tsx` | Sales trend sparkline (currently broken) |
| `InteractiveAreaChart` | `components/ui-tremor/charts/InteractiveChart.tsx` | Area/bar/line charts |
| `DonutChart` | `components/ui-tremor/charts/DonutChart.tsx` | Donut charts |
| `DataTable` | `components/ui-tremor/tables/data-table.tsx` | All tables |
| `GroupBySelector` | `components/filters/GroupBySelector.tsx` | Table group-by dropdown |
| `DateRangePicker` | `components/filters/DateRangePicker.tsx` | Date range filter |
| `MoreFiltersPopover` | `components/filters/MoreFiltersPopover.tsx` | Advanced filters |
| `FilterChips` | `components/filters/FilterChips.tsx` | Active filter badges |
| `Card` | `components/ui-tremor/kpi-cards/Card.tsx` | KPI card wrapper |
| `slicerSlice` | `store/slices/slicerSlice.ts` | Redux filter state |
| `clientSlice` | `store/slices/clientSlice.js` | Redux client state |

---

## Test Commands

### Backend

```bash
# Start backend
cd beastinsights-backend && node app.js

# Test date preset resolution
curl -s http://localhost:8080/api/dev/report/sales-report -X POST \
  -H 'Content-Type: application/json' \
  -d '{"clientId": 10009, "filters": {"dateRange": {"preset": "last_30_days"}}}'

# Test dimension filters
curl -s http://localhost:8080/api/dev/report/sales-report -X POST \
  -H 'Content-Type: application/json' \
  -d '{"clientId": 10009, "filters": {"dateRange": {"preset": "last_30_days"}, "dimensions": {"campaign_id": {"operator": "in", "values": [61]}}}}'

# Test groupBy overrides
curl -s http://localhost:8080/api/dev/report/sales-report -X POST \
  -H 'Content-Type: application/json' \
  -d '{"clientId": 10009, "filters": {"dateRange": {"preset": "last_30_days"}}, "groupByOverrides": {"sales_table": ["product_name"]}}'
```

### Frontend

```bash
# Start frontend
cd beastinsights && npm run dev

# Navigate to
https://localhost:3000/reports/sales-report
```

### Verify Filters Working

1. Change date range → KPI values should change
2. Open "More Filters" → select a campaign → KPI values should filter down
3. Change table group-by → table columns/rows should change
4. Click Refresh → data should re-fetch

---

*Last updated: 2026-02-10*

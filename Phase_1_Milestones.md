# Phase 1 — Milestones & Deliverables

> Build the complete native reporting engine that converts JSON templates into interactive reports with dynamic filters, all visual types, metric slicers, and sub-second load times.

---

## Milestone Sequence

```
M1  Database Architecture
     ↓
M2  JSON Template Format Specification
     ↓                                         ┐
M3  Backend Query Engine                       │ can run in parallel
M4  Frontend Report Renderer                   │
M5  Data Library from Power BI                 ┘
     ↓
M6  Reports, Filters & Navigation
     ↓
M7  End-to-End Testing & Rollout
```

**Why this order:**
- M1 first — everything needs the database tables to exist
- M2 second — the JSON format is the contract that both backend (M3) and frontend (M4) build against. Define it once, build both sides in parallel.
- M3, M4, M5 can run in parallel — backend team builds the query engine, frontend team builds the renderer, analyst/data team extracts the PBI data library. All three depend on M1 and M2 but not on each other.
- M6 brings everything together — creates the actual 11 reports using the engine (M3), the renderer (M4), and the real data (M5)
- M7 validates, optimizes, and prepares for client rollout

---

## Milestone 1: Database Architecture

### What This Is

Create the `report_builder` schema in PostgreSQL with all the tables needed for the entire semantic layer. These tables are the containers that hold the metric catalog, dimension catalog, toggle system, report templates, and all future Phase 2-4 features (bookmarks, filter presets, dashboards, scheduled reports, user dimension access, client modules).

We create ALL tables upfront — even those used in later phases — because altering schema mid-flight is disruptive. Tables for Phase 2+ simply remain empty until needed.

### Deliverables

**Core Semantic Layer Tables (Phase 1 — will be populated)**

| Table | Purpose | Used By |
|-------|---------|---------|
| `data_sources` | Maps logical names (e.g., `order_summary`) to actual materialized view patterns (e.g., `reporting.order_summary_{client_id}`). Stores the date column name, refresh frequency, and whether the source is active. | Query Engine (M3) |
| `metrics` | The full metric catalog. Every metric the platform can compute — its unique key, display name, SQL expression, data type, format pattern, category, whether it's derived, its derivation formula, aggregation type. ~100 rows when seeded. | Query Engine (M3), Data Library (M5) |
| `dimensions` | The full dimension catalog. Every dimension users can group/filter by — its unique key, display name, SQL column expression, which table it joins to, the join condition. ~30 rows when seeded. | Query Engine (M3), Filter System (M6) |
| `data_source_metrics` | Junction table: which metrics are valid for which datasets. Prevents invalid queries (e.g., can't ask for `ltv_30d` from `hourly_revenue`). | Query Engine (M3) |
| `data_source_dimensions` | Junction table: which dimensions can be used with which datasets. | Query Engine (M3) |
| `metric_toggles` | Toggle definitions: the 6 context switches (approval_mode, date_basis, time_granularity, profitability_view, retention_base, display_mode). Stores toggle key, display name, and toggle type. | Query Engine (M3), Slicer Panel (M6) |
| `metric_toggle_options` | Options per toggle (e.g., approval_mode has: standard, organic, net). Stores which option is the default. | Query Engine (M3), Slicer Panel (M6) |
| `metric_variants` | The SQL override per metric/toggle/option combination. When `approval_mode = organic`, the `approvals` metric uses `SUM(approvals_organic)` instead of `SUM(approvals)`. ~20 variant rows. | Query Engine (M3) |
| `report_templates` | Stores the JSON layout for each report. Template key, name, category, the full JSONB layout blob, default filters, default date range, required datasets, navigation section and order. | Template API (M6) |

**Future Phase Tables (created now, populated later)**

| Table | Phase | Purpose |
|-------|-------|---------|
| `bookmarks` | Phase 2 | Per-user customizations of a stock report (filters, layout, metrics) |
| `filter_presets` | Phase 2 | Named, shareable filter combinations |
| `dashboards` | Phase 2 | Personal dashboard layouts |
| `dashboard_tiles` | Phase 2 | Pinned tiles on personal dashboards |
| `scheduled_reports` | Phase 2 | Scheduled email/Telegram delivery configs |
| `user_dimension_access` | Phase 3 | Maps users to allowed dimension values for row-level security |
| `client_modules` | Phase 4 | Per-client industry module enablement |

**Database creation approach:**
- All new tables are defined in raw SQL files and executed directly against PostgreSQL
- After tables are created, run `prisma db pull` to introspect the database and sync the Prisma schema automatically
- Uses the `multiSchema` preview feature (already enabled) so the new `report_builder` schema coexists with existing schemas
- No changes to any existing tables or models

### What You Can Verify

- Connect to the database, query `report_builder.data_sources` — table exists and is empty (ready to be seeded)
- Query `report_builder.metrics` — table exists with all columns
- Check all foreign keys and indexes are created
- Prisma schema reflects the new tables after `prisma db pull`
- Existing application works exactly as before — zero impact on current functionality

### What This Does NOT Include

- No data in the tables yet (that's M5)
- No API endpoints (that's M3)
- No frontend code (that's M4)

---

## Milestone 2: JSON Template Format Specification

### What This Is

This is a **design milestone**, not a code milestone. It produces a specification document that defines the exact JSON structure used to describe any report. This format is the contract between the backend (which serves templates and resolves queries) and the frontend (which renders templates into interactive UI).

Every possible visual type, chart type, filter type, slicer type, layout type, and configuration option must be defined here. Once this spec is locked, backend and frontend teams can build in parallel with confidence that everything will connect.

### Deliverables

**1. Template Root Structure**

Every report template JSON has these top-level keys:

| Key | Type | What It Controls |
|-----|------|-----------------|
| `version` | string | Schema version for future compatibility |
| `source` | string | Primary dataset key (e.g., `"order_summary"`) |
| `sourceConfig` | object | Source-specific behavior (server-side pagination, search columns, date filter type) |
| `defaults` | object | Default date range (`"last_7_days"`, `"last_30_days"`, `"today"`, etc.), auto-refresh interval in seconds |
| `slicerPanel` | object | Complete slicer panel configuration |
| `sections` | array | The ordered list of visual sections that make up the report |

**2. Slicer Panel Configuration**

The `slicerPanel` object controls everything in the filter/slicer area at the top of the report:

| Key | Type | What It Controls |
|-----|------|-----------------|
| `showDateRange` | boolean | Whether to show the date range picker (false for Real-Time Pulse which is always "today") |
| `showSearch` | boolean | Whether to show a text search box (true for Transaction Explorer) |
| `searchPlaceholder` | string | Placeholder text for search |
| `showGroupBy` | boolean | Whether to show the group-by dropdown |
| `groupByOptions` | string[] | Available dimension keys for group-by (e.g., `["campaign_name", "product_name", "gateway_alias"]`) |
| `defaultGroupBy` | string | Which dimension is selected by default |
| `slicers` | array | List of slicer controls to show |
| `filters` | array | List of dimension filter controls to show |
| `parameters` | array | List of parameter input controls (for profitability) |
| `statusFilters` | array | Quick-filter pills with pre-built filter conditions (for Transaction Explorer: All/Approved/Declined/Refunded/Chargeback) |
| `showExport` | boolean | Whether to show the export button |

**Slicer definition within slicerPanel:**

| Key | What It Means |
|-----|--------------|
| `toggleKey` | References a toggle in `metric_toggles` table (e.g., `"approval_mode"`) |
| `position` | `"main"` (always visible) or `"more"` (in a dropdown) |
| `displayAs` | `"pills"` (horizontal buttons) or `"dropdown"` |

**Filter definition within slicerPanel:**

| Key | What It Means |
|-----|--------------|
| `dimensionKey` | References a dimension in `dimensions` table (e.g., `"campaign_name"`) |
| `displayAs` | `"pills"` (few options), `"multiselect"` (many options), `"dropdown"` (single select) |
| `position` | `"main"` or `"more"` |

**Parameter definition within slicerPanel:**

| Key | What It Means |
|-----|--------------|
| `key` | Parameter name passed to query engine (e.g., `"processing_fee_pct"`) |
| `label` | Display label (e.g., `"Processing Fees"`) |
| `type` | `"number"` |
| `prefix` / `suffix` | `"$"` or `"%"` — shown before/after the input |
| `default` | Default value (e.g., `3.5`) |
| `min` / `max` / `step` | Input constraints |

**3. Section Definition**

Each section in the `sections` array represents a visual block of the report:

| Key | Type | What It Controls |
|-----|------|-----------------|
| `id` | string | Unique section identifier |
| `type` | string | Layout type (see below) |
| `title` | string | Optional section heading |
| `columns` | number | Number of grid columns (for `grid` type) |
| `conditionalVisibility` | object | Show section only when a specific toggle value is active |
| `visuals` | array | The visuals inside this section |

**Layout types:**

| Type | Behavior |
|------|----------|
| `card_row` | Horizontal row of equal-width cards (2-4 per row). Wraps to 2 columns on tablet, 1 on mobile. |
| `full_width` | Single column, full content width. Used for charts, tables. |
| `grid` | CSS grid with configurable column count. For side-by-side charts. |
| `tabs` | Tabbed interface. Each visual becomes a tab. Lazy-loads on selection. |

**Conditional visibility** — controls which sections appear based on toggle state:
```
"conditionalVisibility": {
  "toggleKey": "profitability_view",
  "showWhen": "cashflow"
}
```
This means the section only renders when the `profitability_view` toggle is set to `cashflow`.

**4. Visual Types — Complete Specification**

**a) Card**

Shows 1-3 formatted metrics with trend indicators.

| Config Key | What It Controls |
|-----------|-----------------|
| `metrics` | Array of `{ metricKey, label }` — which metrics to display (1-3) |
| `showTrend` | Show trend arrow and comparison % |
| `trendComparison` | `"previous_period"` — what to compare against |
| `invertTrend` | When `true`, a decrease is shown as green (good) instead of red. Used for cancel rate, CB rate, churn. |

Display formatting comes from the metric catalog — currency shows `$45,678`, percentage shows `72.3%`, integer shows `1,234`.

**b) Chart**

Tremor-based visualization supporting multiple chart types.

| Config Key | What It Controls |
|-----------|-----------------|
| `chartType` | `"line"`, `"bar"`, `"area"`, `"stacked_bar"`, `"pie"`, `"donut"` |
| `xAxis` | `{ dimensionKey }` — what goes on the X axis. Supports `"{time_granularity}"` placeholder that resolves to current toggle value. |
| `series` | Array of `{ metricKey, label, color, style }` — each line/bar series. `style: "dashed"` for comparison lines. |
| `dimension` | For stacked charts — the dimension that creates the stack segments (e.g., product_name) |
| `metric` | For stacked charts — which metric to show per segment |
| `stacked` | Whether area/bar series stack on top of each other |
| `showLegend` | Show chart legend |

**c) Table**

TanStack Table-based interactive table.

| Config Key | What It Controls |
|-----------|-----------------|
| `dimension` | What the rows are grouped by. Supports `"{groupBy}"` placeholder for user-selectable grouping. |
| `metrics` | Array of `{ metricKey }` — table columns |
| `sortBy` | Default sort column |
| `sortDirection` | `"asc"` or `"desc"` |
| `pagination` | `{ pageSize }` — rows per page |
| `showTotals` | Show a totals row at the bottom |
| `allowColumnToggle` | Let users show/hide columns |
| `exportable` | Show export button for this table |
| `isDetailTable` | `true` for Transaction Explorer — means it uses `columns` config instead of `dimension`+`metrics` |
| `columns` | For detail tables: array of `{ key, label, format }` defining exact columns |

**d) Matrix**

Cohort/retention matrix with heatmap coloring.

| Config Key | What It Controls |
|-----------|-----------------|
| `rowDimension` | What goes on the rows (e.g., `"date_month"` for cohort months) |
| `columnDimension` | What goes on the columns (e.g., `"billing_cycle"` for cycles 1-12) |
| `valueMetric` | The metric to display in cells (e.g., `"approvals"`) |
| `baseMetric` | The denominator for percentage display (e.g., `"initials"`) |
| `displayAs` | `"percentage"` or `"absolute"` |
| `maxColumns` | Maximum number of columns to show (e.g., 12 cycles) |
| `heatmapColors` | `{ low, mid, high }` — color gradient for cells |

**e) Waterfall**

P&L visualization showing revenue minus costs equals profit.

| Config Key | What It Controls |
|-----------|-----------------|
| `steps` | Ordered array of `{ metricKey, label, type }` where type is `"start"` (revenue), `"subtract"` (cost), or `"end"` (profit) |

**f) ParameterInput**

Interactive number input bound to profitability calculations.

| Config Key | What It Controls |
|-----------|-----------------|
| `key` | Parameter name (e.g., `"processing_fee_pct"`) |
| `label` | Display label |
| `prefix` / `suffix` | `"$"` or `"%"` |
| `default`, `min`, `max`, `step` | Input constraints |

**g) HealthIndicator**

Status badge based on configurable thresholds.

| Config Key | What It Controls |
|-----------|-----------------|
| `metric` | Metric key to evaluate |
| `thresholds` | `{ good: ">= 0.95", warning: ">= 0.90", critical: "< 0.90" }` |

**5. Special Visual Behaviors**

| Behavior | How It's Expressed in JSON |
|----------|--------------------------|
| **Auto-refresh** | `defaults.refreshInterval: 300` (seconds) — TanStack Query refetches on interval |
| **Server-side pagination** | `sourceConfig.serverSidePagination: true` — backend handles page/offset |
| **Server-side search** | `sourceConfig.serverSideSearch: true` + `sourceConfig.searchColumns: [...]` |
| **Cross-source visuals** | Individual visual has `source: "decline_recovery"` overriding the template's default source |
| **Dynamic titles** | `title: "Revenue by {groupBy}"` — placeholder replaced with current group-by selection |
| **Dynamic X axis** | `xAxis.dimensionKey: "{time_granularity}"` — resolves to `date_day`, `date_week`, or `date_month` based on toggle |
| **Month selector** | `sourceConfig.dateFilterType: "month_selector"` — shows month dropdown instead of date range |
| **Compare to previous** | `showTrend: true` + `trendComparison: "previous_period"` on Card |
| **Inverted trend** | `invertTrend: true` — lower value = green (for cancel rate, CB rate, churn) |

**6. Complete Example Template**

A minimal but complete Revenue Analytics template showing all concepts:

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": { "dateRange": "last_30_days" },
  "slicerPanel": {
    "showDateRange": true,
    "showGroupBy": true,
    "groupByOptions": ["campaign_name", "product_name", "gateway_alias"],
    "defaultGroupBy": "campaign_name",
    "slicers": [
      { "toggleKey": "approval_mode", "position": "main", "displayAs": "pills" },
      { "toggleKey": "time_granularity", "position": "main", "displayAs": "pills" }
    ],
    "filters": [
      { "dimensionKey": "sales_type", "displayAs": "pills", "position": "main" },
      { "dimensionKey": "campaign_name", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "more" }
    ],
    "showExport": true
  },
  "sections": [
    {
      "id": "summary",
      "type": "card_row",
      "visuals": [
        {
          "id": "initials_card",
          "type": "card",
          "title": "Initials",
          "metrics": [
            { "metricKey": "initials", "label": "Count" },
            { "metricKey": "revenue_initials", "label": "Revenue" }
          ],
          "showTrend": true,
          "trendComparison": "previous_period"
        },
        {
          "id": "rebills_card",
          "type": "card",
          "title": "Rebills",
          "metrics": [
            { "metricKey": "rebills", "label": "Count" },
            { "metricKey": "revenue_rebills", "label": "Revenue" }
          ],
          "showTrend": true,
          "trendComparison": "previous_period"
        }
      ]
    },
    {
      "id": "trend",
      "type": "full_width",
      "visuals": [
        {
          "id": "revenue_trend",
          "type": "chart",
          "chartType": "area",
          "title": "Revenue Trend",
          "xAxis": { "dimensionKey": "{time_granularity}" },
          "series": [
            { "metricKey": "revenue_initials", "label": "Initials", "color": "#4F46E5" },
            { "metricKey": "revenue_rebills", "label": "Rebills", "color": "#10B981" }
          ],
          "stacked": true,
          "showLegend": true
        }
      ]
    },
    {
      "id": "breakdown",
      "type": "full_width",
      "visuals": [
        {
          "id": "revenue_table",
          "type": "table",
          "title": "Revenue by {groupBy}",
          "dimension": "{groupBy}",
          "metrics": [
            { "metricKey": "attempts" },
            { "metricKey": "approvals" },
            { "metricKey": "approval_rate" },
            { "metricKey": "revenue" },
            { "metricKey": "cancel_rate" },
            { "metricKey": "cb_rate" }
          ],
          "sortBy": "revenue",
          "sortDirection": "desc",
          "pagination": { "pageSize": 25 },
          "showTotals": true,
          "allowColumnToggle": true,
          "exportable": true
        }
      ]
    }
  ]
}
```

### What You Can Verify

- The spec covers every visual type needed across all 11 reports
- The spec covers every filter, slicer, and parameter type
- A developer can read this spec and build backend or frontend without ambiguity
- The example template is valid against the spec
- Every edge case is addressed: dynamic titles, cross-source visuals, conditional visibility, server-side pagination, auto-refresh, month selector, compare to previous period

### What This Does NOT Include

- Not the actual 11 report templates (that's M6)
- Not any code (this is a specification/design document)

---

## Milestone 3: Backend Query Engine

### What This Is

The generic API that takes any combination of metrics, dimensions, filters, and toggles — and returns formatted data. One endpoint handles every report, every visual, every filter combination. No report-specific backend code.

**Repo:** `beastinsights-backend` (new repo). Brings over login/auth routes and necessary existing functionality from `beast-insights-backend`. This is a single backend repo that replaces the existing backend for the native reporting subdomain.

### Deliverables

**1. Single Query Endpoint — `POST /api/v1/query`**

Accepts a request body with:
- `source` — which dataset to query (e.g., `"order_summary"`)
- `metrics` — array of metric keys (e.g., `["approvals", "revenue", "approval_rate"]`)
- `dimensions` — array of dimension keys to GROUP BY (e.g., `["campaign_name"]`)
- `filters` — object containing:
  - `dateRange` — preset (`"last_7_days"`) or custom (`{ start, end }`)
  - `dimensions` — per-dimension filter conditions (`{ "campaign_name": { "operator": "in", "values": ["Summer Sale"] } }`)
  - `search` — free-text search (for Transaction Explorer)
- `toggles` — active toggle states (e.g., `{ "approval_mode": "organic" }`)
- `parameters` — user-provided values (e.g., `{ "processing_fee_pct": 3.5 }`)
- `sort` — `{ "field": "revenue", "direction": "desc" }`
- `pagination` — `{ "page": 0, "pageSize": 25 }`
- `includeTotal` — whether to compute a totals row across all rows

Returns:
- `data.rows` — array of result objects with formatted values
- `data.totals` — aggregated totals (if requested)
- `metadata.totalRows` — total count (for pagination)
- `metadata.queryTimeMs` — execution time for monitoring

**2. Batch Query Endpoint — `POST /api/v1/query/batch`**

When a user opens a report, the frontend sends ONE HTTP request containing all queries for every visual on the page. The backend executes all queries in parallel and returns results keyed by visual ID.

Accepts:
- `filters` — shared across all queries (date range, dimension filters)
- `toggles` — shared toggle state
- `parameters` — shared parameter values
- `queries` — array of individual query objects, each with a unique `visualId` plus its own `source`, `metrics`, `dimensions`, and optional overrides

Returns:
- `results` — object keyed by visualId: `{ "summary_cards": { data, metadata }, "trend_chart": { data, metadata }, ... }`
- `metadata.totalTimeMs` — overall batch execution time

Why this matters: 1 HTTP request instead of 5-10. All queries run in parallel on the backend. Total latency = slowest single query, not sum of all.

**3. Query Processing Pipeline**

The engine resolves each query through these steps:

**Source Resolution:** `"order_summary"` + client ID `10043` → `"reporting.order_summary_10043"`

**Metric Resolution (the core algorithm):**
1. Look up each requested metric key in the `metrics` table
2. Check if any active toggle has a variant for this metric in `metric_variants`
3. If variant exists, use the variant's SQL expression instead of the default
4. If the metric is derived (e.g., `approval_rate` formula: `{approvals} / {attempts}`):
   - Parse the formula
   - Recursively resolve each referenced metric (including THEIR toggle variants)
   - This is "variant cascading" — changing `approval_mode` to `organic` automatically makes `approval_rate` use `approvals_organic / attempts` without any explicit mapping
5. If the metric uses parameters (e.g., `processing_fees = {revenue} * {processing_fee_pct} / 100`):
   - Substitute parameter values from the request

**Dimension Resolution:**
- Look up each dimension key in the `dimensions` table
- If the dimension requires a JOIN (e.g., `campaign_name` lives in `public.campaigns`), add the JOIN clause
- Handle date dimensions: `date_day` = `date`, `date_week` = `DATE_TRUNC('week', date)`, `date_month` = `DATE_TRUNC('month', date)`

**Filter Building:**
- ALWAYS inject `WHERE client_id = {value from JWT}` (tenant isolation)
- Date range: convert presets to actual dates, add `date >= ... AND date <= ...`
- Dimension filters: convert to parameterized `IN`, `NOT IN`, `=`, `LIKE` clauses
- Search: convert to parameterized `ILIKE` across configured search columns
- All values are SQL parameters (`$1`, `$2`, ...) — zero string concatenation

**SQL Assembly:** Combine SELECT (metrics) + FROM (source table) + JOINs (dimensions) + WHERE (filters) + GROUP BY (dimensions) + ORDER BY (sort) + LIMIT/OFFSET (pagination) into a single parameterized query.

**Execution:** Run against a dedicated `pg` connection pool (separate from Prisma, which handles CRUD operations).

**Response Formatting:** Format raw database rows using each metric's `data_type` and `format_pattern` from the catalog. Integers get commas, currencies get `$` prefix with 2 decimals, percentages get `%` suffix.

**4. Comparison Period Queries**

When a visual has `showTrend: true`:
- The engine automatically runs a second query for the comparison period
- "Last 7 Days" → comparison = the 7 days before that
- "Last 30 Days" → comparison = the 30 days before that
- "MTD" → same days last month
- Returns comparison data alongside main data so Cards can show "+12% vs prior"

**5. Security**

| Threat | Protection |
|--------|-----------|
| SQL Injection | All filter values parameterized. Never concatenated. |
| Cross-tenant access | client_id from JWT token, never from request body. Every query includes tenant filter. |
| Column injection | Only metric/dimension keys registered in the catalog are queryable. Arbitrary SQL rejected. |
| Slow query abuse | 30-second timeout per query. Max page size enforced. |
| Auth bypass | All endpoints protected by existing JWT auth middleware (brought over from `beast-insights-backend`). |

**6. Catalog Cache**

The metric/dimension/toggle catalog is loaded into memory on server startup and refreshed every 5 minutes. This avoids hitting the `report_builder` tables on every query — instead, the engine reads from an in-memory Map.

### What You Can Verify

- Single query: request `approvals` and `revenue` grouped by `campaign_name` for `last_7_days` → correct data returns
- Batch: send 5 queries in one request → all 5 results return in one response, keyed by visualId
- Toggle test: same query with `approval_mode: "standard"` vs `"organic"` → different approval numbers
- Derived cascade: request `approval_rate` with organic toggle → rate uses organic approvals automatically
- Parameter test: request `processing_fees` with `processing_fee_pct: 5.0` → correct calculation
- Filter test: add `campaign_name IN ["Summer Sale"]` → only Summer Sale data returns
- Security: try querying with a different client's ID → fails
- Security: try injecting SQL through a filter value → parameterized, no injection possible
- Performance: batch of 5 queries completes in <200ms

---

## Milestone 4: Frontend Report Renderer

### What This Is

The complete React component library that renders any report from a JSON template (defined in M2). Generic, reusable components that receive data and configuration — they don't know which report they're in.

**Repo:** `beastinsights` (new empty repo). Separate subdomain deployment. Uses **Tremor** as the primary charting/analytics UI library, and **shadcn** for sidebar, navigation, and components where Tremor lacks coverage.

> **Design reference:** Refer to `beastinsights-old` for UI design patterns, color palette, and layout decisions.

### Deliverables

**1. State Management Setup**

| System | What It Manages |
|--------|----------------|
| **TanStack Query** (new) | All data fetching for native reports. 5-minute stale time — repeat page visits are instant. Handles caching, deduplication, background refetching, loading/error states. |
| **Zustand** (new) | Report-level UI state: date range, active dimension filters, toggle values, parameter values, group-by selection, compare slicer. Each report page gets its own store instance. When any value changes, TanStack Query automatically refetches affected data. |

**2. Data Fetching Hook — `useReportData`**

The bridge between the template JSON, the Zustand filter state, and the backend API:
1. Reads the template JSON to determine what queries each visual needs
2. Reads current filter/toggle/parameter state from Zustand
3. Constructs the batch query payload (one visualId per visual)
4. Passes to TanStack Query which handles caching and fetching
5. Returns `{ data, isLoading, error }` keyed by visual ID
6. When any filter changes in Zustand, TanStack Query detects the dependency change and refetches

**3. Visual Components**

| Component | File | What It Renders | Key Behaviors |
|-----------|------|----------------|--------------|
| **CardVisual** | `CardVisual.jsx` | Summary card with 1-3 metrics. Each shows a formatted value ($45,678 or 72.3% or 1,234) plus optional trend arrow (+12% up in green or -5% down in red). `invertTrend` flips colors for metrics where lower is better (cancel rate, CB rate). | Formats based on metric catalog's `data_type`. Currency = `$`, percentage = `%`, integer = commas. |
| **ChartVisual** | `ChartVisual.jsx` | Tremor chart wrapper. Line, bar, area, stacked bar, pie, donut. Configurable X axis, multiple series with individual colors, dual Y axes, legend, tooltips. Responds to `{time_granularity}` placeholder in xAxis — switches from daily to weekly to monthly grouping. | Responsive to container width. Theme-aware colors (dark/light). |
| **TableVisual** | `TableVisual.jsx` | TanStack Table wrapper. Sortable columns (click header), pagination (configurable page size), totals row, conditional formatting (red when CB rate > 1%), column toggle (show/hide via dropdown), exportable. Responds to `{groupBy}` placeholder — rows regroup when user changes the group-by selector. | Server-side pagination mode for Transaction Explorer (millions of rows). |
| **MatrixVisual** | `MatrixVisual.jsx` | Cohort/retention matrix. Rows = cohort periods, columns = billing cycles or months. Cells show values with heatmap coloring gradient (configurable colors). Transforms flat data from backend into matrix format on the frontend. | Green-to-red gradient configurable per template. |
| **WaterfallVisual** | `WaterfallVisual.jsx` | P&L visualization. Custom Tremor stacked bar where each bar "hangs" from the previous total. Start bar (Revenue) at top, subtract bars (costs) descend, end bar (Profit) shows the result. | Recalculates when parameter inputs change. |
| **ParameterInput** | `ParameterInput.jsx` | Number input with $ prefix or % suffix. Step controls for precision. Bound to Zustand `parameters` state — changing a value triggers recalculation of all dependent metrics. | Used in Financial Performance report for processing fees, reserve, CB fees, CPA. |
| **HealthIndicator** | `HealthIndicator.jsx` | Status badge: green (Good), yellow (Warning), red (Critical) based on configurable thresholds. | Used for MID health, payment success rates. |

All visual components live in `components/visuals/`.

**4. Layout Components**

| Layout | Behavior |
|--------|----------|
| `CardRow` | Equal-width cards in a horizontal row. 4 per row on desktop, 2 on tablet, 1 on mobile. |
| `FullWidth` | Single column, full content width. |
| `Grid` | CSS grid with configurable columns (1-4). |
| `TabsSection` | Tabbed interface using shadcn Tabs. Lazy-loads content when tab is selected. |

**5. ReportRenderer Component**

The orchestrator that reads a template JSON and builds the full report:
1. Iterates through `template.sections[]`
2. Checks `conditionalVisibility` — skips sections hidden by current toggle state
3. Renders each section in its layout type
4. Maps each `visual.type` to the appropriate component
5. Passes `data[visual.id]` from the batch query response to each visual
6. Per-visual loading state: shows VisualSkeleton matching the visual's expected dimensions
7. Per-visual error state: shows VisualError with retry button — one visual failing does NOT break the rest of the page

**6. Sidebar & Navigation**

Sidebar and navigation components built with shadcn. Reports organized by category (Monitor/Grow/Retain/Acquire/Profit/Explore).

**7. Dark Mode & Responsive**

All components support the existing dark/light theme system (no new theme code — uses existing `theme-provider.jsx`). Charts use theme-aware colors. On mobile (<768px), card rows stack, charts resize, tables become horizontally scrollable with a sticky first column. The slicer panel collapses into a "Filters" drawer.

### What You Can Verify

- Render each visual type with sample data — correct display and formatting
- CardVisual shows "$45,678" not "45678", "72.3%" not "0.723", "+12% up" for positive trends
- ChartVisual renders line, bar, area, pie, stacked bar correctly with legends
- TableVisual sorts by clicking headers, paginates, shows totals row
- MatrixVisual shows a proper matrix with heatmap coloring
- WaterfallVisual shows revenue minus costs equals profit
- Change a Zustand value → data refetches automatically
- Dark mode toggle → all components re-render with dark colors
- Mobile viewport (375px) → everything stacks and is usable
- One visual error → other visuals unaffected

---

## Milestone 5: Data Library from Power BI

### What This Is

This is the research and extraction milestone. We read the PBI report files from the `PBI-beastinsights-prod` folder, extract every DAX measure, every dimension relationship, and every slicer definition from the existing PBI semantic model — and translate them into SQL expressions stored in the `report_builder` tables created in M1.

This is where the 100+ metric catalog, 30+ dimension catalog, and toggle variant system get populated with real data that matches what Power BI currently calculates.

### Why This Is Critical

The native reports must produce **exactly the same numbers** as Power BI. If Revenue Analytics shows $427,450 in Power BI, it must show $427,450 in the native version. This milestone ensures 1:1 calculation parity by directly translating every DAX measure to its PostgreSQL equivalent.

### Source

All PBI report files are read from the **`PBI-beastinsights-prod`** folder. This contains the production Power BI semantic model with all DAX measures, dimension relationships, and slicer configurations. The extraction process reads these files to systematically catalog every measure and translate it to SQL.

### Deliverables

**1. Metric Extraction: DAX → SQL Translation**

We read the PBI report files from `PBI-beastinsights-prod` to extract every DAX measure and translate it into a SQL expression that works against the materialized views.

**Sales & Volume Metrics (from `order_summary`):**

| PBI DAX Measure | Native SQL Expression | Notes |
|----------------|----------------------|-------|
| `# Attempts = SUM(smz-order_details[attempts])` | `SUM(attempts)` | Direct 1:1 |
| `# Approvals = SUM(smz-order_details[approvals])` | `SUM(approvals)` | Direct 1:1 |
| `# Net Approvals = SUM(smz-order_details[net_approvals])` | `SUM(net_approvals)` | Toggle variant |
| `# Organic Approvals = SUM(smz-order_details[approvals_organic])` | `SUM(approvals_organic)` | Toggle variant |
| `# Cancels = SUM(smz-order_details[cancel])` | `SUM(cancel)` | Direct 1:1 |
| `$ Revenue = SUM(smz-order_details[revenue])` | `SUM(revenue)` | Direct 1:1 |
| `# Approval Toggle` | Resolves via `metric_variants` table | Backend resolves based on active toggle value |

**Rate Metrics (derived):**

| PBI DAX Measure | Native SQL Expression | Notes |
|----------------|----------------------|-------|
| `Approval % = DIVIDE([# Approval Toggle], [# Attempts])` | `CASE WHEN SUM(attempts) > 0 THEN SUM(approvals)::decimal / SUM(attempts) ELSE 0 END` | Derived: `{approvals} / {attempts}`. Cascades with approval_mode toggle. |
| `CB % = DIVIDE([# CB], [# Approval Toggle])` | `CASE WHEN SUM(approvals) > 0 THEN SUM(cb)::decimal / SUM(approvals) ELSE 0 END` | Derived: `{cb_count} / {approvals}` |
| `Cancel % = DIVIDE([# Cancels], [# Approval Toggle])` | `CASE WHEN SUM(approvals) > 0 THEN SUM(cancel)::decimal / SUM(approvals) ELSE 0 END` | Derived: `{cancels} / {approvals}` |
| `Refund % = DIVIDE([# Refund], [# Approval Toggle])` | `CASE WHEN SUM(approvals) > 0 THEN SUM(refund)::decimal / SUM(approvals) ELSE 0 END` | Derived |

**Sales Type Segmentation:**

| PBI DAX | Native Approach |
|---------|----------------|
| `# Initials = CALCULATE([# Approval Toggle], FILTER(..., sales_type="Initials"))` | Use dimension filter: `WHERE sales_type = 'Initials'`. Or: dedicated metric `SUM(CASE WHEN sales_type = 'Initials' THEN approvals ELSE 0 END)` |
| `# Rebills = CALCULATE([# Approval Toggle], FILTER(..., sales_type="Rebills"))` | Same approach with `sales_type = 'Rebills'` |
| `$ Initials`, `$ Rebills`, `$ Straight Sales` | Same pattern for revenue split by sales_type |

**Dispute Metrics with Date Basis Toggle:**

| PBI DAX | Native SQL | Toggle |
|---------|-----------|--------|
| `# CB (by order date) = SUM(cb)` | `SUM(cb)` | `date_basis = "order_date"` |
| `# CB (by CB date) = SUM(cb_cb_date)` | `SUM(cb_cb_date)` | `date_basis = "cb_date"` |
| `$ CB (by order date) = SUM(cb_dollar)` | `SUM(cb_dollar)` | `date_basis = "order_date"` |
| `$ CB (by CB date) = SUM(cb_cb_date_dollar)` | `SUM(cb_cb_date_dollar)` | `date_basis = "cb_date"` |
| Same pattern for refund_count, refund_dollar | Same columns with `_refund_date` suffix | Same toggle |

**Profitability Metrics (parameterized):**

| PBI DAX | Native SQL | Parameters |
|---------|-----------|-----------|
| `$ Processing Fees = prof_processing_fees[value] * [$ Revenue] / 100` | `SUM(revenue) * {processing_fee_pct} / 100` | `processing_fee_pct` (default: 3.5) |
| `$ CB Fees = prof_cb_fees[value] * [# CB]` | `SUM(cb) * {cb_fee_amount}` | `cb_fee_amount` (default: 25) |
| `$ Reserve = prof_reserve[value] * [$ Revenue] / 100` | `SUM(revenue) * {reserve_pct} / 100` | `reserve_pct` (default: 5) |
| `$ CPA Cost = SUM(cpa)` | `SUM(cpa)` | (uses campaign CPA from dimension table, or `{cpa_amount} * initials`) |
| `$ Gross Profit = [$ Revenue] - [$ Refund] - [$ CB] - [$ Processing] - [$ CB Fees] - [$ CPA] - [$ Reserve]` | Derived formula: `{revenue} - {refund_dollar} - {cb_dollar} - {processing_fees} - {cb_fees} - {cpa_cost} - {reserve}` | All parameters cascade |
| `% Gross Margin = DIVIDE([$ Gross Profit], [$ Revenue])` | Derived: `{gross_profit} / {revenue}` | |

**Decline & Recovery Metrics (from `decline_recovery`):**

| Metric | SQL | Source |
|--------|-----|--------|
| Organic Declines | `SUM(organic_declines)` | `decline_recovery` |
| Reattempts | `SUM(reattempts)` | `decline_recovery` |
| Recovered | `SUM(recovered)` | `decline_recovery` |
| Recovery Rate | `{recovered} / {organic_declines}` | Derived |
| Recovered $ | `SUM(recovered_dollar)` | `decline_recovery` |
| Recovery Gap | `{organic_declines} - {recovered}` | Derived |

**Cohort & LTV Metrics (from `cohort_summary`):**

| Metric | SQL | Notes |
|--------|-----|-------|
| LTV 30/60/90/180 day | Cumulative revenue per cohort up to N days | Requires cycle-based aggregation |
| Retention Cycle 1-12 | `SUM(approvals) WHERE billing_cycle = N` / base | Base changes with retention_base toggle |

**Hourly / Real-Time Metrics (from `hourly_revenue`):**

| Metric | SQL |
|--------|-----|
| Today Revenue | `SUM(today_revenue)` |
| Today Initials | `SUM(today_initial)` |
| Today Rebills | `SUM(today_rebill)` |
| 7-Day Avg Revenue | `SUM(avg_7d_revenue)` |
| Today Success Rate | Derived from today's approvals / attempts |

**MID Health Metrics (from `mid_summary`):**

| Metric | SQL |
|--------|-----|
| Volume | `SUM(volume)` |
| CB Rate | `cb_rate` (pre-computed) |
| CB Rate (Visa/MC) | `cb_visa_rate`, `cb_master_rate` |
| Health Tag | `health_tag` (categorical) |
| Capacity Left | `capacity_left` |

**Total metric count: ~100 metrics** across Sales, Revenue, Rates, Disputes, Profitability, Recovery, Cohort/LTV, Real-Time, and MID Health categories.

All extracted DAX measures are translated to SQL and stored in `report_builder.metrics`.

**2. Dimension Extraction**

Every dimension users can group or filter by, with their JOIN information:

| Dimension | Column | Source Table | JOIN Required? |
|-----------|--------|-------------|---------------|
| campaign_name | `c.campaign_name` | `public.campaigns` | Yes — JOIN on `campaign_id` |
| campaign_type | `c.campaign_type` | `public.campaigns` | Yes — same JOIN |
| product_name | Product column or JOIN | `public.offers` or direct | Depends on view |
| gateway_alias | `p.gateway_alias` | `beast_insights_v2.payments` | Yes — JOIN on `gateway_id` |
| acquirer | `p.lender` | `beast_insights_v2.payments` | Yes — same JOIN |
| mid_corp | `p.corp` | `beast_insights_v2.payments` | Yes — same JOIN |
| card_brand | `bl.card` | `public.bin_lookup` | Yes — JOIN on `bin` |
| card_type | `bl.type` | `public.bin_lookup` | Yes — same JOIN |
| bank | `bl.bank` | `public.bin_lookup` | Yes — same JOIN |
| card_country | `bl.country` | `public.bin_lookup` | Yes — same JOIN |
| sales_type | `os.sales_type` | Direct on view | No |
| billing_cycle | `os.billing_cycle` | Direct on view | No |
| decline_group | `os.decline_group` or `dg.decline_group` | Direct or `public.decline_groups` | Depends |
| affid | `os.affid` | Direct on view | No |
| sub_affid | `os.sub_affid` | Direct on view | No |
| date_day | `os.date` | Direct | No |
| date_week | `DATE_TRUNC('week', os.date)` | Computed | No |
| date_month | `DATE_TRUNC('month', os.date)` | Computed | No |
| date_hour | `hr.hour` | Direct on hourly_revenue | No |
| price_point | `os.price_point` | Direct on view | No |
| refund_type | `os.refund_type` | Direct on view | No |
| alert_type | `os.alert_type` | Direct on view | No |

**3. Toggle & Variant Mapping**

Direct translation from PBI slicers/parameter tables:

| PBI Slicer/Table | Native Toggle | Options | Affected Metrics |
|-----------------|---------------|---------|-----------------|
| `cust_approval_page_toggle` | `approval_mode` | standard (= Approvals), organic (= Organic), net (= Net) | approvals, and cascades to approval_rate, cb_rate, cancel_rate, refund_rate, initials, rebills |
| `cust_cb_refund_date_toggle` | `date_basis` | order_date, cb_date | cb_count, cb_dollar, refund_count, refund_dollar |
| `param_date_selection` (Day/Week/Month) | `time_granularity` | day, week, month | Changes GROUP BY dimension, not metric SQL |
| Profitability slider tables (`prof_*`) | `parameters` | Numeric inputs | processing_fees, cb_fees, reserve, cpa_cost, gross_profit, gross_margin |
| `cust_month_year` | `month_selector` | Available months | MID Summary date filtering |

**4. Dataset Mapping**

| Source Key | Materialized View Pattern | Primary Date Column | PBI Equivalent Table |
|-----------|--------------------------|--------------------|--------------------|
| `order_summary` | `reporting.order_summary_{client_id}` | `date` | `smz-order_details` |
| `mid_summary` | `reporting.mid_summary_{client_id}` | `month_year` | `smz_mid_table` |
| `cb_refund_alert` | `reporting.cb_refund_alert_{client_id}` | `date` | `smz-order_details` (with cb_no_of_days) |
| `cohort_summary` | `reporting.cohort_summary_{client_id}` | `date` | `smz_cohort` / `smz_ltv` |
| `decline_recovery` | `reporting.decline_recovery_{client_id}` | `date` | `smz_decline_recovery` |
| `hourly_revenue` | `reporting.hourly_revenue_{client_id}` | `hour` | Hourly aggregation tables |
| `alert_summary` | `reporting.alert_summary_{client_id}` | `date` | `smz_alert` |
| `alert_details` | `reporting.alert_details_{client_id}` | `alert_date` | `fact_alert` |
| `order_details` | `reporting.order_details_{client_id}` | `date` | `smz-order_details` (row-level) |
| `products` | `reporting.products_{client_id}` | — | `dim_products` |

**5. Junction Tables**

For each metric, map which datasets it's available in. For each dimension, map which datasets support it. This prevents the query engine from generating invalid SQL (e.g., requesting `ltv_30d` from `hourly_revenue`).

**6. Seed Scripts**

Everything above is encoded into database seed scripts that populate the tables from M1:
- All ~100 metrics with SQL expressions
- All ~30 dimensions with column expressions and JOIN info
- All 10 datasets with table patterns
- All 6 toggles with options
- All ~20 metric variants
- All junction table mappings

### What You Can Verify

- Query `report_builder.metrics` → ~100 rows, each with valid SQL
- Query `report_builder.dimensions` → ~30 rows, each with column and join info
- Query `report_builder.data_source_metrics` → every metric linked to its valid sources
- Run a test query through the engine (M3) with real seeded metrics → correct numbers returned
- Compare specific metric calculations: native `approval_rate` for client 10000 for Jan 2026 matches PBI's `Approval %` for the same client and period
- Toggle variant lookup: `(approvals, approval_mode, organic)` → returns `SUM(approvals_organic)`
- Derived metric expansion: `gross_profit` formula references 7 sub-metrics, all resolve correctly

---

## Milestone 6: Reports, Filters & Navigation

### What This Is

Everything comes together. The 11 stock report templates are created as JSON documents (following the M2 spec), stored in the database, and made accessible through the sidebar navigation. The slicer panel is fully wired. The frontend is deployed on a separate subdomain as a fresh application.

### Deliverables

**1. All 11 Report JSON Templates**

Each template follows the format spec from M2 and references metrics/dimensions from the data library (M5). Templates are stored in `report_builder.report_templates` and served through the API.

| # | Report | Category | Primary Source | Key Features |
|---|--------|----------|---------------|--------------|
| 1 | Business Command Center | MONITOR | `order_summary` + `hourly_revenue` | Summary cards with trends, hourly revenue chart (today vs 7d avg), daily summary table. Auto-refreshes every 5 min. |
| 2 | Real-Time Pulse | MONITOR | `hourly_revenue` | Fixed to "today". Running total cards, hourly charts (revenue + success rate), hourly breakdown table. Auto-refreshes. |
| 3 | Revenue Analytics | GROW | `order_summary` | Slicers: approval_mode, time_granularity. Group by: campaign/product/gateway/sales_type/card_brand. Summary cards (initials/rebills/total/rates), stacked trend chart, full breakdown table with column toggle. |
| 4 | Subscription Intelligence | GROW | `order_summary` | 6-month default. MRR cards, stacked revenue trend (new + recurring), monthly subscriber flow table. |
| 5 | Customer Lifecycle | GROW | `cohort_summary` + `order_summary` | Slicer: retention_base. LTV cards (30/90/180d), cohort retention heatmap (Matrix), LTV by acquisition channel table. |
| 6 | Churn Analysis | RETAIN | `order_summary` | Group by: product/campaign/billing_cycle. Cancel/CB/refund rate cards (inverted trends), churn trend chart, breakdown table. |
| 7 | Payment Health & Recovery | RETAIN | `order_summary` + `decline_recovery` | Slicers: approval_mode, time_granularity. Approval rate card, trend chart, recovery table by decline group, approval breakdown table. |
| 8 | Channel & Acquisition | ACQUIRE | `order_summary` | 90-day default. New customer cards, channel scorecard table with approval/cancel/CB rates per campaign. |
| 9 | Financial Performance | PROFIT | `order_summary` | Slicer: profitability_view. Parameters: processing %, reserve %, CB fee $, CPA $. P&L waterfall chart (conditional on cashflow view), profit cards, profitability table by product/campaign. |
| 10 | Product & Plan Performance | PROFIT | `order_summary` | Product scorecard table with all key metrics, stacked bar chart showing revenue by product over time. |
| 11 | Transaction Explorer | EXPLORE | `order_details` | Search bar, status filter pills, server-side pagination (50/page), server-side sorting, column toggle, full transaction detail columns. |

**2. Template API**

| Endpoint | Returns |
|----------|---------|
| `GET /api/v1/reports/templates` | Lightweight list of available templates for the authenticated user — key, name, category, icon. Used by the sidebar. |
| `GET /api/v1/reports/templates/:key` | Full template with layout JSON. Used by the report page to render. |

**3. Filter Options API**

| Endpoint | Returns |
|----------|---------|
| `GET /api/v1/filters/options/:dimensionKey` | Distinct values for a dimension, scoped to the authenticated client. Used to populate filter dropdowns. E.g., `/filters/options/campaign_name` → `["Summer Sale", "Google Ads", "Facebook", "Email"]` |

**4. Complete Slicer Panel**

Fully wired slicer panel that works across all 11 reports:
- **Date range picker** — presets (Today, Last 7/14/30/90 Days, MTD, QTD, YTD, Last Month) + custom calendar
- **Dimension filter dropdowns** — populated from the filter options API, multiselect with search
- **Slicer controls** — pills or dropdown depending on template config
- **Parameter inputs** — number fields with $/% for profitability
- **Group-by selector** — dropdown of available dimensions per report
- **Compare slicer** — enables previous period comparison on Cards
- **Search bar** — for Transaction Explorer
- **Status filter pills** — for Transaction Explorer (All/Approved/Declined/Refunded/Chargeback)

All filter changes write to the Zustand store → TanStack Query detects the change → new batch query fires → all visuals update. The cycle completes in ~200-400ms.

**5. Dynamic Report Page**

Single route (`/reports/[reportSlug]`) that:
1. Reads the slug from URL
2. Fetches the template from the API
3. Creates a Zustand store for this report's filter state
4. Renders SlicerPanel + ReportRenderer
5. Manages loading skeleton and error states

**6. Navigation**

Fresh sidebar navigation built with shadcn (new frontend repo on separate subdomain). Reports organized by category:
- **Monitor:** Business Command Center, Real-Time Pulse
- **Grow:** Revenue Analytics, Subscription Intelligence, Customer Lifecycle
- **Retain:** Churn Analysis, Payment Health & Recovery
- **Acquire:** Channel & Acquisition
- **Profit:** Financial Performance, Product & Plan Performance
- **Explore:** Transaction Explorer

### What You Can Verify

- Navigate to each of the 11 reports → renders with real data from the seeded data library
- Revenue Analytics: change date to Last 30 Days → numbers update
- Revenue Analytics: switch Approval Mode to "First Attempt" → approval numbers change across all visuals
- Revenue Analytics: change Group By from Campaign to Product → table rows change
- Financial Performance: change Processing Fee from 3.5% to 5% → profit numbers change, waterfall chart updates
- Customer Lifecycle: cohort heatmap shows matrix with colored cells
- Transaction Explorer: search by email → matching transactions appear. Paginate → page 2 shows different rows.
- Command Center: leave page open 5+ minutes → data refreshes silently
- Filter dropdowns populate with real dimension values from the database
- Sidebar navigation shows all 11 reports organized by category

---

## Milestone 7: End-to-End Testing & Rollout

### What This Is

Everything from M1-M6 is functional. This milestone validates performance, verifies data accuracy against Power BI, hardens error handling, and prepares for client rollout on the new subdomain.

### Deliverables

**1. Performance Validation**

Target: every report loads in under 500ms (navigation click to all visuals rendered with data).

Expected breakdown:
| Step | Target |
|------|--------|
| HTTP round trip + auth | ~25ms |
| Query engine (SQL generation) | ~15ms |
| PostgreSQL (parallel queries on indexed materialized views) | 50-100ms |
| Response formatting + transfer | ~20ms |
| React rendering (charts, tables) | 100-200ms |
| **Total** | **~250-400ms** |

If any report exceeds 500ms:
- Profile with `EXPLAIN ANALYZE` to find slow queries
- Add missing indexes on materialized views (most common fix)
- Optimize complex derived metric SQL expressions
- Reduce visual count per page if excessive

**2. Data Accuracy Validation**

The single most important test. For each of the 11 reports:
- Open the native report and the equivalent Power BI report side by side
- Same client, same date range, same filters, same slicers
- Compare every summary card number, every chart data point, every table row
- Numbers must match exactly (or within rounding tolerance for percentages)
- Document any discrepancy, find the root cause, fix it
- **No client gets access until accuracy is verified**

Specific comparisons:
| Native Report | PBI Page to Compare Against | Key Numbers to Match |
|--------------|----------------------------|---------------------|
| Revenue Analytics | Sales page | Initials count, Rebills count, Revenue, AOV per campaign |
| Payment Health | Approval % page | Approval rate, Attempts, Approvals per gateway |
| Customer Lifecycle | LTV / Retention pages | Retention % per cohort/cycle, LTV values |
| Financial Performance | Profitability page | Gross profit, margin % (with same fee parameters) |
| Churn Analysis | CB & Refunds page | CB rate, refund rate, cancel rate per product |
| Transaction Explorer | Order Details page | Row counts, amounts, statuses match |

Slicer comparisons:
- Switch to organic approval mode in both native and PBI → numbers must match
- Switch date basis to CB date in both → CB numbers must match

**3. Frontend Performance Optimization**

- Visual components wrapped in `React.memo` — prevents re-renders when sibling visuals update
- Heavy components (Charts, Tables) loaded via `React.lazy` + `Suspense` — code splitting
- Large tables use `react-window` (already installed) for virtual scrolling
- Template JSON prefetched on sidebar hover
- Skeleton loaders match visual dimensions — no layout shift when data loads

**4. Error Resilience**

| Scenario | Behavior |
|----------|---------|
| One visual query fails | That visual shows error with retry button. All others work. |
| Entire batch fails | Page shows error with "Retry All". |
| Backend down | TanStack Query retries 3x. Shows cached data if available from previous load. |
| Query exceeds 30s | Visual shows "Query timed out — try a narrower date range." |
| No data for date range | Visuals show empty state: "No data for this period." |
| Template not found | Page shows "Report not available." |

Per-visual error boundaries ensure one broken visual never crashes the page.

**5. Mobile Responsiveness**

All 11 reports checked on mobile (375px viewport):
- Card rows: 2 columns on tablet, 1 on mobile
- Charts: resize to viewport width
- Tables: horizontal scroll with sticky first column
- Slicer panel: collapses into a "Filters" button → opens a drawer/sheet
- Slicer pills: wrap to multiple lines

**6. Monitoring**

Every query is logged:
- Client ID, dataset, metric count, dimension count
- Execution time (ms)
- Error status
- Total rows returned

Admin visibility into:
- P50 / P95 / P99 query latencies per dataset
- Error rate per report
- Most queried reports and metrics

**7. Rollout Plan**

Recommended sequence:
1. Deploy `beastinsights` frontend to the new subdomain and `beastinsights-backend` to its endpoint
2. Internal team (Beast Demo account) validates all 11 reports over 1 week
3. Grant 2-3 beta clients access to the new subdomain
4. Gather feedback, fix issues
5. Gradually roll out access to remaining clients
6. Eventually decommission Power BI workspace (end of rollout)

### What You Have at the End of Phase 1

```
A fully functional native reporting platform:

  11 interactive reports with real data
  ~100 queryable metrics across 10 datasets
  ~30 filterable dimensions
  6 metric toggles with automatic variant cascading
  Dynamic filters, group-by, and compare to previous period
  All chart types: line, bar, area, pie, stacked bar, waterfall
  Sortable, paginated tables with column toggle and totals
  Cohort retention heatmaps with configurable coloring
  Parameterized profitability calculations
  Auto-refreshing real-time reports
  Server-side paginated transaction explorer with search
  Dark mode and mobile responsive
  <500ms page load
  Validated data accuracy vs Power BI
  Separate subdomain deployment (beastinsights + beastinsights-backend)
  Tremor + shadcn UI component library
  Zero changes to existing login, auth, or billing code

Ready for Phase 2: Custom Report Builder, bookmarks, filter presets,
                    scheduled delivery, role-based page access
```

---

*Phase 1 Milestones — Beast Insights*

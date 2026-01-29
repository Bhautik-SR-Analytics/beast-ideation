# Milestone 2 — JSON Template Format Specification

> Definitive specification for the `report_templates.layout` JSONB column. This is the contract between the backend query engine and the frontend report renderer.

> **Terminology:** This document uses Power BI terminology — visuals (not widgets), slicers (not toggles), slicer panel (not filter bar), matrix (not pivot table), bookmarks (not saved views), tiles (not dashboard widgets).

---

## Table of Contents

1. [Overview](#1-overview)
2. [Template Root Structure](#2-template-root-structure)
3. [Slicer Panel](#3-slicer-panel)
4. [Section Types](#4-section-types)
   - [4.8 Visual Size Constraints](#48-visual-size-constraints)
5. [Visual Types](#5-visual-types)
   - [5.1 Card](#51-card) (7 card variants)
6. [Cross-Cutting Concerns](#6-cross-cutting-concerns)
   - [6.9 URL Deep Linking](#69-url-deep-linking)
7. [Complete Stock Report Examples](#7-complete-stock-report-examples)
   - [7.5 Sales Report](#75-sales-report)
   - [7.6 Complete Reference Template](#76-complete-reference-template-all-features)
8. [Naming Conventions & Validation Checklist](#8-naming-conventions--validation-checklist)
9. [Changes from Earlier Documents](#9-changes-from-earlier-documents)

---

## 1. Overview

### 1.1 What This Document Defines

The JSON template stored in `report_builder.report_templates.layout` (JSONB column) defines everything about how a report looks and behaves:

- **What data to fetch** — which metrics, dimensions, and data sources each visual needs
- **How to render it** — visual type, chart configuration, table columns, layout sections
- **What controls appear** — slicers, filters, parameters, search, table-level group-by selectors
- **Special behaviors** — auto-refresh, conditional visibility, inverted trends, server-side pagination

The JSON does **NOT** contain:
- SQL expressions (those live in `report_builder.metrics` and `report_builder.dimensions`)
- API URLs or endpoint paths (the frontend always uses `POST /api/v1/query/batch`)
- Actual data (fetched at render time)
- Filter option values (fetched via `GET /api/v1/filters/options/:dimensionKey`)

### 1.2 How It's Used

```
1. User navigates to a report
2. Frontend fetches: GET /api/v1/reports/templates/:templateKey
3. Response includes the layout JSON
4. Frontend ReportRenderer parses the JSON:
   a. Renders the SlicerPanel from slicerPanel config
   b. Renders sections in order
   c. For each visual, extracts metric/dimension keys
   d. Combines with current slicer state (date range, filters, toggles)
   e. Builds ONE batch request: POST /api/v1/query/batch
5. Backend resolves metric keys → SQL, runs queries in parallel
6. Frontend distributes results by visualId, each visual renders by type
```

### 1.3 Where It's Stored

```sql
-- report_builder.report_templates
SELECT layout FROM report_builder.report_templates WHERE template_key = 'revenue-analytics';
-- Returns the full JSON defined in this spec
```

---

## 2. Template Root Structure

### 2.1 Type Definition

```typescript
interface ReportTemplate {
  version: "1.0";
  source: string;                    // Primary data source key (e.g., "order_summary")
  sourceConfig?: SourceConfig;       // Optional source-level overrides
  defaults: TemplateDefaults;
  slicerPanel: SlicerPanel;
  sections: Section[];
}

interface SourceConfig {
  serverSidePagination?: boolean;    // Default: false
  serverSideSort?: boolean;          // Default: false
  serverSideSearch?: boolean;        // Default: false
  searchColumns?: string[];          // Columns searchable server-side
  dateFilterType?: "date_range" | "month_selector";  // Default: "date_range"
  dateColumn?: string;               // Override default date column
  dateFormat?: string;               // Display format for month_selector (e.g., "MMM YYYY")
}

interface TemplateDefaults {
  dateRange: DateRangePreset;        // Default date range preset when report first loads
  refreshInterval?: number | null;   // Auto-refresh in seconds, null = manual only
}

type DateRangePreset =
  | "today"
  | "yesterday"
  | "last_7_days"
  | "last_calendar_week"
  | "last_4_weeks"
  | "last_30_days"
  | "last_month"
  | "last_3_months"
  | "last_6_months"
  | "last_90_days"
  | "mtd"                            // Month-to-date
  | "qtd"                            // Quarter-to-date
  | "ytd";                           // Year-to-date
```

### 2.2 Field Reference

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `version` | `string` | Yes | — | Always `"1.0"`. For future schema evolution. |
| `source` | `string` | Yes | — | Primary data source key. Maps to `report_builder.data_sources.source_key`. Individual visuals can override this. |
| `sourceConfig` | `object` | No | `{}` | Source-level behavior overrides (server-side pagination, month selector, etc.) |
| `defaults.dateRange` | `string` | Yes | — | Default date range preset when the report first loads. |
| `defaults.refreshInterval` | `number\|null` | No | `null` | Auto-refresh interval in seconds. `300` = 5 minutes. `null` = no auto-refresh. |
| `slicerPanel` | `object` | Yes | — | Configuration for the slicer panel (top bar with filters, toggles, etc.) |
| `sections` | `array` | Yes | — | Ordered array of layout sections containing visuals. |

---

## 3. Slicer Panel

The slicer panel is the control bar at the top of every report. It contains date range picker, slicers (toggles), dimension filters, parameter inputs, search, and status filter pills.

### 3.1 Type Definition

```typescript
interface SlicerPanel {
  showDateRange: boolean;             // Show the date range picker
  showComparison?: boolean;           // Show comparison date range picker. Default: false
  comparisonPresets?: ComparisonPreset[];  // Available comparison options
  showSearch?: boolean;               // Show search input (for server-side search)
  searchPlaceholder?: string;         // Placeholder text for search input
  slicers?: SlicerConfig[];           // Toggle/slicer controls
  filters?: FilterConfig[];           // Dimension filter dropdowns
  parameters?: ParameterConfig[];     // User-input parameter fields
  statusFilters?: StatusFilterConfig[]; // Quick-filter pills (e.g., Approved/Declined)
  showExport: boolean;                // Show export button
  showSchedule?: boolean;             // Show "Schedule" button for report delivery. Default: false
  showReset?: boolean;                // Show "Reset Filters" button. Default: true
  showLastUpdated?: boolean;          // Show "Last updated: X ago" indicator. Default: false
  showShare?: boolean;                // Show "Share" button for URL deep linking. Default: false
}

type ComparisonPreset = "previous_period" | "previous_week" | "previous_month" | "custom";
```

### 3.2 Slicers

Slicers are context switches that change report behavior. There are 5 types, each handled differently by frontend and backend.

```typescript
interface SlicerConfig {
  slicerKey: string;                  // Maps to report_builder.metric_toggles.toggle_key
  position: "main" | "more";         // "main" = always visible, "more" = in overflow menu
  displayAs: "pills" | "select";     // UI rendering style
}
```

**The 6 slicers defined in the database:**

| Slicer Key | Type | Display | What It Does |
|------------|------|---------|-------------|
| `approval_mode` | metric_variant | pills | Changes approval SQL: standard/organic/net |
| `date_basis` | metric_variant | select | Changes CB/refund SQL: order_date/cb_date |
| `time_granularity` | dimension_switch | pills | Changes GROUP BY: day/week/month |
| `profitability_view` | layout_switch | pills | Shows/hides sections: gross/net/detailed |
| `retention_base` | metric_variant | pills | Changes cohort denominator: initials/rebills |
| `display_mode` | visibility | pills | Shows/hides metrics: count/dollar/percentage |

**How each type works:**

| Type | Frontend | Backend | API Field |
|------|----------|---------|-----------|
| `metric_variant` | Sends slicer state in request | Looks up variant SQL in `metric_variants` table | `toggles: { "approval_mode": "organic" }` |
| `dimension_switch` | Swaps dimension key in request | Groups by whatever dimension is requested | Dimension key changes in `dimensions` array |
| `visibility` | Shows/hides metrics in rendered visuals | No change (or frontend can optimize by excluding metrics) | No backend change |
| `layout_switch` | Shows/hides entire sections | Different sections may lazy-load different queries | Section `conditionalVisibility` |
| `parameter_input` | Sends parameter values in request | Uses values in metric calculations | `parameters: { "processing_fee_pct": 3.5 }` |

### 3.3 Filters

Filters are dimension-based controls populated from `report_template_filters` and dimension option values. The JSON template does **not** define filters directly — they come from the `report_template_filters` table joined with `dimensions`. However, the template references filter behavior:

```typescript
interface FilterConfig {
  dimensionKey: string;               // Maps to report_builder.dimensions.dimension_key
  displayAs: "pills" | "multiselect" | "select" | "search" | "tree";
  position: "main" | "more";         // "main" = always visible, "more" = in filter popover
  dependsOn?: string;                 // Parent dimension key — filter options narrow based on parent selection
}
```

**Filter dependencies (`dependsOn`):**

When a filter has `dependsOn` set, its available options are dynamically filtered based on the parent filter's selection. This prevents users from selecting combinations that have no data.

**Example — Product depends on Campaign:**
```json
{
  "filters": [
    { "dimensionKey": "campaign_name", "displayAs": "multiselect", "position": "main" },
    { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "main", "dependsOn": "campaign_name" }
  ]
}
```

**Behavior:**
1. User selects Campaign = "Summer Sale"
2. Frontend requests: `GET /api/v1/filters/options/product_name?campaign_name=Summer%20Sale`
3. Backend returns only products that have data for the selected campaign
4. Product filter dropdown shows the narrowed list

**Backend API:**
- Without dependency: `GET /api/v1/filters/options/:dimensionKey`
- With dependency: `GET /api/v1/filters/options/:dimensionKey?parentDimension=value`

The backend query adds a WHERE clause: `WHERE campaign_name IN ('Summer Sale')` before fetching distinct product names.

**Note:** Filter definitions (which filters appear on which report, display order, defaults) are stored in `report_template_filters`, not in the JSON. The JSON `filters` array is a rendering hint — it tells the frontend HOW to display each filter. The actual filter assignments, defaults, and cascade overrides are managed via:

1. `report_template_filters` — base config
2. `client_template_filter_overrides` — client admin overrides
3. `user_template_filter_overrides` — user personal overrides

**Filter operators in batch query requests:**

Filters are sent to the backend with an operator that controls the SQL WHERE clause:

```typescript
// Inside the batch query request:
interface FilterValue {
  operator: "in" | "not_in";         // "in" = include matching rows, "not_in" = exclude matching rows
  values: (string | number | boolean)[];
}

// Example batch query filters object:
{
  "filters": {
    "dateRange": { "from": "2026-01-01", "to": "2026-01-28" },
    "dimensions": {
      "campaign_name": { "operator": "in", "values": ["Summer Sale", "Fall Promo"] },
      "sales_type": { "operator": "in", "values": ["Initials"] }
    }
  }
}
```

When ALL values for a filter are selected (or no values selected), the filter is omitted from the request entirely — equivalent to "no filter applied." The backend treats a missing dimension filter as "include all."

**Nested filters and parent-child cascading:**

Filters with `displayAs: "tree"` or dimensions with `parent_dimension_id` set in the DB exhibit parent-child cascading. For example, `sales_type` (nested) has children: `Straight Sales`, `Initials`, `Rebills → [Cycle 1, Cycle 2, ... Cycle 6+]`. The cascading behavior is:

- Selecting a parent selects all its children
- Deselecting all children deselects the parent
- Partial child selection shows an indeterminate state on the parent

This is driven by `report_builder.dimensions.parent_dimension_id` in the DB schema. The frontend `NestedFilter` component reads the hierarchy. No additional JSON template config is needed.

### 3.4 Parameters

Parameters are user-input values used in backend metric calculations (e.g., processing fee %, CPA amount).

```typescript
interface ParameterConfig {
  parameterKey: string;               // Key sent to backend in "parameters" object
  label: string;                      // Display label
  type: "number";                     // Input type (currently only number)
  prefix?: string;                    // "$" prefix for currency
  suffix?: string;                    // "%" suffix for percentage
  default: number;                    // Default value
  step: number;                       // Increment step
  min: number;                        // Minimum value
  max: number;                        // Maximum value
  group?: string;                     // Group key — parameters with same group render in one popover
}
```

**Example — Financial Performance parameters (ungrouped):**
```json
{
  "parameters": [
    { "parameterKey": "processing_fee_pct", "label": "Processing Fees", "type": "number", "suffix": "%", "default": 3.5, "step": 0.1, "min": 0, "max": 20 },
    { "parameterKey": "reserve_pct", "label": "Reserve", "type": "number", "suffix": "%", "default": 5.0, "step": 0.5, "min": 0, "max": 50 },
    { "parameterKey": "cb_fee_amount", "label": "CB Fee", "type": "number", "prefix": "$", "default": 25.00, "step": 1.0, "min": 0, "max": 100 },
    { "parameterKey": "cpa_amount", "label": "CPA", "type": "number", "prefix": "$", "default": 45.00, "step": 1.0, "min": 0, "max": 500 }
  ]
}
```

**Example — Alert Service parameters (grouped):**
```json
{
  "parameters": [
    { "parameterKey": "rdr_amount", "label": "RDR", "type": "number", "prefix": "$", "default": 0, "step": 1, "min": 0, "max": 100, "group": "alert_costs" },
    { "parameterKey": "ethoca_amount", "label": "Ethoca", "type": "number", "prefix": "$", "default": 0, "step": 1, "min": 0, "max": 100, "group": "alert_costs" },
    { "parameterKey": "cdrn_amount", "label": "CDRN", "type": "number", "prefix": "$", "default": 0, "step": 1, "min": 0, "max": 100, "group": "alert_costs" },
    { "parameterKey": "other_alert_amount", "label": "Other", "type": "number", "prefix": "$", "default": 0, "step": 1, "min": 0, "max": 100, "group": "alert_costs" }
  ]
}
```

Parameters with the same `group` value render together in a single popover. The group label is the first parameter's `group` value, title-cased (e.g., `"alert_costs"` → "Alert Costs"). Ungrouped parameters render as individual inputs.
```

### 3.5 Status Filters

Status filter pills are quick-filter controls specific to detail tables (e.g., Transaction Explorer). They apply pre-defined filter conditions.

```typescript
interface StatusFilterConfig {
  key: string;                        // Unique key (e.g., "approved", "declined")
  label: string;                      // Display label
  filter?: Record<string, any>;       // Filter condition applied when selected. Omit for "all".
}
```

**Example — Transaction Explorer:**
```json
{
  "statusFilters": [
    { "key": "all", "label": "All" },
    { "key": "approved", "label": "Approved", "filter": { "is_approved": true } },
    { "key": "declined", "label": "Declined", "filter": { "is_approved": false } },
    { "key": "refunded", "label": "Refunded", "filter": { "is_refund": true } },
    { "key": "chargeback", "label": "Chargeback", "filter": { "is_chargeback": true } }
  ]
}
```

### 3.6 Group By

Group-by is a **table-level** feature, not a report-level slicer panel control. Each aggregated table can independently offer a group-by dimension selector in its table header area.

See [Section 5.3.1 — Aggregated Table](#531-aggregated-table) for the `groupByOptions` and `defaultGroupBy` fields on `AggregatedTableVisual`.

### 3.7 Comparison Date Range

The comparison date range lets users compare current period data against a previous period. When enabled, cards show "+12% vs previous period" and charts can overlay comparison series.

```typescript
// Defined in SlicerPanel:
{
  showComparison: true,
  comparisonPresets: ["previous_period", "previous_week", "previous_month", "custom"]
}
```

**Comparison presets:**

| Preset | Description |
|--------|------------|
| `previous_period` | Same duration, immediately preceding. If current = Jan 15-28, comparison = Jan 1-14. |
| `previous_week` | Same days of the previous week. |
| `previous_month` | Same day range in the previous month. |
| `custom` | User picks a custom comparison date range. |

**How comparison data flows:**

1. User selects a comparison preset (or it defaults to `previous_period`)
2. Frontend calculates the comparison date range from the current date range + preset
3. Frontend sends the comparison range in the batch query: `"comparison": { "from": "...", "to": "..." }`
4. Backend runs each visual's query twice — once for current period, once for comparison period
5. Frontend uses comparison data for:
   - Card trend arrows and percentage change
   - Chart series with `comparison` field set (overlay as dashed line)

**Note:** If `showComparison` is `false` or omitted, card trend comparisons use the `trendComparison` field on each card visual to auto-calculate the comparison period. When `showComparison: true`, the user-selected comparison overrides per-card `trendComparison`.

---

## 4. Section Types

Sections are the layout containers that hold visuals. They define how visuals are arranged on the page.

### 4.1 Base Section Type

```typescript
interface Section {
  id: string;                         // Unique section identifier
  type: "card_row" | "full_width" | "grid" | "split" | "tabs";
  title?: string;                     // Optional section heading
  conditionalVisibility?: ConditionalVisibility;  // Show/hide based on slicer state
  visuals?: Visual[];                 // Visuals in this section (not used by tabs)
  // Type-specific fields below
}

interface ConditionalVisibility {
  slicerKey: string;                  // Which slicer controls visibility
  showWhen: string | string[];        // Show when slicer value matches (string or array of values)
}
```

### 4.2 card_row

A horizontal row of card visuals. Cards auto-distribute evenly across the row.

```typescript
interface CardRowSection extends Section {
  type: "card_row";
  visuals: CardVisual[];              // Typically 3-5 cards
}
```

**Layout behavior:** Cards are rendered in a flex row. On mobile, they stack vertically (2 per row on tablet, 1 per row on phone).

### 4.3 full_width

A single visual occupying the full width of the page.

```typescript
interface FullWidthSection extends Section {
  type: "full_width";
  visuals: Visual[];                  // Typically 1 visual (chart or table)
}
```

### 4.4 grid

A multi-column grid layout. Visuals are placed left-to-right, wrapping to next row.

```typescript
interface GridSection extends Section {
  type: "grid";
  columns: number;                    // Number of columns (typically 2 or 3)
  visuals: Visual[];
}
```

**Layout behavior:** Each visual occupies one grid cell. If there are more visuals than columns, they wrap to the next row.

### 4.5 split

Shorthand for a 2-column grid. Two visuals side by side.

```typescript
interface SplitSection extends Section {
  type: "split";
  visuals: [Visual, Visual];          // Exactly 2 visuals
}
```

Equivalent to `{ type: "grid", columns: 2, visuals: [...] }` but semantically clearer for the common 50/50 layout.

### 4.6 tabs

A tabbed container where each tab holds one visual. Only the active tab's visual is rendered (lazy loading).

```typescript
interface TabsSection extends Section {
  type: "tabs";
  tabs: TabConfig[];                  // Array of tab definitions
}

interface TabConfig {
  label: string;                      // Tab label text
  visual: Visual;                     // The visual rendered when this tab is active
}
```

**Lazy loading:** Only the active tab's visual is included in the batch query. When the user clicks a different tab, the frontend fetches that tab's data (or serves from TanStack Query cache if already fetched).

### 4.7 Conditional Visibility

Any section can be conditionally shown/hidden based on a slicer value.

```json
{
  "id": "cashflow_section",
  "type": "full_width",
  "conditionalVisibility": {
    "slicerKey": "profitability_view",
    "showWhen": "cashflow"
  },
  "visuals": [...]
}
```

When `profitability_view` is set to `"cashflow"`, this section is visible. When set to any other value, it's hidden. `showWhen` can be a string or an array of strings (match any).

### 4.8 Visual Size Constraints

Visual sizes are determined by their containing section type, not by individual visual configuration. This keeps the layout system simple and consistent. The frontend enforces these rules — no JSON configuration needed.

#### Size Rules by Section Type

| Section Type | Visual Size | Frontend Behavior |
|--------------|-------------|-------------------|
| `card_row` | **Auto-fit** | Cards distribute evenly across the row (3-5 cards typical). Responsive: 2 per row on tablet, 1 per row on mobile. |
| `full_width` | **100% width** | Single visual spans the full container width. For tables/charts, height auto-expands based on content. |
| `grid` | **Equal columns** | Each visual takes 1/N width (where N = `columns`). 2-column grid = 50% each, 3-column = 33% each. |
| `split` | **50% each** | Exactly 2 visuals, each 50% width (semantic shorthand for `grid` with `columns: 2`). |
| `tabs` | **100% width** | Active tab's visual spans full width. Only one visual visible at a time. |

#### Visual Type Size Behaviors

| Visual Type | Width | Height |
|-------------|-------|--------|
| `card` | Determined by section | Fixed (approx. 120px). Card rows auto-fit 3-5 cards. |
| `chart` | Determined by section | Fixed (approx. 300px for half-width, 400px for full-width). |
| `table` | Determined by section | Auto-expand based on `pagination.pageSize`. Min ~200px, max ~600px before scroll. |
| `matrix` | Full width preferred | Auto based on rows. Typically used in `full_width` sections. |
| `waterfall` | Full width preferred | Fixed (approx. 400px). Typically used in `full_width` sections. |
| `healthIndicator` | Same as card | Fixed (approx. 80px). Rendered in `card_row` sections. |

#### Responsive Breakpoints

| Breakpoint | Card Row | Grid (2-col) | Grid (3-col) |
|------------|----------|--------------|--------------|
| Desktop (≥1280px) | 3-5 per row | 50% each | 33% each |
| Tablet (768-1279px) | 2 per row | 100% (stack) | 50% each |
| Mobile (<768px) | 1 per row | 100% (stack) | 100% (stack) |

**Important:** The JSON template does **NOT** include size/width/height fields on visuals. Sizes are 100% determined by the section type and frontend CSS. This prevents inconsistent layouts and ensures responsive behavior works correctly.

---

## 5. Visual Types

Every visual in the template follows this base structure:

```typescript
interface BaseVisual {
  visualId: string;                   // Unique identifier (used to key batch query results)
  type: "card" | "chart" | "table" | "matrix" | "waterfall" | "parameterInput" | "healthIndicator";
  title?: string;                     // Visual heading
  source?: string;                    // Override template's primary source for this visual
  interactionMode?: "crossFilter" | "none";  // Click-to-filter behavior. Default: "crossFilter"

  // === State Handling ===
  emptyState?: EmptyStateConfig;      // What to show when query returns 0 rows
  errorState?: ErrorStateConfig;      // What to show when query fails

  // === Navigation ===
  drillThrough?: DrillThroughConfig;  // Click row/segment to navigate to detail report
}

interface EmptyStateConfig {
  message: string;                    // "No data for selected filters"
  icon?: string;                      // Icon name (e.g., "database", "search", "filter")
  actionLabel?: string;               // Optional action button label (e.g., "Adjust Filters")
}

interface ErrorStateConfig {
  retryable: boolean;                 // Show retry button. Default: true
  message?: string;                   // Override default error message
}

interface DrillThroughConfig {
  targetReport: string;               // Report template key to navigate to
  filterMapping: Record<string, string>;  // { "clicked_dimension": "target_filter_dimension" }
  openIn: "same_tab" | "new_tab" | "drawer";  // How to open the target report
}
```

**Empty state example:**
```json
{
  "visualId": "revenue_table",
  "type": "table",
  "emptyState": {
    "message": "No transactions found for the selected filters",
    "icon": "search",
    "actionLabel": "Clear Filters"
  }
}
```

**Error state example:**
```json
{
  "visualId": "revenue_table",
  "type": "table",
  "errorState": {
    "retryable": true,
    "message": "Failed to load transaction data"
  }
}
```

**Drill-through example:**
```json
{
  "visualId": "campaign_table",
  "type": "table",
  "dimension": "campaign_name",
  "drillThrough": {
    "targetReport": "transaction-explorer",
    "filterMapping": { "campaign_name": "campaign_name" },
    "openIn": "same_tab"
  }
}
```

When a user clicks a row in `campaign_table` (e.g., "Summer Sale"), the frontend navigates to `transaction-explorer` with `campaign_name=Summer Sale` pre-applied as a filter.

**Important:** If `source` is omitted, the visual inherits the template-level `source`. This allows cross-source reports (e.g., Command Center pulls from both `order_summary` and `hourly_revenue`).

**Cross-visual filtering:** When `interactionMode` is `"crossFilter"` (the default), clicking a chart segment, table row, or card filters all other visuals on the same page to that selection. For example, clicking "Campaign A" in a table row filters all charts and cards to show only Campaign A data. Set to `"none"` for visuals that should not trigger cross-filtering (e.g., parameter inputs, health indicators).

### 5.1 Card

A summary metric card showing 1-3 metrics with optional trend comparison. Cards support 7 visual variants to match different display needs.

```typescript
interface CardVisual extends BaseVisual {
  type: "card";
  cardVariant?: CardVariant;          // Visual style variant. Default: "metric"
  metrics: CardMetric[];              // 1-3 metrics to display
  showTrend?: boolean;                // Show trend arrow and % change. Default: false
  trendComparison?: "previous_period" | "previous_year";  // Comparison basis. Default: "previous_period"
  invertTrend?: boolean;              // If true, decrease = green (for rates like cancel_rate, cb_rate). Default: false

  // === Variant-specific fields (used by certain cardVariants) ===
  breakdownDimension?: string;        // For "breakdown" variant: dimension to break down by
  breakdownLimit?: number;            // For "breakdown" variant: max items to show. Default: 5
  sparklineMetric?: string;           // For "sparkline" variant: metric to plot over time
  sparklinePeriod?: string;           // For "sparkline" variant: x-axis dimension. Default: "date_day"
  chartType?: "donut" | "bar";        // For "chart" variant: embedded chart type
  chartDimension?: string;            // For "chart" variant: breakdown dimension
  summaryTemplate?: string;           // For "summary" variant: template string with {metricKey} placeholders
  insightType?: InsightType;          // For "insight" variant: type of AI insight to generate
}

type CardVariant =
  | "metric"       // Default — big number with optional trend (current behavior)
  | "breakdown"    // Main metric + breakdown list (e.g., "Revenue" then "by Campaign: A=$5K, B=$3K")
  | "donut"        // Main metric + inline donut chart showing dimension breakdown
  | "sparkline"    // Main metric + inline sparkline showing trend over time
  | "summary"      // Text-based summary using template with metric placeholders
  | "chart"        // Main metric + embedded mini chart (donut or bar)
  | "insight";     // AI-generated insight text based on metric trends

type InsightType = "anomaly" | "trend" | "comparison" | "recommendation";

interface CardMetric {
  metricKey: string;                  // Maps to report_builder.metrics.metric_key
  label?: string;                     // Override display label (default: metric's display_name from DB)
}
```

#### 5.1.1 Card Variants

| Variant | Description | Extra Fields Required |
|---------|-------------|----------------------|
| `metric` | Default. Big number with optional trend arrow. | None |
| `breakdown` | Main metric prominently, followed by a breakdown list showing top N values by a dimension. | `breakdownDimension`, `breakdownLimit` (optional) |
| `donut` | Main metric in center of a small donut chart showing dimension breakdown. | `chartDimension` |
| `sparkline` | Main metric with a small line graph showing trend over time period. | `sparklineMetric` (optional, defaults to first metric), `sparklinePeriod` |
| `summary` | Text-based card using template string. E.g., "You made {revenue} this week, {approval_rate}% approved." | `summaryTemplate` |
| `chart` | Main metric with an embedded mini bar or donut chart. | `chartType`, `chartDimension` |
| `insight` | AI-generated insight about the metric (anomaly detection, trend analysis, etc.). | `insightType` |

**Rendering:** Each card renders based on its `cardVariant`. If omitted, defaults to `"metric"` which shows the metric value prominently, with optional smaller label and trend indicator (green up arrow / red down arrow, with percentage change).

**Example — Basic metric card (default variant):**
```json
{
  "visualId": "revenue_card",
  "type": "card",
  "title": "Revenue",
  "metrics": [{ "metricKey": "revenue", "label": "Total" }],
  "showTrend": true,
  "trendComparison": "previous_period"
}
```

**Example — Breakdown card:**
```json
{
  "visualId": "revenue_by_campaign",
  "type": "card",
  "cardVariant": "breakdown",
  "title": "Revenue",
  "metrics": [{ "metricKey": "revenue" }],
  "breakdownDimension": "campaign_name",
  "breakdownLimit": 5,
  "showTrend": true
}
```

**Example — Donut card:**
```json
{
  "visualId": "revenue_donut",
  "type": "card",
  "cardVariant": "donut",
  "title": "Revenue Distribution",
  "metrics": [{ "metricKey": "revenue" }],
  "chartDimension": "sales_type"
}
```

**Example — Sparkline card:**
```json
{
  "visualId": "revenue_sparkline",
  "type": "card",
  "cardVariant": "sparkline",
  "title": "Revenue Trend",
  "metrics": [{ "metricKey": "revenue" }],
  "sparklinePeriod": "date_day"
}
```

**Example — Summary card:**
```json
{
  "visualId": "weekly_summary",
  "type": "card",
  "cardVariant": "summary",
  "title": "Weekly Summary",
  "metrics": [
    { "metricKey": "revenue" },
    { "metricKey": "approval_rate" },
    { "metricKey": "initials" }
  ],
  "summaryTemplate": "You made {revenue} this week with {initials} new customers at {approval_rate} approval."
}
```

**Example — Chart card:**
```json
{
  "visualId": "sales_mix_card",
  "type": "card",
  "cardVariant": "chart",
  "title": "Sales Mix",
  "metrics": [{ "metricKey": "revenue" }],
  "chartType": "bar",
  "chartDimension": "sales_type"
}
```

**Example — Insight card:**
```json
{
  "visualId": "revenue_insight",
  "type": "card",
  "cardVariant": "insight",
  "title": "Revenue Insights",
  "metrics": [{ "metricKey": "revenue" }],
  "insightType": "anomaly"
}
```

### 5.2 Chart

Charts support two data binding modes depending on the chart type.

```typescript
interface ChartVisual extends BaseVisual {
  type: "chart";
  chartType: "line" | "area" | "bar" | "stacked_bar" | "combo" | "pie" | "donut";

  // === Series Mode (line, area, bar, combo) ===
  // Each series is a separate metric plotted over an x-axis dimension
  xAxis?: AxisConfig;                 // X-axis dimension
  series?: SeriesConfig[];            // Array of metric series

  // === Dimension+Metric Mode (stacked_bar, pie, donut) ===
  // One metric broken down by a dimension (each dimension value = one segment/slice)
  dimension?: string;                 // Dimension key for breakdown
  metric?: string;                    // Single metric key for values

  // === Common ===
  stacked?: boolean;                  // Stack series (for area/bar). Default: false
  showLegend?: boolean;               // Show legend. Default: false
  showDataLabels?: boolean;           // Show values on chart. Default: false
  yAxisFormat?: string;               // Y-axis format: "number", "currency", "percentage"
  colors?: string[];                  // Custom color palette
  visualTimeGranularity?: VisualTimeGranularity;  // Chart-level time toggle (overrides report slicer)
}

interface VisualTimeGranularity {
  enabled: boolean;                   // Show time granularity toggle on this visual
  position: "top-right" | "top-left"; // Position of the toggle pills
  displayAs: "pills" | "select";      // Display style
  options: ("day" | "week" | "month")[];  // Available granularity options
  default: "day" | "week" | "month";  // Default selection
}

interface AxisConfig {
  dimensionKey: string;               // Dimension key (supports {time_granularity} placeholder)
  label?: string;                     // Axis label
}

interface SeriesConfig {
  metricKey: string;                  // Maps to report_builder.metrics.metric_key
  label?: string;                     // Legend label
  color?: string;                     // Hex color (e.g., "#4F46E5")
  style?: "solid" | "dashed";        // Line style. Default: "solid"
  chartType?: "line" | "bar";        // Override for combo charts
  comparison?: ComparisonPreset;      // Auto-generate a comparison series for this metric
  comparisonLabel?: string;           // Label for the comparison series (default: "Previous Period")
  comparisonColor?: string;           // Color for comparison series (default: "#94A3B8")
  comparisonStyle?: "solid" | "dashed"; // Style for comparison series. Default: "dashed"
}
```

**Series Mode** — used by `line`, `area`, `bar`, `combo`:
```json
{
  "visualId": "revenue_trend",
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
```

**Dimension+Metric Mode** — used by `stacked_bar`, `pie`, `donut`:
```json
{
  "visualId": "product_chart",
  "type": "chart",
  "chartType": "stacked_bar",
  "title": "Revenue by Product (Weekly)",
  "xAxis": { "dimensionKey": "date_week" },
  "dimension": "product_name",
  "metric": "revenue",
  "showLegend": true
}
```

**Combo chart** — mix line and bar in one chart:
```json
{
  "visualId": "combo_example",
  "type": "chart",
  "chartType": "combo",
  "xAxis": { "dimensionKey": "date_day" },
  "series": [
    { "metricKey": "revenue", "label": "Revenue", "chartType": "bar", "color": "#4F46E5" },
    { "metricKey": "approval_rate", "label": "Approval %", "chartType": "line", "color": "#10B981" }
  ],
  "showLegend": true
}
```

### 5.3 Table

Tables support two modes: **aggregated** (GROUP BY dimension, with metric columns) and **detail** (raw row-level data with explicit columns).

#### 5.3.1 Aggregated Table

Used by most reports. Groups data by a dimension and shows metric columns.

```typescript
interface AggregatedTableVisual extends BaseVisual {
  type: "table";
  isDetailTable?: false;              // Default: false (aggregated mode)
  dimension: string;                  // GROUP BY dimension key — fixed key or "{groupBy}" placeholder
  metrics: TableMetric[];             // Metric columns

  // === Group By (optional — enables dimension selector in table header) ===
  groupByOptions?: string[];          // Dimension keys user can pick from
  defaultGroupBy?: string[];          // Initially selected group-by dimensions (must be subset of groupByOptions)
  allowMultiLevelGroupBy?: boolean;   // Allow GROUP BY + THEN BY multi-level grouping. Default: false
  maxGroupByLevels?: number;          // Maximum grouping levels when multi-level is enabled. Default: 2
  groupByPosition?: "slicer_panel" | "table_header";  // Where to render the group-by selector. Default: "table_header"

  // === Display ===
  sortBy?: string;                    // Default sort column (metric or dimension key)
  sortDirection?: "asc" | "desc";     // Default: "desc"
  pagination?: PaginationConfig;
  showTotals?: boolean;               // Show totals row. Default: false
  allowColumnToggle?: boolean;        // Allow user to show/hide columns. Default: false
  defaultVisibleColumns?: string[];   // Metric keys visible by default (omit = all visible)
  columnPresets?: ColumnPreset[];     // Named column visibility presets
  exportable?: boolean;               // Show export button on table. Default: false
  metricConfig?: Record<string, MetricDisplayConfig>;  // Per-metric rendering overrides

  // === Expandable Rows ===
  isExpandable?: boolean;             // Rows can expand to show child dimension. Default: false
  expandDimension?: string;           // Child dimension to show when row expands (e.g., "sub_affiliate_id")
}

interface ColumnPreset {
  label: string;                      // Preset name (e.g., "Approvals", "Revenue", "Full View")
  columns: string[];                  // Metric keys to show when this preset is selected
}

interface TableMetric {
  metricKey: string;
  label?: string;                     // Override display label
  isSparkline?: boolean;              // Render as inline sparkline chart instead of number
  sparklineMetric?: string;           // Metric to plot in sparkline (if different from metricKey)
  sparklinePeriod?: string;           // Time dimension for sparkline x-axis. Default: "date_day"
}

interface MetricDisplayConfig {
  showBar?: boolean;                  // Show inline bar chart in cell (for rates)
  conditionalColor?: ConditionalColor;  // Color cell based on value
}

interface ConditionalColor {
  thresholds: { value: number; color: string }[];  // Ordered thresholds
}

interface PaginationConfig {
  pageSize: number;                   // Rows per page (10, 25, 50)
}
```

**Example — aggregated table with group-by selector:**
```json
{
  "visualId": "revenue_table",
  "type": "table",
  "title": "Revenue by {groupBy}",
  "dimension": "{groupBy}",
  "groupByOptions": ["campaign_name", "product_name", "gateway_alias", "sales_type", "card_brand"],
  "defaultGroupBy": ["campaign_name"],
  "metrics": [
    { "metricKey": "attempts" },
    { "metricKey": "approvals" },
    { "metricKey": "approval_rate" },
    { "metricKey": "revenue" },
    { "metricKey": "aov" }
  ],
  "sortBy": "revenue",
  "sortDirection": "desc",
  "pagination": { "pageSize": 25 },
  "showTotals": true,
  "allowColumnToggle": true
}
```

When `groupByOptions` is present, the table renders a multi-select dimension selector in its header. Users can select one or more dimensions to group by simultaneously (e.g., `["campaign_name", "product_name"]` → rows grouped by both Campaign and Product).

The `{groupBy}` placeholder in `dimension` and `title` resolves to the user-selected (or default) group-by dimension keys. When multiple are selected, `dimension` becomes an array and the backend adds multiple GROUP BY columns. Tables without `groupByOptions` use a fixed `dimension` value.

**Example — fixed dimension table (no group-by selector):**
```json
{
  "visualId": "daily_summary",
  "type": "table",
  "title": "Daily Summary",
  "dimension": "date_day",
  "metrics": [
    { "metricKey": "revenue" },
    { "metricKey": "initials" },
    { "metricKey": "rebills" }
  ],
  "sortBy": "date_day",
  "sortDirection": "desc",
  "showTotals": true
}
```

#### 5.3.2 Detail Table

Used by Transaction Explorer. Shows raw row-level data with explicit column definitions. No GROUP BY.

```typescript
interface DetailTableVisual extends BaseVisual {
  type: "table";
  isDetailTable: true;
  columns: DetailColumn[];            // Explicit column definitions
  sortBy?: string;                    // Default sort column key
  sortDirection?: "asc" | "desc";
  pagination?: PaginationConfig;
  allowColumnToggle?: boolean;
  exportable?: boolean;
}

interface DetailColumn {
  key: string;                        // Column key (raw column from data source)
  label: string;                      // Display header
  format?: "text" | "currency" | "number" | "percentage" | "date" | "status_badge";  // Cell format
  width?: number;                     // Column width in pixels
}
```

**Example — Transaction Explorer detail table:**
```json
{
  "visualId": "transaction_table",
  "type": "table",
  "title": "Transactions",
  "isDetailTable": true,
  "columns": [
    { "key": "date_of_sale", "label": "Date", "format": "date" },
    { "key": "order_id", "label": "Order ID" },
    { "key": "bill_email", "label": "Email" },
    { "key": "bill_first", "label": "First Name" },
    { "key": "bill_last", "label": "Last Name" },
    { "key": "order_total", "label": "Amount", "format": "currency" },
    { "key": "product_name", "label": "Product" },
    { "key": "campaign_name", "label": "Campaign" },
    { "key": "sales_type", "label": "Type" },
    { "key": "is_approved", "label": "Status", "format": "status_badge" },
    { "key": "gateway_alias", "label": "Gateway" },
    { "key": "billing_cycle", "label": "Cycle" }
  ],
  "sortBy": "date_of_sale",
  "sortDirection": "desc",
  "pagination": { "pageSize": 50 },
  "allowColumnToggle": true,
  "exportable": true
}
```

### 5.4 Matrix

Cohort heatmap for retention/LTV reports. Rows are cohort periods, columns are billing cycles, cells are metric values displayed as percentages.

```typescript
interface MatrixVisual extends BaseVisual {
  type: "matrix";
  source?: string;                    // Typically "cohort_summary"
  rowDimension: string;               // Row grouping dimension (e.g., "date_month")
  columnDimension: string;            // Column grouping dimension (e.g., "billing_cycle")
  valueMetric: string;                // Metric for cell values (e.g., "approvals")
  displayAs: "percentage" | "count" | "currency";  // How to show cell values
  baseMetric?: string;                // Denominator for percentage display (e.g., "initials")
  maxColumns?: number;                // Limit column count (e.g., 12 billing cycles)
  heatmapColors?: HeatmapColors;     // Color scale for cells
}

interface HeatmapColors {
  low: string;                        // Color for low values (e.g., "#FEE2E2" red tint)
  mid: string;                        // Color for mid values (e.g., "#FEF3C7" yellow tint)
  high: string;                       // Color for high values (e.g., "#D1FAE5" green tint)
}
```

**How the backend handles matrix queries:**

The backend returns flat rows: `{ cohort_month, billing_cycle, value, base }`. The frontend pivots this into a matrix grid. The backend query:

```sql
SELECT
  DATE_TRUNC('month', date) AS cohort_month,
  billing_cycle,
  SUM(approvals) AS value,
  SUM(CASE WHEN billing_cycle = 0 THEN approvals END)
    OVER (PARTITION BY DATE_TRUNC('month', date)) AS base
FROM reporting.cohort_summary_{clientId}
WHERE date >= '2025-07-01'
GROUP BY DATE_TRUNC('month', date), billing_cycle
ORDER BY cohort_month, billing_cycle
```

**Example:**
```json
{
  "visualId": "cohort_heatmap",
  "type": "matrix",
  "title": "Retention Cohort Heatmap",
  "source": "cohort_summary",
  "rowDimension": "date_month",
  "columnDimension": "billing_cycle",
  "valueMetric": "approvals",
  "displayAs": "percentage",
  "baseMetric": "initials",
  "maxColumns": 12,
  "heatmapColors": { "low": "#FEE2E2", "mid": "#FEF3C7", "high": "#D1FAE5" }
}
```

### 5.5 Waterfall

P&L waterfall chart showing revenue flowing through deductions to gross profit.

```typescript
interface WaterfallVisual extends BaseVisual {
  type: "waterfall";
  steps: WaterfallStep[];
  colors?: {
    increase: string;                 // Color for positive/start values
    decrease: string;                 // Color for deductions
    total: string;                    // Color for the final total
  };
}

interface WaterfallStep {
  metricKey: string;                  // Metric for this step's value
  label: string;                      // Display label on chart
  type: "start" | "add" | "subtract" | "end";  // Step behavior
}
```

**Step types:**
- `start` — First bar, shows the starting value (e.g., Gross Revenue)
- `add` — Adds to the running total (green bar going up)
- `subtract` — Subtracts from the running total (red bar going down)
- `end` — Final bar, shows the result (e.g., Gross Profit). Automatically calculated as start + adds - subtracts.

**Example — P&L Waterfall:**
```json
{
  "visualId": "cashflow_waterfall",
  "type": "waterfall",
  "title": "P&L Waterfall",
  "steps": [
    { "metricKey": "revenue", "label": "Gross Revenue", "type": "start" },
    { "metricKey": "refund_dollar", "label": "Refunds", "type": "subtract" },
    { "metricKey": "cb_dollar", "label": "Chargebacks", "type": "subtract" },
    { "metricKey": "processing_fees", "label": "Processing", "type": "subtract" },
    { "metricKey": "cb_fees", "label": "CB Fees", "type": "subtract" },
    { "metricKey": "cpa_cost", "label": "CPA Cost", "type": "subtract" },
    { "metricKey": "reserve", "label": "Reserve", "type": "subtract" },
    { "metricKey": "gross_profit", "label": "Gross Profit", "type": "end" }
  ]
}
```

### 5.6 ParameterInput

Not a visual in the traditional sense — renders as an input field inside a card layout. Used for Financial Performance (processing fee %, reserve %, etc.). Defined in the `slicerPanel.parameters` array, not as a section visual.

See [Section 3.4 — Parameters](#34-parameters).

### 5.7 HealthIndicator

A status badge showing green/yellow/red based on a metric value against thresholds.

```typescript
interface HealthIndicatorVisual extends BaseVisual {
  type: "healthIndicator";
  metricKey: string;                  // The metric to evaluate
  thresholds: {
    green: { min: number; max: number };   // "Healthy" range
    yellow: { min: number; max: number };  // "Warning" range
    red: { min: number; max: number };     // "Critical" range
  };
  showValue?: boolean;                // Show the numeric value alongside the badge. Default: true
}
```

**Example:**
```json
{
  "visualId": "cb_health",
  "type": "healthIndicator",
  "title": "CB Rate Health",
  "metricKey": "cb_rate",
  "thresholds": {
    "green": { "min": 0, "max": 0.8 },
    "yellow": { "min": 0.8, "max": 1.0 },
    "red": { "min": 1.0, "max": 100 }
  }
}
```

---

## 6. Cross-Cutting Concerns

### 6.1 Cross-Source Visuals

Individual visuals can override the template's primary `source` by specifying their own `source` field.

```json
{
  "source": "order_summary",
  "sections": [
    {
      "id": "overview",
      "type": "card_row",
      "visuals": [
        { "visualId": "revenue_card", "type": "card", "metrics": [{ "metricKey": "revenue" }] }
      ]
    },
    {
      "id": "hourly",
      "type": "full_width",
      "visuals": [
        {
          "visualId": "hourly_chart",
          "type": "chart",
          "source": "hourly_revenue",
          "chartType": "area",
          "xAxis": { "dimensionKey": "date_hour" },
          "series": [
            { "metricKey": "today_revenue", "label": "Today" },
            { "metricKey": "avg_7d_revenue", "label": "7-Day Avg", "style": "dashed" }
          ]
        }
      ]
    }
  ]
}
```

The batch query handles multiple sources — each query specifies its own source, and the backend runs them all in parallel.

### 6.2 Dynamic Placeholders

Two placeholders are resolved by the frontend at render time:

| Placeholder | Resolved From | Used In |
|------------|--------------|---------|
| `{groupBy}` | Table's own `defaultGroupBy` or current user selection from that table's group-by selector | Table `dimension`, `title` fields |
| `{time_granularity}` | Current value of `time_granularity` slicer (day→date_day, week→date_week, month→date_month) | `xAxis.dimensionKey` |

**Resolution rules:**
- `{groupBy}` → replaced with the currently selected groupBy dimension key from the table's own `groupByOptions` (e.g., `"campaign_name"`). Each table manages its own group-by state independently.
- `{time_granularity}` → mapped to a date dimension: `day` → `"date_day"`, `week` → `"date_week"`, `month` → `"date_month"`
- Placeholders in `title` fields are resolved to the display name: `{groupBy}` with `campaign_name` selected → `"Campaign"`

### 6.3 Auto-Refresh

Set `defaults.refreshInterval` to enable automatic data refresh:

```json
{
  "defaults": {
    "dateRange": "last_7_days",
    "refreshInterval": 300
  }
}
```

The frontend uses TanStack Query's `refetchInterval` option. When `refreshInterval` is set:
- Data refetches every N seconds in the background
- The UI does NOT flash/reload — new data smoothly replaces old data
- Refetch pauses when the browser tab is not active

Used by: Business Command Center (300s), Real-Time Pulse (300s).

### 6.4 Source Config

Source-level configuration overrides default behaviors.

#### Server-Side Pagination / Sort / Search

Used by Transaction Explorer (`order_details` source):

```json
{
  "source": "order_details",
  "sourceConfig": {
    "serverSidePagination": true,
    "serverSideSort": true,
    "serverSideSearch": true,
    "searchColumns": ["order_id", "bill_email", "bill_first", "bill_last", "transaction_id"]
  }
}
```

When `serverSidePagination: true`:
- Frontend sends `pagination: { page: 0, size: 50 }` in the query
- Backend adds `LIMIT` and `OFFSET` to SQL
- Backend returns `{ rows: [...], total: 12345 }` (total for pagination controls)

When `serverSideSort: true`:
- Frontend sends `sort: { field: "date_of_sale", dir: "desc" }` in the query
- Backend adds `ORDER BY` to SQL

When `serverSideSearch: true`:
- Frontend sends `search: { term: "john@email.com", columns: ["order_id", "bill_email", ...] }` in the query
- Backend adds `WHERE (col1 ILIKE $1 OR col2 ILIKE $1 ...)` with parameterized `%term%`

#### Month Selector

Used by MID Performance reports where data is keyed by `month_year` instead of a date range:

```json
{
  "source": "mid_summary",
  "sourceConfig": {
    "dateFilterType": "month_selector",
    "dateColumn": "month_year",
    "dateFormat": "MMM YYYY"
  }
}
```

When `dateFilterType: "month_selector"`:
- Frontend shows a month picker instead of a date range picker
- Query uses `WHERE month_year = 'Jan 2026'` instead of `WHERE date BETWEEN ... AND ...`

### 6.5 Inverted Trends

For metrics where a decrease is positive (cancel rate, CB rate, refund rate), set `invertTrend: true` on card visuals:

```json
{
  "visualId": "cancel_rate_card",
  "type": "card",
  "title": "Cancel Rate",
  "metrics": [{ "metricKey": "cancel_rate" }],
  "showTrend": true,
  "invertTrend": true
}
```

When `invertTrend: true`:
- Decrease → green arrow (good)
- Increase → red arrow (bad)

Default behavior (invertTrend: false or omitted):
- Increase → green arrow (good)
- Decrease → red arrow (bad)

### 6.6 Fixed Date Reports

Reports like Real-Time Pulse are always "today" — the date picker is hidden:

```json
{
  "defaults": { "dateRange": "today", "refreshInterval": 300 },
  "slicerPanel": { "showDateRange": false }
}
```

### 6.7 Cross-Visual Filtering

When a user clicks on a chart segment (e.g., a bar for "Campaign A") or a table row, all other visuals on the same page filter to that selection. This replicates Power BI's `drillFilterOtherVisuals` behavior.

**How it works:**
1. User clicks on a data point in a visual (e.g., "Campaign A" in a bar chart)
2. Frontend adds a temporary dimension filter: `{ "campaign_name": { "operator": "in", "values": ["Campaign A"] } }`
3. All other visuals on the page re-query with this additional filter
4. A "Clear selection" button appears to remove the cross-filter

**Controlled by `interactionMode` on BaseVisual:**

| Value | Behavior |
|-------|---------|
| `"crossFilter"` (default) | Clicking this visual filters other visuals. Other visuals react to cross-filter selections. |
| `"none"` | This visual does not emit cross-filter events and does not react to them. Use for parameter inputs, health indicators, or standalone visuals. |

```json
{
  "visualId": "revenue_card",
  "type": "card",
  "interactionMode": "none",
  "metrics": [{ "metricKey": "revenue" }]
}
```

### 6.8 Comparison Overlay on Charts

Chart series can automatically include a comparison period overlay using the `comparison` field on `SeriesConfig`:

```json
{
  "visualId": "revenue_trend",
  "type": "chart",
  "chartType": "area",
  "xAxis": { "dimensionKey": "date_day" },
  "series": [
    {
      "metricKey": "revenue",
      "label": "Revenue",
      "color": "#4F46E5",
      "comparison": "previous_period",
      "comparisonLabel": "Previous Period",
      "comparisonColor": "#94A3B8",
      "comparisonStyle": "dashed"
    }
  ]
}
```

When `comparison` is set on a series, the frontend auto-generates a second series using data from the comparison date range. This requires either `slicerPanel.showComparison: true` (user-selected comparison) or uses the preset value to auto-calculate the comparison range.

### 6.9 URL Deep Linking

URL deep linking allows users to share filtered report views with colleagues. The entire report state is encoded in the URL, so recipients see the exact same filters, toggles, and table configurations.

#### URL Format

```
/reports/:templateKey?state=<base64_encoded_json>
```

#### State Object

```typescript
interface ReportState {
  // Filter state
  filters: {
    dateRange: { from: string; to: string };        // ISO date strings
    comparison?: { from: string; to: string };      // Comparison date range if active
    dimensions: Record<string, FilterValue>;        // Dimension filters
  };

  // Slicer state
  toggles: Record<string, string>;                  // { "approval_mode": "organic", ... }

  // Parameter values
  parameters?: Record<string, number>;              // { "processing_fee_pct": 3.5, ... }

  // Per-visual state
  tableState?: Record<string, TableState>;          // Keyed by visualId
}

interface FilterValue {
  operator: "in" | "not_in";
  values: (string | number | boolean)[];
}

interface TableState {
  groupBy?: string[];                               // Current group-by selection
  sortBy?: string;                                  // Current sort column
  sortDirection?: "asc" | "desc";
  page?: number;                                    // Current page (0-indexed)
  visibleColumns?: string[];                        // Currently visible columns
}
```

#### Example

**Readable state object:**
```json
{
  "filters": {
    "dateRange": { "from": "2026-01-01", "to": "2026-01-28" },
    "dimensions": {
      "campaign_name": { "operator": "in", "values": ["Summer Sale"] },
      "sales_type": { "operator": "in", "values": ["Initials", "Rebills"] }
    }
  },
  "toggles": { "approval_mode": "organic", "time_granularity": "day" },
  "parameters": { "processing_fee_pct": 4.0 },
  "tableState": {
    "revenue_breakdown_table": {
      "groupBy": ["campaign_name", "product_name"],
      "sortBy": "revenue",
      "sortDirection": "desc"
    }
  }
}
```

**Encoded URL:**
```
/reports/revenue-analytics?state=eyJmaWx0ZXJzIjp7ImRhdGVSYW5nZSI6eyJmcm9tIjoiMjAyNi0wMS0wMSIsInRvIjoiMjAyNi0wMS0yOCJ9LCJkaW1lbnNpb25zIjp7ImNhbXBhaWduX25hbWUiOnsib3BlcmF0b3IiOiJpbiIsInZhbHVlcyI6WyJTdW1tZXIgU2FsZSJdfSwic2FsZXNfdHlwZSI6eyJvcGVyYXRvciI6ImluIiwidmFsdWVzIjpbIkluaXRpYWxzIiwiUmViaWxscyJdfX19LCJ0b2dnbGVzIjp7ImFwcHJvdmFsX21vZGUiOiJvcmdhbmljIiwidGltZV9ncmFudWxhcml0eSI6ImRheSJ9LCJwYXJhbWV0ZXJzIjp7InByb2Nlc3NpbmdfZmVlX3BjdCI6NC4wfX0
```

#### Frontend Implementation

**Encoding (when user clicks "Copy Link"):**
```typescript
const state: ReportState = buildCurrentState();
const encoded = btoa(JSON.stringify(state));
const url = `${window.location.origin}/reports/${templateKey}?state=${encoded}`;
navigator.clipboard.writeText(url);
```

**Decoding (on page load):**
```typescript
const params = new URLSearchParams(window.location.search);
const stateParam = params.get('state');
if (stateParam) {
  try {
    const state: ReportState = JSON.parse(atob(stateParam));
    applyReportState(state);
  } catch (e) {
    console.warn('Invalid state parameter, using defaults');
  }
}
```

#### State Priority

When loading a report, state is applied in this order (later overrides earlier):

1. **Template defaults** — `defaults.dateRange`, slicer default values
2. **User preferences** — Stored in `user_preferences` table (Phase 2)
3. **Bookmark** — If user has a bookmark for this report (Phase 2)
4. **URL state** — Always wins if present

#### UI Integration

Add a "Share" or "Copy Link" button to the slicer panel (next to Export):

```json
{
  "slicerPanel": {
    "showExport": true,
    "showShare": true
  }
}
```

When clicked, the frontend serializes current state to URL and copies to clipboard. A toast notification confirms: "Link copied to clipboard".

---

## 7. Complete Stock Report Examples

Four representative templates covering all special behaviors. The remaining 7 templates follow the same patterns.

### 7.1 Business Command Center

**Features demonstrated:** Auto-refresh, cross-source visuals (order_summary + hourly_revenue), trend comparison.

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": {
    "dateRange": "last_7_days",
    "refreshInterval": 300
  },
  "slicerPanel": {
    "showDateRange": true,
    "slicers": [],
    "filters": [],
    "showExport": true
  },
  "sections": [
    {
      "id": "summary_cards",
      "type": "card_row",
      "visuals": [
        {
          "visualId": "revenue_card",
          "type": "card",
          "title": "Revenue",
          "metrics": [{ "metricKey": "revenue", "label": "Total" }],
          "showTrend": true,
          "trendComparison": "previous_period"
        },
        {
          "visualId": "new_customers_card",
          "type": "card",
          "title": "New Customers",
          "metrics": [{ "metricKey": "initials", "label": "Count" }],
          "showTrend": true,
          "trendComparison": "previous_period"
        },
        {
          "visualId": "active_subs_card",
          "type": "card",
          "title": "Active Subscribers",
          "metrics": [{ "metricKey": "approvals", "label": "Count" }],
          "showTrend": true,
          "trendComparison": "previous_period"
        },
        {
          "visualId": "churn_card",
          "type": "card",
          "title": "Cancel Rate",
          "metrics": [{ "metricKey": "cancel_rate", "label": "Rate" }],
          "showTrend": true,
          "trendComparison": "previous_period",
          "invertTrend": true
        }
      ]
    },
    {
      "id": "hourly_chart",
      "type": "full_width",
      "visuals": [
        {
          "visualId": "hourly_revenue",
          "type": "chart",
          "source": "hourly_revenue",
          "chartType": "area",
          "title": "Revenue Today vs 7-Day Avg",
          "xAxis": { "dimensionKey": "date_hour" },
          "series": [
            { "metricKey": "today_revenue", "label": "Today", "color": "#4F46E5" },
            { "metricKey": "avg_7d_revenue", "label": "7-Day Avg", "color": "#94A3B8", "style": "dashed" }
          ],
          "showLegend": true
        }
      ]
    },
    {
      "id": "daily_table",
      "type": "full_width",
      "visuals": [
        {
          "visualId": "daily_summary",
          "type": "table",
          "title": "Daily Summary",
          "dimension": "date_day",
          "metrics": [
            { "metricKey": "revenue" },
            { "metricKey": "initials" },
            { "metricKey": "rebills" },
            { "metricKey": "approval_rate" },
            { "metricKey": "cancel_rate" },
            { "metricKey": "cb_count" },
            { "metricKey": "refund_count" }
          ],
          "sortBy": "date_day",
          "sortDirection": "desc",
          "pagination": { "pageSize": 10 },
          "showTotals": true
        }
      ]
    }
  ]
}
```

### 7.2 Financial Performance

**Features demonstrated:** Parameter inputs, waterfall chart, conditional visibility (profitability_view slicer), table-level group-by.

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": { "dateRange": "last_30_days" },
  "slicerPanel": {
    "showDateRange": true,
    "slicers": [
      { "slicerKey": "profitability_view", "position": "main", "displayAs": "pills" }
    ],
    "parameters": [
      { "parameterKey": "processing_fee_pct", "label": "Processing Fees", "type": "number", "suffix": "%", "default": 3.5, "step": 0.1, "min": 0, "max": 20 },
      { "parameterKey": "reserve_pct", "label": "Reserve", "type": "number", "suffix": "%", "default": 5.0, "step": 0.5, "min": 0, "max": 50 },
      { "parameterKey": "cb_fee_amount", "label": "CB Fee", "type": "number", "prefix": "$", "default": 25.00, "step": 1.0, "min": 0, "max": 100 },
      { "parameterKey": "cpa_amount", "label": "CPA", "type": "number", "prefix": "$", "default": 45.00, "step": 1.0, "min": 0, "max": 500 }
    ],
    "filters": [
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "more" }
    ],
    "showExport": true
  },
  "sections": [
    {
      "id": "waterfall",
      "type": "full_width",
      "conditionalVisibility": { "slicerKey": "profitability_view", "showWhen": "cashflow" },
      "visuals": [
        {
          "visualId": "cashflow_waterfall",
          "type": "waterfall",
          "title": "P&L Waterfall",
          "steps": [
            { "metricKey": "revenue", "label": "Gross Revenue", "type": "start" },
            { "metricKey": "refund_dollar", "label": "Refunds", "type": "subtract" },
            { "metricKey": "cb_dollar", "label": "Chargebacks", "type": "subtract" },
            { "metricKey": "processing_fees", "label": "Processing", "type": "subtract" },
            { "metricKey": "cb_fees", "label": "CB Fees", "type": "subtract" },
            { "metricKey": "cpa_cost", "label": "CPA Cost", "type": "subtract" },
            { "metricKey": "reserve", "label": "Reserve", "type": "subtract" },
            { "metricKey": "gross_profit", "label": "Gross Profit", "type": "end" }
          ]
        }
      ]
    },
    {
      "id": "profit_cards",
      "type": "card_row",
      "visuals": [
        { "visualId": "revenue_card", "type": "card", "title": "Gross Revenue", "metrics": [{ "metricKey": "revenue" }], "showTrend": true },
        { "visualId": "net_revenue_card", "type": "card", "title": "Net Revenue", "metrics": [{ "metricKey": "net_revenue" }], "showTrend": true },
        { "visualId": "profit_card", "type": "card", "title": "Gross Profit", "metrics": [{ "metricKey": "gross_profit" }], "showTrend": true },
        { "visualId": "margin_card", "type": "card", "title": "Gross Margin", "metrics": [{ "metricKey": "gross_margin" }], "showTrend": true }
      ]
    },
    {
      "id": "profit_table",
      "type": "full_width",
      "visuals": [
        {
          "visualId": "profit_by_product",
          "type": "table",
          "title": "Profitability by {groupBy}",
          "dimension": "{groupBy}",
          "groupByOptions": ["product_name", "campaign_name"],
          "defaultGroupBy": ["product_name"],
          "metrics": [
            { "metricKey": "revenue" },
            { "metricKey": "refund_dollar" },
            { "metricKey": "cb_dollar" },
            { "metricKey": "processing_fees" },
            { "metricKey": "cb_fees" },
            { "metricKey": "cpa_cost" },
            { "metricKey": "gross_profit" },
            { "metricKey": "gross_margin" }
          ],
          "sortBy": "gross_profit",
          "sortDirection": "desc",
          "showTotals": true
        }
      ]
    }
  ]
}
```

### 7.3 Transaction Explorer

**Features demonstrated:** Detail table (isDetailTable), server-side pagination/sort/search, status filter pills, search input, no aggregation.

```json
{
  "version": "1.0",
  "source": "order_details",
  "sourceConfig": {
    "serverSidePagination": true,
    "serverSideSort": true,
    "serverSideSearch": true,
    "searchColumns": ["order_id", "bill_email", "bill_first", "bill_last", "transaction_id"]
  },
  "defaults": { "dateRange": "last_7_days" },
  "slicerPanel": {
    "showDateRange": true,
    "showSearch": true,
    "searchPlaceholder": "Search by ID, email, name...",
    "slicers": [],
    "filters": [
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "main" },
      { "dimensionKey": "campaign_name", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "gateway_alias", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "sales_type", "displayAs": "pills", "position": "main" }
    ],
    "statusFilters": [
      { "key": "all", "label": "All" },
      { "key": "approved", "label": "Approved", "filter": { "is_approved": true } },
      { "key": "declined", "label": "Declined", "filter": { "is_approved": false } },
      { "key": "refunded", "label": "Refunded", "filter": { "is_refund": true } },
      { "key": "chargeback", "label": "Chargeback", "filter": { "is_chargeback": true } }
    ],
    "showExport": true
  },
  "sections": [
    {
      "id": "transactions",
      "type": "full_width",
      "visuals": [
        {
          "visualId": "transaction_table",
          "type": "table",
          "title": "Transactions",
          "isDetailTable": true,
          "columns": [
            { "key": "date_of_sale", "label": "Date", "format": "date" },
            { "key": "order_id", "label": "Order ID" },
            { "key": "bill_email", "label": "Email" },
            { "key": "bill_first", "label": "First Name" },
            { "key": "bill_last", "label": "Last Name" },
            { "key": "order_total", "label": "Amount", "format": "currency" },
            { "key": "product_name", "label": "Product" },
            { "key": "campaign_name", "label": "Campaign" },
            { "key": "sales_type", "label": "Type" },
            { "key": "is_approved", "label": "Status", "format": "status_badge" },
            { "key": "gateway_alias", "label": "Gateway" },
            { "key": "billing_cycle", "label": "Cycle" }
          ],
          "sortBy": "date_of_sale",
          "sortDirection": "desc",
          "pagination": { "pageSize": 50 },
          "allowColumnToggle": true,
          "exportable": true
        }
      ]
    }
  ]
}
```

### 7.4 Customer Lifecycle

**Features demonstrated:** Matrix (cohort heatmap), cross-source (cohort_summary + order_summary), retention_base slicer, LTV cards.

```json
{
  "version": "1.0",
  "source": "cohort_summary",
  "defaults": { "dateRange": "last_6_months" },
  "slicerPanel": {
    "showDateRange": true,
    "slicers": [
      { "slicerKey": "retention_base", "position": "main", "displayAs": "pills" }
    ],
    "filters": [
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "campaign_name", "displayAs": "multiselect", "position": "more" }
    ],
    "showExport": true
  },
  "sections": [
    {
      "id": "ltv_cards",
      "type": "card_row",
      "visuals": [
        { "visualId": "ltv_30", "type": "card", "title": "30-Day LTV", "metrics": [{ "metricKey": "ltv_30d" }], "showTrend": true },
        { "visualId": "ltv_90", "type": "card", "title": "90-Day LTV", "metrics": [{ "metricKey": "ltv_90d" }], "showTrend": true },
        { "visualId": "ltv_180", "type": "card", "title": "180-Day LTV", "metrics": [{ "metricKey": "ltv_180d" }], "showTrend": true },
        { "visualId": "retention_m1", "type": "card", "title": "M1 Retention", "metrics": [{ "metricKey": "retention_cycle_1" }], "showTrend": true }
      ]
    },
    {
      "id": "retention_matrix",
      "type": "full_width",
      "visuals": [
        {
          "visualId": "cohort_heatmap",
          "type": "matrix",
          "title": "Retention Cohort Heatmap",
          "source": "cohort_summary",
          "rowDimension": "date_month",
          "columnDimension": "billing_cycle",
          "valueMetric": "approvals",
          "displayAs": "percentage",
          "baseMetric": "initials",
          "maxColumns": 12,
          "heatmapColors": { "low": "#FEE2E2", "mid": "#FEF3C7", "high": "#D1FAE5" }
        }
      ]
    },
    {
      "id": "ltv_table",
      "type": "full_width",
      "visuals": [
        {
          "visualId": "ltv_by_source",
          "type": "table",
          "source": "order_summary",
          "title": "LTV by Acquisition Channel",
          "dimension": "campaign_name",
          "metrics": [
            { "metricKey": "initials" },
            { "metricKey": "aov" },
            { "metricKey": "revenue" },
            { "metricKey": "cancel_rate" },
            { "metricKey": "cb_rate" }
          ],
          "sortBy": "revenue",
          "sortDirection": "desc",
          "pagination": { "pageSize": 25 },
          "showTotals": true
        }
      ]
    }
  ]
}
```

### 7.5 Sales Report

**Features demonstrated:** Summary card with horizontal stacked bar, insights card (AI-generated), sparkline columns in tables, multi-level group-by (GROUP BY + THEN BY), daily pivot matrix, visual-level time granularity toggle, schedule button, traffic quality metrics.

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": {
    "dateRange": "last_30_days",
    "refreshInterval": null
  },
  "slicerPanel": {
    "showDateRange": true,
    "showComparison": false,
    "slicers": [],
    "filters": [
      { "dimensionKey": "campaign_name", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "campaign_type", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "more", "dependsOn": "campaign_name" },
      { "dimensionKey": "product_id", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "product_group", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "bank", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "gateway_id", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "mcc", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "acquirer", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "gateway_alias", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "mid_corp", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "billing_cycle", "displayAs": "tree", "position": "more" }
    ],
    "showExport": true,
    "showSchedule": true,
    "showReset": true,
    "showShare": true,
    "showLastUpdated": true
  },
  "sections": [
    {
      "id": "summary_row",
      "type": "grid",
      "columns": 3,
      "visuals": [
        {
          "visualId": "summary_card",
          "type": "card",
          "cardVariant": "breakdown",
          "title": "Summary",
          "metrics": [
            { "metricKey": "rebills", "label": "Rebills" },
            { "metricKey": "revenue_rebills", "label": "Revenue" }
          ],
          "breakdownDimension": "sales_type",
          "breakdownLimit": 5,
          "showTrend": false,
          "showStackedBar": true,
          "emptyState": {
            "message": "No sales data for selected period",
            "icon": "chart"
          }
        },
        {
          "visualId": "sales_trend_card",
          "type": "card",
          "cardVariant": "sparkline",
          "title": "Sales Trend",
          "metrics": [
            { "metricKey": "approvals", "label": "Total" }
          ],
          "sparklinePeriod": "date_day",
          "showTrend": false
        },
        {
          "visualId": "insights_card",
          "type": "card",
          "cardVariant": "insight",
          "title": "Insights",
          "metrics": [
            { "metricKey": "initials" },
            { "metricKey": "rebills" },
            { "metricKey": "revenue" },
            { "metricKey": "aov" }
          ],
          "insightType": "trend",
          "insightConfig": {
            "showTrendAnalysis": true,
            "showPercentageBreakdown": true,
            "showAOVComparison": true
          }
        }
      ]
    },

    {
      "id": "sales_table_section",
      "type": "full_width",
      "title": "Sales",
      "visuals": [
        {
          "visualId": "sales_breakdown_table",
          "type": "table",
          "title": "Sales",
          "dimension": "{groupBy}",
          "groupByOptions": [
            "campaign_name",
            "campaign_type",
            "campaign_id",
            "product_name",
            "product_id",
            "product_group",
            "gateway_id",
            "acquirer",
            "gateway_alias",
            "mid_corp"
          ],
          "defaultGroupBy": ["campaign_name", "product_name"],
          "allowMultiLevelGroupBy": true,
          "maxGroupByLevels": 2,
          "groupByPosition": "slicer_panel",
          "metrics": [
            { "metricKey": "sales_trend_sparkline", "isSparkline": true, "sparklineMetric": "approvals", "sparklinePeriod": "date_day" },
            { "metricKey": "initials" },
            { "metricKey": "revenue_initials" },
            { "metricKey": "rebills" },
            { "metricKey": "revenue_rebills" },
            { "metricKey": "straight_sales" },
            { "metricKey": "revenue_straight" }
          ],
          "sortBy": "rebills",
          "sortDirection": "desc",
          "pagination": { "pageSize": 25 },
          "showTotals": true,
          "exportable": true,
          "emptyState": {
            "message": "No sales data found for selected filters",
            "icon": "search",
            "actionLabel": "Reset Filters"
          },
          "errorState": {
            "retryable": true
          },
          "drillThrough": {
            "targetReport": "transaction-explorer",
            "filterMapping": { "product_name": "product_name", "campaign_name": "campaign_name" },
            "openIn": "same_tab"
          }
        }
      ]
    },

    {
      "id": "traffic_quality_section",
      "type": "full_width",
      "title": "Traffic Quality",
      "visuals": [
        {
          "visualId": "traffic_quality_table",
          "type": "table",
          "title": "Traffic Quality",
          "dimension": "affiliate_id",
          "metrics": [
            { "metricKey": "attempts_initials", "label": "Initials Attempts" },
            { "metricKey": "initials", "label": "Initial Approved" },
            { "metricKey": "initial_approval_rate", "label": "Initial Approval %" },
            { "metricKey": "initial_cancel_rate", "label": "Initial Cancel %" },
            { "metricKey": "initial_refund_rate", "label": "Initial Refund %" },
            { "metricKey": "initial_cb_rate", "label": "Initial CB %" },
            { "metricKey": "cycle1_approvals", "label": "C1 Approval" },
            { "metricKey": "cycle1_approval_rate", "label": "C1 Approval %" },
            { "metricKey": "cycle1_cancel_rate", "label": "C1 Cancel %" },
            { "metricKey": "cycle1_refund_rate", "label": "C1 Refund %" },
            { "metricKey": "cycle1_cb_rate", "label": "C1 CB %" }
          ],
          "sortBy": "initials",
          "sortDirection": "desc",
          "pagination": { "pageSize": 25 },
          "showTotals": true,
          "isExpandable": true,
          "expandDimension": "sub_affiliate_id",
          "exportable": true,
          "emptyState": {
            "message": "No affiliate data available",
            "icon": "users"
          },
          "drillThrough": {
            "targetReport": "transaction-explorer",
            "filterMapping": { "affiliate_id": "affiliate_id" },
            "openIn": "new_tab"
          },
          "metricConfig": {
            "initial_approval_rate": {
              "conditionalColor": {
                "thresholds": [
                  { "value": 50, "color": "#FEE2E2" },
                  { "value": 70, "color": "#FEF3C7" },
                  { "value": 85, "color": "#D1FAE5" }
                ]
              }
            },
            "initial_cb_rate": {
              "conditionalColor": {
                "thresholds": [
                  { "value": 0.5, "color": "#D1FAE5" },
                  { "value": 0.8, "color": "#FEF3C7" },
                  { "value": 1.0, "color": "#FEE2E2" }
                ]
              }
            }
          }
        }
      ]
    },

    {
      "id": "sales_trend_section",
      "type": "full_width",
      "title": "Sales Trend",
      "visuals": [
        {
          "visualId": "sales_trend_area_chart",
          "type": "chart",
          "chartType": "area",
          "title": "Sales Trend",
          "xAxis": { "dimensionKey": "date_day", "label": "Date" },
          "series": [
            { "metricKey": "initials", "label": "# Initials", "color": "#6366F1" },
            { "metricKey": "rebills", "label": "# Rebills", "color": "#8B5CF6" },
            { "metricKey": "straight_sales", "label": "# Straight Sales", "color": "#A78BFA" }
          ],
          "stacked": false,
          "showLegend": true,
          "yAxisFormat": "number",
          "visualTimeGranularity": {
            "enabled": true,
            "position": "top-right",
            "displayAs": "pills",
            "options": ["day", "week", "month"],
            "default": "day"
          },
          "emptyState": {
            "message": "No trend data for selected filters",
            "icon": "chart",
            "actionLabel": "Expand Date Range"
          },
          "errorState": {
            "retryable": true,
            "message": "Failed to load sales trend data"
          }
        }
      ]
    },

    {
      "id": "daily_pivot_section",
      "type": "full_width",
      "title": "Daily Breakdown",
      "visuals": [
        {
          "visualId": "daily_pivot_matrix",
          "type": "matrix",
          "title": "Daily Sales Detail",
          "rowDimension": "metric_name",
          "columnDimension": "date_day",
          "valueMetrics": [
            { "metricKey": "initials", "label": "# Initials" },
            { "metricKey": "revenue_initials", "label": "$ Initials" },
            { "metricKey": "rebills", "label": "# Rebills" },
            { "metricKey": "revenue_rebills", "label": "$ Rebills" },
            { "metricKey": "straight_sales", "label": "# Straight Sales" },
            { "metricKey": "revenue_straight", "label": "$ Straight Sales" }
          ],
          "displayAs": "values",
          "pivotMode": "metrics_as_rows",
          "maxColumns": 17,
          "showTotals": false,
          "exportable": true,
          "emptyState": {
            "message": "No daily data for selected period",
            "icon": "calendar"
          }
        }
      ]
    }
  ]
}
```

**Features exercised in this Sales Report template:**

| Category | Features |
|----------|---------|
| **Visual types** | `card` (3 cards: breakdown, sparkline, insight), `chart` (1 area chart), `table` (2 tables), `matrix` (1 daily pivot) |
| **Card variants** | `breakdown` (Summary with stacked bar), `sparkline` (Sales Trend), `insight` (AI-generated insights) |
| **Chart features** | Area chart with `visualTimeGranularity` toggle (Day/Week/Month pills on chart) |
| **Table features** | Multi-level `groupByOptions` with `groupByPosition: "slicer_panel"`, sparkline columns, `isExpandable` rows |
| **Matrix features** | `pivotMode: "metrics_as_rows"` for daily breakdown with dates as columns |
| **Multi-level Group By** | `allowMultiLevelGroupBy: true` for Campaign → Product grouping (table-level) |
| **Filter features** | `dependsOn` (product depends on campaign), 12 dimension filters in More Filters |
| **State handling** | `emptyState` on 5 visuals, `errorState` on 2 visuals |
| **Drill-through** | 2 tables with drill-through to Transaction Explorer |
| **Slicer panel** | `showSchedule`, `showReset`, `showShare`, `showLastUpdated` |
| **Traffic Quality** | Affiliate metrics with expandable sub-affiliate rows, conditional coloring |

**New concepts introduced in this template (Rev 4.1):**

| Concept | Description |
|---------|-------------|
| `allowMultiLevelGroupBy` | Table-level GROUP BY + THEN BY multi-level grouping support |
| `groupByPosition` | Position group-by selector in `"slicer_panel"` or `"table_header"` |
| `showSchedule` | Schedule button for report delivery |
| `visualTimeGranularity` | Chart-level time granularity toggle (independent of report-level slicer) |
| `isSparkline` on metrics | Table column that renders as inline sparkline chart |
| `isExpandable` | Table rows can expand to show child dimension (e.g., Affiliate → Sub-Affiliate) |
| `pivotMode: "metrics_as_rows"` | Matrix with metrics as rows and dates as columns |
| `insightConfig` | Configuration for AI-generated insight cards |

---

### 7.6 Complete Reference Template (All Features)

This template exercises every visual type, section type, slicer, filter mode, parameter, and special behavior defined in this spec. It is not a real report — it is a reference for implementors to validate that the renderer supports every feature.

```json
{
  "version": "1.0",
  "source": "order_summary",
  "sourceConfig": {
    "serverSidePagination": false,
    "serverSideSort": false,
    "serverSideSearch": false
  },
  "defaults": {
    "dateRange": "last_30_days",
    "refreshInterval": 300
  },
  "slicerPanel": {
    "showDateRange": true,
    "showComparison": true,
    "comparisonPresets": ["previous_period", "previous_week", "previous_month", "custom"],
    "showSearch": true,
    "searchPlaceholder": "Search by order ID, email, name...",
    "slicers": [
      { "slicerKey": "approval_mode", "position": "main", "displayAs": "pills" },
      { "slicerKey": "date_basis", "position": "more", "displayAs": "select" },
      { "slicerKey": "time_granularity", "position": "main", "displayAs": "pills" },
      { "slicerKey": "profitability_view", "position": "main", "displayAs": "pills" },
      { "slicerKey": "retention_base", "position": "more", "displayAs": "pills" },
      { "slicerKey": "display_mode", "position": "more", "displayAs": "pills" }
    ],
    "filters": [
      { "dimensionKey": "sales_type", "displayAs": "pills", "position": "main" },
      { "dimensionKey": "campaign_name", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "more", "dependsOn": "campaign_name" },
      { "dimensionKey": "gateway_alias", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "card_brand", "displayAs": "select", "position": "more" },
      { "dimensionKey": "affiliate_id", "displayAs": "search", "position": "more" },
      { "dimensionKey": "billing_cycle", "displayAs": "tree", "position": "more" }
    ],
    "parameters": [
      { "parameterKey": "processing_fee_pct", "label": "Processing Fees", "type": "number", "suffix": "%", "default": 3.5, "step": 0.1, "min": 0, "max": 20 },
      { "parameterKey": "reserve_pct", "label": "Reserve", "type": "number", "suffix": "%", "default": 5.0, "step": 0.5, "min": 0, "max": 50 },
      { "parameterKey": "cb_fee_amount", "label": "CB Fee", "type": "number", "prefix": "$", "default": 25.00, "step": 1.0, "min": 0, "max": 100 },
      { "parameterKey": "cpa_amount", "label": "CPA", "type": "number", "prefix": "$", "default": 45.00, "step": 1.0, "min": 0, "max": 500 },
      { "parameterKey": "rdr_amount", "label": "RDR", "type": "number", "prefix": "$", "default": 0, "step": 1, "min": 0, "max": 100, "group": "alert_costs" },
      { "parameterKey": "ethoca_amount", "label": "Ethoca", "type": "number", "prefix": "$", "default": 0, "step": 1, "min": 0, "max": 100, "group": "alert_costs" },
      { "parameterKey": "cdrn_amount", "label": "CDRN", "type": "number", "prefix": "$", "default": 0, "step": 1, "min": 0, "max": 100, "group": "alert_costs" },
      { "parameterKey": "other_alert_amount", "label": "Other", "type": "number", "prefix": "$", "default": 0, "step": 1, "min": 0, "max": 100, "group": "alert_costs" }
    ],
    "statusFilters": [
      { "key": "all", "label": "All" },
      { "key": "approved", "label": "Approved", "filter": { "is_approved": true } },
      { "key": "declined", "label": "Declined", "filter": { "is_approved": false } },
      { "key": "refunded", "label": "Refunded", "filter": { "is_refund": true } },
      { "key": "chargeback", "label": "Chargeback", "filter": { "is_chargeback": true } }
    ],
    "showExport": true,
    "showReset": true,
    "showLastUpdated": true,
    "showShare": true
  },
  "sections": [
    {
      "id": "health_indicators",
      "type": "card_row",
      "title": "System Health",
      "visuals": [
        {
          "visualId": "cb_health",
          "type": "healthIndicator",
          "title": "CB Rate Health",
          "metricKey": "cb_rate",
          "interactionMode": "none",
          "thresholds": {
            "green": { "min": 0, "max": 0.8 },
            "yellow": { "min": 0.8, "max": 1.0 },
            "red": { "min": 1.0, "max": 100 }
          }
        },
        {
          "visualId": "approval_health",
          "type": "healthIndicator",
          "title": "Approval Rate Health",
          "metricKey": "approval_rate",
          "interactionMode": "none",
          "thresholds": {
            "green": { "min": 70, "max": 100 },
            "yellow": { "min": 50, "max": 70 },
            "red": { "min": 0, "max": 50 }
          }
        }
      ]
    },

    {
      "id": "summary_cards",
      "type": "card_row",
      "title": "Key Metrics (Standard Cards)",
      "visuals": [
        {
          "visualId": "revenue_card",
          "type": "card",
          "title": "Revenue",
          "metrics": [
            { "metricKey": "revenue", "label": "Total" },
            { "metricKey": "revenue_initials", "label": "Initials" },
            { "metricKey": "revenue_rebills", "label": "Rebills" }
          ],
          "showTrend": true,
          "trendComparison": "previous_period"
        },
        {
          "visualId": "approvals_card",
          "type": "card",
          "title": "Approvals",
          "metrics": [
            { "metricKey": "approvals", "label": "Count" },
            { "metricKey": "approval_rate", "label": "Rate" }
          ],
          "showTrend": true,
          "trendComparison": "previous_period"
        },
        {
          "visualId": "new_customers_card",
          "type": "card",
          "title": "New Customers",
          "metrics": [{ "metricKey": "initials", "label": "Count" }],
          "showTrend": true,
          "trendComparison": "previous_period"
        },
        {
          "visualId": "cancel_rate_card",
          "type": "card",
          "title": "Cancel Rate",
          "metrics": [{ "metricKey": "cancel_rate", "label": "Rate" }],
          "showTrend": true,
          "trendComparison": "previous_period",
          "invertTrend": true
        },
        {
          "visualId": "cb_rate_card",
          "type": "card",
          "title": "CB Rate",
          "metrics": [
            { "metricKey": "cb_rate", "label": "Rate" },
            { "metricKey": "cb_count", "label": "Count" }
          ],
          "showTrend": true,
          "invertTrend": true
        }
      ]
    },

    {
      "id": "card_variants",
      "type": "card_row",
      "title": "Card Variants (All 7 Types)",
      "visuals": [
        {
          "visualId": "breakdown_card",
          "type": "card",
          "cardVariant": "breakdown",
          "title": "Revenue by Campaign",
          "metrics": [{ "metricKey": "revenue" }],
          "breakdownDimension": "campaign_name",
          "breakdownLimit": 5,
          "showTrend": true
        },
        {
          "visualId": "donut_card",
          "type": "card",
          "cardVariant": "donut",
          "title": "Revenue Mix",
          "metrics": [{ "metricKey": "revenue" }],
          "chartDimension": "sales_type"
        },
        {
          "visualId": "sparkline_card",
          "type": "card",
          "cardVariant": "sparkline",
          "title": "Revenue Trend",
          "metrics": [{ "metricKey": "revenue" }],
          "sparklinePeriod": "date_day",
          "showTrend": true
        },
        {
          "visualId": "summary_card",
          "type": "card",
          "cardVariant": "summary",
          "title": "Weekly Summary",
          "metrics": [
            { "metricKey": "revenue" },
            { "metricKey": "initials" },
            { "metricKey": "approval_rate" }
          ],
          "summaryTemplate": "You made {revenue} this week with {initials} new customers at {approval_rate} approval."
        },
        {
          "visualId": "chart_card",
          "type": "card",
          "cardVariant": "chart",
          "title": "Sales Type Mix",
          "metrics": [{ "metricKey": "revenue" }],
          "chartType": "bar",
          "chartDimension": "sales_type"
        }
      ]
    },

    {
      "id": "trend_charts",
      "type": "grid",
      "columns": 2,
      "title": "Trend Analysis",
      "visuals": [
        {
          "visualId": "revenue_area_chart",
          "type": "chart",
          "chartType": "area",
          "title": "Revenue Trend",
          "xAxis": { "dimensionKey": "{time_granularity}" },
          "series": [
            {
              "metricKey": "revenue_initials",
              "label": "Initials",
              "color": "#4F46E5",
              "comparison": "previous_period",
              "comparisonLabel": "Initials (Prev)",
              "comparisonColor": "#A5B4FC",
              "comparisonStyle": "dashed"
            },
            { "metricKey": "revenue_rebills", "label": "Rebills", "color": "#10B981" }
          ],
          "stacked": true,
          "showLegend": true,
          "yAxisFormat": "currency"
        },
        {
          "visualId": "approval_line_chart",
          "type": "chart",
          "chartType": "line",
          "title": "Approval Rate Trend",
          "xAxis": { "dimensionKey": "{time_granularity}" },
          "series": [
            { "metricKey": "approval_rate", "label": "Approval %", "color": "#10B981" },
            { "metricKey": "cancel_rate", "label": "Cancel %", "color": "#EF4444", "style": "dashed" }
          ],
          "showLegend": true,
          "yAxisFormat": "percentage"
        }
      ]
    },

    {
      "id": "hourly_cross_source",
      "type": "full_width",
      "title": "Hourly Revenue (Cross-Source)",
      "visuals": [
        {
          "visualId": "hourly_revenue_chart",
          "type": "chart",
          "source": "hourly_revenue",
          "chartType": "area",
          "title": "Revenue Today vs 7-Day Avg",
          "xAxis": { "dimensionKey": "date_hour" },
          "series": [
            { "metricKey": "today_revenue", "label": "Today", "color": "#4F46E5" },
            { "metricKey": "avg_7d_revenue", "label": "7-Day Avg", "color": "#94A3B8", "style": "dashed" }
          ],
          "showLegend": true,
          "yAxisFormat": "currency"
        }
      ]
    },

    {
      "id": "combo_and_bar",
      "type": "split",
      "title": "Combo & Bar Charts",
      "visuals": [
        {
          "visualId": "revenue_vs_rate_combo",
          "type": "chart",
          "chartType": "combo",
          "title": "Revenue vs Approval Rate",
          "xAxis": { "dimensionKey": "{time_granularity}" },
          "series": [
            { "metricKey": "revenue", "label": "Revenue", "chartType": "bar", "color": "#4F46E5" },
            { "metricKey": "approval_rate", "label": "Approval %", "chartType": "line", "color": "#10B981" }
          ],
          "showLegend": true
        },
        {
          "visualId": "attempts_bar_chart",
          "type": "chart",
          "chartType": "bar",
          "title": "Attempts by Gateway",
          "xAxis": { "dimensionKey": "gateway_alias" },
          "series": [
            { "metricKey": "attempts", "label": "Attempts", "color": "#6366F1" },
            { "metricKey": "approvals", "label": "Approvals", "color": "#10B981" }
          ],
          "showLegend": true,
          "showDataLabels": false
        }
      ]
    },

    {
      "id": "pie_and_donut",
      "type": "split",
      "title": "Distribution Charts",
      "visuals": [
        {
          "visualId": "revenue_by_sales_type_pie",
          "type": "chart",
          "chartType": "pie",
          "title": "Revenue by Sales Type",
          "dimension": "sales_type",
          "metric": "revenue",
          "showLegend": true,
          "showDataLabels": true,
          "colors": ["#4F46E5", "#10B981", "#F59E0B", "#EF4444"]
        },
        {
          "visualId": "approvals_by_card_brand_donut",
          "type": "chart",
          "chartType": "donut",
          "title": "Approvals by Card Brand",
          "dimension": "card_brand",
          "metric": "approvals",
          "showLegend": true,
          "colors": ["#1E40AF", "#047857", "#B45309", "#B91C1C", "#6B21A8"]
        }
      ]
    },

    {
      "id": "stacked_bar_section",
      "type": "full_width",
      "title": "Revenue Composition",
      "visuals": [
        {
          "visualId": "revenue_by_product_stacked",
          "type": "chart",
          "chartType": "stacked_bar",
          "title": "Revenue by Product (Weekly)",
          "xAxis": { "dimensionKey": "date_week" },
          "dimension": "product_name",
          "metric": "revenue",
          "showLegend": true
        }
      ]
    },

    {
      "id": "waterfall_section",
      "type": "full_width",
      "title": "Profitability Waterfall",
      "conditionalVisibility": {
        "slicerKey": "profitability_view",
        "showWhen": "cashflow"
      },
      "visuals": [
        {
          "visualId": "pnl_waterfall",
          "type": "waterfall",
          "title": "P&L Waterfall",
          "steps": [
            { "metricKey": "revenue", "label": "Gross Revenue", "type": "start" },
            { "metricKey": "refund_dollar", "label": "Refunds", "type": "subtract" },
            { "metricKey": "cb_dollar", "label": "Chargebacks", "type": "subtract" },
            { "metricKey": "processing_fees", "label": "Processing Fees", "type": "subtract" },
            { "metricKey": "cb_fees", "label": "CB Fees", "type": "subtract" },
            { "metricKey": "cpa_cost", "label": "CPA Cost", "type": "subtract" },
            { "metricKey": "reserve", "label": "Reserve", "type": "subtract" },
            { "metricKey": "gross_profit", "label": "Gross Profit", "type": "end" }
          ],
          "colors": {
            "increase": "#10B981",
            "decrease": "#EF4444",
            "total": "#4F46E5"
          }
        }
      ]
    },

    {
      "id": "aggregated_table_section",
      "type": "full_width",
      "title": "Revenue Breakdown",
      "visuals": [
        {
          "visualId": "revenue_breakdown_table",
          "type": "table",
          "title": "Revenue by {groupBy}",
          "dimension": "{groupBy}",
          "groupByOptions": [
            "campaign_name",
            "product_name",
            "gateway_alias",
            "sales_type",
            "card_brand",
            "affiliate_id",
            "billing_cycle"
          ],
          "defaultGroupBy": ["campaign_name"],
          "metrics": [
            { "metricKey": "attempts" },
            { "metricKey": "approvals" },
            { "metricKey": "approval_rate" },
            { "metricKey": "revenue" },
            { "metricKey": "aov" },
            { "metricKey": "initials" },
            { "metricKey": "rebills" },
            { "metricKey": "cancels" },
            { "metricKey": "cancel_rate" },
            { "metricKey": "cb_count" },
            { "metricKey": "cb_rate" },
            { "metricKey": "refund_count" },
            { "metricKey": "refund_rate" }
          ],
          "sortBy": "revenue",
          "sortDirection": "desc",
          "pagination": { "pageSize": 25 },
          "showTotals": true,
          "allowColumnToggle": true,
          "defaultVisibleColumns": ["attempts", "approvals", "approval_rate", "revenue", "aov", "cancel_rate", "cb_rate"],
          "columnPresets": [
            {
              "label": "Revenue Focus",
              "columns": ["approvals", "revenue", "aov", "initials", "rebills"]
            },
            {
              "label": "Risk Focus",
              "columns": ["approvals", "cancels", "cancel_rate", "cb_count", "cb_rate", "refund_count", "refund_rate"]
            },
            {
              "label": "Full View",
              "columns": ["attempts", "approvals", "approval_rate", "revenue", "aov", "initials", "rebills", "cancels", "cancel_rate", "cb_count", "cb_rate", "refund_count", "refund_rate"]
            }
          ],
          "exportable": true,
          "emptyState": {
            "message": "No data found for the selected filters",
            "icon": "search",
            "actionLabel": "Reset Filters"
          },
          "errorState": {
            "retryable": true,
            "message": "Failed to load revenue data"
          },
          "drillThrough": {
            "targetReport": "transaction-explorer",
            "filterMapping": { "campaign_name": "campaign_name", "product_name": "product_name" },
            "openIn": "same_tab"
          },
          "metricConfig": {
            "approval_rate": {
              "showBar": true,
              "conditionalColor": {
                "thresholds": [
                  { "value": 50, "color": "#FEE2E2" },
                  { "value": 70, "color": "#FEF3C7" },
                  { "value": 85, "color": "#D1FAE5" }
                ]
              }
            },
            "cb_rate": {
              "conditionalColor": {
                "thresholds": [
                  { "value": 0.5, "color": "#D1FAE5" },
                  { "value": 0.8, "color": "#FEF3C7" },
                  { "value": 1.0, "color": "#FEE2E2" }
                ]
              }
            }
          }
        }
      ]
    },

    {
      "id": "multi_groupby_table_section",
      "type": "full_width",
      "title": "Multi-Dimension Breakdown",
      "visuals": [
        {
          "visualId": "multi_groupby_table",
          "type": "table",
          "title": "Breakdown by {groupBy}",
          "dimension": "{groupBy}",
          "groupByOptions": [
            "campaign_name",
            "product_name",
            "gateway_alias",
            "card_brand",
            "affiliate_id"
          ],
          "defaultGroupBy": ["campaign_name", "product_name"],
          "metrics": [
            { "metricKey": "approvals" },
            { "metricKey": "revenue" },
            { "metricKey": "approval_rate" },
            { "metricKey": "cancel_rate" },
            { "metricKey": "cb_rate" }
          ],
          "sortBy": "revenue",
          "sortDirection": "desc",
          "pagination": { "pageSize": 25 },
          "showTotals": true
        }
      ]
    },

    {
      "id": "profitability_table_section",
      "type": "full_width",
      "title": "Profitability Analysis",
      "conditionalVisibility": {
        "slicerKey": "profitability_view",
        "showWhen": ["cashflow", "detailed"]
      },
      "visuals": [
        {
          "visualId": "profitability_table",
          "type": "table",
          "title": "Profitability by {groupBy}",
          "dimension": "{groupBy}",
          "groupByOptions": ["product_name", "campaign_name"],
          "defaultGroupBy": ["product_name"],
          "metrics": [
            { "metricKey": "revenue" },
            { "metricKey": "refund_dollar" },
            { "metricKey": "cb_dollar" },
            { "metricKey": "processing_fees" },
            { "metricKey": "cb_fees" },
            { "metricKey": "cpa_cost" },
            { "metricKey": "reserve" },
            { "metricKey": "net_revenue" },
            { "metricKey": "gross_profit" },
            { "metricKey": "gross_margin" }
          ],
          "sortBy": "gross_profit",
          "sortDirection": "desc",
          "showTotals": true,
          "exportable": true
        }
      ]
    },

    {
      "id": "tabbed_charts",
      "type": "tabs",
      "title": "Deep Dive (Tabbed)",
      "tabs": [
        {
          "label": "Churn Trend",
          "visual": {
            "visualId": "churn_trend_tab",
            "type": "chart",
            "chartType": "line",
            "title": "Churn Metrics Over Time",
            "xAxis": { "dimensionKey": "{time_granularity}" },
            "series": [
              { "metricKey": "cancel_rate", "label": "Cancel %", "color": "#EF4444" },
              { "metricKey": "cb_rate", "label": "CB %", "color": "#F59E0B" },
              { "metricKey": "refund_rate", "label": "Refund %", "color": "#6366F1" }
            ],
            "showLegend": true,
            "yAxisFormat": "percentage"
          }
        },
        {
          "label": "Gateway Performance",
          "visual": {
            "visualId": "gateway_table_tab",
            "type": "table",
            "title": "Gateway Scorecard",
            "dimension": "gateway_alias",
            "metrics": [
              { "metricKey": "attempts" },
              { "metricKey": "approvals" },
              { "metricKey": "approval_rate" },
              { "metricKey": "revenue" },
              { "metricKey": "declines" }
            ],
            "sortBy": "attempts",
            "sortDirection": "desc",
            "showTotals": true
          }
        },
        {
          "label": "Card Brand Mix",
          "visual": {
            "visualId": "card_brand_donut_tab",
            "type": "chart",
            "chartType": "donut",
            "title": "Revenue by Card Brand",
            "dimension": "card_brand",
            "metric": "revenue",
            "showLegend": true,
            "showDataLabels": true
          }
        }
      ]
    },

    {
      "id": "retention_matrix_section",
      "type": "full_width",
      "title": "Retention Cohort Heatmap",
      "visuals": [
        {
          "visualId": "cohort_heatmap",
          "type": "matrix",
          "title": "Retention by Cohort Month",
          "source": "cohort_summary",
          "rowDimension": "date_month",
          "columnDimension": "billing_cycle",
          "valueMetric": "approvals",
          "displayAs": "percentage",
          "baseMetric": "initials",
          "maxColumns": 12,
          "heatmapColors": {
            "low": "#FEE2E2",
            "mid": "#FEF3C7",
            "high": "#D1FAE5"
          }
        }
      ]
    },

    {
      "id": "detail_table_section",
      "type": "full_width",
      "title": "Transaction Explorer",
      "visuals": [
        {
          "visualId": "transaction_detail_table",
          "type": "table",
          "source": "order_details",
          "title": "Transactions",
          "isDetailTable": true,
          "columns": [
            { "key": "date_of_sale", "label": "Date", "format": "date" },
            { "key": "order_id", "label": "Order ID" },
            { "key": "bill_email", "label": "Email" },
            { "key": "bill_first", "label": "First Name" },
            { "key": "bill_last", "label": "Last Name" },
            { "key": "order_total", "label": "Amount", "format": "currency" },
            { "key": "product_name", "label": "Product" },
            { "key": "campaign_name", "label": "Campaign" },
            { "key": "gateway_alias", "label": "Gateway" },
            { "key": "sales_type", "label": "Type" },
            { "key": "billing_cycle", "label": "Cycle", "format": "number" },
            { "key": "is_approved", "label": "Status", "format": "status_badge" },
            { "key": "card_brand", "label": "Card Brand" },
            { "key": "transaction_id", "label": "Transaction ID" }
          ],
          "sortBy": "date_of_sale",
          "sortDirection": "desc",
          "pagination": { "pageSize": 50 },
          "allowColumnToggle": true,
          "exportable": true,
          "interactionMode": "none"
        }
      ]
    }
  ]
}
```

**Features exercised in this template:**

| Category | Features |
|----------|---------|
| **Visual types (7)** | `card` (10 cards, 6 variants), `chart` (7 charts across all chartTypes), `table` (3 aggregated + 1 detail), `matrix` (cohort heatmap), `waterfall` (P&L), `healthIndicator` (2 badges) |
| **Card variants (6)** | `metric` (5 standard cards), `breakdown` (with dimension breakdown), `donut` (with inline donut), `sparkline` (with trend line), `summary` (template text), `chart` (with embedded bar) |
| **Chart types (7)** | `area`, `line`, `bar`, `stacked_bar`, `combo`, `pie`, `donut` |
| **Section types (5)** | `card_row` (2), `full_width` (7), `grid` (1, columns:2), `split` (2), `tabs` (1, 3 tabs) |
| **Slicers (6)** | `approval_mode`, `date_basis`, `time_granularity`, `profitability_view`, `retention_base`, `display_mode` |
| **Filter displayAs (5)** | `pills`, `multiselect`, `select`, `search`, `tree` |
| **Filter positions (2)** | `main`, `more` |
| **Parameters (8)** | 4 ungrouped (processing, reserve, CB fee, CPA), 4 grouped under `alert_costs` (RDR, Ethoca, CDRN, Other) |
| **Status filters (5)** | All, Approved, Declined, Refunded, Chargeback |
| **Table features** | `groupByOptions` with 7 options, `defaultGroupBy` as array (single and multi), `allowColumnToggle`, `defaultVisibleColumns`, `columnPresets` (3 presets), `showTotals`, `exportable`, `metricConfig` with `showBar` + `conditionalColor`, `pagination` |
| **Detail table** | `isDetailTable: true`, 14 columns, 6 `format` types (date, currency, number, status_badge, text implicit) |
| **Cross-source** | Main source `order_summary`, hourly chart from `hourly_revenue`, matrix from `cohort_summary`, detail table from `order_details` |
| **Comparison** | `showComparison: true` with 4 presets, series-level `comparison: "previous_period"` on revenue chart |
| **Conditional visibility** | Waterfall + profitability table show only when `profitability_view` matches |
| **Placeholders** | `{groupBy}` on 3 tables, `{time_granularity}` on 3 charts |
| **Special behaviors** | `refreshInterval: 300`, `invertTrend: true` on cancel/CB cards, `interactionMode: "none"` on health indicators + detail table, `showSearch: true` with placeholder |
| **Filter dependencies** | `dependsOn: "campaign_name"` on product filter (Rev 4) |
| **State handling (Rev 4)** | `emptyState` with message/icon/action, `errorState` with retryable + message on `revenue_breakdown_table` |
| **Drill-through (Rev 4)** | `drillThrough` targeting `transaction-explorer` with filter mapping on `revenue_breakdown_table` |
| **Slicer panel (Rev 4)** | `showReset`, `showLastUpdated`, `showShare` for filter reset, data freshness, and URL deep linking |

---

## 8. Naming Conventions & Validation Checklist

### 8.1 Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| `visualId` | `snake_case`, descriptive | `"revenue_card"`, `"cohort_heatmap"`, `"transaction_table"` |
| Section `id` | `snake_case`, descriptive | `"summary_cards"`, `"hourly_chart"`, `"breakdown"` |
| `type` (visual) | `lowercase` | `"card"`, `"chart"`, `"table"`, `"matrix"`, `"waterfall"` |
| `type` (section) | `snake_case` | `"card_row"`, `"full_width"`, `"grid"`, `"split"`, `"tabs"` |
| `chartType` | `snake_case` | `"line"`, `"area"`, `"bar"`, `"stacked_bar"`, `"combo"`, `"pie"`, `"donut"` |
| `metricKey` | `snake_case` | `"revenue"`, `"approval_rate"`, `"ltv_30d"` |
| `dimensionKey` | `snake_case` | `"campaign_name"`, `"date_day"`, `"billing_cycle"` |
| `slicerKey` | `snake_case` | `"approval_mode"`, `"time_granularity"` |
| `parameterKey` | `snake_case` | `"processing_fee_pct"`, `"cb_fee_amount"` |
| Colors | Hex with `#` | `"#4F46E5"`, `"#10B981"` |

### 8.2 Validation Checklist

When creating or modifying a template, verify:

**Structure:**
- [ ] `version` is `"1.0"`
- [ ] `source` matches a key in `report_builder.data_sources`
- [ ] `defaults.dateRange` is a valid preset
- [ ] `slicerPanel` object is present
- [ ] `sections` array is non-empty

**Visuals:**
- [ ] Every visual has a unique `visualId` (unique within the template)
- [ ] Every visual has a valid `type`
- [ ] Every `metricKey` exists in `report_builder.metrics`
- [ ] Every `dimensionKey` exists in `report_builder.dimensions`
- [ ] Every visual's `source` (if overridden) exists in `report_builder.data_sources`
- [ ] Metrics referenced are compatible with the visual's data source (check `data_source_metrics`)
- [ ] If `drillThrough.targetReport` is set, it must be a valid `template_key` in `report_builder.report_templates`
- [ ] If `drillThrough.filterMapping` is set, all keys must match dimension keys in the target report
- [ ] `emptyState.message` is required if `emptyState` is present
- [ ] `errorState.retryable` is required if `errorState` is present

**Sections:**
- [ ] Every section has a unique `id`
- [ ] Section `type` is one of: `card_row`, `full_width`, `grid`, `split`, `tabs`
- [ ] `grid` sections have a `columns` field
- [ ] `split` sections have exactly 2 visuals
- [ ] `tabs` sections use `tabs` array (not `visuals`)
- [ ] `conditionalVisibility.slicerKey` matches a slicer in `slicerPanel.slicers`

**Slicer Panel:**
- [ ] Every `slicerKey` in `slicers` exists in `report_builder.metric_toggles`
- [ ] Every `dimensionKey` in `filters` exists in `report_builder.dimensions`
- [ ] Every `parameterKey` in `parameters` has `min`, `max`, `default`, `step`
- [ ] If `showComparison: true`, at least one `comparisonPresets` value is provided
- [ ] Parameters with same `group` value have consistent `prefix`/`suffix`
- [ ] If `dependsOn` is set on a filter, the parent `dimensionKey` must exist in the same `filters` array

**Tables:**
- [ ] `isDetailTable: true` tables use `columns` array (not `dimension`+`metrics`)
- [ ] `isDetailTable: false` (or omitted) tables use `dimension`+`metrics` (not `columns`)
- [ ] `defaultVisibleColumns` (if present) only contains metricKeys from the table's `metrics` array
- [ ] `columnPresets` (if present) only reference metricKeys from the table's `metrics` array
- [ ] `groupByOptions` (if present) only contains dimension keys from `report_builder.dimensions`
- [ ] `defaultGroupBy` (if present) is a subset of `groupByOptions`
- [ ] `{groupBy}` placeholder in `dimension`/`title` only appears on tables that have `groupByOptions`

**Cards:**
- [ ] If `cardVariant` is `"breakdown"`, `breakdownDimension` must be set
- [ ] If `cardVariant` is `"donut"` or `"chart"`, `chartDimension` must be set
- [ ] If `cardVariant` is `"sparkline"`, `sparklinePeriod` is a valid date dimension (or uses default `date_day`)
- [ ] If `cardVariant` is `"summary"`, `summaryTemplate` must be set and contain valid `{metricKey}` placeholders
- [ ] If `cardVariant` is `"chart"`, `chartType` must be `"donut"` or `"bar"`
- [ ] If `cardVariant` is `"insight"`, `insightType` is one of: `anomaly`, `trend`, `comparison`, `recommendation`

**Special Behaviors:**
- [ ] Waterfall `steps` has exactly one `"start"` and one `"end"` step
- [ ] `invertTrend` is only used on card visuals where decrease = good
- [ ] `{time_granularity}` placeholder only appears when a `time_granularity` slicer exists
- [ ] `interactionMode: "none"` is set on parameter inputs and health indicators
- [ ] Series with `comparison` field requires `showComparison: true` in slicerPanel or a valid auto-calculable preset

---

## 9. Changes from Earlier Documents

This specification standardizes inconsistencies found in `Development_Phase_Plan.md` (Section 7) and `JSON_Driven_System_Design.md`.

| Element | Old (Dev Phase Plan / JSON Design) | New (This Spec) | Reason |
|---------|-----------------------------------|-----------------|--------|
| Filter area key | `filterBar` | `slicerPanel` | PBI terminology (SESSION_CONTEXT section 4) |
| Toggle array key | `filterBar.toggles` | `slicerPanel.slicers` | PBI terminology — "slicers" not "toggles" in UI context |
| Toggle key field | `toggleKey` | `slicerKey` | Consistency with "slicer" terminology |
| Filter array key | `filterBar.filters` | `slicerPanel.filters` | Parent renamed |
| Visual type casing | `"Card"`, `"Chart"`, `"Table"`, `"Matrix"`, `"Waterfall"` | `"card"`, `"chart"`, `"table"`, `"matrix"`, `"waterfall"` | Lowercase matches convention for type discriminators |
| Visual identifier | `"id"` on some visuals | `"visualId"` on all visuals | Distinguish from section `"id"`. `visualId` keys batch query results. |
| Widget terminology | `"widget"`, `"widgets"` | `"visual"`, `"visuals"` | PBI terminology |
| Date range show flag | `showDateRange` in filterBar | `showDateRange` in slicerPanel | Same field, parent renamed |
| Group-by location | `showGroupBy` in filterBar (report-level) | `groupByOptions` on table visual (table-level) | Group-by only applies to tables, not entire report. Each table manages its own group-by selector. |
| Search show flag | `showSearch` in filterBar | `showSearch` in slicerPanel | Same field, parent renamed |
| Export show flag | `showExport` in filterBar | `showExport` in slicerPanel | Same field, parent renamed |
| Parameter key field | `key` | `parameterKey` | Explicit naming, matches `metricKey`/`slicerKey` pattern |
| Status filter key field | `key` (ambiguous) | `key` (kept — simple enough) | No change needed |
| Conditional visibility toggle ref | `toggleKey` | `slicerKey` | Consistency with slicer rename |
| Chart series metric ref | `metric` in series | `metricKey` in series | Consistency with `metricKey` everywhere |
| Summary table type | `"summary_table"` | Removed — use `"table"` with custom layout | Simplification |
| Data table type | `"data_table"` | `"table"` | One table type with two modes |

### 9.2 Gap Analysis Additions (Round 2)

After cross-referencing against PBI production reports (53 reports, 11,828 visuals), `beast_insights_v2.filter_definitions` (157 filters, 6 types), and `beastinsights-old` frontend components, 8 gaps were identified and fixed:

| # | Gap | What Was Added | Section |
|---|-----|---------------|---------|
| 1 | Comparison date range | `showComparison`, `comparisonPresets` on SlicerPanel; Section 3.7 | 3.1, 3.7 |
| 2 | Filter operators | `operator: "in" \| "not_in"` in batch query filter format; "all" = omit filter | 3.3 |
| 3 | Missing date range presets | Added `yesterday`, `last_calendar_week`, `last_4_weeks`, `last_month` to `DateRangePreset` | 2.1 |
| 4 | Parameter grouping | `group` field on `ParameterConfig` for grouped popovers (e.g., Alert $ inputs) | 3.4 |
| 5 | Column visibility presets | `defaultVisibleColumns`, `columnPresets` on aggregated tables | 5.3.1 |
| 6 | Nested filter cascading | Documentation note on parent-child behavior driven by `dimensions.parent_dimension_id` | 3.3 |
| 7 | Cross-visual filtering | `interactionMode: "crossFilter" \| "none"` on BaseVisual; Section 6.7 | 5, 6.7 |
| 8 | Chart comparison overlay | `comparison`, `comparisonLabel`, `comparisonColor`, `comparisonStyle` on SeriesConfig; Section 6.8 | 5.2, 6.8 |

**Not applicable (PBI-specific):** `actionButton`, `shape`, `textbox`, `image`, `multiRowCard`, mobile dual-layouts, bookmark navigation, theme management — all handled by React/Tailwind/Tremor at the application level, not per-template JSON.

**Migration for existing template examples:** When seeding the 11 stock reports in M5, use this spec's format (not the Development_Phase_Plan.md format). The Development_Phase_Plan.md Section 7 templates are reference examples only — this document is the definitive source.

### 9.3 Architectural Additions (Round 3)

After architectural review, 3 significant additions were made:

| # | Addition | What Was Added | Section |
|---|----------|---------------|---------|
| 1 | Visual size constraints | Frontend-enforced size rules by section type. No size fields in JSON — layout determined by section type (card_row, grid, split, etc.). Responsive breakpoints documented. | 4.8 |
| 2 | Card variants | 7 `cardVariant` types: `metric` (default), `breakdown`, `donut`, `sparkline`, `summary`, `chart`, `insight`. Each variant has specific extra fields for its display mode. | 5.1, 5.1.1 |
| 3 | DB-driven filter types | New `filter_types` table in SQL schema for extensible filter type registry. 8 core types seeded. New filter types can be added via INSERT without code changes. | SQL schema only (not JSON spec) |

**Card variant rationale:** PBI reports use different card visualizations (big number, number with breakdown list, number with sparkline, number in donut center, etc.). The `cardVariant` field makes this explicit in the JSON rather than requiring frontend to infer from field combinations.

**Visual size constraint rationale:** Keeps JSON simple (no width/height on visuals) while ensuring consistent responsive behavior. All sizing handled by CSS based on section type.

### 9.4 Gap Analysis Additions (Round 4)

After comprehensive gap analysis (53 PBI reports, 11,828 visuals, 157 DB filters, 201 frontend components), 7 must-have features were identified and added:

| # | Gap | What Was Added | Section |
|---|-----|---------------|---------|
| 1 | **Filter dependencies** | `dependsOn?: string` on FilterConfig — child filter options narrow based on parent selection | 3.3 |
| 2 | **Filter reset button** | `showReset?: boolean` on SlicerPanel — "Reset Filters" button to restore defaults | 3.1 |
| 3 | **Drill-through navigation** | `drillThrough?: DrillThroughConfig` on BaseVisual — click row to navigate to detail report with filters applied | 5 |
| 4 | **Empty state per visual** | `emptyState?: EmptyStateConfig` on BaseVisual — custom message, icon, action when query returns 0 rows | 5 |
| 5 | **Error state per visual** | `errorState?: ErrorStateConfig` on BaseVisual — retry button and custom error message when query fails | 5 |
| 6 | **Data freshness indicator** | `showLastUpdated?: boolean` on SlicerPanel — "Last updated: X ago" for auto-refresh reports | 3.1 |
| 7 | **URL deep linking** | New Section 6.9 — full state serialization to URL for shareable filtered views | 6.9 |

**Additional fields added:**

| Field | On | Purpose |
|-------|-----|---------|
| `showShare?: boolean` | SlicerPanel | Show "Share" button to copy deep link URL |

**Gap analysis coverage:**

| Source | Items Analyzed |
|--------|---------------|
| PBI reports | 53 reports, 11,828 visuals |
| Database filters | 157 filter configurations across 11 reports |
| Frontend components | 201 React components in beastinsights-old |

**Deferred to Phase 2:**
- B1: Cross-filter counts (performance overhead)
- C2: Hierarchical expand/collapse (complex backend)
- E2: Drill-down in hierarchy (depends on C2)
- G2: Bookmark integration

---

*JSON Template Format Specification — Milestone 2*
*Last updated: 2026-01-29 (Rev 4 — filter dependencies, drill-through, empty/error states, URL deep linking)*

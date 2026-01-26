# JSON-Driven Report System - Technical Design

## The Core Idea

**One generic query endpoint. One generic report renderer. JSON controls everything.**

```
┌─────────────────────────────────────────────────────────┐
│                    JSON Template                         │
│  (stored in DB, defines EVERYTHING about a report)       │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ What data   │  │ How to       │  │ What toggles / │  │
│  │ to fetch    │  │ render it    │  │ modes exist    │  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬────────┘  │
└─────────┼────────────────┼──────────────────┼───────────┘
          │                │                  │
     ┌────▼────┐     ┌─────▼─────┐     ┌──────▼──────┐
     │ Backend │     │ Frontend  │     │ Both use    │
     │ Query   │     │ Renderer  │     │ these to    │
     │ Engine  │     │           │     │ switch      │
     │         │     │           │     │ behavior    │
     └─────────┘     └───────────┘     └─────────────┘
```

---

## Part 1: How the Backend Works (Generic Query Engine)

### There is only ONE endpoint

```
POST /api/v1/query
```

It accepts a structured JSON body and returns data. No report-specific endpoints.

### Request Structure

```jsonc
{
  "source": "order_summary",          // Which materialized view
  "metrics": ["approvals", "revenue"], // What to calculate
  "dimensions": ["campaign_name"],     // How to group
  "filters": {                         // What to filter
    "dateRange": { "preset": "last_7_days" },
    "sales_type": { "op": "in", "values": ["Initials"] }
  },
  "sort": { "field": "revenue", "dir": "desc" },
  "pagination": { "page": 0, "size": 25 }
}
```

### What the backend does with this

```
Step 1: Look up "order_summary" → resolves to "reporting.order_summary_{clientId}"
Step 2: Look up "approvals" metric → finds SQL: "SUM(approvals)"
Step 3: Look up "revenue" metric → finds SQL: "SUM(revenue)"
Step 4: Look up "campaign_name" dimension → finds it needs JOIN to public.campaigns
Step 5: Build WHERE clause from filters
Step 6: Assemble SQL, execute, return rows
```

The backend has **methods** for each step (resolve source, resolve metric, resolve dimension, resolve join, build where, assemble SQL) but it never has report-specific logic. The JSON request drives everything.

### Backend Methods (Generic, Reusable)

```
┌──────────────────────────────────────────────────────────┐
│                    Query Engine                            │
│                                                           │
│  resolveSource(sourceName, clientId) → table name         │
│  resolveMetrics(metricKeys[]) → SQL expressions           │
│  resolveDimensions(dimensionKeys[]) → columns + JOINs     │
│  resolveFilters(filterObj) → WHERE clauses                │
│  buildSQL(select, from, joins, where, group, order, limit)│
│  executeQuery(sql, params) → rows                         │
│  formatResponse(rows, metricDefs) → formatted output      │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

No method knows about "Sales Report" or "Approval Report". They only know about metrics, dimensions, sources, and filters.

---

## Part 2: How the Frontend Works (Generic Report Renderer)

### The Rendering Pipeline

```
1. Page loads → fetch report template JSON from API
2. Parse template → extract widget list
3. For each widget → extract data requirements (metrics, dimensions, filters)
4. Build API requests from widget configs
5. Call POST /api/v1/query/batch (all widgets in one call)
6. Distribute results to widgets
7. Each widget renders itself based on its type config
```

### Example Flow: Sales Report

**Template JSON says:**
```jsonc
{
  "sections": [
    {
      "widgets": [
        {
          "id": "summary",
          "type": "metric_card_row",
          "cards": [
            { "title": "Initials", "metrics": ["initials", "revenue_initials"] },
            { "title": "Rebills", "metrics": ["rebills", "revenue_rebills"] }
          ]
          // No API url, no endpoint name - just metric keys
        },
        {
          "id": "trend",
          "type": "chart",
          "chartType": "line",
          "xAxis": "date_day",
          "series": [{ "metric": "approvals" }]
          // Again, just metric key + dimension key
        },
        {
          "id": "table",
          "type": "data_table",
          "dimension": "{groupBy}",
          "metrics": ["initials", "revenue_initials", "rebills", "revenue_rebills"]
        }
      ]
    }
  ]
}
```

**Frontend automatically builds these API calls:**
```jsonc
// One batch request with 3 queries:
{
  "queries": [
    {
      "id": "summary",
      "source": "order_summary",
      "metrics": ["initials", "revenue_initials", "rebills", "revenue_rebills"],
      "filters": { /* current filter state */ }
      // No dimensions = returns single row totals
    },
    {
      "id": "trend",
      "source": "order_summary",
      "metrics": ["approvals"],
      "dimensions": ["date_day"],
      "filters": { /* current filter state */ }
    },
    {
      "id": "table",
      "source": "order_summary",
      "metrics": ["initials", "revenue_initials", "rebills", "revenue_rebills"],
      "dimensions": ["campaign_name"],    // resolved from groupBy selection
      "filters": { /* current filter state */ },
      "sort": { "field": "revenue_initials", "dir": "desc" },
      "pagination": { "page": 0, "size": 25 }
    }
  ]
}
```

**The frontend never constructs URLs or knows endpoint paths per report.** It only:
1. Reads the template JSON
2. Extracts metric/dimension keys from each widget
3. Combines with current filter state
4. Sends to the ONE generic query endpoint
5. Renders the response using the widget type component

---

## Part 3: The Hard Problem — Context-Dependent Metrics

This is the most important design challenge. In your current system:

- **"# Approvals"** can mean 3 different things depending on a toggle:
  - Standard: `SUM(approvals)`
  - Organic (first attempt): `SUM(approvals_organic)`
  - Net (after retries): `SUM(net_approvals)`

- **"# CB"** can mean 2 different things depending on date basis:
  - By Order Date: `SUM(cb)` (CB counted on the date the order was placed)
  - By CB Date: `SUM(cb_cb_date)` (CB counted on the date the CB was filed)

- **"# Refunds"** similarly has:
  - By Order Date: `SUM(refund)`
  - By Refund Date: `SUM(refund_refund_date)`

- **"Refund"** can further split by type:
  - Refund Alert (alert-triggered)
  - Refund CS (customer service)
  - All Refunds

These are NOT just filters. **They change which column is used in the SQL.** A filter says "only show rows where X = Y". A mode says "use column A instead of column B".

### Solution: Metric Variants System

#### Concept

A **Metric Group** contains multiple **Metric Variants**. A **Toggle** controls which variant is active.

```
┌─────────────────────────────────────────────────┐
│ Metric Group: "approvals"                        │
│                                                  │
│  Toggle: "approval_mode"                         │
│  ┌─────────────────────────────────────────────┐ │
│  │ Variant: "standard"  → SUM(approvals)       │ │
│  │ Variant: "organic"   → SUM(approvals_organic)│ │
│  │ Variant: "net"       → SUM(net_approvals)   │ │
│  └─────────────────────────────────────────────┘ │
│                                                  │
│  When UI toggle = "organic", query engine uses   │
│  SUM(approvals_organic) instead of SUM(approvals)│
└─────────────────────────────────────────────────┘
```

#### Database Schema for Metric Variants

```sql
-- Toggles that change metric behavior
CREATE TABLE report_builder.metric_toggles (
    id SERIAL PRIMARY KEY,
    toggle_key VARCHAR(100) UNIQUE NOT NULL,    -- "approval_mode", "date_basis", "refund_type"
    display_name VARCHAR(200) NOT NULL,         -- "Approval % is based on"
    description TEXT,
    toggle_type VARCHAR(50) NOT NULL,           -- "select" | "radio" | "pills"
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Options for each toggle
CREATE TABLE report_builder.metric_toggle_options (
    id SERIAL PRIMARY KEY,
    toggle_id INTEGER REFERENCES report_builder.metric_toggles(id),
    option_key VARCHAR(100) NOT NULL,           -- "standard", "organic", "net"
    display_name VARCHAR(200) NOT NULL,         -- "First Attempt Approval %"
    description TEXT,                           -- "Counts as approved only if first attempt succeeds"
    display_order INTEGER DEFAULT 0,
    is_default BOOLEAN DEFAULT false,
    UNIQUE(toggle_id, option_key)
);

-- Metric variants: different SQL per toggle option
CREATE TABLE report_builder.metric_variants (
    id SERIAL PRIMARY KEY,
    metric_id INTEGER REFERENCES report_builder.metrics(id),
    toggle_id INTEGER REFERENCES report_builder.metric_toggles(id),
    option_key VARCHAR(100) NOT NULL,           -- Which toggle option activates this variant
    sql_expression TEXT NOT NULL,               -- The SQL for this variant
    display_name_override VARCHAR(200),         -- Optional override of metric name
    UNIQUE(metric_id, toggle_id, option_key)
);
```

#### Example Data

**metric_toggles:**
| toggle_key | display_name | toggle_type |
|---|---|---|
| `approval_mode` | Approval Type | pills |
| `date_basis` | Date Type | select |
| `refund_type_filter` | Refund Type | pills |

**metric_toggle_options:**
| toggle_key | option_key | display_name | description | is_default |
|---|---|---|---|---|
| `approval_mode` | `standard` | All Approvals | Counts all approved transactions | true |
| `approval_mode` | `organic` | First Attempt | Only counts approved if first attempt succeeds. Retries ignored. | false |
| `approval_mode` | `net` | Net (After Retries) | Counts as approved if any attempt succeeds. | false |
| `date_basis` | `order_date` | Order Date | Metrics counted on the date the order was placed | true |
| `date_basis` | `cb_date` | CB/Refund Date | Metrics counted on the date the CB/refund occurred | false |
| `refund_type_filter` | `all` | All Refunds | All refund types combined | true |
| `refund_type_filter` | `alert` | Refund Alert | Alert-triggered refunds only | false |
| `refund_type_filter` | `cs` | Refund CS | Customer service refunds only | false |

**metric_variants:**
| metric_key | toggle_key | option_key | sql_expression |
|---|---|---|---|
| `approvals` | `approval_mode` | `standard` | `SUM(approvals)` |
| `approvals` | `approval_mode` | `organic` | `SUM(approvals_organic)` |
| `approvals` | `approval_mode` | `net` | `SUM(net_approvals)` |
| `cb_count` | `date_basis` | `order_date` | `SUM(cb)` |
| `cb_count` | `date_basis` | `cb_date` | `SUM(cb_cb_date)` |
| `cb_dollar` | `date_basis` | `order_date` | `SUM(cb_dollar)` |
| `cb_dollar` | `date_basis` | `cb_date` | `SUM(cb_cb_date_dollar)` |
| `refund_count` | `date_basis` | `order_date` | `SUM(refund)` |
| `refund_count` | `date_basis` | `cb_date` | `SUM(refund_refund_date)` |
| `refund_dollar` | `date_basis` | `order_date` | `SUM(refund_dollar)` |
| `refund_dollar` | `date_basis` | `cb_date` | `SUM(refund_refund_date_dollar)` |

#### How Toggles Appear in the Report Template JSON

```jsonc
{
  "filterBar": {
    "toggles": [
      {
        "toggleKey": "approval_mode",
        "position": "main",           // Show prominently
        "displayAs": "pills"           // Three pills: Standard | Organic | Net
      },
      {
        "toggleKey": "date_basis",
        "position": "main",
        "displayAs": "select"          // Dropdown: Order Date | CB/Refund Date
      }
    ]
  },
  "sections": [
    {
      "widgets": [
        {
          "type": "data_table",
          "metrics": ["approvals", "approval_rate", "cb_count", "cb_rate"],
          // These metric keys are "generic" - the toggle state determines
          // which SQL variant the backend uses
        }
      ]
    }
  ]
}
```

#### How the API Request Carries Toggle State

```jsonc
POST /api/v1/query
{
  "source": "order_summary",
  "metrics": ["approvals", "approval_rate", "cb_count", "cb_rate"],
  "dimensions": ["campaign_name"],
  "filters": {
    "dateRange": { "preset": "last_7_days" }
  },
  "toggles": {
    "approval_mode": "organic",
    "date_basis": "cb_date"
  }
}
```

#### How the Query Engine Resolves It

```
1. Metric "approvals" requested
2. Toggle state: approval_mode = "organic"
3. Check metric_variants table:
   → Found variant for (approvals, approval_mode, organic)
   → SQL: SUM(approvals_organic)
4. Use that instead of default SUM(approvals)

5. Metric "cb_count" requested
6. Toggle state: date_basis = "cb_date"
7. Check metric_variants table:
   → Found variant for (cb_count, date_basis, cb_date)
   → SQL: SUM(cb_cb_date)
8. Use that instead of default SUM(cb)
```

```javascript
// Query Engine - metric resolution with toggle support
function resolveMetricSQL(metricKey, toggleState) {
  // 1. Get base metric definition
  const metric = metricCatalog.get(metricKey);

  // 2. Check if any active toggles have variants for this metric
  for (const [toggleKey, selectedOption] of Object.entries(toggleState)) {
    const variant = metricVariants.find(v =>
      v.metric_key === metricKey &&
      v.toggle_key === toggleKey &&
      v.option_key === selectedOption
    );
    if (variant) {
      return variant.sql_expression;  // Use variant SQL
    }
  }

  // 3. No variant found, use default SQL
  return metric.sql_expression;
}
```

---

## Part 4: Complete Example — Approval Report End to End

### Step 1: Template JSON (stored in DB)

```jsonc
{
  "version": "1.0",
  "source": "order_summary",

  "filterBar": {
    "toggles": [
      {
        "toggleKey": "approval_mode",
        "position": "main",
        "displayAs": "pills"
      }
    ],
    "filters": [
      { "dimensionKey": "sales_type", "displayAs": "pills", "position": "main",
        "options": ["Initials", "Rebills"], "multiSelect": true },
      { "dimensionKey": "campaign_name", "position": "more" },
      { "dimensionKey": "product_name", "position": "more" },
      { "dimensionKey": "gateway_alias", "position": "more" },
      { "dimensionKey": "bank", "position": "more" }
    ],
    "showDateRange": true,
    "showExport": true
  },

  "sections": [
    {
      "id": "summary",
      "type": "card_row",
      "widgets": [
        {
          "id": "approval_summary",
          "type": "summary_table",
          "rows": [
            { "label": "Initials", "metrics": ["attempts_initials", "approvals_initials", "approval_rate_initials"] },
            { "label": "Rebills", "metrics": ["attempts_rebills", "approvals_rebills", "approval_rate_rebills"] },
            { "label": "Total", "metrics": ["attempts", "approvals", "approval_rate"] }
          ]
        }
      ]
    },
    {
      "id": "charts",
      "type": "grid",
      "columns": 2,
      "widgets": [
        {
          "id": "approvals_chart",
          "type": "chart",
          "chartType": "area",
          "title": "# Approvals",
          "xAxis": "date_day",
          "series": [{ "metric": "approvals" }]
        },
        {
          "id": "rate_chart",
          "type": "chart",
          "chartType": "line",
          "title": "Approval Rate",
          "xAxis": "date_day",
          "series": [{ "metric": "approval_rate", "format": "percentage" }]
        }
      ]
    },
    {
      "id": "tables",
      "type": "tabs",
      "tabs": [
        {
          "label": "By Product",
          "widget": {
            "id": "product_table",
            "type": "data_table",
            "dimension": "product_name",
            "metrics": ["attempts", "approvals", "approval_rate"],
            "metricConfig": {
              "approval_rate": { "showBar": true }
            }
          }
        },
        {
          "label": "By Campaign",
          "widget": {
            "id": "campaign_table",
            "type": "data_table",
            "dimension": "campaign_name",
            "metrics": ["attempts", "approvals", "approval_rate"],
            "metricConfig": {
              "approval_rate": { "showBar": true }
            }
          }
        },
        {
          "label": "By Acquirer",
          "widget": {
            "id": "acquirer_table",
            "type": "data_table",
            "dimension": "acquirer",
            "metrics": ["attempts", "approvals", "approval_rate"],
            "metricConfig": {
              "approval_rate": { "showBar": true }
            }
          }
        },
        {
          "label": "By Bank",
          "widget": {
            "id": "bank_table",
            "type": "data_table",
            "dimension": "bank",
            "metrics": ["attempts", "approvals", "approval_rate"],
            "metricConfig": {
              "approval_rate": { "showBar": true }
            }
          }
        }
      ]
    }
  ]
}
```

### Step 2: User Opens Report, Selects "Organic" Toggle

Frontend state becomes:
```jsonc
{
  "toggles": { "approval_mode": "organic" },
  "filters": {
    "dateRange": { "preset": "last_7_days" },
    "sales_type": { "op": "in", "values": ["Initials", "Rebills"] }
  }
}
```

### Step 3: Frontend Builds Batch Request

The `ReportRenderer` walks the template JSON, finds all widgets, and builds:

```jsonc
POST /api/v1/query/batch
{
  "toggles": { "approval_mode": "organic" },
  "queries": [
    {
      "id": "approval_summary",
      "source": "order_summary",
      "metrics": [
        "attempts_initials", "approvals_initials", "approval_rate_initials",
        "attempts_rebills", "approvals_rebills", "approval_rate_rebills",
        "attempts", "approvals", "approval_rate"
      ],
      "filters": { "dateRange": { "preset": "last_7_days" } }
    },
    {
      "id": "approvals_chart",
      "source": "order_summary",
      "metrics": ["approvals"],
      "dimensions": ["date_day"],
      "filters": { "dateRange": { "preset": "last_7_days" } }
    },
    {
      "id": "rate_chart",
      "source": "order_summary",
      "metrics": ["approval_rate"],
      "dimensions": ["date_day"],
      "filters": { "dateRange": { "preset": "last_7_days" } }
    },
    {
      "id": "product_table",
      "source": "order_summary",
      "metrics": ["attempts", "approvals", "approval_rate"],
      "dimensions": ["product_name"],
      "filters": { "dateRange": { "preset": "last_7_days" } },
      "sort": { "field": "approvals", "dir": "desc" }
    }
    // ... other tab tables loaded on tab click (lazy)
  ]
}
```

### Step 4: Backend Resolves Metrics with Toggle

For every metric in every query, the engine checks the toggle state:

```
"approvals" + toggle { approval_mode: "organic" }
  → metric_variants lookup
  → Found: (approvals, approval_mode, organic) → SUM(approvals_organic)

"approval_rate" is derived: {approvals} / {attempts}
  → resolve "approvals" → SUM(approvals_organic)  (toggle applied!)
  → resolve "attempts"  → SUM(attempts)           (no variant for this toggle)
  → final: SUM(approvals_organic) / NULLIF(SUM(attempts), 0)
```

**The toggle cascades through derived metrics automatically.** Since `approval_rate` is defined as `{approvals} / {attempts}`, and `{approvals}` resolves to the organic variant, the rate calculation automatically uses organic approvals. No special logic needed.

### Step 5: Generated SQL

```sql
-- For product_table widget:
SELECT
  p.product_name AS "product_name",
  SUM(os.attempts) AS "attempts",
  SUM(os.approvals_organic) AS "approvals",       -- organic variant!
  SUM(os.approvals_organic)::numeric
    / NULLIF(SUM(os.attempts), 0) AS "approval_rate"
FROM reporting.order_summary_10000 os
LEFT JOIN public.campaigns c
  ON c.campaign_id = os.campaign_id AND c.client_id = os.client_id
LEFT JOIN public.offers p
  ON p.product_id = os.product_id::varchar AND p.client_id = os.client_id
WHERE os.date >= CURRENT_DATE - INTERVAL '7 days'
  AND os.date <= CURRENT_DATE
GROUP BY p.product_name
ORDER BY "approvals" DESC
```

### Step 6: Frontend Renders

Each widget gets its data by `id` and renders using its `type`:

- `metric_card_row` → renders summary cards
- `chart` with `chartType: "area"` → renders area chart using Recharts/ECharts
- `data_table` → renders sortable table with TanStack Table

**User clicks "Net" toggle** → frontend updates toggle state → rebuilds queries (only toggle changes) → refetches → all widgets update. The SQL now uses `SUM(net_approvals)` instead.

---

## Part 5: All Context-Dependent Metrics Mapped

Here's every metric toggle needed across your reports:

### Toggle 1: Approval Mode

| Context | Used In | Default |
|---|---|---|
| Standard, Organic, Net | Approval %, Sales, Dashboard, CB & Refunds | Standard |

**Affected Metrics:**

| Metric | Standard | Organic | Net |
|---|---|---|---|
| `approvals` | `SUM(approvals)` | `SUM(approvals_organic)` | `SUM(net_approvals)` |
| `approval_rate` | derived from above | derived from above | derived from above |
| `initials` | `SUM(approvals) FILTER (WHERE sales_type='Initials')` | `SUM(approvals_organic) FILTER (...)` | `SUM(net_approvals) FILTER (...)` |
| `rebills` | same pattern | same pattern | same pattern |

All rate metrics (`cb_rate`, `cancel_rate`, `refund_rate`) that use approvals as denominator automatically cascade.

### Toggle 2: Date Basis (CB/Refund Date Class)

| Context | Used In | Default |
|---|---|---|
| Order Date, CB/Refund Date | CB & Refunds report | Order Date |

**Affected Metrics:**

| Metric | Order Date | CB/Refund Date |
|---|---|---|
| `cb_count` | `SUM(cb)` | `SUM(cb_cb_date)` |
| `cb_dollar` | `SUM(cb_dollar)` | `SUM(cb_cb_date_dollar)` |
| `refund_count` | `SUM(refund)` | `SUM(refund_refund_date)` |
| `refund_dollar` | `SUM(refund_dollar)` | `SUM(refund_refund_date_dollar)` |
| `cb_rate` | derived (uses cb_count) | derived (cascades) |
| `refund_rate` | derived (uses refund_count) | derived (cascades) |
| `net_revenue` | derived (uses refund_dollar + cb_dollar) | derived (cascades) |

### Toggle 3: Profitability View

| Context | Used In | Default |
|---|---|---|
| Cashflow View, Unit Economics View | Profitability | Cashflow |

This is less of a metric toggle and more of a **layout toggle** — it changes which widgets are visible. Can be handled in the template JSON:

```jsonc
{
  "conditionalVisibility": {
    "toggleKey": "profitability_view",
    "showWhen": "cashflow"
  }
}
```

### Toggle 4: Retention Base

| Context | Used In | Default |
|---|---|---|
| Initial Order Base, Approved Base | Retention | Initial Order Base |

**Affected Metrics:**

| Metric | Initial Order Base | Approved Base |
|---|---|---|
| `retention_cycle_1` | `cycle_1_approvals / initial_approvals` | `cycle_1_approvals / cycle_0_approvals` |

### Toggle 5: Time Aggregation

| Context | Used In | Default |
|---|---|---|
| Day, Week, Month | Retention, CB & Refunds, Profitability trends | Day |

This is NOT a metric variant — it changes the **dimension**:

```jsonc
{
  "toggleKey": "time_granularity",
  "type": "dimension_switch",
  "options": {
    "day": { "dimension": "date_day" },
    "week": { "dimension": "date_week" },
    "month": { "dimension": "date_month" }
  }
}
```

The frontend swaps the dimension key in the API request. The backend just groups by the requested dimension.

### Toggle 6: Dollar vs Count Display

| Context | Used In | Default |
|---|---|---|
| Show as $ or # | Various cards | Both shown |

This is a **frontend-only toggle** — it just shows/hides metrics in the UI. No backend change needed.

```jsonc
{
  "toggleKey": "display_mode",
  "type": "visibility",
  "options": {
    "count": { "showMetrics": ["approvals", "cb_count", "refund_count"] },
    "dollar": { "showMetrics": ["revenue", "cb_dollar", "refund_dollar"] },
    "both": { "showMetrics": ["all"] }
  }
}
```

---

## Part 6: Toggle Classification

Not all toggles work the same way. Here's the taxonomy:

### Type 1: Metric Variant Toggle (changes SQL column)

**Examples:** Approval Mode, Date Basis, Retention Base

```jsonc
{
  "toggleKey": "approval_mode",
  "type": "metric_variant",      // Backend resolves different SQL
  "options": [
    { "key": "standard", "label": "All Approvals" },
    { "key": "organic", "label": "First Attempt" },
    { "key": "net", "label": "After Retries" }
  ]
}
```

**How it works:**
- Frontend sends toggle state in API request
- Backend looks up metric variant → uses different SQL expression
- Derived metrics automatically cascade

### Type 2: Dimension Switch Toggle (changes GROUP BY)

**Examples:** Time Aggregation (Day/Week/Month)

```jsonc
{
  "toggleKey": "time_granularity",
  "type": "dimension_switch",     // Frontend swaps dimension key in request
  "options": [
    { "key": "day", "dimension": "date_day" },
    { "key": "week", "dimension": "date_week" },
    { "key": "month", "dimension": "date_month" }
  ]
}
```

**How it works:**
- Frontend swaps the dimension in the API request
- Backend just groups by whatever dimension is requested
- No special backend logic needed

### Type 3: Visibility Toggle (shows/hides frontend elements)

**Examples:** Dollar vs Count, Show/Hide specific widgets

```jsonc
{
  "toggleKey": "display_mode",
  "type": "visibility",          // Frontend only, no API change
  "options": [
    { "key": "count", "showMetrics": ["approvals", "cb_count"] },
    { "key": "dollar", "showMetrics": ["revenue", "cb_dollar"] }
  ]
}
```

**How it works:**
- Pure frontend — hides/shows metrics in the rendered widgets
- Backend always fetches all metrics (or frontend can optimize by only requesting visible ones)

### Type 4: Layout Toggle (shows different widget sections)

**Examples:** Profitability View (Cashflow vs Unit Economics), Report tabs

```jsonc
{
  "toggleKey": "profitability_view",
  "type": "layout_switch",       // Frontend shows different sections
  "options": [
    { "key": "cashflow", "showSections": ["cashflow_summary", "cashflow_table"] },
    { "key": "unit_economics", "showSections": ["unit_summary", "unit_table"] }
  ]
}
```

**How it works:**
- Frontend shows/hides entire report sections
- Different sections may need different API queries (lazy loaded on switch)

### Type 5: Parameter Input (user-provided values for calculations)

**Examples:** Processing Fee %, Reserve %, CB Fee $, CPA $

```jsonc
{
  "toggleKey": "processing_fee_pct",
  "type": "parameter_input",      // User enters a value used in calculations
  "inputType": "number",
  "suffix": "%",
  "default": 3.5,
  "min": 0,
  "max": 100
}
```

**How it works:**
- Frontend sends parameter values in the API request
- Backend uses them in metric calculations: `SUM(revenue) * {processing_fee_pct} / 100`

API request with parameters:
```jsonc
{
  "source": "order_summary",
  "metrics": ["revenue", "processing_fees", "gross_profit"],
  "parameters": {
    "processing_fee_pct": 3.5,
    "reserve_pct": 5.0,
    "cb_fee_amount": 25.00,
    "cpa_amount": 45.00
  }
}
```

---

## Part 7: Summary - How Everything Connects

```
┌──────────────────────────────────────────────────────────────┐
│                     TEMPLATE JSON                             │
│                                                               │
│  Defines: layout, widgets, toggles, filters, metrics,         │
│           dimensions, chart types, conditional visibility      │
│                                                               │
│  Does NOT contain: SQL, API urls, data                        │
└─────────────────────────┬────────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
      ┌───────▼────────┐     ┌───────▼────────┐
      │    FRONTEND     │     │    BACKEND      │
      │                 │     │                 │
      │ Reads template  │     │ Receives:       │
      │ Renders widgets │     │  - metric keys  │
      │ Manages toggles │     │  - dimensions   │
      │ Builds queries  │     │  - filters      │
      │ from widget     │     │  - toggles      │
      │ configs         │     │  - parameters   │
      │                 │     │                 │
      │ Sends:          │     │ Resolves:       │
      │  metric keys    │     │  - metric SQL   │
      │  + toggle state │     │  - variants     │
      │  + filters      │     │  - joins        │
      │  → ONE endpoint │     │  - builds SQL   │
      │                 │     │  - executes     │
      └────────────────┘     └────────────────┘
```

### What the Frontend Knows:
- Template JSON (layout + widget configs)
- Metric keys (not SQL, not column names)
- Dimension keys (not SQL, not column names)
- Current toggle state
- Current filter state

### What the Backend Knows:
- Metric catalog (metric key → SQL expression + variants)
- Dimension catalog (dimension key → column + joins)
- Data source catalog (source name → actual table)
- How to assemble SQL from these pieces

### What JSON Controls:
- Which metrics appear on which report
- How they're laid out (cards, charts, tables)
- Which toggles are available per report
- Which filters show per report
- Chart types and configurations
- Conditional visibility rules
- Default values for everything

### What Requires Code Changes:
- New widget type (e.g., "venn_diagram") → new React component
- New toggle type (e.g., "parameter_input") → new query engine handler
- New data source type → new source resolver
- New chart type → new chart component

### What Does NOT Require Code Changes:
- New metric → insert row in metric catalog
- New dimension → insert row in dimension catalog
- New report → insert JSON template in DB
- New metric variant → insert row in metric_variants
- New toggle → insert rows in metric_toggles + options
- Change report layout → update JSON
- Add filter to report → update JSON
- Change default date range → update JSON
- New client → create materialized views (existing ETL), assign report templates

---

## Part 8: Handling Complex Scenarios

### Scenario 1: Metric Needs Different Data Source

Some metrics come from different materialized views. Example:
- `approvals` comes from `order_summary`
- `organic_declines` comes from `decline_recovery`
- `alert_count` comes from `alert_summary`

**Solution:** Each widget specifies its `source`. A report can have widgets pulling from different sources.

```jsonc
{
  "sections": [
    {
      "widgets": [
        {
          "id": "sales_summary",
          "source": "order_summary",        // This widget queries order_summary
          "metrics": ["approvals", "revenue"]
        },
        {
          "id": "recovery_summary",
          "source": "decline_recovery",     // This widget queries decline_recovery
          "metrics": ["organic_declines", "recovered", "recovery_rate"]
        }
      ]
    }
  ]
}
```

The batch query handles multiple sources in one request — they just become separate SQL queries executed in parallel.

### Scenario 2: Cohort / Pivot Table

LTV and Retention reports need pivot/cohort tables where columns are dynamic (Month 0, Month 1, Month 2...).

**Solution:** A `pivot_table` widget type with special config:

```jsonc
{
  "type": "pivot_table",
  "source": "cohort_summary",
  "rowDimension": "date_month",            // Cohort month
  "columnDimension": "billing_cycle",       // Becomes columns: Cycle 0, 1, 2, 3...
  "valueMetric": "approvals",              // What goes in the cells
  "displayAs": "percentage",               // Show as % of initial
  "baseMetric": "initials",               // Base for percentage calc
  "maxColumns": 12                         // Limit columns
}
```

The backend query for this is different — it returns unpivoted data, and the frontend pivots it:

```sql
SELECT
  DATE_TRUNC('month', date) AS cohort_month,
  billing_cycle,
  SUM(approvals) AS value,
  SUM(CASE WHEN billing_cycle = 0 THEN approvals END) OVER (PARTITION BY DATE_TRUNC('month', date)) AS base
FROM reporting.cohort_summary_10000
WHERE date >= '2025-07-01'
GROUP BY DATE_TRUNC('month', date), billing_cycle
ORDER BY cohort_month, billing_cycle
```

Frontend transforms this into a matrix.

### Scenario 3: MID Performance (Monthly Snapshot, Not Date Range)

MID Performance uses `mid_summary` which is keyed by `month_year` not a date range. The filter works differently.

**Solution:** Source-level config in the template:

```jsonc
{
  "source": "mid_summary",
  "sourceConfig": {
    "dateFilterType": "month_selector",    // Shows month picker, not date range
    "dateColumn": "month_year",
    "dateFormat": "MMM YYYY"               // "Jan 2026"
  }
}
```

The query engine sees `dateFilterType: "month_selector"` and generates:
```sql
WHERE month_year = 'Jan 2026'
```
instead of the usual date range `WHERE date BETWEEN ... AND ...`

### Scenario 4: Real-time Dashboard (Hourly Revenue)

The Dashboard's "Today's Performance" section needs `hourly_revenue` which has a different structure.

**Solution:** Just another source:

```jsonc
{
  "id": "today_performance",
  "source": "hourly_revenue",
  "type": "chart",
  "chartType": "area",
  "xAxis": "hour",
  "series": [
    { "metric": "today_revenue", "label": "Today" },
    { "metric": "avg_7d_revenue", "label": "7-Day Avg", "style": "dashed" }
  ],
  "autoRefresh": 300                    // Refresh every 5 minutes
}
```

### Scenario 5: Profitability with User Input Parameters

The Profitability report has sliders for Processing Fee %, Reserve %, CB Fee, CPA.

```jsonc
{
  "filterBar": {
    "parameters": [
      { "key": "processing_fee_pct", "label": "Processing Fees", "type": "number",
        "suffix": "%", "default": 3.5, "step": 0.1 },
      { "key": "reserve_pct", "label": "Reserve", "type": "number",
        "suffix": "%", "default": 5.0, "step": 0.5 },
      { "key": "cb_fee_amount", "label": "CB Fees", "type": "number",
        "prefix": "$", "default": 25.00, "step": 1.0 },
      { "key": "cpa_amount", "label": "CPA", "type": "number",
        "prefix": "$", "default": 45.00, "step": 1.0 }
    ]
  }
}
```

These values are sent in the API request under `parameters` and the backend uses them in metric calculations:

```javascript
// In metric catalog:
{
  metric_key: "processing_fees",
  sql_expression: "SUM(revenue) * :processing_fee_pct / 100"
  // :processing_fee_pct is replaced from request parameters
}
```

---

## Part 9: What Needs to Be Code vs What's JSON

### Built Once as Code (Reusable Components)

| Component | Type | Description |
|---|---|---|
| Query Engine | Backend | Resolves metrics, dimensions, toggles → SQL |
| Batch Query Executor | Backend | Runs multiple queries in parallel |
| Cache Layer | Backend | Redis cache with smart invalidation |
| ReportRenderer | Frontend | Reads JSON template, orchestrates widgets |
| MetricCard | Frontend | Renders a summary metric card |
| Chart | Frontend | Renders any chart type via config |
| DataTable | Frontend | Renders sortable/paginated table |
| PivotTable | Frontend | Renders cohort/retention matrices |
| FilterBar | Frontend | Renders filters, toggles, date pickers |
| DashboardGrid | Frontend | Drag-and-drop widget layout |
| ReportBuilder | Frontend | UI for creating report templates |

### Managed via JSON / DB (No Code Needed)

| Item | Where Stored | Changed How |
|---|---|---|
| Metric definitions | `report_builder.metrics` table | INSERT/UPDATE SQL |
| Metric variants | `report_builder.metric_variants` table | INSERT SQL |
| Dimension definitions | `report_builder.dimensions` table | INSERT/UPDATE SQL |
| Toggle definitions | `report_builder.metric_toggles` table | INSERT SQL |
| Report templates | `report_builder.report_templates` JSONB | Edit JSON |
| Saved views | `report_builder.saved_views` JSONB | User action |
| Filter presets | `report_builder.filter_presets` JSONB | User action |
| Dashboard layouts | `report_builder.dashboards` JSONB | User action |
| Navigation structure | DB table | UPDATE SQL |

### When You DO Need Code Changes

1. **New widget type** (e.g., "gauge", "treemap") → New React component, ~1-2 days
2. **New data source type** (e.g., external API) → New source resolver, ~1 day
3. **New toggle type** (e.g., "color_picker") → New frontend component + backend handler, ~1 day
4. **New export format** (e.g., Google Sheets) → New export handler, ~2-3 days
5. **New chart type** → Usually just config in ECharts/Recharts, often zero code

### What This Means Practically

- **New client onboarded:** Zero code changes. Create materialized views (existing ETL), assign report templates.
- **New metric added for all clients:** One INSERT into metrics table. If it uses new columns, also update materialized views.
- **Client wants a custom report:** Create a JSON template, insert into DB for that client. No deployment.
- **Change report layout for everyone:** Update the JSON in report_templates. No deployment.
- **User wants their own report:** They use the Report Builder UI. Zero involvement from your team.

---

*Document generated: 2026-01-26*
*Technical design for JSON-driven report system*

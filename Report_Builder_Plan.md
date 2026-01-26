# Beast Insights - Dynamic Report Builder Plan

## Table of Contents

1. [Goal](#1-goal)
2. [Approach Options](#2-approach-options)
3. [Recommended Architecture](#3-recommended-architecture)
4. [Data Model Design](#4-data-model-design)
5. [Metric Catalog Design](#5-metric-catalog-design)
6. [Dimension Catalog Design](#6-dimension-catalog-design)
7. [Report Template System](#7-report-template-system)
8. [Widget System](#8-widget-system)
9. [Filter System](#9-filter-system)
10. [Query Engine](#10-query-engine)
11. [User Customization Features](#11-user-customization-features)
12. [API Design](#12-api-design)
13. [Frontend Architecture](#13-frontend-architecture)
14. [Performance Strategy](#14-performance-strategy)
15. [Migration Path](#15-migration-path)
16. [Possibilities & Advanced Features](#16-possibilities--advanced-features)

---

## 1. Goal

Replace Power BI embedded reports with a native React reporting system that:

- Loads in <1 second (vs current 5-10 seconds)
- Provides pre-built "stock" reports (matching current 13 reports)
- Allows users to customize existing reports (add/remove metrics, change dimensions, save views)
- Allows users to create entirely new reports from a metrics/dimensions catalog
- Is managed dynamically via database and JSON (no code changes for new reports)
- Supports saved views, saved filters, scheduled exports, and personal dashboards

---

## 2. Approach Options

### Option A: Pure JSON Report Definitions

**How it works:** Each report is a JSON document stored in the database. The JSON defines layout (rows/columns of widgets), each widget's type (card, chart, table), the metrics it displays, and its filter bindings. The frontend reads the JSON and renders accordingly.

**Pros:**
- Simple to understand and implement
- Easy to version, clone, and share report definitions
- Frontend is purely declarative

**Cons:**
- JSON alone doesn't capture how to compute metrics - you'd still need a metric registry
- Complex reports with conditional logic are hard to express in pure JSON
- No standard for this, so you build everything from scratch

---

### Option B: Semantic Layer + JSON Templates

**How it works:** Build a **semantic layer** in the database that defines all available metrics (with their SQL expressions), dimensions (with their column mappings), and relationships. Then JSON templates reference metrics/dimensions by ID. A query engine translates user selections into SQL queries against the materialized views.

**Pros:**
- Clean separation: semantic layer knows *how to query*, JSON knows *what to show*
- Adding new metrics = inserting a row in the DB, no code changes
- Users can build arbitrary reports by picking from the catalog
- Same metric definition used everywhere (consistency)
- Closest to what VRIO and other BI tools do

**Cons:**
- More upfront design work for the semantic layer
- Need a robust query builder / SQL generator
- Need to handle edge cases (metrics that need different views, cross-view joins)

---

### Option C: Pre-built API Endpoints + JSON Layout

**How it works:** Each report has dedicated backend API endpoints that return pre-shaped data. JSON templates only control layout and which endpoints to call. The backend has hardcoded query logic per report type.

**Pros:**
- Fastest to build initially
- Full control over each query's performance
- Easy to optimize specific reports

**Cons:**
- Adding new metrics/reports requires backend code changes
- Not truly dynamic or user-customizable
- Doesn't scale for "users create their own reports"
- Essentially recreates the Power BI rigidity in your own code

---

### Option D: Embed an Open-Source BI Tool (Apache Superset, Metabase, etc.)

**How it works:** Deploy Superset or Metabase, connect it to your PostgreSQL, and embed it in your React app via iframe or SDK.

**Pros:**
- Massive feature set out of the box
- Community maintained
- Charts, dashboards, SQL explorer, saved views all built-in

**Cons:**
- Heavy infrastructure (Superset needs Python/Redis/Celery)
- Embedding experience is often janky (iframes, styling conflicts)
- Limited control over UX - won't match your brand/design
- Users see a generic BI tool, not "Beast Insights"
- You trade Power BI dependency for Superset/Metabase dependency
- Customizing their internals is painful

---

### Recommendation: **Option B (Semantic Layer + JSON Templates)**

This gives you the best balance of:
- Dynamic, no-code report creation
- User self-service report building
- Clean architecture that scales
- Full control over UX and performance
- The semantic layer becomes your "product moat" - it understands your payment analytics domain

---

## 3. Recommended Architecture

```
                    ┌─────────────────────────────────────┐
                    │           React Frontend             │
                    │  ┌──────────┐  ┌──────────────────┐ │
                    │  │ Report   │  │ Report Builder    │ │
                    │  │ Renderer │  │ (Drag & Drop)     │ │
                    │  └────┬─────┘  └────────┬──────────┘ │
                    │       │                 │            │
                    └───────┼─────────────────┼────────────┘
                            │                 │
                    ┌───────▼─────────────────▼────────────┐
                    │           Node.js API                 │
                    │  ┌──────────────────────────────────┐│
                    │  │     Query Engine                  ││
                    │  │  (Semantic Layer → SQL Generator) ││
                    │  └──────────────┬───────────────────┘│
                    │  ┌──────────────┴───────────────────┐│
                    │  │     Cache Layer (Redis)           ││
                    │  └──────────────┬───────────────────┘│
                    └─────────────────┼────────────────────┘
                                      │
                    ┌─────────────────▼────────────────────┐
                    │         PostgreSQL                    │
                    │  ┌────────────┐ ┌──────────────────┐ │
                    │  │ Semantic   │ │ Materialized     │ │
                    │  │ Layer      │ │ Views            │ │
                    │  │ (configs)  │ │ (reporting.*)    │ │
                    │  └────────────┘ └──────────────────┘ │
                    └──────────────────────────────────────┘
```

### Core Components:

1. **Semantic Layer (DB)** - Metric definitions, dimension definitions, data source mappings
2. **Report Template Store (DB + JSON)** - Report definitions as JSON, stored in DB
3. **Query Engine (Node.js)** - Translates metric/dimension/filter selections into SQL
4. **Cache Layer (Redis)** - Caches query results for repeated access
5. **Report Renderer (React)** - Reads JSON template, fetches data from API, renders widgets
6. **Report Builder (React)** - UI for creating/editing report templates

---

## 4. Data Model Design

### New Tables Required

```sql
-- ============================================================
-- SEMANTIC LAYER
-- ============================================================

-- Data sources: which materialized views/tables are available
CREATE TABLE report_builder.data_sources (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,           -- e.g., "order_summary"
    display_name VARCHAR(200),            -- e.g., "Order Summary"
    description TEXT,
    source_type VARCHAR(50) NOT NULL,     -- "materialized_view" | "table" | "custom_query"
    schema_name VARCHAR(100),             -- e.g., "reporting"
    table_pattern VARCHAR(200),           -- e.g., "order_summary_{client_id}"
    base_query TEXT,                      -- For custom_query type, the base SQL
    refresh_frequency VARCHAR(50),        -- "hourly", "daily", "realtime"
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Metrics catalog: every available metric
CREATE TABLE report_builder.metrics (
    id SERIAL PRIMARY KEY,
    metric_key VARCHAR(100) UNIQUE NOT NULL,   -- e.g., "approvals_count"
    display_name VARCHAR(200) NOT NULL,        -- e.g., "# Approvals"
    description TEXT,                          -- Tooltip text
    category VARCHAR(100),                     -- "Sales", "Processing", "Disputes", etc.
    sub_category VARCHAR(100),                 -- "Count", "Dollar", "Percentage"
    data_type VARCHAR(50) NOT NULL,            -- "integer", "decimal", "percentage", "currency"
    format_pattern VARCHAR(100),               -- "#,##0", "$#,##0.00", "0.00%"
    decimal_places INTEGER DEFAULT 0,
    data_source_id INTEGER REFERENCES report_builder.data_sources(id),
    sql_expression TEXT NOT NULL,              -- SQL to compute, e.g., "SUM(approvals)"
    -- For derived metrics that use other metrics
    is_derived BOOLEAN DEFAULT false,
    derived_formula TEXT,                      -- e.g., "{approvals_count} / {attempts_count}"
    derived_metric_ids INTEGER[],              -- References to component metrics
    -- Aggregation info
    aggregation_type VARCHAR(50),              -- "sum", "avg", "count", "max", "min", "custom"
    -- Conditional metrics
    condition_sql TEXT,                        -- e.g., "sales_type = 'Initials'"
    -- Comparison support
    supports_comparison BOOLEAN DEFAULT true,
    supports_trend BOOLEAN DEFAULT true,
    -- Sorting
    default_sort_direction VARCHAR(4) DEFAULT 'DESC',
    -- Visibility
    is_active BOOLEAN DEFAULT true,
    is_default BOOLEAN DEFAULT false,         -- Show by default in category
    display_order INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Dimensions catalog: every available dimension for grouping
CREATE TABLE report_builder.dimensions (
    id SERIAL PRIMARY KEY,
    dimension_key VARCHAR(100) UNIQUE NOT NULL,  -- e.g., "campaign_name"
    display_name VARCHAR(200) NOT NULL,          -- e.g., "Campaign"
    description TEXT,
    category VARCHAR(100),                       -- "Marketing", "Payment", "Product", etc.
    data_type VARCHAR(50),                       -- "string", "number", "date"
    data_source_id INTEGER REFERENCES report_builder.data_sources(id),
    column_expression TEXT NOT NULL,             -- SQL column, e.g., "c.campaign_name"
    -- Join info (if dimension comes from a join)
    join_table VARCHAR(200),                     -- e.g., "public.campaigns"
    join_alias VARCHAR(50),                      -- e.g., "c"
    join_condition TEXT,                         -- e.g., "c.campaign_id = os.campaign_id AND c.client_id = os.client_id"
    -- For hierarchical dimensions
    parent_dimension_id INTEGER REFERENCES report_builder.dimensions(id),
    -- For date dimensions
    is_date_dimension BOOLEAN DEFAULT false,
    date_granularity VARCHAR(50),               -- "day", "week", "month", "quarter", "year", "hour"
    date_column VARCHAR(100),                   -- Source date column
    -- Filter support
    is_filterable BOOLEAN DEFAULT true,
    filter_query TEXT,                          -- SQL to get distinct values for filter dropdown
    -- Visibility
    is_active BOOLEAN DEFAULT true,
    display_order INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Which metrics are available from which data sources
CREATE TABLE report_builder.data_source_metrics (
    id SERIAL PRIMARY KEY,
    data_source_id INTEGER REFERENCES report_builder.data_sources(id),
    metric_id INTEGER REFERENCES report_builder.metrics(id),
    is_active BOOLEAN DEFAULT true,
    UNIQUE(data_source_id, metric_id)
);

-- Which dimensions are available from which data sources
CREATE TABLE report_builder.data_source_dimensions (
    id SERIAL PRIMARY KEY,
    data_source_id INTEGER REFERENCES report_builder.data_sources(id),
    dimension_id INTEGER REFERENCES report_builder.dimensions(id),
    is_active BOOLEAN DEFAULT true,
    UNIQUE(data_source_id, dimension_id)
);


-- ============================================================
-- REPORT TEMPLATES
-- ============================================================

-- Report templates (both stock and user-created)
CREATE TABLE report_builder.report_templates (
    id SERIAL PRIMARY KEY,
    template_key VARCHAR(100),                -- e.g., "sales_report", null for user-created
    name VARCHAR(200) NOT NULL,
    description TEXT,
    category VARCHAR(100),                    -- "Analytics", "Processing", "Profits", etc.
    icon VARCHAR(100),
    template_type VARCHAR(50) NOT NULL,       -- "stock" | "custom" | "user_created"
    -- JSON definition of the full report layout
    layout JSONB NOT NULL,                    -- See Report Template JSON Schema below
    -- Default filters
    default_filters JSONB,
    -- Default date range
    default_date_range VARCHAR(50),           -- "last_7_days", "last_30_days", etc.
    -- Data source requirements
    required_data_sources INTEGER[],          -- Which data sources this report needs
    -- Permissions
    required_package_level INTEGER DEFAULT 0,
    -- Ownership
    created_by_user_id INTEGER,               -- null = system template
    client_id INTEGER,                        -- null = global template
    is_public BOOLEAN DEFAULT false,          -- Can other users see this?
    is_active BOOLEAN DEFAULT true,
    display_order INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Saved views: user customizations on top of a template
CREATE TABLE report_builder.saved_views (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    report_template_id INTEGER REFERENCES report_builder.report_templates(id),
    user_id INTEGER NOT NULL,
    client_id INTEGER NOT NULL,
    -- What the user customized (overrides template defaults)
    custom_filters JSONB,                    -- Saved filter state
    custom_metrics JSONB,                    -- Which metrics shown/hidden
    custom_dimensions JSONB,                 -- Which dimensions selected
    custom_sort JSONB,                       -- Sort configuration
    custom_layout JSONB,                     -- Layout overrides (widget sizes, positions)
    custom_date_range VARCHAR(50),
    -- Quick access
    is_pinned BOOLEAN DEFAULT false,
    is_default BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Filter presets (rulesets)
CREATE TABLE report_builder.filter_presets (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    filters JSONB NOT NULL,                  -- Saved filter combination
    user_id INTEGER,                         -- null = global preset
    client_id INTEGER,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);


-- ============================================================
-- USER DASHBOARDS
-- ============================================================

-- User dashboards (personal landing pages)
CREATE TABLE report_builder.dashboards (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    user_id INTEGER NOT NULL,
    client_id INTEGER NOT NULL,
    layout JSONB NOT NULL,                   -- Grid layout of dashboard widgets
    is_default BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Dashboard widgets (pinned from reports)
CREATE TABLE report_builder.dashboard_widgets (
    id SERIAL PRIMARY KEY,
    dashboard_id INTEGER REFERENCES report_builder.dashboards(id),
    widget_type VARCHAR(50) NOT NULL,        -- "metric_card", "chart", "table", "counter"
    title VARCHAR(200),
    -- What data to show
    data_source_id INTEGER,
    metrics JSONB,                           -- Metric IDs and config
    dimensions JSONB,                        -- Dimension config
    filters JSONB,                           -- Applied filters
    date_range VARCHAR(50),
    -- Visual config
    chart_type VARCHAR(50),                  -- "line", "bar", "pie", "area", etc.
    widget_config JSONB,                     -- Colors, labels, formatting options
    -- Layout position
    grid_x INTEGER,
    grid_y INTEGER,
    grid_w INTEGER,
    grid_h INTEGER,
    -- Source
    source_report_id INTEGER,
    source_widget_key VARCHAR(100),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);


-- ============================================================
-- SCHEDULING & EXPORTS
-- ============================================================

CREATE TABLE report_builder.scheduled_reports (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    report_template_id INTEGER,
    saved_view_id INTEGER,
    user_id INTEGER NOT NULL,
    client_id INTEGER NOT NULL,
    -- Schedule
    frequency VARCHAR(50) NOT NULL,          -- "daily", "weekly", "monthly"
    day_of_week INTEGER,                     -- 0-6 for weekly
    day_of_month INTEGER,                    -- 1-31 for monthly
    time_of_day TIME,
    timezone VARCHAR(100),
    -- Delivery
    delivery_method VARCHAR(50),             -- "email", "slack", "telegram"
    delivery_config JSONB,                   -- Email addresses, channel IDs, etc.
    export_format VARCHAR(50),               -- "pdf", "csv", "excel"
    -- Filters
    filters JSONB,
    date_range VARCHAR(50),
    is_active BOOLEAN DEFAULT true,
    last_sent_at TIMESTAMP,
    next_send_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

---

## 5. Metric Catalog Design

### Metric Categories & Examples

Below is the full metric catalog needed to match and exceed current Power BI reports. Each metric maps to an SQL expression against your materialized views.

### 5.1 Sales Metrics

| Metric Key | Display Name | SQL Expression | Data Source | Type |
|---|---|---|---|---|
| `attempts_total` | # Total Attempts | `SUM(attempts_total)` | order_summary | integer |
| `attempts` | # Attempts | `SUM(attempts)` | order_summary | integer |
| `approvals` | # Approvals | `SUM(approvals)` | order_summary | integer |
| `net_approvals` | # Net Approvals | `SUM(net_approvals)` | order_summary | integer |
| `organic_approvals` | # Organic Approvals | `SUM(approvals_organic)` | order_summary | integer |
| `cancels` | # Cancels | `SUM(cancel)` | order_summary | integer |
| `initials` | # Initials | `SUM(approvals) FILTER (WHERE sales_type = 'Initials')` | order_summary | integer |
| `rebills` | # Rebills | `SUM(approvals) FILTER (WHERE sales_type = 'Rebills')` | order_summary | integer |
| `straight_sales` | # Straight Sales | `SUM(approvals) FILTER (WHERE sales_type = 'Straight Sales')` | order_summary | integer |

### 5.2 Revenue Metrics

| Metric Key | Display Name | SQL Expression | Type |
|---|---|---|---|
| `revenue` | $ Gross Revenue | `SUM(revenue)` | currency |
| `revenue_initials` | $ Initials Revenue | `SUM(revenue) FILTER (WHERE sales_type = 'Initials')` | currency |
| `revenue_rebills` | $ Rebills Revenue | `SUM(revenue) FILTER (WHERE sales_type = 'Rebills')` | currency |
| `net_revenue` | $ Net Revenue | `SUM(revenue) - SUM(refund_dollar) - SUM(cb_dollar)` | currency |
| `aov` | AOV | `SUM(revenue) / NULLIF(SUM(approvals), 0)` | currency |
| `aov_initials` | AOV (Initials) | `SUM(revenue) FILTER (WHERE sales_type='Initials') / NULLIF(SUM(approvals) FILTER (WHERE sales_type='Initials'), 0)` | currency |

### 5.3 Rate Metrics (Derived)

| Metric Key | Display Name | Formula | Type |
|---|---|---|---|
| `approval_rate` | Approval % | `{approvals} / {attempts}` | percentage |
| `net_approval_rate` | Net Approval % | `{net_approvals} / {attempts}` | percentage |
| `organic_approval_rate` | First Attempt Approval % | `{organic_approvals} / {attempts}` | percentage |
| `cancel_rate` | Cancel % | `{cancels} / {approvals}` | percentage |
| `cb_rate` | CB % | `{cb_count} / {approvals}` | percentage |
| `refund_rate` | Refund % | `{refund_count} / {approvals}` | percentage |
| `refund_alert_rate` | Refund Alert % | `{refund_alert_count} / {approvals}` | percentage |
| `refund_cs_rate` | Refund CS % | `{refund_cs_count} / {approvals}` | percentage |

### 5.4 Chargeback & Refund Metrics

| Metric Key | Display Name | SQL Expression | Type |
|---|---|---|---|
| `cb_count` | # Chargebacks | `SUM(cb)` | integer |
| `cb_dollar` | $ Chargebacks | `SUM(cb_dollar)` | currency |
| `cb_by_cb_date` | # CB (by CB Date) | `SUM(cb_cb_date)` | integer |
| `cb_dollar_by_cb_date` | $ CB (by CB Date) | `SUM(cb_cb_date_dollar)` | currency |
| `refund_count` | # Refunds | `SUM(refund)` | integer |
| `refund_dollar` | $ Refunds | `SUM(refund_dollar)` | currency |
| `refund_by_refund_date` | # Refunds (by Refund Date) | `SUM(refund_refund_date)` | integer |

### 5.5 Profitability Metrics (Parameter-Dependent)

| Metric Key | Display Name | Formula | Type |
|---|---|---|---|
| `processing_fees` | $ Processing Fees | `{revenue} * {param:processing_fee_pct} / 100` | currency |
| `cb_fees` | $ CB Fees | `{cb_count} * {param:cb_fee_amount}` | currency |
| `reserve` | $ Reserve | `{revenue} * {param:reserve_pct} / 100` | currency |
| `product_cost` | $ Product Cost | _requires join to products for per-product cost_ | currency |
| `cpa_cost` | $ CPA Cost | `SUM(cpa * approvals) FILTER (WHERE sales_type = 'Initials')` | currency |
| `alert_cost` | $ Alert Cost | _sum of RDR + Ethoca + CDRN alert costs_ | currency |
| `gross_profit` | $ Gross Profit | `{revenue} - {refund_dollar} - {cb_dollar} - {processing_fees} - {cb_fees} - {product_cost} - {cpa_cost} - {alert_cost} - {reserve}` | currency |
| `gross_margin` | Gross Margin % | `{gross_profit} / {revenue}` | percentage |

### 5.6 Decline Recovery Metrics

| Metric Key | Display Name | SQL Expression (decline_recovery view) | Type |
|---|---|---|---|
| `organic_declines` | # Organic Declines | `SUM(organic_declines)` | integer |
| `declines` | # Declines | `SUM(declines)` | integer |
| `reattempts` | # Reattempts | `SUM(reattempts)` | integer |
| `recovered` | # Recovered | `SUM(recovered)` | integer |
| `recovery_rate` | Recovery Rate | `SUM(recovered) / NULLIF(SUM(organic_declines), 0)` | percentage |
| `recovered_dollar` | $ Recovered Revenue | `SUM(recovered_dollar)` | currency |
| `missed_recovery` | $ Missed Recovery | `SUM(organic_declines_dollar) - SUM(recovered_dollar)` | currency |

### 5.7 MID Health Metrics

| Metric Key | Display Name | Column (mid_summary view) | Type |
|---|---|---|---|
| `mid_volume` | $ Volume | `volume` | currency |
| `mid_capacity_left` | $ Capacity Left | `capacity_left` | currency |
| `mid_cb_rate` | CB Rate | `cb_rate` | percentage |
| `mid_cb_visa_rate` | Visa CB Rate | `cb_visa_rate` | percentage |
| `mid_cb_master_rate` | MC CB Rate | `cb_master_rate` | percentage |
| `mid_decline_rate` | Decline Rate | `decline_rate` | percentage |
| `mid_alert_rate` | Alert Rate | `alert_rate` | percentage |
| `mid_health_tag` | Health Tag | `health_tag` | string |

### 5.8 Alert Metrics

| Metric Key | Display Name | SQL Expression (alert_summary view) | Type |
|---|---|---|---|
| `alert_count` | # Alerts | `SUM(alert_count)` | integer |
| `alert_dollar` | $ Alerts | `SUM(alert_dollar)` | currency |
| `rdr_count` | # RDR | `SUM(rdr)` | integer |
| `ethoca_count` | # Ethoca | `SUM(ethoca)` | integer |
| `cdrn_count` | # CDRN | `SUM(cdrn)` | integer |
| `rdr_effective_pct` | RDR Effectiveness % | _from alert analysis_ | percentage |
| `ethoca_effective_pct` | Ethoca Effectiveness % | _from alert analysis_ | percentage |

### 5.9 Cohort / LTV / Retention Metrics

| Metric Key | Display Name | Description | Type |
|---|---|---|---|
| `cohort_customers` | New Customers | Count of unique initial customers in cohort | integer |
| `cohort_revenue_m0` | First Order $ | Revenue at month 0 | currency |
| `cohort_revenue_m1` | Month 1 $ | Revenue at month 1 since first order | currency |
| `cohort_revenue_m2` - `m12` | Month 2-12 $ | Revenue at each subsequent month | currency |
| `retention_cycle_1` - `cycle_6` | Cycle 1-6 % | Retention percentage per billing cycle | percentage |
| `ltv_30d` | 30-Day LTV | Cumulative revenue per customer at 30 days | currency |
| `ltv_60d` | 60-Day LTV | Cumulative revenue per customer at 60 days | currency |
| `ltv_90d` | 90-Day LTV | Cumulative revenue per customer at 90 days | currency |

**Total: ~80-100 metrics** (expandable to 200+ as new views and data points are added).

---

## 6. Dimension Catalog Design

### Available Dimensions by Category

#### Marketing Dimensions
| Dimension Key | Display Name | Source Table | Column |
|---|---|---|---|
| `campaign_name` | Campaign | campaigns | campaign_name |
| `campaign_type` | Campaign Type | campaigns | campaign_type |
| `campaign_id` | Campaign ID | order_summary | campaign_id |
| `affid` | Affiliate ID | order_summary | affid |
| `sub_affid` | Sub Affiliate ID | order_summary | sub_affid |
| `c` | Affiliate Tier 3 | order_summary | c |

#### Product Dimensions
| Dimension Key | Display Name | Source Table | Column |
|---|---|---|---|
| `product_name` | Product | products | product_name |
| `product_id` | Product ID | order_summary | product_id |
| `product_group` | Product Group | products | product_group |

#### Payment Dimensions
| Dimension Key | Display Name | Source Table | Column |
|---|---|---|---|
| `gateway_id` | Gateway ID | order_summary | gateway_id |
| `gateway_alias` | Gateway Alias | payments | gateway_alias |
| `acquirer` | Acquirer | payments | lender |
| `mid_corp` | MID Corp | payments | corp |
| `mcc` | MCC | payments | mcc |

#### Card Dimensions
| Dimension Key | Display Name | Source Table | Column |
|---|---|---|---|
| `bin` | BIN | order_summary | bin |
| `bank` | Bank | bin_lookup | bank |
| `card_brand` | Card Brand | bin_lookup | card |
| `card_type` | Card Type | bin_lookup | type |
| `card_class` | Card Class | bin_lookup | class |
| `card_country` | Card Country | bin_lookup | country |

#### Transaction Dimensions
| Dimension Key | Display Name | Source Table | Column |
|---|---|---|---|
| `sales_type` | Sales Type | order_summary | sales_type |
| `billing_cycle` | Billing Cycle | order_summary | billing_cycle |
| `price_point` | Price Point | order_summary | price_point |
| `refund_type` | Refund Type | order_summary | refund_type |
| `alert_type` | Alert Type | order_summary | alert_type |
| `decline_group` | Decline Group | decline_recovery | decline_group |

#### Time Dimensions
| Dimension Key | Display Name | Expression | Granularity |
|---|---|---|---|
| `date_day` | Day | `date` | day |
| `date_week` | Week | `DATE_TRUNC('week', date)` | week |
| `date_month` | Month | `DATE_TRUNC('month', date)` | month |
| `date_quarter` | Quarter | `DATE_TRUNC('quarter', date)` | quarter |
| `date_year` | Year | `DATE_TRUNC('year', date)` | year |
| `date_day_of_week` | Day of Week | `EXTRACT(DOW FROM date)` | - |
| `date_hour` | Hour | `hour` (for hourly_revenue) | hour |

---

## 7. Report Template System

### 7.1 Template JSON Schema

Each report template is stored as a JSONB `layout` column. Here is the schema:

```jsonc
{
  // Report-level metadata
  "version": "1.0",
  "dataSource": "order_summary",           // Primary data source
  "additionalDataSources": ["hourly_revenue"], // Secondary data sources if needed

  // Default configuration
  "defaults": {
    "dateRange": "last_7_days",
    "filters": {},
    "refreshInterval": null                 // null = manual, or seconds
  },

  // Page-level filter bar configuration
  "filterBar": {
    "position": "top",                      // "top" | "sidebar"
    "showDateRange": true,
    "showGroupBy": true,                    // Show group-by dropdown
    "groupByOptions": ["campaign_name", "product_name", "gateway_alias"],
    "defaultGroupBy": "campaign_name",
    "filters": [
      {
        "dimensionKey": "sales_type",
        "displayAs": "pills",               // "pills" | "dropdown" | "multiselect"
        "position": "main",                  // "main" | "more"
        "defaultValue": null                 // null = all
      },
      {
        "dimensionKey": "campaign_name",
        "displayAs": "multiselect",
        "position": "more"
      }
    ],
    "showExport": true,
    "showSchedule": true,
    "showRefresh": true
  },

  // Layout sections (rendered top to bottom)
  "sections": [
    {
      "id": "summary_cards",
      "type": "card_row",                    // "card_row" | "grid" | "full_width"
      "title": null,                         // Optional section title
      "widgets": [
        {
          "id": "initials_card",
          "type": "metric_card",
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
          "type": "metric_card",
          "title": "Rebills",
          "metrics": [
            { "metricKey": "rebills", "label": "Count" },
            { "metricKey": "revenue_rebills", "label": "Revenue" }
          ],
          "showTrend": true
        },
        {
          "id": "total_card",
          "type": "metric_card",
          "title": "Total",
          "metrics": [
            { "metricKey": "approvals", "label": "Count" },
            { "metricKey": "revenue", "label": "Revenue" }
          ],
          "showTrend": true
        }
      ]
    },
    {
      "id": "trend_chart",
      "type": "full_width",
      "widgets": [
        {
          "id": "sales_trend",
          "type": "chart",
          "chartType": "line",               // "line" | "bar" | "area" | "stacked_bar" | "pie" | "donut" | "waterfall"
          "title": "Sales Trend",
          "xAxis": { "dimensionKey": "date_day" },
          "series": [
            { "metricKey": "approvals", "label": "# Approvals", "color": "#4F46E5" },
            { "metricKey": "revenue", "label": "$ Revenue", "color": "#10B981", "yAxis": "right" }
          ],
          "showLegend": true,
          "enableZoom": true
        }
      ]
    },
    {
      "id": "data_table",
      "type": "full_width",
      "widgets": [
        {
          "id": "sales_table",
          "type": "table",
          "title": "Sales by {groupBy}",     // {groupBy} is replaced dynamically
          "groupBy": "{groupBy}",            // Uses the page-level group-by selection
          "metrics": [
            { "metricKey": "initials", "showSparkline": true },
            { "metricKey": "revenue_initials" },
            { "metricKey": "rebills" },
            { "metricKey": "revenue_rebills" },
            { "metricKey": "straight_sales" },
            { "metricKey": "revenue" }
          ],
          "sortBy": "revenue",
          "sortDirection": "desc",
          "pagination": { "pageSize": 25 },
          "showTotals": true,
          "allowColumnToggle": true,         // Users can show/hide columns
          "allowColumnReorder": true,
          "exportable": true
        }
      ]
    }
  ]
}
```

### 7.2 Stock Report Templates to Create

These map to your current 13 Power BI reports:

| # | Template Key | Name | Primary Data Source |
|---|---|---|---|
| 1 | `dashboard` | Dashboard | order_summary + hourly_revenue |
| 2 | `sales` | Sales | order_summary |
| 3 | `mid_performance` | MID Performance | mid_summary |
| 4 | `approval_rate` | Approval % | order_summary |
| 5 | `decline_recovery` | Decline Recovery | decline_recovery |
| 6 | `profitability` | Profitability | order_summary |
| 7 | `ltv` | LTV | cohort_summary |
| 8 | `retention` | Retention | cohort_summary |
| 9 | `cb_refunds` | CB & Refunds | order_summary |
| 10 | `cb_refunds_growth` | CB & Refunds Growth | cb_refund_alert |
| 11 | `alert_analytics` | Alert Analytics | alert_summary + alert_details |
| 12 | `order_details` | Order Details | order_details |
| 13 | `routing_insights` | Routing Insights | fact_orders (raw) |

---

## 8. Widget System

### 8.1 Widget Types

| Widget Type | Description | Config Options |
|---|---|---|
| `metric_card` | Single or multi-metric summary card | metrics[], showTrend, trendComparison, icon |
| `chart` | Any chart visualization | chartType (line/bar/area/pie/donut/waterfall/stacked_bar/scatter/heatmap), xAxis, series[], yAxes |
| `table` | Data table with sorting, pagination | groupBy, metrics[], sortBy, pagination, showTotals, sparkline support |
| `pivot_table` | Cohort/pivot matrix | rowDimension, columnDimension, valueMetric (for LTV/Retention) |
| `counter` | Single big number | metricKey, comparison, color |
| `sparkline_row` | Horizontal sparkline cards | metrics[] with inline sparklines |
| `health_indicator` | Health status (for MID Performance) | metric, thresholds[], colors |
| `venn_diagram` | Overlap visualization (Alert duplication) | Custom config |
| `funnel` | Funnel chart | stages[] |
| `custom_input` | User-adjustable parameters (profitability sliders) | inputs[], formula |

### 8.2 Chart Types

| Chart Type | Use Case | Example Report |
|---|---|---|
| `line` | Trends over time | Sales trend, Approval rate trend |
| `bar` | Comparisons by category | Recovery by decline group |
| `stacked_bar` | Part-of-whole by category | Sales type breakdown |
| `area` | Volume over time | Revenue trend |
| `pie` / `donut` | Distribution | Dispute type breakdown |
| `waterfall` | Flow analysis | Cashflow / Reserve tracking |
| `heatmap` | Matrix intensity | Retention cohort heatmap |
| `scatter` | Correlation | Approval rate vs volume |
| `gauge` | Health indicator | VAMP %, CB % vs threshold |

---

## 9. Filter System

### 9.1 Filter Architecture

```jsonc
// A filter state object (saved in views, passed to API)
{
  "dateRange": {
    "type": "preset",                   // "preset" | "custom"
    "preset": "last_7_days",            // Preset name
    "startDate": null,                  // For custom
    "endDate": null
  },
  "dimensions": {
    "campaign_name": {
      "operator": "in",                 // "in" | "not_in" | "equals" | "contains" | "starts_with"
      "values": ["Campaign A", "Campaign B"]
    },
    "sales_type": {
      "operator": "in",
      "values": ["Initials"]
    },
    "gateway_id": {
      "operator": "in",
      "values": [7, 88, 100]
    }
  },
  "search": null,                       // Free-text search (for order details)
  "approvalMode": "standard"            // "standard" | "organic" | "net" (report-specific toggle)
}
```

### 9.2 Date Range Presets

| Preset Key | Label | Logic |
|---|---|---|
| `today` | Today | Current day |
| `yesterday` | Yesterday | Previous day |
| `last_7_days` | Last 7 Days | 7 days back from today |
| `last_calendar_week` | Last Calendar Week | Previous Mon-Sun |
| `last_4_weeks` | Last 4 Weeks | 28 days back |
| `last_3_months` | Last 3 Months | 3 months back |
| `last_6_months` | Last 6 Months | 6 months back |
| `month_to_date` | Month to Date | 1st of current month to today |
| `quarter_to_date` | Quarter to Date | 1st of current quarter to today |
| `year_to_date` | Year to Date | Jan 1 to today |
| `custom` | Custom | User-selected start/end |

### 9.3 Filter Presets (Rulesets)

Users can save filter combinations as named presets:

```jsonc
{
  "name": "High Value MIDs",
  "filters": {
    "gateway_id": { "operator": "in", "values": [7, 88, 100, 105] }
  }
}
```

These are stored in `report_builder.filter_presets` and available as quick-apply options across any report.

---

## 10. Query Engine

### 10.1 How It Works

The Query Engine is the core backend component that translates a report widget's configuration into SQL:

```
Input:  { dataSource, metrics[], dimensions[], filters, dateRange, sort, limit }
Output: { sql, params } → Execute → { rows, totals, metadata }
```

### 10.2 Query Generation Flow

```
1. Widget requests data:
   metrics: ["approvals", "revenue", "approval_rate"]
   dimensions: ["campaign_name"]
   filters: { dateRange: "last_7_days", sales_type: ["Initials"] }

2. Query Engine resolves metrics:
   - "approvals" → SUM(os.approvals)
   - "revenue" → SUM(os.revenue)
   - "approval_rate" → derived: SUM(os.approvals) / NULLIF(SUM(os.attempts), 0)

3. Query Engine resolves dimensions:
   - "campaign_name" → requires JOIN to campaigns table

4. Query Engine builds SQL:

   SELECT
     c.campaign_name AS "Campaign",
     SUM(os.approvals) AS "approvals",
     SUM(os.revenue) AS "revenue",
     SUM(os.approvals)::numeric / NULLIF(SUM(os.attempts), 0) AS "approval_rate"
   FROM reporting.order_summary_10000 os
   LEFT JOIN public.campaigns c
     ON c.campaign_id = os.campaign_id
     AND c.client_id = os.client_id
   WHERE os.date >= CURRENT_DATE - INTERVAL '7 days'
     AND os.date <= CURRENT_DATE
     AND os.sales_type = 'Initials'
   GROUP BY c.campaign_name
   ORDER BY "revenue" DESC

5. Execute query, format results, return to frontend
```

### 10.3 Query Engine Pseudocode

```javascript
class QueryEngine {
  async executeWidgetQuery({ clientId, dataSource, metrics, dimensions, filters, sort, pagination }) {
    // 1. Resolve the data source table
    const sourceTable = resolveTable(dataSource, clientId);
    // e.g., "reporting.order_summary_10000"

    // 2. Resolve metric definitions
    const metricDefs = await this.getMetricDefinitions(metrics);

    // 3. Resolve dimension definitions
    const dimensionDefs = await this.getDimensionDefinitions(dimensions);

    // 4. Determine required JOINs
    const joins = this.resolveJoins(metricDefs, dimensionDefs, sourceTable);

    // 5. Build SELECT clause
    const selectClauses = [
      ...dimensionDefs.map(d => `${d.column_expression} AS "${d.display_name}"`),
      ...metricDefs.map(m => m.is_derived
        ? this.buildDerivedExpression(m)
        : `${m.sql_expression} AS "${m.metric_key}"`)
    ];

    // 6. Build WHERE clause from filters
    const whereClauses = this.buildWhereClause(filters, clientId);

    // 7. Build GROUP BY
    const groupBy = dimensionDefs.map(d => d.column_expression);

    // 8. Build ORDER BY
    const orderBy = this.buildOrderBy(sort, metricDefs);

    // 9. Assemble final SQL
    const sql = this.assembleSql({
      select: selectClauses,
      from: sourceTable,
      joins,
      where: whereClauses,
      groupBy,
      orderBy,
      limit: pagination?.pageSize,
      offset: pagination?.page * pagination?.pageSize
    });

    // 10. Execute with parameterized values
    return this.db.query(sql, params);
  }
}
```

### 10.4 Security Considerations

- **SQL injection prevention:** All filter values are parameterized (`$1, $2, ...`), never concatenated
- **Client isolation:** Every query includes `client_id = {clientId}` in the WHERE clause, enforced at the engine level (not trusting frontend)
- **Read-only access:** The query engine uses a read-only database role
- **Rate limiting:** Per-client query rate limits to prevent abuse
- **Query timeout:** Maximum query execution time (e.g., 30 seconds)
- **Column whitelisting:** Only columns defined in the semantic layer can be queried

---

## 11. User Customization Features

### 11.1 Feature Matrix

| Feature | Stock Reports | Saved Views | Custom Reports | Dashboards |
|---|---|---|---|---|
| View pre-built reports | Yes | - | - | - |
| Change date range | Yes | Saved | Yes | Per widget |
| Apply filters | Yes | Saved | Yes | Per widget |
| Show/hide metrics | Yes | Saved | Yes | - |
| Change group-by | Yes | Saved | Yes | - |
| Change chart type | - | - | Yes | Yes |
| Reorder columns | Yes | Saved | Yes | - |
| Save configuration | - | Yes | Yes | Yes |
| Pick metrics from catalog | - | Yes | Yes | Yes |
| Pick dimensions | - | Yes | Yes | Yes |
| Drag-and-drop layout | - | - | Yes | Yes |
| Add widgets | - | - | Yes | Yes |
| Share with team | - | Yes | Yes | Yes |
| Schedule delivery | Yes | Yes | Yes | - |
| Export (CSV/PDF/Excel) | Yes | Yes | Yes | Yes |

### 11.2 Saved Views Workflow

1. User opens a stock report (e.g., Sales Report)
2. Applies filters (date range, campaign filter)
3. Hides some columns, changes sort order
4. Clicks "Save View" → Names it "Q1 Initials by Campaign"
5. View appears in their sidebar under "Saved Reports"
6. Next time, they open the saved view and it restores all customizations
7. They can also set it as "default" for that report page

### 11.3 Custom Report Builder Workflow

1. User clicks "Create Report" in sidebar
2. Selects a data source (Order Summary, Decline Recovery, etc.)
3. Gets a blank canvas with a metric catalog panel on the right
4. Drags widgets onto the canvas:
   - Adds a "metric card row" at top
   - Adds a "line chart" in the middle
   - Adds a "data table" at the bottom
5. For each widget, selects metrics from the catalog
6. Selects dimensions for grouping
7. Configures filters
8. Saves as a named report
9. Can share with other users in their organization

### 11.4 Dashboard Builder Workflow

1. User goes to "My Dashboard"
2. Clicks "Add Widget"
3. Options:
   - "From Report" → Pick a widget from an existing report to pin
   - "New Counter" → Pick a single metric as a big number
   - "New Chart" → Configure a chart from scratch
4. Arranges widgets on a grid (drag & resize)
5. Each widget has its own date range and filters
6. Dashboard auto-refreshes or manual refresh

---

## 12. API Design

### 12.1 Core Endpoints

#### Report Templates
```
GET    /api/v1/reports/templates                    - List available templates
GET    /api/v1/reports/templates/:id                - Get template with layout JSON
POST   /api/v1/reports/templates                    - Create custom template
PUT    /api/v1/reports/templates/:id                - Update template
DELETE /api/v1/reports/templates/:id                - Delete (user-created only)
```

#### Report Data
```
POST   /api/v1/reports/query                        - Execute a widget query
POST   /api/v1/reports/query/batch                  - Execute multiple widget queries in parallel
POST   /api/v1/reports/export                       - Export report data
```

#### Semantic Layer
```
GET    /api/v1/semantic/metrics                     - List all available metrics
GET    /api/v1/semantic/metrics/:category            - Metrics by category
GET    /api/v1/semantic/dimensions                  - List all available dimensions
GET    /api/v1/semantic/dimensions/:category         - Dimensions by category
GET    /api/v1/semantic/data-sources                - List data sources
```

#### Filters
```
GET    /api/v1/filters/options/:dimensionKey         - Get filter options (distinct values)
GET    /api/v1/filters/presets                       - List saved filter presets
POST   /api/v1/filters/presets                       - Save a filter preset
DELETE /api/v1/filters/presets/:id                   - Delete preset
```

#### Saved Views
```
GET    /api/v1/views                                - List user's saved views
POST   /api/v1/views                                - Create saved view
PUT    /api/v1/views/:id                            - Update saved view
DELETE /api/v1/views/:id                            - Delete saved view
```

#### Dashboards
```
GET    /api/v1/dashboards                           - List user's dashboards
GET    /api/v1/dashboards/:id                       - Get dashboard with widgets
POST   /api/v1/dashboards                           - Create dashboard
PUT    /api/v1/dashboards/:id                       - Update dashboard
POST   /api/v1/dashboards/:id/widgets               - Add widget
PUT    /api/v1/dashboards/:id/widgets/:widgetId     - Update widget
DELETE /api/v1/dashboards/:id/widgets/:widgetId     - Remove widget
```

#### Scheduling
```
GET    /api/v1/schedules                            - List schedules
POST   /api/v1/schedules                            - Create schedule
PUT    /api/v1/schedules/:id                        - Update schedule
DELETE /api/v1/schedules/:id                        - Delete schedule
```

### 12.2 Query API Request/Response

**Request:**
```jsonc
POST /api/v1/reports/query
{
  "dataSource": "order_summary",
  "metrics": ["approvals", "revenue", "approval_rate", "cb_count", "cb_rate"],
  "dimensions": ["campaign_name"],
  "filters": {
    "dateRange": { "type": "preset", "preset": "last_7_days" },
    "dimensions": {
      "sales_type": { "operator": "in", "values": ["Initials"] }
    }
  },
  "sort": { "field": "revenue", "direction": "desc" },
  "pagination": { "page": 0, "pageSize": 25 },
  "includeTotal": true,
  "includeTrend": true,
  "trendGranularity": "day"
}
```

**Response:**
```jsonc
{
  "data": {
    "rows": [
      {
        "campaign_name": "Summer Sale",
        "approvals": 1234,
        "revenue": 45678.90,
        "approval_rate": 0.7823,
        "cb_count": 12,
        "cb_rate": 0.0097
      }
      // ...
    ],
    "totals": {
      "approvals": 15678,
      "revenue": 567890.12,
      "approval_rate": 0.7456,
      "cb_count": 145,
      "cb_rate": 0.0092
    },
    "trend": [
      { "date": "2026-01-20", "approvals": 2100, "revenue": 78000 },
      { "date": "2026-01-21", "approvals": 2300, "revenue": 85000 }
      // ...
    ]
  },
  "metadata": {
    "totalRows": 156,
    "page": 0,
    "pageSize": 25,
    "queryTimeMs": 45,
    "dataLastUpdated": "2026-01-26T10:30:00Z"
  }
}
```

### 12.3 Batch Query (for loading full reports)

```jsonc
POST /api/v1/reports/query/batch
{
  "queries": [
    {
      "widgetId": "summary_cards",
      "dataSource": "order_summary",
      "metrics": ["initials", "rebills", "revenue_initials", "revenue_rebills"],
      "filters": { "dateRange": { "type": "preset", "preset": "last_7_days" } }
    },
    {
      "widgetId": "trend_chart",
      "dataSource": "order_summary",
      "metrics": ["approvals"],
      "dimensions": ["date_day"],
      "filters": { "dateRange": { "type": "preset", "preset": "last_7_days" } }
    },
    {
      "widgetId": "sales_table",
      "dataSource": "order_summary",
      "metrics": ["initials", "revenue_initials", "rebills", "revenue_rebills"],
      "dimensions": ["campaign_name"],
      "filters": { "dateRange": { "type": "preset", "preset": "last_7_days" } },
      "sort": { "field": "revenue_initials", "direction": "desc" }
    }
  ]
}
```

The backend executes all queries in parallel and returns results keyed by widgetId. This means a full report page with 5 widgets = 1 HTTP request, all queries run concurrently.

---

## 13. Frontend Architecture

### 13.1 Component Hierarchy

```
<App>
  <ReportPage>
    ├── <FilterBar />                    // Date range, filters, group-by, export
    ├── <ReportRenderer layout={json}>   // Reads template JSON, renders sections
    │   ├── <Section type="card_row">
    │   │   ├── <MetricCard />
    │   │   ├── <MetricCard />
    │   │   └── <MetricCard />
    │   ├── <Section type="full_width">
    │   │   └── <Chart type="line" />
    │   └── <Section type="full_width">
    │       └── <DataTable />
    └── <SaveViewDialog />

  <ReportBuilder>                        // Custom report creation
    ├── <WidgetPalette />                // Drag widget types
    ├── <MetricCatalog />                // Browse/search metrics
    ├── <DimensionCatalog />             // Browse/search dimensions
    ├── <BuilderCanvas />                // Drop zone with grid
    │   └── <WidgetConfigurator />       // Per-widget settings
    └── <PreviewPanel />                 // Live preview

  <DashboardPage>
    ├── <DashboardGrid>                  // react-grid-layout
    │   ├── <DashboardWidget />
    │   ├── <DashboardWidget />
    │   └── <AddWidgetButton />
    └── <WidgetConfigDrawer />
```

### 13.2 Key Libraries

| Purpose | Library | Why |
|---|---|---|
| Charts | **Recharts** or **Apache ECharts (via echarts-for-react)** | Recharts is React-native and simple. ECharts is more powerful for complex charts (waterfall, heatmap, gauge). |
| Data Tables | **TanStack Table (React Table v8)** | Best-in-class for sortable, filterable, paginated, column-resizable tables. Headless so full design control. |
| Dashboard Grid | **react-grid-layout** | Drag-and-resize dashboard widgets. Used by Grafana. |
| Date Picker | **react-day-picker** or **date-fns** based | Lightweight, customizable date range picker. |
| State Management | **Zustand** or **React Query (TanStack Query)** | TanStack Query for server state (caching, refetching). Zustand for UI state (filter selections, layout). |
| Drag & Drop | **dnd-kit** | For report builder widget palette. Modern, performant. |

### 13.3 Report Renderer Logic

```jsx
// Pseudocode for the report renderer
function ReportRenderer({ template, clientId, filters }) {
  // 1. Parse template JSON
  const { sections, filterBar, defaults } = template.layout;

  // 2. Build batch query from all widgets
  const queries = sections.flatMap(section =>
    section.widgets.map(widget => buildQuery(widget, filters))
  );

  // 3. Fetch all data in one batch request
  const { data } = useBatchQuery(queries, clientId);

  // 4. Render sections
  return (
    <>
      <FilterBar config={filterBar} />
      {sections.map(section => (
        <Section key={section.id} type={section.type}>
          {section.widgets.map(widget => (
            <Widget
              key={widget.id}
              config={widget}
              data={data[widget.id]}
            />
          ))}
        </Section>
      ))}
    </>
  );
}
```

---

## 14. Performance Strategy

### 14.1 Why This Will Be Fast

| Current (Power BI) | New (Direct SQL) |
|---|---|
| Browser → Power BI embed SDK → Power BI service → DAX engine → Data source | Browser → Node.js API → PostgreSQL materialized view |
| 5-10 seconds | <500ms target |

### 14.2 Performance Optimizations

1. **Materialized views are pre-aggregated** - Most queries are simple GROUP BY + SUM on already-computed data. These execute in <100ms.

2. **Batch queries** - One HTTP request per page, backend runs all widget queries in parallel using `Promise.all()`.

3. **Redis caching** - Cache query results with keys based on `(clientId, dataSource, metrics, dimensions, filters, dateRange)`. TTL based on data freshness (e.g., 5 minutes for hourly data, 1 hour for daily).

4. **Frontend caching** - TanStack Query caches responses client-side. Filter changes that match cached data are instant.

5. **Incremental loading** - Summary cards load first (smallest queries), then charts, then tables. Each widget has its own loading state.

6. **Pagination** - Tables load page 1 immediately, subsequent pages on demand.

7. **Index optimization** - Ensure materialized views have indexes on (client_id, date) and frequently-filtered columns.

8. **Connection pooling** - Use `pg-pool` with appropriate pool size for concurrent queries.

### 14.3 Caching Strategy

```
Layer 1: Frontend (TanStack Query)
  - Cache key: query params hash
  - TTL: 5 minutes (staleTime)
  - Instant for repeat visits to same report

Layer 2: Redis
  - Cache key: SHA256(clientId + query + filters)
  - TTL: varies by data source
    - hourly_revenue: 5 minutes
    - order_summary: 15 minutes (or until mat view refresh)
    - mid_summary: 1 hour
    - cohort_summary: 6 hours

Layer 3: Materialized Views (PostgreSQL)
  - Already pre-computed
  - Refresh on schedule (your existing ETL)
```

---

## 15. Migration Path

### Phase 1: Foundation (Semantic Layer + Query Engine)

- Create the `report_builder` schema and tables
- Populate metric catalog (~80-100 metrics)
- Populate dimension catalog (~30 dimensions)
- Build the Query Engine in Node.js
- Build the batch query API
- Build filter options API

### Phase 2: Stock Reports

- Create JSON templates for all 13 current reports
- Build React widget components (MetricCard, Chart, DataTable, PivotTable)
- Build ReportRenderer component
- Build FilterBar component
- Deploy as new pages alongside existing Power BI pages (feature flag per client)

### Phase 3: Saved Views & Filters

- Implement saved views (save/load/delete)
- Implement filter presets
- Implement "Save as default" for reports
- Export functionality (CSV, Excel)

### Phase 4: Custom Reports & Dashboards

- Build Report Builder UI (drag & drop)
- Build MetricCatalog and DimensionCatalog browsing UI
- Build Dashboard page with react-grid-layout
- Build widget pinning from reports to dashboards

### Phase 5: Advanced Features

- Scheduled report delivery (email, Slack, Telegram)
- Alerts (threshold-based notifications)
- AI insights integration
- Cross-client analytics (for admin users)

---

## 16. Possibilities & Advanced Features

### 16.1 What Becomes Possible

| Feature | Description |
|---|---|
| **Instant report creation** | New report = JSON template inserted in DB. No code deployment needed. |
| **Client-specific report customization** | Override any stock report's template per client without affecting others. |
| **User-built reports** | Users pick metrics + dimensions from catalog, arrange widgets, save & share. |
| **Saved views** | Every user can have personalized views of every report. |
| **Filter presets (rulesets)** | Save complex filter combos, apply with one click. |
| **Personal dashboards** | Pin key metrics from different reports into one view. |
| **Sub-second loading** | Direct SQL against materialized views + Redis cache. |
| **Real-time dashboard** | hourly_revenue view + WebSocket push for live updates. |
| **Drill-down** | Click a chart segment → opens filtered detail view. |
| **Metric comparison** | Compare any metric vs previous period, vs goal, vs other segment. |
| **Conditional formatting** | Color cells in tables based on thresholds (red CB rate > 1%). |
| **Alerts** | "Notify me when CB rate > 1% for any MID" → email/Slack/Telegram. |
| **Scheduled exports** | Auto-send PDF/CSV reports daily/weekly/monthly. |
| **Report sharing** | Share a saved view URL with colleagues. |
| **Template marketplace** | Create report templates once, assign to multiple clients. |
| **A/B testing reports** | Different report layouts for different users, measure engagement. |
| **Annotation/comments** | Users can add notes to specific data points or date ranges. |
| **Goal tracking** | Set targets for metrics, show progress bars. |
| **Calculated fields** | Users define custom metrics from existing ones (e.g., "Revenue - CPA Cost"). |
| **Cross-source reports** | Combine data from order_summary + decline_recovery in one view. |
| **Embedded analytics** | Expose report widgets via SDK for clients' own apps. |
| **White-label** | Complete brand customization per client. |
| **Mobile responsive** | Widgets reflow for mobile with simplified views. |
| **Export to Google Sheets** | Live-connected Google Sheet that auto-updates. |
| **AI chat over data** | "Why did approval rate drop last Tuesday?" → AI analyzes the data. |

### 16.2 Comparison: Current vs New

| Capability | Current (Power BI) | New (React + Semantic Layer) |
|---|---|---|
| Load time | 5-10 seconds | <500ms |
| Custom reports by users | No | Yes |
| Saved views | No | Yes |
| Saved filters (rulesets) | No | Yes |
| Personal dashboard | No | Yes |
| Adding new metric | Edit DAX + republish | Insert row in DB |
| New report for client | Clone PBI workspace + parameterize | Insert JSON template |
| Export | Limited PBI export | CSV, Excel, PDF, Google Sheets |
| Mobile experience | Poor (PBI embed) | Responsive design |
| Cost | Power BI Premium license per client | Self-hosted, no per-seat cost |
| Branding | Limited PBI theming | Full control |
| AI integration | Separate page | Inline insights per widget |

---

*Document generated: 2026-01-26*
*Architecture plan for Beast Insights Report Builder*

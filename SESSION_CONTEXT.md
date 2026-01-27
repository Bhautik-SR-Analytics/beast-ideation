# Beast Insights — Session Context & Project Brief

> **Purpose:** This document gives a new AI session full context on the Beast Insights migration project. Read this first before doing any work.

---

## 1. What Is Beast Insights

Beast Insights (`app.beastinsights.co`) is a **multi-tenant payment analytics platform**. Clients (merchants) connect their CRM data (Sticky, Konnektive, VRIO, etc.) and Beast Insights provides reports on revenue, approvals, chargebacks, refunds, retention, LTV, profitability, and more.

**Current clients:** High-risk subscription merchants (supplements, skincare, digital products). The platform is expanding to telehealth, SaaS, and other subscription industries.

---

## 2. Current Tech Stack

| Layer | Technology |
|---|---|
| Database | PostgreSQL (hosted on Azure) |
| Backend | Node.js |
| Frontend | React |
| Reports | **Power BI Embedded** (being removed) |
| Billing | Stripe |
| Notifications | Email + Telegram |

---

## 3. The Migration Goal

**Remove Power BI entirely.** Replace it with a native React reporting system that is:

1. **Fast** — <1 second load (currently 5-10 seconds with Power BI)
2. **JSON-driven** — Reports defined as JSON templates in DB, no code changes for new reports
3. **Customizable** — Users can save views, create custom reports, build personal dashboards
4. **Cross-industry** — Universal core reports + industry-specific modules
5. **Scalable** — Adding new clients = assigning report templates, no Power BI workspace setup

---

## 4. Database Architecture

### 4.1 Connection (Read-Only)

```
Host: db.beastinsights.com
Port: 5432
Database: postgres
User: beastinsights_ro
Password: 5QwxpH889T3qansugXQA
```

### 4.2 Schemas

| Schema | Purpose |
|---|---|
| `beast_insights_v2` | Application layer — clients, users, pages, reports, filters, navigation, metrics |
| `data` | Raw transactional data per client: `orders_{client_id}` (81 columns each) |
| `public` | Shared dimension tables: `campaigns`, `bin_lookup`, `offers`, `decline_groups` |
| `reporting` | Pre-computed materialized views per client (10 types) |
| `beastinsights_alert` | Alert-specific data |
| `routing` | Routing-related data |

### 4.3 Key Tables

**Clients:** `beast_insights_v2.clients` — id, name, package_id, is_active, data_timezone, client_timezone, stripe_customer_id

**Raw Data:** `data.orders_{client_id}` — 81 columns per client including: order identifiers, dates (date_of_sale, chargeback_date, refund_date, recurring_date), product/campaign/gateway IDs, customer info, financials (order_total, refund_amount), boolean flags (is_approved, is_chargeback, is_refund, is_recurring, is_cancelled, is_cascaded, is_fraud), affiliate tracking (affid, sub_affid, c), card details (bin, cycle, attempt, decline_reason, decline_group)

**Dimension Tables:**
- `public.campaigns` — campaign_id, campaign_name, campaign_type, crm, client_id, cpa
- `public.bin_lookup` — bin, bank, card, type, class, country
- `public.offers` — campaign_id, offer_id, product_id, product_name, billing_model_name
- `public.decline_groups` — decline_reason, decline_group

### 4.4 Materialized Views (reporting schema)

10 types, one instance per client (e.g., `order_summary_10000`):

| View | Purpose | Key Columns |
|---|---|---|
| `order_summary_{id}` | Core aggregated data. Powers most reports. | date, campaign_id, product_id, gateway_id, bin, affid, sales_type, billing_cycle, attempts, approvals, net_approvals, approvals_organic, cancel, revenue, cb, cb_dollar, cb_cb_date, refund, refund_dollar, refund_refund_date, cpa |
| `mid_summary_{id}` | MID health, monthly snapshots | gateway_id, month_year, volume, cb rates (visa/mc), decline_rate, alert_rate, health_tag, monthly_cap, capacity_left |
| `cb_refund_alert_{id}` | CB/refund growth tracking | Same as order_summary + cb_no_of_days, refund_no_of_days, dispute_no_of_days |
| `cohort_summary_{id}` | Retention & LTV cohorts | date, billing_cycle, trial_campaign_id, trial_product_id, trial_gateway_id, attempts, approvals, revenue, cb, refund |
| `decline_recovery_{id}` | Retry analysis | decline_group, recovery_attempts, organic_declines, reattempts, recovered, recovered_dollar |
| `hourly_revenue_{id}` | Real-time hourly data | hour, avg_7d_revenue, today_revenue (split by initial/rebill/straight_sales) |
| `alert_summary_{id}` | Alert aggregations | alert_type (RDR/Ethoca/CDRN), alert_status, alert_count, alert_dollar |
| `alert_details_{id}` | Individual alert records | alert_id, alert_date, alert_type, order_id, transaction_amount |
| `order_details_{id}` | Transaction-level detail | Mirrors raw orders table for drill-down |
| `products_{id}` | Product dimension | Product attributes per client |

### 4.5 Application Layer Tables

- `beast_insights_v2.reports` — Maps to Power BI reports per client (report_name, powerbi_report_id, powerbi_workspace_id)
- `beast_insights_v2.pages` — Individual report pages (page_display_name, report_id, route_id)
- `beast_insights_v2.navigation_items` — Hierarchical nav tree (parent_id, nav_item_type, title, icon, page_id)
- `beast_insights_v2.filter_definitions` — 30+ filter configs (filter_key, name, table, column, category)
- `beast_insights_v2.metrics` — 17 currently defined metrics (name, power_bi_metric_key)
- `beast_insights_v2.order_details_column_definitions` — Maps DB columns to display names for order details

---

## 5. Architecture Decision: Semantic Layer + JSON Templates

We evaluated 4 approaches and chose **Option B: Semantic Layer + JSON Templates**.

**How it works:**
- **Semantic Layer (DB):** Defines all metrics (with SQL expressions), dimensions (with column mappings), and data sources. Adding a new metric = inserting a row.
- **JSON Templates (DB):** Define report layouts — which widgets, which metrics per widget, chart types, toggles, filters. Adding a new report = inserting JSON.
- **Query Engine (Node.js):** Translates metric/dimension/filter selections into SQL against materialized views. ONE generic endpoint (`POST /api/v1/query`), no report-specific APIs.
- **Report Renderer (React):** Reads JSON template, builds API requests from widget configs, renders using generic widget components.

**Why not the others:**
- Option A (Pure JSON): Doesn't capture how to compute metrics
- Option C (Pre-built APIs): Not dynamic, requires code changes for new reports
- Option D (Superset/Metabase): Trades one dependency for another, limited UX control

---

## 6. JSON-Driven System Design

### 6.1 One Generic Endpoint

```
POST /api/v1/query
{
  "source": "order_summary",
  "metrics": ["approvals", "revenue"],
  "dimensions": ["campaign_name"],
  "filters": { "dateRange": { "preset": "last_7_days" } },
  "toggles": { "approval_mode": "organic" },
  "sort": { "field": "revenue", "dir": "desc" },
  "pagination": { "page": 0, "size": 25 }
}
```

Backend resolves metric keys to SQL expressions, dimension keys to columns + JOINs, builds and executes SQL. No report-specific logic.

### 6.2 Batch Queries

`POST /api/v1/query/batch` — One HTTP request per page load. Backend runs all widget queries in parallel.

### 6.3 Generic Report Renderer

Frontend reads template JSON, extracts metric/dimension keys from each widget, combines with current filter state, sends to generic query endpoint, renders response using widget type component. The frontend never constructs SQL or knows column names.

---

## 7. Metric Variants System (Context-Dependent Metrics)

The hardest design problem. Some metrics change SQL based on toggles:

- **"# Approvals"** can mean `SUM(approvals)` OR `SUM(approvals_organic)` OR `SUM(net_approvals)` — controlled by "approval_mode" toggle
- **"# CB"** can mean `SUM(cb)` (by order date) OR `SUM(cb_cb_date)` (by CB date) — controlled by "date_basis" toggle
- **"# Refunds"** similarly switches between `SUM(refund)` and `SUM(refund_refund_date)`

### Solution: 5 Toggle Types

| Type | What It Does | Example |
|---|---|---|
| **metric_variant** | Changes which SQL column is used | Approval mode (standard/organic/net) |
| **dimension_switch** | Changes GROUP BY dimension | Time granularity (day/week/month) |
| **visibility** | Shows/hides frontend elements | Dollar vs count display |
| **layout_switch** | Shows different widget sections | Profitability view (cashflow vs unit economics) |
| **parameter_input** | User-provided values for calculations | Processing fee %, reserve %, CB fee $ |

### Derived Metric Cascading

When a toggle changes a base metric (e.g., approvals → organic), all derived metrics using it automatically cascade. `approval_rate = {approvals} / {attempts}` automatically uses the organic variant when the toggle is set.

### DB Schema for Variants

```sql
report_builder.metric_toggles (toggle_key, display_name, toggle_type)
report_builder.metric_toggle_options (toggle_id, option_key, display_name, is_default)
report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
```

---

## 8. Cross-Industry Expansion

### 8.1 Core Platform (12 Universal Reports)

These work for ANY recurring/subscription business:

```
MONITOR
  1. Business Command Center — "What needs attention right now?"
  2. Real-Time Pulse — "How is today going hour by hour?"

GROW
  3. Revenue Analytics — "Where is money coming from?"
  4. Subscription Intelligence — "Is MRR growing? What's moving it?"
  5. Customer Lifecycle — "How long do customers stay? Where/why do they leave?"

RETAIN
  6. Churn Analysis — "Voluntary vs involuntary churn, and how to fix each"
  7. Payment Health & Recovery — "Are payments succeeding? How much lost to failures?"

ACQUIRE
  8. Channel & Acquisition Performance — "Which channels bring customers worth keeping?"

PROFIT
  9. Financial Performance — "After all costs, am I profitable?"
  10. Product & Plan Performance — "Which plans work, plan migration flows"

EXPLORE
  11. Transaction Explorer — "Look up any transaction, see customer journey"
  12. Custom Report Builder — "Build any report from metrics/dimensions catalog"
```

### 8.2 Industry Modules (Opt-In Add-Ons)

**High-Risk Merchant Module:**
- MID Health & Compliance (VAMP/MC thresholds, days-to-breach)
- BIN Routing Optimizer (gateway x BIN matrix, routing recommendations)
- Decline Recovery Intelligence (retry windows, recovery gap)
- Alert Service Management (RDR/Ethoca/CDRN effectiveness)
- Gateway Performance (per-gateway approval rates, capacity)
- Affiliate Quality (per-affiliate scoring, fraud signals)

**Telehealth Module:**
- Provider Performance, Visit-to-Payment Conversion, Insurance vs Self-Pay

**SaaS Module:**
- ARR Tracking, Trial-to-Paid, Seat/Usage Analytics, Net Dollar Retention

**eCommerce Module:**
- Order Frequency & Skip Rate, Returns, Seasonal Demand, Basket Analysis

### 8.3 Navigation Adapts

Core navigation is always present. When a module is enabled, its reports appear in the relevant sections (e.g., enabling High-Risk adds "MID Health" under RETAIN and "BIN Routing" under a new OPTIMIZE section).

---

## 9. Universal Metrics & Dimensions

### 9.1 Metrics (50 core, expandable)

**Tier 1 (Everyone):** Total Revenue, Net Revenue, New Customer Revenue, Recurring Revenue, MRR, New Customers, Active Subscribers, Net Subscriber Growth, Churn Rate (total/voluntary/involuntary), Payment Success Rate

**Tier 2 (Operational):** AOV, ARPU, Revenue Growth Rate, New/Expansion/Contraction/Churned MRR, Net MRR Movement, Retention by month (M1/M3/M6/M12), Churn by reason, Failed Payment Count/$/Recovery Rate/Recovery Gap, Refund/CB counts and rates

**Tier 3 (Growth):** CAC, LTV (30/60/90/180d), LTV:CAC, Payback Period, Conversion Rate, Channel Quality Score, Revenue Concentration %, Plan Distribution, Upgrade/Downgrade Rates

**Tier 4 (Financial):** Gross Profit, Gross Margin %, Processing Cost, CAC Cost, COGS, Dispute Fees, Net Dollar Retention

### 9.2 Dimensions (Universal)

Product/Plan, Channel/Source, Date (Day/Week/Month), Customer Segment, Geography, Payment Method, Customer Age (tenure), Billing Cycle, Price Tier

High-risk module adds: Gateway, BIN, Bank, Affiliate, MID, Offer
Telehealth adds: Provider, Service Type, State, Insurance Type
SaaS adds: Company Size, Seat Count, Usage Tier

---

## 10. New Database Schema (report_builder)

Key tables to create:

```
report_builder.data_sources — Available materialized views/tables
report_builder.metrics — Metric catalog (metric_key, sql_expression, data_type, format, category)
report_builder.dimensions — Dimension catalog (dimension_key, column_expression, join info)
report_builder.data_source_metrics — Which metrics available from which sources
report_builder.data_source_dimensions — Which dimensions available from which sources
report_builder.metric_toggles — Toggle definitions (approval_mode, date_basis, etc.)
report_builder.metric_toggle_options — Options per toggle (standard/organic/net)
report_builder.metric_variants — SQL overrides per metric per toggle option
report_builder.report_templates — JSON layout templates (stock and custom)
report_builder.saved_views — User customizations on top of templates
report_builder.filter_presets — Named filter combinations
report_builder.dashboards — User dashboard layouts
report_builder.dashboard_widgets — Pinned widgets with their own data config
report_builder.scheduled_reports — Automated delivery config
```

Full CREATE TABLE statements are in `Report_Builder_Plan.md`.

---

## 11. Frontend Architecture

### Components

```
<ReportPage>
  ├── <FilterBar /> — Date range, filters, toggles, group-by, export
  ├── <ReportRenderer layout={json}> — Reads template JSON, renders sections
  │   ├── <MetricCard /> — Summary metric cards with trend
  │   ├── <Chart /> — Line/bar/area/pie/waterfall/heatmap via config
  │   ├── <DataTable /> — Sortable/paginated tables (TanStack Table)
  │   └── <PivotTable /> — Cohort/retention matrices
  └── <SaveViewDialog />

<ReportBuilder> — Custom report creation
<DashboardPage> — Personal dashboard with drag-and-drop grid
```

### Libraries

| Purpose | Library |
|---|---|
| Charts | Recharts or ECharts |
| Tables | TanStack Table v8 |
| Dashboard grid | react-grid-layout |
| Server state | TanStack Query |
| UI state | Zustand |
| Drag & drop | dnd-kit |

---

## 12. Performance Strategy

```
Layer 1: Frontend (TanStack Query) — 5 min stale time
Layer 2: Redis — TTL varies (5min for hourly, 15min for daily, 1hr for monthly)
Layer 3: Materialized Views — Pre-computed by existing ETL
```

Target: <500ms per page load (vs 5-10 seconds with Power BI).

Batch queries: One HTTP request per page, all widget queries run in parallel on backend.

---

## 13. Implementation Phases

### Phase 1: Core Foundation
- Semantic layer DB tables + populate metric/dimension catalogs
- Query Engine (Node.js) — resolves metrics/dimensions/toggles to SQL
- Batch query API
- React widget components (MetricCard, Chart, DataTable, PivotTable)
- ReportRenderer + FilterBar
- First 6 core reports: Command Center, Revenue Analytics, Subscription Intelligence, Payment Health, Transaction Explorer, Customer Lifecycle

### Phase 2: Retention + Acquisition + Profit
- Churn Analysis, Channel & Acquisition, Financial Performance, Product & Plan Performance
- Saved filter presets, scheduled email delivery, Real-Time Pulse

### Phase 3: Self-Service + High-Risk Module
- Custom Report Builder, Personal Dashboard, Smart Alerts
- High-Risk Module (all 6 reports — serve existing customer base)

### Phase 4: Industry Expansion
- Telehealth Module, SaaS Module, eCommerce Module

---

## 14. Existing Documents in This Repo

| File | What It Contains |
|---|---|
| `Beast_Insights_Platform_Documentation.md` | Original platform docs — all 13 current Power BI reports with filters, widgets, columns |
| `Current_System_Analysis.md` | Deep technical analysis of current system — all 81 order columns, all 10 materialized views with full schemas, application layer tables, Power BI DAX measures, data model relationships |
| `Report_Builder_Plan.md` | Complete technical architecture — 4 approach options, recommended Semantic Layer + JSON approach, full CREATE TABLE SQL, ~100 metrics cataloged with SQL expressions, ~30 dimensions, JSON template schema, widget system, filter system, query engine pseudocode, 20+ API endpoints, frontend component hierarchy, 3-layer caching strategy, 5-phase migration |
| `JSON_Driven_System_Design.md` | Deep dive on JSON-driven design — one generic endpoint, generic renderer, metric variants system, 5 toggle types, complete end-to-end example (Approval Report), handling complex scenarios (different data sources, cohort tables, MID snapshots, real-time dashboard, profitability parameters) |
| `Product_Vision_Reports_From_Scratch.md` | 13 reports designed from first principles (high-risk focused) — organized by user intent (Monitor/Grow/Optimize/Protect/Profit/Explore), 5 user personas, 70 metrics across 5 tiers, detailed layouts and key innovations |
| `Beast_Insights_Product_Vision.md` | Non-technical/marketing version of above — written for clients, sales team, stakeholders |
| `Beast_Insights_Product_Overview.md` | Simplified visual version — ASCII wireframes for all 13 reports, before/after comparison, personalization features |
| `Cross_Industry_Expansion_Plan.md` | Cross-industry strategy — 12 universal core reports, 4 industry modules (high-risk/telehealth/SaaS/eCommerce), universal metrics catalog (50), competitive positioning, data model changes, implementation phases |
| `vrio.md` | Competitor analysis — VRIO Analytics with 926+ metrics, 50+ dimensions, saved reports, rulesets (used as inspiration) |

---

## 15. What's Been Decided

| Decision | Choice | Documented In |
|---|---|---|
| Architecture approach | Semantic Layer + JSON Templates (Option B) | Report_Builder_Plan.md |
| Remove Power BI | Yes, fully replace with native React | All docs |
| Query endpoint design | One generic `POST /api/v1/query` endpoint | JSON_Driven_System_Design.md |
| Context-dependent metrics | Metric Variants system with 5 toggle types | JSON_Driven_System_Design.md |
| Report organization | By user intent, not data type | Product_Vision_Reports_From_Scratch.md |
| Cross-industry strategy | 12 universal core reports + industry modules | Cross_Industry_Expansion_Plan.md |
| High-risk specifics become module | BIN routing, MID health, alerts → opt-in module | Cross_Industry_Expansion_Plan.md |
| Caching strategy | 3-layer: TanStack Query → Redis → Materialized Views | Report_Builder_Plan.md |
| Frontend libraries | Recharts/ECharts, TanStack Table, react-grid-layout, Zustand | Report_Builder_Plan.md |

## 16. What's Still Open / Not Yet Decided

- Exact API route structure (draft exists but not finalized)
- Authentication/authorization approach for the new system
- How to handle per-client module enablement in the DB
- Exact ETL changes needed for new universal data model
- How to handle data from non-CRM sources (Stripe direct, custom APIs)
- Telehealth/SaaS/eCommerce data ingestion pipelines
- White-labeling implementation details
- Mobile responsive strategy specifics
- Real-time WebSocket push for live dashboards
- AI insights integration approach
- Alert/notification system architecture (email/Slack/Telegram)
- Pricing/packaging for modules
- Migration timeline and client rollout sequence

---

## 17. Key Technical Patterns

### How a Report Page Works (End to End)

```
1. User navigates to "Revenue Analytics"
2. Frontend fetches report template JSON from API
3. Template defines: 3 metric cards, 1 line chart, 1 data table
4. Frontend reads each widget's config, extracts metric/dimension keys
5. Frontend combines with current filter state (date range, toggles)
6. Frontend sends ONE batch request to POST /api/v1/query/batch
7. Backend Query Engine for each widget query:
   a. Resolves metric keys → SQL expressions (checking for toggle variants)
   b. Resolves dimension keys → column names + required JOINs
   c. Resolves filters → WHERE clauses
   d. Assembles SQL, executes against reporting.order_summary_{clientId}
8. All queries run in parallel, results returned keyed by widget ID
9. Frontend distributes results to widgets, each renders by type
10. Total time: <500ms
```

### How a New Metric Is Added

```
1. INSERT INTO report_builder.metrics (metric_key, display_name, sql_expression, ...)
2. INSERT INTO report_builder.data_source_metrics (data_source_id, metric_id)
3. If toggle variants needed: INSERT INTO report_builder.metric_variants
4. Done. No code deployment. Available immediately in all reports and Custom Report Builder.
```

### How a New Report Is Created

```
1. INSERT INTO report_builder.report_templates (name, layout) with JSON defining widgets
2. Assign to clients via navigation_items
3. Done. No code deployment.
```

---

*Last updated: 2026-01-26*
*Beast Insights — Migration from Power BI to native React reporting platform*

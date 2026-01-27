# Beast Insights — Product Roadmap

> Native JSON-driven reporting platform replacing Power BI. Deployed on a separate subdomain.

---

## Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Frontend | Next.js 16 + React 19 | Repo: `beastinsights` |
| Analytics UI | Tremor | Cards, charts, tables |
| Navigation UI | shadcn/ui + Radix + Tailwind | Sidebar, inputs, popovers |
| Tables | TanStack React Table v8 | Tremor wraps this |
| State (client) | Zustand | Filters, UI state, auth |
| State (server) | TanStack Query | Data fetching, 5-min stale cache |
| Backend | Express.js 4.19 | Repo: `beastinsights-backend` |
| ORM | Prisma 5.19 | CRUD only |
| Query Engine | `pg` driver | Dynamic SQL generation |
| Database | PostgreSQL | Shared DB, new `report_builder` schema |
| Auth | JWT HS512 + bcrypt | Brought from `beast-insights-backend` |

**Database approach:** Raw SQL files to create schema, then `prisma db pull` to generate models. No Prisma migration.

---

## Phase Overview

| Phase | Name | What It Delivers |
|-------|------|-----------------|
| **1** | Core Reporting Engine | 11 interactive reports with real data, dynamic filters, slicers, <500ms load |
| **2** | User Customization | Custom reports, bookmarks, filter presets, notifications, report access control, export, scheduled delivery |
| **3** | Row-Level Security | Dimension-level data access per user — auto-filtered across all reports |
| **4** | Industry Modules | Opt-in specialized report packs per vertical (High-Risk, Telehealth, SaaS, eCommerce) |

---

## Phase 1: Core Reporting Engine

> Build the engine that converts JSON templates into interactive reports with dynamic filters, all visual types, metric slicers, and sub-second load times.

### Milestone Dependency Graph

```
M1  Database Architecture
 |
 v
M2  JSON Template Format Specification        (design document, no code)
 |
 ├──────────────────────────┐
 v                          v
M3  Backend Query Engine    M4  Frontend Report Renderer    M5  Data Library from PBI
 |                          |                                |
 └──────────┬───────────────┘                                |
            v                                                |
M6  Reports, Filters & Navigation  ◄─────────────────────────┘
 |
 v
M7  End-to-End Testing & Rollout
```

M3, M4, M5 can run in parallel — backend builds query engine, frontend builds renderer with mock data, analyst extracts PBI data library. All three depend on M1 and M2 but not on each other.

---

### Milestone 1: Database Architecture

**Goal:** Create the `report_builder` PostgreSQL schema with all tables needed for Phase 1 through Phase 4.

We create ALL tables upfront — even those used in later phases — because altering schema mid-flight is disruptive. Tables for Phase 2+ simply remain empty until needed.

**Phase 1 tables (populated):**

| Table | Purpose | Rows |
|-------|---------|------|
| `data_sources` | Maps logical names to materialized view patterns | ~10 |
| `metrics` | Full metric catalog with SQL expressions | ~100 |
| `dimensions` | Dimension catalog with columns and JOINs | ~30 |
| `data_source_metrics` | Which metrics work with which datasets | ~500 |
| `data_source_dimensions` | Which dimensions work with which datasets | ~200 |
| `metric_toggles` | Toggle definitions (approval_mode, date_basis, etc.) | 6 |
| `metric_toggle_options` | Options per toggle (standard, organic, net) | ~20 |
| `metric_variants` | SQL overrides per metric/toggle/option | ~50 |
| `report_templates` | JSON report layout definitions | 11 |

**Future phase tables (created empty):**

| Table | Phase | Purpose |
|-------|-------|---------|
| `bookmarks` | 2 | Per-user saved report state |
| `filter_presets` | 2 | Named, shareable filter sets |
| `dashboards` | 2 | Personal dashboard layouts |
| `dashboard_tiles` | 2 | Pinned visuals on dashboards |
| `scheduled_reports` | 2 | Scheduled delivery configs |
| `export_jobs` | 2 | Export tracking |
| `user_report_access` | 2 | Which reports each user/role can access |
| `notification_rules` | 2 | Threshold-based alert rules |
| `notification_log` | 2 | Notification delivery history |
| `user_dimension_access` | 3 | Row-level dimension permissions |
| `dimension_access_audit` | 3 | Audit log for access changes |
| `client_modules` | 4 | Per-client industry module enablement |

**Tasks:**
1. Write raw SQL file for entire `report_builder` schema (all tables)
2. Execute SQL against PostgreSQL
3. Run `prisma db pull` to generate Prisma models
4. Verify all tables, foreign keys, and indexes exist

**Files:**
```
beastinsights-backend/
  prisma/sql/create_report_builder_schema.sql
  prisma/schema.prisma  (generated via prisma db pull)
```

**Verify:** Connect to DB, query `report_builder.data_sources` — table exists. Prisma schema reflects new tables. Existing app unaffected.

> Full table definitions, column specs, constraints, ER diagrams, and CREATE TABLE SQL are in **Database_Design_M1.md**.

---

### Milestone 2: JSON Template Format Specification

**Goal:** Define the exact JSON structure that describes any report. This is the contract between backend and frontend.

This is a **design milestone** — no code. Once locked, backend and frontend teams build in parallel.

**What the spec covers:**

| Section | Key Decisions |
|---------|--------------|
| Template root | `version`, `source`, `sourceConfig`, `defaults`, `slicerPanel`, `sections` |
| Slicer panel | Date range, dimension filters, slicer controls (pills/dropdown), parameters, group-by, search, status pills, export |
| Sections | Layout types: `card_row`, `full_width`, `grid`, `tabs`. Conditional visibility based on toggle state. |
| Visual types | **Card** (1-3 metrics + trend), **Chart** (line/bar/area/pie/stacked/donut), **Table** (sortable, paginated, totals, column toggle), **Matrix** (cohort heatmap), **Waterfall** (P&L), **ParameterInput** (number with $/%), **HealthIndicator** (status badge) |
| Special behaviors | Auto-refresh, server-side pagination, server-side search, cross-source visuals, dynamic titles, dynamic X axis, month selector, compare to previous, inverted trends |

**Deliverable:** A specification document that a developer can read and build backend or frontend without ambiguity.

**Verify:** Every visual type, chart type, filter type, slicer type is defined. An example template is valid against the spec.

---

### Milestone 3: Backend Query Engine

**Goal:** Build the generic API that takes any combination of metrics, dimensions, filters, and toggles — returns formatted data.

**Repo:** `beastinsights-backend` (brings over login/auth from `beast-insights-backend`)

**Endpoints:**

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/query` | Single visual query |
| `POST /api/v1/query/batch` | All visuals for a page in one HTTP call |

**Query processing pipeline:**
1. **Source resolution** — `"order_summary"` + client `10043` → `reporting.order_summary_10043`
2. **Metric resolution** — look up SQL expression, check for toggle variants, resolve derived metrics (cascade variants automatically)
3. **Dimension resolution** — resolve columns + required JOINs
4. **Filter building** — always inject `client_id` from JWT, add date range, dimension filters, search. All parameterized (`$1, $2, ...`)
5. **SQL assembly** — SELECT + FROM + JOINs + WHERE + GROUP BY + ORDER BY + LIMIT/OFFSET
6. **Execution** — dedicated `pg` pool (separate from Prisma)
7. **Response formatting** — format values using metric catalog (currency, percentage, integer)

**Batch query:** Frontend sends ONE HTTP request with queries for every visual. Backend runs all in parallel via `Promise.all()`. Returns results keyed by `visualId`.

**Security:** Parameterized queries (no concatenation), client_id from JWT only, column whitelist enforcement, 30s query timeout.

**Catalog cache:** Metrics/dimensions/toggles loaded into memory on startup, refreshed every 5 minutes.

**Files:**
```
beastinsights-backend/
  db/queryPool.js
  services/queryEngine/
    index.js, sourceResolver.js, metricResolver.js,
    dimensionResolver.js, filterBuilder.js, sqlAssembler.js,
    responseFormatter.js, batchExecutor.js
  routes/queryRoutes.js
  controller/queryController.js
```

**Verify:** Single query returns correct data. Batch 5 queries returns all 5 results. Toggle changes metric values. Derived metric cascading works. Cross-tenant access blocked.

---

### Milestone 4: Frontend Report Renderer

**Goal:** All React components needed to render any report from a JSON template.

**Repo:** `beastinsights` (new empty repo, separate subdomain). Design reference: `beastinsights-old` for colors/layout.

**Visual components:**

| Component | File | What It Renders |
|-----------|------|----------------|
| CardVisual | `CardVisual.jsx` | 1-3 formatted metrics + trend arrow. Formats: $45,678 / 72.3% / 1,234. `invertTrend` for metrics where lower is better. |
| ChartVisual | `ChartVisual.jsx` | Tremor chart: line, bar, area, stacked bar, pie, donut. Configurable axes, series, legend. Responds to `{time_granularity}` placeholder. |
| TableVisual | `TableVisual.jsx` | TanStack Table: sortable, paginated, totals row, conditional formatting, column toggle. Server-side pagination mode for large datasets. Responds to `{groupBy}` placeholder. |
| MatrixVisual | `MatrixVisual.jsx` | Cohort/retention matrix with heatmap coloring. Transforms flat data into row/column matrix. |
| WaterfallVisual | `WaterfallVisual.jsx` | P&L: Revenue minus costs equals profit. Custom Tremor stacked bar. |
| ParameterInput | `ParameterInput.jsx` | Number input with $/% bound to Zustand parameters state. |
| HealthIndicator | `HealthIndicator.jsx` | Status badge: green/yellow/red based on thresholds. |

**Infrastructure:**

| Component | Purpose |
|-----------|---------|
| Zustand stores | `reportFilterStore.js` (date, filters, toggles, params, groupBy), `reportUIStore.js` (tabs, collapsed sections) |
| `useReportData` hook | Template + Zustand state → batch query payload → TanStack Query → `{ data, isLoading, error }` per visual |
| `ReportRenderer` | Iterates sections, checks conditional visibility, maps visual types to components, per-visual loading/error states |
| Layout components | `CardRow`, `FullWidth`, `Grid`, `TabsSection` |
| `SlicerPanel` | Date range, dimension filters, slicer pills, parameters, group-by, compare, search |

**Files:**
```
beastinsights/
  stores/reportFilterStore.js, reportUIStore.js
  hooks/useReportData.js
  components/
    ReportRenderer.jsx, SlicerPanel.jsx
    visuals/CardVisual.jsx, ChartVisual.jsx, TableVisual.jsx,
            MatrixVisual.jsx, WaterfallVisual.jsx, ParameterInput.jsx,
            HealthIndicator.jsx, VisualSkeleton.jsx, VisualError.jsx
    layouts/CardRow.jsx, FullWidth.jsx, Grid.jsx, TabsSection.jsx
```

**Verify:** Each visual type renders correctly with sample data. Changing Zustand state triggers refetch. Dark mode works. Mobile stacks vertically. One visual error doesn't crash the page.

---

### Milestone 5: Data Library from Power BI

**Goal:** Extract every DAX measure, dimension, and slicer from `PBI-beastinsights-prod` and translate to SQL for the semantic layer.

**What gets extracted and seeded:**

| What | Count | Destination Table |
|------|-------|-------------------|
| Metrics (DAX → SQL) | ~100 | `report_builder.metrics` |
| Dimensions (columns + JOINs) | ~30 | `report_builder.dimensions` |
| Datasets (materialized views) | 10 | `report_builder.data_sources` |
| Toggles | 6 | `report_builder.metric_toggles` |
| Toggle options | ~20 | `report_builder.metric_toggle_options` |
| Metric variants (SQL overrides) | ~50 | `report_builder.metric_variants` |
| Metric-dataset mappings | ~500 | `report_builder.data_source_metrics` |
| Dimension-dataset mappings | ~200 | `report_builder.data_source_dimensions` |

**Metric categories:** Sales & Volume, Rates (derived), Sales Type Segmentation, Disputes (date_basis toggle), Profitability (parameterized), Decline & Recovery, Cohort & LTV, Hourly/Real-Time, MID Health, Alerts.

**Toggle system:**

| Toggle | Options | Affects |
|--------|---------|---------|
| `approval_mode` | standard, organic, net | approvals, rates, initials, rebills |
| `date_basis` | transaction_date, cb_date, refund_date | CB counts, refund counts |
| `time_granularity` | day, week, month | GROUP BY dimension |
| `profitability_view` | gross, net, detailed | profit section visibility |
| `retention_base` | initials, rebills | cohort retention denominator |
| `display_mode` | count, dollar, percentage | MID summary display |

**Critical requirement:** Native reports must produce the exact same numbers as Power BI for the same client, date range, and slicer settings.

**Files:**
```
beastinsights-backend/
  prisma/seed-report-builder.js
```

**Verify:** `report_builder.metrics` has ~100 rows with valid SQL. Variant lookup `(approvals, approval_mode, organic)` returns `SUM(approvals_organic)`. Derived formula expansion works.

---

### Milestone 6: Reports, Filters & Navigation

**Goal:** Create all 11 stock report JSON templates, wire the slicer panel, build navigation.

**11 reports:**

| Report | Category | Primary Source | Key Features |
|--------|----------|---------------|--------------|
| Business Command Center | Monitor | `order_summary` + `hourly_revenue` | Summary cards, hourly chart (today vs 7d avg), auto-refresh 5 min |
| Real-Time Pulse | Monitor | `hourly_revenue` | Fixed to "today", running totals, hourly breakdown, auto-refresh |
| Revenue Analytics | Grow | `order_summary` | Slicers: approval_mode, time_granularity. Group-by. Stacked trend chart, breakdown table. |
| Subscription Intelligence | Grow | `order_summary` | MRR cards, stacked revenue trend, monthly subscriber flow |
| Customer Lifecycle | Grow | `cohort_summary` + `order_summary` | LTV cards, cohort retention matrix (heatmap), LTV by channel |
| Churn Analysis | Retain | `order_summary` | Cancel/CB/refund rate cards (inverted trends), churn trend, breakdown |
| Payment Health & Recovery | Retain | `order_summary` + `decline_recovery` | Approval rate, recovery table by decline group |
| Channel & Acquisition | Acquire | `order_summary` | 90-day default, channel scorecard with approval/cancel/CB rates |
| Financial Performance | Profit | `order_summary` | Parameters (processing %, reserve %, CB fee, CPA). Waterfall chart. |
| Product & Plan Performance | Profit | `order_summary` | Product scorecard, stacked revenue by product over time |
| Transaction Explorer | Explore | `order_details` | Search, status filter pills, server-side pagination, column toggle |

**APIs:**

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/reports/templates` | List available reports for sidebar |
| `GET /api/v1/reports/templates/:key` | Full template JSON for rendering |
| `GET /api/v1/filters/options/:dimensionKey` | Distinct dimension values for filter dropdowns |

**Slicer panel features:** Date range picker (presets + custom), dimension filter dropdowns, slicer pill groups, parameter inputs, group-by selector, compare toggle, search bar, status filter pills.

**Navigation:** Sidebar built with shadcn. Reports organized by category: Monitor, Grow, Retain, Acquire, Profit, Explore.

**Dynamic report route:** `/reports/[reportSlug]` — fetches template, creates Zustand store, renders SlicerPanel + ReportRenderer.

**Files:**
```
beastinsights-backend/
  routes/templateRoutes.js, filterOptionRoutes.js
  controller/templateController.js, filterOptionController.js
  services/templateService.js, filterOptionService.js
  prisma/seed-templates.js

beastinsights/
  app/(user-type)/(user)/reports/[reportSlug]/page.js
  components/sidebar/AppSidebar.jsx
```

**Verify:** Navigate to each report → renders with real data. Change date range → all visuals update. Toggle approval_mode → numbers change. Filter by campaign → data filters. Transaction Explorer paginates. Command Center auto-refreshes.

---

### Milestone 7: End-to-End Testing & Rollout

**Goal:** Validate performance, verify data accuracy against Power BI, harden error handling, prepare for launch.

**Performance target:** Every report loads in <500ms (click to rendered data).

| Step | Target |
|------|--------|
| HTTP + auth | ~25ms |
| SQL generation | ~15ms |
| PostgreSQL queries (parallel) | 50-100ms |
| Response formatting | ~20ms |
| React rendering | 100-200ms |
| **Total** | **~250-400ms** |

**Data accuracy validation:** For each of the 11 reports, compare every number against Power BI for the same client, date range, and slicer settings. Numbers must match exactly. No client gets access until accuracy is verified.

**Frontend optimization:** `React.lazy` for visuals, `React.memo` for preventing re-renders, `react-window` for large tables, template prefetch on sidebar hover.

**Error resilience:** Per-visual error boundaries, query timeout handling ("try a narrower date range"), retry buttons, cached data fallback.

**Mobile:** Card rows stack, charts resize, tables scroll horizontally with sticky first column, slicer panel collapses into a drawer.

**Monitoring:** Query execution times, error rates per dataset, P50/P95/P99 latencies.

**Rollout plan:**
1. Deploy frontend + backend to new subdomain
2. Internal team validates all 11 reports for 1 week
3. 2-3 beta clients get access
4. Gather feedback, fix issues
5. Roll out to remaining clients
6. Decommission Power BI workspace

### What You Have at the End of Phase 1

```
11 interactive reports with real data
~100 queryable metrics across 10 datasets
~30 filterable dimensions
6 metric toggles with automatic variant cascading
Dynamic filters, group-by, compare to previous period
Charts: line, bar, area, pie, stacked bar, waterfall
Sortable, paginated tables with column toggle and totals
Cohort retention heatmaps
Parameterized profitability calculations
Auto-refreshing real-time reports
Server-side paginated transaction explorer with search
Dark mode and mobile responsive
<500ms page load
Validated against Power BI
```

---

## Phase 2: User Customization & Self-Service

> Let users personalize the platform, build their own reports, and control who sees what.

### 2.1 User-Based Report Access Management

**What:** Admins control which reports each user or role can access.

**How it works:**
- `user_report_access` table maps users/roles to allowed report templates
- When a user logs in, the sidebar only shows reports they have access to
- Template API filters results based on user's access rules
- Default: all reports visible. Admin can restrict per user or per role.
- Super Admins and Admins always see everything

**Admin UI:**
- Settings page → Report Access tab
- Select a user or role → check/uncheck which reports they can see
- Bulk assign: "Give all users with role USER access to these 5 reports"

**API endpoints:**
| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/access/reports` | List user's accessible reports |
| `PUT /api/v1/admin/access/reports/:userId` | Set report access for a user |
| `PUT /api/v1/admin/access/reports/role/:roleId` | Set report access for a role |

---

### 2.2 Custom Report Creation

**What:** Users build their own reports by selecting metrics and dimensions from the catalog.

**How it works:**
- Report builder UI: drag metrics/dimensions from the catalog into a canvas
- User picks visual type (card, chart, table) per section
- Picks dataset, slicers, default filters
- Saves as a custom report template (`report_templates` table with `template_type = 'custom'`)
- Custom reports appear in the sidebar under "My Reports"
- Optional: share with other users in the same client

**Builder UI features:**
- Metric catalog browser (searchable, grouped by category)
- Dimension catalog browser
- Visual type picker
- Live preview (renders as you build)
- Save, edit, duplicate, delete

**API endpoints:**
| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/reports/templates` | Create custom report |
| `PUT /api/v1/reports/templates/:id` | Update custom report |
| `DELETE /api/v1/reports/templates/:id` | Delete custom report |
| `GET /api/v1/catalog/metrics` | Browse metric catalog |
| `GET /api/v1/catalog/dimensions` | Browse dimension catalog |

---

### 2.3 Filter Presets

**What:** Named, shareable filter combinations that can be applied to any report.

**How it works:**
- User sets up filters (date range, campaigns, products, etc.) → clicks "Save Preset"
- Names the preset (e.g., "Q4 Top Campaigns")
- Preset appears in a dropdown on the slicer panel
- Selecting a preset applies all saved filter values at once
- Optional: mark as shared → visible to all users in the same client
- Works across reports (preset stores filter values, not report-specific config)

**API endpoints:**
| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/filters/presets` | List user's presets (+ shared) |
| `POST /api/v1/filters/presets` | Create preset |
| `PUT /api/v1/filters/presets/:id` | Update preset |
| `DELETE /api/v1/filters/presets/:id` | Delete preset |

---

### 2.4 Bookmarks (Save Views)

**What:** Per-user saved state of a specific report — filters, toggles, layout customizations.

**How it works:**
- User opens Revenue Analytics, applies filters, changes toggles, hides some columns
- Clicks "Save Bookmark" → names it (e.g., "My Daily View")
- Next time they open Revenue Analytics, they can select the bookmark to restore that exact state
- One bookmark per report can be set as "default" — auto-loads when opening that report
- Bookmarks are per-user, per-report, per-client

**What gets saved:**
- Active filters (date range, dimension filters)
- Toggle/slicer states
- Parameter values
- Group-by selection
- Visual visibility/order overrides

**API endpoints:**
| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/bookmarks?templateId=X` | List bookmarks for a report |
| `POST /api/v1/bookmarks` | Create bookmark |
| `PUT /api/v1/bookmarks/:id` | Update bookmark |
| `DELETE /api/v1/bookmarks/:id` | Delete bookmark |
| `PUT /api/v1/bookmarks/:id/default` | Set as default |

---

### 2.5 Notifications

**What:** Threshold-based alerts that notify users when metrics cross defined boundaries.

**How it works:**
- User creates a notification rule: "Alert me when CB Rate exceeds 1% for any gateway"
- System evaluates rules on a schedule (e.g., every hour or after each data refresh)
- When a threshold is crossed, notification is sent via configured channel

**Rule definition:**
- Select a metric (from catalog)
- Select a dimension to group by (optional — e.g., "per gateway")
- Set condition: `>`, `<`, `>=`, `<=`, `crosses above`, `crosses below`
- Set threshold value
- Set frequency: real-time (on refresh), hourly, daily
- Set delivery: in-app, email, Telegram

**Tables:**
- `notification_rules` — rule definitions (metric, dimension, condition, threshold, frequency, delivery channel)
- `notification_log` — delivery history (rule_id, triggered_at, metric_value, delivery_status)

**API endpoints:**
| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/notifications/rules` | List user's notification rules |
| `POST /api/v1/notifications/rules` | Create rule |
| `PUT /api/v1/notifications/rules/:id` | Update rule |
| `DELETE /api/v1/notifications/rules/:id` | Delete rule |
| `GET /api/v1/notifications/log` | Notification history |

---

### 2.6 Export & Scheduled Delivery

**What:** Export any report to CSV, Excel, or PDF. Set up scheduled delivery via email or Telegram.

**Export:**
- User clicks "Export" on any report → selects format (CSV, Excel, PDF)
- Export runs with current filter state
- File is generated asynchronously → download link provided
- Tracked in `export_jobs` table

**Scheduled delivery:**
- User sets up a schedule: "Send me Revenue Analytics as PDF every Monday at 8am EST"
- Optionally locks specific filter values
- System generates the report on schedule and delivers via email or Telegram
- Tracked in `scheduled_reports` table

---

### 2.7 Personal Dashboards

**What:** Users pin visuals from any report onto a personal dashboard.

**How it works:**
- While viewing a report, user clicks "Pin to Dashboard" on any visual
- Visual is copied to their personal dashboard with its data config
- Dashboard uses `react-grid-layout` for drag-and-drop positioning
- Each dashboard tile queries independently (uses the same batch query API)

### Phase 2 Summary

| Feature | Tables Used |
|---------|------------|
| Report access management | `user_report_access` |
| Custom report creation | `report_templates` (template_type = 'custom') |
| Filter presets | `filter_presets` |
| Bookmarks (save views) | `bookmarks` |
| Notifications | `notification_rules`, `notification_log` |
| Export | `export_jobs` |
| Scheduled delivery | `scheduled_reports` |
| Personal dashboards | `dashboards`, `dashboard_tiles` |

---

## Phase 3: Row-Level Security

> Users only see data for their assigned products, campaigns, gateways, etc.

### 3.1 What It Is

Dimension-level access control. An admin assigns a user to specific dimension values, and the query engine automatically injects WHERE clauses into every query — invisible to the user.

**Example:** User `john@client.com` is restricted to Gateway 7 and Campaigns 101, 205. Every report, every visual, every export they access automatically filters to only those gateways and campaigns. They never see data outside their permitted scope.

### 3.2 How It Works

**`user_dimension_access` table:**
- Maps `user_id` + `dimension_id` → `allowed_values` (JSONB array)
- `access_type`: `include` (only these values) or `exclude` (everything except)
- If no entry exists for a user/dimension → user sees all values (no restriction)

**Query engine integration:**
1. On every query, engine checks `user_dimension_access` for the requesting user
2. For each restricted dimension, injects `WHERE dimension_column IN (allowed_values)`
3. This happens after normal filter building — user's own filter choices are further constrained by their access rules
4. Zero code changes to reports or visuals — access control is purely at the query layer

### 3.3 Admin Features

| Feature | Description |
|---------|------------|
| Assign dimension access | Admin selects a user → selects a dimension (e.g., campaign) → selects allowed values |
| Cascade rules | Access to Campaign A → automatically see all products under that campaign |
| "View As" mode | Admin can preview what a restricted user sees without switching accounts |
| Client-level defaults | New users inherit default dimension access rules set at the client level |
| Audit logging | Every access change is logged in `dimension_access_audit` (who changed what, when, old/new values) |

### 3.4 API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/admin/access/dimensions/:userId` | View user's dimension access |
| `PUT /api/v1/admin/access/dimensions/:userId` | Set dimension access for a user |
| `DELETE /api/v1/admin/access/dimensions/:userId/:dimensionId` | Remove a restriction |
| `GET /api/v1/admin/access/audit` | View audit log |
| `POST /api/v1/admin/access/view-as` | Preview as restricted user |

### 3.5 Tables

| Table | Purpose |
|-------|---------|
| `user_dimension_access` | Maps users to allowed dimension values |
| `dimension_access_audit` | Audit trail of all access changes |

---

## Phase 4: Industry Modules & Expansion

> Opt-in specialized report packs per industry vertical.

### 4.1 Module System

- `client_modules` table stores which modules are enabled per client
- Enabling a module adds industry-specific metrics, dimensions, and reports to the semantic layer
- Navigation adapts — module reports appear in a new sidebar section
- Each client gets the 11 universal reports + their industry module reports

### 4.2 Modules

| Module | Reports | Examples |
|--------|---------|---------|
| **High-Risk** | 6 | MID Health & Compliance, BIN Routing Optimizer, Decline Recovery Intelligence, Alert Service Management, Gateway Performance, Affiliate Quality |
| **Telehealth** | 5 | Patient Membership, Provider Performance, Visit-to-Payment, Insurance vs Self-Pay, Compliance |
| **SaaS** | 5 | ARR Tracking, Trial-to-Paid, Seat/Usage Analytics, Plan Migration, Net Dollar Retention |
| **eCommerce** | 5 | Order Frequency, Returns, Seasonal Demand, Basket Analysis, Delivery & Fulfillment |

### 4.3 What's Needed Per Module

- New metrics added to `report_builder.metrics` (industry-specific calculations)
- New dimensions added to `report_builder.dimensions` (if applicable)
- New report templates added to `report_builder.report_templates`
- Possibly new materialized views for module-specific data aggregations
- Navigation updates to show module section when enabled

### 4.4 Table

| Table | Purpose |
|-------|---------|
| `client_modules` | Per-client module enablement (client_id, module_key, is_enabled, config) |

---

## Appendix: Phase Summary

### Phase 1 — Core Reporting Engine
- `report_builder` schema (all tables)
- Query engine (single + batch API)
- Frontend renderer (7 visual types + layouts)
- Data library extracted from PBI
- 11 stock reports with full filter/slicer support
- <500ms page load, validated against Power BI

### Phase 2 — User Customization & Self-Service
- User-based report access management
- Custom report builder
- Filter presets (save & share)
- Bookmarks (save views per report)
- Notifications (threshold alerts)
- Export (CSV, Excel, PDF)
- Scheduled delivery (email, Telegram)
- Personal dashboards (pin tiles)

### Phase 3 — Row-Level Security
- Dimension-level access control per user
- Auto-injected WHERE clauses in query engine
- Admin UI for assignment
- Cascade rules
- "View As" mode
- Audit logging

### Phase 4 — Industry Modules
- Module enablement system
- High-Risk, Telehealth, SaaS, eCommerce report packs
- Industry-specific metrics/dimensions
- Adaptive navigation

---

*Beast Insights — Product Roadmap*

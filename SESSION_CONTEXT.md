# Beast Insights — Session Context

> **Purpose:** Feed this document to every new AI session. It contains the full understanding of the project, current architecture, what's changing, and where all decisions are documented.

> **Last updated:** 2026-02-02

---

## 1. What Is Beast Insights

Beast Insights (`app.beastinsights.co`) is a **multi-tenant payment analytics platform**. Clients (merchants) connect their CRM data (Sticky, Konnektive, VRIO, etc.) and Beast Insights provides reports on revenue, approvals, chargebacks, refunds, retention, LTV, profitability, and more.

**Current clients:** ~42 high-risk subscription merchants (supplements, skincare, digital products). Expanding to telehealth, SaaS, and other subscription verticals.

**The migration:** Replacing Power BI Embedded with a native React reporting system on a **separate subdomain**. Power BI currently takes 5-10 seconds to load. Target: <500ms.

---

## 2. Repositories

| Repo | Purpose | Status |
|------|---------|--------|
| `beast-insights-backend` | Current production backend (Express.js) | Existing, running |
| `beastinsights-backend` | **New backend** for native reporting | ✅ SET UP — all routes from `beast-insights-backend` + new query engine |
| `beastinsights` | **New frontend** for native reporting | New, empty — separate subdomain |
| `beast-ideation` | Planning & documentation repo | This repo |
| `beastinsights-old` | Old frontend reference | Design reference for colors, UI patterns |
| `PBI-beastinsights-prod` | Power BI production files | Source for DAX measures → SQL translation |

**Key decision:** New repos, not modifications to existing ones. The native reporting platform deploys on a **separate subdomain** — no feature flags, no coexistence logic, no `report_mode` column. Power BI continues running on the existing domain until the native platform is validated and clients are migrated.

---

## 3. Tech Stack (New Platform)

| Layer | Technology | Notes |
|-------|-----------|-------|
| Frontend | Next.js 16 + React 19 | Repo: `beastinsights` |
| Analytics UI | **Tremor** | Cards, charts, tables — primary visual library |
| Navigation UI | **shadcn/ui** + Radix + Tailwind | Sidebar, inputs, popovers, form elements |
| Tables | TanStack React Table v8 | Tremor's Table wraps this |
| State (client) | **Zustand** | All client state: auth, filters, UI state |
| State (server) | **TanStack Query** | Data fetching + frontend cache (5-min stale) |
| Backend | Express.js 4.19 | Repo: `beastinsights-backend` |
| ORM | Prisma 5.19 | CRUD operations only (users, clients, templates) |
| Query Engine | `pg` driver directly | Dynamic SQL generation — NOT through Prisma |
| Database | PostgreSQL | Shared database, new `report_builder` schema |
| Auth | JWT HS512 + bcrypt | Brought over from `beast-insights-backend` |

**What's NOT in the stack:**
- No Redux (Zustand replaces it entirely in the new repo)
- No Redis in Phase 1 (TanStack Query handles frontend caching; Redis is a future optimization)
- No Recharts (Tremor replaces it)
- No Power BI anything

**Database approach:** Write raw SQL files to create the `report_builder` schema. Execute SQL directly. Then run `prisma db pull` to introspect and generate Prisma models. No Prisma migration.

---

## 4. Terminology

We use **Power BI terminology** everywhere — code, database, documents, UI.

| Concept | Term We Use | NOT This |
|---------|------------|----------|
| A chart/card/table on a report | **Visual** | Widget |
| The filter/toggle area at the top | **Slicer Panel** | FilterBar |
| A context switch (approval_mode, date_basis) | **Slicer** (UI) / **Toggle** (DB) | Toggle is only used for DB table names |
| A summary metric card | **Card** | MetricCard |
| An interactive data table | **Table** | DataTable |
| A cohort/retention heatmap matrix | **Matrix** | PivotTable |
| A P&L waterfall visualization | **Waterfall** | WaterfallChart |
| A saved report state per user | **Bookmark** | Saved view |
| A pinned visual on a dashboard | **Tile** | Dashboard widget |
| The JSON that defines a report | **Report template** | — |
| A materialized view mapping | **Dataset** | Data source (in descriptive text) |

**Component file names:** `CardVisual.jsx`, `ChartVisual.jsx`, `TableVisual.jsx`, `MatrixVisual.jsx`, `WaterfallVisual.jsx` — all in `components/visuals/`.

---

## 5. Database Architecture

### 5.1 Connection (Read-Only for Analysis)

```
Host: db.beastinsights.com
Port: 5432
Database: postgres
User: beastinsights_ro
Password: 5QwxpH889T3qansugXQA
```

### 5.2 Existing Schemas (Unchanged)

| Schema | Purpose |
|--------|---------|
| `beast_insights_v2` | App layer: clients, users, roles, pages, reports, navigation, filters, metrics, alerts, goals |
| `data` | Raw transactional data: `orders_{client_id}` (81 columns each) |
| `public` | Shared dimension tables: `campaigns`, `bin_lookup`, `offers`, `decline_groups` |
| `reporting` | Materialized views per client: 10 types × ~42 clients = ~348 views |
| `beastinsights_alert` | Alert-specific data |
| `routing` | Routing data |

### 5.3 New Schema: `report_builder`

**33 tables total.** All created upfront (Phase 2-4 tables remain empty until needed).

**Phase 1 tables (11 tables, populated via seed):**

| Table | Purpose | Rows |
|-------|---------|------|
| `data_sources` | Maps logical names to materialized view patterns | ~10 |
| `metrics` | Metric catalog with SQL expressions, comparison/trend flags, decimal places | ~100 |
| `dimensions` | Dimension + filter catalog (31 columns). Core: columns, JOINs, filter metadata. Options loading: `options_loading` (static/full/paginated/top_n/search_only), `options_cache_ttl`, `options_sort`. Selection: `default_selection` (all/none/specific), `max_selections`. UI: `ui_config` JSONB catch-all (~25 keys). Supports 8 filter types: multi_select, single_select, date_range, search, boolean, nested, tree, input | ~30 |
| `filter_types` | DB-driven filter type registry. 8 core types seeded. New filter types via INSERT without code changes. Stores display_component (React component name), config_schema (JSON Schema), default_config. | 8 |
| `data_source_metrics` | Junction: which metrics work with which datasets (with optional `sql_expression_override` per source) | ~500 |
| `data_source_dimensions` | Junction: which dimensions work with which datasets | ~200 |
| `metric_toggles` | 6 toggle/slicer definitions | 6 |
| `metric_toggle_options` | Options per toggle | ~20 |
| `metric_variants` | SQL overrides per metric/toggle/option | ~50 |
| `report_templates` | JSON report layouts with 3-layer system (stock/client/custom), `user_id` for ownership, `parent_template_id` for duplication lineage, `version` for change tracking | 11 |
| `report_template_filters` | Base filter assignments per report (15 columns). Which dimensions appear on each report, display_order, defaults. Per-report overrides: `filter_type_override`, `max_selections_override`, `ui_config_override` | ~200 |

**Phase 2 tables (18 tables, created empty):**

| Table | Purpose |
|-------|---------|
| `bookmarks` | Per-user saved report state (filters, toggles, layout) |
| `filter_presets` | Named filter combinations |
| `dashboards` | Personal dashboard layouts |
| `dashboard_tiles` | Pinned visuals on dashboards |
| `scheduled_reports` | Automated report/dashboard delivery (template_id OR dashboard_id, bookmark_id, toggles) |
| `export_jobs` | Export tracking with filter + toggle state (supports reports and dashboards) |
| `user_report_access` | Report visibility per user/role (partial unique indexes) |
| `notification_rules` | Threshold alerts (multi-channel via `delivery_channels` array) |
| `notification_log` | Notification delivery history (with `is_read` for in-app) |
| `report_template_shares` | Share custom reports with specific users (permission: view/edit/duplicate) |
| `bookmark_shares` | Share bookmarks with specific users |
| `filter_preset_shares` | Share filter presets with specific users |
| `dashboard_shares` | Share dashboards with specific users (permission: view/edit/duplicate) |
| `client_template_filter_overrides` | Client-level filter cascade overrides (admin hides/reorders/defaults filters per client) |
| `user_template_filter_overrides` | User-level filter cascade overrides (highest priority — personal filter preferences) |
| `client_report_access` | Client-level report visibility (dynamic equivalent of `client_features`) |
| `user_preferences` | Per-user platform settings as JSONB (timezone, default date range, landing page, etc.) |
| `client_toggle_overrides` | Per-client toggle option visibility (hide irrelevant toggle options per client) |

**Phase 3 tables (3 tables, created empty):** `user_dimension_access`, `client_dimension_access`, `dimension_access_audit`

**Phase 4 tables (1 table, created empty):** `client_modules`

**Three-layer template system:**
- `stock` → common (client_id=NULL, user_id=NULL) — visible to all
- `client` → client admin created (client_id=SET, user_id=NULL) — visible to all users of that client
- `custom` → user created (client_id=SET, user_id=SET) — visible only to that user until shared

**3-level filter cascade:**
1. `report_template_filters` → base config (which filters on which report)
2. `client_template_filter_overrides` → client admin overrides (NULL = inherit base)
3. `user_template_filter_overrides` → user personal overrides (NULL = inherit, highest priority)

**Dimension access cascade:**
1. `client_dimension_access` → client-level restrictions (all users of client)
2. `user_dimension_access` → user-level restrictions (intersection with client = most restrictive)

**Sharing model:** Custom reports, bookmarks, filter presets, and dashboards support per-user sharing via junction tables (NOT blanket `is_shared` boolean). Each sharing table stores `shared_with_user_id`, `shared_by_user_id`, `client_id`, and permission level. Query pattern: own items `UNION ALL` shared items.

**Report access cascade:**
1. `client_report_access` → is report enabled for this client? (default open — no row = visible)
2. `user_report_access` → is report enabled for this user/role?
3. `client_modules` → is the module enabled? (module reports only)

### 5.4 Materialized Views (10 Types)

These already exist in the `reporting` schema. One instance per client (e.g., `order_summary_10043`).

| View | Columns | Purpose |
|------|---------|---------|
| `order_summary_{id}` | 37 | Core aggregated data — powers most reports |
| `mid_summary_{id}` | 51 | MID health, monthly snapshots per gateway |
| `cohort_summary_{id}` | 25 | Retention & LTV cohorts by billing cycle |
| `cb_refund_alert_{id}` | 33 | CB/refund growth with days-since columns |
| `decline_recovery_{id}` | 27 | Decline retry analysis |
| `hourly_revenue_{id}` | 11 | Real-time hourly revenue (today vs 7d avg) |
| `alert_summary_{id}` | 31 | Alert aggregations (RDR/Ethoca/CDRN) |
| `alert_details_{id}` | 31 | Individual alert records |
| `order_details_{id}` | 84 | Transaction-level detail for explorer |
| `products_{id}` | 21 | Product dimension per client |

### 5.5 Key Existing Tables

| Table | Key Columns |
|-------|------------|
| `beast_insights_v2.users` | `id` (VARCHAR PK), `email`, `role_id`, `client_id` |
| `beast_insights_v2.clients` | `id` (BIGINT autoincrement), `name`, `is_active`, `package_id` |
| `beast_insights_v2.roles` | 4 roles: SUPER_ADMIN (level 1), CONSULTANT (level 2), ADMIN (level 3), USER (level 4) |
| `public.campaigns` | `campaign_id`, `campaign_name`, `campaign_type`, `crm`, `client_id`, `cpa` |
| `public.bin_lookup` | `bin`, `bank`, `card`, `type`, `class`, `country` |

### 5.6 Naming Conventions (from Existing DB)

| Element | Convention |
|---------|-----------|
| Tables | `snake_case`, plural |
| Columns | `snake_case` |
| PK | `id BIGINT` autoincrement, constraint: `{table}_pk` |
| FK column | `{referenced_table_singular}_id` |
| FK constraint | `{table}_{referenced_table}_id_fk` |
| Booleans | `is_` prefix (`is_active`, `is_deleted`) |
| Timestamps | `created_at`, `updated_at` as `TIMESTAMP(6)` |
| Audit | `created_by VARCHAR`, `modified_by VARCHAR` → FK to `users.id` |

---

## 6. Architecture: How It Works

### 6.1 Core Concept: Semantic Layer + JSON Templates

- **Semantic Layer (DB):** Defines all metrics (with SQL expressions), dimensions (with column mappings), toggles (with variant SQL overrides)
- **JSON Templates (DB):** Define report layouts — which visuals, which metrics per visual, chart types, slicer configs, section layouts
- **Query Engine (Node.js):** Translates metric/dimension/filter/toggle combinations into parameterized SQL against materialized views. ONE generic endpoint, no report-specific APIs.
- **Report Renderer (React):** Reads JSON template, builds batch query from visual configs + current slicer state, renders using generic visual components

### 6.2 End-to-End Flow

```
1. User navigates to "Revenue Analytics"
2. Frontend fetches report template JSON: GET /api/v1/reports/templates/revenue-analytics
3. Template defines: 4 cards, 1 area chart, 1 breakdown table
4. Frontend reads each visual's config, extracts metric/dimension keys
5. Frontend combines with current slicer state (date range, toggles, filters) from Zustand
6. Frontend sends ONE batch request: POST /api/v1/query/batch
7. Backend Query Engine for each visual query:
   a. Resolves metric keys → SQL expressions (checking for toggle variants)
   b. Handles derived metrics (approval_rate = approvals / attempts — cascades variants)
   c. Resolves dimension keys → column names + required JOINs
   d. Resolves filters → parameterized WHERE clauses (always includes client_id from JWT)
   e. Assembles SQL, executes against reporting.order_summary_{clientId}
8. All queries run in parallel via Promise.all(), results keyed by visualId
9. Frontend distributes results to visuals, each renders by type
10. Total time: <500ms
```

### 6.3 Metric Variant System (The Hardest Design Problem)

Some metrics change their SQL based on slicer state:

| Slicer | Options | What Changes |
|--------|---------|--------------|
| `approval_mode` | standard, organic, net | `SUM(approvals)` vs `SUM(approvals_organic)` vs `SUM(net_approvals)` |
| `date_basis` | transaction_date, cb_date, refund_date | `SUM(cb)` vs `SUM(cb_cb_date)` |
| `time_granularity` | day, week, month | GROUP BY dimension changes |
| `profitability_view` | gross, net, detailed | Section visibility |
| `retention_base` | initials, rebills | Cohort denominator |
| `display_mode` | count, dollar, percentage | MID summary display |

**Derived metric cascading:** When `approval_mode = organic`, `approvals` uses `SUM(approvals_organic)`. Any derived metric using `approvals` (like `approval_rate = approvals / attempts`) automatically picks up the organic variant without explicit mapping. The query engine resolves base metrics first, then substitutes into derived formulas.

### 6.4 API Design

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/query` | Single visual query |
| `POST /api/v1/query/batch` | All visuals for a page (one HTTP call) |
| `GET /api/v1/reports/templates` | List available reports (for sidebar) |
| `GET /api/v1/reports/templates/:key` | Full template JSON (for rendering) |
| `GET /api/v1/filters/options/:dimensionKey` | Distinct values for filter dropdowns |

**Security:** All values parameterized (no SQL concatenation), `client_id` extracted from JWT (never from request body), only catalog-registered columns queryable, 30s query timeout.

---

## 7. Development Phases

### Phase 1: Core Reporting Engine (7 Milestones)

Build the engine that converts JSON templates into interactive reports.

| Milestone | What | Dependencies |
|-----------|------|-------------|
| M1 | Database architecture — create all `report_builder` tables | None |
| M2 | JSON template format specification (design doc, no code) | M1 |
| M3 | Backend query engine + batch API | M1, M2 |
| M4 | Frontend report renderer (all visual types + layouts) | M2 (can start with mock data) |
| M5 | Data library — extract DAX from PBI, translate to SQL, seed DB | M1 |
| M6 | Reports, filters & navigation — 11 templates, slicer panel, sidebar | M3, M4, M5 |
| M7 | End-to-end testing, performance validation, rollout prep | M6 |

M3, M4, M5 can run in parallel. Backend builds query engine, frontend builds renderer with mock data, analyst extracts PBI data library.

**Deliverable:** 11 interactive reports with real data, dynamic filters, slicers, <500ms load. Validated against Power BI numbers.

### Phase 2: User Customization & Self-Service

| Feature | Description |
|---------|------------|
| Report access management | Admin controls which reports each user/role can see |
| Custom report creation | Users build reports from metric/dimension catalog; share with specific users (view/edit/duplicate) |
| Filter presets | Named filter combinations; shareable with specific users |
| Bookmarks (save views) | Per-user saved report state (filters, toggles, layout); shareable with specific users |
| Notifications | Threshold-based alerts when metrics cross boundaries; multi-channel delivery (email, in-app, Telegram) |
| Export | CSV, Excel, PDF with filter + toggle state preserved |
| Scheduled delivery | Automated report delivery via email/Telegram; can schedule a specific bookmark |
| Personal dashboards | Pin tiles from any report onto a personal dashboard |

### Phase 3: Row-Level Security

Dimension-level access control. Admin assigns a user to specific campaigns/gateways/products → query engine auto-injects WHERE clauses → user only sees their permitted data across all reports.

Includes: cascade rules, "View As" mode for admins, audit logging.

### Phase 4: Industry Modules

Opt-in report packs per vertical: High-Risk (6 reports), Telehealth (5), SaaS (5), eCommerce (5). Module enablement per client. Navigation adapts.

---

## 8. The 11 Stock Reports

| # | Report | Category | Primary Source | Special Features |
|---|--------|----------|---------------|-----------------|
| 1 | Business Command Center | Monitor | `order_summary` + `hourly_revenue` | Auto-refresh 5 min, today vs 7d avg |
| 2 | Real-Time Pulse | Monitor | `hourly_revenue` | Fixed to "today", auto-refresh |
| 3 | Revenue Analytics | Grow | `order_summary` | Slicers: approval_mode, time_granularity. Group-by. |
| 4 | Subscription Intelligence | Grow | `order_summary` | MRR cards, stacked revenue trend |
| 5 | Customer Lifecycle | Grow | `cohort_summary` + `order_summary` | LTV cards, cohort matrix (heatmap) |
| 6 | Churn Analysis | Retain | `order_summary` | Inverted trends (lower = green) |
| 7 | Payment Health & Recovery | Retain | `order_summary` + `decline_recovery` | Recovery table by decline group |
| 8 | Channel & Acquisition | Acquire | `order_summary` | 90-day default, channel scorecard |
| 9 | Financial Performance | Profit | `order_summary` | Parameter inputs (fees, reserve). Waterfall chart. |
| 10 | Product & Plan Performance | Profit | `order_summary` | Product scorecard, stacked revenue by product |
| 11 | Transaction Explorer | Explore | `order_details` | Search, status pills, server-side pagination |

---

## 9. Visual Types

| Visual | Component | Library | Notes |
|--------|-----------|---------|-------|
| Card | `CardVisual.jsx` | Tremor | 1-3 metrics + trend arrow |
| Chart | `ChartVisual.jsx` | Tremor | Line, bar, area, stacked bar, pie, donut |
| Table | `TableVisual.jsx` | TanStack Table | Sortable, paginated, totals, column toggle |
| Matrix | `MatrixVisual.jsx` | Custom | Cohort heatmap — Tremor doesn't have this |
| Waterfall | `WaterfallVisual.jsx` | Custom | P&L — Tremor doesn't have this |
| ParameterInput | `ParameterInput.jsx` | shadcn | Number input with $/% |
| HealthIndicator | `HealthIndicator.jsx` | Custom | Green/yellow/red status badge |

---

## 10. What's Been Decided

| Decision | Choice |
|----------|--------|
| Architecture | Semantic Layer + JSON Templates |
| Deployment | Separate subdomain (no feature flags) |
| Frontend framework | Next.js 16 + React 19 |
| Analytics UI | Tremor (not Recharts) |
| Navigation UI | shadcn/ui |
| State management | Zustand (not Redux) |
| Server state | TanStack Query |
| Database changes | Raw SQL + `prisma db pull` (no Prisma migration) |
| Backend repo | New `beastinsights-backend` (brings auth from old repo) |
| Frontend repo | New `beastinsights` (empty, separate subdomain) |
| Terminology | Power BI terms everywhere (visuals, slicers, bookmarks, tiles) |
| PBI source | `PBI-beastinsights-prod` for DAX measure extraction |
| Design reference | `beastinsights-old` for colors and UI patterns |
| Query security | Parameterized SQL, client_id from JWT only, column whitelist |
| Performance target | <500ms page load |
| Phase 1 caching | TanStack Query only (no Redis) |
| Custom components needed | Matrix, Waterfall, Hierarchical Table, Server-paginated Table |
| Existing tables | Zero modifications to any existing table |
| Template ownership | `user_id` column on `report_templates` (not just `created_by`) for explicit ownership |
| Template layers | 3-layer system: stock (common), client (client admin), custom (user-owned) |
| Template uniqueness | Partial unique indexes scoped by template_type (stock=global, client=per-client, custom=per-user) |
| Filter cascade | 3-level priority: report_template_filters → client_template_filter_overrides → user_template_filter_overrides |
| Filter types | 8 types matching existing system: multi_select, single_select, date_range, search, boolean, nested, tree, input |
| Dimension access | 2-level cascade: client_dimension_access → user_dimension_access (intersection = most restrictive) |
| Schema size | 32 tables total (up from initial 24 → 28 → 32) |
| Dashboard scheduling | `scheduled_reports` and `export_jobs` support both reports and dashboards via `dashboard_id` |
| Client report visibility | `client_report_access` table replaces `client_features` boolean columns (dynamic, scales with any number of reports) |
| User preferences | Per-user JSONB settings scoped by user + client (extensible without schema changes) |
| Toggle visibility | `client_toggle_overrides` controls per-client toggle option visibility (hide irrelevant options) |
| Dashboard sharing | `dashboard_shares` table for sharing dashboards (same pattern as other sharing tables) |

---

## 11. What's Still Open

| Question | Context |
|----------|---------|
| Subdomain URL | What domain/subdomain for the native platform? |
| Hosting/infra | Where are the new repos deployed? Same Azure setup? |
| Hierarchical group-by | Expand/collapse rows in tables — spec defined but implementation detail TBD |
| Redis | Future optimization (not Phase 1) — when to add it? |
| Report builder UI | Phase 2 — drag-and-drop design not yet specified |
| Notification delivery | Phase 2 — exact channels (in-app, email, Telegram, Slack) |
| Mobile app | Not planned yet |
| White-labeling | Not specified |
| AI insights | Not specified |

---

## 12. Documents in This Repo

### Active Planning Documents (Current)

| File | What It Contains |
|------|-----------------|
| `Product_Roadmap.md` | **Single source of truth.** All 4 phases, Phase 1 milestones (M1-M7), Phase 2-4 feature details with API endpoints. |
| `Database_Design_M1.md` | Complete schema design — 32 tables with all columns, types, constraints, ER diagrams, indexes, CREATE TABLE SQL. |
| `JSON_Template_Format_M2.md` | **M2 deliverable.** Definitive JSON template specification — TypeScript type definitions for all visual types, section types, slicer panel, cross-cutting concerns, 4 complete stock report examples, validation checklist. |
| `Development_Phase_Plan.md` | Deep technical blueprint — tech stack, database architecture with full SQL, backend architecture (query engine modules, file structure), frontend architecture (components, stores, hooks), all 11 JSON template examples, API endpoints, performance strategy. |
| `Phase_1_Milestones.md` | Detailed Phase 1 milestones — M1 through M7 with full deliverables, DAX→SQL translation tables, dimension mappings, toggle variant mappings, dataset mappings, seed data specs. |

### Reference Documents (Earlier Planning)

| File | What It Contains |
|------|-----------------|
| `Current_System_Analysis.md` | Deep analysis of current system — all materialized view schemas, application tables, Power BI DAX measures |
| `Report_Builder_Plan.md` | Original architecture options analysis (4 approaches), recommended Semantic Layer + JSON |
| `JSON_Driven_System_Design.md` | Deep dive on JSON-driven design — one generic endpoint, generic renderer, metric variants system, complete examples |
| `Cross_Industry_Expansion_Plan.md` | Cross-industry strategy — 12 universal reports, 4 industry modules, universal metrics catalog |
| `Beast_Insights_Platform_Documentation.md` | Original platform docs — all 13 current Power BI reports |
| `Product_Vision_Reports_From_Scratch.md` | Reports designed from first principles (high-risk focused) |
| `Beast_Insights_Product_Vision.md` | Non-technical marketing version |
| `Beast_Insights_Product_Overview.md` | Simplified visual version with ASCII wireframes |
| `vrio.md` | Competitor analysis — VRIO Analytics |

### Document Hierarchy

```
Product_Roadmap.md              ← Start here. Overview of everything.
├── Database_Design_M1.md       ← Detailed table specs (when building schema)
├── JSON_Template_Format_M2.md  ← JSON template spec (contract between backend & frontend)
├── Development_Phase_Plan.md   ← Technical blueprint (when writing code)
└── Phase_1_Milestones.md       ← Milestone details (when planning sprints)
```

---

## 13. Key Technical Patterns

### Adding a New Metric (No Code Deployment)

```sql
INSERT INTO report_builder.metrics (metric_key, display_name, sql_expression, data_type, format_pattern, category)
VALUES ('new_metric', 'New Metric', 'SUM(column)', 'integer', '#,##0', 'sales');

INSERT INTO report_builder.data_source_metrics (data_source_id, metric_id)
VALUES (1, currval('report_builder.metrics_id_seq'));

-- If it has toggle variants:
INSERT INTO report_builder.metric_variants (metric_id, toggle_option_id, sql_expression)
VALUES (currval('report_builder.metrics_id_seq'), 2, 'SUM(column_organic)');
```

Available immediately in all reports and the catalog. No code changes.

### Adding a New Report (No Code Deployment)

```sql
INSERT INTO report_builder.report_templates (template_key, name, category, layout, required_data_sources, nav_section)
VALUES ('new-report', 'New Report', 'analytics', '{"version":"1.0","source":"order_summary",...}', ARRAY['order_summary'], 'analytics');
```

The frontend renders it automatically. No code changes.

### How Tenant Isolation Works

Every query has `WHERE client_id = {value_from_JWT}` injected automatically. The `client_id` is extracted from the JWT token by the auth middleware — it is NEVER accepted from the request body. Combined with parameterized SQL, this prevents cross-tenant data access.

---

## 14. Working Preferences

- **Never commit or push without asking first.** Always ask before running `git commit` or `git push`.
- Use raw SQL files for schema changes (no Prisma migration).
- Reference `beastinsights-old` for UI design decisions.
- Reference `PBI-beastinsights-prod` for DAX measure extraction.

---

## 15. Current Status

**M1 SQL schema complete and expanded to 33 tables.** After three rounds of analysis — the existing filter system, PBI production reports, a comprehensive gap analysis, and architectural review — the schema was expanded from 24 → 28 → 32 → 33 tables.

**Round 1 changes (2026-01-28, committed):**
- `report_templates` — added `user_id` (ownership), `parent_template_id` (duplication lineage), `version` (change tracking); added `template_type='client'` for 3-layer system; replaced global unique constraint with 4 partial unique indexes scoped by template type
- `dimensions` — added 12 columns for filter metadata: `is_multiple`, `data_source`, `options_query`, `static_options`, `default_value`, `filter_group`, `icon`, `parent_dimension_id`, `min_value`, `max_value`, `step_value`, `input_prefix`; expanded `filter_type` to 8 values (added `boolean`, `nested`, `tree`, `input`); added CHECK constraints and 2 new indexes
- NEW: `report_template_filters` (Phase 1) — base filter assignments per report template (~200 seed rows)
- NEW: `client_template_filter_overrides` (Phase 2) — client-level filter cascade overrides
- NEW: `user_template_filter_overrides` (Phase 2) — user-level filter cascade overrides (highest priority)
- NEW: `client_dimension_access` (Phase 3) — client-level dimension value restrictions

**Round 2 changes (2026-01-28):**
- `data_sources` — added `last_refreshed_at` (data freshness tracking for UI staleness badge)
- `scheduled_reports` — added `dashboard_id` (schedule dashboard delivery); `template_id` now nullable; added CHECK constraint (template_id OR dashboard_id must be set); `delivery_channels` array → `delivery_channel` singular VARCHAR
- `export_jobs` — added `dashboard_id` (export dashboards); added FK to dashboards
- NEW: `dashboard_shares` (Phase 2) — share dashboards with specific users (permission: view/edit/duplicate)
- NEW: `client_report_access` (Phase 2) — client-level report visibility (dynamic replacement for `client_features` boolean columns)
- NEW: `user_preferences` (Phase 2) — per-user platform settings as JSONB (timezone, landing page, sidebar state, etc.)
- NEW: `client_toggle_overrides` (Phase 2) — per-client toggle option visibility (hide irrelevant options)

**Round 3 changes (2026-01-28):**
- NEW: `filter_types` (Phase 1) — DB-driven filter type registry. 8 core types seeded with display_component, config_schema, default_config. New filter types via INSERT, no code changes needed. Replaces the CHECK constraint approach.
- Table renumbering: Phase 1 = 11 tables (was 10), total = 33 tables (was 32)

**Round 4 changes (2026-01-29) — Comprehensive Filter System Design:**

After deep analysis of 17 filter scenarios (loading, display, search, cascading, caching, validation, etc.), the `dimensions` and `report_template_filters` tables were expanded to ensure no future schema changes needed for filters.

*`dimensions` table — new columns (6):*
| Column | Type | Default | Purpose |
|--------|------|---------|---------|
| `options_loading` | VARCHAR(20) | `'full'` | Backend SQL branching: `static`, `full`, `paginated`, `top_n`, `search_only` |
| `options_cache_ttl` | INTEGER | `300` | Seconds. Returned in API → frontend TanStack Query `staleTime` |
| `default_selection` | VARCHAR(20) | `'all'` | WHERE clause logic: `'all'`=no filter, `'none'`=must pick, `'specific'`=use default_value |
| `max_selections` | INTEGER | NULL | Backend validation. Prevents massive IN clauses. NULL=unlimited |
| `options_sort` | VARCHAR(20) | `'alphabetical'` | SQL ORDER BY: `alphabetical`, `frequency`, `recent` |
| `ui_config` | JSONB | `'{}'` | **Catch-all** for ~25 UI presentation keys (never queried by backend) |

*`dimensions.ui_config` JSONB keys (all optional):*
- Loading: `top_n_count`, `min_search_chars`
- Boolean: `true_label`, `false_label`, `is_three_state`
- Input: `input_suffix`, `decimal_places`, `show_slider`
- Date: `min_date`, `max_date`, `date_presets`
- Nested/Tree: `max_depth`, `default_expand`, `select_children`, `cascade_behavior`
- Display: `placeholder`, `tooltip`, `width`, `is_collapsed`, `display_as`, `search_placeholder`
- Empty: `empty_message`, `hide_when_empty`
- Advanced: `option_sections`, `cross_filter_counts`

*`report_template_filters` table — new columns (3):*
| Column | Purpose |
|--------|---------|
| `filter_type_override` | Same dimension, different filter_type per report (e.g., multi_select on Revenue, single_select on Explorer) |
| `max_selections_override` | Per-report selection cap |
| `ui_config_override` | Per-report UI overrides, deep-merged with dimension's ui_config |

*Design principle:* Columns for anything that affects SQL queries or backend validation. JSONB for UI presentation. This ensures no schema migrations for future filter features.

**M1 verified in production database (2026-01-28):**
- 32/32 tables present in `report_builder` schema *(NOTE: Round 3 + Round 4 changes need re-apply — filter_types table, dimensions columns, report_template_filters columns)*
- All column counts match SQL definitions
- 98 indexes confirmed via `pg_indexes`
- 83 constraints (FKs + CHECKs) confirmed via `pg_constraint`
- All 32 tables empty (correct pre-seeding state)
- Post-M1 step: `prisma db pull` to generate Prisma models (deferred to M3)

**M2 COMPLETE (2026-01-28, Rev 3):**

Created `JSON_Template_Format_M2.md` — the definitive JSON template specification for `report_templates.layout` JSONB column.

*Initial spec:*
- Standardizes from `Development_Phase_Plan.md` Section 7 templates
- Key naming: `filterBar` → `slicerPanel`, `toggles` → `slicers`, PascalCase → lowercase visual types, `visualId` on all visuals
- Two chart modes: series (line/area/bar/combo) vs dimension+metric (stacked/pie/donut)
- Two table modes: aggregated (dimension+metrics) vs detail (isDetailTable+columns)
- 4 stock report examples, validation checklist, migration guide

*Gap analysis (Rev 2) — cross-referenced PBI reports (53 reports, 11,828 visuals), DB filter system (157 filters), beastinsights-old frontend:*
1. Comparison date range — `showComparison`, `comparisonPresets` + Section 3.7
2. Filter operators — `in`/`not_in` format, "all" = omit filter
3. Date range presets — added `yesterday`, `last_calendar_week`, `last_4_weeks`, `last_month`
4. Parameter grouping — `group` field for popover grouping
5. Column visibility — `defaultVisibleColumns`, `columnPresets` on tables
6. Nested filter cascading — note on `parent_dimension_id` behavior
7. Cross-visual filtering — `interactionMode` on BaseVisual + Section 6.7
8. Chart comparison — `comparison` fields on SeriesConfig + Section 6.8

*Architectural fixes (Rev 3):*
- **Group-by migration:** Report-level → table-level. `groupByOptions` and `defaultGroupBy` now on `AggregatedTableVisual`, not `SlicerPanel`. `defaultGroupBy` is `string[]` (multiple dimensions can be selected simultaneously).
- **Card variants (7 types):** `cardVariant` field on CardVisual: `metric` (default big number), `breakdown` (number + list), `donut` (number in donut center), `sparkline` (number + trend line), `summary` (template text), `chart` (number + embedded bar/donut), `insight` (AI-generated text). Each variant has variant-specific fields.
- **Visual size constraints (Section 4.8):** Sizes determined by section type, NOT by individual visual config. Frontend enforces rules: card_row = auto-fit, full_width = 100%, grid = equal columns, split = 50% each, tabs = 100%. Responsive breakpoints documented. No width/height fields in JSON.
- **DB-driven filter types:** New `filter_types` table (33rd table) makes filter types extensible without code changes. 8 core types seeded.
- **Complete Reference Template (Section 7.5):** ~500-line JSON exercising ALL features — 7 visual types, 6 card variants, 7 chart types, 5 section types, 6 slicers, 5 filter displayAs modes, 8 parameters (4 grouped), 5 status filters, 4 cross-source visuals, conditional visibility, comparison overlays, column presets, metric config with conditional colors, 14-column detail table.

---

## 16. Current Progress

| Milestone | Status | Notes |
|-----------|--------|-------|
| **M1: Database Schema** | ✅ COMPLETE | 33 tables in `report_builder` schema (32 original + `filter_types` for DB-driven filter registry) |
| **M2: JSON Template Format** | ✅ COMPLETE | `JSON_Template_Format_M2.md` — Rev 4.1: filter dependencies, drill-through, empty/error states, URL deep linking, Sales Report template |
| **M3: Backend Query Engine** | ✅ COMPLETE | Query Engine V2 — two endpoints (`/batch`, `/table`), array response format, parallel execution, no Prisma dependency |
| **M4: Frontend Report Renderer** | ⏳ NOT STARTED | React components, TanStack Query, Zustand stores |
| **M5: Data Library** | ✅ COMPLETE | 64 active metrics, 27 active dimensions, 6 toggles — all tested and working |
| **M6: Reports & Navigation** | ⏳ NOT STARTED | 11 JSON templates, slicer panel, sidebar |
| **M7: Performance & Launch** | ⏳ NOT STARTED | <500ms target, error boundaries, mobile |

**M3 Deliverables (2026-01-29 → 2026-01-30):**
- `beastinsights-backend/` — Full backend repo with all existing routes + new query engine
- New services: `queryEngineV2.js` (replaces queryService.js), `toggleService.js`
- New controllers: `queryController.js` (V2 with two endpoints)
- New routes:
  - `POST /api/v1/query/batch` — All visuals at once (initial load, filter/toggle changes)
  - `POST /api/v1/query/table` — Single table (sort, page, groupBy changes)
- **Removed Prisma dependency** from query engine — uses raw pg pool directly for all queries
- Array-based response format: `columns` (metadata) + `rows` (2D arrays) — 63% smaller payloads
- Parallel query execution via Promise.all()
- Toggle variants correctly applied to metrics
- Complete API documentation: `COMPLETE_API_DOCUMENTATION.md` (4,000+ lines with curl examples)
- Comprehensive test suite: `tests/comprehensiveM3M5.test.js` (792 lines, 107 tests)

**M3 & M5 Test Results (2026-02-02):**
- **100% pass rate (106/106 tests)**
- All query engine code working correctly
- Toggle variants (standard/organic/net) confirmed working
- Parallel execution: 5 visuals in ~5 seconds
- Multi-dimension groupBy (up to 3 dimensions) working
- Pagination, sorting, filters all working
- All 64 active metrics verified working
- All 27 active dimensions verified working

**M5 Deliverables (2026-01-29):**

Seed scripts created in `sql/seed/`:
| File | Contents | Rows (approx) |
|------|----------|---------------|
| `00_run_all_seeds.sql` | Master script — runs all seeds in order with TRUNCATE | — |
| `01_data_sources.sql` | 10 data sources mapping to materialized views | ~10 |
| `02_toggles.sql` | 6 toggles + ~15 toggle options | ~21 |
| `03_dimensions.sql` | ~30 dimensions across 10 categories (time, campaign, product, gateway, card, transaction, dispute, affiliate, decline, MID) | ~30 |
| `04_metrics.sql` | Core metrics — volume, revenue, dispute, rates, profitability, recovery, LTV | ~70 |
| `04b_metrics_supplemental.sql` | Additional metrics — MID health, cycle counts, growth tracking, alerts, explorer | ~40 |
| `05_metric_variants.sql` | Toggle-based SQL overrides for approval_mode, date_basis, dollar_count, retention_base | ~25 |
| `06_data_source_metrics.sql` | Junction mapping: which metrics work with which data sources | ~300 |
| `07_data_source_dimensions.sql` | Junction mapping: which dimensions work with which data sources | ~200 |
| `08_filter_types.sql` | 8 filter type definitions (multi_select, single_select, date_range, search, boolean, nested, tree, input) | 8 |

**DAX translation approach:**
- Extracted measures from `PBI-beastinsights-prod` TMDL files
- Identified toggle patterns (SWITCH/SELECTEDVALUE) and converted to metric_variants
- Mapped column expressions to materialized view columns
- Preserved Power BI naming for consistency

**M5 Test Results (2026-02-02) — Data Library COMPLETE:**

After comprehensive testing and database fixes via `sql/fix_m5_data_library.sql`:

*Active metrics (64):*
- Volume: `attempts`, `approvals`, `net_approvals`, `approvals_organic`, `declines`
- Revenue: `revenue`, `revenue_initials`, `revenue_rebills`, `aov`
- Disputes: `cb`, `cb_dollar`, `refund`, `refund_dollar`, `alert_count`
- Rates: `approval_rate`, `cb_rate`, `refund_rate`, `decline_rate`
- Retention: `retention_c1`, `retention_c2`, `retention_c3`, `retention_c6`, `retention_c12` (fixed SQL expressions)
- Profitability: `processing_fees`, `cb_fees`, `reserve`, `gross_profit`, `gross_margin` (with default parameters)

*Disabled metrics (45) — require specialized views not yet integrated:*
- Card brand metrics (require bin_lookup JOIN)
- Recovery metrics (require decline_recovery view)
- LTV metrics (require cohort_summary view)
- MID health metrics (require mid_summary view)
- Time-window CB/refund metrics (require cb_refund_alert view)
- Alert detail metrics (require alert_summary view)
- Hourly/today metrics (require hourly_revenue view)

*Active dimensions (27):*
- Time: `date`, `date_week`, `date_month`, `date_year`
- Campaign: `campaign_name`, `campaign_id`, `campaign_type`
- Product: `product_name`, `product_id`, `price_point`
- Gateway: `gateway_alias`, `gateway_id`, `trial_gateway_alias`
- Transaction: `sales_type`, `billing_cycle`, `refund_type`, `alert_type`
- Affiliate: `affid`, `sub_affid`, `c`
- BIN: `bin`

*Disabled dimensions (4) — require specialized views:*
- `hour` (hourly_revenue)
- `decline_group` (decline_recovery)
- `month_year` (mid_summary)
- `health_tag` (mid_summary)

*SQL fix script applied:* `sql/fix_m5_data_library.sql`
- Fixed retention metrics with correct billing_cycle calculations
- Fixed parameter-based metrics with default values (2.9% processing, $25 CB fee, 10% reserve)
- Disabled metrics referencing unavailable columns
- Added missing data_source_metrics mappings

---

*Beast Insights — Session Context*
*Last updated: 2026-02-02*

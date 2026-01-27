# Beast Insights — Development Phase Plan

> Complete technical blueprint for migrating from Power BI to a native JSON-driven reporting platform.

---

## Table of Contents

1. [Tech Stack](#1-tech-stack)
2. [The 4 Phases](#2-the-4-phases)
3. [Phase 1 Milestones](#3-phase-1-milestones)
4. [Database Architecture](#4-database-architecture)
5. [Backend Architecture](#5-backend-architecture)
6. [Frontend Architecture](#6-frontend-architecture)
7. [Report JSON Templates](#7-report-json-templates)
8. [API Endpoints](#8-api-endpoints)
9. [Performance Strategy](#9-performance-strategy)
10. [Integration with Existing System](#10-integration-with-existing-system)

---

## 1. Tech Stack

**Existing stack is solid. No framework changes needed.**

| Layer | Technology | Status |
|-------|-----------|--------|
| Frontend | Next.js 16 + React 19 | Keep |
| UI Components | Radix UI + shadcn/ui + Tailwind CSS | Keep |
| Charts | Recharts (already installed) | Keep |
| Tables | TanStack React Table v8 (already installed) | Keep |
| Backend | Express.js 4.19 | Keep |
| ORM (app layer) | Prisma 5.19 | Keep for CRUD |
| Query Engine | `pg` driver (already installed) | Use for dynamic SQL |
| Database | PostgreSQL | Keep |
| Auth | JWT HS512 + bcrypt | Keep unchanged |
| Existing State | Redux Toolkit + Redux Persist | Keep for auth/client |

**New additions for Phase 1:**

| Addition | Purpose |
|----------|---------|
| `@tanstack/react-query` | Server state management + frontend cache (Layer 1) |
| `zustand` | Lightweight UI state for report filters/toggles/parameters |

**NOT needed in Phase 1:**
- Redis (materialized views are already pre-computed; with proper indexes, queries return in 10-50ms)
- WebSockets (add later for real-time features)
- Socket.io (Phase 4)

**State management coexistence:**
- **Redux** — auth state, client selection, existing Power BI features (unchanged)
- **Zustand** — report filter state, toggle state, parameter values, UI state (new)
- **TanStack Query** — all data fetching for native reports with 5-min stale time (new)

No existing Redux code needs to change. The `clientId` used by the query engine comes from the existing Redux `clientSlice`, bridged via a custom hook.

---

## 2. The 4 Phases

### Phase 1: Core Reporting Engine
> Build the engine that converts JSON templates into interactive reports

**Scope:**
- `report_builder` PostgreSQL schema with full semantic layer
- Query Engine: `POST /api/v1/query` and `/api/v1/query/batch`
- Frontend report renderer (JSON template -> React widgets)
- All widget types: MetricCard, Chart, DataTable, PivotTable, SummaryTable, WaterfallChart
- FilterBar: date range, dimension filters, toggles, parameters, group-by
- Metric variants system (5 toggle types) fully operational
- 11 stock report templates seeded in DB (Custom Report Builder is Phase 2)
- Feature-flagged rollout: Power BI continues, native reports opt-in per client
- TanStack Query frontend caching (Layer 1)
- Target: <500ms page load

**Deliverable:** All 11 universal reports running natively with real data, dynamic filters, toggle support, and sub-second load times.

---

### Phase 2: User Customization & Self-Service
> Let users build their own reports and personalize the platform

**Scope:**
- Custom Report Builder (drag-and-drop from metric/dimension catalog)
- Saved filter presets (named, shareable across reports)
- Saved views (per-user customizations per report)
- Personal dashboards (pin widgets from any report, react-grid-layout)
- Export: CSV, Excel, PDF
- Scheduled report delivery (email/Telegram)
- Role-based page access (admin assigns which reports each role sees)
- Notification system (threshold alerts, report-level triggers)

**Deliverable:** Users create custom reports, save views, share presets, set up scheduled delivery. Admins control page access per role.

---

### Phase 3: Dimension-Level Access Control
> Users only see data for their assigned products, campaigns, gateways, etc.

**Scope:**
- `user_dimension_access` table mapping users to allowed dimension values
- Query engine auto-injects WHERE clauses based on user's dimension permissions
- Admin UI for assigning dimension access per user
- Cascade rules (access to Campaign A -> see all products under it)
- "View As" mode for admins to preview restricted user experience
- Audit logging for access control changes
- Client-level defaults (new users inherit default dimension access)

**Deliverable:** Admin creates a user restricted to Gateway 7 and Campaign "Summer Sale". Every query, every report, every widget is automatically filtered. Zero data leakage.

---

### Phase 4: Industry Modules & Expansion
> Opt-in specialized report packs per industry vertical

**Scope:**
- Module enablement system (per-client configuration in DB)
- **High-Risk Module** (6 reports): MID Health & Compliance, BIN Routing Optimizer, Decline Recovery Intelligence, Alert Service Management, Gateway Performance, Affiliate Quality
- **Telehealth Module** (5 reports): Patient Membership, Provider Performance, Visit-to-Payment, Insurance vs Self-Pay, Compliance
- **SaaS Module** (5 reports): ARR Tracking, Trial-to-Paid, Seat/Usage Analytics, Plan Migration, Net Dollar Retention
- **eCommerce Module** (5 reports): Order Frequency, Returns, Seasonal Demand, Basket Analysis, Delivery & Fulfillment
- Industry-specific metrics/dimensions added to semantic layer
- Navigation adapts based on enabled modules
- New materialized views for module-specific data

**Deliverable:** Beast Insights becomes multi-industry. Each client gets 12 universal reports + their industry module reports.

---

## 3. Phase 1 Milestones

### Milestone 1: Database Schema & Semantic Layer

**Goal:** Create the `report_builder` schema and seed the full metric/dimension/toggle catalog.

**Tasks:**
1. Create `report_builder` schema with all tables (full SQL in Section 4)
2. Add Prisma models for the new schema (uses existing `multiSchema` preview feature)
3. Write seed script for data sources (10 materialized view types)
4. Write seed script for metrics (~100 metrics)
5. Write seed script for dimensions (~30 dimensions)
6. Write seed script for toggles, toggle options, and metric variants
7. Write seed script for data_source_metrics and data_source_dimensions junction tables

**Files to create:**
```
beast-insights-backend/
  prisma/
    migrations/YYYYMMDD_report_builder_schema/migration.sql
    seed-report-builder.js
  prisma/schema.prisma (modify - add new models)
```

**Testable:** Query `report_builder.metrics` returns full catalog. Variant lookup for `(approvals, approval_mode, organic)` returns `SUM(approvals_organic)`.

**Dependencies:** None - this is the foundation.

---

### Milestone 2: Query Engine & Batch API

**Goal:** Build the generic query engine that translates metric/dimension/filter/toggle requests into parameterized SQL.

**Tasks:**
1. Create `db/queryPool.js` - dedicated `pg` connection pool (separate from Prisma)
2. Build query engine modules:
   - `sourceResolver.js` - resolves `"order_summary"` -> `"reporting.order_summary_{clientId}"`
   - `metricResolver.js` - resolves metric keys to SQL with variant support + derived metric cascading
   - `dimensionResolver.js` - resolves dimension keys to columns + required JOINs
   - `filterBuilder.js` - converts filter objects into parameterized WHERE clauses
   - `sqlAssembler.js` - combines all parts into final SQL
   - `responseFormatter.js` - formats raw rows with proper types
   - `batchExecutor.js` - runs multiple queries via `Promise.all()`
3. Create routes + controller for query endpoints
4. Register in `routes/handleRequests.js`
5. Security: parameterized queries, client isolation from JWT, column whitelisting, 30s timeout

**Files to create:**
```
beast-insights-backend/
  db/queryPool.js
  services/queryEngine/
    index.js
    sourceResolver.js
    metricResolver.js
    dimensionResolver.js
    filterBuilder.js
    sqlAssembler.js
    responseFormatter.js
    batchExecutor.js
  routes/queryRoutes.js
  controller/queryController.js
  routes/handleRequests.js (modify - add 1 line)
```

**Testable:** POST `/api/v1/query` with metrics/dimensions/filters returns correct data. Batch 5 queries in one call. Toggle test returns different values.

**Dependencies:** Milestone 1

---

### Milestone 3: Frontend Report Rendering Pipeline

**Goal:** All React components needed to render any report from a JSON template.

**Tasks:**
1. Install `@tanstack/react-query` and `zustand`
2. Add `QueryClientProvider` in `app/providers.jsx`
3. Create Zustand stores for filter and UI state
4. Build `useReportData` hook (template + filters -> batch query -> TanStack Query)
5. Build all widget components
6. Build `ReportRenderer` component
7. Build section layout components

**Files to create:**
```
beastinsights/
  stores/
    reportFilterStore.js
    reportUIStore.js
  hooks/
    useReportData.js
    useReportTemplate.js
  components/native-reports/
    ReportRenderer.jsx
    ReportPage.jsx
    layouts/
      Section.jsx
      CardRow.jsx
      Grid.jsx
      FullWidth.jsx
      TabsSection.jsx
    widgets/
      MetricCard.jsx
      Chart.jsx
      DataTable.jsx
      PivotTable.jsx
      SummaryTable.jsx
      WaterfallChart.jsx
      ParameterInput.jsx
      HealthIndicator.jsx
    FilterBar.jsx
    ToggleControl.jsx
    DateRangePicker.jsx
    DimensionFilter.jsx
    GroupBySelector.jsx
  lib/
    queryClient.js
    api.js
  app/providers.jsx (modify)
  package.json (modify)
```

**Testable:** Render a hardcoded template with mock data. All widget types display correctly. Changing Zustand state triggers TanStack Query refetch.

**Dependencies:** Milestone 2 (for real data; can start with mock data while M2 is in progress)

---

### Milestone 4: Filter Bar & Filter Options API

**Goal:** Complete filter system - date range, dimension filters, toggles, parameters, group-by.

**Tasks:**
1. Build filter options API on backend (`GET /api/v1/filters/options/:dimensionKey`)
2. Build the `FilterBar` component with:
   - Date range picker with presets (today, last 7d, last 30d, MTD, QTD, YTD, custom)
   - Dimension filter dropdowns (populated via API)
   - Toggle controls (pills for metric_variant, dropdown for dimension_switch)
   - Parameter inputs (number inputs with $/% for profitability)
   - GroupBy selector
   - Compare toggle (vs previous period)
   - Export button (skeleton for Phase 2)
3. Wire filter state changes to Zustand store -> TanStack Query refetch
4. Build comparison period logic (auto-calculate previous period from current date range)

**Files to create:**
```
beast-insights-backend/
  routes/filterOptionRoutes.js
  controller/filterOptionController.js
  services/filterOptionService.js

beastinsights/
  components/native-reports/
    FilterBar.jsx (complete implementation)
    ToggleControl.jsx
    DimensionFilter.jsx
    DateRangePicker.jsx
    GroupBySelector.jsx
    CompareToggle.jsx
```

**Testable:** Filter dropdowns populate with real dimension values. Changing date range updates all widgets. Toggling approval_mode changes numbers. Parameter inputs affect profitability calculations.

**Dependencies:** Milestone 2 (backend), Milestone 3 (frontend components)

---

### Milestone 5: Report Templates & Navigation Integration

**Goal:** All 11 stock report JSON templates seeded in DB, template API, and sidebar routing.

**Tasks:**
1. Build template API (`GET /api/v1/reports/templates` and `GET /api/v1/reports/templates/:key`)
2. Create all 11 JSON templates (full JSON in Section 7)
3. Write seed script for templates
4. Create Next.js dynamic route `app/(user-type)/(user)/reports/native/[reportSlug]/page.js`
5. Add `report_mode` to client configuration (`powerbi` | `native` | `both`)
6. Integrate with sidebar: map existing nav items to native report slugs
7. Feature flag: when `report_mode = 'both'`, show toggle in toolbar

**Files to create:**
```
beast-insights-backend/
  routes/templateRoutes.js
  controller/templateController.js
  services/templateService.js
  db/templateMethods.js
  prisma/seed-templates.js

beastinsights/
  app/(user-type)/(user)/reports/native/[reportSlug]/page.js
  components/sidebar/AppSidebar.jsx (modify)
```

**Testable:** Navigate to `/reports/native/revenue-analytics` and see the full report with real data. All 11 reports render. Feature flag controls which mode each client uses.

**Dependencies:** Milestones 1-4

---

### Milestone 6: Specialized Report Handling

**Goal:** Handle reports that don't fit the standard pattern.

**Tasks:**
1. **MID Performance** - MonthSelector component; query engine recognizes `dateFilterType: "month_selector"`
2. **LTV/Retention** - PivotTable transforms flat cohort rows into matrix with heatmap coloring
3. **Command Center** - auto-refresh via TanStack Query `refetchInterval` (5 min)
4. **Transaction Explorer** - server-side pagination, sorting, search; column toggle
5. **Waterfall chart** - Financial Performance: Revenue -> minus costs -> Gross Profit
6. **Compare to previous period** - MetricCards show "+12% vs last period"; query engine runs comparison query
7. **Cross-source reports** - different widgets query different materialized views (handled by batch)
8. **"Needs Attention" section** - Command Center anomaly detection (simple threshold-based for Phase 1)

**Files to create/modify:**
```
beastinsights/
  components/native-reports/widgets/
    MonthSelector.jsx
    PivotTable.jsx (enhance for heatmap)
    WaterfallChart.jsx (custom Recharts stacked bar)
  hooks/
    useComparisonData.js
    useServerPagination.js
```

**Testable:** MID report with month selector. LTV cohort matrix with heatmap. Auto-refresh on Command Center. Transaction Explorer paginates server-side. Waterfall chart on Financial Performance.

**Dependencies:** Milestone 5

---

### Milestone 7: Performance, Testing & Migration Readiness

**Goal:** Hit <500ms target, validate data accuracy, prepare for client rollout.

**Tasks:**
1. **Performance profiling** - instrument query times, add missing indexes on materialized views
2. **Frontend optimization** - React.lazy for widgets, React.memo, react-window for large tables
3. **Data validation** - compare metric values between Power BI and native for same date ranges
4. **Error resilience** - per-widget error boundaries, query timeout handling
5. **Feature flag UI** - superadmin panel to switch clients between powerbi/native/both
6. **Monitoring** - log query execution times, error rates per data source
7. **Mobile responsiveness** - widgets stack vertically on small viewports

**Testable:** All 11 reports load in <500ms. Same metrics match Power BI values. Mobile layout works. Error boundaries catch widget failures without crashing the page.

**Dependencies:** Milestones 1-6

---

### Milestone Dependency Graph

```
M1 (Schema & Seed)
 |
 v
M2 (Query Engine & API)
 |
 v
M3 (Frontend Rendering) --- can start with mock data while M2 builds
 |
 v
M4 (Filter Bar & Filter API)
 |
 v
M5 (Templates & Navigation)
 |
 v
M6 (Specialized Reports)
 |
 v
M7 (Performance & Migration)
```

Backend team: M1 -> M2 in sequence. Frontend team: start M3 with mock data while backend finishes M2. They converge at M4.

---

## 4. Database Architecture

### 4.1 Schema Overview

```
PostgreSQL Database: postgres
├── beast_insights_v2    (existing - app layer, users, clients, auth)
├── data                 (existing - raw orders per client)
├── public               (existing - shared dimension tables)
├── reporting            (existing - materialized views per client)
├── beastinsights_alert  (existing - alert data)
├── routing              (existing - routing data)
└── report_builder       (NEW - semantic layer, templates, user customizations)
```

### 4.2 New Schema: report_builder

```sql
-- ================================================================
-- SCHEMA CREATION
-- ================================================================
CREATE SCHEMA IF NOT EXISTS report_builder;

-- ================================================================
-- DATA SOURCES
-- ================================================================
CREATE TABLE report_builder.data_sources (
    id SERIAL PRIMARY KEY,
    source_key VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    description TEXT,
    source_type VARCHAR(50) NOT NULL DEFAULT 'materialized_view',
    schema_name VARCHAR(100) NOT NULL DEFAULT 'reporting',
    table_pattern VARCHAR(200) NOT NULL,
    date_column VARCHAR(100) DEFAULT 'date',
    client_id_column VARCHAR(100) DEFAULT 'client_id',
    refresh_frequency VARCHAR(50),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ================================================================
-- METRICS CATALOG
-- ================================================================
CREATE TABLE report_builder.metrics (
    id SERIAL PRIMARY KEY,
    metric_key VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    description TEXT,
    category VARCHAR(100) NOT NULL,
    sub_category VARCHAR(100),
    data_type VARCHAR(50) NOT NULL,
    format_pattern VARCHAR(100),
    decimal_places INTEGER DEFAULT 0,
    sql_expression TEXT NOT NULL,
    is_derived BOOLEAN DEFAULT false,
    derived_formula TEXT,
    aggregation_type VARCHAR(50),
    condition_sql TEXT,
    supports_comparison BOOLEAN DEFAULT true,
    supports_trend BOOLEAN DEFAULT true,
    default_sort_direction VARCHAR(4) DEFAULT 'DESC',
    is_active BOOLEAN DEFAULT true,
    display_order INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ================================================================
-- DIMENSIONS CATALOG
-- ================================================================
CREATE TABLE report_builder.dimensions (
    id SERIAL PRIMARY KEY,
    dimension_key VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    data_type VARCHAR(50) DEFAULT 'string',
    column_expression TEXT NOT NULL,
    source_alias VARCHAR(10) DEFAULT 'os',
    join_table VARCHAR(200),
    join_alias VARCHAR(50),
    join_condition TEXT,
    is_date_dimension BOOLEAN DEFAULT false,
    date_granularity VARCHAR(50),
    date_expression TEXT,
    is_filterable BOOLEAN DEFAULT true,
    filter_query TEXT,
    is_active BOOLEAN DEFAULT true,
    display_order INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ================================================================
-- DATA SOURCE <-> METRIC JUNCTION
-- ================================================================
CREATE TABLE report_builder.data_source_metrics (
    id SERIAL PRIMARY KEY,
    data_source_id INTEGER NOT NULL REFERENCES report_builder.data_sources(id),
    metric_id INTEGER NOT NULL REFERENCES report_builder.metrics(id),
    is_active BOOLEAN DEFAULT true,
    UNIQUE(data_source_id, metric_id)
);

-- ================================================================
-- DATA SOURCE <-> DIMENSION JUNCTION
-- ================================================================
CREATE TABLE report_builder.data_source_dimensions (
    id SERIAL PRIMARY KEY,
    data_source_id INTEGER NOT NULL REFERENCES report_builder.data_sources(id),
    dimension_id INTEGER NOT NULL REFERENCES report_builder.dimensions(id),
    is_active BOOLEAN DEFAULT true,
    UNIQUE(data_source_id, dimension_id)
);

-- ================================================================
-- METRIC TOGGLES (context switches)
-- ================================================================
CREATE TABLE report_builder.metric_toggles (
    id SERIAL PRIMARY KEY,
    toggle_key VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    description TEXT,
    toggle_type VARCHAR(50) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

-- ================================================================
-- METRIC TOGGLE OPTIONS
-- ================================================================
CREATE TABLE report_builder.metric_toggle_options (
    id SERIAL PRIMARY KEY,
    toggle_id INTEGER NOT NULL REFERENCES report_builder.metric_toggles(id),
    option_key VARCHAR(100) NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    description TEXT,
    display_order INTEGER DEFAULT 0,
    is_default BOOLEAN DEFAULT false,
    UNIQUE(toggle_id, option_key)
);

-- ================================================================
-- METRIC VARIANTS (SQL overrides per toggle option)
-- ================================================================
CREATE TABLE report_builder.metric_variants (
    id SERIAL PRIMARY KEY,
    metric_id INTEGER NOT NULL REFERENCES report_builder.metrics(id),
    toggle_id INTEGER NOT NULL REFERENCES report_builder.metric_toggles(id),
    option_key VARCHAR(100) NOT NULL,
    sql_expression TEXT NOT NULL,
    display_name_override VARCHAR(200),
    UNIQUE(metric_id, toggle_id, option_key)
);

-- ================================================================
-- REPORT TEMPLATES
-- ================================================================
CREATE TABLE report_builder.report_templates (
    id SERIAL PRIMARY KEY,
    template_key VARCHAR(100) UNIQUE,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    icon VARCHAR(100),
    template_type VARCHAR(50) NOT NULL DEFAULT 'stock',
    layout JSONB NOT NULL,
    default_filters JSONB,
    default_date_range VARCHAR(50) DEFAULT 'last_7_days',
    required_data_sources TEXT[],
    nav_section VARCHAR(100),
    nav_order INTEGER DEFAULT 0,
    created_by_user_id INTEGER,
    client_id INTEGER,
    is_public BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    display_order INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ================================================================
-- SAVED VIEWS (Phase 2 - create table now, use later)
-- ================================================================
CREATE TABLE report_builder.saved_views (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    report_template_id INTEGER REFERENCES report_builder.report_templates(id),
    user_id INTEGER NOT NULL,
    client_id INTEGER NOT NULL,
    custom_filters JSONB,
    custom_metrics JSONB,
    custom_dimensions JSONB,
    custom_sort JSONB,
    custom_layout JSONB,
    custom_date_range VARCHAR(50),
    is_pinned BOOLEAN DEFAULT false,
    is_default BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ================================================================
-- FILTER PRESETS (Phase 2 - create table now)
-- ================================================================
CREATE TABLE report_builder.filter_presets (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    filters JSONB NOT NULL,
    user_id INTEGER,
    client_id INTEGER,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ================================================================
-- DASHBOARDS (Phase 2 - create table now)
-- ================================================================
CREATE TABLE report_builder.dashboards (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    user_id INTEGER NOT NULL,
    client_id INTEGER NOT NULL,
    layout JSONB NOT NULL,
    is_default BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ================================================================
-- DASHBOARD WIDGETS (Phase 2 - create table now)
-- ================================================================
CREATE TABLE report_builder.dashboard_widgets (
    id SERIAL PRIMARY KEY,
    dashboard_id INTEGER REFERENCES report_builder.dashboards(id),
    widget_type VARCHAR(50) NOT NULL,
    title VARCHAR(200),
    data_source_key VARCHAR(100),
    metrics JSONB,
    dimensions JSONB,
    filters JSONB,
    date_range VARCHAR(50),
    chart_type VARCHAR(50),
    widget_config JSONB,
    grid_x INTEGER,
    grid_y INTEGER,
    grid_w INTEGER,
    grid_h INTEGER,
    source_report_id INTEGER,
    source_widget_key VARCHAR(100),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ================================================================
-- SCHEDULED REPORTS (Phase 2 - create table now)
-- ================================================================
CREATE TABLE report_builder.scheduled_reports (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    report_template_id INTEGER,
    saved_view_id INTEGER,
    user_id INTEGER NOT NULL,
    client_id INTEGER NOT NULL,
    frequency VARCHAR(50) NOT NULL,
    day_of_week INTEGER,
    day_of_month INTEGER,
    time_of_day TIME,
    timezone VARCHAR(100),
    delivery_method VARCHAR(50),
    delivery_config JSONB,
    export_format VARCHAR(50),
    filters JSONB,
    date_range VARCHAR(50),
    is_active BOOLEAN DEFAULT true,
    last_sent_at TIMESTAMP,
    next_send_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ================================================================
-- USER DIMENSION ACCESS (Phase 3 - create table now)
-- ================================================================
CREATE TABLE report_builder.user_dimension_access (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    client_id INTEGER NOT NULL,
    dimension_key VARCHAR(100) NOT NULL,
    allowed_values JSONB NOT NULL,
    created_by INTEGER,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, client_id, dimension_key)
);

-- ================================================================
-- CLIENT MODULE ENABLEMENT (Phase 4 - create table now)
-- ================================================================
CREATE TABLE report_builder.client_modules (
    id SERIAL PRIMARY KEY,
    client_id INTEGER NOT NULL,
    module_key VARCHAR(100) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    enabled_at TIMESTAMP DEFAULT NOW(),
    enabled_by INTEGER,
    UNIQUE(client_id, module_key)
);

-- ================================================================
-- INDEXES
-- ================================================================
CREATE INDEX idx_metrics_category ON report_builder.metrics(category);
CREATE INDEX idx_metrics_active ON report_builder.metrics(is_active);
CREATE INDEX idx_dimensions_category ON report_builder.dimensions(category);
CREATE INDEX idx_dsm_data_source ON report_builder.data_source_metrics(data_source_id);
CREATE INDEX idx_dsm_metric ON report_builder.data_source_metrics(metric_id);
CREATE INDEX idx_dsd_data_source ON report_builder.data_source_dimensions(data_source_id);
CREATE INDEX idx_dsd_dimension ON report_builder.data_source_dimensions(dimension_id);
CREATE INDEX idx_variants_metric ON report_builder.metric_variants(metric_id);
CREATE INDEX idx_variants_toggle ON report_builder.metric_variants(toggle_id);
CREATE INDEX idx_templates_key ON report_builder.report_templates(template_key);
CREATE INDEX idx_templates_type ON report_builder.report_templates(template_type);
CREATE INDEX idx_saved_views_user ON report_builder.saved_views(user_id, client_id);
CREATE INDEX idx_filter_presets_user ON report_builder.filter_presets(user_id, client_id);
CREATE INDEX idx_dashboards_user ON report_builder.dashboards(user_id, client_id);
CREATE INDEX idx_uda_user ON report_builder.user_dimension_access(user_id, client_id);
CREATE INDEX idx_client_modules ON report_builder.client_modules(client_id);
```

### 4.3 Seed Data: Data Sources

```sql
INSERT INTO report_builder.data_sources (source_key, display_name, schema_name, table_pattern, date_column, refresh_frequency) VALUES
('order_summary', 'Order Summary', 'reporting', 'order_summary_{client_id}', 'date', 'hourly'),
('mid_summary', 'MID Summary', 'reporting', 'mid_summary_{client_id}', 'month_year', 'daily'),
('cb_refund_alert', 'CB & Refund Alert', 'reporting', 'cb_refund_alert_{client_id}', 'date', 'hourly'),
('cohort_summary', 'Cohort Summary', 'reporting', 'cohort_summary_{client_id}', 'date', 'daily'),
('decline_recovery', 'Decline Recovery', 'reporting', 'decline_recovery_{client_id}', 'date', 'hourly'),
('hourly_revenue', 'Hourly Revenue', 'reporting', 'hourly_revenue_{client_id}', 'hour', 'hourly'),
('alert_summary', 'Alert Summary', 'reporting', 'alert_summary_{client_id}', 'date', 'hourly'),
('alert_details', 'Alert Details', 'reporting', 'alert_details_{client_id}', 'alert_date', 'hourly'),
('order_details', 'Order Details', 'reporting', 'order_details_{client_id}', 'date', 'hourly'),
('products', 'Products', 'reporting', 'products_{client_id}', NULL, 'daily');
```

### 4.4 Seed Data: Metric Toggles & Options

```sql
-- Toggle: Approval Mode
INSERT INTO report_builder.metric_toggles (toggle_key, display_name, toggle_type) VALUES
('approval_mode', 'Approval Type', 'metric_variant'),
('date_basis', 'Date Type', 'metric_variant'),
('time_granularity', 'Time Period', 'dimension_switch'),
('profitability_view', 'View', 'layout_switch'),
('retention_base', 'Base', 'metric_variant'),
('display_mode', 'Show', 'visibility');

-- Approval Mode Options
INSERT INTO report_builder.metric_toggle_options (toggle_id, option_key, display_name, is_default, display_order) VALUES
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'approval_mode'), 'standard', 'All Approvals', true, 0),
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'approval_mode'), 'organic', 'First Attempt', false, 1),
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'approval_mode'), 'net', 'After Retries', false, 2);

-- Date Basis Options
INSERT INTO report_builder.metric_toggle_options (toggle_id, option_key, display_name, is_default, display_order) VALUES
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'date_basis'), 'order_date', 'Order Date', true, 0),
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'date_basis'), 'cb_date', 'CB/Refund Date', false, 1);

-- Time Granularity Options
INSERT INTO report_builder.metric_toggle_options (toggle_id, option_key, display_name, is_default, display_order) VALUES
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'time_granularity'), 'day', 'Daily', true, 0),
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'time_granularity'), 'week', 'Weekly', false, 1),
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'time_granularity'), 'month', 'Monthly', false, 2);

-- Profitability View Options
INSERT INTO report_builder.metric_toggle_options (toggle_id, option_key, display_name, is_default, display_order) VALUES
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'profitability_view'), 'cashflow', 'Cashflow', true, 0),
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'profitability_view'), 'unit_economics', 'Unit Economics', false, 1);

-- Retention Base Options
INSERT INTO report_builder.metric_toggle_options (toggle_id, option_key, display_name, is_default, display_order) VALUES
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'retention_base'), 'initial_order', 'Initial Order', true, 0),
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'retention_base'), 'approved', 'Approved', false, 1);

-- Display Mode Options
INSERT INTO report_builder.metric_toggle_options (toggle_id, option_key, display_name, is_default, display_order) VALUES
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'display_mode'), 'both', 'Count & Dollar', true, 0),
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'display_mode'), 'count', 'Count Only', false, 1),
((SELECT id FROM report_builder.metric_toggles WHERE toggle_key = 'display_mode'), 'dollar', 'Dollar Only', false, 2);
```

### 4.5 Seed Data: Metric Variants

```sql
-- Approval Mode Variants
INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'standard', 'SUM(approvals)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'approvals' AND t.toggle_key = 'approval_mode';

INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'organic', 'SUM(approvals_organic)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'approvals' AND t.toggle_key = 'approval_mode';

INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'net', 'SUM(net_approvals)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'approvals' AND t.toggle_key = 'approval_mode';

-- Date Basis Variants (CB)
INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'order_date', 'SUM(cb)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'cb_count' AND t.toggle_key = 'date_basis';

INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'cb_date', 'SUM(cb_cb_date)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'cb_count' AND t.toggle_key = 'date_basis';

INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'order_date', 'SUM(cb_dollar)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'cb_dollar' AND t.toggle_key = 'date_basis';

INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'cb_date', 'SUM(cb_cb_date_dollar)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'cb_dollar' AND t.toggle_key = 'date_basis';

-- Date Basis Variants (Refund)
INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'order_date', 'SUM(refund)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'refund_count' AND t.toggle_key = 'date_basis';

INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'cb_date', 'SUM(refund_refund_date)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'refund_count' AND t.toggle_key = 'date_basis';

INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'order_date', 'SUM(refund_dollar)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'refund_dollar' AND t.toggle_key = 'date_basis';

INSERT INTO report_builder.metric_variants (metric_id, toggle_id, option_key, sql_expression)
SELECT m.id, t.id, 'cb_date', 'SUM(refund_refund_date_dollar)'
FROM report_builder.metrics m, report_builder.metric_toggles t
WHERE m.metric_key = 'refund_dollar' AND t.toggle_key = 'date_basis';
```

> **Note:** Full metric and dimension seed data (80+ metrics, 30+ dimensions) is documented in `Report_Builder_Plan.md` sections 5 and 6. The seed script will reference those definitions.

---

## 5. Backend Architecture

### 5.1 New Folder Structure

```
beast-insights-backend/
  app.js                              (existing - no changes)
  package.json                        (modify - no new deps for Phase 1 backend)

  db/
    config.js                         (existing - Prisma client)
    queryPool.js                      (NEW - pg connection pool for query engine)
    templateMethods.js                (NEW - CRUD for report_templates via Prisma)
    filterOptionMethods.js            (NEW - distinct dimension values)

  services/
    queryEngine/
      index.js                        (NEW - main orchestrator)
      sourceResolver.js               (NEW - source_key -> table name)
      metricResolver.js               (NEW - metric keys -> SQL expressions)
      dimensionResolver.js            (NEW - dimension keys -> columns + JOINs)
      filterBuilder.js                (NEW - filter objects -> WHERE clauses)
      sqlAssembler.js                 (NEW - assembles final SQL)
      responseFormatter.js            (NEW - formats raw rows)
      batchExecutor.js                (NEW - parallel query execution)
      catalogCache.js                 (NEW - in-memory cache of metric/dimension catalogs)
    templateService.js                (NEW - template business logic)
    filterOptionService.js            (NEW - filter option business logic)
    middlewares/
      auth.js                         (existing - JWT verification, NO CHANGES)
      apiAuth.js                      (existing - NO CHANGES)
      apiRateLimit.js                 (existing - NO CHANGES)
      respond.js                      (existing - NO CHANGES)

  controller/
    queryController.js                (NEW - handles query/batch requests)
    templateController.js             (NEW - handles template CRUD)
    filterOptionController.js         (NEW - handles filter option requests)

  routes/
    handleRequests.js                 (existing - ADD new route registrations)
    queryRoutes.js                    (NEW - POST /api/v1/query, /api/v1/query/batch)
    templateRoutes.js                 (NEW - GET /api/v1/reports/templates)
    filterOptionRoutes.js             (NEW - GET /api/v1/filters/options/:key)

  prisma/
    schema.prisma                     (existing - ADD report_builder models)
    migrations/
      YYYYMMDD_report_builder/
        migration.sql                 (NEW - the full schema from Section 4)
    seed-report-builder.js            (NEW - seeds data sources, metrics, dims, toggles)
    seed-templates.js                 (NEW - seeds all 11 report templates)
```

### 5.2 Query Engine Design

```
Request Flow:

  POST /api/v1/query
    |
    v
  queryController.executeQuery(req, res)
    |-- Extract clientId from req.decodedToken
    |-- Validate request body
    |
    v
  queryEngine.execute({ clientId, source, metrics, dimensions, filters, toggles, parameters, sort, pagination })
    |
    |-- 1. sourceResolver.resolve(source, clientId)
    |       -> "reporting.order_summary_10043"
    |
    |-- 2. metricResolver.resolve(metrics, toggles, parameters)
    |       -> For each metric:
    |          a. Check metric_variants for active toggle overrides
    |          b. If derived, expand formula recursively
    |          c. If parameterized, substitute parameter values
    |       -> Returns array of { key, sql, alias, dataType, format }
    |
    |-- 3. dimensionResolver.resolve(dimensions)
    |       -> Returns { columns[], joins[], groupBy[] }
    |
    |-- 4. filterBuilder.build(filters, clientId)
    |       -> Returns { whereClauses[], params[] }
    |       -> ALWAYS includes: client_id = $N
    |       -> Date range: date >= $N AND date <= $N
    |       -> Dimension filters: column IN ($N, $N, ...)
    |
    |-- 5. sqlAssembler.assemble({ select, from, joins, where, groupBy, orderBy, limit, offset })
    |       -> Returns { sql: "SELECT ...", params: [...] }
    |
    |-- 6. queryPool.query(sql, params)
    |       -> Returns raw rows
    |
    |-- 7. responseFormatter.format(rows, metricDefs)
    |       -> Formats numbers, currencies, percentages
    |       -> Adds totals row if requested
    |
    v
  Return { data: { rows, totals }, metadata: { totalRows, queryTimeMs } }
```

### 5.3 Batch Query Flow

```
  POST /api/v1/query/batch
    |
    v
  queryController.executeBatch(req, res)
    |-- Extract clientId from req.decodedToken
    |-- Validate array of queries
    |
    v
  batchExecutor.execute(queries, clientId)
    |
    |-- For each query in parallel (Promise.all):
    |     queryEngine.execute(query)
    |
    |-- Results keyed by widgetId:
    |     { "summary_cards": { rows, totals }, "trend_chart": { rows }, ... }
    |
    v
  Return { results: { [widgetId]: { data, metadata } }, totalTimeMs }
```

### 5.4 Catalog Cache

The metric/dimension/toggle catalogs rarely change. Load them into memory on startup and refresh every 5 minutes (or on demand).

```javascript
// services/queryEngine/catalogCache.js
class CatalogCache {
  constructor() {
    this.metrics = new Map();      // metric_key -> { id, sql_expression, data_type, ... }
    this.dimensions = new Map();   // dimension_key -> { column_expression, join_table, ... }
    this.dataSources = new Map();  // source_key -> { schema_name, table_pattern, ... }
    this.variants = new Map();     // "metric_key:toggle_key:option_key" -> sql_expression
    this.lastRefresh = null;
  }

  async refresh() {
    // Load from report_builder tables via Prisma
    // Called on startup and every 5 minutes
  }

  getMetric(key) { return this.metrics.get(key); }
  getDimension(key) { return this.dimensions.get(key); }
  getDataSource(key) { return this.dataSources.get(key); }
  getVariant(metricKey, toggleKey, optionKey) {
    return this.variants.get(`${metricKey}:${toggleKey}:${optionKey}`);
  }
}
```

### 5.5 Security Model

| Concern | Implementation |
|---------|---------------|
| SQL Injection | All values parameterized ($1, $2, ...). Zero string concatenation. |
| Tenant Isolation | `clientId` from JWT token (never from request body). Every query includes `WHERE client_id = $N`. |
| Column Whitelisting | Only columns in `report_builder.metrics` and `report_builder.dimensions` can be queried. |
| Query Timeout | 30 second max execution time per query. |
| Auth | Reuse existing `auth.js` middleware. All new routes protected. |
| Rate Limiting | Reuse existing `apiRateLimit.js` if needed. |

### 5.6 handleRequests.js Changes

```javascript
// ADD these lines to routes/handleRequests.js
const { queryRoutes } = require("./queryRoutes");
const { templateRoutes } = require("./templateRoutes");
const { filterOptionRoutes } = require("./filterOptionRoutes");

function handleRequests(app) {
  // ... existing routes ...

  // NEW: Report Builder API
  app.use("/api/v1/query", queryRoutes);
  app.use("/api/v1/reports", templateRoutes);
  app.use("/api/v1/filters", filterOptionRoutes);
}
```

---

## 6. Frontend Architecture

### 6.1 New Folder Structure

```
beastinsights/
  app/
    providers.jsx                     (modify - add QueryClientProvider)
    (user-type)/
      (user)/
        reports/
          native/
            [reportSlug]/
              page.js                 (NEW - dynamic native report page)
              loading.js              (NEW - skeleton loader)

  lib/
    queryClient.js                    (NEW - TanStack Query client config)
    reportApi.js                      (NEW - API functions for reports)

  stores/
    reportFilterStore.js              (NEW - Zustand: date range, filters, toggles)
    reportUIStore.js                  (NEW - Zustand: active tab, collapsed sections)

  hooks/
    useReportTemplate.js              (NEW - fetches template JSON)
    useReportData.js                  (NEW - builds batch query from template + filters)
    useFilterOptions.js               (NEW - fetches dimension filter options)
    useComparisonData.js              (NEW - previous period comparison)
    useServerPagination.js            (NEW - for Transaction Explorer)

  components/
    native-reports/
      ReportPage.jsx                  (NEW - wrapper: FilterBar + ReportRenderer)
      ReportRenderer.jsx              (NEW - reads JSON, renders sections)
      FilterBar.jsx                   (NEW - date range, filters, toggles)
      ToggleControl.jsx               (NEW - pill/dropdown toggle UI)
      DimensionFilter.jsx             (NEW - multiselect dimension filter)
      GroupBySelector.jsx             (NEW - dimension group-by dropdown)
      CompareToggle.jsx               (NEW - compare to previous period)
      MonthSelector.jsx               (NEW - for MID Performance)

      layouts/
        Section.jsx                   (NEW - section wrapper)
        CardRow.jsx                   (NEW - horizontal card layout)
        Grid.jsx                      (NEW - CSS grid layout)
        FullWidth.jsx                 (NEW - single column)
        TabsSection.jsx               (NEW - tabbed sections)

      widgets/
        MetricCard.jsx                (NEW - summary card with trend)
        Chart.jsx                     (NEW - Recharts wrapper)
        DataTable.jsx                 (NEW - TanStack Table wrapper)
        PivotTable.jsx                (NEW - cohort/retention matrix)
        SummaryTable.jsx              (NEW - multi-row summary)
        WaterfallChart.jsx            (NEW - custom waterfall)
        ParameterInput.jsx            (NEW - $/% input for profitability)
        HealthIndicator.jsx           (NEW - status badge widget)
        WidgetSkeleton.jsx            (NEW - loading skeleton per widget type)
        WidgetError.jsx               (NEW - error state with retry)
```

### 6.2 providers.jsx Changes

```jsx
'use client';

import { ThemeProvider } from '@/components/theme-provider';
import { Toaster as Sonner } from '@/components/ui/sonner';
import { TooltipProvider } from '@/components/ui/tooltip';
import { persistor, store } from '@/store/store';
import { Provider as ReduxProvider } from 'react-redux';
import { PersistGate } from 'redux-persist/integration/react';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@/lib/queryClient';

function Provider({ children }) {
  return (
    <ThemeProvider defaultTheme="light" storageKey="theme" disableTransitionOnChange>
      <TooltipProvider delayDuration={0}>
        <Sonner richColors />
        <QueryClientProvider client={queryClient}>
          <ReduxProvider store={store}>
            <PersistGate loading={null} persistor={persistor}>
              {children}
            </PersistGate>
          </ReduxProvider>
        </QueryClientProvider>
      </TooltipProvider>
    </ThemeProvider>
  );
}

export default Provider;
```

### 6.3 Zustand Store: Report Filter State

```javascript
// stores/reportFilterStore.js
import { create } from 'zustand';

export const createReportFilterStore = (templateKey) =>
  create((set, get) => ({
    // Date range
    dateRange: { type: 'preset', preset: 'last_7_days' },
    setDateRange: (dateRange) => set({ dateRange }),

    // Dimension filters
    dimensionFilters: {},
    setDimensionFilter: (key, filter) =>
      set((state) => ({
        dimensionFilters: { ...state.dimensionFilters, [key]: filter },
      })),
    clearDimensionFilter: (key) =>
      set((state) => {
        const { [key]: _, ...rest } = state.dimensionFilters;
        return { dimensionFilters: rest };
      }),
    clearAllFilters: () => set({ dimensionFilters: {} }),

    // Toggles (metric_variant, dimension_switch)
    toggles: {},
    setToggle: (key, value) =>
      set((state) => ({
        toggles: { ...state.toggles, [key]: value },
      })),

    // Parameters (profitability inputs)
    parameters: {},
    setParameter: (key, value) =>
      set((state) => ({
        parameters: { ...state.parameters, [key]: value },
      })),

    // Group by
    groupBy: null,
    setGroupBy: (groupBy) => set({ groupBy }),

    // Compare to previous period
    compareEnabled: false,
    setCompareEnabled: (enabled) => set({ compareEnabled: enabled }),

    // Get full filter state for API request
    getFilterState: () => {
      const state = get();
      return {
        dateRange: state.dateRange,
        dimensions: state.dimensionFilters,
        toggles: state.toggles,
        parameters: state.parameters,
        groupBy: state.groupBy,
        compare: state.compareEnabled,
      };
    },
  }));
```

### 6.4 Component Hierarchy

```
<NativeReportPage reportSlug="revenue-analytics">
  |
  |-- useReportTemplate(reportSlug)     → fetches template JSON from API
  |-- createReportFilterStore()          → creates Zustand store for this report
  |-- useReportData(template, filters)   → builds batch query, fetches via TanStack Query
  |
  ├── <FilterBar template={template.filterBar}>
  │   ├── <DateRangePicker />            (reuses existing custom-date-range-picker.tsx)
  │   ├── <ToggleControl />              (pills for approval_mode, dropdown for date_basis)
  │   ├── <DimensionFilter />            (multiselect dropdowns)
  │   ├── <GroupBySelector />            (dimension dropdown)
  │   ├── <ParameterInput />             ($/% inputs for profitability)
  │   └── <CompareToggle />              (vs previous period)
  │
  └── <ReportRenderer template={template} data={data}>
      ├── <Section type="card_row">
      │   ├── <MetricCard />             → formatted value + trend arrow + comparison %
      │   ├── <MetricCard />
      │   └── <MetricCard />
      ├── <Section type="full_width">
      │   └── <Chart chartType="line" /> → Recharts line/bar/area/pie/stacked
      ├── <Section type="tabs">
      │   ├── Tab: "By Product"
      │   │   └── <DataTable />          → TanStack Table, sortable, paginated
      │   ├── Tab: "By Campaign"
      │   │   └── <DataTable />
      │   └── Tab: "By Gateway"
      │       └── <DataTable />
      └── <Section type="full_width">
          └── <PivotTable />             → cohort matrix with heatmap
```

---

## 7. Report JSON Templates

### Template JSON Schema

Every template follows this structure:

```jsonc
{
  "version": "1.0",
  "source": "order_summary",              // primary data source
  "sourceConfig": {},                      // optional source-specific config
  "defaults": {
    "dateRange": "last_7_days",
    "refreshInterval": null                // seconds, or null for manual
  },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": false,
    "groupByOptions": [],
    "defaultGroupBy": null,
    "toggles": [],
    "filters": [],
    "parameters": [],
    "showExport": true
  },
  "sections": []                           // array of section objects
}
```

### Template 1: Business Command Center

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": {
    "dateRange": "last_7_days",
    "refreshInterval": 300
  },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": false,
    "toggles": [],
    "filters": [],
    "showExport": true
  },
  "sections": [
    {
      "id": "summary_cards",
      "type": "card_row",
      "widgets": [
        {
          "id": "revenue_card",
          "type": "metric_card",
          "title": "Revenue",
          "metrics": [{ "metricKey": "revenue", "label": "Total" }],
          "showTrend": true,
          "trendComparison": "previous_period"
        },
        {
          "id": "new_customers_card",
          "type": "metric_card",
          "title": "New Customers",
          "metrics": [{ "metricKey": "initials", "label": "Count" }],
          "showTrend": true,
          "trendComparison": "previous_period"
        },
        {
          "id": "active_subs_card",
          "type": "metric_card",
          "title": "Active Subscribers",
          "metrics": [{ "metricKey": "approvals", "label": "Count" }],
          "showTrend": true,
          "trendComparison": "previous_period"
        },
        {
          "id": "churn_card",
          "type": "metric_card",
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
      "widgets": [
        {
          "id": "hourly_revenue",
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
      "widgets": [
        {
          "id": "daily_summary",
          "type": "data_table",
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

### Template 2: Real-Time Pulse

```json
{
  "version": "1.0",
  "source": "hourly_revenue",
  "defaults": {
    "dateRange": "today",
    "refreshInterval": 300
  },
  "filterBar": {
    "showDateRange": false,
    "showGroupBy": false,
    "toggles": [],
    "filters": [],
    "showExport": true
  },
  "sections": [
    {
      "id": "running_totals",
      "type": "card_row",
      "widgets": [
        {
          "id": "today_revenue",
          "type": "metric_card",
          "title": "Revenue (Running)",
          "source": "hourly_revenue",
          "metrics": [{ "metricKey": "today_revenue_total", "label": "Today" }],
          "showTrend": false
        },
        {
          "id": "today_orders",
          "type": "metric_card",
          "title": "Orders (Running)",
          "source": "hourly_revenue",
          "metrics": [{ "metricKey": "today_orders_total", "label": "Today" }],
          "showTrend": false
        },
        {
          "id": "today_new",
          "type": "metric_card",
          "title": "New Customers",
          "source": "hourly_revenue",
          "metrics": [{ "metricKey": "today_initials_total", "label": "Today" }],
          "showTrend": false
        },
        {
          "id": "today_failures",
          "type": "metric_card",
          "title": "Payment Failures",
          "source": "hourly_revenue",
          "metrics": [{ "metricKey": "today_failures_total", "label": "Today" }],
          "showTrend": false
        }
      ]
    },
    {
      "id": "hourly_charts",
      "type": "grid",
      "columns": 2,
      "widgets": [
        {
          "id": "revenue_hourly",
          "type": "chart",
          "chartType": "area",
          "title": "Revenue by Hour",
          "xAxis": { "dimensionKey": "date_hour" },
          "series": [
            { "metricKey": "today_revenue", "label": "Today", "color": "#4F46E5" },
            { "metricKey": "avg_7d_revenue", "label": "7-Day Avg", "color": "#94A3B8", "style": "dashed" }
          ]
        },
        {
          "id": "success_hourly",
          "type": "chart",
          "chartType": "line",
          "title": "Payment Success Rate by Hour",
          "xAxis": { "dimensionKey": "date_hour" },
          "series": [
            { "metricKey": "today_success_rate", "label": "Success %", "color": "#10B981" }
          ]
        }
      ]
    },
    {
      "id": "hourly_table",
      "type": "full_width",
      "widgets": [
        {
          "id": "hourly_breakdown",
          "type": "data_table",
          "title": "Hour by Hour",
          "dimension": "date_hour",
          "metrics": [
            { "metricKey": "today_revenue" },
            { "metricKey": "today_orders" },
            { "metricKey": "today_initials" },
            { "metricKey": "today_rebills" },
            { "metricKey": "today_failures" },
            { "metricKey": "today_success_rate" }
          ],
          "sortBy": "date_hour",
          "sortDirection": "asc",
          "showTotals": true
        }
      ]
    }
  ]
}
```

### Template 3: Revenue Analytics

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": { "dateRange": "last_30_days" },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": true,
    "groupByOptions": ["campaign_name", "product_name", "gateway_alias", "sales_type", "card_brand"],
    "defaultGroupBy": "campaign_name",
    "toggles": [
      { "toggleKey": "approval_mode", "position": "main", "displayAs": "pills" },
      { "toggleKey": "time_granularity", "position": "main", "displayAs": "pills" }
    ],
    "filters": [
      { "dimensionKey": "sales_type", "displayAs": "pills", "position": "main" },
      { "dimensionKey": "campaign_name", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "gateway_alias", "displayAs": "multiselect", "position": "more" }
    ],
    "showExport": true
  },
  "sections": [
    {
      "id": "summary",
      "type": "card_row",
      "widgets": [
        {
          "id": "initials_card",
          "type": "metric_card",
          "title": "Initials",
          "metrics": [
            { "metricKey": "initials", "label": "Count" },
            { "metricKey": "revenue_initials", "label": "Revenue" }
          ],
          "showTrend": true
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
        },
        {
          "id": "rates_card",
          "type": "metric_card",
          "title": "Rates",
          "metrics": [
            { "metricKey": "approval_rate", "label": "Approval %" },
            { "metricKey": "aov", "label": "AOV" }
          ],
          "showTrend": true
        }
      ]
    },
    {
      "id": "trend",
      "type": "full_width",
      "widgets": [
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
      "widgets": [
        {
          "id": "revenue_table",
          "type": "data_table",
          "title": "Revenue by {groupBy}",
          "dimension": "{groupBy}",
          "metrics": [
            { "metricKey": "attempts" },
            { "metricKey": "approvals" },
            { "metricKey": "approval_rate" },
            { "metricKey": "revenue" },
            { "metricKey": "aov" },
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
          "exportable": true
        }
      ]
    }
  ]
}
```

### Template 4: Subscription Intelligence

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": { "dateRange": "last_6_months" },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": false,
    "toggles": [
      { "toggleKey": "time_granularity", "position": "main", "displayAs": "pills" }
    ],
    "filters": [
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "campaign_name", "displayAs": "multiselect", "position": "more" }
    ],
    "showExport": true
  },
  "sections": [
    {
      "id": "mrr_cards",
      "type": "card_row",
      "widgets": [
        { "id": "mrr_card", "type": "metric_card", "title": "MRR", "metrics": [{ "metricKey": "revenue_rebills", "label": "Monthly" }], "showTrend": true },
        { "id": "active_card", "type": "metric_card", "title": "Active Subscribers", "metrics": [{ "metricKey": "rebills", "label": "Count" }], "showTrend": true },
        { "id": "net_growth_card", "type": "metric_card", "title": "Net Growth", "metrics": [{ "metricKey": "approvals", "label": "New" }, { "metricKey": "cancels", "label": "Cancelled" }], "showTrend": true },
        { "id": "cancel_rate_card", "type": "metric_card", "title": "Cancel Rate", "metrics": [{ "metricKey": "cancel_rate", "label": "Rate" }], "showTrend": true, "invertTrend": true }
      ]
    },
    {
      "id": "mrr_trend",
      "type": "full_width",
      "widgets": [
        {
          "id": "mrr_chart",
          "type": "chart",
          "chartType": "area",
          "title": "Revenue Trend",
          "xAxis": { "dimensionKey": "{time_granularity}" },
          "series": [
            { "metricKey": "revenue_initials", "label": "New Revenue", "color": "#10B981" },
            { "metricKey": "revenue_rebills", "label": "Recurring Revenue", "color": "#4F46E5" }
          ],
          "stacked": true,
          "showLegend": true
        }
      ]
    },
    {
      "id": "subscriber_table",
      "type": "full_width",
      "widgets": [
        {
          "id": "monthly_flow",
          "type": "data_table",
          "title": "Monthly Subscriber Flow",
          "dimension": "date_month",
          "metrics": [
            { "metricKey": "initials" },
            { "metricKey": "rebills" },
            { "metricKey": "cancels" },
            { "metricKey": "cancel_rate" },
            { "metricKey": "revenue" }
          ],
          "sortBy": "date_month",
          "sortDirection": "desc",
          "showTotals": true
        }
      ]
    }
  ]
}
```

### Template 5: Customer Lifecycle

```json
{
  "version": "1.0",
  "source": "cohort_summary",
  "defaults": { "dateRange": "last_6_months" },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": false,
    "toggles": [
      { "toggleKey": "retention_base", "position": "main", "displayAs": "pills" }
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
      "widgets": [
        { "id": "ltv_30", "type": "metric_card", "title": "30-Day LTV", "metrics": [{ "metricKey": "ltv_30d" }], "showTrend": true },
        { "id": "ltv_90", "type": "metric_card", "title": "90-Day LTV", "metrics": [{ "metricKey": "ltv_90d" }], "showTrend": true },
        { "id": "ltv_180", "type": "metric_card", "title": "180-Day LTV", "metrics": [{ "metricKey": "ltv_180d" }], "showTrend": true },
        { "id": "retention_m1", "type": "metric_card", "title": "M1 Retention", "metrics": [{ "metricKey": "retention_cycle_1" }], "showTrend": true }
      ]
    },
    {
      "id": "retention_matrix",
      "type": "full_width",
      "widgets": [
        {
          "id": "cohort_heatmap",
          "type": "pivot_table",
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
      "widgets": [
        {
          "id": "ltv_by_source",
          "type": "data_table",
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

### Template 6: Churn Analysis

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": { "dateRange": "last_3_months" },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": true,
    "groupByOptions": ["product_name", "campaign_name", "billing_cycle"],
    "defaultGroupBy": "product_name",
    "toggles": [
      { "toggleKey": "time_granularity", "position": "main", "displayAs": "pills" }
    ],
    "filters": [
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "more" }
    ],
    "showExport": true
  },
  "sections": [
    {
      "id": "churn_cards",
      "type": "card_row",
      "widgets": [
        { "id": "total_churn", "type": "metric_card", "title": "Total Cancel Rate", "metrics": [{ "metricKey": "cancel_rate" }], "showTrend": true, "invertTrend": true },
        { "id": "cancels", "type": "metric_card", "title": "Cancels", "metrics": [{ "metricKey": "cancels" }], "showTrend": true, "invertTrend": true },
        { "id": "cb_rate", "type": "metric_card", "title": "CB Rate", "metrics": [{ "metricKey": "cb_rate" }], "showTrend": true, "invertTrend": true },
        { "id": "refund_rate", "type": "metric_card", "title": "Refund Rate", "metrics": [{ "metricKey": "refund_rate" }], "showTrend": true, "invertTrend": true }
      ]
    },
    {
      "id": "churn_trend",
      "type": "full_width",
      "widgets": [
        {
          "id": "churn_chart",
          "type": "chart",
          "chartType": "line",
          "title": "Churn Trend",
          "xAxis": { "dimensionKey": "{time_granularity}" },
          "series": [
            { "metricKey": "cancel_rate", "label": "Cancel %", "color": "#EF4444" },
            { "metricKey": "cb_rate", "label": "CB %", "color": "#F59E0B" },
            { "metricKey": "refund_rate", "label": "Refund %", "color": "#6366F1" }
          ],
          "showLegend": true
        }
      ]
    },
    {
      "id": "churn_table",
      "type": "full_width",
      "widgets": [
        {
          "id": "churn_by_dimension",
          "type": "data_table",
          "title": "Churn by {groupBy}",
          "dimension": "{groupBy}",
          "metrics": [
            { "metricKey": "approvals" },
            { "metricKey": "cancels" },
            { "metricKey": "cancel_rate" },
            { "metricKey": "cb_count" },
            { "metricKey": "cb_rate" },
            { "metricKey": "refund_count" },
            { "metricKey": "refund_rate" }
          ],
          "sortBy": "cancel_rate",
          "sortDirection": "desc",
          "pagination": { "pageSize": 25 },
          "showTotals": true
        }
      ]
    }
  ]
}
```

### Template 7: Payment Health & Recovery

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": { "dateRange": "last_30_days" },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": true,
    "groupByOptions": ["campaign_name", "product_name", "gateway_alias", "card_brand", "decline_group"],
    "defaultGroupBy": "campaign_name",
    "toggles": [
      { "toggleKey": "approval_mode", "position": "main", "displayAs": "pills" },
      { "toggleKey": "time_granularity", "position": "main", "displayAs": "pills" }
    ],
    "filters": [
      { "dimensionKey": "sales_type", "displayAs": "pills", "position": "main" },
      { "dimensionKey": "gateway_alias", "displayAs": "multiselect", "position": "more" },
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "more" }
    ],
    "showExport": true
  },
  "sections": [
    {
      "id": "health_cards",
      "type": "card_row",
      "widgets": [
        { "id": "approval_card", "type": "metric_card", "title": "Approval Rate", "metrics": [{ "metricKey": "approval_rate" }], "showTrend": true },
        { "id": "attempts_card", "type": "metric_card", "title": "Attempts", "metrics": [{ "metricKey": "attempts" }, { "metricKey": "approvals" }], "showTrend": true },
        { "id": "decline_card", "type": "metric_card", "title": "Declines", "metrics": [{ "metricKey": "declines" }], "showTrend": true, "invertTrend": true },
        { "id": "recovery_card", "type": "metric_card", "title": "Recovery Rate", "source": "decline_recovery", "metrics": [{ "metricKey": "recovery_rate" }], "showTrend": true }
      ]
    },
    {
      "id": "approval_trend",
      "type": "full_width",
      "widgets": [
        {
          "id": "approval_chart",
          "type": "chart",
          "chartType": "line",
          "title": "Approval Rate Trend",
          "xAxis": { "dimensionKey": "{time_granularity}" },
          "series": [
            { "metricKey": "approval_rate", "label": "Approval %", "color": "#10B981" }
          ]
        }
      ]
    },
    {
      "id": "recovery_section",
      "type": "full_width",
      "widgets": [
        {
          "id": "recovery_table",
          "type": "data_table",
          "source": "decline_recovery",
          "title": "Recovery by Decline Group",
          "dimension": "decline_group",
          "metrics": [
            { "metricKey": "organic_declines" },
            { "metricKey": "reattempts" },
            { "metricKey": "recovered" },
            { "metricKey": "recovery_rate" },
            { "metricKey": "recovered_dollar" }
          ],
          "sortBy": "organic_declines",
          "sortDirection": "desc",
          "showTotals": true
        }
      ]
    },
    {
      "id": "breakdown",
      "type": "full_width",
      "widgets": [
        {
          "id": "approval_table",
          "type": "data_table",
          "title": "Approval Rate by {groupBy}",
          "dimension": "{groupBy}",
          "metrics": [
            { "metricKey": "attempts" },
            { "metricKey": "approvals" },
            { "metricKey": "approval_rate" },
            { "metricKey": "revenue" }
          ],
          "sortBy": "attempts",
          "sortDirection": "desc",
          "pagination": { "pageSize": 25 },
          "showTotals": true
        }
      ]
    }
  ]
}
```

### Template 8: Channel & Acquisition Performance

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": { "dateRange": "last_90_days" },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": false,
    "toggles": [],
    "filters": [
      { "dimensionKey": "product_name", "displayAs": "multiselect", "position": "more" }
    ],
    "showExport": true
  },
  "sections": [
    {
      "id": "channel_cards",
      "type": "card_row",
      "widgets": [
        { "id": "total_new", "type": "metric_card", "title": "New Customers", "metrics": [{ "metricKey": "initials" }], "showTrend": true },
        { "id": "avg_cpa", "type": "metric_card", "title": "Avg CPA", "metrics": [{ "metricKey": "cpa_cost" }], "showTrend": true, "invertTrend": true },
        { "id": "best_channel", "type": "metric_card", "title": "Revenue (Initials)", "metrics": [{ "metricKey": "revenue_initials" }], "showTrend": true }
      ]
    },
    {
      "id": "channel_table",
      "type": "full_width",
      "widgets": [
        {
          "id": "channel_scorecard",
          "type": "data_table",
          "title": "Channel Scorecard",
          "dimension": "campaign_name",
          "metrics": [
            { "metricKey": "initials" },
            { "metricKey": "revenue_initials" },
            { "metricKey": "approval_rate" },
            { "metricKey": "cancel_rate" },
            { "metricKey": "cb_rate" },
            { "metricKey": "aov" }
          ],
          "sortBy": "revenue_initials",
          "sortDirection": "desc",
          "pagination": { "pageSize": 25 },
          "showTotals": true,
          "allowColumnToggle": true
        }
      ]
    }
  ]
}
```

### Template 9: Financial Performance

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": { "dateRange": "last_30_days" },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": true,
    "groupByOptions": ["product_name", "campaign_name"],
    "defaultGroupBy": "product_name",
    "toggles": [
      { "toggleKey": "profitability_view", "position": "main", "displayAs": "pills" }
    ],
    "parameters": [
      { "key": "processing_fee_pct", "label": "Processing Fees", "type": "number", "suffix": "%", "default": 3.5, "step": 0.1, "min": 0, "max": 20 },
      { "key": "reserve_pct", "label": "Reserve", "type": "number", "suffix": "%", "default": 5.0, "step": 0.5, "min": 0, "max": 50 },
      { "key": "cb_fee_amount", "label": "CB Fee", "type": "number", "prefix": "$", "default": 25.00, "step": 1.0, "min": 0, "max": 100 },
      { "key": "cpa_amount", "label": "CPA", "type": "number", "prefix": "$", "default": 45.00, "step": 1.0, "min": 0, "max": 500 }
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
      "conditionalVisibility": { "toggleKey": "profitability_view", "showWhen": "cashflow" },
      "widgets": [
        {
          "id": "cashflow_waterfall",
          "type": "waterfall_chart",
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
      "widgets": [
        { "id": "revenue_card", "type": "metric_card", "title": "Gross Revenue", "metrics": [{ "metricKey": "revenue" }], "showTrend": true },
        { "id": "net_revenue_card", "type": "metric_card", "title": "Net Revenue", "metrics": [{ "metricKey": "net_revenue" }], "showTrend": true },
        { "id": "profit_card", "type": "metric_card", "title": "Gross Profit", "metrics": [{ "metricKey": "gross_profit" }], "showTrend": true },
        { "id": "margin_card", "type": "metric_card", "title": "Gross Margin", "metrics": [{ "metricKey": "gross_margin" }], "showTrend": true }
      ]
    },
    {
      "id": "profit_table",
      "type": "full_width",
      "widgets": [
        {
          "id": "profit_by_product",
          "type": "data_table",
          "title": "Profitability by {groupBy}",
          "dimension": "{groupBy}",
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

### Template 10: Product & Plan Performance

```json
{
  "version": "1.0",
  "source": "order_summary",
  "defaults": { "dateRange": "last_30_days" },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": false,
    "toggles": [],
    "filters": [],
    "showExport": true
  },
  "sections": [
    {
      "id": "product_table",
      "type": "full_width",
      "widgets": [
        {
          "id": "product_scorecard",
          "type": "data_table",
          "title": "Product Performance",
          "dimension": "product_name",
          "metrics": [
            { "metricKey": "approvals" },
            { "metricKey": "revenue" },
            { "metricKey": "aov" },
            { "metricKey": "approval_rate" },
            { "metricKey": "cancel_rate" },
            { "metricKey": "cb_rate" },
            { "metricKey": "refund_rate" }
          ],
          "sortBy": "revenue",
          "sortDirection": "desc",
          "showTotals": true,
          "allowColumnToggle": true
        }
      ]
    },
    {
      "id": "product_trend",
      "type": "full_width",
      "widgets": [
        {
          "id": "product_chart",
          "type": "chart",
          "chartType": "stacked_bar",
          "title": "Revenue by Product (Weekly)",
          "xAxis": { "dimensionKey": "date_week" },
          "dimension": "product_name",
          "metric": "revenue",
          "showLegend": true
        }
      ]
    }
  ]
}
```

### Template 11: Transaction Explorer

```json
{
  "version": "1.0",
  "source": "order_details",
  "defaults": { "dateRange": "last_7_days" },
  "sourceConfig": {
    "serverSidePagination": true,
    "serverSideSort": true,
    "serverSideSearch": true,
    "searchColumns": ["order_id", "bill_email", "bill_first", "bill_last", "transaction_id"]
  },
  "filterBar": {
    "showDateRange": true,
    "showGroupBy": false,
    "showSearch": true,
    "searchPlaceholder": "Search by ID, email, name...",
    "toggles": [],
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
      "widgets": [
        {
          "id": "transaction_table",
          "type": "data_table",
          "title": "Transactions",
          "isDetailTable": true,
          "columns": [
            { "key": "date_of_sale", "label": "Date" },
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

---

## 8. API Endpoints

### Phase 1 Endpoints

```
QUERY (protected by auth middleware)
  POST   /api/v1/query              Single widget query
  POST   /api/v1/query/batch        Multiple widget queries in parallel

TEMPLATES (protected by auth middleware)
  GET    /api/v1/reports/templates              List available templates for user
  GET    /api/v1/reports/templates/:templateKey  Get template with full layout JSON

FILTERS (protected by auth middleware)
  GET    /api/v1/filters/options/:dimensionKey  Get distinct values for filter dropdown
```

### Phase 2 Endpoints (future)

```
TEMPLATES
  POST   /api/v1/reports/templates              Create custom template
  PUT    /api/v1/reports/templates/:id           Update template
  DELETE /api/v1/reports/templates/:id           Delete (user-created only)

SAVED VIEWS
  GET    /api/v1/views                          List user's saved views
  POST   /api/v1/views                          Create saved view
  PUT    /api/v1/views/:id                      Update saved view
  DELETE /api/v1/views/:id                      Delete saved view

FILTER PRESETS
  GET    /api/v1/filters/presets                List presets
  POST   /api/v1/filters/presets                Save preset
  DELETE /api/v1/filters/presets/:id            Delete preset

DASHBOARDS
  GET    /api/v1/dashboards                     List dashboards
  GET    /api/v1/dashboards/:id                 Get dashboard
  POST   /api/v1/dashboards                     Create dashboard
  PUT    /api/v1/dashboards/:id                 Update dashboard
  POST   /api/v1/dashboards/:id/widgets         Add widget
  PUT    /api/v1/dashboards/:id/widgets/:wid    Update widget
  DELETE /api/v1/dashboards/:id/widgets/:wid    Remove widget

SCHEDULES
  GET    /api/v1/schedules                      List schedules
  POST   /api/v1/schedules                      Create schedule
  PUT    /api/v1/schedules/:id                  Update schedule
  DELETE /api/v1/schedules/:id                  Delete schedule

EXPORT
  POST   /api/v1/reports/export                 Export report data

SEMANTIC LAYER
  GET    /api/v1/semantic/metrics               List all metrics
  GET    /api/v1/semantic/metrics/:category      Metrics by category
  GET    /api/v1/semantic/dimensions            List all dimensions
  GET    /api/v1/semantic/data-sources          List data sources
```

### Query API Request/Response

**Single Query:**
```jsonc
POST /api/v1/query
{
  "source": "order_summary",
  "metrics": ["approvals", "revenue", "approval_rate"],
  "dimensions": ["campaign_name"],
  "filters": {
    "dateRange": { "type": "preset", "preset": "last_7_days" },
    "dimensions": {
      "sales_type": { "operator": "in", "values": ["Initials"] }
    }
  },
  "toggles": { "approval_mode": "organic" },
  "parameters": {},
  "sort": { "field": "revenue", "direction": "desc" },
  "pagination": { "page": 0, "pageSize": 25 },
  "includeTotal": true
}
```

**Response:**
```jsonc
{
  "code": 200,
  "data": {
    "rows": [
      { "campaign_name": "Summer Sale", "approvals": 1234, "revenue": 45678.90, "approval_rate": 0.7823 }
    ],
    "totals": { "approvals": 15678, "revenue": 567890.12, "approval_rate": 0.7456 }
  },
  "metadata": {
    "totalRows": 156,
    "page": 0,
    "pageSize": 25,
    "queryTimeMs": 45
  }
}
```

**Batch Query:**
```jsonc
POST /api/v1/query/batch
{
  "toggles": { "approval_mode": "organic" },
  "filters": {
    "dateRange": { "type": "preset", "preset": "last_7_days" }
  },
  "queries": [
    {
      "widgetId": "summary_cards",
      "source": "order_summary",
      "metrics": ["initials", "rebills", "revenue"]
    },
    {
      "widgetId": "trend_chart",
      "source": "order_summary",
      "metrics": ["approvals"],
      "dimensions": ["date_day"]
    },
    {
      "widgetId": "breakdown_table",
      "source": "order_summary",
      "metrics": ["approvals", "revenue", "approval_rate"],
      "dimensions": ["campaign_name"],
      "sort": { "field": "revenue", "direction": "desc" },
      "pagination": { "page": 0, "pageSize": 25 }
    }
  ]
}
```

**Batch Response:**
```jsonc
{
  "code": 200,
  "results": {
    "summary_cards": { "data": { "rows": [...] }, "metadata": { "queryTimeMs": 12 } },
    "trend_chart": { "data": { "rows": [...] }, "metadata": { "queryTimeMs": 18 } },
    "breakdown_table": { "data": { "rows": [...], "totals": {...} }, "metadata": { "queryTimeMs": 35, "totalRows": 156 } }
  },
  "metadata": { "totalTimeMs": 38 }
}
```

---

## 9. Performance Strategy

### Target: <500ms per page load

```
Layer 1: TanStack Query (browser)
  - 5 min stale time
  - Repeat page visits = instant (0ms)
  - Tab switching = instant
  - gcTime: 30 minutes

Layer 2: PostgreSQL Materialized Views
  - Pre-computed aggregated data per client
  - With proper indexes: 10-50ms per query
  - 5-10 queries in parallel: 50-150ms total
  - Existing ETL handles refresh schedule
```

### Why This Is Fast Enough Without Redis

| Component | Latency |
|-----------|---------|
| HTTP request overhead | ~20ms |
| Auth middleware | ~5ms |
| Query engine (SQL generation) | ~10ms |
| PostgreSQL execution (indexed materialized view) | 10-50ms |
| Response formatting | ~5ms |
| **Total per query** | **~50-90ms** |
| **Batch (5 queries in parallel)** | **~90-150ms** |
| **Frontend rendering** | **~100-200ms** |
| **Total page load** | **~200-350ms** |

### Required Indexes on Materialized Views

```sql
-- Ensure these exist on all materialized views
CREATE INDEX IF NOT EXISTS idx_os_{cid}_date ON reporting.order_summary_{client_id}(date);
CREATE INDEX IF NOT EXISTS idx_os_{cid}_campaign ON reporting.order_summary_{client_id}(campaign_id);
CREATE INDEX IF NOT EXISTS idx_os_{cid}_product ON reporting.order_summary_{client_id}(product_id);
CREATE INDEX IF NOT EXISTS idx_os_{cid}_gateway ON reporting.order_summary_{client_id}(gateway_id);
CREATE INDEX IF NOT EXISTS idx_os_{cid}_sales_type ON reporting.order_summary_{client_id}(sales_type);
CREATE INDEX IF NOT EXISTS idx_os_{cid}_composite ON reporting.order_summary_{client_id}(date, campaign_id, product_id);
```

### Connection Pool Configuration

```javascript
// db/queryPool.js
const { Pool } = require('pg');

const queryPool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  database: process.env.DB_NAME,
  user: process.env.DB_QUERY_USER,
  password: process.env.DB_QUERY_PASSWORD,
  max: 20,                    // max connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
  statement_timeout: 30000,   // 30 second query timeout
});
```

---

## 10. Integration with Existing System

### What Does NOT Change

| Component | Path | Status |
|-----------|------|--------|
| Login routes | `/api/user/*` | Unchanged |
| Auth middleware | `services/middlewares/auth.js` | Reused as-is |
| JWT token structure | `{ email, id }` | Unchanged |
| User roles | superadmin, consultant, admin, user | Unchanged |
| Client management | `/api/client/*` | Unchanged |
| Raw data routes | `/api/data/*` | Unchanged |
| BIN routing | `/api/bin-routing/*` | Unchanged |
| Prisma schema | `prisma/schema.prisma` | Extended (not modified) |
| Power BI reports | `components/reports/Reports.jsx` | Runs in parallel |
| Redux store | `store/store.js` | Unchanged |
| Sidebar | `components/sidebar/AppSidebar.jsx` | Modified to support dual routing |

### Feature Flag: Report Mode

Add `report_mode` column to `beast_insights_v2.clients`:

```sql
ALTER TABLE beast_insights_v2.clients
ADD COLUMN report_mode VARCHAR(20) DEFAULT 'powerbi';
-- Values: 'powerbi' | 'native' | 'both'
```

**Behavior:**
- `powerbi` - existing behavior, no changes
- `native` - sidebar routes to `/reports/native/{slug}` instead of Power BI
- `both` - sidebar shows both options, toggle button in toolbar

### Sidebar Integration

The existing `AppSidebar.jsx` reads navigation items from `beast_insights_v2.navigation_items`. For native mode:

1. Add a `native_template_key` column to `navigation_items` or maintain a mapping table
2. When `report_mode != 'powerbi'`, sidebar links route to `/reports/native/{template_key}` instead of `/reports/{power_bi_page_route}`
3. When `report_mode == 'both'`, show an icon toggle per report to switch between views

### Client ID Flow

```
Existing Redux clientSlice
  -> stores selectedClientId
  -> used by all existing features

New reports:
  -> Custom hook reads clientId from Redux
  -> Passes to TanStack Query as query key
  -> Sent to backend in API requests
  -> Backend extracts from JWT (never trusts frontend clientId)
```

```javascript
// hooks/useClientId.js
import { useSelector } from 'react-redux';

export function useClientId() {
  return useSelector((state) => state.client.selectedClientId);
}
```

### Existing Components Reused

| Component | Path | Used For |
|-----------|------|----------|
| `custom-date-range-picker.tsx` | `components/ui/` | Date range picker in FilterBar |
| `tabs.tsx` | `components/ui/` | Tabbed sections in reports |
| `select.tsx` | `components/ui/` | Dropdown filters and selectors |
| `button.tsx` | `components/ui/` | All buttons |
| `card.tsx` | `components/ui/` | MetricCard wrapper |
| `badge.tsx` | `components/ui/` | Status badges |
| `skeleton.tsx` | `components/ui/` | Loading states |
| `tooltip.tsx` | `components/ui/` | Metric explanations |
| `popover.tsx` | `components/ui/` | Filter dropdowns |
| `table.tsx` | `components/ui/` | Base table styling |
| Theme system | `components/theme-provider.jsx` | Dark/light mode support |

---

## Summary

```
WHAT WE'RE BUILDING:

Phase 1: The Engine
  - 14 new backend files (query engine, routes, controllers)
  - 20+ new frontend files (components, stores, hooks)
  - 11 report JSON templates
  - 1 new database schema with 15 tables
  - 0 changes to existing auth, login, or Power BI code

Phase 2: Customization (after Phase 1)
Phase 3: Access Control (after Phase 2)
Phase 4: Industry Modules (after Phase 3)

KEY PRINCIPLES:
  - No Redis in Phase 1 (materialized views + TanStack Query are sufficient)
  - No changes to login routes
  - Power BI runs in parallel until native is validated
  - Feature-flagged per client
  - One generic endpoint, one generic renderer
  - JSON controls everything — no code deploys for new reports/metrics
```

---

*Beast Insights Development Phase Plan*
*Generated: 2026-01-27*

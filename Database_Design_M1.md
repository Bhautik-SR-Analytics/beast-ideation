# Milestone 1 — Database Design Document

> Complete schema design for the `report_builder` PostgreSQL schema — tables, columns, relationships, indexes, and ER diagrams.

---

## Table of Contents

1. [Naming Conventions](#1-naming-conventions)
2. [Schema Overview](#2-schema-overview)
3. [ER Diagram](#3-er-diagram)
4. [Table Definitions — Phase 1 (Core)](#4-table-definitions--phase-1-core)
5. [Table Definitions — Phase 2 (Customization)](#5-table-definitions--phase-2-customization)
6. [Table Definitions — Phase 3 (Access Control)](#6-table-definitions--phase-3-access-control)
7. [Table Definitions — Phase 4 (Industry Modules)](#7-table-definitions--phase-4-industry-modules)
8. [Existing Table Modifications](#8-existing-table-modifications)
9. [Index Strategy](#9-index-strategy)
10. [Relationship Map](#10-relationship-map)
11. [Seed Data Summary](#11-seed-data-summary)

---

## 1. Naming Conventions

Derived from analyzing the existing `beast_insights_v2` schema (98 tables):

| Element | Convention | Examples from Current DB |
|---------|-----------|------------------------|
| Schema name | `snake_case` | `beast_insights_v2`, `reporting`, `routing` |
| Table names | `snake_case`, plural | `clients`, `users`, `alerts`, `roles`, `metrics` |
| Column names | `snake_case` | `client_id`, `created_at`, `is_active` |
| Primary key | `id` as `BIGINT`, autoincrement | `id BIGINT PRIMARY KEY DEFAULT autoincrement()` |
| PK constraint | `{table}_pk` | `clients_pk`, `alerts_pk`, `roles_pk` |
| Foreign key column | `{referenced_table_singular}_id` | `client_id`, `role_id`, `user_id`, `page_id` |
| FK constraint | `{table}_{referenced_table}_id_fk` | `alerts_clients_id_fk`, `users_roles_id_fk` |
| Unique constraint | `{table}_pk_2` or descriptive | `users_pk_2` (email unique) |
| Index | `{table}_{columns}_index` | `daily_data_sync_client_id_date_index` |
| Timestamps | `created_at`, `updated_at` as `TIMESTAMP(6)` | Both default to `CURRENT_TIMESTAMP` |
| Soft delete | `is_active BOOLEAN DEFAULT true` + `is_deleted BOOLEAN DEFAULT false` | Every major table uses this pair |
| Booleans | `is_` prefix | `is_active`, `is_deleted`, `is_internal`, `is_new_etl` |
| Audit columns | `created_by VARCHAR`, `modified_by VARCHAR` → FK to `users.id` | Used on `alerts`, `goals`, `pages`, `reports` |
| String PKs | Only `users.id` uses `VARCHAR` PK | All others use `BIGINT` autoincrement |
| JSON columns | `JSONB` type | `filter_data`, `options`, `gateways_returned` |

**New schema will follow these conventions exactly.**

New schema name: **`report_builder`**

---

## 2. Schema Overview

### Phase 1 Tables (populated during Milestone 5)

| # | Table | Purpose | Approx Rows |
|---|-------|---------|-------------|
| 1 | `data_sources` | Materialized view registry | ~10 |
| 2 | `metrics` | Metric catalog (SQL expressions, formats, types) | ~100 |
| 3 | `dimensions` | Dimension catalog (columns, joins) | ~30 |
| 4 | `data_source_metrics` | Junction: which metrics work with which sources | ~500 |
| 5 | `data_source_dimensions` | Junction: which dimensions work with which sources | ~200 |
| 6 | `metric_toggles` | Toggle definitions (approval_mode, date_basis, etc.) | ~6 |
| 7 | `metric_toggle_options` | Options per toggle (standard, organic, net, etc.) | ~20 |
| 8 | `metric_variants` | SQL overrides per metric/toggle/option combination | ~50 |
| 9 | `report_templates` | JSON report layout definitions | ~11 |

### Phase 2 Tables (created empty, populated in Phase 2)

| # | Table | Purpose |
|---|-------|---------|
| 10 | `saved_views` | Per-user report customizations |
| 11 | `filter_presets` | Named, shareable filter sets |
| 12 | `dashboards` | Personal dashboard layouts |
| 13 | `dashboard_widgets` | Pinned widgets on dashboards |
| 14 | `scheduled_reports` | Scheduled email/Telegram delivery |
| 15 | `export_jobs` | CSV/Excel/PDF export tracking |

### Phase 3 Tables (created empty, populated in Phase 3)

| # | Table | Purpose |
|---|-------|---------|
| 16 | `user_dimension_access` | Row-level dimension permissions |
| 17 | `dimension_access_audit` | Audit log for access changes |

### Phase 4 Tables (created empty, populated in Phase 4)

| # | Table | Purpose |
|---|-------|---------|
| 18 | `client_modules` | Per-client industry module enablement |

### Existing Table Modification

| Table | Change |
|-------|--------|
| `beast_insights_v2.clients` | Add `report_mode VARCHAR DEFAULT 'powerbi'` |
| `beast_insights_v2.client_features` | Add `is_native_reports_enabled BOOLEAN DEFAULT false` |

**Total: 18 new tables + 2 column additions to existing tables**

---

## 3. ER Diagram

### Phase 1 — Core Semantic Layer

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           REPORT TEMPLATE LAYER                                  │
│                                                                                  │
│  ┌─────────────────────┐                                                         │
│  │  report_templates    │                                                         │
│  ├─────────────────────┤                                                         │
│  │ id (PK)             │                                                         │
│  │ template_key (UQ)   │                                                         │
│  │ name                │                                                         │
│  │ category            │                                                         │
│  │ layout (JSONB)      │                                                         │
│  │ default_filters     │                                                         │
│  │ default_date_range  │                                                         │
│  │ nav_section         │                                                         │
│  │ nav_order           │                                                         │
│  └─────────────────────┘                                                         │
└──────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────┐
│                             SEMANTIC LAYER                                       │
│                                                                                  │
│  ┌──────────────────┐     ┌──────────────────────┐     ┌──────────────────┐      │
│  │  data_sources     │     │ data_source_metrics   │     │  metrics          │     │
│  ├──────────────────┤     ├──────────────────────┤     ├──────────────────┤      │
│  │ id (PK)          │◄────│ data_source_id (FK)  │────►│ id (PK)          │      │
│  │ source_key (UQ)  │     │ metric_id (FK)       │     │ metric_key (UQ)  │      │
│  │ display_name     │     │ id (PK)              │     │ display_name     │      │
│  │ view_pattern     │     └──────────────────────┘     │ sql_expression   │      │
│  │ date_column      │                                   │ data_type        │      │
│  │ refresh_frequency│     ┌──────────────────────┐     │ format_pattern   │      │
│  └──────────────────┘     │data_source_dimensions│     │ category         │      │
│           ▲               ├──────────────────────┤     │ aggregation_type │      │
│           │               │ data_source_id (FK)  │     │ is_derived       │      │
│           └───────────────│ dimension_id (FK)    │     │ derived_formula  │      │
│                           │ id (PK)              │     └──────────────────┘      │
│                           └──────────────────────┘              ▲                 │
│                                      │                          │                 │
│                                      ▼                          │                 │
│                           ┌──────────────────┐                  │                 │
│                           │  dimensions       │                  │                 │
│                           ├──────────────────┤                  │                 │
│                           │ id (PK)          │                  │                 │
│                           │ dimension_key(UQ)│                  │                 │
│                           │ display_name     │                  │                 │
│                           │ column_expression│                  │                 │
│                           │ join_table       │                  │                 │
│                           │ join_condition   │                  │                 │
│                           │ data_type        │                  │                 │
│                           └──────────────────┘                  │                 │
└──────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────┐
│                             TOGGLE / VARIANT LAYER                               │
│                                                                                  │
│  ┌──────────────────┐     ┌──────────────────────┐     ┌──────────────────┐      │
│  │  metric_toggles   │     │metric_toggle_options  │     │ metric_variants  │      │
│  ├──────────────────┤     ├──────────────────────┤     ├──────────────────┤      │
│  │ id (PK)          │◄────│ toggle_id (FK)       │◄────│ toggle_option_id │      │
│  │ toggle_key (UQ)  │     │ option_key           │     │   (FK)           │      │
│  │ display_name     │     │ display_name         │     │ metric_id (FK) ──┼──────┘
│  │ toggle_type      │     │ is_default           │     │ sql_expression   │
│  │ description      │     │ id (PK)              │     │ id (PK)          │
│  └──────────────────┘     └──────────────────────┘     └──────────────────┘
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Phase 2 — User Customization

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  ┌──────────────────┐     ┌──────────────────────┐                               │
│  │  saved_views      │     │  filter_presets       │                              │
│  ├──────────────────┤     ├──────────────────────┤                               │
│  │ id (PK)          │     │ id (PK)              │                               │
│  │ user_id (FK)─────┼─┐   │ user_id (FK)─────────┼─┐                            │
│  │ template_id (FK) │ │   │ client_id (FK)       │ │                             │
│  │ name             │ │   │ name                 │ │                              │
│  │ filters (JSONB)  │ │   │ filters (JSONB)      │ │                              │
│  │ layout (JSONB)   │ │   │ is_shared            │ │                              │
│  └──────────────────┘ │   └──────────────────────┘ │                              │
│                        │                             │   ┌──────────────┐         │
│  ┌──────────────────┐ │   ┌──────────────────────┐  │   │ users        │         │
│  │  dashboards       │ │   │  dashboard_widgets   │  │   │ (existing)   │         │
│  ├──────────────────┤ │   ├──────────────────────┤  │   │ beast_       │         │
│  │ id (PK)          │ │   │ id (PK)              │  └──►│ insights_v2  │         │
│  │ user_id (FK)─────┼─┤   │ dashboard_id (FK)    │      └──────────────┘         │
│  │ client_id (FK)   │ │   │ widget_config (JSONB)│                                │
│  │ name             │ │   │ position (JSONB)     │                                │
│  │ layout (JSONB)   │ │   └──────────────────────┘                                │
│  └──────────────────┘ │                                                           │
│                        │   ┌──────────────────────┐                               │
│  ┌──────────────────┐ │   │  export_jobs          │                               │
│  │scheduled_reports  │ │   ├──────────────────────┤                               │
│  ├──────────────────┤ │   │ id (PK)              │                               │
│  │ id (PK)          │ │   │ user_id (FK)─────────┼──┘                            │
│  │ user_id (FK)─────┼─┘   │ template_id (FK)     │                               │
│  │ template_id (FK) │     │ format               │                                │
│  │ frequency        │     │ status               │                                │
│  │ delivery_channel │     │ file_url             │                                │
│  │ filters (JSONB)  │     └──────────────────────┘                                │
│  └──────────────────┘                                                             │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Phase 3 — Access Control

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  ┌──────────────────────────┐      ┌──────────────────────────────┐              │
│  │ user_dimension_access     │      │  dimension_access_audit       │             │
│  ├──────────────────────────┤      ├──────────────────────────────┤              │
│  │ id (PK)                  │      │ id (PK)                      │              │
│  │ user_id (FK) ────────────┼──┐   │ user_dimension_access_id(FK) │              │
│  │ dimension_id (FK)        │  │   │ action                       │              │
│  │ allowed_values (JSONB)   │  │   │ performed_by (FK)            │              │
│  │ client_id (FK)           │  │   │ old_values (JSONB)           │              │
│  │ created_by (FK)          │  │   │ new_values (JSONB)           │              │
│  └──────────────────────────┘  │   │ created_at                   │              │
│                                 │   └──────────────────────────────┘              │
│                                 ▼                                                 │
│                          ┌──────────────┐                                        │
│                          │ users        │                                         │
│                          │ (existing)   │                                         │
│                          └──────────────┘                                         │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Phase 4 — Industry Modules

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  ┌──────────────────────────┐                                                    │
│  │  client_modules           │                                                   │
│  ├──────────────────────────┤      ┌──────────────┐                              │
│  │ id (PK)                  │      │ clients      │                              │
│  │ client_id (FK) ──────────┼─────►│ (existing)   │                              │
│  │ module_key               │      └──────────────┘                              │
│  │ is_enabled               │                                                    │
│  │ enabled_at               │                                                    │
│  │ enabled_by (FK)          │                                                    │
│  └──────────────────────────┘                                                    │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Table Definitions — Phase 1 (Core)

### 4.1 `data_sources`

Maps logical data source names to their physical materialized view patterns in the `reporting` schema.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `source_key` | `VARCHAR(100)` | NO | — | Unique logical name (e.g., `order_summary`) |
| `display_name` | `VARCHAR(200)` | NO | — | Human-readable name (e.g., `Order Summary`) |
| `view_pattern` | `VARCHAR(500)` | NO | — | Pattern: `reporting.order_summary_{client_id}` |
| `date_column` | `VARCHAR(100)` | YES | `'date'` | Name of the date column used for date range filters |
| `client_id_column` | `VARCHAR(100)` | NO | `'client_id'` | Name of the client_id column |
| `refresh_frequency_minutes` | `INTEGER` | YES | `60` | How often the materialized view refreshes |
| `description` | `TEXT` | YES | — | What this data source contains |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record update time |

**Constraints:**
- `PK`: `data_sources_pk` on `id`
- `UNIQUE`: `data_sources_source_key_uq` on `source_key`

**Existing materialized views this maps to (10 sources):**

| `source_key` | `view_pattern` | `date_column` | Columns |
|--------------|----------------|---------------|---------|
| `order_summary` | `reporting.order_summary_{client_id}` | `date` | 37 columns: client_id, date, campaign_id, product_id, gateway_id, bin, affid, sub_affid, c, price_point, sales_type, billing_cycle, refund_type, alert_type, cpa, attempts_total, attempts, approvals, net_approvals, approvals_organic, cancel, cancel_organic, revenue, cb, cb_dollar, cb_organic, cb_non_organic, cb_organic_dollar, cb_cb_date, cb_cb_date_dollar, refund, refund_dollar, refund_organic, refund_organic_dollar, refund_refund_date, refund_refund_date_dollar, trial_gateway_id |
| `mid_summary` | `reporting.mid_summary_{client_id}` | `month_year` | 51 columns: client_id, gateway_id, month_year, volume, initials, rebills, overall_cb, cb_visa, cb_master, overall_declines, decline_percent, attempts, approvals, overall_refund, alert, rdr, ethoca, cb_rate, cb_visa_rate, cb_master_rate, decline_rate, alert_rate, health_tag, monthly_cap, capacity_left, near_capacity, etc. |
| `cohort_summary` | `reporting.cohort_summary_{client_id}` | `date` | 25 columns: client_id, date, sales_type, billing_cycle, attempt_col, refund_type, alert_type, cpa, trial_price_point, trial_bin, trial_sub_affid, trial_c, trial_affid, trial_campaign_id, trial_product_id, trial_gateway_id, attempts, approvals, net_approvals, cancel, revenue, cb, cb_dollar, refund, refund_dollar |
| `cb_refund_alert` | `reporting.cb_refund_alert_{client_id}` | `date` | 33 columns: client_id, date, campaign_id, product_id, gateway_id, bin, affid, sub_affid, c, price_point, sales_type, billing_cycle, refund_type, alert_type, cpa, cb_no_of_days, refund_no_of_days, dispute_no_of_days, attempts_total, attempts, approvals, net_approvals, approvals_organic, cancel, revenue, cb, cb_dollar, cb_cb_date, cb_cb_date_dollar, refund, refund_dollar, refund_refund_date, refund_refund_date_dollar |
| `decline_recovery` | `reporting.decline_recovery_{client_id}` | `date` | 27 columns: client_id, date, decline_group, gateway_id, bin, campaign_id, product_id, affid, sub_affid, c, price_point, sales_type, billing_cycle, recovery_attempts, organic_declines, declines, reattempts, recovered, cancel, cb, refund, cb_cb_date, refund_refund_date, organic_declines_dollar, reattempts_dollar, not_reattempts_dollar, recovered_dollar |
| `hourly_revenue` | `reporting.hourly_revenue_{client_id}` | `hour` | 11 columns: client_id, sort_order, hour, avg_7d_revenue, avg_7d_initial, avg_7d_rebill, avg_7d_straight_sales, today_revenue, today_initial, today_rebill, today_straight_sales |
| `alert_summary` | `reporting.alert_summary_{client_id}` | `date` | 31 columns: date, client_id, gateway_id, alert_type, alert_status, alert_duplication, card_bin, is_crm, is_refund_void, is_cb_final, campaign_id, product_id, billing_cycle, attempt_sort, price_point, affid, sub_affid, c, is_tc40, transaction_amount, alert_count, alert_dollar, rdr, rdr_dollar, ethoca, ethoca_dollar, cdrn, cdrn_dollar, other_alert, other_alert_dollar, distinct_alert_count |
| `alert_details` | `reporting.alert_details_{client_id}` | `alert_date` | 31 columns: client_id, alert_id, alert_date, alert_type, alert_status, alert_duplication, order_id, bill_email, is_cb, is_refund_void, transaction_amount, transaction_date, gateway_id, gateway_alias, is_approved, is_crm, card_bin, card_group, campaign_id, product_id, billing_cycle, attempt_sort, price_point, affid, sub_affid, c, subscription_type, product_group, count_of_alert, alert_dollar, alert_count |
| `order_details` | `reporting.order_details_{client_id}` | `date_of_sale` | 84 columns: client_name, client_id, order_id, bill_first, bill_last, bill_city, bill_state, bill_zip, bill_country, bill_phone, bill_email, order_total, date_of_sale, time_of_sale, campaign_id, gateway_id, order_status, decline_reason, is_chargeback, chargeback_date, is_refund, refund_amount, refund_date, product_id, product_name, affid, sub_affid, cycle, attempt, bin, alert_type, alert_date, decline_group, c, products_json, etc. |
| `products` | `reporting.products_{client_id}` | — (no date) | 21 columns: product_id, product_name, product_description, product_sku, product_price, vertical_name, product_is_shippable, cost_of_goods_sold, crm, client_id, client_name, product_group, subscription_flag, product_cost, cpa, is_initial, is_exclude, subscription_type, etc. |

---

### 4.2 `metrics`

The full metric catalog. Every metric the platform can compute.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `metric_key` | `VARCHAR(100)` | NO | — | Unique identifier (e.g., `approvals`, `approval_rate`, `revenue`) |
| `display_name` | `VARCHAR(200)` | NO | — | UI label (e.g., `Approvals`, `Approval Rate`, `Revenue`) |
| `description` | `TEXT` | YES | — | Human-readable description of what this metric measures |
| `sql_expression` | `TEXT` | NO | — | SQL aggregation (e.g., `SUM(approvals)`, `SUM(revenue) / NULLIF(SUM(approvals), 0)`) |
| `data_type` | `VARCHAR(50)` | NO | `'number'` | Output type: `integer`, `decimal`, `currency`, `percentage`, `text` |
| `format_pattern` | `VARCHAR(50)` | YES | — | Display format (e.g., `$#,##0.00`, `#,##0`, `0.00%`) |
| `category` | `VARCHAR(100)` | NO | — | Grouping: `sales`, `revenue`, `chargebacks`, `refunds`, `recovery`, `ltv`, `profitability`, `real_time`, `mid_health`, `alerts` |
| `aggregation_type` | `VARCHAR(50)` | NO | `'sum'` | How to aggregate: `sum`, `count`, `avg`, `ratio`, `custom` |
| `is_derived` | `BOOLEAN` | NO | `false` | True if this metric is calculated from other metrics (not a direct column) |
| `derived_formula` | `TEXT` | YES | — | Formula using other metric_keys: e.g., `approval_rate = approvals / attempts * 100` |
| `sort_order` | `INTEGER` | YES | `0` | Display ordering within category |
| `is_invertable` | `BOOLEAN` | NO | `false` | Whether higher = worse (e.g., CB rate — higher is bad) |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record update time |

**Constraints:**
- `PK`: `metrics_pk` on `id`
- `UNIQUE`: `metrics_metric_key_uq` on `metric_key`
- `INDEX`: `metrics_category_idx` on `category`

**Note:** This table is different from the existing `beast_insights_v2.metrics` table which only has `id`, `name`, `is_active`, `is_deleted`, `created_at`, `updated_at`, `is_display`, `powerbi_metric`, `data_type`. The existing table is used for alerts/goals (6 columns). The new `report_builder.metrics` table is the full semantic layer catalog (15 columns) with SQL expressions, formats, and derivation formulas.

---

### 4.3 `dimensions`

Every dimension users can group by or filter on.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `dimension_key` | `VARCHAR(100)` | NO | — | Unique identifier (e.g., `campaign_name`, `product_name`, `gateway_alias`) |
| `display_name` | `VARCHAR(200)` | NO | — | UI label (e.g., `Campaign`, `Product`, `Gateway`) |
| `description` | `TEXT` | YES | — | What this dimension represents |
| `column_expression` | `TEXT` | NO | — | SQL column or expression (e.g., `campaign_id`, `TO_CHAR(date, 'YYYY-MM')`) |
| `display_column` | `VARCHAR(200)` | YES | — | Column for display value if different from filter value (e.g., filter by `campaign_id`, display `campaign_name`) |
| `join_table` | `VARCHAR(200)` | YES | — | Table to JOIN for display name lookup (e.g., `public.campaigns`) |
| `join_condition` | `TEXT` | YES | — | JOIN ON clause (e.g., `{source}.campaign_id = campaigns.campaign_id AND campaigns.client_id = {client_id}`) |
| `data_type` | `VARCHAR(50)` | NO | `'text'` | `text`, `integer`, `date`, `boolean` |
| `filter_type` | `VARCHAR(50)` | NO | `'multi_select'` | How this appears in the filter bar: `multi_select`, `single_select`, `date_range`, `search` |
| `is_groupable` | `BOOLEAN` | NO | `true` | Can this be used in GROUP BY |
| `is_filterable` | `BOOLEAN` | NO | `true` | Can this be used as a WHERE filter |
| `category` | `VARCHAR(100)` | YES | — | Grouping for filter bar UI (e.g., `transaction`, `marketing`, `payment`, `time`) |
| `sort_order` | `INTEGER` | YES | `0` | Display ordering |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record update time |

**Constraints:**
- `PK`: `dimensions_pk` on `id`
- `UNIQUE`: `dimensions_dimension_key_uq` on `dimension_key`
- `INDEX`: `dimensions_category_idx` on `category`

---

### 4.4 `data_source_metrics`

Junction table: which metrics are valid for which data sources.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `data_source_id` | `BIGINT` | NO | — | FK → `data_sources.id` |
| `metric_id` | `BIGINT` | NO | — | FK → `metrics.id` |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |

**Constraints:**
- `PK`: `data_source_metrics_pk` on `id`
- `UNIQUE`: `data_source_metrics_uq` on `(data_source_id, metric_id)`
- `FK`: `data_source_metrics_data_sources_id_fk` → `data_sources.id` ON DELETE CASCADE
- `FK`: `data_source_metrics_metrics_id_fk` → `metrics.id` ON DELETE CASCADE

---

### 4.5 `data_source_dimensions`

Junction table: which dimensions can be used with which data sources.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `data_source_id` | `BIGINT` | NO | — | FK → `data_sources.id` |
| `dimension_id` | `BIGINT` | NO | — | FK → `dimensions.id` |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |

**Constraints:**
- `PK`: `data_source_dimensions_pk` on `id`
- `UNIQUE`: `data_source_dimensions_uq` on `(data_source_id, dimension_id)`
- `FK`: `data_source_dimensions_data_sources_id_fk` → `data_sources.id` ON DELETE CASCADE
- `FK`: `data_source_dimensions_dimensions_id_fk` → `dimensions.id` ON DELETE CASCADE

---

### 4.6 `metric_toggles`

Toggle definitions — context switches that change how metrics are calculated.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `toggle_key` | `VARCHAR(100)` | NO | — | Unique identifier (e.g., `approval_mode`, `date_basis`) |
| `display_name` | `VARCHAR(200)` | NO | — | UI label (e.g., `Approval Mode`, `Date Basis`) |
| `description` | `TEXT` | YES | — | What this toggle controls |
| `toggle_type` | `VARCHAR(50)` | NO | `'metric_variant'` | `metric_variant` (changes SQL expression) or `dimension_switch` (changes GROUP BY dimension) |
| `ui_component` | `VARCHAR(50)` | NO | `'pill_group'` | How rendered in FilterBar: `pill_group`, `dropdown`, `toggle_switch` |
| `sort_order` | `INTEGER` | YES | `0` | Display order in filter bar |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record update time |

**Constraints:**
- `PK`: `metric_toggles_pk` on `id`
- `UNIQUE`: `metric_toggles_toggle_key_uq` on `toggle_key`

**Seed data (6 toggles):**

| `toggle_key` | `display_name` | `toggle_type` | `ui_component` |
|--------------|----------------|---------------|-----------------|
| `approval_mode` | Approval Mode | `metric_variant` | `pill_group` |
| `date_basis` | Date Basis | `metric_variant` | `pill_group` |
| `time_granularity` | Time Period | `dimension_switch` | `dropdown` |
| `profitability_view` | Profitability View | `metric_variant` | `dropdown` |
| `retention_base` | Retention Base | `metric_variant` | `pill_group` |
| `display_mode` | Display Mode | `metric_variant` | `pill_group` |

---

### 4.7 `metric_toggle_options`

Available options for each toggle.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `toggle_id` | `BIGINT` | NO | — | FK → `metric_toggles.id` |
| `option_key` | `VARCHAR(100)` | NO | — | Unique within toggle (e.g., `standard`, `organic`, `net`) |
| `display_name` | `VARCHAR(200)` | NO | — | UI label |
| `description` | `TEXT` | YES | — | What this option does |
| `is_default` | `BOOLEAN` | NO | `false` | Is this the default selected option |
| `sort_order` | `INTEGER` | YES | `0` | Display order |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |

**Constraints:**
- `PK`: `metric_toggle_options_pk` on `id`
- `UNIQUE`: `metric_toggle_options_uq` on `(toggle_id, option_key)`
- `FK`: `metric_toggle_options_metric_toggles_id_fk` → `metric_toggles.id` ON DELETE CASCADE

**Seed data examples:**

| Toggle | Options |
|--------|---------|
| `approval_mode` | `standard` (default), `organic`, `net` |
| `date_basis` | `transaction_date` (default), `initial_date`, `cb_date`, `refund_date` |
| `time_granularity` | `day` (default), `week`, `month` |
| `profitability_view` | `gross` (default), `net`, `detailed` |
| `retention_base` | `initials` (default), `rebills` |
| `display_mode` | `count` (default), `dollar`, `percentage` |

---

### 4.8 `metric_variants`

SQL expression overrides when a specific toggle option is active.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `metric_id` | `BIGINT` | NO | — | FK → `metrics.id` |
| `toggle_option_id` | `BIGINT` | NO | — | FK → `metric_toggle_options.id` |
| `sql_expression` | `TEXT` | NO | — | Override SQL (e.g., `SUM(approvals_organic)`) |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |

**Constraints:**
- `PK`: `metric_variants_pk` on `id`
- `UNIQUE`: `metric_variants_uq` on `(metric_id, toggle_option_id)`
- `FK`: `metric_variants_metrics_id_fk` → `metrics.id` ON DELETE CASCADE
- `FK`: `metric_variants_metric_toggle_options_id_fk` → `metric_toggle_options.id` ON DELETE CASCADE

**How the query engine uses this:**

1. Frontend sends: `metrics: ["approvals"], toggles: { approval_mode: "organic" }`
2. Query engine looks up `metric_variants` where `metric_id = (approvals)` AND `toggle_option_id = (organic)`
3. Found: `sql_expression = "SUM(approvals_organic)"` → uses this instead of the default `SUM(approvals)`
4. If no variant found → falls back to `metrics.sql_expression` (the default)

**Cascading rule:** If `approvals` has a variant for `organic`, then any derived metric using `approvals` (like `approval_rate = approvals / attempts * 100`) automatically picks up the variant — the query engine resolves base metrics first, then substitutes into derived formulas.

---

### 4.9 `report_templates`

JSON report layout definitions. Each report is stored as a single JSONB blob.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `template_key` | `VARCHAR(100)` | NO | — | URL slug (e.g., `revenue-analytics`, `command-center`) |
| `name` | `VARCHAR(200)` | NO | — | Display name (e.g., `Revenue Analytics`, `Command Center`) |
| `description` | `TEXT` | YES | — | Report description |
| `category` | `VARCHAR(100)` | NO | — | Report grouping: `operations`, `analytics`, `financial`, `lifecycle`, `risk` |
| `layout` | `JSONB` | NO | — | Full report layout JSON (sections, widgets, configs) |
| `default_filters` | `JSONB` | YES | `'{}'` | Default filter values (e.g., `{"dateRange": "last_30_days"}`) |
| `default_date_range` | `VARCHAR(50)` | YES | `'last_30_days'` | Default date range preset |
| `required_data_sources` | `VARCHAR[]` | NO | — | Array of `source_key` values this report needs |
| `required_toggles` | `VARCHAR[]` | YES | — | Array of `toggle_key` values used in this report |
| `nav_section` | `VARCHAR(100)` | YES | — | Sidebar section (e.g., `operations`, `analytics`) |
| `nav_order` | `INTEGER` | YES | `0` | Order within nav section |
| `nav_icon` | `VARCHAR(100)` | YES | — | Icon name for sidebar |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `is_deleted` | `BOOLEAN` | NO | `false` | Soft-delete flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record update time |
| `created_by` | `VARCHAR` | YES | — | FK → `beast_insights_v2.users.id` |

**Constraints:**
- `PK`: `report_templates_pk` on `id`
- `UNIQUE`: `report_templates_template_key_uq` on `template_key`
- `INDEX`: `report_templates_category_idx` on `category`
- `INDEX`: `report_templates_nav_section_idx` on `nav_section`

**11 stock templates to seed:**

| `template_key` | `name` | `category` | `nav_section` | Data Sources |
|----------------|--------|------------|---------------|--------------|
| `command-center` | Command Center | `operations` | `operations` | `hourly_revenue`, `order_summary` |
| `real-time-pulse` | Real-Time Pulse | `operations` | `operations` | `hourly_revenue` |
| `revenue-analytics` | Revenue Analytics | `analytics` | `analytics` | `order_summary` |
| `subscription-intelligence` | Subscription Intelligence | `analytics` | `analytics` | `order_summary`, `cohort_summary` |
| `customer-lifecycle` | Customer Lifecycle | `lifecycle` | `lifecycle` | `cohort_summary` |
| `churn-analysis` | Churn & Retention | `lifecycle` | `lifecycle` | `order_summary`, `cohort_summary` |
| `payment-health-recovery` | Payment Health & Recovery | `risk` | `risk` | `decline_recovery`, `mid_summary` |
| `channel-acquisition` | Channel & Acquisition | `analytics` | `analytics` | `order_summary` |
| `financial-performance` | Financial Performance | `financial` | `financial` | `order_summary` |
| `product-plan-performance` | Product & Plan Performance | `analytics` | `analytics` | `order_summary`, `products` |
| `transaction-explorer` | Transaction Explorer | `operations` | `operations` | `order_details` |

---

## 5. Table Definitions — Phase 2 (Customization)

> All Phase 2 tables are created empty during Milestone 1. No data, no APIs — just the schema.

### 5.1 `saved_views`

Per-user saved customizations of a stock report.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `template_id` | `BIGINT` | NO | — | FK → `report_templates.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `name` | `VARCHAR(200)` | NO | — | User-given name (e.g., `My Q4 View`) |
| `filters` | `JSONB` | YES | `'{}'` | Saved filter state |
| `layout_overrides` | `JSONB` | YES | `'{}'` | Widget visibility/order overrides |
| `toggles` | `JSONB` | YES | `'{}'` | Saved toggle state |
| `is_default` | `BOOLEAN` | NO | `false` | Auto-load this view when opening the report |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `is_deleted` | `BOOLEAN` | NO | `false` | Soft-delete flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `saved_views_pk` on `id`
- `FK`: `saved_views_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `saved_views_report_templates_id_fk` → `report_templates.id` ON DELETE CASCADE
- `FK`: `saved_views_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `INDEX`: `saved_views_user_id_template_id_idx` on `(user_id, template_id)`

---

### 5.2 `filter_presets`

Named, shareable filter combinations.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `name` | `VARCHAR(200)` | NO | — | Preset name (e.g., `Last 90 Days - Top Campaigns`) |
| `filters` | `JSONB` | NO | — | Filter state JSON |
| `is_shared` | `BOOLEAN` | NO | `false` | Visible to all users in this client |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `is_deleted` | `BOOLEAN` | NO | `false` | |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `filter_presets_pk` on `id`
- `FK`: `filter_presets_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `filter_presets_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `INDEX`: `filter_presets_user_id_client_id_idx` on `(user_id, client_id)`

---

### 5.3 `dashboards`

Personal dashboard layouts.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `name` | `VARCHAR(200)` | NO | — | Dashboard name |
| `description` | `TEXT` | YES | — | |
| `layout` | `JSONB` | YES | `'[]'` | react-grid-layout positions |
| `is_default` | `BOOLEAN` | NO | `false` | Default dashboard for this user |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `is_deleted` | `BOOLEAN` | NO | `false` | |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `dashboards_pk` on `id`
- `FK`: `dashboards_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `dashboards_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE

---

### 5.4 `dashboard_widgets`

Pinned widgets on a personal dashboard.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `dashboard_id` | `BIGINT` | NO | — | FK → `dashboards.id` |
| `source_template_id` | `BIGINT` | YES | — | FK → `report_templates.id` (which report this widget came from) |
| `widget_type` | `VARCHAR(50)` | NO | — | `metric_card`, `chart`, `data_table`, etc. |
| `widget_config` | `JSONB` | NO | — | Complete widget configuration (metrics, dimensions, filters, chart type) |
| `position` | `JSONB` | NO | — | `{ x, y, w, h }` for react-grid-layout |
| `title` | `VARCHAR(200)` | YES | — | Custom title override |
| `sort_order` | `INTEGER` | YES | `0` | |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `dashboard_widgets_pk` on `id`
- `FK`: `dashboard_widgets_dashboards_id_fk` → `dashboards.id` ON DELETE CASCADE
- `FK`: `dashboard_widgets_report_templates_id_fk` → `report_templates.id` ON DELETE SET NULL

---

### 5.5 `scheduled_reports`

Scheduled email/Telegram report delivery.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `template_id` | `BIGINT` | NO | — | FK → `report_templates.id` |
| `name` | `VARCHAR(200)` | NO | — | Schedule name |
| `frequency` | `VARCHAR(50)` | NO | — | `daily`, `weekly`, `monthly` |
| `day_of_week` | `INTEGER` | YES | — | 0-6 for weekly schedules |
| `day_of_month` | `INTEGER` | YES | — | 1-31 for monthly schedules |
| `time_of_day` | `TIME` | NO | `'08:00'` | When to send |
| `timezone` | `VARCHAR(100)` | NO | `'America/New_York'` | User's timezone |
| `delivery_channel` | `VARCHAR(50)` | NO | `'email'` | `email`, `telegram`, `slack` |
| `delivery_target` | `VARCHAR(500)` | YES | — | Email address or chat ID |
| `filters` | `JSONB` | YES | `'{}'` | Locked filter state for this schedule |
| `export_format` | `VARCHAR(50)` | NO | `'pdf'` | `pdf`, `csv`, `excel` |
| `last_sent_at` | `TIMESTAMP(6)` | YES | — | Last successful delivery |
| `next_send_at` | `TIMESTAMP(6)` | YES | — | Pre-computed next send time |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `is_deleted` | `BOOLEAN` | NO | `false` | |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `scheduled_reports_pk` on `id`
- `FK`: `scheduled_reports_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `scheduled_reports_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `FK`: `scheduled_reports_report_templates_id_fk` → `report_templates.id` ON DELETE CASCADE
- `INDEX`: `scheduled_reports_next_send_at_idx` on `next_send_at` (for cron job pickup)

---

### 5.6 `export_jobs`

Tracks export requests (CSV, Excel, PDF).

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `template_id` | `BIGINT` | YES | — | FK → `report_templates.id` |
| `format` | `VARCHAR(50)` | NO | — | `csv`, `excel`, `pdf` |
| `status` | `VARCHAR(50)` | NO | `'pending'` | `pending`, `processing`, `completed`, `failed` |
| `filters` | `JSONB` | YES | — | Filter state at time of export |
| `file_url` | `TEXT` | YES | — | S3/storage URL once generated |
| `file_size_bytes` | `BIGINT` | YES | — | |
| `error_message` | `TEXT` | YES | — | Error detail if failed |
| `completed_at` | `TIMESTAMP(6)` | YES | — | |
| `expires_at` | `TIMESTAMP(6)` | YES | — | Auto-delete exported file after |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `export_jobs_pk` on `id`
- `FK`: `export_jobs_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `INDEX`: `export_jobs_user_id_status_idx` on `(user_id, status)`

---

## 6. Table Definitions — Phase 3 (Access Control)

### 6.1 `user_dimension_access`

Maps users to the dimension values they are allowed to see. The query engine auto-injects WHERE clauses based on these entries.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `dimension_id` | `BIGINT` | NO | — | FK → `dimensions.id` |
| `allowed_values` | `JSONB` | NO | — | Array of allowed IDs/values: `[101, 203, 305]` |
| `access_type` | `VARCHAR(50)` | NO | `'include'` | `include` (only these) or `exclude` (everything except) |
| `created_by` | `VARCHAR` | YES | — | FK → `beast_insights_v2.users.id` |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `user_dimension_access_pk` on `id`
- `UNIQUE`: `user_dimension_access_uq` on `(user_id, client_id, dimension_id)`
- `FK`: `user_dimension_access_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `user_dimension_access_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `FK`: `user_dimension_access_dimensions_id_fk` → `dimensions.id` ON DELETE CASCADE
- `INDEX`: `user_dimension_access_user_client_idx` on `(user_id, client_id)`

**Example entries:**

| user_id | dimension_key | allowed_values | Meaning |
|---------|---------------|----------------|---------|
| `usr_123` | `campaign_name` (via dimension_id) | `[101, 205]` | User can only see campaigns 101 and 205 |
| `usr_123` | `product_name` (via dimension_id) | `[40, 41, 42]` | User can only see products 40, 41, 42 |
| `usr_456` | `gateway_alias` (via dimension_id) | `[1001]` | User can only see gateway 1001 |

**Query engine injection:** When user `usr_123` queries `order_summary` grouped by campaign, the engine adds: `WHERE campaign_id IN (101, 205)` — invisible to the user.

---

### 6.2 `dimension_access_audit`

Audit trail for dimension access changes.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_dimension_access_id` | `BIGINT` | YES | — | FK → `user_dimension_access.id` (nullable for deletes) |
| `target_user_id` | `VARCHAR` | NO | — | The user whose access was changed |
| `action` | `VARCHAR(50)` | NO | — | `created`, `updated`, `deleted` |
| `performed_by` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` (admin who made the change) |
| `old_values` | `JSONB` | YES | — | Previous allowed_values |
| `new_values` | `JSONB` | YES | — | New allowed_values |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `dimension_access_audit_pk` on `id`
- `INDEX`: `dimension_access_audit_target_user_idx` on `target_user_id`
- `INDEX`: `dimension_access_audit_created_at_idx` on `created_at`

---

## 7. Table Definitions — Phase 4 (Industry Modules)

### 7.1 `client_modules`

Per-client industry module enablement.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `module_key` | `VARCHAR(100)` | NO | — | Module identifier: `high_risk`, `telehealth`, `saas`, `ecommerce` |
| `is_enabled` | `BOOLEAN` | NO | `false` | |
| `enabled_at` | `TIMESTAMP(6)` | YES | — | When the module was turned on |
| `enabled_by` | `VARCHAR` | YES | — | FK → `beast_insights_v2.users.id` |
| `config` | `JSONB` | YES | `'{}'` | Module-specific configuration |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `client_modules_pk` on `id`
- `UNIQUE`: `client_modules_uq` on `(client_id, module_key)`
- `FK`: `client_modules_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE

---

## 8. Existing Table Modifications

### 8.1 `beast_insights_v2.clients` — Add `report_mode`

```sql
ALTER TABLE beast_insights_v2.clients
ADD COLUMN report_mode VARCHAR(20) NOT NULL DEFAULT 'powerbi';
```

| Column | Type | Default | Allowed Values |
|--------|------|---------|---------------|
| `report_mode` | `VARCHAR(20)` | `'powerbi'` | `powerbi`, `native`, `both` |

**Purpose:** Controls which reporting system each client uses.
- `powerbi` — existing Power BI embedded reports (current default)
- `native` — new native report renderer only
- `both` — shows a toggle switch in the UI to flip between them

### 8.2 `beast_insights_v2.client_features` — Add `is_native_reports_enabled`

```sql
ALTER TABLE beast_insights_v2.client_features
ADD COLUMN is_native_reports_enabled BOOLEAN NOT NULL DEFAULT false;
```

**Purpose:** Feature flag at the client level. Works alongside `report_mode` — allows gradual rollout per-client.

Current `client_features` columns:
- `id`, `client_id`, `is_bin_routing_enabled`, `created_at`, `updated_at`, `is_dashboard_enabled`, `is_mid_enabled`, `is_router_enabled`, `is_order_details_enabled`

Adding `is_native_reports_enabled` follows the exact same boolean flag pattern.

---

## 9. Index Strategy

### Performance-Critical Indexes

| Table | Index Name | Columns | Reason |
|-------|-----------|---------|--------|
| `metrics` | `metrics_category_idx` | `category` | Filter metrics by category in UI |
| `metrics` | `metrics_metric_key_uq` | `metric_key` UNIQUE | Lookup by key (query engine) |
| `dimensions` | `dimensions_category_idx` | `category` | Filter dimensions by category |
| `dimensions` | `dimensions_dimension_key_uq` | `dimension_key` UNIQUE | Lookup by key (query engine) |
| `data_sources` | `data_sources_source_key_uq` | `source_key` UNIQUE | Resolve source by key |
| `data_source_metrics` | `data_source_metrics_uq` | `(data_source_id, metric_id)` UNIQUE | Validate metric-source pair |
| `data_source_metrics` | `data_source_metrics_data_source_id_idx` | `data_source_id` | List metrics for a source |
| `data_source_dimensions` | `data_source_dimensions_uq` | `(data_source_id, dimension_id)` UNIQUE | Validate dimension-source pair |
| `data_source_dimensions` | `data_source_dimensions_data_source_id_idx` | `data_source_id` | List dimensions for a source |
| `metric_variants` | `metric_variants_uq` | `(metric_id, toggle_option_id)` UNIQUE | Fast variant lookup |
| `metric_variants` | `metric_variants_metric_id_idx` | `metric_id` | All variants for a metric |
| `metric_toggle_options` | `metric_toggle_options_uq` | `(toggle_id, option_key)` UNIQUE | Resolve option within toggle |
| `report_templates` | `report_templates_template_key_uq` | `template_key` UNIQUE | URL slug lookup |
| `report_templates` | `report_templates_category_idx` | `category` | List reports by category |
| `saved_views` | `saved_views_user_id_template_id_idx` | `(user_id, template_id)` | List user's views for a report |
| `user_dimension_access` | `user_dimension_access_user_client_idx` | `(user_id, client_id)` | Fetch user's access rules |
| `scheduled_reports` | `scheduled_reports_next_send_at_idx` | `next_send_at` | Cron job pickup |
| `export_jobs` | `export_jobs_user_id_status_idx` | `(user_id, status)` | User's export history |

---

## 10. Relationship Map

### Cross-Schema References

The `report_builder` schema references the existing `beast_insights_v2` schema in these places:

| report_builder Table | Column | References |
|---------------------|--------|------------|
| `report_templates` | `created_by` | `beast_insights_v2.users.id` |
| `saved_views` | `user_id` | `beast_insights_v2.users.id` |
| `saved_views` | `client_id` | `beast_insights_v2.clients.id` |
| `filter_presets` | `user_id` | `beast_insights_v2.users.id` |
| `filter_presets` | `client_id` | `beast_insights_v2.clients.id` |
| `dashboards` | `user_id` | `beast_insights_v2.users.id` |
| `dashboards` | `client_id` | `beast_insights_v2.clients.id` |
| `scheduled_reports` | `user_id` | `beast_insights_v2.users.id` |
| `scheduled_reports` | `client_id` | `beast_insights_v2.clients.id` |
| `export_jobs` | `user_id` | `beast_insights_v2.users.id` |
| `export_jobs` | `client_id` | `beast_insights_v2.clients.id` |
| `user_dimension_access` | `user_id` | `beast_insights_v2.users.id` |
| `user_dimension_access` | `client_id` | `beast_insights_v2.clients.id` |
| `user_dimension_access` | `created_by` | `beast_insights_v2.users.id` |
| `client_modules` | `client_id` | `beast_insights_v2.clients.id` |
| `client_modules` | `enabled_by` | `beast_insights_v2.users.id` |

### Internal Relationships (within `report_builder`)

```
data_sources ──┐
               ├── data_source_metrics ──── metrics
               └── data_source_dimensions ── dimensions

metric_toggles ── metric_toggle_options ── metric_variants ── metrics

report_templates ◄── saved_views
report_templates ◄── scheduled_reports
report_templates ◄── dashboard_widgets
report_templates ◄── export_jobs

dashboards ◄── dashboard_widgets

dimensions ◄── user_dimension_access
user_dimension_access ◄── dimension_access_audit
```

### Existing Relationships Unchanged

All existing `beast_insights_v2` relationships remain exactly as they are:

| Relationship | Stays |
|-------------|-------|
| `users` → `roles` (via `role_id`) | Unchanged |
| `users_clients` → `users` + `clients` | Unchanged |
| `user_pages` → `users` + `pages` | Unchanged |
| `pages` → `reports` (via `report_id`) | Unchanged |
| `reports` → `clients` (via `client_id`) | Unchanged |
| `navigation_items` → `pages` (via `page_id`) | Unchanged |
| `alerts` → `users` + `metrics` + `clients` | Unchanged |
| `goals` → `users` + `metrics` + `clients` | Unchanged |
| `filter_definitions` → `filter_options` | Unchanged |
| `client_filter_assignments` → `clients` + `filter_definitions` | Unchanged |
| `user_filter_assignments` → `users` + `filter_definitions` | Unchanged |
| `client_features` → `clients` | Unchanged |

---

## 11. Seed Data Summary

Seeding happens during **Milestone 5** (Data Library from Power BI), but this is what the seed script will populate:

| Table | Rows | Source |
|-------|------|--------|
| `data_sources` | 10 | Manual mapping of existing `reporting.*` materialized views |
| `metrics` | ~100 | Translated from Power BI DAX measures |
| `dimensions` | ~30 | Extracted from materialized view columns + lookup tables |
| `data_source_metrics` | ~500 | Which metrics work with which materialized views |
| `data_source_dimensions` | ~200 | Which dimensions are available per materialized view |
| `metric_toggles` | 6 | The 6 context switches (approval_mode, date_basis, etc.) |
| `metric_toggle_options` | ~20 | Options per toggle |
| `metric_variants` | ~50 | SQL overrides per metric/toggle/option |
| `report_templates` | 11 | The 11 stock report JSON layouts |

Existing `beast_insights_v2` tables seeded:
| Table | Change |
|-------|--------|
| `clients.report_mode` | Set to `'powerbi'` for all existing clients (no visible change) |
| `client_features.is_native_reports_enabled` | Set to `false` for all existing clients (no visible change) |

---

## Appendix: Full CREATE TABLE SQL

> The actual migration SQL to be run via Prisma Migrate.

```sql
-- ============================================================
-- Schema: report_builder
-- ============================================================

CREATE SCHEMA IF NOT EXISTS report_builder;

-- ============================================================
-- 1. data_sources
-- ============================================================
CREATE TABLE report_builder.data_sources (
    id                          BIGSERIAL PRIMARY KEY
                                CONSTRAINT data_sources_pk,
    source_key                  VARCHAR(100) NOT NULL,
    display_name                VARCHAR(200) NOT NULL,
    view_pattern                VARCHAR(500) NOT NULL,
    date_column                 VARCHAR(100) DEFAULT 'date',
    client_id_column            VARCHAR(100) NOT NULL DEFAULT 'client_id',
    refresh_frequency_minutes   INTEGER DEFAULT 60,
    description                 TEXT,
    is_active                   BOOLEAN NOT NULL DEFAULT true,
    created_at                  TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT data_sources_source_key_uq UNIQUE (source_key)
);

-- ============================================================
-- 2. metrics
-- ============================================================
CREATE TABLE report_builder.metrics (
    id                  BIGSERIAL PRIMARY KEY
                        CONSTRAINT metrics_pk,
    metric_key          VARCHAR(100) NOT NULL,
    display_name        VARCHAR(200) NOT NULL,
    description         TEXT,
    sql_expression      TEXT NOT NULL,
    data_type           VARCHAR(50) NOT NULL DEFAULT 'number',
    format_pattern      VARCHAR(50),
    category            VARCHAR(100) NOT NULL,
    aggregation_type    VARCHAR(50) NOT NULL DEFAULT 'sum',
    is_derived          BOOLEAN NOT NULL DEFAULT false,
    derived_formula     TEXT,
    sort_order          INTEGER DEFAULT 0,
    is_invertable       BOOLEAN NOT NULL DEFAULT false,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT metrics_metric_key_uq UNIQUE (metric_key)
);

CREATE INDEX metrics_category_idx ON report_builder.metrics (category);

-- ============================================================
-- 3. dimensions
-- ============================================================
CREATE TABLE report_builder.dimensions (
    id                  BIGSERIAL PRIMARY KEY
                        CONSTRAINT dimensions_pk,
    dimension_key       VARCHAR(100) NOT NULL,
    display_name        VARCHAR(200) NOT NULL,
    description         TEXT,
    column_expression   TEXT NOT NULL,
    display_column      VARCHAR(200),
    join_table          VARCHAR(200),
    join_condition      TEXT,
    data_type           VARCHAR(50) NOT NULL DEFAULT 'text',
    filter_type         VARCHAR(50) NOT NULL DEFAULT 'multi_select',
    is_groupable        BOOLEAN NOT NULL DEFAULT true,
    is_filterable       BOOLEAN NOT NULL DEFAULT true,
    category            VARCHAR(100),
    sort_order          INTEGER DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT dimensions_dimension_key_uq UNIQUE (dimension_key)
);

CREATE INDEX dimensions_category_idx ON report_builder.dimensions (category);

-- ============================================================
-- 4. data_source_metrics
-- ============================================================
CREATE TABLE report_builder.data_source_metrics (
    id              BIGSERIAL PRIMARY KEY
                    CONSTRAINT data_source_metrics_pk,
    data_source_id  BIGINT NOT NULL,
    metric_id       BIGINT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT data_source_metrics_uq
        UNIQUE (data_source_id, metric_id),
    CONSTRAINT data_source_metrics_data_sources_id_fk
        FOREIGN KEY (data_source_id) REFERENCES report_builder.data_sources(id)
        ON DELETE CASCADE,
    CONSTRAINT data_source_metrics_metrics_id_fk
        FOREIGN KEY (metric_id) REFERENCES report_builder.metrics(id)
        ON DELETE CASCADE
);

CREATE INDEX data_source_metrics_data_source_id_idx
    ON report_builder.data_source_metrics (data_source_id);

-- ============================================================
-- 5. data_source_dimensions
-- ============================================================
CREATE TABLE report_builder.data_source_dimensions (
    id              BIGSERIAL PRIMARY KEY
                    CONSTRAINT data_source_dimensions_pk,
    data_source_id  BIGINT NOT NULL,
    dimension_id    BIGINT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT data_source_dimensions_uq
        UNIQUE (data_source_id, dimension_id),
    CONSTRAINT data_source_dimensions_data_sources_id_fk
        FOREIGN KEY (data_source_id) REFERENCES report_builder.data_sources(id)
        ON DELETE CASCADE,
    CONSTRAINT data_source_dimensions_dimensions_id_fk
        FOREIGN KEY (dimension_id) REFERENCES report_builder.dimensions(id)
        ON DELETE CASCADE
);

CREATE INDEX data_source_dimensions_data_source_id_idx
    ON report_builder.data_source_dimensions (data_source_id);

-- ============================================================
-- 6. metric_toggles
-- ============================================================
CREATE TABLE report_builder.metric_toggles (
    id              BIGSERIAL PRIMARY KEY
                    CONSTRAINT metric_toggles_pk,
    toggle_key      VARCHAR(100) NOT NULL,
    display_name    VARCHAR(200) NOT NULL,
    description     TEXT,
    toggle_type     VARCHAR(50) NOT NULL DEFAULT 'metric_variant',
    ui_component    VARCHAR(50) NOT NULL DEFAULT 'pill_group',
    sort_order      INTEGER DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT metric_toggles_toggle_key_uq UNIQUE (toggle_key)
);

-- ============================================================
-- 7. metric_toggle_options
-- ============================================================
CREATE TABLE report_builder.metric_toggle_options (
    id              BIGSERIAL PRIMARY KEY
                    CONSTRAINT metric_toggle_options_pk,
    toggle_id       BIGINT NOT NULL,
    option_key      VARCHAR(100) NOT NULL,
    display_name    VARCHAR(200) NOT NULL,
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    sort_order      INTEGER DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT metric_toggle_options_uq
        UNIQUE (toggle_id, option_key),
    CONSTRAINT metric_toggle_options_metric_toggles_id_fk
        FOREIGN KEY (toggle_id) REFERENCES report_builder.metric_toggles(id)
        ON DELETE CASCADE
);

-- ============================================================
-- 8. metric_variants
-- ============================================================
CREATE TABLE report_builder.metric_variants (
    id                  BIGSERIAL PRIMARY KEY
                        CONSTRAINT metric_variants_pk,
    metric_id           BIGINT NOT NULL,
    toggle_option_id    BIGINT NOT NULL,
    sql_expression      TEXT NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT metric_variants_uq
        UNIQUE (metric_id, toggle_option_id),
    CONSTRAINT metric_variants_metrics_id_fk
        FOREIGN KEY (metric_id) REFERENCES report_builder.metrics(id)
        ON DELETE CASCADE,
    CONSTRAINT metric_variants_metric_toggle_options_id_fk
        FOREIGN KEY (toggle_option_id) REFERENCES report_builder.metric_toggle_options(id)
        ON DELETE CASCADE
);

CREATE INDEX metric_variants_metric_id_idx
    ON report_builder.metric_variants (metric_id);

-- ============================================================
-- 9. report_templates
-- ============================================================
CREATE TABLE report_builder.report_templates (
    id                      BIGSERIAL PRIMARY KEY
                            CONSTRAINT report_templates_pk,
    template_key            VARCHAR(100) NOT NULL,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,
    category                VARCHAR(100) NOT NULL,
    layout                  JSONB NOT NULL,
    default_filters         JSONB DEFAULT '{}',
    default_date_range      VARCHAR(50) DEFAULT 'last_30_days',
    required_data_sources   VARCHAR(100)[] NOT NULL,
    required_toggles        VARCHAR(100)[],
    nav_section             VARCHAR(100),
    nav_order               INTEGER DEFAULT 0,
    nav_icon                VARCHAR(100),
    is_active               BOOLEAN NOT NULL DEFAULT true,
    is_deleted              BOOLEAN NOT NULL DEFAULT false,
    created_at              TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by              VARCHAR,

    CONSTRAINT report_templates_template_key_uq UNIQUE (template_key)
);

CREATE INDEX report_templates_category_idx ON report_builder.report_templates (category);
CREATE INDEX report_templates_nav_section_idx ON report_builder.report_templates (nav_section);

-- ============================================================
-- 10. saved_views (Phase 2 — created empty)
-- ============================================================
CREATE TABLE report_builder.saved_views (
    id                  BIGSERIAL PRIMARY KEY
                        CONSTRAINT saved_views_pk,
    user_id             VARCHAR NOT NULL,
    template_id         BIGINT NOT NULL,
    client_id           BIGINT NOT NULL,
    name                VARCHAR(200) NOT NULL,
    filters             JSONB DEFAULT '{}',
    layout_overrides    JSONB DEFAULT '{}',
    toggles             JSONB DEFAULT '{}',
    is_default          BOOLEAN NOT NULL DEFAULT false,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    is_deleted          BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT saved_views_users_id_fk
        FOREIGN KEY (user_id) REFERENCES beast_insights_v2.users(id)
        ON DELETE CASCADE,
    CONSTRAINT saved_views_report_templates_id_fk
        FOREIGN KEY (template_id) REFERENCES report_builder.report_templates(id)
        ON DELETE CASCADE,
    CONSTRAINT saved_views_clients_id_fk
        FOREIGN KEY (client_id) REFERENCES beast_insights_v2.clients(id)
        ON DELETE CASCADE
);

CREATE INDEX saved_views_user_id_template_id_idx
    ON report_builder.saved_views (user_id, template_id);

-- ============================================================
-- 11. filter_presets (Phase 2 — created empty)
-- ============================================================
CREATE TABLE report_builder.filter_presets (
    id          BIGSERIAL PRIMARY KEY
                CONSTRAINT filter_presets_pk,
    user_id     VARCHAR NOT NULL,
    client_id   BIGINT NOT NULL,
    name        VARCHAR(200) NOT NULL,
    filters     JSONB NOT NULL,
    is_shared   BOOLEAN NOT NULL DEFAULT false,
    is_active   BOOLEAN NOT NULL DEFAULT true,
    is_deleted  BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT filter_presets_users_id_fk
        FOREIGN KEY (user_id) REFERENCES beast_insights_v2.users(id)
        ON DELETE CASCADE,
    CONSTRAINT filter_presets_clients_id_fk
        FOREIGN KEY (client_id) REFERENCES beast_insights_v2.clients(id)
        ON DELETE CASCADE
);

CREATE INDEX filter_presets_user_id_client_id_idx
    ON report_builder.filter_presets (user_id, client_id);

-- ============================================================
-- 12. dashboards (Phase 2 — created empty)
-- ============================================================
CREATE TABLE report_builder.dashboards (
    id          BIGSERIAL PRIMARY KEY
                CONSTRAINT dashboards_pk,
    user_id     VARCHAR NOT NULL,
    client_id   BIGINT NOT NULL,
    name        VARCHAR(200) NOT NULL,
    description TEXT,
    layout      JSONB DEFAULT '[]',
    is_default  BOOLEAN NOT NULL DEFAULT false,
    is_active   BOOLEAN NOT NULL DEFAULT true,
    is_deleted  BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT dashboards_users_id_fk
        FOREIGN KEY (user_id) REFERENCES beast_insights_v2.users(id)
        ON DELETE CASCADE,
    CONSTRAINT dashboards_clients_id_fk
        FOREIGN KEY (client_id) REFERENCES beast_insights_v2.clients(id)
        ON DELETE CASCADE
);

-- ============================================================
-- 13. dashboard_widgets (Phase 2 — created empty)
-- ============================================================
CREATE TABLE report_builder.dashboard_widgets (
    id                  BIGSERIAL PRIMARY KEY
                        CONSTRAINT dashboard_widgets_pk,
    dashboard_id        BIGINT NOT NULL,
    source_template_id  BIGINT,
    widget_type         VARCHAR(50) NOT NULL,
    widget_config       JSONB NOT NULL,
    position            JSONB NOT NULL,
    title               VARCHAR(200),
    sort_order          INTEGER DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT dashboard_widgets_dashboards_id_fk
        FOREIGN KEY (dashboard_id) REFERENCES report_builder.dashboards(id)
        ON DELETE CASCADE,
    CONSTRAINT dashboard_widgets_report_templates_id_fk
        FOREIGN KEY (source_template_id) REFERENCES report_builder.report_templates(id)
        ON DELETE SET NULL
);

-- ============================================================
-- 14. scheduled_reports (Phase 2 — created empty)
-- ============================================================
CREATE TABLE report_builder.scheduled_reports (
    id                  BIGSERIAL PRIMARY KEY
                        CONSTRAINT scheduled_reports_pk,
    user_id             VARCHAR NOT NULL,
    client_id           BIGINT NOT NULL,
    template_id         BIGINT NOT NULL,
    name                VARCHAR(200) NOT NULL,
    frequency           VARCHAR(50) NOT NULL,
    day_of_week         INTEGER,
    day_of_month        INTEGER,
    time_of_day         TIME NOT NULL DEFAULT '08:00',
    timezone            VARCHAR(100) NOT NULL DEFAULT 'America/New_York',
    delivery_channel    VARCHAR(50) NOT NULL DEFAULT 'email',
    delivery_target     VARCHAR(500),
    filters             JSONB DEFAULT '{}',
    export_format       VARCHAR(50) NOT NULL DEFAULT 'pdf',
    last_sent_at        TIMESTAMP(6),
    next_send_at        TIMESTAMP(6),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    is_deleted          BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT scheduled_reports_users_id_fk
        FOREIGN KEY (user_id) REFERENCES beast_insights_v2.users(id)
        ON DELETE CASCADE,
    CONSTRAINT scheduled_reports_clients_id_fk
        FOREIGN KEY (client_id) REFERENCES beast_insights_v2.clients(id)
        ON DELETE CASCADE,
    CONSTRAINT scheduled_reports_report_templates_id_fk
        FOREIGN KEY (template_id) REFERENCES report_builder.report_templates(id)
        ON DELETE CASCADE
);

CREATE INDEX scheduled_reports_next_send_at_idx
    ON report_builder.scheduled_reports (next_send_at);

-- ============================================================
-- 15. export_jobs (Phase 2 — created empty)
-- ============================================================
CREATE TABLE report_builder.export_jobs (
    id              BIGSERIAL PRIMARY KEY
                    CONSTRAINT export_jobs_pk,
    user_id         VARCHAR NOT NULL,
    client_id       BIGINT NOT NULL,
    template_id     BIGINT,
    format          VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    filters         JSONB,
    file_url        TEXT,
    file_size_bytes BIGINT,
    error_message   TEXT,
    completed_at    TIMESTAMP(6),
    expires_at      TIMESTAMP(6),
    created_at      TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT export_jobs_users_id_fk
        FOREIGN KEY (user_id) REFERENCES beast_insights_v2.users(id)
        ON DELETE CASCADE,
    CONSTRAINT export_jobs_clients_id_fk
        FOREIGN KEY (client_id) REFERENCES beast_insights_v2.clients(id)
        ON DELETE CASCADE,
    CONSTRAINT export_jobs_report_templates_id_fk
        FOREIGN KEY (template_id) REFERENCES report_builder.report_templates(id)
        ON DELETE SET NULL
);

CREATE INDEX export_jobs_user_id_status_idx
    ON report_builder.export_jobs (user_id, status);

-- ============================================================
-- 16. user_dimension_access (Phase 3 — created empty)
-- ============================================================
CREATE TABLE report_builder.user_dimension_access (
    id              BIGSERIAL PRIMARY KEY
                    CONSTRAINT user_dimension_access_pk,
    user_id         VARCHAR NOT NULL,
    client_id       BIGINT NOT NULL,
    dimension_id    BIGINT NOT NULL,
    allowed_values  JSONB NOT NULL,
    access_type     VARCHAR(50) NOT NULL DEFAULT 'include',
    created_by      VARCHAR,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT user_dimension_access_uq
        UNIQUE (user_id, client_id, dimension_id),
    CONSTRAINT user_dimension_access_users_id_fk
        FOREIGN KEY (user_id) REFERENCES beast_insights_v2.users(id)
        ON DELETE CASCADE,
    CONSTRAINT user_dimension_access_clients_id_fk
        FOREIGN KEY (client_id) REFERENCES beast_insights_v2.clients(id)
        ON DELETE CASCADE,
    CONSTRAINT user_dimension_access_dimensions_id_fk
        FOREIGN KEY (dimension_id) REFERENCES report_builder.dimensions(id)
        ON DELETE CASCADE
);

CREATE INDEX user_dimension_access_user_client_idx
    ON report_builder.user_dimension_access (user_id, client_id);

-- ============================================================
-- 17. dimension_access_audit (Phase 3 — created empty)
-- ============================================================
CREATE TABLE report_builder.dimension_access_audit (
    id                          BIGSERIAL PRIMARY KEY
                                CONSTRAINT dimension_access_audit_pk,
    user_dimension_access_id    BIGINT,
    target_user_id              VARCHAR NOT NULL,
    action                      VARCHAR(50) NOT NULL,
    performed_by                VARCHAR NOT NULL,
    old_values                  JSONB,
    new_values                  JSONB,
    created_at                  TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX dimension_access_audit_target_user_idx
    ON report_builder.dimension_access_audit (target_user_id);
CREATE INDEX dimension_access_audit_created_at_idx
    ON report_builder.dimension_access_audit (created_at);

-- ============================================================
-- 18. client_modules (Phase 4 — created empty)
-- ============================================================
CREATE TABLE report_builder.client_modules (
    id          BIGSERIAL PRIMARY KEY
                CONSTRAINT client_modules_pk,
    client_id   BIGINT NOT NULL,
    module_key  VARCHAR(100) NOT NULL,
    is_enabled  BOOLEAN NOT NULL DEFAULT false,
    enabled_at  TIMESTAMP(6),
    enabled_by  VARCHAR,
    config      JSONB DEFAULT '{}',
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT client_modules_uq UNIQUE (client_id, module_key),
    CONSTRAINT client_modules_clients_id_fk
        FOREIGN KEY (client_id) REFERENCES beast_insights_v2.clients(id)
        ON DELETE CASCADE
);

-- ============================================================
-- Existing table modifications
-- ============================================================
ALTER TABLE beast_insights_v2.clients
    ADD COLUMN IF NOT EXISTS report_mode VARCHAR(20) NOT NULL DEFAULT 'powerbi';

ALTER TABLE beast_insights_v2.client_features
    ADD COLUMN IF NOT EXISTS is_native_reports_enabled BOOLEAN NOT NULL DEFAULT false;
```

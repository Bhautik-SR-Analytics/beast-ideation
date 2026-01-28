# Milestone 1 — Database Design Document

> Complete schema design for the `report_builder` PostgreSQL schema — tables, columns, relationships, indexes, and ER diagrams.

> **Terminology:** This document uses Power BI terminology — visuals (not widgets), slicers (not toggles/filters), matrix (not pivot table), bookmarks (not saved views), tiles (not dashboard widgets), reports (not report templates).

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
| 1 | `data_sources` | Materialized view registry (dataset catalog) | ~10 |
| 2 | `metrics` | Metric catalog (SQL expressions, formats, types) | ~100 |
| 3 | `dimensions` | Dimension + filter catalog (columns, joins, filter metadata) | ~30 |
| 4 | `data_source_metrics` | Junction: which metrics work with which datasets | ~500 |
| 5 | `data_source_dimensions` | Junction: which dimensions work with which datasets | ~200 |
| 6 | `metric_toggles` | Toggle definitions (approval_mode, date_basis, etc.) | ~6 |
| 7 | `metric_toggle_options` | Options per toggle (standard, organic, net, etc.) | ~20 |
| 8 | `metric_variants` | SQL overrides per metric/toggle/option combination | ~50 |
| 9 | `report_templates` | JSON report layout definitions (3-layer: stock/client/custom) | ~11 |
| 10 | `report_template_filters` | Base filter assignments per report (which filters on which report) | ~200 |

### Phase 2 Tables (created empty, populated in Phase 2)

| # | Table | Purpose |
|---|-------|---------|
| 11 | `bookmarks` | Per-user report customizations |
| 12 | `filter_presets` | Named filter sets (shared via `filter_preset_shares`) |
| 13 | `dashboards` | Personal dashboard layouts |
| 14 | `dashboard_tiles` | Pinned visuals on dashboards |
| 15 | `scheduled_reports` | Scheduled email/Telegram delivery |
| 16 | `export_jobs` | CSV/Excel/PDF export tracking |
| 17 | `user_report_access` | Report visibility per user/role |
| 18 | `notification_rules` | Threshold-based alert rules |
| 19 | `notification_log` | Notification delivery history |
| 20 | `report_template_shares` | Share custom reports with specific users |
| 21 | `bookmark_shares` | Share bookmarks with specific users |
| 22 | `filter_preset_shares` | Share filter presets with specific users |
| 23 | `client_template_filter_overrides` | Client-level filter cascade overrides |
| 24 | `user_template_filter_overrides` | User-level filter cascade overrides |

### Phase 3 Tables (created empty, populated in Phase 3)

| # | Table | Purpose |
|---|-------|---------|
| 25 | `user_dimension_access` | User-level dimension value restrictions |
| 26 | `client_dimension_access` | Client-level dimension value restrictions |
| 27 | `dimension_access_audit` | Audit log for access changes |

### Phase 4 Tables (created empty, populated in Phase 4)

| # | Table | Purpose |
|---|-------|---------|
| 28 | `client_modules` | Per-client industry module enablement |

**Total: 28 new tables in the `report_builder` schema (no existing tables are modified)**

---

## 3. ER Diagram

### Phase 1 — Core Semantic Layer

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         REPORT + FILTER ASSIGNMENT LAYER                         │
│                                                                                  │
│  ┌─────────────────────┐        ┌──────────────────────────┐                     │
│  │  report_templates    │        │ report_template_filters   │                    │
│  ├─────────────────────┤        ├──────────────────────────┤                     │
│  │ id (PK)             │◄───────│ template_id (FK)         │                     │
│  │ template_key        │        │ dimension_id (FK) ───────┼──┐                  │
│  │ name                │        │ is_enabled               │  │                  │
│  │ category            │        │ display_order            │  │                  │
│  │ layout (JSONB)      │        │ filter_group_override    │  │                  │
│  │ template_type       │        │ default_value_override   │  │                  │
│  │ version             │        │ is_required              │  │                  │
│  │ client_id (FK)      │        └──────────────────────────┘  │                  │
│  │ user_id (FK)        │                                       │                  │
│  │ parent_template_id  │                                       │                  │
│  │ module_key          │                                       │                  │
│  └─────────────────────┘                                       │                  │
└──────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────┐
│                             SEMANTIC LAYER                                       │
│                                                                                  │
│  ┌──────────────────┐     ┌──────────────────────┐     ┌──────────────────┐      │
│  │  data_sources     │     │ data_source_metrics   │     │  metrics          │     │
│  ├──────────────────┤     ├──────────────────────┤     ├──────────────────┤      │
│  │ id (PK)          │◄────│ data_source_id (FK)  │────►│ id (PK)          │      │
│  │ source_key (UQ)  │     │ metric_id (FK)       │     │ metric_key (UQ)  │      │
│  │ display_name     │     │ sql_expression_      │     │ display_name     │      │
│  │ view_pattern     │     │   override           │     │ sql_expression   │      │
│  │ date_column      │     │ id (PK)              │     │ data_type        │      │
│  │ refresh_frequency│     └──────────────────────┘     │ format_pattern   │      │
│  └──────────────────┘                                   │ decimal_places   │      │
│           ▲               ┌──────────────────────┐     │ category         │      │
│           │               │data_source_dimensions│     │ aggregation_type │      │
│           └───────────────│ data_source_id (FK)  │     │ is_derived       │      │
│                           │ dimension_id (FK)    │     │ derived_formula  │      │
│                           │ id (PK)              │     │ supports_comparison│    │
│                           └──────────────────────┘     │ supports_trend   │      │
│                                      │                  └──────────────────┘      │
│                                      ▼                          ▲                 │
│                           ┌──────────────────────┐              │                 │
│                           │  dimensions           │◄────────────┘                 │
│                           ├──────────────────────┤   (from report_template_filters)│
│                           │ id (PK)              │                                │
│                           │ dimension_key (UQ)   │                                │
│                           │ display_name         │                                │
│                           │ column_expression    │                                │
│                           │ filter_column        │                                │
│                           │ filter_type          │ ← multi_select, boolean,       │
│                           │ is_multiple          │   nested, tree, input, etc.     │
│                           │ data_source          │ ← 'static' or 'dynamic'        │
│                           │ static_options(JSONB)│                                │
│                           │ parent_dimension_id  │ ← self-ref for nested/tree     │
│                           │ filter_group         │                                │
│                           │ min/max/step_value   │ ← for input type               │
│                           └──────────────────────┘                                │
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
│                         BOOKMARKS & FILTER PRESETS                                │
│                                                                                  │
│  ┌──────────────────┐     ┌──────────────────────┐                               │
│  │  bookmarks        │     │  filter_presets       │                              │
│  ├──────────────────┤     ├──────────────────────┤                               │
│  │ id (PK)          │     │ id (PK)              │                               │
│  │ user_id (FK)─────┼─┐   │ user_id (FK)─────────┼─┐                            │
│  │ template_id (FK) │ │   │ client_id (FK)       │ │                             │
│  │ name             │ │   │ name                 │ │                              │
│  │ filters (JSONB)  │ │   │ filters (JSONB)      │ │                              │
│  │ layout (JSONB)   │ │   └──────────────────────┘ │                              │
│  └──────────┬───────┘ │             │               │   ┌──────────────┐         │
│             │          │             │               │   │ users        │         │
│             ▼          │             ▼               │   │ (existing)   │         │
│  ┌──────────────────┐ │   ┌──────────────────────┐  │   │ beast_       │         │
│  │ bookmark_shares   │ │   │filter_preset_shares  │  └──►│ insights_v2  │        │
│  ├──────────────────┤ │   ├──────────────────────┤      └──────────────┘         │
│  │ id (PK)          │ │   │ id (PK)              │                                │
│  │ bookmark_id (FK) │ │   │ preset_id (FK)       │                                │
│  │ shared_with (FK) │ │   │ shared_with (FK)     │                                │
│  │ shared_by (FK)   │ │   │ shared_by (FK)       │                                │
│  │ client_id (FK)   │ │   │ client_id (FK)       │                                │
│  └──────────────────┘ │   └──────────────────────┘                                │
│                        │                                                           │
│  ┌──────────────────┐ │   ┌──────────────────────┐                               │
│  │  dashboards       │ │   │  dashboard_tiles     │                               │
│  ├──────────────────┤ │   ├──────────────────────┤                                │
│  │ id (PK)          │ │   │ id (PK)              │                                │
│  │ user_id (FK)─────┼─┤   │ dashboard_id (FK)    │                                │
│  │ client_id (FK)   │ │   │ visual_config (JSONB)│                                │
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
│  │ bookmark_id (FK) │     │ status               │                                │
│  │ frequency        │     │ toggles (JSONB)      │                                │
│  │ delivery_channels│     │ file_url             │                                │
│  │ toggles (JSONB)  │     └──────────────────────┘                                │
│  │ filters (JSONB)  │                                                             │
│  └──────────────────┘                                                             │
│                                                                                  │
│                   REPORT ACCESS & NOTIFICATIONS                                  │
│                                                                                  │
│  ┌─────────────────────┐  ┌──────────────────────┐   ┌──────────────────────┐    │
│  │ user_report_access   │  │  notification_rules   │   │  notification_log    │   │
│  ├─────────────────────┤  ├──────────────────────┤   ├──────────────────────┤    │
│  │ id (PK)             │  │ id (PK)              │──►│ id (PK)              │    │
│  │ user_id             │  │ user_id (FK)         │   │ rule_id (FK)         │    │
│  │ role_id             │  │ client_id (FK)       │   │ user_id              │    │
│  │ client_id (FK)      │  │ metric_key           │   │ triggered_at         │    │
│  │ template_id (FK)    │  │ condition            │   │ metric_value         │    │
│  │ granted_by          │  │ threshold_value      │   │ delivery_channel     │    │
│  │ is_active           │  │ frequency            │   │ delivery_status      │    │
│  └─────────────────────┘  │ delivery_channels[]  │   │ is_read              │    │
│                            └──────────────────────┘   └──────────────────────┘    │
│                                                                                  │
│  ┌──────────────────────┐                                                        │
│  │report_template_shares │                                                       │
│  ├──────────────────────┤                                                        │
│  │ id (PK)              │                                                        │
│  │ template_id (FK)     │                                                        │
│  │ shared_with (FK)     │                                                        │
│  │ shared_by (FK)       │                                                        │
│  │ permission           │                                                        │
│  └──────────────────────┘                                                        │
│                                                                                  │
│                   FILTER CASCADE OVERRIDES                                       │
│  (Overrides report_template_filters base config per client/user)                 │
│                                                                                  │
│  ┌──────────────────────────────┐  ┌──────────────────────────────┐              │
│  │client_template_filter_overrides│  │user_template_filter_overrides│             │
│  ├──────────────────────────────┤  ├──────────────────────────────┤              │
│  │ id (PK)                      │  │ id (PK)                      │              │
│  │ client_id (FK)               │  │ user_id (FK)                 │              │
│  │ template_id (FK)             │  │ template_id (FK)             │              │
│  │ dimension_id (FK)            │  │ dimension_id (FK)            │              │
│  │ is_enabled (nullable)        │  │ is_enabled (nullable)        │              │
│  │ display_order (nullable)     │  │ display_order (nullable)     │              │
│  │ default_value (nullable)     │  │ default_value (nullable)     │              │
│  └──────────────────────────────┘  └──────────────────────────────┘              │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Phase 3 — Access Control

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  ┌──────────────────────────┐      ┌──────────────────────────┐                  │
│  │ user_dimension_access     │      │ client_dimension_access   │                 │
│  ├──────────────────────────┤      ├──────────────────────────┤                  │
│  │ id (PK)                  │      │ id (PK)                  │                  │
│  │ user_id (FK) ────────────┼──┐   │ client_id (FK) ──────────┼──┐              │
│  │ dimension_id (FK)        │  │   │ dimension_id (FK)        │  │              │
│  │ allowed_values (JSONB)   │  │   │ allowed_values (JSONB)   │  │              │
│  │ access_type              │  │   │ access_type              │  │              │
│  │ client_id (FK)           │  │   │ created_by (FK)          │  │              │
│  │ created_by (FK)          │  │   └──────────────────────────┘  │              │
│  └──────────────────────────┘  │                                  │              │
│                                 │   ┌──────────────────────────┐  │              │
│         Cascade:                │   │ dimension_access_audit    │  │              │
│         client_dimension_access │   ├──────────────────────────┤  │              │
│         → all users of client   │   │ id (PK)                  │  │              │
│         user_dimension_access   │   │ target_user_id           │  │              │
│         → specific user         │   │ action                   │  │              │
│         (intersection = most    │   │ performed_by (FK)        │  │              │
│          restrictive)           │   │ old_values (JSONB)       │  │              │
│                                 │   │ new_values (JSONB)       │  │              │
│                                 │   └──────────────────────────┘  │              │
│                                 ▼                                  ▼              │
│                          ┌──────────────┐             ┌──────────────┐           │
│                          │ users        │             │ clients       │           │
│                          │ (existing)   │             │ (existing)    │           │
│                          └──────────────┘             └──────────────┘           │
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

Maps logical dataset names to their physical materialized view patterns in the `reporting` schema.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `source_key` | `VARCHAR(100)` | NO | — | Unique logical name (e.g., `order_summary`) |
| `display_name` | `VARCHAR(200)` | NO | — | Human-readable name (e.g., `Order Summary`) |
| `view_pattern` | `VARCHAR(500)` | NO | — | Pattern: `reporting.order_summary_{client_id}` |
| `date_column` | `VARCHAR(100)` | YES | `'date'` | Name of the date column used for date range filters |
| `client_id_column` | `VARCHAR(100)` | NO | `'client_id'` | Name of the client_id column |
| `refresh_frequency_minutes` | `INTEGER` | YES | `60` | How often the materialized view refreshes |
| `description` | `TEXT` | YES | — | What this dataset contains |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record update time |

**Constraints:**
- `PK`: `data_sources_pk` on `id`
- `UNIQUE`: `data_sources_source_key_uq` on `source_key`

**Existing materialized views this maps to (10 datasets):**

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
| `decimal_places` | `INTEGER` | NO | `0` | Number of decimal places for display |
| `category` | `VARCHAR(100)` | NO | — | Grouping: `sales`, `revenue`, `chargebacks`, `refunds`, `recovery`, `ltv`, `profitability`, `real_time`, `mid_health`, `alerts` |
| `sub_category` | `VARCHAR(100)` | YES | — | Sub-grouping within category (e.g., `initials`, `rebills` within `sales`) |
| `aggregation_type` | `VARCHAR(50)` | NO | `'sum'` | How to aggregate: `sum`, `count`, `avg`, `ratio`, `custom` |
| `is_derived` | `BOOLEAN` | NO | `false` | True if this metric is calculated from other metrics (not a direct column) |
| `derived_formula` | `TEXT` | YES | — | Formula using other metric_keys: e.g., `approval_rate = approvals / attempts * 100` |
| `supports_comparison` | `BOOLEAN` | NO | `true` | Whether this metric supports "compare to previous period" |
| `supports_trend` | `BOOLEAN` | NO | `true` | Whether this metric shows trend arrow on cards |
| `is_invertable` | `BOOLEAN` | NO | `false` | Whether higher = worse (e.g., CB rate — higher is bad) |
| `sort_order` | `INTEGER` | YES | `0` | Display ordering within category |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record update time |

**Constraints:**
- `PK`: `metrics_pk` on `id`
- `UNIQUE`: `metrics_metric_key_uq` on `metric_key`
- `INDEX`: `metrics_category_idx` on `category`
- `INDEX`: `metrics_sub_category_idx` on `sub_category`

**Note:** This table is different from the existing `beast_insights_v2.metrics` table which only has `id`, `name`, `is_active`, `is_deleted`, `created_at`, `updated_at`, `is_display`, `powerbi_metric`, `data_type`. The existing table is used for alerts/goals (6 columns). The new `report_builder.metrics` table is the full semantic layer catalog (20 columns) with SQL expressions, formats, and derivation formulas.

---

### 4.3 `dimensions`

Every dimension users can group by or filter on. Also serves as the filter definition catalog (equivalent of `beast_insights_v2.filter_definitions` in the existing system).

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `dimension_key` | `VARCHAR(100)` | NO | — | Unique identifier (e.g., `campaign_name`, `product_name`, `gateway_alias`) |
| `display_name` | `VARCHAR(200)` | NO | — | UI label (e.g., `Campaign`, `Product`, `Gateway`) |
| `description` | `TEXT` | YES | — | What this dimension represents |
| `column_expression` | `TEXT` | NO | — | SQL column or expression (e.g., `campaign_id`, `TO_CHAR(date, 'YYYY-MM')`) |
| `filter_column` | `VARCHAR(200)` | YES | — | Column used in WHERE clause if different from column_expression (e.g., filter on `campaign_id`, display `campaign_name`) |
| `display_column` | `VARCHAR(200)` | YES | — | Column for display value if different from filter value (e.g., filter by `campaign_id`, display `campaign_name`) |
| `join_table` | `VARCHAR(200)` | YES | — | Table to JOIN for display name lookup (e.g., `public.campaigns`) |
| `join_condition` | `TEXT` | YES | — | JOIN ON clause (e.g., `{source}.campaign_id = campaigns.campaign_id AND campaigns.client_id = {client_id}`) |
| `data_type` | `VARCHAR(50)` | NO | `'text'` | `text`, `integer`, `date`, `boolean` |
| `filter_type` | `VARCHAR(50)` | NO | `'multi_select'` | How this appears in the slicer panel (see filter types below) |
| `is_groupable` | `BOOLEAN` | NO | `true` | Can this be used in GROUP BY |
| `is_filterable` | `BOOLEAN` | NO | `true` | Can this be used as a WHERE filter |
| `category` | `VARCHAR(100)` | YES | — | Grouping for slicer panel UI (e.g., `transaction`, `marketing`, `payment`, `time`) |
| `sort_order` | `INTEGER` | YES | `0` | Display ordering |
| `is_multiple` | `BOOLEAN` | NO | `true` | Multi-select vs single-select (from `filter_definitions.is_multiple`) |
| `data_source` | `VARCHAR(50)` | NO | `'dynamic'` | `'static'` (pre-stored options) or `'dynamic'` (SQL query at runtime) |
| `options_query` | `TEXT` | YES | — | Custom SQL to fetch filter options for dynamic data sources (from `filter_definitions.query`) |
| `static_options` | `JSONB` | YES | — | Pre-stored options for static filters — array of `{label, value}` (from `filter_options.options`) |
| `default_value` | `JSONB` | YES | — | Default selected value(s) when filter first loads |
| `filter_group` | `VARCHAR(100)` | YES | — | UI group: `'main_filters'`, `'more_filters'`, `'input_filters'` (from `filter_definitions.filter_group`) |
| `icon` | `VARCHAR(100)` | YES | — | Icon identifier for slicer panel display (from `filter_definitions.icon`) |
| `parent_dimension_id` | `BIGINT` | YES | — | FK → `dimensions.id` (self-ref). For hierarchical filters — e.g., Billing Cycle's parent is Sales Type |
| `min_value` | `DECIMAL(18,4)` | YES | — | For `input` type: minimum allowed value |
| `max_value` | `DECIMAL(18,4)` | YES | — | For `input` type: maximum allowed value |
| `step_value` | `DECIMAL(18,4)` | YES | — | For `input` type: step increment |
| `input_prefix` | `VARCHAR(20)` | YES | — | For `input` type: display prefix (`$`, `%`) |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record update time |

**Constraints:**
- `PK`: `dimensions_pk` on `id`
- `UNIQUE`: `dimensions_dimension_key_uq` on `dimension_key`
- `FK`: `dimensions_parent_dimension_id_fk` → `dimensions.id` ON DELETE SET NULL
- `CHECK`: `dimensions_filter_type_check` — `filter_type IN ('multi_select', 'single_select', 'date_range', 'search', 'boolean', 'nested', 'tree', 'input')`
- `CHECK`: `dimensions_data_source_check` — `data_source IN ('static', 'dynamic')`
- `INDEX`: `dimensions_category_idx` on `category`
- `INDEX`: `dimensions_filter_group_idx` on `filter_group`
- `INDEX`: `dimensions_parent_dimension_id_idx` on `parent_dimension_id`

**Filter types (matches existing `filter_type_enum` from `beast_insights_v2`):**

| `filter_type` | UI Pattern | Example | Existing Equivalent |
|---------------|-----------|---------|---------------------|
| `multi_select` | Dropdown with checkboxes | Campaigns, Products, Gateways | `basic` (is_multiple=true) |
| `single_select` | Dropdown, one selection | Country, Card Type | `basic` (is_multiple=false) |
| `date_range` | Date range picker with presets | Date Range | `advanced` |
| `search` | Text search input | Order search | — |
| `boolean` | Toggle switch (include/exclude) | Include Trials | `boolean` |
| `nested` | Parent-child hierarchical checkboxes | Sales Type → Billing Cycle | `nested` |
| `tree` | 3-pane sequential selection (A→B→C) | Parameter grouping | `tree` |
| `input` | Numeric min/max range with step | CPA input, Reserve Rate | `input` |

---

### 4.4 `data_source_metrics`

Junction table: which metrics are valid for which datasets.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `data_source_id` | `BIGINT` | NO | — | FK → `data_sources.id` |
| `metric_id` | `BIGINT` | NO | — | FK → `metrics.id` |
| `sql_expression_override` | `TEXT` | YES | — | Per-source SQL override (for Phase 4 modules using same metric key with different SQL) |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |

**Constraints:**
- `PK`: `data_source_metrics_pk` on `id`
- `UNIQUE`: `data_source_metrics_uq` on `(data_source_id, metric_id)`
- `FK`: `data_source_metrics_data_sources_id_fk` → `data_sources.id` ON DELETE CASCADE
- `FK`: `data_source_metrics_metrics_id_fk` → `metrics.id` ON DELETE CASCADE

---

### 4.5 `data_source_dimensions`

Junction table: which dimensions can be used with which datasets.

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
| `ui_component` | `VARCHAR(50)` | NO | `'pill_group'` | How rendered in Slicer Panel: `pill_group`, `dropdown`, `toggle_switch` |
| `sort_order` | `INTEGER` | YES | `0` | Display order in slicer panel |
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

JSON report layout definitions. Each report is stored as a single JSONB blob. Supports a **three-layer template system**:

| Layer | `template_type` | `client_id` | `user_id` | Visibility |
|-------|----------------|-------------|-----------|------------|
| Common | `stock` | NULL | NULL | All clients, all users |
| Client | `client` | SET | NULL | All users of that client |
| User | `custom` | SET | SET | Only that user (until shared via `report_template_shares`) |
| Module | `module` | NULL | NULL | Clients with module enabled (Phase 4) |

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `template_key` | `VARCHAR(100)` | NO | — | URL slug (e.g., `revenue-analytics`, `command-center`) |
| `name` | `VARCHAR(200)` | NO | — | Display name (e.g., `Revenue Analytics`, `Command Center`) |
| `description` | `TEXT` | YES | — | Report description |
| `category` | `VARCHAR(100)` | NO | — | Report grouping: `operations`, `analytics`, `financial`, `lifecycle`, `risk` |
| `layout` | `JSONB` | NO | — | Full report layout JSON (sections, visuals, configs) |
| `default_filters` | `JSONB` | YES | `'{}'` | Default slicer values (e.g., `{"dateRange": "last_30_days"}`) |
| `default_date_range` | `VARCHAR(50)` | YES | `'last_30_days'` | Default date range preset |
| `required_data_sources` | `VARCHAR[]` | NO | — | Array of `source_key` values this report needs |
| `required_toggles` | `VARCHAR[]` | YES | — | Array of `toggle_key` values used in this report |
| `nav_section` | `VARCHAR(100)` | YES | — | Sidebar section (e.g., `operations`, `analytics`) |
| `nav_order` | `INTEGER` | YES | `0` | Order within nav section |
| `nav_icon` | `VARCHAR(100)` | YES | — | Icon name for sidebar |
| `template_type` | `VARCHAR(50)` | NO | `'stock'` | `stock`, `client`, `custom`, `module` (see layer table above) |
| `module_key` | `VARCHAR(100)` | YES | — | Module identifier for Phase 4 reports (e.g., `high_risk`, `telehealth`) |
| `version` | `INTEGER` | NO | `1` | Incremented on layout changes; bookmarks can detect stale visual references |
| `client_id` | `BIGINT` | YES | — | FK → `beast_insights_v2.clients.id` (NULL for stock/module reports) |
| `user_id` | `VARCHAR` | YES | — | FK → `beast_insights_v2.users.id` (NULL for stock/client/module; set for user-owned custom templates) |
| `parent_template_id` | `BIGINT` | YES | — | FK → `report_templates.id` ON DELETE SET NULL (tracks duplication lineage when user duplicates a shared template) |
| `created_by` | `VARCHAR` | YES | — | FK → `beast_insights_v2.users.id` (audit: who created it) |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `is_deleted` | `BOOLEAN` | NO | `false` | Soft-delete flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record update time |

**Constraints:**
- `PK`: `report_templates_pk` on `id`
- `FK`: `report_templates_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `FK`: `report_templates_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `report_templates_parent_id_fk` → `report_templates.id` ON DELETE SET NULL
- `CHECK`: `report_templates_type_check` — `template_type IN ('stock', 'client', 'custom', 'module')`
- `PARTIAL UNIQUE INDEX`: `report_templates_stock_key_uq` on `(template_key)` WHERE `template_type = 'stock'`
- `PARTIAL UNIQUE INDEX`: `report_templates_client_key_uq` on `(template_key, client_id)` WHERE `template_type = 'client' AND client_id IS NOT NULL`
- `PARTIAL UNIQUE INDEX`: `report_templates_custom_key_uq` on `(template_key, client_id, user_id)` WHERE `template_type = 'custom' AND user_id IS NOT NULL`
- `PARTIAL UNIQUE INDEX`: `report_templates_module_key_uq` on `(template_key, module_key)` WHERE `template_type = 'module'`
- `INDEX`: `report_templates_category_idx` on `category`
- `INDEX`: `report_templates_nav_section_idx` on `nav_section`
- `INDEX`: `report_templates_client_id_idx` on `client_id`
- `INDEX`: `report_templates_user_id_idx` on `user_id`
- `INDEX`: `report_templates_module_key_idx` on `module_key`
- `INDEX`: `report_templates_type_client_idx` on `(template_type, client_id)`

**11 stock reports to seed:**

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

### 4.10 `report_template_filters`

Base-level filter assignments: which dimensions appear as filters on each report template, in what order, with what defaults. This is the equivalent of `beast_insights_v2.base_filter_assignments` + `page_filter_assignments` combined (since "page" = "report template" in the new system).

This is the **BASE level** of the 3-level filter cascade:
1. `report_template_filters` → base config (Phase 1)
2. `client_template_filter_overrides` → client admin overrides (Phase 2)
3. `user_template_filter_overrides` → user personal overrides (Phase 2)

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `template_id` | `BIGINT` | NO | — | FK → `report_templates.id` |
| `dimension_id` | `BIGINT` | NO | — | FK → `dimensions.id` |
| `is_enabled` | `BOOLEAN` | NO | `true` | Whether this filter is visible on this report |
| `display_order` | `INTEGER` | YES | `0` | Order in slicer panel |
| `filter_group_override` | `VARCHAR(100)` | YES | — | Override the dimension's default group for this report (NULL = inherit from dimension) |
| `default_value_override` | `JSONB` | YES | — | Override default value for this report (NULL = inherit from dimension) |
| `is_required` | `BOOLEAN` | NO | `false` | Must have a selection before querying |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_by` | `VARCHAR` | YES | — | FK → `beast_insights_v2.users.id` |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record creation time |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | Record update time |

**Constraints:**
- `PK`: `report_template_filters_pk` on `id`
- `UNIQUE`: `report_template_filters_uq` on `(template_id, dimension_id)`
- `FK`: `report_template_filters_report_templates_id_fk` → `report_templates.id` ON DELETE CASCADE
- `FK`: `report_template_filters_dimensions_id_fk` → `dimensions.id` ON DELETE CASCADE
- `INDEX`: `report_template_filters_template_id_idx` on `(template_id, display_order)`

**Seeded during M5** with ~200 rows (11 reports × ~18 filters each).

---

## 5. Table Definitions — Phase 2 (Customization)

> All Phase 2 tables are created empty during Milestone 1. No data, no APIs — just the schema.

### 5.1 `bookmarks`

Per-user saved customizations of a stock report (saved report state).

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `template_id` | `BIGINT` | NO | — | FK → `report_templates.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `name` | `VARCHAR(200)` | NO | — | User-given name (e.g., `My Q4 View`) |
| `filters` | `JSONB` | YES | `'{}'` | Saved slicer state |
| `layout_overrides` | `JSONB` | YES | `'{}'` | Visual visibility/order overrides |
| `toggles` | `JSONB` | YES | `'{}'` | Saved toggle state |
| `is_default` | `BOOLEAN` | NO | `false` | Auto-load this bookmark when opening the report |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `is_deleted` | `BOOLEAN` | NO | `false` | Soft-delete flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `bookmarks_pk` on `id`
- `FK`: `bookmarks_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `bookmarks_report_templates_id_fk` → `report_templates.id` ON DELETE CASCADE
- `FK`: `bookmarks_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `INDEX`: `bookmarks_user_id_template_id_idx` on `(user_id, template_id)`

---

### 5.2 `filter_presets`

Named filter combinations. Sharing is handled via the `filter_preset_shares` junction table for per-user sharing.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `name` | `VARCHAR(200)` | NO | — | Preset name (e.g., `Last 90 Days - Top Campaigns`) |
| `filters` | `JSONB` | NO | — | Filter state JSON |
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

### 5.4 `dashboard_tiles`

Pinned visuals on a personal dashboard.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `dashboard_id` | `BIGINT` | NO | — | FK → `dashboards.id` |
| `source_template_id` | `BIGINT` | YES | — | FK → `report_templates.id` (which report this visual came from) |
| `visual_type` | `VARCHAR(50)` | NO | — | `card`, `chart`, `table`, etc. |
| `visual_config` | `JSONB` | NO | — | Complete visual configuration (metrics, dimensions, slicers, chart type) |
| `position` | `JSONB` | NO | — | `{ x, y, w, h }` for react-grid-layout |
| `title` | `VARCHAR(200)` | YES | — | Custom title override |
| `sort_order` | `INTEGER` | YES | `0` | |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `dashboard_tiles_pk` on `id`
- `FK`: `dashboard_tiles_dashboards_id_fk` → `dashboards.id` ON DELETE CASCADE
- `FK`: `dashboard_tiles_report_templates_id_fk` → `report_templates.id` ON DELETE SET NULL

---

### 5.5 `scheduled_reports`

Scheduled email/Telegram report delivery.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `template_id` | `BIGINT` | NO | — | FK → `report_templates.id` |
| `bookmark_id` | `BIGINT` | YES | — | FK → `bookmarks.id` (optionally schedule a specific bookmark) |
| `name` | `VARCHAR(200)` | NO | — | Schedule name |
| `frequency` | `VARCHAR(50)` | NO | — | `daily`, `weekly`, `monthly` |
| `day_of_week` | `INTEGER` | YES | — | 0-6 for weekly schedules |
| `day_of_month` | `INTEGER` | YES | — | 1-31 for monthly schedules |
| `time_of_day` | `TIME` | NO | `'08:00'` | When to send |
| `timezone` | `VARCHAR(100)` | NO | `'America/New_York'` | User's timezone |
| `delivery_channels` | `VARCHAR(50)[] NOT NULL` | NO | `'{email}'` | Array of delivery channels: `email`, `telegram`, `slack` |
| `delivery_target` | `VARCHAR(500)` | YES | — | Email address or chat ID |
| `filters` | `JSONB` | YES | `'{}'` | Locked slicer state for this schedule |
| `toggles` | `JSONB` | YES | `'{}'` | Toggle state to apply when generating |
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
- `FK`: `scheduled_reports_bookmarks_id_fk` → `bookmarks.id` ON DELETE SET NULL
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
| `filters` | `JSONB` | YES | — | Slicer state at time of export |
| `toggles` | `JSONB` | YES | — | Toggle state at time of export |
| `file_url` | `TEXT` | YES | — | S3/storage URL once generated |
| `file_size_bytes` | `BIGINT` | YES | — | |
| `error_message` | `TEXT` | YES | — | Error detail if failed |
| `completed_at` | `TIMESTAMP(6)` | YES | — | |
| `expires_at` | `TIMESTAMP(6)` | YES | — | Auto-delete exported file after |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `export_jobs_pk` on `id`
- `FK`: `export_jobs_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `export_jobs_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `FK`: `export_jobs_report_templates_id_fk` → `report_templates.id` ON DELETE SET NULL
- `INDEX`: `export_jobs_user_id_status_idx` on `(user_id, status)`

---

### 5.7 `user_report_access`

Report visibility per user/role. Controls which reports each user or role can access. If no rows exist for a user/client, they see all reports (default open). When rows exist, the user only sees listed reports. SUPER_ADMIN and ADMIN bypass this check.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | YES | — | FK → `beast_insights_v2.users.id` (nullable — set when granting to a specific user) |
| `role_id` | `BIGINT` | YES | — | Role ID (nullable — set when granting to a role) |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `template_id` | `BIGINT` | NO | — | FK → `report_templates.id` |
| `granted_by` | `VARCHAR` | YES | — | FK → `beast_insights_v2.users.id` (admin who granted access) |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `user_report_access_pk` on `id`
- `FK`: `user_report_access_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `FK`: `user_report_access_report_templates_id_fk` → `report_templates.id` ON DELETE CASCADE
- `CHECK`: `user_report_access_target_check` — at least one of `user_id` or `role_id` must be set
- `PARTIAL UNIQUE INDEX`: `user_report_access_user_uq` on `(user_id, client_id, template_id)` WHERE `user_id IS NOT NULL`
- `PARTIAL UNIQUE INDEX`: `user_report_access_role_uq` on `(role_id, client_id, template_id)` WHERE `role_id IS NOT NULL`
- `INDEX`: `user_report_access_user_id_idx` on `user_id`
- `INDEX`: `user_report_access_role_id_idx` on `role_id`
- `INDEX`: `user_report_access_client_template_idx` on `(client_id, template_id)`

---

### 5.8 `notification_rules`

Threshold-based alert rules. Example: "Alert me when CB Rate > 1% for any gateway." Evaluated on schedule (hourly, daily, or on data refresh).

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `name` | `VARCHAR(200)` | NO | — | Rule name (e.g., `High CB Rate Alert`) |
| `description` | `TEXT` | YES | — | Human-readable description |
| `metric_id` | `BIGINT` | NO | — | FK → `metrics.id` |
| `dimension_id` | `BIGINT` | YES | — | FK → `dimensions.id` (optional: evaluate per dimension value) |
| `data_source_id` | `BIGINT` | NO | — | FK → `data_sources.id` |
| `condition_operator` | `VARCHAR(20)` | NO | — | `>`, `<`, `>=`, `<=`, `=`, `!=`, `crosses_above`, `crosses_below` |
| `threshold_value` | `DECIMAL(18,4)` | NO | — | Threshold to compare against |
| `frequency` | `VARCHAR(50)` | NO | `'daily'` | Evaluation frequency: `hourly`, `daily`, `on_refresh` |
| `delivery_channels` | `VARCHAR(50)[] NOT NULL` | NO | `'{email}'` | Array of channels: `email`, `in_app`, `telegram`, `slack` |
| `delivery_config` | `JSONB` | YES | `'{}'` | Channel-specific config (e.g., email recipients, chat IDs) |
| `last_triggered_at` | `TIMESTAMP(6)` | YES | — | Last time this rule fired |
| `last_evaluated_at` | `TIMESTAMP(6)` | YES | — | Last time this rule was checked |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `is_deleted` | `BOOLEAN` | NO | `false` | Soft-delete flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `notification_rules_pk` on `id`
- `FK`: `notification_rules_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `notification_rules_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `FK`: `notification_rules_metrics_id_fk` → `metrics.id` ON DELETE CASCADE
- `FK`: `notification_rules_dimensions_id_fk` → `dimensions.id` ON DELETE SET NULL
- `FK`: `notification_rules_data_sources_id_fk` → `data_sources.id` ON DELETE CASCADE
- `CHECK`: `notification_rules_operator_check` — `condition_operator IN ('>', '<', '>=', '<=', '=', '!=', 'crosses_above', 'crosses_below')`
- `INDEX`: `notification_rules_user_client_idx` on `(user_id, client_id)`
- `INDEX`: `notification_rules_frequency_idx` on `frequency` WHERE `is_active = true AND is_deleted = false`

---

### 5.9 `notification_log`

Notification delivery history. Each row represents one delivery attempt on one channel. For in-app notifications, `is_read` / `read_at` tracks whether the user has seen it.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `rule_id` | `BIGINT` | NO | — | FK → `notification_rules.id` |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `triggered_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | When the rule fired |
| `metric_value` | `DECIMAL(18,4)` | YES | — | Actual metric value that triggered the rule |
| `dimension_value` | `VARCHAR(500)` | YES | — | Dimension value if rule is per-dimension |
| `delivery_channel` | `VARCHAR(50)` | NO | — | Channel used: `email`, `in_app`, `telegram`, `slack` |
| `delivery_status` | `VARCHAR(50)` | NO | `'pending'` | `pending`, `sent`, `failed` |
| `is_read` | `BOOLEAN` | NO | `false` | For in-app: has the user seen this notification |
| `read_at` | `TIMESTAMP(6)` | YES | — | When the user marked it as read |
| `error_message` | `TEXT` | YES | — | Error detail if delivery failed |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `notification_log_pk` on `id`
- `FK`: `notification_log_rules_id_fk` → `notification_rules.id` ON DELETE CASCADE
- `INDEX`: `notification_log_rule_id_idx` on `rule_id`
- `INDEX`: `notification_log_triggered_at_idx` on `triggered_at`
- `PARTIAL INDEX`: `notification_log_user_unread_idx` on `(user_id, is_read)` WHERE `is_read = false`

---

### 5.10 `report_template_shares`

Share custom reports with specific users. The owner (`report_templates.created_by`) has full control. Shared users get the specified permission level.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `template_id` | `BIGINT` | NO | — | FK → `report_templates.id` |
| `shared_with_user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` (recipient) |
| `shared_by_user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` (sharer) |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `permission` | `VARCHAR(50)` | NO | `'view'` | Permission level: `view`, `edit`, `duplicate` |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `report_template_shares_pk` on `id`
- `FK`: `report_template_shares_report_templates_id_fk` → `report_templates.id` ON DELETE CASCADE
- `FK`: `report_template_shares_shared_with_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `report_template_shares_shared_by_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `report_template_shares_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `CHECK`: `report_template_shares_permission_check` — `permission IN ('view', 'edit', 'duplicate')`
- `UNIQUE INDEX`: `report_template_shares_uq` on `(template_id, shared_with_user_id)`
- `INDEX`: `report_template_shares_shared_with_idx` on `shared_with_user_id`

---

### 5.11 `bookmark_shares`

Share bookmarks with specific users. Shared users can apply the bookmark to see the same saved view but cannot edit or delete the original.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `bookmark_id` | `BIGINT` | NO | — | FK → `bookmarks.id` |
| `shared_with_user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` (recipient) |
| `shared_by_user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` (sharer) |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `bookmark_shares_pk` on `id`
- `FK`: `bookmark_shares_bookmarks_id_fk` → `bookmarks.id` ON DELETE CASCADE
- `FK`: `bookmark_shares_shared_with_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `bookmark_shares_shared_by_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `bookmark_shares_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `UNIQUE INDEX`: `bookmark_shares_uq` on `(bookmark_id, shared_with_user_id)`
- `INDEX`: `bookmark_shares_shared_with_idx` on `shared_with_user_id`

---

### 5.12 `filter_preset_shares`

Share filter presets with specific users. Shared users can apply the preset to any report but cannot edit or delete the original.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `preset_id` | `BIGINT` | NO | — | FK → `filter_presets.id` |
| `shared_with_user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` (recipient) |
| `shared_by_user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` (sharer) |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `is_active` | `BOOLEAN` | NO | `true` | |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `filter_preset_shares_pk` on `id`
- `FK`: `filter_preset_shares_filter_presets_id_fk` → `filter_presets.id` ON DELETE CASCADE
- `FK`: `filter_preset_shares_shared_with_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `filter_preset_shares_shared_by_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `filter_preset_shares_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `UNIQUE INDEX`: `filter_preset_shares_uq` on `(preset_id, shared_with_user_id)`
- `INDEX`: `filter_preset_shares_shared_with_idx` on `shared_with_user_id`

---

### 5.13 `client_template_filter_overrides`

Per-client overrides for template filters. A client admin can hide a filter from a report, reorder filters, or set different defaults for their organization. Equivalent of `beast_insights_v2.client_filter_assignments`.

**NULL values mean "inherit from base level"** (`report_template_filters`). Explicit values override the base.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `template_id` | `BIGINT` | NO | — | FK → `report_templates.id` |
| `dimension_id` | `BIGINT` | NO | — | FK → `dimensions.id` |
| `is_enabled` | `BOOLEAN` | YES | — | Override: NULL = inherit base, true/false = explicit |
| `display_order` | `INTEGER` | YES | — | Override: NULL = inherit base |
| `default_value` | `JSONB` | YES | — | Override: NULL = inherit base |
| `created_by` | `VARCHAR` | YES | — | FK → `beast_insights_v2.users.id` (admin who set this) |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `client_template_filter_overrides_pk` on `id`
- `UNIQUE`: `client_template_filter_overrides_uq` on `(client_id, template_id, dimension_id)`
- `FK`: `client_template_filter_overrides_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `FK`: `client_template_filter_overrides_templates_id_fk` → `report_templates.id` ON DELETE CASCADE
- `FK`: `client_template_filter_overrides_dimensions_id_fk` → `dimensions.id` ON DELETE CASCADE
- `INDEX`: `client_template_filter_overrides_client_template_idx` on `(client_id, template_id)`

---

### 5.14 `user_template_filter_overrides`

Per-user overrides for template filters — **highest priority** in the 3-level cascade. A user can personalize which filters they see on each report. Equivalent of `beast_insights_v2.user_filter_assignments`.

**NULL values mean "inherit from client level** (or base if no client override). Explicit values override all parent levels.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `user_id` | `VARCHAR` | NO | — | FK → `beast_insights_v2.users.id` |
| `template_id` | `BIGINT` | NO | — | FK → `report_templates.id` |
| `dimension_id` | `BIGINT` | NO | — | FK → `dimensions.id` |
| `is_enabled` | `BOOLEAN` | YES | — | Override: NULL = inherit, true/false = explicit |
| `display_order` | `INTEGER` | YES | — | Override: NULL = inherit |
| `default_value` | `JSONB` | YES | — | Override: NULL = inherit |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `user_template_filter_overrides_pk` on `id`
- `UNIQUE`: `user_template_filter_overrides_uq` on `(user_id, template_id, dimension_id)`
- `FK`: `user_template_filter_overrides_users_id_fk` → `beast_insights_v2.users.id` ON DELETE CASCADE
- `FK`: `user_template_filter_overrides_templates_id_fk` → `report_templates.id` ON DELETE CASCADE
- `FK`: `user_template_filter_overrides_dimensions_id_fk` → `dimensions.id` ON DELETE CASCADE
- `INDEX`: `user_template_filter_overrides_user_template_idx` on `(user_id, template_id)`

**Filter cascade resolution at query time:**

```
1. Load report_template_filters WHERE template_id = $1
   → Base: which filters, what order, what defaults

2. Load client_template_filter_overrides WHERE client_id = $2 AND template_id = $1
   → Merge: non-NULL values override base

3. Load user_template_filter_overrides WHERE user_id = $3 AND template_id = $1
   → Merge: non-NULL values override client/base

Result: final filter config for this user + client + report
```

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

### 6.2 `client_dimension_access`

Client-level dimension value restrictions. All users of a client are restricted to these values. Works in conjunction with `user_dimension_access`:

- `client_dimension_access` → applies to ALL users of that client
- `user_dimension_access` → applies to a SPECIFIC user (more restrictive)

If both exist for the same dimension, the query engine takes the **intersection** (most restrictive set of values). These same restrictions also limit filter dropdown options — the `/api/v1/filters/options/:dimensionKey` endpoint applies these before returning available options.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | `BIGINT` | NO | `autoincrement` | Primary key |
| `client_id` | `BIGINT` | NO | — | FK → `beast_insights_v2.clients.id` |
| `dimension_id` | `BIGINT` | NO | — | FK → `dimensions.id` |
| `allowed_values` | `JSONB` | NO | — | Array of permitted values |
| `access_type` | `VARCHAR(50)` | NO | `'include'` | `include` (only these) or `exclude` (everything except) |
| `created_by` | `VARCHAR` | YES | — | FK → `beast_insights_v2.users.id` |
| `is_active` | `BOOLEAN` | NO | `true` | Soft-active flag |
| `created_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |
| `updated_at` | `TIMESTAMP(6)` | NO | `CURRENT_TIMESTAMP` | |

**Constraints:**
- `PK`: `client_dimension_access_pk` on `id`
- `UNIQUE`: `client_dimension_access_uq` on `(client_id, dimension_id)`
- `FK`: `client_dimension_access_clients_id_fk` → `beast_insights_v2.clients.id` ON DELETE CASCADE
- `FK`: `client_dimension_access_dimensions_id_fk` → `dimensions.id` ON DELETE CASCADE
- `CHECK`: `client_dimension_access_type_check` — `access_type IN ('include', 'exclude')`
- `INDEX`: `client_dimension_access_client_id_idx` on `client_id`

**Dimension access cascade at query time:**

```
1. client_dimension_access → client-level restrictions (all users)
2. user_dimension_access   → user-level restrictions (further narrows)
   → Result: WHERE clause injected into every query
```

---

### 6.3 `dimension_access_audit`

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

No existing tables in `beast_insights_v2` are modified. The `report_builder` schema is entirely self-contained — it references existing tables via foreign keys but does not add columns or alter any existing table structure.

---

## 9. Index Strategy

### Performance-Critical Indexes

| Table | Index Name | Columns | Reason |
|-------|-----------|---------|--------|
| `metrics` | `metrics_category_idx` | `category` | Filter metrics by category in UI |
| `metrics` | `metrics_metric_key_uq` | `metric_key` UNIQUE | Lookup by key (query engine) |
| `metrics` | `metrics_sub_category_idx` | `sub_category` | Filter metrics by sub-category |
| `dimensions` | `dimensions_category_idx` | `category` | Filter dimensions by category |
| `dimensions` | `dimensions_dimension_key_uq` | `dimension_key` UNIQUE | Lookup by key (query engine) |
| `dimensions` | `dimensions_filter_group_idx` | `filter_group` | Filter dimensions by UI group |
| `dimensions` | `dimensions_parent_dimension_id_idx` | `parent_dimension_id` | Hierarchical filter lookups |
| `data_sources` | `data_sources_source_key_uq` | `source_key` UNIQUE | Resolve dataset by key |
| `data_source_metrics` | `data_source_metrics_uq` | `(data_source_id, metric_id)` UNIQUE | Validate metric-dataset pair |
| `data_source_metrics` | `data_source_metrics_data_source_id_idx` | `data_source_id` | List metrics for a dataset |
| `data_source_dimensions` | `data_source_dimensions_uq` | `(data_source_id, dimension_id)` UNIQUE | Validate dimension-dataset pair |
| `data_source_dimensions` | `data_source_dimensions_data_source_id_idx` | `data_source_id` | List dimensions for a dataset |
| `metric_variants` | `metric_variants_uq` | `(metric_id, toggle_option_id)` UNIQUE | Fast variant lookup |
| `metric_variants` | `metric_variants_metric_id_idx` | `metric_id` | All variants for a metric |
| `metric_toggle_options` | `metric_toggle_options_uq` | `(toggle_id, option_key)` UNIQUE | Resolve option within toggle |
| `report_templates` | `report_templates_stock_key_uq` | partial unique `(template_key)` WHERE `template_type = 'stock'` | Stock template slug uniqueness |
| `report_templates` | `report_templates_client_key_uq` | partial unique `(template_key, client_id)` WHERE `template_type = 'client'` | Client template slug uniqueness |
| `report_templates` | `report_templates_custom_key_uq` | partial unique `(template_key, client_id, user_id)` WHERE `template_type = 'custom'` | User template slug uniqueness |
| `report_templates` | `report_templates_module_key_uq` | partial unique `(template_key, module_key)` WHERE `template_type = 'module'` | Module template slug uniqueness |
| `report_templates` | `report_templates_category_idx` | `category` | List reports by category |
| `report_templates` | `report_templates_client_id_idx` | `client_id` | List custom reports per client |
| `report_templates` | `report_templates_user_id_idx` | `user_id` | List user's custom reports |
| `report_templates` | `report_templates_module_key_idx` | `module_key` | List module-specific reports |
| `report_templates` | `report_templates_type_client_idx` | `(template_type, client_id)` | Query reports by type + client |
| `report_template_filters` | `report_template_filters_uq` | `(template_id, dimension_id)` UNIQUE | One filter per dimension per report |
| `report_template_filters` | `report_template_filters_template_id_idx` | `(template_id, display_order)` | Ordered filter list per report |
| `bookmarks` | `bookmarks_user_id_template_id_idx` | `(user_id, template_id)` | List user's bookmarks for a report |
| `user_dimension_access` | `user_dimension_access_user_client_idx` | `(user_id, client_id)` | Fetch user's access rules |
| `client_dimension_access` | `client_dimension_access_client_id_idx` | `client_id` | Fetch client's access rules |
| `scheduled_reports` | `scheduled_reports_next_send_at_idx` | `next_send_at` | Cron job pickup |
| `export_jobs` | `export_jobs_user_id_status_idx` | `(user_id, status)` | User's export history |
| `user_report_access` | `user_report_access_user_uq` | partial unique `(user_id, client_id, template_id)` WHERE `user_id IS NOT NULL` | Prevent duplicate user grants |
| `user_report_access` | `user_report_access_role_uq` | partial unique `(role_id, client_id, template_id)` WHERE `role_id IS NOT NULL` | Prevent duplicate role grants |
| `notification_rules` | `notification_rules_user_client_idx` | `(user_id, client_id)` | List user's notification rules |
| `notification_log` | `notification_log_rule_id_idx` | `rule_id` | List logs for a rule |
| `notification_log` | `notification_log_user_unread_idx` | partial `(user_id, is_read)` WHERE `is_read = false` | Unread notification badge count |
| `report_template_shares` | `report_template_shares_uq` | unique `(template_id, shared_with_user_id)` | Prevent duplicate shares |
| `bookmark_shares` | `bookmark_shares_uq` | unique `(bookmark_id, shared_with_user_id)` | Prevent duplicate shares |
| `filter_preset_shares` | `filter_preset_shares_uq` | unique `(preset_id, shared_with_user_id)` | Prevent duplicate shares |
| `client_template_filter_overrides` | `client_template_filter_overrides_uq` | unique `(client_id, template_id, dimension_id)` | One override per client/template/dimension |
| `client_template_filter_overrides` | `client_template_filter_overrides_client_template_idx` | `(client_id, template_id)` | List client overrides per report |
| `user_template_filter_overrides` | `user_template_filter_overrides_uq` | unique `(user_id, template_id, dimension_id)` | One override per user/template/dimension |
| `user_template_filter_overrides` | `user_template_filter_overrides_user_template_idx` | `(user_id, template_id)` | List user overrides per report |

---

## 10. Relationship Map

### Cross-Schema References

The `report_builder` schema references the existing `beast_insights_v2` schema in these places:

| report_builder Table | Column | References |
|---------------------|--------|------------|
| `report_templates` | `created_by` | `beast_insights_v2.users.id` |
| `report_templates` | `client_id` | `beast_insights_v2.clients.id` |
| `report_templates` | `user_id` | `beast_insights_v2.users.id` |
| `bookmarks` | `user_id` | `beast_insights_v2.users.id` |
| `bookmarks` | `client_id` | `beast_insights_v2.clients.id` |
| `filter_presets` | `user_id` | `beast_insights_v2.users.id` |
| `filter_presets` | `client_id` | `beast_insights_v2.clients.id` |
| `dashboards` | `user_id` | `beast_insights_v2.users.id` |
| `dashboards` | `client_id` | `beast_insights_v2.clients.id` |
| `scheduled_reports` | `user_id` | `beast_insights_v2.users.id` |
| `scheduled_reports` | `client_id` | `beast_insights_v2.clients.id` |
| `export_jobs` | `user_id` | `beast_insights_v2.users.id` |
| `export_jobs` | `client_id` | `beast_insights_v2.clients.id` |
| `user_report_access` | `user_id` | `beast_insights_v2.users.id` |
| `user_report_access` | `client_id` | `beast_insights_v2.clients.id` |
| `notification_rules` | `user_id` | `beast_insights_v2.users.id` |
| `notification_rules` | `client_id` | `beast_insights_v2.clients.id` |
| `client_template_filter_overrides` | `client_id` | `beast_insights_v2.clients.id` |
| `user_template_filter_overrides` | `user_id` | `beast_insights_v2.users.id` |
| `client_dimension_access` | `client_id` | `beast_insights_v2.clients.id` |
| `notification_log` | `user_id` | `beast_insights_v2.users.id` |
| `notification_log` | `client_id` | `beast_insights_v2.clients.id` |
| `report_template_shares` | `shared_with_user_id` | `beast_insights_v2.users.id` |
| `report_template_shares` | `shared_by_user_id` | `beast_insights_v2.users.id` |
| `report_template_shares` | `client_id` | `beast_insights_v2.clients.id` |
| `bookmark_shares` | `shared_with_user_id` | `beast_insights_v2.users.id` |
| `bookmark_shares` | `shared_by_user_id` | `beast_insights_v2.users.id` |
| `bookmark_shares` | `client_id` | `beast_insights_v2.clients.id` |
| `filter_preset_shares` | `shared_with_user_id` | `beast_insights_v2.users.id` |
| `filter_preset_shares` | `shared_by_user_id` | `beast_insights_v2.users.id` |
| `filter_preset_shares` | `client_id` | `beast_insights_v2.clients.id` |
| `user_dimension_access` | `user_id` | `beast_insights_v2.users.id` |
| `user_dimension_access` | `client_id` | `beast_insights_v2.clients.id` |
| `user_dimension_access` | `created_by` | `beast_insights_v2.users.id` |
| `client_modules` | `client_id` | `beast_insights_v2.clients.id` |
| `client_modules` | `enabled_by` | `beast_insights_v2.users.id` |
| `client_dimension_access` | `client_id` | `beast_insights_v2.clients.id` |

### Internal Relationships (within `report_builder`)

```
data_sources ──┐
               ├── data_source_metrics ──── metrics
               └── data_source_dimensions ── dimensions

metric_toggles ── metric_toggle_options ── metric_variants ── metrics

dimensions ◄── dimensions (self-ref via parent_dimension_id for nested/tree)

report_templates ◄── report_template_filters ──── dimensions
report_templates ◄── report_templates (self-ref via parent_template_id)
report_templates ◄── bookmarks
report_templates ◄── scheduled_reports
report_templates ◄── dashboard_tiles
report_templates ◄── export_jobs
report_templates ◄── report_template_shares
report_templates ◄── user_report_access
report_templates ◄── client_template_filter_overrides ──── dimensions
report_templates ◄── user_template_filter_overrides ──── dimensions

bookmarks ◄── bookmark_shares
bookmarks ◄── scheduled_reports (via bookmark_id)

filter_presets ◄── filter_preset_shares

notification_rules ◄── notification_log

dashboards ◄── dashboard_tiles

dimensions ◄── user_dimension_access
dimensions ◄── client_dimension_access
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

Seeding happens during **Milestone 5** (Data Library), but this is what the seed script will populate:

| Table | Rows | Source |
|-------|------|--------|
| `data_sources` | 10 | Manual mapping of existing `reporting.*` materialized views |
| `metrics` | ~100 | Translated from existing DAX measures |
| `dimensions` | ~30 | Extracted from materialized view columns + lookup tables (includes filter metadata: is_multiple, data_source, static_options, filter_group, parent_dimension_id, etc.) |
| `data_source_metrics` | ~500 | Which metrics work with which materialized views |
| `data_source_dimensions` | ~200 | Which dimensions are available per materialized view |
| `metric_toggles` | 6 | The 6 context switches (approval_mode, date_basis, etc.) |
| `metric_toggle_options` | ~20 | Options per toggle |
| `metric_variants` | ~50 | SQL overrides per metric/toggle/option |
| `report_templates` | 11 | The 11 stock report JSON layouts (template_type='stock', user_id=NULL, version=1) |
| `report_template_filters` | ~200 | Base filter assignments per report (11 reports × ~18 filters) |

---

## Appendix: Full CREATE TABLE SQL

The full CREATE TABLE SQL is in `sql/001_create_report_builder_schema.sql` (28 tables). See the `sql/` folder for the complete schema.

> **Note:** The SQL file is the source of truth. If there is any discrepancy between this document and the SQL file, the SQL file takes precedence. Run the SQL file directly against PostgreSQL, then use `prisma db pull` to generate Prisma models.

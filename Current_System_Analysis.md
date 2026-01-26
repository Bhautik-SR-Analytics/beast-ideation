# Beast Insights - Current System Analysis

## 1. Platform Overview

Beast Insights is a payment analytics and reporting platform serving multiple clients. It provides reports on sales, approvals, chargebacks, refunds, profitability, retention, LTV, MID health, and routing optimization.

**Current Stack:** PostgreSQL + Node.js + React + Power BI Embedded

**Platform URL:** https://app.beastinsights.co

---

## 2. Database Architecture

### 2.1 Schema Overview

| Schema | Purpose |
|---|---|
| `beast_insights_v2` | Application layer - clients, users, pages, reports, filters, navigation, metrics |
| `data` | Raw transactional data per client (`orders_{client_id}`) |
| `public` | Shared dimension tables (campaigns, bin_lookup, offers, decline_groups, etc.) |
| `reporting` | Pre-computed materialized views per client (10 view types) |
| `beastinsights_alert` | Alert-specific data |
| `routing` | Routing-related data |

### 2.2 Client Management

**Table:** `beast_insights_v2.clients`

| Column | Type | Description |
|---|---|---|
| id | bigint | Client ID (e.g., 10000, 10001) |
| name | varchar | Client name (e.g., "Beast Demo", "SafeWebOrder") |
| package_id | bigint | Subscription package |
| is_active | boolean | Active flag |
| is_new_etl | boolean | ETL version flag |
| data_timezone | varchar | Data timezone |
| client_timezone | varchar | Display timezone |
| snapshot_email_list | varchar | Email list for snapshots |
| snapshot_telegram_chat_id | bigint | Telegram notification |
| stripe_customer_id | varchar | Billing integration |

---

### 2.3 Raw Data Layer

**Table pattern:** `data.orders_{client_id}`

81 columns per client. This is the source-of-truth transactional data.

#### Key Column Groups:

**Identifiers:**
| Column | Type | Description |
|---|---|---|
| id | uuid | Primary key |
| unique_order_key | bigint | Unique order identifier |
| unique_customer_key | bigint | Unique customer identifier |
| order_id | varchar | CRM order ID |
| transaction_id | varchar | Transaction reference |
| customer_number | varchar | Customer number |
| ancestor_order_id | varchar | Parent order (for rebills) |
| auth_id | varchar | Auth reference |

**Client/CRM:**
| Column | Type | Description |
|---|---|---|
| client_id | bigint | Client reference |
| client_name | varchar | Client name |
| crm | varchar | CRM source (Sticky, Konnektive, VRIO, etc.) |

**Dates & Times:**
| Column | Type | Description |
|---|---|---|
| date_of_sale | date | Sale date |
| trial_date | date | Trial start date |
| time_of_sale | time | Sale time |
| time_stamp | timestamp | Full timestamp |
| shipped_date | date | Shipping date |
| chargeback_date | date | CB date |
| recurring_date | date | Recurring billing date |
| hold_date | date | Hold date |
| void_date | date | Void date |
| refund_date | date | Refund date |
| refund_void_date | date | Refund void date |
| processing_date_time | timestamp | Processing timestamp |

**Product & Campaign:**
| Column | Type | Description |
|---|---|---|
| product_id | bigint | Product reference |
| product_name | varchar | Product name |
| campaign_id | bigint | Campaign reference |
| gateway_id | bigint | Payment gateway ID |

**Customer Info:**
| Column | Type | Description |
|---|---|---|
| bill_first | varchar | First name |
| bill_last | varchar | Last name |
| bill_email | varchar | Email |
| bill_phone | varchar | Phone |
| bill_city | varchar | City |
| bill_state | varchar | State |
| bill_zip | varchar | Zip |
| bill_country | varchar | Country |

**Financial:**
| Column | Type | Description |
|---|---|---|
| order_total | numeric | Order amount |
| refund_amount | numeric | Refund amount |
| void_amount | numeric | Void amount |
| refund_void_amount | numeric | Refund void amount |

**Status Flags (all boolean):**
| Column | Description |
|---|---|
| is_approved | Transaction approved |
| final_order_status | Final status |
| is_cascaded | Cascaded to another gateway |
| is_fraud | Flagged as fraud |
| is_chargeback | Has chargeback |
| is_rma | Has RMA |
| is_recurring | Is recurring transaction |
| is_void | Is voided |
| is_refund | Is refunded |
| is_refund_void | Is refund voided |
| is_blacklisted | Blacklisted |
| is_test | Test transaction |
| is_cancelled | Cancelled |
| is_exclude | Excluded from reports |

**Affiliate & Tracking:**
| Column | Type | Description |
|---|---|---|
| affid | varchar | Affiliate ID |
| sub_affid | varchar | Sub-affiliate ID |
| c | varchar | Affiliate tier 3 |

**Transaction Details:**
| Column | Type | Description |
|---|---|---|
| credit_card_number | varchar | Masked card number |
| credit_card_expiration | varchar | Card expiry |
| ip_address | varchar | Customer IP |
| bin | integer | Card BIN (first 6 digits) |
| cycle | integer | Billing cycle number |
| attempt | integer | Attempt number |
| order_type | varchar | Order type |
| decline_reason | varchar | Current decline reason |
| original_decline_reason | varchar | Original decline reason |
| decline_group | varchar | Decline category |

---

### 2.4 Dimension Tables (public schema)

#### `public.campaigns`
| Column | Type | Description |
|---|---|---|
| campaign_id | bigint | Campaign ID |
| campaign_name | varchar | Campaign name |
| campaign_type | varchar | Campaign type |
| crm | varchar | CRM source |
| client_id | bigint | Client reference |
| subscription_flag | integer | Subscription indicator |
| cpa | numeric | Cost per acquisition |
| is_exclude | integer | Exclusion flag |
| is_campaign_type_filled_with_ai | boolean | AI-classified type |

#### `public.bin_lookup`
| Column | Type | Description |
|---|---|---|
| bin | bigint | BIN number |
| bank | varchar | Issuing bank name |
| card | varchar | Card brand (Visa, MC, etc.) |
| type | varchar | Card type |
| class | varchar | Card class (Credit, Debit, Prepaid) |
| country | varchar | Issuing country |
| countrycode | varchar | Country code |

#### `public.offers`
| Column | Type | Description |
|---|---|---|
| campaign_id | integer | Campaign reference |
| campaign_name | varchar | Campaign name |
| offer_id | varchar | Offer ID |
| offer_name | varchar | Offer name |
| product_id | varchar | Product ID |
| product_name | varchar | Product name |
| billing_model_id | varchar | Billing model ID |
| billing_model_name | varchar | Billing model name |
| crm | varchar | CRM source |
| client_id | integer | Client reference |

#### `public.decline_groups`
| Column | Type | Description |
|---|---|---|
| decline_reason | varchar | Original decline reason text |
| decline_group | varchar | Mapped group (Issuer Decline, Insufficient Funds, etc.) |

---

### 2.5 Materialized Views (reporting schema)

10 materialized view types are created **per client** (e.g., `order_summary_10000`, `order_summary_10007`, etc.).

#### 2.5.1 `order_summary_{client_id}` - Core Aggregated Data

**Purpose:** Primary view powering Dashboard, Sales, Approval%, CB & Refunds, and Profitability reports.

| Column | Type | Description |
|---|---|---|
| client_id | bigint | Client ID |
| date | date | Transaction date |
| campaign_id | bigint | Campaign reference |
| product_id | bigint | Product reference |
| trial_gateway_id | bigint | Trial gateway |
| gateway_id | bigint | Gateway reference |
| bin | integer | Card BIN |
| affid | varchar | Affiliate ID |
| sub_affid | varchar | Sub-affiliate ID |
| c | varchar | Affiliate tier 3 |
| price_point | numeric(18,2) | Price point |
| sales_type | text | "Initials" / "Rebills" / "Straight Sales" |
| billing_cycle | integer | Billing cycle number |
| refund_type | text | Refund classification |
| alert_type | text | Alert classification |
| cpa | numeric | Cost per acquisition |
| attempts_total | bigint | Total attempts including cascade |
| attempts | bigint | Non-cascade attempts |
| approvals | bigint | Approved transactions |
| net_approvals | bigint | Net approvals (including retries) |
| approvals_organic | bigint | First-attempt approvals |
| cancel | bigint | Cancellation count |
| revenue | numeric | Revenue amount |
| cb | bigint | Chargeback count |
| cb_dollar | numeric | Chargeback dollar amount |
| cb_cb_date | bigint | CB count by CB date |
| cb_cb_date_dollar | numeric | CB dollar by CB date |
| refund | bigint | Refund count |
| refund_dollar | numeric | Refund dollar amount |
| refund_refund_date | bigint | Refund count by refund date |
| refund_refund_date_dollar | numeric | Refund dollar by refund date |

**Granularity:** One row per unique combination of (date, campaign, product, gateway, BIN, affiliate, sales_type, billing_cycle, refund_type, alert_type).

---

#### 2.5.2 `mid_summary_{client_id}` - MID Health

**Purpose:** Powers MID Performance report.

| Column | Type | Description |
|---|---|---|
| client_id | integer | Client ID |
| gateway_id | bigint | Gateway/MID ID |
| month_year | varchar | Month-Year period |
| volume | numeric | Total volume ($) |
| initials | numeric | Initial transactions |
| rebills | numeric | Rebill transactions |
| overall_cb | numeric | Total chargebacks |
| cb_visa / cb_master | numeric | CB by card network |
| overall_declines | numeric | Total declines |
| decline_percent | numeric | Decline rate |
| attempts / attempts_visa / attempts_master | numeric | Attempts by network |
| approvals / approvals_visa / approvals_master | numeric | Approvals by network |
| overall_refund | numeric | Total refunds |
| refund_alert_visa / refund_alert_master | numeric | Refund alerts by network |
| alert / rdr / ethoca | numeric | Alert counts by type |
| rdr_effective_percent / ethoca_effective_percent | numeric | Alert effectiveness |
| cb_rate / cb_visa_rate / cb_master_rate | numeric | CB rates |
| decline_rate / alert_rate | numeric | Other rates |
| health_tag | text | "healthy" / "at-risk" / "critical" / "inactive" |
| monthly_cap | numeric(18,2) | Monthly processing cap |
| capacity_left | numeric | Remaining capacity ($) |
| initials_performance / rebills_performance | text | Performance rating |
| near_capacity / decline_spike / high_cb_alert_coverage | text | Alert flags |

---

#### 2.5.3 `cb_refund_alert_{client_id}` - CB/Refund Growth Tracking

**Purpose:** Powers CB & Refunds Growth report with day-count tracking.

Same columns as `order_summary` plus:
| Column | Type | Description |
|---|---|---|
| cb_no_of_days | integer | Days from sale to chargeback |
| refund_no_of_days | integer | Days from sale to refund |
| dispute_no_of_days | integer | Days from sale to dispute |

---

#### 2.5.4 `cohort_summary_{client_id}` - Retention & LTV Cohorts

**Purpose:** Powers LTV and Retention reports.

| Column | Type | Description |
|---|---|---|
| client_id | bigint | Client ID |
| date | date | Cohort date |
| sales_type | text | Sales type |
| billing_cycle | integer | Billing cycle |
| attempt_col | integer | Attempt column |
| refund_type / alert_type | text | Classification |
| cpa | numeric | CPA |
| trial_price_point | numeric | Trial price |
| trial_bin | integer | Trial BIN |
| trial_sub_affid / trial_c / trial_affid | varchar | Trial affiliate info |
| trial_campaign_id / trial_product_id / trial_gateway_id | bigint | Trial references |
| attempts / approvals / net_approvals / cancel | bigint | Transaction counts |
| revenue / cb / cb_dollar / refund / refund_dollar | numeric | Financial metrics |

---

#### 2.5.5 `decline_recovery_{client_id}` - Retry Analysis

**Purpose:** Powers Decline Recovery report.

| Column | Type | Description |
|---|---|---|
| decline_group | varchar | Decline category |
| recovery_attempts | integer | Number of recovery attempts |
| organic_declines | bigint | First-time declines |
| declines | bigint | Total declines |
| reattempts | bigint | Retry count |
| recovered | bigint | Successfully recovered |
| cancel / cb / refund | bigint | Outcome counts |
| organic_declines_dollar / recovered_dollar | numeric | Dollar amounts |

Plus standard dimensions: date, gateway_id, bin, campaign_id, product_id, affid, sub_affid, c, sales_type, billing_cycle.

---

#### 2.5.6 `hourly_revenue_{client_id}` - Real-time Dashboard

**Purpose:** Powers Dashboard real-time revenue chart.

| Column | Type | Description |
|---|---|---|
| client_id | integer | Client ID |
| sort_order | integer | Hour sort |
| hour | time | Hour of day |
| avg_7d_revenue / avg_7d_initial / avg_7d_rebill / avg_7d_straight_sales | numeric | 7-day averages |
| today_revenue / today_initial / today_rebill / today_straight_sales | numeric | Today's values |

---

#### 2.5.7 `alert_summary_{client_id}` - Alert Aggregations

**Purpose:** Powers Alert Analytics summary section.

| Column | Type | Description |
|---|---|---|
| date | date | Alert date |
| gateway_id | bigint | Gateway |
| alert_type | varchar | Alert type (RDR, Ethoca, CDRN) |
| alert_status | text | Status |
| alert_duplication | text | Duplication category |
| card_bin | varchar | Card BIN |
| is_crm / is_refund_void / is_cb_final / is_tc40 | boolean | Status flags |
| alert_count / alert_dollar | numeric | Alert metrics |
| rdr / rdr_dollar | bigint/numeric | RDR metrics |
| ethoca / ethoca_dollar | bigint/numeric | Ethoca metrics |
| cdrn / cdrn_dollar | bigint/numeric | CDRN metrics |

Plus standard dimensions: campaign_id, product_id, billing_cycle, affid, sub_affid, c.

---

#### 2.5.8 `alert_details_{client_id}` - Individual Alert Records

**Purpose:** Powers Alert Analytics raw data table.

| Column | Type | Description |
|---|---|---|
| alert_id | varchar | Alert identifier |
| alert_date | date | Alert date |
| alert_type | varchar | Type (RDR, Ethoca, CDRN) |
| alert_status | text | Status |
| alert_duplication | text | Duplication info |
| order_id | varchar | Order reference |
| bill_email | varchar | Customer email |
| is_cb / is_refund_void / is_approved / is_crm | boolean | Flags |
| transaction_amount | numeric | Amount |
| transaction_date | date | Transaction date |
| gateway_id / gateway_alias | bigint/varchar | Gateway info |
| card_bin / card_group | varchar/text | Card info |
| count_of_alert / alert_dollar / alert_count | numeric | Counts |

---

#### 2.5.9 `order_details_{client_id}` - Transaction Detail

**Purpose:** Powers Order Details report (transaction-level drill-down).

_(Structure mirrors the raw `data.orders_{id}` table with selected columns for display)_

---

#### 2.5.10 `products_{client_id}` - Product Dimension

**Purpose:** Client-specific product dimension with enriched data.

_(Product ID, name, group, cost, and other product attributes)_

---

## 3. Application Layer (beast_insights_v2 schema)

### 3.1 Reports & Pages

**`reports`** - Maps to Power BI reports per client
| Column | Description |
|---|---|
| id | Report ID |
| report_name | Internal name (e.g., "Starter Reports - Dev_10000") |
| report_display_name | Display name ("Starter Report") |
| powerbi_report_id | Power BI report GUID |
| powerbi_workspace_id | Power BI workspace GUID |
| client_id | Client reference |

**`pages`** - Individual report pages
| Column | Description |
|---|---|
| id | Page ID |
| page_name | Power BI page identifier (e.g., "77ad27f2e20a9de8760a") |
| page_display_name | Display name ("Sales", "MID Performance", etc.) |
| report_id | FK to reports |
| route_id | FK to routes |
| display_order | Sort order |

**Current Pages:** Sales, MID Performance, Approval %, CB & Refunds, Retention, Profitability, Dashboard, LTV, Decline Recovery, Alert Analytics, Order Details, Routing, Ask AI

### 3.2 Navigation

**`navigation_items`** - Hierarchical navigation tree
| Column | Description |
|---|---|
| nav_item_id | Navigation item ID |
| parent_id | Parent nav item (null = top-level) |
| nav_item_type | "page" / "section" |
| title | Display title |
| icon | Icon name (e.g., "GrCart") |
| page_id | FK to pages |
| display_order | Sort order |

### 3.3 Filters

**`filter_definitions`** - 30+ filter configurations

| ID | Filter Key | Name | Table | Column | Category |
|---|---|---|---|---|---|
| 1 | campaignSelected | Campaign | campaigns | Campaign | more |
| 2 | campaignTypeSelected | Campaign Type | campaigns | Campaign Type | more |
| 3 | productSelected | Product | products | Product | more |
| 4 | productIdSelected | Product ID | products | product_id | more |
| 5 | productGroupSelected | Product Group | products | product_group | more |
| 6 | bankSelected | Bank | bin_lookup | Bank | more |
| 7 | gatewayIdSelected | Gateway ID | payments | gateway_id | more |
| 8 | mccSelected | MCC | payments | mcc | more |
| 9 | lenderSelected | Acquirer | payments | lender | more |
| 10 | gatewayAliasSelected | Gateway Alias | payments | gateway_alias | more |
| 11 | midCorpSelected | MID Corp | payments | corp | more |
| 12 | affiliateIdSelected | Affiliate ID | AffID | affid | more |
| 15 | cardBrandSelected | Card Brand | bin_lookup | card | more |
| 16 | pricePointSelected | Price Point | PricePoint | order_total | more |
| 19 | billingCycle | Billing Cycle | order_details | Billing Cycle | more |
| 26 | salesTypeSelected | Sales Type | order_details | Sales Type | main |
| 27 | dateRange | Date Range | Calendar | Date | main |
| 32 | processingFeesSelected | Processing Fees | - | Value | input |

**`filter_options`** - Per-client filter values stored as JSONB

### 3.4 Metrics

**`metrics`** - 17 currently defined metrics
| ID | Name | Power BI Metric Key |
|---|---|---|
| 1 | Cancels | cancels |
| 2 | Cancels (0-3 Days) | cancels_0_to_3_days |
| 3 | Chargebacks $ | chargebacks |
| 4 | Chargebacks Count | chargebacks_count |
| 5 | Initials | initials |
| 6 | Initials Approval Rate | initials_approval_rate |
| 7 | Initials Revenue | initials_revenue |
| 8 | Gross Revenue | gross_revenue |
| 9 | Net Revenue | net_revenue |
| 10 | % Cancels | per_cancels |
| 11 | Rebills | rebills |
| 12 | Rebills Revenue | rebills_revenue |
| 13 | Rebill Cycle 1 % | rebill_cycle_1_per |
| 14 | Straight Sales | straight_sales |
| 15 | Straight Sales Revenue | straight_sales_revenue |
| 16 | Refund $ | refund |
| 17 | Refund Count | refund_count |

### 3.5 Order Details Column Mapping

**`order_details_column_definitions`** - Maps DB columns to display names
| db_table | db_column | display_name |
|---|---|---|
| bl | bank | Bank |
| bl | card | Card Brand |
| bl | class | Card Class |
| bl | type | Card Type |
| c | campaign_id | Campaign ID |
| c | campaign_name | Campaign Name |
| c | campaign_type | Campaign Type |
| od | affid | Affiliate ID |
| od | bin | BIN |
| od | cycle | Billing Cycle |
| od | date_of_sale | Date |
| od | decline_group | Decline Group |
| od | gateway_id | Gateway ID |
| od | product_name | Product Name |
| p | gateway_alias | Gateway Alias |
| p | corp | Corp |

---

## 4. Power BI Layer (Current)

### 4.1 Report Structure

Each client has:
- **1 Standard Report** (`Standard - {client_id}.Report`) - Contains 22 pages with visual definitions
- **1 Standard SemanticModel** (`Standard - {client_id}.SemanticModel`) - Contains 75+ tables, 100+ DAX measures
- **Optional Custom Reports** (`Custom - {client_id}.Report`) - References the Standard SemanticModel

### 4.2 Semantic Model Architecture

**Data Connection:** PostgreSQL on `beastinsights-database.postgres.database.azure.com`, parameterized by `client_id`.

**Table Categories (75+ tables):**

| Prefix | Purpose | Examples |
|---|---|---|
| `smz_` | Summarized/fact tables from materialized views | `smz-order_details`, `smz_alert`, `smz_ltv`, `smz_cohort`, `smz_decline_recovery`, `smz_mid_table` |
| `dim_` | Dimension tables | `dim_calendar`, `dim_payments`, `dim_campaigns`, `dim_products`, `dim_billing_cycle`, `dim_bin_lookup` |
| `param_` | Parameter/toggle tables for UI | `param_date_selection`, `param_groupby_column`, `param_groupby_measure`, `param_affid_sub_affid_c` |
| `prof_` | Profitability parameter tables | `prof_processing_fees`, `prof_reserve`, `prof_cb_fees`, `prof_cpa`, `prof_rdr_alert`, `prof_ethoca_alert` |
| `cust_` | Customer/toggle settings | `cust_approval_page_toggle`, `cust_cb_alert`, `cust_cb_refund_date_toggle`, `cust_month_year` |
| `fact_` | Additional fact tables | `fact_orders` (BIN routing), `fact_alert`, `fact_subscription_control` |
| `dp_` | Datapoint tables for specific visuals | Various |

### 4.3 Key DAX Measures

**Core Counting:**
```dax
# Attempts = SUM('smz-order_details'[attempts])
# Approvals = SUM('smz-order_details'[approvals])
# Net Approvals = SUM('smz-order_details'[net_approvals])
# Organic Approvals = SUM('smz-order_details'[approvals_organic])
# Cancels = SUM('smz-order_details'[cancel])
$ Revenue = SUM('smz-order_details'[revenue])
```

**Dynamic Toggle (Organic vs Net):**
```dax
# Approval Toggle =
  IF(SELECTEDVALUE(cust_approval_page_toggle[Hint])="Organic", [# Organic Approvals],
    IF(SELECTEDVALUE(cust_approval_page_toggle[Hint])="Net", [# Net Approvals],
      [# Approvals]))
```

**Rate Calculations:**
```dax
Approval % = DIVIDE([# Approval Toggle], [# Attempts], 0)
CB % = DIVIDE([# CB], [# Approval Toggle], 0)
Cancel % = DIVIDE([# Cancels], [# Approval Toggle], 0)
```

**Sales Type Segmentation:**
```dax
# Initials = CALCULATE([# Approval Toggle], FILTER('smz-order_details', [sales_type]="Initials"))
# Rebills = CALCULATE([# Approval Toggle], FILTER('smz-order_details', [sales_type]="Rebills"))
$ Initials = CALCULATE([$ Revenue], FILTER('smz-order_details', [sales_type]="Initials"))
```

**Profitability (User-Adjustable Parameters):**
```dax
$ Processing Fees = prof_processing_fees[Processing Fees] * [$ Revenue Toggle] / 100
$ CB Fees = prof_cb_fees[CB Fees] * [# CB Toggle]
$ Reserve = prof_reserve[Reserve] * [$ Revenue Toggle] / 100
$ Product Cost = SUMX('smz-order_details', RELATED('dim_products'[product_cost]) * 'smz-order_details'[approvals])

$ Gross Profit = [$ Revenue Toggle] - [$ Refund Alert Toggle] - [$ Refund CS Toggle]
  - [$ CB Toggle] - [$ Processing Fees] - [$ CB Fees]
  - [$ Product Cost] - [$ CPA Cost] - [$ Alert Cost] - [$ Reserve]

% Gross Margin = DIVIDE([$ Gross Profit], [$ Revenue Toggle], 0)
```

**BIN Routing Impact:**
```dax
# Approvals BIN Routing = COUNTROWS(FILTER(fact_orders, fact_orders[is_approved]=TRUE()))
$ Revenue Impact = [complex VAR-based calculation comparing actual vs optimal gateway routing]
Approval % Lift = [comparison of current vs best available approval rate per BIN]
```

### 4.4 Data Model Relationships

```
smz-order_details.date         --> dim_calendar.Date
smz-order_details.campaign_id  --> dim_campaigns.campaign_id
smz-order_details.product_id   --> dim_products.product_id
smz-order_details.gateway_id   --> dim_payments.gateway_id
smz-order_details.bin          --> dim_bin_lookup.BIN
smz-order_details.billing_cycle --> dim_billing_cycle.index

smz_alert.date                 --> dim_calendar.Date
smz_alert.campaign_id          --> dim_campaigns.campaign_id
smz_alert.product_id           --> dim_products.product_id
smz_alert.gateway_id           --> dim_payments.gateway_id

smz_cohort.date                --> dim_calendar.Date
dim_offers.campaign_id         --> dim_campaigns.campaign_id
dim_offers.product_id          --> dim_products.product_id (bidirectional)
```

---

## 5. Report-to-Data Mapping

| Report | Primary View(s) | Key Dimensions Used | Key Metrics |
|---|---|---|---|
| **Dashboard** | `order_summary` + `hourly_revenue` | date, card_brand | Approvals, Revenue, CB, Refunds, Alerts, Visa/MC Health |
| **Sales** | `order_summary` | date, campaign, product, affiliate | Initials, Rebills, Revenue, Traffic Quality |
| **MID Performance** | `mid_summary` | gateway_id, month_year | CB rate, Alert rate, Capacity, Health Tag |
| **Approval %** | `order_summary` | date, campaign, product, gateway, bank, BIN | Attempts, Approvals, Approval Rate |
| **Decline Recovery** | `decline_recovery` | decline_group, campaign, gateway | Organic Declines, Reattempts, Recovered, Recovery Rate |
| **Profitability** | `order_summary` | date, campaign, product | Revenue, Refunds, CB, Fees, Gross Profit, Margin |
| **LTV** | `cohort_summary` | trial_campaign, trial_product, cohort_date | Revenue per cohort period, LTV |
| **Retention** | `cohort_summary` | date, campaign, product, billing_cycle | Initial %, Cycle 1-5 retention % |
| **CB & Refunds** | `order_summary` + `cb_refund_alert` | date, campaign, product | CB count/$, Refund count/$, CB% |
| **CB & Refunds Growth** | `cb_refund_alert` | date, cb_no_of_days | CB% over time (0-150 days) |
| **Alert Analytics** | `alert_summary` + `alert_details` | gateway, alert_type | Alert count, Effectiveness, Duplication |
| **Order Details** | `order_details` | All columns | Transaction-level data |
| **Routing Insights** | Raw orders + `bin_lookup` | BIN, bank, gateway | Approval rate by BIN/gateway, Revenue impact |

---

## 6. Current Limitations

1. **Power BI embedding takes 5-10 seconds** to load, creating a poor user experience
2. **No user-created custom reports** - all reports are predefined by Beast team
3. **No saved views/filters** - users must re-apply filters every session
4. **Only 17 metrics defined** in the metrics table vs 100+ measures in Power BI DAX
5. **Fixed report layouts** - users cannot choose which widgets/charts to see
6. **No cross-report dashboard** - users cannot pin widgets from different reports
7. **Per-client Power BI setup required** - each new client needs a new SemanticModel and Report cloned and parameterized
8. **No real-time API** - data goes through Power BI's refresh cycle
9. **Profitability parameters are Power BI slicers** - not saved per user
10. **Limited export** - depends on Power BI export capabilities

---

*Document generated: 2026-01-26*
*Based on analysis of PostgreSQL database, Power BI report files, and platform documentation*

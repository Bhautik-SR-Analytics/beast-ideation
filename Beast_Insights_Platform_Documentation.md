# Beast Insights Platform Documentation

## Overview

Beast Insights is a comprehensive payment analytics and reporting platform that provides businesses with detailed insights into their payment processing, sales performance, chargebacks, refunds, and routing optimization.

**Platform URL:** https://app.beastinsights.co

---

## Navigation Structure

The platform is organized into the following main sections:

### Analytics
- Dashboard
- Sales
- MID Performance

### Processing
- Approval %
- Decline Recovery

### Profits
- Profitability
- LTV (Lifetime Value)
- Retention

### CB, Refunds & Alert
- CB & Refunds
- CB & Refunds Growth
- Alert Analytics

### Order Details (NEW)
- Transaction-level data viewer

### Routing
- Routing Insights
- Routing Setup (NEW) *May not be available for all clients*

### Reconciliation (May not be available for all clients)
- Bank True Up
- Reserve Tracking

### Configuration
- User Management
- Reporting Setup
- Integration
- Schedule

---

## Reports Detail

### 1. Dashboard

**URL:** /home

**Description:** Main overview dashboard providing a snapshot of business performance.

#### Top Section - Today's Performance
- **Gross Revenue:** Current day revenue with 7-day average comparison
- Real-time revenue tracking chart (hourly breakdown)

#### Overview Section

**Filters Available:**
- Date Range Picker with presets:
  - Today
  - Yesterday
  - Last 7 days
  - Last calendar week
  - Last 4 weeks
  - Last 3 months
  - Last 6 months
  - Month to date
- Download/Export
- Schedule Reports

**Cards/Widgets:**

1. **Sales Breakdown**
   - Initials (count & percentage)
   - Rebills (count & percentage)
   - Total (count & percentage)

2. **Gross Volume**
   - Total dollar amount
   - Trend chart over time

3. **Cashflow Breakdown**
   - Sales Type breakdown:
     - Initials ($)
     - Rebills ($)
     - Straight Sales
   - Gross Revenue ($)
   - Refund Alert ($)
   - Refund CS ($)
   - Chargeback ($)
   - Net Revenue ($)

4. **Approval Rate**
   - Percentage with trend chart

5. **Sales Count**
   - Total sales with trend chart

6. **Visa Health**
   - Partial VAMP %
   - Alerts %
   - Declines %
   - # Chargebacks
   - # Alerts
   - # Approvals

7. **Mastercard Health**
   - Chargeback %
   - Alerts %
   - Declines %
   - # Chargebacks
   - # Alerts
   - # Approvals

8. **Disputes**
   - Total count
   - Breakdown by type:
     - CB (count & %)
     - Ethoca (count & %)
     - RDR (count & %)

#### Metrics Trends Section

**Metric Toggle Options:**
- # Approvals
- Approval Rate
- # Refund Alert
- # Refund CS
- # CB
- # Alert
- $ Gross Revenue
- $ Net Revenue

**Time Aggregation:** Day

**Card Network Filters:**
- Deselect all
- Visa
- Mastercard
- Discover
- Other

#### All Metrics Table
Daily breakdown with columns:
- Date
- # Approval
- Approval Rate
- % Cancel
- # Refund Alert
- % Refund Alert
- # Refund CS
- % Refund CS
- # CB
- % CB
- # Alert
- $ Gross Revenue
- $ Net Revenue

---

### 2. Sales Report

**URL:** /reports/sales

**Description:** Spot trends in sales by product, campaign, or affiliate to improve conversion.

**Filters:**
- **Date Range:** Preset options + custom date picker
- **Group By Options:**
  - Campaign
  - Campaign Type
  - Campaign ID
  - Product
  - Product ID
  - Product Group
  - Gateway ID
  - Acquirer
  - Gateway Alias
  - MID Corp
- **More Filters:**
  - Campaign
  - Campaign Type
  - Product
  - Product ID
  - Product Group
  - Bank
  - Gateway ID
  - MCC
  - Acquirer
  - Gateway Alias
  - MID Corp
  - Billing Cycle
- Download/Export
- Schedule

**Sections:**

1. **Summary Card**
   - Initials (count & $)
   - Rebills (count & $)
   - Total (count & $)

2. **Sales Trend Card**
   - Total sales count with trend visualization

3. **Insights Card**
   - AI-generated insights about sales trends
   - Trend direction analysis
   - Revenue attribution
   - AOV (Average Order Value) for Initials and Rebills

4. **Sales Table**
   Columns:
   - Sales Trend (sparkline)
   - # Initials
   - $ Initials
   - # Rebills
   - $ Rebills
   - # Straight Sales
   - $ Straight Sales

5. **Traffic Quality Table**
   Columns:
   - Affiliate
   - Initials Attempts
   - Initial Approved
   - Initial Approval %
   - Initial Cancel %
   - Initial Refund %
   - Initial CB %
   - C1 Approval
   - C1 Approval %
   - C1 Cancel %
   - C1 Refund %
   - C1 CB %

---

### 3. MID Performance Report

**URL:** /reports/mid-performance

**Description:** Monitor MID health and avoid shutdowns by tracking approval and risk trends.

**Filters:**
- **Month Selector:** Monthly view
- **Notifications Settings:** Configure alerts
- **More Filters:**
  - Gateway ID
  - MCC
  - Acquirer
  - Gateway Alias
  - Health Tag
  - Affiliate Tier 3
- Download/Export
- Schedule

**Sections:**

1. **MID Health Overview**
   - Total MID count
   - Health status breakdown:
     - at-risk (count vs. last month)
     - critical (count vs. last month)
     - healthy (count vs. last month)
     - inactive (count vs. last month)

2. **Visa Health**
   - Partial VAMP %
   - Alerts %
   - Approvals %
   - Last Month comparisons
   - # Chargebacks
   - # Refunds
   - # Approvals

3. **Mastercard Health**
   - Chargebacks %
   - Alerts %
   - Approvals %
   - Last Month comparisons
   - # Chargebacks
   - # Refunds
   - # Approvals

4. **MID Portfolio Tabs:**
   - All GW (total count)
   - Top Initial MIDs
   - Best Rebill MIDs
   - Near GW Limit
   - Decline Spike
   - High CB
   - High Alerts
   - No Alerts

5. **MID Portfolio Table**
   Columns:
   - GW ID
   - Gateway Alias
   - Acquirer
   - Health Tag
   - Initials Performance (star rating)
   - Rebills Performance (star rating)
   - # Approval
   - $ Volume
   - $ Capacity Left
   - # Overall CB
   - Overall

---

### 4. Approval % Report

**URL:** /reports/approval

**Description:** Understand what's driving approvals or declines to optimize processing strategies.

**Filters:**
- **Approval Type Selector:**
  - First Attempt Approval %: Counts as approved only if first attempt succeeds. Retries are ignored.
  - Net Approval (After Retries) %: Marks an order as approved if any attempt succeeds, no matter how many attempts failed before.
- **Date Range:** Preset options + custom
- **Sales Type:** Initials, Rebills (multi-select)
- **More Filters:**
  - Campaign
  - Campaign Type
  - Product
  - Product ID
  - Product Group
  - Bank
  - Gateway ID
  - MCC
  - Acquirer
  - Gateway Alias
  - MID Corp
  - Affiliate ID
- Download/Export
- Schedule

**Sections:**

1. **Summary**
   - Sales Type
   - # Approvals
   - Approvals Rate (with progress bar)

2. **# Approvals Chart**
   - Total count with trend line

3. **Approval Rate Chart**
   - Percentage with trend line

4. **Products Table**
   Columns:
   - Product
   - # Attempt
   - # Approval
   - Approval Rate (with bar)

5. **Campaign Table**
   Columns:
   - Campaign
   - # Attempt
   - # Approval
   - Approval Rate (with bar)

6. **Acquirer and Gateway Table**
   Columns:
   - Acquirer
   - # Attempt
   - # Approval
   - Approval Rate (with bar)

7. **Bank and BIN Table**
   Columns:
   - Bank
   - # Attempt
   - # Approval
   - Approval Rate (with bar)

---

### 5. Decline Recovery Report

**URL:** /reports/decline-recovery

**Description:** Measure retry effectiveness and uncover why customers fail to convert.

**Filters:**
- **Date Range:** Preset options + custom
- **Sales Type:** Rebills (dropdown)
- **More Filters**
- Download/Export
- Schedule

**Sections:**

1. **Missed Recovery Opportunity**
   - Dollar amount
   - Bar chart by Cycle (1, 2, 3, 4, 5, 6+)

2. **Recovery Performance Over Time**
   - Trend chart

3. **Recovery Rate**
   - Percentage
   - Bar chart by Recovery Attempts (1-7 + Total)

4. **Recovery Rate by Decline Group Table**
   Columns:
   - Decline Group
   - Organic Decline Trend (sparkline)
   - Organic Decline %
   - # Decline
   - # Reattempt
   - # Recovered
   - Recovery Rate
   - Cancel %
   - CB %
   - Refund %

   Decline Groups:
   - Issuer Decline
   - Insufficient Funds
   - Fraudulent
   - Gateway/Network Issue
   - Customer Account Issue
   - Invalid Card/Type/Amount
   - CVV Mismatch
   - 3DS Required
   - Expired Card

5. **Recovery Rate by Campaign Table**
   Columns:
   - Campaign
   - # Recovered
   - Recovery Rate (with bar)

6. **Recovery Rate by Gateway Table**
   Columns:
   - Acquirer
   - # Recovered
   - Recovery Rate (with bar)

---

### 6. Profitability Report

**URL:** /reports/profitability

**Description:** Compare revenue vs. cost across processors, MIDs, and traffic sources.

**Filters:**
- **View Type:** Cashflow view
- **Date Range:** Preset options + custom
- **Group By:** Campaign → Product (hierarchical)
- **More Filters**
- Download/Export
- Schedule

**Sections:**

1. **Profitability Calculator (Interactive)**
   - Processing Fees: Adjustable %
   - Reserve: Adjustable %
   - CPA: Adjustable $ amount
   - CB Fees: Adjustable $ amount
   - Alert: Editable settings
   - Reset button

2. **Profitability Summary**
   - Revenue ($)
   - Refund Alert ($)
   - Refund CS ($)
   - Chargeback ($)
   - Processing Fees ($)
   - CB Fees ($)
   - Product Cost ($)
   - CPA Cost ($)
   - Alert Cost ($)
   - Reserve ($)
   - Gross Profit ($)

3. **Revenue Summary Card**
   - Total Revenue ($)
   - Gross Profit ($)
   - Gross Margin (%)
   - Trend chart with weekly data

4. **Profitability Analysis Table**
   Columns:
   - Campaign
   - $ Revenue
   - $ Ref Alert
   - $ Ref CS
   - $ CB
   - $ Processing Fees
   - $ CB Fees
   - $ Product Cost
   - $ CPA Cost
   - $ Alert Cost
   - $ Reserve
   - $ Gross Profit
   - % Gross Margin

5. **Profitability Trends Table**
   Time views: Day / Week / Month
   Columns:
   - Time period
   - $ Revenue
   - $ Ref Alert
   - $ Ref CS
   - $ CB
   - $ Processing Fees
   - $ CB Fees
   - $ Product Cost
   - $ CPA Cost
   - $ Alert Cost
   - $ Reserve
   - $ Gross Profit
   - % Gross Margin

---

### 7. LTV (Lifetime Value) Report

**URL:** /reports/ltv

**Description:** See how much revenue each customer brings over time — by month or week cohort.

**Filters:**
- **Date Range:** Last 6 months (default)
- **More Filters**
- **Raw Data Export**

**Sections:**

1. **Customer Lifetime Value Analysis (Cohort Table)**
   Time views: Month / Week
   Columns:
   - First Order At (cohort month/week)
   - New Customers (count)
   - First Order ($)
   - 1 month since first order ($)
   - 2 months since first order ($)
   - 3 months since first order ($)
   - 4 months since first order ($)
   - 5 months since first order ($)
   - 6 months since first order ($)
   - 7 months since first order ($)
   - Average row

2. **Top LTV Performance (60 days)**

   **Top 5 Campaigns:**
   - Campaign name
   - Count
   - LTV value ($)

   **Top 5 Products:**
   - Product name
   - Count
   - LTV value ($)

   **Top 5 Gateways:**
   - Gateway name
   - Count
   - LTV value ($)

---

### 8. Retention Report

**URL:** /reports/retention

**Description:** Track how many customers make it past each billing cycle to evaluate subscription health.

**Filters:**
- **Retention % is based on:** Initial Order Base (dropdown)
- **Date Range:** Preset options + custom
- **Group By:** Campaign → Product (hierarchical)
- **More Filters**
- Download/Export
- Schedule

**Sections:**

1. **How is Rebill Rate Trending Over Time? (Cohort Table)**
   Time views: Day / Week / Month
   Columns:
   - Time period (Month/Week/Day)
   - # Initial
   - Initial %
   - Cycle 1 (%)
   - Cycle 2 (%)
   - Cycle 3 (%)
   - Cycle 4 (%)
   - Cycle 5 (%)
   
2. **Which Campaign & Product had higher retention? (Table)**
   Columns:
   - Campaign
   - # Initial
   - Initial %
   - Cycle 1 (%)
   - Cycle 2 (%)
   - Cycle 3 (%)
   - Cycle 4 (%)
   - Cycle 5 (%)

---

### 9. CB & Refunds Report

**URL:** /reports/cb-refunds

**Description:** Analyze chargebacks and refunds by campaign or product to identify high-risk traffic patterns.

**Filters:**
- **Date Type:** CB/Refund Date
- **Date Range:** Preset options + custom
- **Group By:** Campaign → Product (hierarchical)
- **More Filters**
- Download/Export
- Schedule

**Sections:**

1. **Summary**
   - # Refund Alert (count, $, %)
   - # Refund CS (count, $, %)
   - # CB (count, $, %)

2. **CB & Refund Trends Chart**
   - Trend line over time

3. **Refund Breakdown**
   - Refund Alert (count, $)
   - Refund CS (count, $)
   - Total (count, $)

4. **CB & Refund Table**
   Columns:
   - Product
   - # Approval
   - $ Revenue
   - # CB
   - $ CB
   - CB %
   - # RDR
   - $ RDR
   - RDR %

5. **Trend Table**
   Time views: Day / Week / Month
   Columns:
   - Time period
   - # Approval
   - $ Revenue
   - # CB
   - $ CB
   - CB %
   - # RDR
   - $ RDR
   - RDR %

---

### 10. CB & Refunds Growth Report

**URL:** /reports/cb-refunds-growth

**Description:** Compare the speed of chargebacks and refunds across cohorts to catch early warning signs.

**Filters:**
- **Date Range:** Preset options + custom
- **Details by:** Select option
- **More Filters**
- Download/Export
- Schedule

**Sections:**

1. **CB Development Chart**
   Toggle tabs:
   - CB
   - Refund CS
   - Refund Alert
   - Dispute
   
   Axes:
   - X-axis: # Days to Hit CB (0-150+)
   - Y-axis: CB % (0.0% - 1.4%+)
   
   Markers:
   - % after 30 Days
   - % after 90 Days

---

### 11. Alert Analytics Report

**URL:** /reports/alert-analytics

**Description:** Break down alerts by source and effectiveness to evaluate refund tools and detect coverage gaps.

**Filters:**
- **Date Range:** Preset options + custom
- **Group By:** Acquirer
- Download/Export
- Schedule

**Sections:**

1. **Alert Summary**
   - Total count
   - Breakdown by provider:
     - VERIFI (count, $)
     - RDR (count, $)
     - ETHOCA (count, $)
   - Total (count, $)

2. **Alert Effectiveness**
   - Overall percentage
   - Breakdown:
     - Alert not refunded in CRM (count, $)
     - Alert turned into CB (count, $)
     - Alert with Invalid Orders (count, $)
     - Effective Alert (count, $)
     - RDR refunded in CRM (count, $)
   - Total (count, $)

3. **Alert Duplication (Venn Diagram)**
   - Visual overlap of:
     - RDR
     - Ethoca
     - CDRN

4. **Alert Analysis Table**
   Columns:
   - Acquirer
   - # Alerts
   - Alerts %
   - $ Alert
   - # RDR
   - RDR Effective %
   - # Ethoca
   - Ethoca Effective %
   - # CDRN
   - CDRN Effective %
   - # CB
   - # Refund
   - CB Prevention %

5. **Alert Raw Data Table**
   Columns:
   - Alert ID
   - Alert Date
   - Alert Type
   - Alert Status
   - Order ID
   - Bill Email
   - is_cb
   - is_refund_void
   - Transaction Amount
   - Gateway ID
   - Gateway Alias

---

### 12. Order Details Report

**URL:** /reports/order-details

**Description:** Dive deep into transaction-level data to identify trends and optimize performance.

**Filters:**
- **Date Range:** Preset options + custom
- **More Filters**
- **Search:** Search by Order ID or Email
- Download/Export
- Settings

**Tabs:**
- All Transactions
- Approved
- Declined
- Chargeback
- Refund CS
- Refund Alerts

**Table Columns:**
- Order ID & Email
- Date
- Amount
- Cycle & Attempt
- Gateway & Acquirer
- Decline Reason (visible for declined orders)

**Pagination:** Configurable (e.g., 50 per page)

---

### 13. Routing Insights Report

**URL:** /routing-insights

**Description:** Discover top-performing BINs and Banks to optimize your approval rates.

**Filters:**
- **Time Period:** Last 30 Days
- **Sales Type:** Rebills (dropdown)
- **View Mode:** BIN

**Sections:**

1. **Revenue Impact**
   - Current revenue ($)
   - Projected revenue ($)
   - Impact amount ($)
   - Impact percentage
   - Visual progress bar

2. **BIN Performance Cards**
   Showing BINs by highest traffic
   Each card displays:
   - BIN number
   - Card Network (VISA/MASTERCARD)
   - Priority level (High/Medium/Low)
   - Bank name
   - Potential revenue lift ($)
   - Lift percentage

3. **Why Routing Matters (Info Panel)**
   - Approval rate variance info
   - Soft decline information
   - Best-performing MID benefits

4. **Next Steps (Action Items)**
   1. Share insights with payment operations team
   2. Update MID priorities in payment orchestration platform
   3. Schedule weekly performance reviews

5. **Export/Share**
   - CSV export
   - JSON export

6. **Disclaimer**
   - Usage guidance note

---

## Global Features

### Common UI Elements

1. **Client Selector (Top Left)**
   - Searchable dropdown
   - Lists all available clients

2. **Date Range Picker**
   - Presets: Today, Yesterday, Last 7 days, Last calendar week, Last 4 weeks, Last 3 months, Last 6 months, Month to date
   - Custom date range with dual calendar

3. **Dark/Light Mode Toggle**
   - Switch between themes

4. **Download/Export**
   - Available on most reports

5. **Schedule**
   - Schedule automated report delivery

6. **Refresh Data**
   - Manual data refresh option

7. **Last Updated Timestamp**
   - Shows when data was last updated

### Configuration Section

1. **User Management**
   - Manage user access and permissions

2. **Reporting Setup**
   - Configure report parameters

3. **Integration**
   - Set up data integrations

4. **Schedule**
   - Configure scheduled reports

---

## Data Metrics Glossary

| Metric | Description |
|--------|-------------|
| # Initials | Number of initial/first-time orders |
| # Rebills | Number of recurring/subscription orders |
| # Straight Sales | Number of one-time sales |
| Approval Rate | Percentage of successful transactions |
| Net Approval | Approval rate including retry successes |
| First Attempt Approval | Approval rate excluding retries |
| Gross Revenue | Total revenue before deductions |
| Net Revenue | Revenue after refunds, chargebacks, and costs |
| CB | Chargeback |
| CB % | Chargeback percentage |
| RDR | Rapid Dispute Resolution |
| Ethoca | Ethoca alert service |
| CDRN | Cardholder Dispute Resolution Network |
| VAMP | Visa Acquirer Monitoring Program |
| Refund Alert | Alert-triggered refunds |
| Refund CS | Customer service refunds |
| MID | Merchant Identification Number |
| GW | Gateway |
| BIN | Bank Identification Number |
| MCC | Merchant Category Code |
| AOV | Average Order Value |
| LTV | Lifetime Value |
| CPA | Cost Per Acquisition |

---

## Notes

- Some reports may not be available depending on client configuration
- Bank True Up and Reserve Tracking under Reconciliation may require additional setup
- Routing Setup feature may not be enabled for all clients
- Data availability depends on client data integration status

---

*Document generated on: 2026-01-22T03:49:24.096Z*
*Client analyzed: 88startech*

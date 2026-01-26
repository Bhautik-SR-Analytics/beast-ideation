# Vrio Analytics & Reports Platform Documentation

## Table of Contents
1. [Overview](#overview)
2. [Report Categories](#report-categories)
3. [Report Customization Features](#report-customization-features)
4. [Available Dimensions](#available-dimensions)
5. [Metrics Library](#metrics-library)
6. [Customer Report Possibilities](#customer-report-possibilities)
7. [Date & Time Options](#date--time-options)
8. [Export & Integration Options](#export--integration-options)
9. [Saved Reports & Rulesets](#saved-reports--rulesets)

---

## Overview

Vrio's Analytics module provides a comprehensive reporting system organized into three main sections:

| Tab | Description |
|-----|-------------|
| **Stock Reports** | Pre-built reports organized by 9 business categories |
| **Saved Reports** | User-created custom reports for quick access |
| **Metrics** | Library of 926+ available metrics with detailed descriptions |

---

## Report Categories

### 1. Sales Channel
*Purpose: Track customer purchase journey and advertising effectiveness*

| Report | Description |
|--------|-------------|
| Website Activity | Tracks referrer domains, clicks, page views, orders, conversion rates, and order revenue |
| Ad Spend | Monitors advertising expenditure and ROI |
| Abandons | Analyzes abandoned carts and incomplete purchases |
| New Sales | Tracks new customer transactions, success rates, and customer acquisition |
| Average Order Value (AOV) | Monitors average transaction values |

### 2. Finance
*Purpose: Track revenue performance and financial optimization*

| Report | Description |
|--------|-------------|
| Transactions | Comprehensive view of attempted/successful charges, revenue, adjustments, net revenue, cost, and profit |
| Profit & Loss - Cash Basis | Financial P&L including customers, capture rates, gross revenue, processed amounts, adjustments, profit, and profit margin |
| Dunning | Payment recovery metrics including first attempt success, decline rates, reattempt rates, recovery cycles, and recovered revenue |
| Merchant Capacity | Monitors merchant account capacity and limits |
| Daily Router Capacity | Tracks daily payment routing capacity |

### 3. Lifetime Value
*Purpose: Analyze customer profitability and value over time*

| Report | Description |
|--------|-------------|
| Customer LTV | Tracks customers by acquisition period with charges per customer, retention rates, renewal cycles, discount rates, recurring revenue rates, adjustment rates, and profit margins |
| Sales LTV | Analyzes lifetime value by individual sales |
| Subscriptions LTV | Focuses on subscription-based lifetime value metrics |

### 4. Retention & Subscription
*Purpose: Monitor subscription health and customer retention*

| Report | Description |
|--------|-------------|
| Snapshot | Point-in-time view of subscription status |
| Net Subscriptions | Net subscription changes over time |
| Active Subscriptions | Current active subscribers, subscriptions, scheduled charges, and shipments |
| Cancellations | Tracks subscription cancellation metrics |
| First Renewal | Monitors first renewal success rates |
| 6 Cycle Retention | Retention metrics through 6 billing cycles |
| 36 Cycle Retention | Long-term retention through 36 billing cycles |

### 5. Fulfillment
*Purpose: Track physical product delivery and shipping*

| Report | Description |
|--------|-------------|
| Shipments by Scheduled Date | Shipments organized by scheduled ship date |
| Shipments by Shipped Date | Shipments organized by actual ship date |
| Prepaid Shipments | Tracks prepaid shipping orders |

**Metrics include:** Shipments, Items, Item Quantity, Cancellations, Cancelled Rate, Fulfilled, Fulfillment Rate, Days to Fulfill, Error, Pending Tracking, Tracking Numbers, Shipped

### 6. Forecasting
*Purpose: Forward-looking revenue and inventory projections*

| Report | Description |
|--------|-------------|
| Revenue Forecasting | Projects scheduled recurring revenue, pending revenue, successful recurring, real-time forecasted revenue, new revenue, and forecast accuracy |
| Inventory Forecasting | Projects inventory needs based on scheduled shipments |

### 7. Disputes
*Purpose: Monitor chargebacks and payment disputes*

| Report | Description |
|--------|-------------|
| By Charge Date | Disputes organized by original charge date |
| By Dispute Date | Disputes organized by dispute filing date |

### 8. Operations
*Purpose: System performance, troubleshooting, and monitoring*

| Report | Description |
|--------|-------------|
| Responders | Customer communication and response tracking |
| Order Modifications | Tracks changes made to orders |
| API Requests | Monitors API call activity |
| Gateway | Payment gateway performance metrics |
| Order Fraud Requests | Fraud detection and prevention metrics |

### 9. Other (System Reports)
*Purpose: Technical system monitoring*

| Report | Description |
|--------|-------------|
| Route Log | Payment routing logs |
| Cron Activity | Scheduled task execution logs |
| Events | System event tracking |

---

## Report Customization Features

### Dimension Selection
Each report allows selection of a primary dimension to group data by. Multiple dimensions can be added for cross-tabulated analysis.

### Metric Customization
- **Show All / Hide All**: Bulk toggle all metrics
- **Individual Toggles**: Enable/disable specific metrics
- **Categories**: Summary vs Detail views
- **Types**: Show Counts, Show Percentages, Show Revenue

### Filtering
- **Search**: Filter dimension values
- **Rulesets**: Apply predefined filter sets
- **Date Range**: Flexible date selection

---

## Available Dimensions

### Customer Dimensions
| Dimension | Description |
|-----------|-------------|
| Customer Email | Group by customer email address |
| Customer ID | Group by unique customer identifier |
| Customer Information | Extended customer details |
| Connection | Customer connection/referral source |

### Marketing Dimensions
| Dimension | Description |
|-----------|-------------|
| Ad | Specific advertisement |
| Ad Campaign | Marketing campaign grouping |
| Ad Group | Ad group within campaigns |
| Referrer Base Domain | Traffic source domain |

### Product Dimensions
| Dimension | Description |
|-----------|-------------|
| Item | Product/item name |
| Item Category | Product category grouping |
| Item SKU | Stock keeping unit |
| Discount Code | Applied discount codes |
| Discount Name | Discount promotion names |
| Is Prepaid Offer | Prepaid vs standard offers |
| Is Recurring | One-time vs recurring purchases |

### Payment Dimensions
| Dimension | Description |
|-----------|-------------|
| Card Type | Visa, Mastercard, Amex, etc. |
| Card Category | Credit, Debit, Prepaid |
| Card Country | Card issuing country |
| Card Issuer | Issuing bank |
| Card Last 4 | Last 4 digits of card |
| Is Prepaid Card | Prepaid card indicator |
| Currency | Transaction currency |
| Gateway | Payment gateway used |
| Gateway Transaction ID | Gateway transaction reference |
| Merchant | Merchant account |
| Merchant Descriptor | Statement descriptor |
| Merchant Group | Merchant grouping |
| Merchant MCC Code | Merchant category code |
| Processor | Payment processor |
| Processor Response | Processor response codes |

### Shipping Dimensions
| Dimension | Description |
|-----------|-------------|
| Ship Country | Destination country |

### Transaction Dimensions
| Dimension | Description |
|-----------|-------------|
| Transaction Type | Type of transaction |
| Transaction Created User | User who created transaction |
| Secondary Merchant Category | Secondary categorization |

### Timeframe Dimensions
| Dimension | Description |
|-----------|-------------|
| Order/Transaction/Acquired/Click Date | Specific date |
| Day Of Week | Day of the week (Mon-Sun) |
| Hour | Hour of day (0-23) |
| Month | Calendar month |
| Week | Calendar week |
| Year | Calendar year |

### Tracking Dimensions
| Dimension | Description |
|-----------|-------------|
| Tracking 1-13 | Custom tracking fields for attribution |

---

## Metrics Library

Vrio provides **926+ metrics** organized by report type. Key metric categories include:

### Count Metrics
Customers, Subscribers, Transactions, Orders, Charges, Shipments, Cancellations, Declines, Disputes

### Revenue Metrics
Total Revenue, Net Revenue, Gross Revenue, Recurring Revenue, Order Revenue, Recovered Revenue, Forecasted Revenue

### Rate Metrics
Conversion Rate, Success Rate, Capture Rate, Decline Rate, Retention Rate, Cancel Rate, Fulfillment Rate, Recovery Rate

### Financial Metrics
Profit, Profit Margin, Cost of Revenue, Adjustments, Adjustment Rate

### Time Metrics
Days to Fulfill, Average Recovery Cycle, Average Days to Recover

---

## Customer Report Possibilities

Using Vrio's dimensions and metrics, you can create the following customer-focused reports:

### Customer Acquisition Reports
| Report Type | Dimensions to Use | Key Metrics |
|-------------|-------------------|-------------|
| Customer Source Analysis | Referrer Base Domain, Ad Campaign, Ad Group | New Customers, Conversion Rate, Customer Acquisition Cost |
| Acquisition Cohort Analysis | Acquired Week/Month/Year | Customers, LTV, Retention Rate |
| Marketing Channel Performance | Ad, Ad Campaign | Total Customers, Successful Customer Rate, Revenue |

### Customer Behavior Reports
| Report Type | Dimensions to Use | Key Metrics |
|-------------|-------------------|-------------|
| Purchase Pattern Analysis | Order Created Day Of Week, Hour | Orders, Average Order Value, Conversion Rate |
| Product Preference Report | Item, Item Category, Customer Email | Orders, Revenue, Items Purchased |
| Discount Usage Report | Discount Code, Discount Name | Customers, Discount Rate, Revenue Impact |
| Geographic Analysis | Ship Country, Card Country | Customers, Orders, Revenue by Region |

### Customer Value Reports
| Report Type | Dimensions to Use | Key Metrics |
|-------------|-------------------|-------------|
| Customer LTV by Cohort | Acquired Week/Month | Customer LTV, Charges Per Customer, Profit Margin |
| High-Value Customer Identification | Customer Email, Customer ID | Total Revenue, Order Count, LTV |
| Customer Segmentation by Payment | Card Type, Card Category | Customers, Success Rate, Average Transaction |

### Customer Retention Reports
| Report Type | Dimensions to Use | Key Metrics |
|-------------|-------------------|-------------|
| Retention Cohort Analysis | Acquired Month | 6 Cycle Retention, 36 Cycle Retention |
| Subscription Health Dashboard | Overall, Acquired Week | Active Subscriptions, Cancellations, Net Subscriptions |
| Churn Analysis | Cancellation Date, Customer Information | Cancellations, Active Cancel Rate, Passive Cancel Rate |
| First Renewal Success | Acquired Week | First Renewal Rate, Customers Retained |

### Customer Transaction Reports
| Report Type | Dimensions to Use | Key Metrics |
|-------------|-------------------|-------------|
| Transaction Success by Customer Segment | Customer Email, Card Type | Attempted Charges, Successful Charges, Success Rate |
| Payment Method Analysis | Card Type, Card Issuer, Gateway | Transactions, Success Rate, Decline Rate |
| Recurring vs One-Time Analysis | Is Recurring | Customers, Revenue, Retention |

### Customer Recovery Reports
| Report Type | Dimensions to Use | Key Metrics |
|-------------|-------------------|-------------|
| Dunning Effectiveness by Customer | Customer Email, Original Attempt Date | Decline Rate, Reattempt Rate, Recovered Rate, Recovered Revenue |
| Payment Recovery Trends | Transaction Date, Card Type | Recovery Cycle, Days to Recover, Recovered Revenue |

### Multi-Dimensional Customer Reports
| Report Type | Dimensions (Multiple) | Insights |
|-------------|----------------------|----------|
| Campaign > Product > Customer | Ad Campaign + Item + Customer Email | Which campaigns drive which product purchases by customer |
| Source > Time > Value | Referrer + Acquired Month + Customer LTV | Traffic source quality over time |
| Geography > Payment > Success | Ship Country + Card Type + Gateway | Payment success patterns by region |

---

## Date & Time Options

### Preset Date Ranges
- Today
- Yesterday
- Tomorrow
- Last 7 Days
- Last 14 Days
- Last 30 Days
- Last 60 Days
- Last 90 Days
- This Week
- This Month

### Custom Date Selection
- Interactive calendar with start and end date selection
- Support for both date ranges (most reports) and single dates (snapshot reports)

---

## Export & Integration Options

### From More Options Menu
| Option | Description |
|--------|-------------|
| Export Report | Download report data |
| Create Schedule | Set up automated report delivery |
| Create Alert | Configure threshold-based notifications |
| Create Dashboard Widget | Add report visualization to dashboard |
| Create Dashboard Counter | Add numeric counter to dashboard |

### Save As
- Save customized report configurations for reuse
- Saved reports appear in the Saved Reports tab

---

## Saved Reports & Rulesets

### Saved Reports Structure
Saved reports are organized by category and can include:
- Custom dimension configurations
- Specific metric selections
- Predefined date ranges
- Applied rulesets/filters

### Example Saved Report Categories
- Sales Channel (custom sales analyses)
- Operations (daily monitoring reports)
- Cycle Approvals (BIN and Card Issuer approval tracking)
- Finance (transaction drilldowns)

### Rulesets
Predefined filter sets for quick data segmentation:
- **BIN Alert BINS**: Filter by flagged BIN numbers
- **CPAs**: Cost Per Acquisition tracking
- **Prepaid**: Prepaid card transactions only
- **Internal**: Internal transaction filtering
- **Custom rulesets**: User-defined filter combinations

---

## Best Practices

1. **Start with Stock Reports** - Use pre-built reports as templates before creating custom versions

2. **Use Multi-Dimensional Analysis** - Add secondary dimensions to uncover deeper insights

3. **Save Frequently Used Configurations** - Use Save As to create quick-access reports

4. **Leverage Rulesets** - Create rulesets for commonly used filter combinations

5. **Schedule Critical Reports** - Automate delivery of key metrics to stakeholders

6. **Set Up Alerts** - Configure notifications for threshold breaches (declines, cancellations, etc.)

7. **Build Dashboard Widgets** - Create visual dashboards for real-time monitoring

---

*Document Generated: January 2026*
*Platform: Vrio Analytics Module*
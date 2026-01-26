# Beast Insights — Reports From Scratch

## The Problem with Traditional Payment Analytics

Most payment analytics platforms (including the current Beast Insights) are organized by **data type**: "Here's your sales data. Here's your chargeback data. Here's your approval data."

That forces the user to:
1. Open 5 different reports
2. Cross-reference numbers mentally
3. Figure out what's wrong themselves
4. Go to a separate system to take action

**The result:** Users check reports once a day (or not at all), miss problems until they become emergencies, and never feel like the platform is truly helping them run their business.

---

## The Design Principle: Problem → Cause → Action

Every report should answer three questions:

```
1. WHAT is happening?     → Clear metrics with context (vs yesterday, vs last week, vs target)
2. WHY is it happening?   → Automatic drill-down to root cause
3. WHAT do I do about it? → Specific, actionable next steps
```

If a report only answers #1, it's a dashboard. If it answers all three, it's a decision-making tool.

---

## Who Uses This Platform?

### User Personas

| Persona | Role | Checks Platform | Cares Most About | Action They Take |
|---|---|---|---|---|
| **Operations Manager** | Daily operations, fire-fighting | Multiple times/day | MID health, approval rates, decline spikes | Pause MIDs, shift volume, contact processor |
| **Business Owner / CEO** | Strategic decisions | Once/day or weekly | Revenue, profitability, growth trends | Adjust campaigns, budget allocation |
| **Media Buyer / Marketing** | Traffic source management | Daily | Campaign ROI, traffic quality, conversion | Pause/scale affiliates, adjust CPA bids |
| **Finance / Accounting** | P&L, reconciliation | Weekly/monthly | Revenue, costs, margins, reserves | Adjust pricing, fee negotiations |
| **Compliance / Risk** | Card network programs | Weekly | CB rates by MID, VAMP/MC thresholds | Shut down risky traffic, add alert coverage |

---

## Report Structure: Organized by Intent, Not Data Type

### Navigation Structure

```
MONITOR
  ├── Command Center (home)
  ├── Live Pulse

GROW
  ├── Revenue Intelligence
  ├── Customer Economics
  ├── Traffic & Acquisition Quality

OPTIMIZE
  ├── Approval Optimization
  ├── Decline Intelligence & Recovery
  ├── Routing Performance

PROTECT
  ├── Risk Control Center
  ├── Dispute & Alert Management
  ├── MID Health Monitor

PROFIT
  ├── Profitability Analysis
  ├── Cashflow & Reserves

EXPLORE
  ├── Transaction Explorer
  ├── Custom Report Builder
```

---

## Report 1: Command Center (Home)

**Purpose:** "Is everything OK? What needs my attention right now?"

This is NOT a dashboard of numbers. It's an **action queue** — things that need attention, sorted by urgency.

### Layout

```
┌─────────────────────────────────────────────────────────────┐
│  TODAY'S PULSE                                    10:32 AM   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐          │
│  │ Revenue  │ │ Orders  │ │ Approval│ │ Net Rev  │          │
│  │ $14,230  │ │ 342     │ │  74.2%  │ │ $12,100  │          │
│  │ ▲ 12%    │ │ ▲ 8%    │ │ ▼ 2.1%  │ │ ▲ 15%   │          │
│  │ vs 7d avg│ │ vs 7d   │ │ vs 7d   │ │ vs 7d    │          │
│  └─────────┘ └─────────┘ └─────────┘ └──────────┘          │
│                                                              │
│  ⚡ NEEDS ATTENTION (3)                                      │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ 🔴 CRITICAL: GW-262 CB rate 3.5% — above Visa 1.5%     ││
│  │    → 60% of CBs from Campaign "Summer Trial"            ││
│  │    → [View MID] [Pause Campaign] [See CB Details]       ││
│  ├──────────────────────────────────────────────────────────┤│
│  │ 🟡 WARNING: Approval rate dropped 5% since yesterday    ││
│  │    → Gateway 307 decline rate spiked to 65%             ││
│  │    → Top decline: "Issuer Decline" (was 40%, now 58%)   ││
│  │    → [View Declines] [Check Gateway] [View Routing]     ││
│  ├──────────────────────────────────────────────────────────┤│
│  │ 🟡 WARNING: GW-165 at 85% monthly capacity ($46k used)  ││
│  │    → Projected to hit cap in 4 days                     ││
│  │    → [View MID] [Adjust Routing]                        ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  📊 TODAY'S REVENUE (vs 7-day average)                       │
│  [====hourly chart showing today vs avg================]     │
│                                                              │
│  📈 KEY METRICS (Last 7 Days)                                │
│  ┌───────────────────────────────────────────────────────── │
│  │ Date  │ Revenue │ Orders │ Appr% │ CB# │ CB% │ Net Rev │ │
│  │ Today │ $14.2k  │  342   │ 74.2% │  2  │ 1.2%│ $12.1k  │ │
│  │ Jan 25│ $18.1k  │  412   │ 76.3% │  3  │ 0.9%│ $15.8k  │ │
│  │ ...   │         │        │       │     │     │         │ │
│  └───────────────────────────────────────────────────────── │
│                                                              │
│  🏥 NETWORK HEALTH SNAPSHOT                                  │
│  Visa:  VAMP 0.8% ✅  |  Alerts 0.4% ✅  |  Declines 62% ⚠️│
│  MC:    CB 1.1% ⚠️     |  Alerts 0.3% ✅  |  Declines 58% ⚠️│
└─────────────────────────────────────────────────────────────┘
```

### What Makes This Different

- **"Needs Attention" section** is auto-generated from rules: CB rate thresholds, approval rate drops, capacity warnings, decline spikes. Users see problems without hunting for them.
- **Every alert has context** — not just "CB rate is high" but "60% of CBs from Campaign X". The cause is surfaced automatically.
- **Every alert has action links** — one click to view details, pause the problem source, or adjust routing. Not just awareness, but resolution.

### Key Metrics

| Metric | Comparison | Alert Threshold |
|---|---|---|
| Today's Revenue | vs 7-day avg | < 70% of avg |
| Today's Orders | vs 7-day avg | < 70% of avg |
| Approval Rate | vs 7-day avg | Drop > 3% |
| Net Revenue | vs 7-day avg | < 70% of avg |
| Visa VAMP % | vs threshold | > 0.9% (warning), > 1.5% (critical) |
| MC CB % | vs threshold | > 1.0% (warning), > 1.5% (critical) |
| MID Capacity | vs cap | > 80% (warning), > 95% (critical) |
| Decline Rate | vs 7-day avg | Increase > 5% |

### Alerts Engine (Automatic Problem Detection)

These run in the background and feed the "Needs Attention" section:

| Alert Rule | Trigger | Auto-Diagnosis |
|---|---|---|
| CB Threshold Breach | Any MID CB% > Visa/MC threshold | Top campaigns + affiliates contributing |
| Approval Rate Drop | Rate drops > 3% vs 7-day avg | Top decline reasons + affected gateways |
| Revenue Drop | Revenue < 70% of 7-day avg | Affected campaigns, products, gateways |
| MID Capacity Warning | Usage > 80% of monthly cap | Days until cap, suggested redistribution |
| Decline Spike | Any decline reason spikes > 2x | Specific gateway + BIN + decline reason |
| Refund Spike | Refund rate > 2x normal | Campaign + product causing refunds |
| Alert Coverage Gap | MID has CB but no alert service | List of unprotected MIDs |
| New Fraud Pattern | Fraud flags spike from specific BIN/country | BIN range + country + IP pattern |

---

## Report 2: Live Pulse

**Purpose:** "What's happening right now, hour by hour?"

### Key Metrics (Real-time, updated every 5 min)

- Revenue by hour (today vs 7-day avg line)
- Orders by hour (initials vs rebills stacked)
- Approval rate by hour
- Running total: revenue, orders, approvals

### Why It Exists Separately from Command Center

Command Center is about **problems and actions**. Live Pulse is for people who want to watch the business run in real-time — useful during high-traffic periods, after making routing changes, or during campaign launches.

---

## Report 3: Revenue Intelligence

**Purpose:** "Where is my revenue coming from, and how is it trending?"

### This Replaces: Sales Report

### Key Difference from Current

Current Sales report shows numbers by campaign/product. The new version answers:
- "Which campaigns are **growing** vs **declining**?"
- "What's my **revenue concentration risk**?" (if 1 campaign = 80% revenue)
- "What's the **revenue per customer** by source?"

### Layout

**Section 1: Revenue Summary**
| Metric | Value | vs Last Period | Trend (sparkline) |
|---|---|---|---|
| Gross Revenue | $127,450 | +12% | ↗ |
| Net Revenue | $108,330 | +15% | ↗ |
| Initials Revenue | $45,200 | +8% | ↗ |
| Rebills Revenue | $72,100 | +18% | ↗ |
| Straight Sales Revenue | $10,150 | -5% | ↘ |
| Avg Order Value (Initials) | $47.20 | +2% | → |
| Avg Order Value (Rebills) | $32.80 | +1% | → |

**Section 2: Revenue Trend**
- Line chart: Revenue over time (daily/weekly/monthly)
- Toggle: Gross vs Net, by Sales Type
- Comparison overlay: vs previous period

**Section 3: Revenue Breakdown (Group By)**
- Default group: Campaign
- Table with: Campaign, # Initials, $ Initials, # Rebills, $ Rebills, $ Total, % of Total, Trend
- **NEW: "% of Total" column** — instantly shows revenue concentration
- **NEW: Trend sparkline** — see at a glance which campaigns are growing/declining

**Section 4: Revenue Quality Score** (NEW)
Each campaign/product gets a quality score:
```
Quality Score = weighted average of:
  - Approval Rate (higher = better)
  - CB Rate (lower = better)
  - Cancel Rate (lower = better)
  - Rebill Retention (higher = better)
```

This answers: "Which campaigns bring QUALITY revenue, not just volume?"

### Key Metrics for Revenue Intelligence

| Metric | Why It Matters |
|---|---|
| Gross Revenue | Total money in |
| Net Revenue | Money after refunds + chargebacks |
| Revenue by Sales Type | Initial vs recurring revenue split |
| AOV (Initial / Rebill) | Price point optimization |
| Revenue per Customer | Customer value efficiency |
| Revenue Concentration % | Risk — if top campaign dies, what happens? |
| Revenue Quality Score | Which sources bring sustainable revenue |
| Revenue Growth Rate | Trend direction |
| Revenue vs Capacity | Are you leaving money on table due to MID caps? |

---

## Report 4: Customer Economics

**Purpose:** "Are my customers profitable, and how long do they stay?"

### This Replaces: LTV + Retention (combined into one coherent story)

### Key Difference from Current

Current LTV and Retention are separate reports. But they answer the same question: "Is my subscription business healthy?" Combining them tells the full story.

### Layout

**Section 1: Customer Health Summary**
| Metric | Value | Trend |
|---|---|---|
| New Customers (this period) | 1,240 | ↗ +8% |
| Active Subscribers | 4,520 | ↗ +3% |
| Avg Revenue per Customer (30d) | $67.40 | → |
| Avg Revenue per Customer (60d) | $94.20 | ↗ |
| Avg Revenue per Customer (90d) | $112.50 | ↗ |
| Cycle 1 Retention | 42% | ↘ -2% |
| Cycle 3 Retention | 28% | → |
| Cycle 6 Retention | 18% | ↗ +1% |

**Section 2: Retention Heatmap** (NEW)
Instead of a plain table of percentages, show a **color-coded heatmap**:
- Rows = cohort month
- Columns = billing cycle 0, 1, 2, 3, 4, 5, 6+
- Color = green (good retention) → yellow → red (bad)
- **Instantly spot which cohorts dropped off** and which held strong

**Section 3: LTV Cohort Matrix**
Standard cohort LTV table but with:
- **Goal line**: show target LTV at each period
- **Below target highlighting**: red cells where LTV is below target
- **Best/worst cohort callout**: "Jun 2025 cohort is 20% above average — driven by Campaign X"

**Section 4: Customer Lifecycle Funnel** (NEW)
```
Initial Purchase    → 1,240 customers  (100%)
  ↓
Cycle 1 Rebill      →   521 customers  (42%)  — Lost 719 (58%)
  ↓                                              Top loss reasons:
Cycle 2 Rebill      →   347 customers  (28%)    - Cancel: 45%
  ↓                                              - Decline: 35%
Cycle 3 Rebill      →   248 customers  (20%)    - Chargeback: 12%
  ↓                                              - Refund: 8%
Cycle 6+ (Loyal)    →   149 customers  (12%)
```

**This funnel is extremely powerful** because it shows WHERE customers leave and WHY. A user sees "35% of Cycle 1 losses are declines" and immediately knows: "I need better retry logic for first rebills."

**Section 5: Customer Value by Source**
| Campaign | New Customers | 30d LTV | 60d LTV | 90d LTV | Retention C1 | Quality Score |
|---|---|---|---|---|---|---|
| Campaign A | 450 | $72 | $105 | $128 | 48% | 85/100 |
| Campaign B | 320 | $54 | $71 | $82 | 35% | 62/100 |

### Key Metrics for Customer Economics

| Metric | Why It Matters |
|---|---|
| LTV at 30/60/90/180 days | How much a customer is worth over time |
| Retention by Cycle | Where in the lifecycle customers leave |
| Churn Reason Split | WHY they leave (cancel vs decline vs CB vs refund) |
| Customer Acquisition Cost (CPA) | Cost to acquire |
| LTV:CPA Ratio | Profitability per customer (should be > 3:1) |
| Time to Payback | How many days until CPA is recovered |
| Active Subscriber Count | Total recurring base |
| Net Subscriber Growth | New subscribers minus cancels |

---

## Report 5: Traffic & Acquisition Quality

**Purpose:** "Which traffic sources are worth paying for?"

### This Is NEW (no current equivalent — data exists but isn't surfaced this way)

### Key Difference

Currently, affiliates are just a filter dimension. This report makes traffic quality the PRIMARY focus.

### Layout

**Section 1: Traffic Quality Scorecard**
| Affiliate | Initials | Approval % | C1 Retention | CB % | Cancel % | 60d LTV | CPA | LTV:CPA | Score |
|---|---|---|---|---|---|---|---|---|---|
| aff_23 | 450 | 78% | 45% | 0.8% | 12% | $94 | $45 | 2.1x | ⚠️ 68 |
| aff_31 | 320 | 82% | 52% | 0.3% | 8% | $112 | $40 | 2.8x | ✅ 85 |
| aff_45 | 180 | 65% | 28% | 2.1% | 22% | $48 | $50 | 0.96x | 🔴 32 |

**Section 2: Fraud & Risk by Source** (NEW)
- Which affiliates have highest CB rates?
- Which affiliates have highest cancel rates in first 3 days? (indicates forced sales)
- Which affiliates have mismatched BIN countries? (potential fraud signal)

**Section 3: Affiliate Trend**
- Was this affiliate always bad, or did quality recently change?
- Trend chart per affiliate over last 90 days

### Key Metrics for Traffic Quality

| Metric | Why It Matters |
|---|---|
| Initial Approval % by source | Low approval = bad traffic quality |
| Cancel % (0-3 days) by source | High early cancel = deceptive marketing |
| CB % by source | High CB = fraud or misleading offers |
| C1 Retention by source | Low retention = wrong customer expectation |
| LTV by source | Long-term value of acquired customers |
| LTV:CPA Ratio | Is this source profitable? |
| Fraud flag rate by source | Direct fraud indicators |

---

## Report 6: Approval Optimization

**Purpose:** "How do I approve more transactions?"

### This Replaces: Approval % Report

### Key Difference from Current

Current report shows approval rates by various dimensions. The new version answers:
- "WHERE am I losing the most approvals?" (quantified in $ lost)
- "Which specific changes would recover the most revenue?"
- "How does each gateway perform for each BIN/bank?"

### Layout

**Section 1: Approval Summary**
| Metric | Initials | Rebills | Total |
|---|---|---|---|
| Attempts | 5,200 | 8,400 | 13,600 |
| Approvals | 3,900 | 5,880 | 9,780 |
| Approval Rate | 75.0% | 70.0% | 71.9% |
| **$ Lost to Declines** | **$61,100** | **$83,200** | **$144,300** |

**"$ Lost to Declines" is the most important number.** It turns an abstract percentage into real money. When the user sees "$144k lost this week to declines," it creates urgency.

**Section 2: Approval Rate Trend**
- Line chart with trend
- Toggles: First Attempt / Net (After Retries) / All
- Split by: Initials / Rebills

**Section 3: Decline Root Cause Analysis** (NEW)
```
$144,300 lost to declines this week
├── Issuer Decline:         $72,150 (50%) — spread across all gateways
├── Insufficient Funds:     $28,860 (20%) — mostly rebills cycle 2+
├── Fraudulent:             $14,430 (10%) — concentrated in aff_45
├── Customer Account Issue: $11,544 (8%)  — mostly expired cards
├── Invalid Card:           $8,658  (6%)  — possible data entry issue
├── Gateway/Network Issue:  $4,329  (3%)  — Gateway 307 on Jan 24
├── CVV Mismatch:           $2,886  (2%)  — campaign "Direct Trial"
└── Other:                  $1,443  (1%)
```

Each decline reason links to: "What can I do about this?"
- Issuer Decline → "Try different MIDs for these BINs" → [View Routing Suggestions]
- Insufficient Funds → "Retry with lower amount or different date" → [View Retry Settings]
- Fraudulent → "Block this traffic source" → [View Affiliate Details]

**Section 4: Approval by Dimension Tables**
- By Product, By Campaign, By Gateway, By Bank/BIN
- Each with: Attempts, Approvals, Rate, **$ Impact** (how much revenue this entity's approval rate costs/gains vs average)

**Section 5: Gateway x BIN Performance Matrix** (NEW)
```
         | GW-7 (78%) | GW-88 (72%) | GW-100 (80%) | GW-165 (65%)
---------+------------+-------------+--------------+-------------
BIN 4147 |    82%     |    71%      |     85%      |    60%
BIN 5469 |    75%     |    78%      |     72%      |    68%
BIN 4023 |    68%     |    65%      |     81%      |    55%
```

**This matrix is gold for routing optimization.** Users can immediately see: "BIN 4023 should go to GW-100 (81%) instead of GW-7 (68%)."

---

## Report 7: Decline Intelligence & Recovery

**Purpose:** "Why are transactions failing and how do I recover them?"

### This Replaces: Decline Recovery Report

### Key Difference from Current

Current report shows recovery rates. The new version focuses on:
- Total **revenue impact** of declines
- **Which declines are recoverable** vs permanent
- **Optimal retry strategy** based on historical data

### Layout

**Section 1: Decline & Recovery Summary**
| Metric | Value | Impact |
|---|---|---|
| Total Declines | 3,820 | $144,300 lost |
| Recoverable Declines | 1,910 (50%) | $72,150 potential |
| Actually Recovered | 956 (25%) | $36,075 recovered |
| **Recovery Gap** | **954 orders** | **$36,075 still recoverable** |
| Recovery Rate | 50.1% | |
| Avg Days to Recover | 2.3 days | |

**"Recovery Gap"** — the difference between what you COULD recover and what you DID recover. This is the #1 actionable number.

**Section 2: Recovery by Decline Reason**
| Decline Group | # Declines | Recoverable? | # Recovered | Recovery Rate | $ Gap |
|---|---|---|---|---|---|
| Insufficient Funds | 980 | ✅ High | 490 | 50% | $18,620 |
| Issuer Decline | 1,200 | ⚠️ Medium | 360 | 30% | $31,920 |
| Expired Card | 120 | ✅ High (updater) | 84 | 70% | $1,368 |
| Gateway/Network | 280 | ✅ High (retry diff GW) | 196 | 70% | $3,192 |
| Fraudulent | 450 | 🚫 Not recoverable | 0 | 0% | - |
| CVV Mismatch | 90 | 🚫 Not recoverable | 0 | 0% | - |

**Section 3: Optimal Retry Window** (NEW)
Based on historical data, show when retries are most likely to succeed:
```
Insufficient Funds:
  Retry after 1 day:  45% success
  Retry after 3 days: 52% success  ← Best window
  Retry after 7 days: 38% success

Issuer Decline:
  Retry after 1 day:  22% success
  Retry after 3 days: 28% success
  Retry after 7 days: 31% success  ← Best window
```

**Section 4: Recovery by Billing Cycle** (NEW)
| Cycle | # Declines | Recovery Rate | Insight |
|---|---|---|---|
| Cycle 1 | 1,200 | 55% | First rebill — highest recovery potential |
| Cycle 2 | 800 | 42% | |
| Cycle 3 | 520 | 35% | |
| Cycle 6+ | 300 | 20% | Long-term subscribers rarely recover |

---

## Report 8: Routing Performance

**Purpose:** "Is each transaction going to the best possible gateway?"

### This Replaces: Routing Insights

### Key Difference from Current

Current routing report shows BIN-level data. The new version:
- Shows **actual $ revenue impact** of suboptimal routing
- Provides **specific routing recommendations** per BIN/bank
- Tracks **routing changes over time** (did the change you made last week actually help?)

### Layout

**Section 1: Routing Impact Summary**
```
Current Revenue:     $127,450
Optimal Revenue:     $141,200   (if every transaction went to best gateway)
Revenue Gap:         $13,750    (10.8% lift available)
```

**Section 2: Top Routing Opportunities**
| BIN | Bank | Current GW | Current Rate | Best GW | Best Rate | $ Impact | Priority |
|---|---|---|---|---|---|---|---|
| 414720 | Chase | GW-88 | 68% | GW-100 | 85% | $4,200 | 🔴 High |
| 546680 | Citi | GW-7 | 71% | GW-307 | 82% | $3,100 | 🔴 High |
| 409758 | BofA | GW-165 | 60% | GW-315 | 78% | $2,800 | 🟡 Medium |

**Section 3: Gateway Performance Comparison**
For each gateway: approval rate by initial/rebill, volume, capacity, health status. Side-by-side comparison.

**Section 4: Routing Change Tracker** (NEW)
"You changed BIN 414720 from GW-88 to GW-100 on Jan 15. Here's the result:"
- Before: 68% approval, $12k revenue/week
- After: 83% approval, $14.8k revenue/week
- **Impact: +$2.8k/week (+23%)**

---

## Report 9: Risk Control Center

**Purpose:** "Am I going to lose a MID? Am I in compliance?"

### This Replaces: MID Performance + part of CB & Refunds

### Key Difference from Current

Current MID Performance is a monthly snapshot. The new version:
- Shows **real-time risk scoring** not just monthly
- **Projects forward** — "at current rate, MID X will breach threshold in 12 days"
- Groups MIDs by **urgency** not alphabetically

### Layout

**Section 1: Risk Dashboard**
```
352 Total MIDs
├── 🔴 Critical (3)     — Requires immediate action
├── 🟡 At-Risk (8)      — Approaching thresholds
├── ✅ Healthy (289)     — Normal operations
└── ⚪ Inactive (52)     — No recent volume
```

**Section 2: Critical & At-Risk MIDs (auto-sorted by urgency)**
| MID | Acquirer | Risk | Issue | Visa CB% | MC CB% | Days to Breach | Action |
|---|---|---|---|---|---|---|---|
| GW-262 | Bank A | 🔴 | CB above Visa threshold | 3.5% | 1.2% | BREACHED | [Pause] [View CBs] |
| GW-165 | Bank B | 🔴 | Capacity 95% | 1.1% | 0.8% | 2 days | [Redistribute] |
| GW-307 | Bank C | 🟡 | CB trending up | 1.2% | 0.7% | 14 days | [Monitor] [View] |

**"Days to Breach"** — projects current CB rate forward. If CB rate trend continues, when will this MID cross the threshold? This is an early warning system.

**Section 3: Card Network Compliance**
| Program | Current % | Threshold | Status | MIDs Affected |
|---|---|---|---|---|
| Visa VAMP (Partial) | 0.82% | 0.90% | ⚠️ Near | GW-262, GW-165 |
| Visa VAMP (Full) | 0.82% | 1.50% | ✅ OK | - |
| MC EFM | 1.1% | 1.50% | ✅ OK | - |
| MC ECM | 1.1% | 2.00% | ✅ OK | - |

**Section 4: MID Portfolio Table**
Full table with all MIDs, sortable by any column, filterable by health status.

### Key Metrics for Risk

| Metric | Why It Matters |
|---|---|
| CB Rate (by network) | Direct compliance requirement |
| Alert Rate | Must have alert coverage for active MIDs |
| Days to Breach | Predictive early warning |
| CB Source Concentration | Which campaign/affiliate is causing CBs |
| MID Age | Newer MIDs are higher risk for shutdowns |
| Capacity Utilization % | Avoid hitting caps |
| Decline Rate by MID | High decline rate can indicate processor issues |

---

## Report 10: Dispute & Alert Management

**Purpose:** "How effective are my chargeback prevention tools?"

### This Replaces: CB & Refunds + CB Growth + Alert Analytics (combined)

### Key Difference

Currently three separate reports. Combined into one story:
1. What disputes do I have?
2. Are my alert tools preventing them?
3. How fast are disputes happening?

### Layout

**Section 1: Dispute Summary**
| Type | Count | $ Amount | % of Approvals | vs Last Period |
|---|---|---|---|---|
| Chargebacks | 145 | $12,400 | 0.9% | ↗ +0.1% |
| Refund (Alert) | 89 | $7,200 | 0.6% | → |
| Refund (CS) | 67 | $5,100 | 0.4% | ↘ -0.1% |
| Total Disputes | 301 | $24,700 | 1.9% | ↗ |

**Section 2: Alert Effectiveness**
```
Total Alerts Received: 156
├── Effective (prevented CB):     112 (72%)
├── Turned into CB anyway:         18 (12%)
├── Invalid (no matching order):    14 (9%)
└── Already refunded in CRM:       12 (8%)

Alert Coverage:
├── RDR:    Active on 45 MIDs → 78% effective
├── Ethoca: Active on 38 MIDs → 65% effective
├── CDRN:   Active on 22 MIDs → 71% effective
└── ⚠️ 12 active MIDs have NO alert coverage
```

**Section 3: CB Growth Curve** (Critical for compliance)
Chart showing: After a sale is made, how quickly do CBs accumulate?
- X-axis: Days since sale (0-180)
- Y-axis: Cumulative CB %
- Lines: by month cohort
- Markers: "30-day CB%" and "90-day CB%"

**Purpose:** If January cohort hits 1% CB at 30 days, but February cohort hits 1% at 20 days, something changed and needs investigation.

**Section 4: CB Root Cause**
| Source | # CBs | CB % | Top Reason | Action |
|---|---|---|---|---|
| Campaign "Trial Offer" | 52 | 2.1% | "Not as described" | Review offer page |
| Affiliate aff_45 | 38 | 3.5% | "Unauthorized" | Pause affiliate |
| Product "Premium Bundle" | 28 | 1.8% | "Recurring not expected" | Add disclosure |

---

## Report 11: Profitability Analysis

**Purpose:** "Am I actually making money?"

### This Replaces: Profitability Report

### Key Difference from Current

Current profitability uses sliders for fee inputs. The new version:
- Stores **actual fee rates per gateway** (not a global slider)
- Shows profitability **per customer cohort**, not just per period
- Answers: "Which campaigns are profitable AFTER all costs?"

### Layout

**Section 1: P&L Summary**
```
Revenue:           $127,450
- Refunds (Alert):  -$7,200
- Refunds (CS):     -$5,100
- Chargebacks:     -$12,400
= Net Revenue:     $102,750

- Processing Fees:  -$4,460  (3.5%)
- CB Fees:          -$3,625  (145 × $25)
- Product Cost:     -$8,400
- CPA Cost:        -$15,200
- Alert Cost:       -$1,560
- Reserve:          -$6,373  (5%)
= Gross Profit:    $63,132
  Gross Margin:     49.5%
```

**Section 2: Profitability by Campaign**
| Campaign | Revenue | All Costs | Gross Profit | Margin | LTV:CPA |
|---|---|---|---|---|---|
| Campaign A | $45,200 | $22,100 | $23,100 | 51% | 2.8x |
| Campaign B | $32,100 | $19,800 | $12,300 | 38% | 1.6x |
| Campaign C | $28,700 | $28,200 | **$500** | **1.7%** | **1.02x** |

Campaign C is barely profitable — user immediately knows this needs attention.

**Section 3: Profitability Trend**
- Weekly P&L bar chart showing revenue vs costs over time
- Margin % trend line overlay

**Section 4: Cost Breakdown**
- Where is money going? Processing fees vs CB fees vs CPA vs product cost
- Pie/donut chart with drill-down

### Key Metrics for Profitability

| Metric | Why It Matters |
|---|---|
| Gross Revenue | Top line |
| Net Revenue | After disputes |
| Gross Profit | After all costs |
| Gross Margin % | Efficiency |
| LTV:CPA Ratio | Per-customer profitability |
| Revenue per MID | MID-level economics |
| Processing cost % | Fee efficiency |
| CB cost per month | Dispute costs |

---

## Report 12: Transaction Explorer

**Purpose:** "Let me look up specific transactions and find patterns."

### This Replaces: Order Details

### Key Difference

Current Order Details is a basic data grid. The new version:
- **Smart search**: search by Order ID, Email, or even partial matches
- **Transaction timeline**: click an order, see its full lifecycle (approved → rebilled → refunded → CB)
- **Bulk analysis**: select a set of transactions and get aggregate stats

### Layout

**Section 1: Search & Filter Bar**
- Search: Order ID, Email, Customer Name
- Tabs: All | Approved | Declined | Chargebacks | Refunds | Alerts
- Filters: Date, Campaign, Product, Gateway, Amount range, Cycle, Affiliate

**Section 2: Transaction Table**
Standard data grid with all columns, sortable, paginated.

**Section 3: Transaction Detail Panel** (NEW - slides in on row click)
```
Order #12345 — John Doe (john@email.com)
Timeline:
  Jan 10: Initial purchase — $49.99 — GW-88 — ✅ Approved
  Feb 10: Cycle 1 rebill — $29.99 — GW-88 — ✅ Approved
  Mar 10: Cycle 2 rebill — $29.99 — GW-88 — ❌ Declined (Insufficient Funds)
  Mar 11: Retry #1 — $29.99 — GW-100 — ❌ Declined (Issuer Decline)
  Mar 13: Retry #2 — $29.99 — GW-307 — ✅ Recovered
  Apr 10: Cycle 3 rebill — $29.99 — GW-307 — ❌ Declined
  Apr 12: Customer cancelled

Customer Total: $139.96 revenue, 4 transactions, 2 declines, LTV: $139.96
```

This is incredibly useful for customer service and for understanding individual customer journeys.

---

## Report 13: Custom Report Builder

**Purpose:** "Let me build any report I want from available data."

This is the VRIO-style experience where users pick:
1. **Data source** (Order Summary, Decline Recovery, Cohort, etc.)
2. **Dimensions** (Campaign, Product, Gateway, Date, BIN, Affiliate, etc.)
3. **Metrics** (from the full catalog of 100+)
4. **Chart type** (table, line, bar, pivot, etc.)
5. **Filters and toggles**
6. **Save and share**

---

## Metrics Master Catalog

### Tier 1: Core Metrics (Every User Needs These)

| # | Metric | Category | What It Tells You |
|---|---|---|---|
| 1 | Gross Revenue | Revenue | Money coming in |
| 2 | Net Revenue | Revenue | Money after disputes |
| 3 | # Approvals | Volume | Successful transactions |
| 4 | # Attempts | Volume | Total transactions tried |
| 5 | Approval Rate | Processing | % that succeed |
| 6 | # Chargebacks | Disputes | Forced reversals |
| 7 | CB Rate (%) | Disputes | Risk level |
| 8 | # Refunds | Disputes | Voluntary reversals |
| 9 | # Cancels | Retention | Customers leaving |
| 10 | Cancel Rate | Retention | Churn speed |

### Tier 2: Operational Metrics (Operations Manager)

| # | Metric | Category | What It Tells You |
|---|---|---|---|
| 11 | First Attempt Approval % | Processing | Quality of routing |
| 12 | Net Approval % (After Retry) | Processing | Retry effectiveness |
| 13 | # Declines | Processing | Failed transactions |
| 14 | $ Lost to Declines | Processing | Revenue impact of failures |
| 15 | Recovery Rate | Processing | % of declines recovered |
| 16 | $ Recovered Revenue | Processing | Money saved by retries |
| 17 | $ Recovery Gap | Processing | Money left on the table |
| 18 | MID Capacity % Used | Risk | How close to MID cap |
| 19 | Days to CB Threshold Breach | Risk | Early warning |
| 20 | Visa VAMP % | Risk | Card network compliance |
| 21 | MC CB % | Risk | Card network compliance |
| 22 | Alert Effectiveness % | Risk | Are alert tools working |
| 23 | Alert Coverage % | Risk | % MIDs with alert protection |
| 24 | Decline Rate by Reason | Processing | What's causing failures |
| 25 | Gateway Approval Rate | Processing | Per-MID performance |

### Tier 3: Growth Metrics (Business Owner / Marketing)

| # | Metric | Category | What It Tells You |
|---|---|---|---|
| 26 | # Initials | Sales | New customer acquisitions |
| 27 | # Rebills | Sales | Recurring revenue transactions |
| 28 | $ Initial Revenue | Sales | New customer revenue |
| 29 | $ Rebill Revenue | Sales | Recurring revenue |
| 30 | AOV (Initial) | Sales | Avg purchase value |
| 31 | AOV (Rebill) | Sales | Avg recurring value |
| 32 | Revenue Growth Rate | Sales | Trend direction |
| 33 | Revenue Concentration % | Sales | Dependency risk |
| 34 | Cycle 1 Retention % | Retention | First rebill success |
| 35 | Cycle 3 Retention % | Retention | Medium-term retention |
| 36 | Cycle 6 Retention % | Retention | Long-term retention |
| 37 | 30-Day LTV | LTV | Short-term customer value |
| 38 | 60-Day LTV | LTV | Medium-term customer value |
| 39 | 90-Day LTV | LTV | Standard LTV benchmark |
| 40 | LTV:CPA Ratio | LTV | Customer profitability |
| 41 | Time to CPA Payback | LTV | Days to recover acquisition cost |
| 42 | Active Subscribers | Retention | Current subscriber base size |
| 43 | Net Subscriber Growth | Retention | New minus churned |
| 44 | Revenue per Customer | Sales | Efficiency metric |

### Tier 4: Financial Metrics (Finance / Accounting)

| # | Metric | Category | What It Tells You |
|---|---|---|---|
| 45 | Gross Profit | Profit | Revenue minus all costs |
| 46 | Gross Margin % | Profit | Efficiency percentage |
| 47 | Processing Fees | Cost | Gateway/processor costs |
| 48 | CB Fees | Cost | Dispute costs |
| 49 | Product Cost | Cost | COGS |
| 50 | CPA Cost | Cost | Customer acquisition costs |
| 51 | Alert Cost | Cost | Chargeback prevention costs |
| 52 | Reserve Amount | Cost | Held-back funds |
| 53 | Refund Alert $ | Loss | Alert-triggered refund amount |
| 54 | Refund CS $ | Loss | CS-triggered refund amount |
| 55 | CB $ | Loss | Chargeback dollar amount |

### Tier 5: Advanced / Derived Metrics (Power Users)

| # | Metric | Category | What It Tells You |
|---|---|---|---|
| 56 | Revenue Quality Score | Composite | Weighted score of approval + retention - disputes |
| 57 | Traffic Quality Score | Composite | Per-affiliate composite quality |
| 58 | MID Health Score | Composite | Per-gateway composite health |
| 59 | $ Routing Opportunity | Optimization | Revenue gain from better routing |
| 60 | Approval Rate Lift % | Optimization | Improvement from routing changes |
| 61 | Churn by Reason (Cancel/Decline/CB/Refund) | Retention | Why customers leave |
| 62 | CB Development Speed | Risk | How fast CBs accumulate after sale |
| 63 | Refund Development Speed | Risk | How fast refunds accumulate |
| 64 | Gateway Overlap % | Operations | Alert duplication across services |
| 65 | BIN-Level Approval Rate | Routing | Card-level performance |
| 66 | Bank-Level Approval Rate | Routing | Issuing bank performance |
| 67 | Cascade Rate | Processing | % of transactions sent to backup gateway |
| 68 | Cancel Rate (0-3 days) | Quality | Early cancels (indicates bad traffic) |
| 69 | Approval Rate by Hour | Processing | Time-of-day optimization |
| 70 | Revenue per Gateway $ | Operations | Per-MID revenue efficiency |

---

## What Changes Compared to Current Reports

### Reports REMOVED (merged into better ones)

| Old Report | Where It Went | Why |
|---|---|---|
| Dashboard | → **Command Center** | Dashboard was passive. Command Center is action-oriented. |
| Sales | → **Revenue Intelligence** | Added revenue quality, concentration analysis |
| MID Performance | → **Risk Control Center** | Merged with compliance monitoring, added predictive alerts |
| Approval % | → **Approval Optimization** | Added $ impact, decline root cause, gateway×BIN matrix |
| Decline Recovery | → **Decline Intelligence & Recovery** | Added recovery gap analysis, optimal retry windows |
| LTV | → **Customer Economics** | Merged with Retention for complete lifecycle story |
| Retention | → **Customer Economics** | Merged with LTV |
| CB & Refunds | → **Dispute & Alert Management** | Merged with Alert Analytics and CB Growth |
| CB & Refunds Growth | → **Dispute & Alert Management** | Merged |
| Alert Analytics | → **Dispute & Alert Management** | Merged |
| Profitability | → **Profitability Analysis** | Added per-campaign profitability, LTV:CPA |
| Order Details | → **Transaction Explorer** | Added transaction timeline, customer journey |
| Routing Insights | → **Routing Performance** | Added change tracking, specific recommendations |

### Reports ADDED (completely new)

| New Report | Why It's Needed |
|---|---|
| **Command Center** | Users need one place to see "what needs attention now" — not dig through 13 reports |
| **Live Pulse** | Real-time monitoring during campaigns, after changes |
| **Traffic & Acquisition Quality** | Currently no way to evaluate affiliate quality holistically |
| **Custom Report Builder** | Users need to create their own analyses |

### Key Innovations Across All Reports

| Innovation | What It Does | Where Used |
|---|---|---|
| **$ Impact** | Converts every % into real dollar amount | Approval, Decline, Routing, Risk |
| **Auto-Diagnosis** | When something is wrong, automatically surfaces the cause | Command Center, Approval, Risk |
| **Action Links** | Every problem links to a resolution action | Command Center, Risk, Routing |
| **Quality Scores** | Composite scores for traffic, revenue, MID health | Revenue, Traffic, Risk |
| **Predictive Alerts** | "Days until breach" based on trend projection | Risk Control |
| **Recovery Gap** | Shows money recoverable but not yet recovered | Decline Recovery |
| **Customer Lifecycle Funnel** | Visual funnel showing where and why customers leave | Customer Economics |
| **Gateway × BIN Matrix** | Approval rate heatmap for routing decisions | Routing, Approval |
| **Change Tracking** | Track impact of routing/operational changes over time | Routing |
| **Needs Attention Queue** | AI-generated priority list of actions | Command Center |

---

## Report Priority for Development

### Phase 1: Foundation (Must Have)
1. **Command Center** — This IS the product. If nothing else, this alone justifies removing Power BI.
2. **Revenue Intelligence** — Core sales reporting
3. **Approval Optimization** — Biggest revenue impact
4. **Risk Control Center** — Existential risk management
5. **Transaction Explorer** — Daily operational need

### Phase 2: Growth
6. **Decline Intelligence & Recovery** — Revenue recovery
7. **Customer Economics** — Subscription health
8. **Dispute & Alert Management** — Compliance
9. **Profitability Analysis** — Financial management

### Phase 3: Advanced
10. **Routing Performance** — Optimization
11. **Traffic & Acquisition Quality** — Marketing intelligence
12. **Live Pulse** — Real-time monitoring
13. **Custom Report Builder** — Self-service analytics

---

*Document generated: 2026-01-26*
*Beast Insights product vision — reports designed from first principles*

# Beast Insights — Cross-Industry Expansion Plan

---

## Why the Current Product Can't Scale to Other Industries

The current Beast Insights is built for **one type of user**: a high-risk merchant running subscription offers through CRMs like Sticky/Konnektive, managing dozens of MIDs, fighting chargeback thresholds, and optimizing BIN-level routing.

That's a real and valuable niche. But it means the entire report suite assumes:

```
Current assumptions (high-risk specific):
  ✗ User manages 10-300+ MIDs across multiple processors
  ✗ User needs BIN-level routing optimization
  ✗ User worries about Visa VAMP / MC EFM thresholds daily
  ✗ User uses alert services (RDR, Ethoca, CDRN)
  ✗ User has affiliates sending traffic with varying quality
  ✗ User needs cascade/retry logic per decline reason
  ✗ User's primary risk is losing a MID
```

A **telehealth company** managing patient memberships doesn't care about any of that. Neither does a **SaaS company**, a **meal kit subscription**, or a **fitness app** with recurring billing.

What ALL of them care about:

```
Universal concerns (every recurring business):
  ✓ Is revenue growing or shrinking?
  ✓ Are customers staying or leaving?
  ✓ Why are customers leaving?
  ✓ Are payments succeeding or failing?
  ✓ Which products/plans/services perform best?
  ✓ Which marketing channels bring quality customers?
  ✓ Am I actually profitable?
  ✓ What's going to happen next month?
```

---

## Target Industries

| Industry | Business Model | What They Sell | Key Concern |
|---|---|---|---|
| **High-Risk Merchants** | Subscription + trial | Supplements, skincare, digital products | MID compliance, CB prevention |
| **Telehealth** | Membership + per-visit | Consultations, prescriptions, programs | Patient retention, plan sustainability |
| **SaaS / Software** | Subscription (monthly/annual) | Software licenses, seats, usage | MRR growth, churn, expansion revenue |
| **eCommerce (Subscription)** | Recurring delivery | Meal kits, pet food, razors, boxes | Order frequency, skip rate, AOV |
| **Health & Wellness** | Membership | Gym, yoga, meditation, coaching | Member retention, seasonal churn |
| **Digital Media** | Subscription | Streaming, courses, content | Engagement-to-retention correlation |
| **Professional Services** | Retainer + project | Consulting, agencies, coaching | Revenue predictability, client retention |

---

## Platform Architecture: Core + Modules

```
┌─────────────────────────────────────────────────────────────────┐
│                    BEAST INSIGHTS PLATFORM                       │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    CORE PLATFORM                           │  │
│  │         (Every client gets these — universal)              │  │
│  │                                                            │  │
│  │  Revenue · Customers · Subscriptions · Payments            │  │
│  │  Disputes · Products · Channels · Profitability            │  │
│  │  Forecasting · Segmentation · Explorer · Custom Builder    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐│
│  │ HIGH-RISK   │ │ TELEHEALTH  │ │ SAAS        │ │ eCOMMERCE ││
│  │ MODULE      │ │ MODULE      │ │ MODULE      │ │ MODULE    ││
│  │             │ │             │ │             │ │           ││
│  │ MID Health  │ │ Provider    │ │ MRR/ARR     │ │ Order     ││
│  │ BIN Routing │ │ Analytics   │ │ Tracking    │ │ Frequency ││
│  │ Alert Mgmt  │ │ Visit-to-   │ │ Plan        │ │ Return/   ││
│  │ Compliance  │ │ Payment     │ │ Migration   │ │ Skip Rate ││
│  │ Cascade     │ │ Insurance   │ │ Trial →     │ │ Inventory ││
│  │ Gateway     │ │ vs Self-Pay │ │ Paid Conv.  │ │ Demand    ││
│  │ Matrix      │ │ Compliance  │ │ Seat/Usage  │ │ Seasonal  ││
│  └─────────────┘ └─────────────┘ └─────────────┘ └───────────┘│
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    PLATFORM FEATURES                       │  │
│  │  Saved Views · Custom Reports · Personal Dashboard         │  │
│  │  Smart Alerts · Scheduled Exports · Team Sharing           │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**The idea:** Every client gets the Core Platform (12 universal reports). Industry-specific modules add 3-5 specialized reports on top. A client can enable multiple modules.

---

## User Personas (Cross-Industry)

| Persona | Industries | Checks Platform | Cares About | Action |
|---|---|---|---|---|
| **Business Owner** | All | Daily / weekly | Revenue, growth, profitability | Budget, strategy, pricing |
| **Operations / Billing** | All | Multiple times/day | Payment failures, disputes, exceptions | Fix billing issues, refund, retry |
| **Marketing / Growth** | All | Daily | Channel ROI, customer acquisition, quality | Scale/pause campaigns, reallocate spend |
| **Finance** | All | Weekly / monthly | P&L, margins, forecasting, reconciliation | Adjust pricing, negotiate fees |
| **Customer Success** | Telehealth, SaaS | Daily | Churn risk, customer health, retention | Outreach, save offers, plan changes |
| **Compliance / Risk** | High-risk, Telehealth | Weekly | CB rates, regulatory compliance | Pause sources, add protections |

---

## Core Platform — 12 Universal Reports

These reports work for ANY business that processes recurring payments. No industry-specific terminology. No assumptions about MIDs, BINs, or gateways.

```
MONITOR
  ├── 1. Business Command Center
  ├── 2. Real-Time Pulse

GROW
  ├── 3. Revenue Analytics
  ├── 4. Subscription Intelligence
  ├── 5. Customer Lifecycle

RETAIN
  ├── 6. Churn Analysis
  ├── 7. Payment Health & Recovery

ACQUIRE
  ├── 8. Channel & Acquisition Performance

PROFIT
  ├── 9. Financial Performance
  ├── 10. Product & Plan Performance

EXPLORE
  ├── 11. Transaction Explorer
  ├── 12. Custom Report Builder
```

---

### Report 1: Business Command Center

**Who needs it:** Everyone
**Question it answers:** "What needs my attention right now?"

```
┌─────────────────────────────────────────────────────────────────┐
│  COMMAND CENTER                                      ● Jan 26   │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐│
│  │  REVENUE   │  │ NEW CUSTS  │  │ ACTIVE     │  │  CHURN     ││
│  │  $14,230   │  │    82      │  │ SUBS 4,520 │  │   2.8%     ││
│  │  ▲ +12%    │  │  ▲ +15%    │  │  ▲ +3%     │  │  ▼ -0.3%   ││
│  │  vs 7d avg │  │  vs 7d avg │  │  vs prior  │  │  vs prior  ││
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘│
│                                                                  │
│  ⚡ NEEDS ATTENTION (3)                                          │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ 🔴 Revenue dropped 25% vs last week                         ││
│  │    Cause: Campaign "Google Ads" paused 2 days ago            ││
│  │    Impact: ~$4,200/day lost                                  ││
│  │    [View Channel Details] [View Revenue Breakdown]           ││
│  ├──────────────────────────────────────────────────────────────┤│
│  │ 🟡 Payment failure rate spiked to 12% (normally 6%)         ││
│  │    Cause: 78% are "Card Expired" — card updater may help    ││
│  │    Impact: 340 customers at risk of involuntary churn        ││
│  │    [View Failed Payments] [View At-Risk Customers]           ││
│  ├──────────────────────────────────────────────────────────────┤│
│  │ 🟡 Refund rate up 40% this week                              ││
│  │    Cause: Product "Premium Plan" refund rate 8.2% (was 3%)  ││
│  │    [View Refund Details] [View Product Performance]          ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  📊 KEY METRICS (Last 7 Days)                                    │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Date   │Revenue │New Cust│Renewals│Fails│Refunds│Churn │Net ││
│  │ Today  │$14.2k  │  82   │  220   │  28 │   5   │ 12  │+70 ││
│  │ Jan 25 │$16.8k  │  94   │  245   │  22 │   3   │ 10  │+84 ││
│  │ Jan 24 │$15.1k  │  88   │  232   │  31 │   8   │ 14  │+74 ││
│  │ Jan 23 │$17.2k  │  96   │  258   │  19 │   4   │  9  │+87 ││
│  │ ...    │        │       │        │     │       │     │     ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  📈 REVENUE TREND (Today vs 7-Day Avg)                           │
│  $|          ╭──╮                                                │
│   |     ╭──╮ │  │ ╭╮        ── Today                            │
│   |  ╭╮ │  │ │  │ ││        -- 7-day avg                        │
│   |──╯╰─╯  ╰─╯  ╰─╯│                                           │
│   └──────────────────────                                       │
│     6am   9am   12pm   3pm   6pm                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key Metrics:**
| Metric | Why Universal |
|---|---|
| Today's Revenue | Every business tracks revenue |
| New Customers | Growth indicator |
| Active Subscribers | Base size |
| Churn Rate | Retention health |
| Payment Failure Rate | Involuntary churn risk |
| Refund Rate | Customer satisfaction signal |
| Net Customer Growth | New minus churned |

**What makes this different from a dashboard:** The "Needs Attention" section automatically detects anomalies and surfaces the CAUSE, not just the symptom. A telehealth company sees "Patient signups dropped" + why. A SaaS company sees "MRR contraction" + which plan is losing customers.

---

### Report 2: Real-Time Pulse

**Who needs it:** Operations teams, during campaigns/launches
**Question it answers:** "How is today going, hour by hour?"

```
┌─────────────────────────────────────────────────────────────────┐
│  REAL-TIME PULSE                              Today  ● LIVE     │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐│
│  │ REVENUE    │  │ ORDERS     │  │ NEW CUST.  │  │ FAILURES   ││
│  │ $14,230    │  │   342      │  │    82      │  │   28       ││
│  │ running    │  │ running    │  │ running    │  │ 5.8% rate  ││
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘│
│                                                                  │
│  REVENUE BY HOUR                                                 │
│  $2k|                                                            │
│     |          ╭──╮                                              │
│     |     ╭──╮ │  │ ╭╮        ── Today                          │
│     |  ╭╮ │  │ │  │ ││        -- 7-day avg                      │
│     |──╯╰─╯  ╰─╯  ╰─╯│       .. Last week same day             │
│  $0 └──────────────────────                                     │
│      6am   9am   12pm   3pm   6pm   9pm                         │
│                                                                  │
│  TRANSACTIONS BY HOUR                                            │
│  80|     ┌──┐       ┌──┐                                        │
│    |┌──┐ │▓▓│  ┌──┐ │▓▓│        ▓ New customers                 │
│    |│▓▓│ │░░│  │▓▓│ │░░│        ░ Renewals                      │
│    |│░░│ │░░│  │░░│ │░░│        □ One-time purchases             │
│   0└────────────────────                                        │
│     6am   9am   12pm   3pm                                       │
│                                                                  │
│  PAYMENT SUCCESS BY HOUR                                         │
│  100%|  ●──●──●──●                                              │
│   90%|               ╲●──●     ⚠ 2pm drop                      │
│   80%|                                                           │
│      └──────────────────────                                    │
│       6am   9am   12pm   3pm                                     │
│                                                                  │
│  ⚠ 2pm: Payment success dropped to 88% — 3 "processor timeout" │
│    errors detected  [View Failed Transactions]                   │
└─────────────────────────────────────────────────────────────────┘
```

**Use cases across industries:**
- **Telehealth:** Monitor after launching new membership plan
- **SaaS:** Watch real-time during product launch or pricing change
- **eCommerce:** Track flash sale or holiday performance
- **High-risk:** Monitor after routing changes

---

### Report 3: Revenue Analytics

**Who needs it:** Business owners, finance, marketing
**Question it answers:** "Where is money coming from, how is it trending, and is it diversified?"

```
┌─────────────────────────────────────────────────────────────────┐
│  REVENUE ANALYTICS                           Last 30 Days  [▼]  │
│  View: [Revenue ▼]   Split by: [All ▼]                          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ TOTAL REV.   │  │ NEW CUST REV │  │ RECURRING REV│          │
│  │ $427,450     │  │ $142,300     │  │ $285,150     │          │
│  │ ▲ +14%       │  │ ▲ +18%       │  │ ▲ +12%       │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ NET REVENUE  │  │ AVG ORDER    │  │ REV PER CUST │          │
│  │ $398,200     │  │ $52.40       │  │ $88.10       │          │
│  │ (after refund│  │ ▲ +3%        │  │ ▲ +5%        │          │
│  │  + disputes) │  │              │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  REVENUE TREND                                                   │
│  $20k|                      ╭─╮     ╭─╮                         │
│      |   ╭─╮ ╭─╮ ╭─╮  ╭─╮ │ │╭─╮  │ │                         │
│      |╭─╮│ │ │ │ │ │╭─╮│ │ │ ││ │╭─╯ │                         │
│  $10k|│ ││ │ │ │ │ ││ ││ │ │ ││ ││   │                         │
│      └──────────────────────────────────                        │
│       W1    W2    W3    W4    W5    W6                            │
│       ■ New customer revenue  ■ Recurring revenue                │
│                                                                  │
│  REVENUE BREAKDOWN               Group By: [Product/Plan ▼]     │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Product/Plan   │ New Rev │ Recur Rev │ Total  │ % Tot│Trend│ │
│  ├────────────────┼─────────┼───────────┼────────┼──────┼─────┤ │
│  │ Premium Plan   │ $62.4k  │  $128.1k  │$190.5k │  45% │  ↗  │ │
│  │ Basic Plan     │ $48.2k  │   $98.3k  │$146.5k │  34% │  →  │ │
│  │ Starter Plan   │ $22.1k  │   $42.8k  │ $64.9k │  15% │  ↗  │ │
│  │ One-Time Purch │  $9.6k  │     —     │  $9.6k │   2% │  ↘  │ │
│  │ Add-Ons        │   —     │   $15.9k  │ $15.9k │   4% │  ↗  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  REVENUE CONCENTRATION                                           │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Top product = 45% of revenue                          [RISK]││
│  │ Top channel = 38% of revenue                          [RISK]││
│  │ Top 10% of customers = 52% of revenue                 [NOTE]││
│  │                                                               ││
│  │ 💡 Revenue is concentrated — losing "Premium Plan" or        ││
│  │    "Google Ads" channel would impact nearly half of revenue  ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  REVENUE QUALITY SCORE (per product/channel)                     │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Source       │Revenue│Retention│Refund%│Dispute%│LTV │ Score ││
│  │ Premium Plan │ $190k │   68%   │  2.1% │  0.4%  │$188│ ✅ 88 ││
│  │ Basic Plan   │ $146k │   52%   │  3.8% │  0.8%  │$112│ ⚠️ 72 ││
│  │ Starter Plan │  $65k │   41%   │  5.2% │  1.2%  │ $64│ ⚠️ 58 ││
│  └──────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

**Group By options (universal):** Product/Plan, Channel/Source, Date Period, Customer Segment, Geography

**Key Metrics:**
| Metric | Why It Matters |
|---|---|
| Total Revenue | Top line |
| New vs Recurring Revenue | Growth vs stability balance |
| Net Revenue | After refunds + disputes |
| Avg Order Value | Pricing efficiency |
| Revenue per Customer | Customer efficiency |
| Revenue Concentration % | Dependency risk |
| Revenue Quality Score | Sustainability of each revenue source |
| Revenue Growth Rate | Trend direction |

---

### Report 4: Subscription Intelligence

**Who needs it:** Business owners, finance, customer success
**Question it answers:** "Is my recurring revenue engine healthy?"

```
┌─────────────────────────────────────────────────────────────────┐
│  SUBSCRIPTION INTELLIGENCE                   Last 6 Months [▼]  │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐│
│  │ MRR        │  │ ACTIVE     │  │ NET NEW    │  │ GROWTH     ││
│  │ $142,600   │  │ SUBS 4,520 │  │ MRR +$8.2k │  │ RATE       ││
│  │ ▲ +6.1%    │  │ ▲ +180 net │  │ this month │  │ +6.1% MoM  ││
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘│
│                                                                  │
│  MRR MOVEMENT (This Month)                                       │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │                                                               ││
│  │  Starting MRR         $134,400                                ││
│  │  ██████████████████████████████████████████████                ││
│  │                                                               ││
│  │  + New MRR             +$12,800  (new customers)              ││
│  │    ██████                                                     ││
│  │  + Expansion MRR       +$3,400   (upgrades, add-ons)          ││
│  │    ██                                                         ││
│  │  - Contraction MRR     -$1,200   (downgrades)                 ││
│  │    █                                                          ││
│  │  - Churned MRR         -$6,800   (cancels + failed payments)  ││
│  │    ███                                                        ││
│  │  ─────────────────────────────                                ││
│  │  = Ending MRR          $142,600  (net +$8,200)                ││
│  │  ██████████████████████████████████████████████████            ││
│  │                                                               ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  MRR TREND (6 Months)                                            │
│  $150k|                                    ╭────● $142.6k        │
│       |                           ╭────────╯                     │
│  $120k|              ╭────────────╯                              │
│       |    ╭─────────╯                                           │
│  $90k |────╯                                                     │
│       └──────────────────────────────                            │
│        Aug    Sep    Oct    Nov    Dec    Jan                     │
│                                                                  │
│  SUBSCRIBER FLOW                                                 │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Month │ Start │ + New │+Reactivate│ - Cancel│ - Failed│ End ││
│  ├───────┼───────┼───────┼───────────┼─────────┼─────────┼─────┤│
│  │ Jan   │ 4,340 │  320  │     42    │   -112  │    -70  │4,520││
│  │ Dec   │ 4,180 │  298  │     38    │   -108  │    -68  │4,340││
│  │ Nov   │ 4,020 │  312  │     45    │    -98  │    -99  │4,180││
│  │ Oct   │ 3,870 │  280  │     52    │   -104  │    -78  │4,020││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  PLAN DISTRIBUTION                                               │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Plan      │ Subs │  MRR   │ ARPU │ Churn │ LTV  │ Growth   ││
│  ├───────────┼──────┼────────┼──────┼───────┼──────┼──────────┤│
│  │ Premium   │1,240 │ $74.4k │ $60  │  1.8% │ $280 │ ▲ +8%   ││
│  │ Basic     │2,180 │ $52.3k │ $24  │  3.2% │ $112 │ → +1%   ││
│  │ Starter   │1,100 │ $15.9k │ $14  │  5.1% │  $64 │ ▼ -2%   ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ⚠ Starter plan has 5.1% churn — 2x higher than Premium.        │
│    Consider: improve onboarding, adjust pricing, or sunset plan  │
│                                                                  │
│  💡 Expansion MRR ($3.4k) is healthy — upgrades are happening.  │
│    If you can reduce involuntary churn (failed payments), net    │
│    MRR growth jumps from +$8.2k to +$11.8k                      │
└─────────────────────────────────────────────────────────────────┘
```

**Key Metrics:**
| Metric | Why It Matters |
|---|---|
| MRR / ARR | Core subscription health metric |
| New MRR | Growth from new customers |
| Expansion MRR | Upsells, upgrades, add-ons |
| Contraction MRR | Downgrades |
| Churned MRR | Lost revenue from cancels + payment failures |
| Net MRR Growth | The bottom line of subscription health |
| ARPU (Avg Revenue Per User) | Are customers paying more or less over time? |
| Active Subscribers | Total base |
| Net Subscriber Growth | Inflow vs outflow |
| Plan Distribution | Revenue mix risk |

**Why this matters for expansion:** A telehealth company sees "Patient memberships grew 6% but the Basic plan is churning at 5%." A SaaS company sees "Net MRR is +$8.2k but $6.8k churned — reducing churn is 2x more impactful than acquiring more."

---

### Report 5: Customer Lifecycle

**Who needs it:** Growth, marketing, customer success
**Question it answers:** "How long do customers stay, how much are they worth, and where do we lose them?"

```
┌─────────────────────────────────────────────────────────────────┐
│  CUSTOMER LIFECYCLE                           Last 6 Months [▼] │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐│
│  │ AVG LTV    │  │ AVG LTV    │  │ AVG LTV    │  │ LTV : CAC  ││
│  │ (30-Day)   │  │ (90-Day)   │  │ (180-Day)  │  │ RATIO      ││
│  │   $67.40   │  │  $112.50   │  │  $168.20   │  │   3.2x     ││
│  │  ▲ +4%     │  │  ▲ +7%     │  │  ▲ +5%     │  │  ▲ +0.4    ││
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘│
│                                                                  │
│  CUSTOMER RETENTION FUNNEL                                       │
│                                                                  │
│  Signup / Purchase  ████████████████████████████████   1,240 100%│
│      ↓                                                           │
│  Month 1 Active     ██████████████████████              756  61%│
│      ↓               WHY THEY LEFT:                              │
│  Month 2 Active     █████████████████                   584  47%│
│      ↓               Cancel: 52%                                 │
│  Month 3 Active     ████████████████                    521  42%│
│      ↓               Payment Failed: 28%                         │
│  Month 6 Active     ████████████                        372  30%│
│      ↓               Refund: 12%                                 │
│  Month 12 Active    █████████                           248  20%│
│                      Dispute: 8%                                 │
│                                                                  │
│  RETENTION COHORT HEATMAP (% still active)                       │
│  ┌────────┬───────┬───────┬───────┬───────┬───────┬───────┐     │
│  │ Cohort │ M0    │  M1   │  M2   │  M3   │  M4   │  M5   │     │
│  ├────────┼───────┼───────┼───────┼───────┼───────┼───────┤     │
│  │ Aug 25 │  100% │🟢 65% │🟢 52% │🟢 44% │🟢 38% │🟢 34% │     │
│  │ Sep 25 │  100% │🟢 62% │🟡 48% │🟡 40% │🟡 35% │       │     │
│  │ Oct 25 │  100% │🟡 58% │🟡 44% │🔴 32% │       │       │     │
│  │ Nov 25 │  100% │🟢 64% │🟢 50% │       │       │       │     │
│  │ Dec 25 │  100% │🟢 63% │       │       │       │       │     │
│  │ Jan 26 │  100% │       │       │       │       │       │     │
│  └────────┴───────┴───────┴───────┴───────┴───────┴───────┘     │
│                                                                  │
│  ⚠ Oct 2025 cohort: M3 retention (32%) is far below average     │
│    (42%). Channel mix that month: 60% from "Facebook Ads" —      │
│    that channel has lowest quality score.                         │
│    [View Channel Details] [Compare Cohorts]                      │
│                                                                  │
│  LTV BY ACQUISITION CHANNEL                                      │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Channel     │Customers│ 30d LTV│ 90d LTV│ CAC │LTV:CAC│Score││
│  │ Organic     │   180   │  $82   │  $148  │ $12 │ 12.3x │✅ 95││
│  │ Google Ads  │   420   │  $72   │  $125  │ $42 │  3.0x │✅ 82││
│  │ Email       │   240   │  $68   │  $118  │ $8  │ 14.8x │✅ 90││
│  │ Facebook    │   310   │  $52   │   $78  │ $38 │  2.1x │⚠️ 58││
│  │ Affiliates  │    90   │  $44   │   $62  │ $55 │  1.1x │🔴 35││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  💡 Organic and Email customers are 4-5x more profitable than    │
│    Facebook. Consider reallocating $10k/mo from FB to Email.     │
└─────────────────────────────────────────────────────────────────┘
```

**Key Metrics:**
| Metric | Why It Matters |
|---|---|
| LTV (30/90/180 day) | Customer value over time |
| LTV:CAC Ratio | Is acquisition profitable? (should be >3x) |
| Retention by Month | Where in the lifecycle do customers leave? |
| Churn Reason Split | Cancel vs payment failure vs refund vs dispute |
| Payback Period | How many days to recover acquisition cost |
| Cohort Retention % | Compare quality across time periods |
| LTV by Channel | Which sources bring the best long-term customers |

---

### Report 6: Churn Analysis

**Who needs it:** Customer success, growth, operations
**Question it answers:** "Why are customers leaving and how do I stop them?"

```
┌─────────────────────────────────────────────────────────────────┐
│  CHURN ANALYSIS                               Last 3 Months [▼] │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐│
│  │ TOTAL      │  │ VOLUNTARY  │  │ INVOLUNTARY│  │ $ CHURNED  ││
│  │ CHURN RATE │  │ (cancels)  │  │ (pay fail) │  │ MRR        ││
│  │   3.2%     │  │   1.9%     │  │   1.3%     │  │  $6,800    ││
│  │  ▼ -0.3%   │  │  ▼ -0.2%   │  │  ▼ -0.1%   │  │  ▼ -$400   ││
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘│
│                                                                  │
│  CHURN BREAKDOWN                                                 │
│                                                                  │
│  Total Churned: 182 customers ($6,800 MRR)                       │
│                                                                  │
│  VOLUNTARY (112 customers, $4,100 MRR)                           │
│  ├── Self-cancelled           ████████████████      68  (61%)   │
│  │   Top reasons given:                                          │
│  │   "Too expensive" 35% │ "Not using" 28% │ "Found alt." 18%  │
│  ├── Requested refund         ████████              32  (29%)   │
│  └── Disputed / Chargeback    ███                   12  (10%)   │
│                                                                  │
│  INVOLUNTARY (70 customers, $2,700 MRR)                          │
│  ├── Card expired             ██████████████        42  (60%)   │
│  ├── Insufficient funds       ██████                18  (26%)   │
│  ├── Card declined (other)    ███                   10  (14%)   │
│                                                                  │
│  💡 60% of involuntary churn is EXPIRED CARDS.                   │
│     A card updater service could save ~25 customers/month        │
│     = $960/month in recovered MRR                                │
│                                                                  │
│  CHURN BY CUSTOMER AGE                                           │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Customer Age  │ # Churned │ Churn Rate │ Primary Reason      ││
│  ├───────────────┼───────────┼────────────┼─────────────────────┤│
│  │ 0-30 days     │    68     │    8.2%    │ "Not using" (42%)   ││
│  │ 31-90 days    │    52     │    4.1%    │ "Too expensive"(38%)││
│  │ 91-180 days   │    38     │    2.8%    │ Card expired (52%)  ││
│  │ 180+ days     │    24     │    1.2%    │ Card expired (61%)  ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ⚠ First 30 days has 8.2% churn — onboarding problem.           │
│    "Not using" is the #1 reason. Customers aren't finding value. │
│    [View Onboarding Funnel] [View These Customers]               │
│                                                                  │
│  CHURN TREND                                                     │
│   5%|                                                            │
│     | ●──●                                                       │
│   4%|      ╲●──●                                                 │
│     |           ╲●──●──●                                         │
│   3%|                     ── Total churn                         │
│     |                     -- Voluntary                           │
│   2%|                     .. Involuntary                         │
│     └──────────────────────────                                  │
│      Aug  Sep  Oct  Nov  Dec  Jan                                 │
│                                                                  │
│  CHURN BY PRODUCT/PLAN                                           │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Plan      │ Churned│ Rate │ Top Reason        │ $ Impact    ││
│  │ Starter   │   82   │ 5.1% │ "Not using" 45%   │ $1,148/mo  ││
│  │ Basic     │   68   │ 3.2% │ "Too expensive"   │ $1,632/mo  ││
│  │ Premium   │   32   │ 1.8% │ Card expired 58%  │ $1,920/mo  ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  AT-RISK CUSTOMERS (predicted to churn in next 30 days)          │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ 47 customers flagged │ $1,840 MRR at risk                    ││
│  │ Signals: No login 14+ days, payment failed, downgraded       ││
│  │ [View At-Risk List] [Export for Outreach]                    ││
│  └──────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

**Key Innovation:** Splitting churn into **voluntary** (customer chose to leave) vs **involuntary** (payment failed) changes the response entirely. Voluntary churn needs better product/pricing. Involuntary churn needs better payment recovery. Most platforms don't make this distinction.

---

### Report 7: Payment Health & Recovery

**Who needs it:** Operations, finance
**Question it answers:** "Are payments succeeding? How much revenue am I losing to failures?"

This is the universal version of what was previously "Approval Optimization" and "Decline Intelligence." No BIN routing, no gateway matrices — just clean payment health.

```
┌─────────────────────────────────────────────────────────────────┐
│  PAYMENT HEALTH & RECOVERY                    Last 30 Days [▼]  │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐│
│  │ SUCCESS    │  │ FAILED     │  │ RECOVERED  │  │ 💰 $ LOST  ││
│  │ RATE       │  │ PAYMENTS   │  │            │  │ (net)      ││
│  │   94.2%    │  │    812     │  │    348     │  │  $18,400   ││
│  │  ▲ +0.8%   │  │  (5.8%)   │  │ (42.9%)    │  │            ││
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘│
│                                                                  │
│  WHY PAYMENTS FAIL                                               │
│                                                                  │
│  812 failed payments = $38,200 at risk                           │
│  ├── Card Expired            ████████████████████    326 (40%)  │
│  │   → Recoverable with card updater                             │
│  ├── Insufficient Funds      ██████████████          228 (28%)  │
│  │   → Recoverable with smart retry                              │
│  ├── Processor Declined      █████████               162 (20%)  │
│  │   → Partially recoverable                                    │
│  ├── Invalid Card Data       ████                     57  (7%)  │
│  │   → Customer needs to update                                 │
│  └── Other / Unknown         ██                       39  (5%)  │
│                                                                  │
│  RECOVERY PERFORMANCE                                            │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Failure Reason    │ Failed│Retried│Recovered│Rate │$ Saved  ││
│  ├───────────────────┼───────┼───────┼─────────┼─────┼─────────┤│
│  │ Card Expired      │  326  │  326  │   195   │ 60% │ $9,750  ││
│  │ Insufficient Funds│  228  │  228  │   114   │ 50% │ $5,130  ││
│  │ Processor Declined│  162  │  162  │    39   │ 24% │ $1,950  ││
│  │ Invalid Card      │   57  │    0  │     0   │  0% │    —    ││
│  │ Other             │   39  │   39  │     0   │  0% │    —    ││
│  ├───────────────────┼───────┼───────┼─────────┼─────┼─────────┤│
│  │ TOTAL             │  812  │  755  │   348   │ 46% │$16,830  ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ★ RECOVERY GAP: $21,370 still potentially recoverable          │
│    - 131 expired cards not yet updated → $6,550 potential        │
│    - 114 "insufficient funds" not yet retried → $5,700 potential │
│    [View Recovery Opportunities]                                 │
│                                                                  │
│  PAYMENT SUCCESS TREND                                           │
│  100%|  ●──●──●──●──●──●                                        │
│   95%|                    ╲●──●                                  │
│   90%|                         ──●     ⚠ Dec drop               │
│      └──────────────────────────────                             │
│       Aug  Sep  Oct  Nov  Dec  Jan                                │
│                                                                  │
│  PAYMENT METHOD BREAKDOWN                                        │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Method      │ Volume │ Success │ Avg Txn │ Fail Rate │       ││
│  │ Visa        │  5,240 │  94.8%  │  $47.20 │    5.2%   │       ││
│  │ Mastercard  │  3,820 │  93.1%  │  $45.80 │    6.9%   │       ││
│  │ Amex        │  1,180 │  96.2%  │  $62.40 │    3.8%   │       ││
│  │ Discover    │    460 │  91.8%  │  $41.30 │    8.2%   │       ││
│  │ ACH/Bank    │    320 │  98.1%  │  $88.10 │    1.9%   │       ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  💡 ACH/Bank has 98.1% success rate — encourage customers to     │
│    switch from card to ACH to reduce involuntary churn           │
└─────────────────────────────────────────────────────────────────┘
```

**How this differs from the high-risk version:** No BIN-level data. No gateway routing matrix. No MID management. Just clean, universal payment health that any business understands. The high-risk module ADDS the BIN/gateway/MID layer on top.

---

### Report 8: Channel & Acquisition Performance

**Who needs it:** Marketing, growth
**Question it answers:** "Which channels bring customers worth keeping?"

```
┌─────────────────────────────────────────────────────────────────┐
│  CHANNEL & ACQUISITION                       Last 90 Days  [▼]  │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐│
│  │ TOTAL NEW  │  │ AVG CAC    │  │ BEST       │  │ WORST      ││
│  │ CUSTOMERS  │  │            │  │ CHANNEL    │  │ CHANNEL    ││
│  │   2,840    │  │  $38.20    │  │ Organic    │  │ Facebook   ││
│  │  ▲ +12%    │  │  ▲ +$2.10  │  │ LTV:CAC 12x│  │ LTV:CAC 2x ││
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘│
│                                                                  │
│  CHANNEL SCORECARD                                               │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Channel    │ Custs│ CAC │ Conv%│M1 Ret│90d LTV│LTV:CAC│Score│
│  ├────────────┼──────┼─────┼──────┼──────┼───────┼───────┼─────┤
│  │ Organic    │  480 │ $12 │ 4.2% │  72% │  $148 │ 12.3x │✅ 95│
│  │ Email      │  620 │  $8 │ 8.1% │  68% │  $118 │ 14.8x │✅ 92│
│  │ Google Ads │  840 │ $42 │ 2.8% │  58% │  $125 │  3.0x │✅ 78│
│  │ Referral   │  210 │ $25 │ 5.5% │  64% │  $132 │  5.3x │✅ 84│
│  │ Facebook   │  520 │ $38 │ 1.9% │  42% │   $78 │  2.1x │⚠️ 52│
│  │ Affiliates │  170 │ $55 │ 1.2% │  35% │   $62 │  1.1x │🔴 30│
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  CHANNEL QUALITY TREND                                           │
│  Score|                                                          │
│   100 | ──●──●──●──●──●──● Organic (stable)                     │
│    80 | ──▲──▲──▲──▲──▲──▲ Google (stable)                      │
│    60 |            ╲                                             │
│    40 | ──■──■──■──■──■    Facebook (declining since Oct)        │
│    20 |                 ╲■                                       │
│       └──────────────────────                                   │
│        Aug  Sep  Oct  Nov  Dec  Jan                               │
│                                                                  │
│  ACQUISITION FUNNEL (Last 30 Days)                               │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Visitors        ██████████████████████████████   48,200  100%││
│  │ Signups/Trials  ████████████                      4,820   10%││
│  │ First Purchase  ██████                            2,840  5.9%││
│  │ Month 1 Active  ████                              1,704  3.5%││
│  │ Month 3 Active  ██                                  852  1.8%││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  💡 Facebook quality has been declining for 3 months.            │
│    Customers from Facebook have 42% M1 retention vs 58% avg.    │
│    Recommend: Reduce Facebook spend by 30%, reallocate to Email. │
│    Projected impact: +$4,200/mo in retained MRR                  │
└─────────────────────────────────────────────────────────────────┘
```

**This replaces "Traffic & Acquisition Quality"** from the high-risk version, but uses universal language (channels, CAC, conversion) instead of affiliate-specific terms.

---

### Report 9: Financial Performance

**Who needs it:** Finance, business owners
**Question it answers:** "After all costs, am I making money?"

```
┌─────────────────────────────────────────────────────────────────┐
│  FINANCIAL PERFORMANCE                        Last 30 Days [▼]  │
│                                                                  │
│  P&L WATERFALL                                                   │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Gross Revenue        $427,450                                ││
│  │ ████████████████████████████████████████████████████          ││
│  │                                                               ││
│  │  - Refunds             -$12,300   (2.9%)                      ││
│  │  - Chargebacks          -$4,200   (1.0%)                      ││
│  │  - Discounts/Credits    -$3,100   (0.7%)                      ││
│  │  ─────────────────────                                        ││
│  │ Net Revenue          $407,850                                 ││
│  │ ██████████████████████████████████████████████████            ││
│  │                                                               ││
│  │  - Payment Processing  -$12,240   (3.0%)                      ││
│  │  - Platform/Software    -$2,800   (0.7%)                      ││
│  │  - Customer Acq. Cost -$42,600   (10.4%)                      ││
│  │  - Product/Service Cost-$85,200   (20.9%)                     ││
│  │  - Dispute Fees         -$1,050   (0.3%)                      ││
│  │  ─────────────────────                                        ││
│  │ Gross Profit         $263,960                                 ││
│  │ ████████████████████████████████                              ││
│  │ Gross Margin:          61.7%                                  ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  PROFITABILITY BY PRODUCT/PLAN                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Product    │Revenue│ Costs │Profit │Margin│LTV:CAC│Status │ │
│  ├────────────┼───────┼───────┼───────┼──────┼───────┼───────┤ │
│  │ Premium    │$190.5k│$68.2k │$122.3k│  64% │  4.7x │  ✅   │ │
│  │ Basic      │$146.5k│$62.1k │ $84.4k│  58% │  3.2x │  ✅   │ │
│  │ Starter    │ $64.9k│$42.8k │ $22.1k│  34% │  1.4x │  🔴   │ │
│  │ Add-Ons    │ $15.9k│ $4.2k │ $11.7k│  74% │   —   │  ✅   │ │
│  │ One-Time   │  $9.6k│ $6.8k │  $2.8k│  29% │  0.8x │  🔴   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  MARGIN TREND                      COST BREAKDOWN                │
│  ┌─────────────────────┐          ┌─────────────────────┐       │
│  │ 65%|        ╭─╮     │          │                     │       │
│  │    |   ╭─╮╭─╯ │     │          │ COGS      ████ 52%  │       │
│  │ 60%|╭─╮│ ││   │╭─╮  │          │ CAC       ███  26%  │       │
│  │    |│ ╰╯ ╰╯   ╰╯ │  │          │ Processing██    8%  │       │
│  │ 55%|│             │  │          │ Refund/CB █     4%  │       │
│  │    └──────────────│  │          │ Other     ██   10%  │       │
│  │     W1 W2 W3 W4 W5  │          └─────────────────────┘       │
│  └─────────────────────┘                                        │
│                                                                  │
│  🔴 "Starter Plan" has 34% margin and LTV:CAC of 1.4x.          │
│    With 5.1% churn, this plan is marginally profitable.          │
│    Consider: raise price from $14 to $19 or reduce COGS.        │
│                                                                  │
│  🔴 "One-Time" purchases have negative unit economics (0.8x).   │
│    These lose money unless they convert to subscriptions.        │
│    Conversion rate to subscription: 12%                          │
│    [View Conversion Funnel]                                      │
└─────────────────────────────────────────────────────────────────┘
```

**Cost inputs:** Users configure their cost structure once (processing %, CAC per channel, COGS per product). The system calculates profitability automatically. This replaces the slider-based approach from the old profitability report.

---

### Report 10: Product & Plan Performance

**Who needs it:** Product, growth, business owners
**Question it answers:** "Which products/plans are performing and which need attention?"

```
┌─────────────────────────────────────────────────────────────────┐
│  PRODUCT & PLAN PERFORMANCE                   Last 30 Days [▼]  │
│                                                                  │
│  PLAN OVERVIEW                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Plan     │Subs │ Revenue│Growth│Churn│ Refund%│NPS  │Health│ │
│  ├──────────┼─────┼────────┼──────┼─────┼────────┼─────┼──────┤ │
│  │ Premium  │1,240│$190.5k │ +8%  │1.8% │  1.2%  │ 72  │ ✅   │ │
│  │ Basic    │2,180│$146.5k │ +1%  │3.2% │  2.8%  │ 58  │ ⚠️   │ │
│  │ Starter  │1,100│ $64.9k │ -2%  │5.1% │  4.5%  │ 42  │ 🔴   │ │
│  │ Add-Ons  │  820│ $15.9k │+12%  │ —   │  0.8%  │ —   │ ✅   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  PLAN MIGRATION FLOW (This Month)                                │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │                                                               ││
│  │  Starter ──── 42 upgraded ────→ Basic                        ││
│  │           ╲                      │                            ││
│  │            ── 82 churned         ── 28 upgraded ──→ Premium  ││
│  │                                  │                            ││
│  │                                  ── 68 churned                ││
│  │                                                               ││
│  │  Summary: 70 total upgrades │ 150 total churns               ││
│  │           Upgrade rate: 3.2% │ Churn rate: 3.3%               ││
│  │                                                               ││
│  │  💡 Starter → Basic upgrade rate (3.8%) is healthy            ││
│  │     Basic → Premium upgrade rate (1.3%) has room to grow      ││
│  │     Consider mid-tier plan between Basic and Premium          ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  REVENUE BY PRODUCT TREND                                        │
│  $200k|                           ╭── Premium (growing)          │
│       |               ╭──────────╯                               │
│  $150k| ╭─────────────╯                                          │
│       |╭╯    ╭──────────────────── Basic (flat)                   │
│  $100k|╯╭────╯                                                   │
│       |  ╰──────────╲                                            │
│   $50k|              ╰──────────── Starter (declining)           │
│       └──────────────────────────                                │
│        Aug  Sep  Oct  Nov  Dec  Jan                               │
│                                                                  │
│  TOP PERFORMING ADD-ONS                                          │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Add-On           │ Attached To │ Adoption │ Revenue │ Growth ││
│  │ Priority Support │ Premium     │   32%    │  $6.2k  │  +15%  ││
│  │ Extra Storage    │ Basic+      │   18%    │  $4.8k  │  +22%  ││
│  │ Analytics Pack   │ All Plans   │    8%    │  $3.1k  │   +8%  ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ⚠ Starter plan is shrinking (-2%) with 5.1% churn.             │
│    Most Starter churns cite "not enough features" (45%).         │
│    Consider: add more value to Starter or create better upgrade  │
│    path with trial of Basic features.                            │
└─────────────────────────────────────────────────────────────────┘
```

**Key Innovation:** The **Plan Migration Flow** shows how customers move between plans — upgrades, downgrades, and churn at each tier. This doesn't exist in the current product.

---

### Report 11: Transaction Explorer

**Who needs it:** Operations, support, finance
**Question it answers:** "What happened with this specific transaction or customer?"

```
┌─────────────────────────────────────────────────────────────────┐
│  TRANSACTION EXPLORER                                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ 🔍 [Search by ID, email, name, phone...        ] [Search]   ││
│  │                                                               ││
│  │ [All] [Successful] [Failed] [Refunded] [Disputed] [Pending] ││
│  │                                                               ││
│  │ Date [Jan 1-26 ▼] Product [All ▼] Amount [$0-$500]          ││
│  │ Channel [All ▼] Payment Method [All ▼] Status [All ▼]       ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Txn ID  │ Date    │Customer     │Amount│Product │Method│Stat ││
│  ├─────────┼─────────┼─────────────┼──────┼────────┼──────┼─────┤│
│  │ TXN-421 │ Jan 26  │ john@..     │$59.99│Premium │Visa  │ ✅  ││
│  │ TXN-420 │ Jan 26  │ jane@..     │$24.99│Basic   │MC    │ ❌  ││
│  │ TXN-419 │ Jan 25  │ mike@..     │$59.99│Premium │Visa  │ ↩️  ││
│  │ TXN-418 │ Jan 25  │ sara@..     │$14.99│Starter │Amex  │ ⚠️  ││
│  │ ...     │         │             │      │        │      │     ││
│  └──────────────────────────────────────────────────────────────┘│
│  Showing 1-50 of 13,600           [← 1 2 3 ... →] [Export CSV] │
│                                                                  │
│  ── Click a row to see full customer journey ──                  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ TXN-421 — John Doe (john@email.com)                          ││
│  │                                                               ││
│  │ CUSTOMER TIMELINE:                                            ││
│  │                                                               ││
│  │ Oct 15  ●─── Signed up (Starter, $14.99/mo, via Google Ads) ││
│  │ Nov 15  ●─── Renewal — $14.99 — Visa — ✅ Success            ││
│  │ Nov 28  ●─── Upgraded to Basic — $24.99/mo                   ││
│  │ Dec 15  ●─── Renewal — $24.99 — Visa — ✅ Success            ││
│  │ Dec 22  ●─── Added "Priority Support" — +$9.99/mo            ││
│  │ Jan 15  ●─── Renewal — $34.98 — Visa — ❌ Failed (expired)  ││
│  │ Jan 16  │ ●─ Card updated (auto) — ✅ Recovered              ││
│  │ Jan 20  ●─── Upgraded to Premium — $59.99/mo                 ││
│  │ Jan 26  ●─── Renewal — $59.99 — Visa — ✅ Success            ││
│  │                                                               ││
│  │ Lifetime: 103 days │ Total: $229.93 │ 7 transactions         ││
│  │ Status: Active │ Plan: Premium │ Health: ✅                    ││
│  └──────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

**Universal across all industries.** A telehealth company sees a patient's payment history. A SaaS company sees a user's subscription journey. An eCommerce company sees order history.

---

### Report 12: Custom Report Builder

**Who needs it:** Power users, analysts
**Question it answers:** "I have a question none of the standard reports answer."

```
┌─────────────────────────────────────────────────────────────────┐
│  CUSTOM REPORT BUILDER                 [My Reports ▼] [+ New]   │
│                                                                  │
│  ┌────── DATA ──────┐  ┌────── GROUP BY ──────────────────────┐ │
│  │                   │  │                                      │ │
│  │ ● Transactions    │  │ [✓ Product] [✓ Channel] [  Date  ]  │ │
│  │ ○ Subscriptions   │  │ [  Plan  ] [ Region ] [Cust. Age]  │ │
│  │ ○ Customers       │  │ [Pay Method] [ Segment ]            │ │
│  │ ○ Revenue Summary │  │                                      │ │
│  └───────────────────┘  └──────────────────────────────────────┘ │
│                                                                  │
│  ┌────── METRICS ────────────────────────────────────────────┐  │
│  │ Revenue:    [✓ Total Rev] [✓ Net Rev] [  ARPU ] [ MRR  ] │  │
│  │ Customers:  [✓ Count   ] [  New   ] [ Churn%] [ LTV  ] │  │
│  │ Payments:   [  Success%] [ Failed ] [Recov% ] [$ Lost ] │  │
│  │ Retention:  [  M1 Ret  ] [ M3 Ret ] [ M6 Ret] [LTV:CAC] │  │
│  │ Financial:  [  Margin  ] [ Profit ] [ Costs ] [  AOV  ] │  │
│  │             ... 50+ metrics available                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──── CHART ────┐  ┌──── FILTERS ──────────────────────────┐  │
│  │ ● Table       │  │ Date: [Last 30 Days ▼]                │  │
│  │ ○ Line Chart  │  │ Product: [All ▼]                       │  │
│  │ ○ Bar Chart   │  │ Channel: [All ▼]                       │  │
│  │ ○ Pie Chart   │  │ Region: [All ▼]                        │  │
│  │ ○ Pivot Table │  │                                        │  │
│  └───────────────┘  └────────────────────────────────────────┘  │
│                                                                  │
│  ── PREVIEW ──────────────────────────────────────────────────── │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Product │ Channel   │ Total Rev│ Net Rev │Count│          │ │
│  ├─────────┼───────────┼──────────┼─────────┼─────┤          │ │
│  │ Premium │ Google    │  $48.2k  │ $45.1k  │ 380 │          │ │
│  │ Premium │ Organic   │  $42.8k  │ $41.2k  │ 340 │          │ │
│  │ Basic   │ Facebook  │  $38.4k  │ $34.8k  │ 520 │          │ │
│  │ Basic   │ Google    │  $32.1k  │ $30.2k  │ 410 │          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  [Save Report] [Share] [Schedule Email] [Export CSV] [Export PDF]│
└─────────────────────────────────────────────────────────────────┘
```

---

## Industry Modules

These are add-on report packs that extend the core platform for specific verticals.

---

### High-Risk Merchant Module

**For:** Nutraceuticals, supplements, digital products, continuity offers

This is where the current Beast Insights' specialized reports live. Everything that's BIN/MID/gateway-specific moves here.

| Report | What It Does |
|---|---|
| **MID Health & Compliance** | Track CB rates vs Visa VAMP / MC EFM thresholds per MID. "Days to breach" predictions. MID portfolio health scores. |
| **BIN Routing Optimizer** | Gateway × BIN performance matrix. Specific routing recommendations with $ impact. Routing change tracker. |
| **Decline Recovery Intelligence** | Retry windows by decline reason. Recovery gap analysis. Optimal retry strategy per BIN/gateway. |
| **Alert Service Management** | RDR / Ethoca / CDRN effectiveness. Coverage gaps. Alert ROI per service. |
| **Gateway Performance** | Per-gateway approval rates, capacity tracking, cascade rates. Gateway comparison for same BIN. |
| **Affiliate Quality** | Per-affiliate CB rate, cancel rate, LTV, fraud signals. Quality scoring and trending. |

```
┌─────────────────────────────────────────────────────────────────┐
│  HIGH-RISK MODULE — added to navigation:                         │
│                                                                  │
│  PROTECT (expanded)                                              │
│    ├── MID Health & Compliance                                   │
│    ├── Alert Service Management                                  │
│                                                                  │
│  OPTIMIZE (expanded)                                             │
│    ├── BIN Routing Optimizer                                     │
│    ├── Decline Recovery Intelligence                             │
│    ├── Gateway Performance                                       │
│                                                                  │
│  ACQUIRE (expanded)                                              │
│    ├── Affiliate Quality                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

### Telehealth Module

**For:** Virtual care platforms, patient membership models, prescription services

| Report | What It Does |
|---|---|
| **Patient Membership Analytics** | Active patients by plan, geographic distribution (licensed states), membership growth by service type. |
| **Provider Performance** | Revenue per provider, patient retention by provider, consultation completion rates, rebooking rates. |
| **Visit-to-Payment Conversion** | % of visits resulting in payment, average revenue per visit, unbilled visits, insurance claim status. |
| **Insurance vs Self-Pay** | Revenue split by payment type. Reimbursement rates. Denial rates. Collection timelines. |
| **Compliance Dashboard** | State licensing coverage, prescription tracking, regulatory filing status. |

---

### SaaS Module

**For:** Software companies with subscription billing

| Report | What It Does |
|---|---|
| **ARR Tracking** | Annual recurring revenue with expansion/contraction/churn waterfall. ARR by segment (SMB/Mid/Enterprise). |
| **Trial-to-Paid Conversion** | Trial funnel analysis. Conversion by plan, channel, feature usage. Time-to-convert distribution. |
| **Seat & Usage Analytics** | Revenue per seat, utilization rates, overage billing, usage-based revenue trends. |
| **Plan Migration Intelligence** | Upgrade/downgrade flows with reasons. Price sensitivity analysis. Best upgrade triggers. |
| **Net Dollar Retention** | NDR by cohort, segment, plan. Expansion revenue vs churn. Target: >100% NDR. |

---

### eCommerce Subscription Module

**For:** Meal kits, subscription boxes, replenishment models

| Report | What It Does |
|---|---|
| **Order Frequency & Skip Rate** | How often customers order, skip rates by product/month, skip-to-cancel correlation. |
| **Return & Exchange Analytics** | Return rates by product/category, return reasons, impact on LTV, serial returners. |
| **Seasonal Demand Patterns** | Revenue seasonality, inventory implications, best/worst months by product. |
| **Basket Analysis** | What products are bought together, cross-sell opportunities, AOV drivers. |
| **Delivery & Fulfillment** | Order-to-delivery time, delivery failures, impact on retention. |

---

## Universal Metrics Catalog (Core Platform)

### Tier 1: Everyone Sees These

| # | Metric | Category |
|---|---|---|
| 1 | Total Revenue | Revenue |
| 2 | Net Revenue (after refunds + disputes) | Revenue |
| 3 | New Customer Revenue | Revenue |
| 4 | Recurring Revenue | Revenue |
| 5 | MRR (Monthly Recurring Revenue) | Revenue |
| 6 | New Customers | Growth |
| 7 | Active Subscribers | Growth |
| 8 | Net Subscriber Growth | Growth |
| 9 | Churn Rate (total) | Retention |
| 10 | Voluntary Churn Rate | Retention |
| 11 | Involuntary Churn Rate | Retention |
| 12 | Payment Success Rate | Payments |

### Tier 2: Operational

| # | Metric | Category |
|---|---|---|
| 13 | AOV (Average Order Value) | Revenue |
| 14 | ARPU (Avg Revenue Per User) | Revenue |
| 15 | Revenue Growth Rate (MoM/WoW) | Revenue |
| 16 | Revenue per Customer (30/60/90d) | Revenue |
| 17 | New MRR | Subscription |
| 18 | Expansion MRR | Subscription |
| 19 | Contraction MRR | Subscription |
| 20 | Churned MRR | Subscription |
| 21 | Net MRR Movement | Subscription |
| 22 | M1 / M3 / M6 / M12 Retention % | Retention |
| 23 | Churn by Reason (cancel / fail / refund / dispute) | Retention |
| 24 | Failed Payment Count | Payments |
| 25 | $ Lost to Failed Payments | Payments |
| 26 | Recovery Rate | Payments |
| 27 | $ Recovered | Payments |
| 28 | Recovery Gap | Payments |
| 29 | Refund Count | Disputes |
| 30 | Refund Rate | Disputes |
| 31 | Chargeback Count | Disputes |
| 32 | Chargeback Rate | Disputes |

### Tier 3: Growth & Acquisition

| # | Metric | Category |
|---|---|---|
| 33 | Customer Acquisition Cost (CAC) | Acquisition |
| 34 | LTV (30 / 60 / 90 / 180 day) | LTV |
| 35 | LTV:CAC Ratio | LTV |
| 36 | Payback Period (days) | LTV |
| 37 | Conversion Rate (visitor → customer) | Acquisition |
| 38 | Channel Quality Score | Acquisition |
| 39 | Revenue Concentration % | Risk |
| 40 | Revenue Quality Score | Risk |
| 41 | Plan Distribution % | Product |
| 42 | Upgrade Rate | Product |
| 43 | Downgrade Rate | Product |

### Tier 4: Financial

| # | Metric | Category |
|---|---|---|
| 44 | Gross Profit | Financial |
| 45 | Gross Margin % | Financial |
| 46 | Processing Cost | Financial |
| 47 | CAC Cost (total) | Financial |
| 48 | COGS / Service Cost | Financial |
| 49 | Dispute Fees | Financial |
| 50 | Net Dollar Retention (NDR) | Financial |

---

## Universal Dimensions (Group By)

| Dimension | Available In | Example Values |
|---|---|---|
| Product / Plan | All reports | "Premium Plan", "Basic Plan" |
| Channel / Source | Revenue, Acquisition, Lifecycle | "Google Ads", "Organic", "Email" |
| Date (Day/Week/Month) | All reports | Time-based grouping |
| Customer Segment | Revenue, Lifecycle, Churn | "New", "Returning", "At-Risk" |
| Geography (Country/Region) | Revenue, Payments, Acquisition | "US", "Canada", "UK" |
| Payment Method | Payments, Revenue | "Visa", "MC", "ACH", "PayPal" |
| Customer Age (tenure) | Churn, Lifecycle, Retention | "0-30d", "31-90d", "91-180d" |
| Billing Cycle | Retention, Churn, Payments | "Cycle 1", "Cycle 2", "Cycle 6+" |
| Price Tier | Revenue, Product | "$0-25", "$25-50", "$50+" |

High-risk module adds: Gateway, BIN, Bank, Affiliate, MID, Offer
Telehealth module adds: Provider, Service Type, State, Insurance Type
SaaS module adds: Company Size, Seat Count, Usage Tier
eCommerce module adds: Category, SKU, Delivery Method

---

## What Changes vs Current Beast Insights

### Reports Moved to Core (Universal)

| Current Report | Core Equivalent | What Changed |
|---|---|---|
| Dashboard | Business Command Center | Action-oriented, not just numbers |
| Sales | Revenue Analytics | Added concentration risk, quality scores |
| LTV + Retention | Customer Lifecycle | Combined into one story, added churn reasons |
| — (new) | Subscription Intelligence | MRR waterfall, plan migration, subscriber flow |
| — (new) | Churn Analysis | Voluntary vs involuntary, at-risk prediction |
| Approval % + Decline Recovery | Payment Health & Recovery | Simplified — no BIN/gateway, universal language |
| — (new) | Channel & Acquisition | Universal channel scoring (not affiliate-specific) |
| Profitability | Financial Performance | Universal cost structure, not high-risk specific |
| — (new) | Product & Plan Performance | Plan migration flows, add-on adoption |
| Order Details | Transaction Explorer | Added customer journey timeline |
| — (new) | Custom Report Builder | Self-service analytics |

### Reports Moved to High-Risk Module

| Current Report | Module Report | Why It's Industry-Specific |
|---|---|---|
| MID Performance | MID Health & Compliance | Only high-risk merchants manage 100+ MIDs |
| Routing Insights | BIN Routing Optimizer | BIN-level routing is high-risk specific |
| Decline Recovery (advanced) | Decline Recovery Intelligence | Retry windows per BIN/gateway is specialized |
| Alert Analytics | Alert Service Management | RDR/Ethoca/CDRN are high-risk tools |
| CB & Refunds Growth | Part of core Disputes + module | CB growth curves stay in core, MID-level goes to module |

### Reports Removed / Merged

| Removed | Reason |
|---|---|
| CB & Refunds (standalone) | Merged into core Payment Health + Churn Analysis |
| Live Pulse (hourly detail) | Merged into Real-Time Pulse (simpler, universal) |
| MID Health Monitor | Merged into module MID Health & Compliance |

---

## Navigation: What Users See

### Core Platform (All Industries)

```
┌─────────────────────────────────────────────────────────────────┐
│  BEAST INSIGHTS                                    [Client ▼]    │
│                                                                  │
│  MONITOR                                                         │
│    ├── Command Center                                            │
│    ├── Real-Time Pulse                                           │
│                                                                  │
│  GROW                                                            │
│    ├── Revenue Analytics                                         │
│    ├── Subscription Intelligence                                 │
│    ├── Customer Lifecycle                                        │
│                                                                  │
│  RETAIN                                                          │
│    ├── Churn Analysis                                            │
│    ├── Payment Health & Recovery                                 │
│                                                                  │
│  ACQUIRE                                                         │
│    ├── Channel & Acquisition Performance                         │
│                                                                  │
│  PROFIT                                                          │
│    ├── Financial Performance                                     │
│    ├── Product & Plan Performance                                │
│                                                                  │
│  EXPLORE                                                         │
│    ├── Transaction Explorer                                      │
│    ├── Custom Report Builder                                     │
│                                                                  │
│  MY STUFF                                                        │
│    ├── Personal Dashboard                                        │
│    ├── Saved Reports                                             │
│    ├── Smart Alerts                                              │
└─────────────────────────────────────────────────────────────────┘
```

### With High-Risk Module Enabled

```
  RETAIN (expanded)
    ├── Churn Analysis
    ├── Payment Health & Recovery
    ├── MID Health & Compliance          ← MODULE
    ├── Alert Service Management         ← MODULE

  OPTIMIZE                               ← NEW SECTION
    ├── BIN Routing Optimizer            ← MODULE
    ├── Decline Recovery Intelligence    ← MODULE
    ├── Gateway Performance              ← MODULE

  ACQUIRE (expanded)
    ├── Channel & Acquisition Performance
    ├── Affiliate Quality                ← MODULE
```

### With Telehealth Module Enabled

```
  GROW (expanded)
    ├── Revenue Analytics
    ├── Subscription Intelligence
    ├── Customer Lifecycle
    ├── Provider Performance             ← MODULE

  RETAIN (expanded)
    ├── Churn Analysis
    ├── Payment Health & Recovery
    ├── Visit-to-Payment Conversion      ← MODULE

  PROFIT (expanded)
    ├── Financial Performance
    ├── Product & Plan Performance
    ├── Insurance vs Self-Pay            ← MODULE
```

---

## Implementation Approach

### Phase 1: Core Foundation

Build the universal platform that works for ALL industries.

| Priority | Report | Why First |
|---|---|---|
| 1 | Business Command Center | This IS the product — the first thing every user sees |
| 2 | Revenue Analytics | Every business needs revenue reporting |
| 3 | Subscription Intelligence | MRR tracking is the #1 requested feature for subscription businesses |
| 4 | Payment Health & Recovery | Payment failures affect every business |
| 5 | Transaction Explorer | Daily operational need |
| 6 | Customer Lifecycle | LTV and retention — core to every subscription business |

Platform features in Phase 1:
- Core filters and date pickers
- Export (CSV, PDF)
- Saved views

### Phase 2: Retention + Acquisition + Profit

| Priority | Report | Why |
|---|---|---|
| 7 | Churn Analysis | Retention is cheaper than acquisition |
| 8 | Channel & Acquisition Performance | Know where to spend marketing budget |
| 9 | Financial Performance | Profitability reporting |
| 10 | Product & Plan Performance | Product-level insights |

Platform features in Phase 2:
- Saved filter presets
- Scheduled email delivery
- Real-Time Pulse

### Phase 3: Self-Service + Modules

| Priority | Report | Why |
|---|---|---|
| 11 | Custom Report Builder | Self-service analytics |
| 12 | High-Risk Module (all 6 reports) | Serve existing customer base |
| 13 | Personal Dashboard | Customizable home screen |
| 14 | Smart Alerts | Proactive notifications |

### Phase 4: Industry Expansion

| Priority | Module | Why |
|---|---|---|
| 15 | Telehealth Module | First expansion industry |
| 16 | SaaS Module | Large addressable market |
| 17 | eCommerce Module | High volume opportunity |

---

## Competitive Positioning

```
                          Industry-Specific ───────────────────┐
                          │                                     │
                          │  BEAST INSIGHTS                     │
                          │  (Core + Industry Modules)          │
                          │  ┌─────────────────────┐            │
                          │  │ Universal core +     │            │
                          │  │ deep industry modules│            │
                          │  │ + actionable insights│            │
                          │  │ + any processor      │            │
                          │  └─────────────────────┘            │
                          │                                     │
  Generic ────────────────┼─────────────────────── Actionable   │
  │                       │                            │        │
  │  ChartMogul           │           VRIO             │        │
  │  Baremetrics          │     (high-risk only)       │        │
  │  ProfitWell           │                            │        │
  │  (SaaS metrics only,  │                            │        │
  │   Stripe-dependent)   │                            │        │
  │                       │                            │        │
  │  Stripe Dashboard     │                            │        │
  │  (basic, one processor)│                           │        │
  │                       │                            │        │
  Generic ────────────────┼────────────────────────────┘        │
                          │                                     │
                          Industry-Specific ───────────────────┘

Beast Insights' position:
  ✓ Works with ANY payment processor (not just Stripe)
  ✓ Universal core that works for all industries
  ✓ Deep industry modules for specialized needs
  ✓ Actionable (Problem → Cause → Action)
  ✓ Customizable (saved views, custom reports, personal dashboards)
  ✓ Multi-tenant SaaS (serve many clients from one platform)
```

### Key Differentiators

| vs Competitor | Beast Insights Advantage |
|---|---|
| vs Stripe Dashboard | Processor-agnostic, multi-processor, deeper analytics |
| vs ChartMogul/Baremetrics | Industry modules, actionable insights (not just charts), custom reports |
| vs VRIO | Universal core for any industry (not high-risk only), modern UX |
| vs Building In-House | Faster to deploy, maintained platform, industry best practices |

---

## Data Model: What Needs to Change

The current data model (`data.orders_{client_id}`) is heavily high-risk-specific (81 columns with CRM, gateway, BIN, affiliate fields). For cross-industry expansion, the core data model should be:

### Universal Data (every client)

```
Transactions
  - id, customer_id, date, amount, currency
  - type (new, renewal, one-time, upgrade, downgrade)
  - status (success, failed, refunded, disputed, pending)
  - product_id, plan_id
  - payment_method (visa, mc, amex, ach, paypal, etc.)
  - channel / source
  - geography (country, region)
  - failure_reason (if failed)
  - refund_reason (if refunded)

Customers
  - id, created_date, status (active, churned, paused)
  - current_plan, plan_start_date
  - acquisition_channel, acquisition_cost
  - lifetime_value, total_transactions

Subscriptions
  - id, customer_id, plan_id, status
  - start_date, end_date, cancel_date
  - cancel_reason
  - billing_cycle_count

Products / Plans
  - id, name, price, billing_interval
  - category, type (subscription, one-time, add-on)

Channels / Sources
  - id, name, type (organic, paid, referral, email, affiliate)
  - cost_per_acquisition
```

### High-Risk Extension (module)

```
  + gateway_id, mid_id, bin, bank_name
  + affiliate_id, offer_id, campaign_id
  + decline_group, cascade_level, retry_count
  + chargeback_date, alert_type, alert_service
  + routing rules, capacity data
```

### Telehealth Extension (module)

```
  + provider_id, service_type, visit_date
  + insurance_type, claim_status, reimbursement_amount
  + licensed_state, prescription_id
```

The key insight: the **core data model is a subset** of the current high-risk model. High-risk fields become module-specific extensions. New industries add their own extensions.

---

## Summary

```
CURRENT STATE:
  1 platform → 1 industry (high-risk) → 13 specialized reports

FUTURE STATE:
  1 platform → any industry → 12 universal reports + industry modules

What stays:      Problem → Cause → Action philosophy
                 JSON-driven, customizable, saved views
                 Fast (<1 sec), no Power BI dependency

What changes:    Core reports use universal language
                 Industry-specific features become opt-in modules
                 Data model supports extensions per industry
                 Navigation adapts based on enabled modules

Result:          Sell to telehealth, SaaS, eCommerce, wellness,
                 and any subscription business — not just high-risk
```

---

*Beast Insights — Universal payment intelligence for every industry.*

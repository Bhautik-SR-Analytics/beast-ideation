# Decline Reprocessing / Cascade Engine — CCW (10070)

**Status:** built and **proven live** 2026-08-28. Migrations 68–76 applied to prod; API tested locally against real declined orders with `reprocess_dry_run: false`. Not yet deployed, not yet integrated by CCW.
**Scope agreed:** all checkout positions P1–P4 · CCW turns off their 6-attempt backend loop **and** the Sticky
campaign cascade profile · delayed retries allowed with a callback · ~90s ladder, 3 retries.

> **Correction, 28 Aug — the headline claim in the first version of this document was wrong.**
> It said switching MID was worth ~20 points. It is worth about **5**. The other ~20 belong to the **delay**
> before the retry, which CCW's own loop never spends and which the first draft of this plan treated as an
> optional extra. Section 2 carries the decomposition. Everything downstream — the ladder, the policy schema,
> the seeded delays — is built around timing as the primary lever as a result.

Companion docs: [`sticky-io-decline-reprocessing.md`](./sticky-io-decline-reprocessing.md) (API surface, tested),
[`ROUTING-ENGINE-V1-DECISIONS.md`](./ROUTING-ENGINE-V1-DECISIONS.md) (the first-attempt engine this extends),
[`CCW-ROUTING-API-INTEGRATION.md`](./CCW-ROUTING-API-INTEGRATION.md) (the contract CCW already builds against).

This is **not** the `reprocessing/` repo. That one recovers declined subscription **rebills** for Konnektive
hours-to-days later. This is a **checkout-time cascade** for **initials and upsells** on Sticky, measured in
seconds. Different CRM, different grain, different clock. They will eventually share a policy vocabulary; they
do not share code in v1.

---

## 1. What CCW does today, measured

P1 = product 3047, last 30 days, `reporting.orders_enriched_10070`.

| | P1 | P2 | P3 | P4 |
|---|---|---|---|---|
| First attempts | 30,237 | 43,743 | 2,905 | 815 |
| Organic AR | 46.1% | 77.8% | 58.2% | 57.1% |
| First-attempt declines / 30d | 16,292 | 9,719 | 1,214 | 350 |
| Avg ticket | $100 | $31 | $29 | $42 |

**Their retry loop.** Of 16,198 customers whose first P1 attempt declined, **38.9% eventually approve** —
median at attempt 3, median 122 seconds. But they spend **4.56 attempts** on average (median 4, p90 9) to get
there, and the attempts arrive **in pairs on the same MID about 2 seconds apart** — the
`NewOrderWithProspect` → immediate `NewOrder` double from their flowchart. The second half of every pair is a
wasted authorisation.

**Approval by n-th distinct MID tried** (customers whose first attempt declined):

| n-th MID | customers reaching | approved | AR | avg calls burned on that MID |
|---|---|---|---|---|
| 1st (staying put) | 17,467 | 2,078 | **11.9%** | 2.71 |
| 2nd | 9,499 | 3,058 | **32.2%** | 2.12 |
| 3rd | 3,687 | 1,052 | **28.5%** | 2.13 |
| 4th | 1,400 | 336 | 24.0% | 2.13 |
| 5th | 544 | 129 | 23.7% | 2.12 |
| 6th | 201 | 33 | 16.4% | 2.29 |

Two things fall out of that table. Moving to a second MID is worth ~20 points over trying the same one again,
and the return holds up through the fifth MID before it collapses. And **only 9,499 of 17,467 declined
customers ever reach a second MID** — roughly 8,000 customers a month are retried into the same wall and then
abandoned. That gap, not the ladder depth, is the prize.

---

## 2. What the decline reason actually tells us

First-attempt P1 declines, 30 days. Retry AR is measured on the next attempt in the customer's chain.

| First decline | Share of declines | Same-MID retry | Different-MID retry |
|---|---|---|---|
| Pick Up Card - SF | 34.8% | 7.4% | 24.7% |
| Insufficient Funds | 20.9% | 1.7% | 22.0% |
| Do Not Honor | 18.0% | 7.9% | 27.6% |
| Issuer Declined | 16.1% | 7.2% | 21.7% |
| Invalid CVV | 3.2% | 0.9% | 22.8% |
| Account Closed | 0.9% | 0.8% | 14.8% |
| Expired Card | 0.5% | 1.1% | 21.0% |
| Blocked, first used | 0.6% | 0.3% | 12.7% |

The top four are 89.8% of all declines and every one of them responds to a MID switch. Note **Insufficient
Funds behaves like an issuer decline here, not like a funds problem** — 1.7% on the same MID against 22.0% on
another one, inside the same session. The received wisdom ("NSF means wait for payday") is a rebill rule; it
does not describe this traffic.

**Switching acquirer beats switching MID** (45 days, all P1 retries after a decline):

| | retries | AR |
|---|---|---|
| Same MID | 70,270 | 5.7% |
| Different MID, same acquirer | 5,356 | 20.7% |
| Different acquirer | 18,074 | **24.5%** |

### The timing decomposition — the correction that reshaped this plan

The table above is confounded, and badly. CCW's different-MID retries also **waited longer** (median 100s
against 33s at step 2→3), so "switching recovers 24.5%" silently contains an unknown amount of "waited longer".
Splitting the same 45 days by the gap to the retry separates them:

| gap to retry | same MID | different MID |
|---|---|---|
| ≤5s | 4.0% | 8.6% |
| 5–60s | 6.6% | 13.6% |
| 1–5m | **25.6%** | **30.1%** |
| 5–60m | 38.7% | 33.3% |
| >1h | 38.0% | 34.3% |

Read down a column and you get the delay effect: **~20 points**. Read across a row and you get the switch
effect: **~5 points, consistently**. Waiting is four times the lever that switching is, and a same-MID retry
after two minutes beats a different-MID retry after two seconds by 17 points.

CCW's loop fires its duplicate 2 seconds later and collects the 4%.

Two consequences the rest of this document is built on. First, `delay_seconds` is policy, per reason class, and
it is the primary control rather than an escape hatch for scheduled retries. Second — and this is the
uncomfortable one — **the band where approval doubles starts at 1 minute, and a ladder that must finish inside
~90 seconds barely reaches it.** Rungs 1 and 2 land in the 5–60s band. Widening that means the customer waits
longer or the last rung is answered by callback; it is a product decision, not a tuning one.

The gap effect is itself partly selected — a customer who retries an hour later may have moved money or fixed
their card — which is why the 5–60m and >1h rows should be read as an upper bound rather than a target. The
≤5s → 1–5m contrast within the same-MID column (4.0% → 25.6%) is the part that is hard to explain away.

**The CVV case is the sharpest signal in the whole dataset.** After an `Invalid CVV` decline, the next attempt
approves at **0.8%** if it lands on a CVV-enforcing MID (n=118) and **27.0%** if it does not (n=684). And CVV
enforcement is readable straight from our own data — no master sheet needed, though it is worth cross-checking
against one:

| MID | alias | AR | share of its attempts declined `Invalid CVV` |
|---|---|---|---|
| 321 | VAL-ccwapphelp.com-PayArc8542 | 7.9% | **69.4%** |
| 245 | MPAT-registertodayccw-Maverick4058 | 12.0% | **45.7%** |
| 302 | Kunpark-ccwpermitapphelp-Q4372 | 27.6% | 5.5% |
| …all other MIDs with volume | | | < 4% |

MID 321 is declining seven in ten transactions for a missing CVV at a 7.9% approval rate. That is not a routing
subtlety, it is a MID being fed traffic it cannot accept.

### Two findings that change how a reason-based rule must be written

**1. The decline vocabulary is processor-specific.** Paysafe MIDs return `Issuer Declined`; the Q / Chesapeake /
Maverick / SignaPay family returns `Pick Up Card - SF` and `Do Not Honor`. Measured over 30 days, every Paysafe
MID shows **0.0%** `Pick Up Card - SF` and **0.0%** `Do Not Honor`, while the Q-family MIDs run 30–60% `Pick Up
Card - SF`. The same customer refusal reaches us under different strings depending on who declined it. Any
policy keyed on the raw reason will silently mean different things on different MIDs — reasons must be
normalised **per processor family** before they are used to decide anything.

`orders_enriched.decline_group` already provides most of this (`Issuer Decline`, `Insufficient Funds`,
`CVV Mismatch`, `Customer Account Issue`, `Gateway/Network Issue`, `Expired Card`, `Invalid Card/Type/Amount`,
`Fraudulent`) and correctly separates `Pick Up Card - L`/`- S` (Fraudulent, lost/stolen) from `Pick Up Card - SF`
(Issuer Decline). It is the right starting vocabulary; it is not per-processor yet.

**2. Some issuer × acquirer cells are structurally dead.** Capital One on Paysafe approves **0.0%** — 344 first
attempts in the last 30 days, ~8,000 attempts over four months, essentially zero approvals in every single week:

| week | CapOne → Paysafe attempts | AR | CapOne → everyone else | AR |
|---|---|---|---|---|
| 2026-04-26 | 830 | 0.0% | 1,200 | 7.2% |
| 2026-06-28 | 499 | 0.0% | 1,415 | 3.2% |
| 2026-08-02 | 559 | 0.0% | 1,684 | 3.0% |
| 2026-08-23 | 106 | 0.0% | 1,561 | 24.3% |

Over the same 30 days Capital One approves **45.8% on Synovus**. A cascade that does not know this will spend a
rung on a cell that cannot approve. This belongs in the **first-attempt engine too**, not only in the retry path.

### Downstream cost of recovered orders

Orders 60–150 days old, so chargebacks have matured:

| First outcome | approvals | CB | Alerts | Refunds |
|---|---|---|---|---|
| Approved on attempt 1 | 36,702 | 1.26% | 3.68% | 11.89% |
| Recovered after Pick Up Card - SF | 6,831 | 1.04% | **5.97%** | 11.36% |
| Recovered after Do Not Honor | 3,833 | 0.89% | 1.10% | 9.65% |
| Recovered after Insufficient Funds | 2,326 | 1.20% | 2.88% | 11.05% |
| Recovered after Invalid CVV | 483 | 1.24% | **7.66%** | 10.14% |

Recovery does **not** buy chargebacks — CB rate is flat or better than a first-attempt approval. Alerts run
1.6–2× higher on the Pick-Up-Card and CVV classes. Worth tracking as a cost of the programme; not a reason to
withhold the retry.

### The honest caveat on every number above

CCW's MID choice on a retry is not random — their loop picks MIDs for reasons we cannot see, so
"different MID recovers 24.5%" is an observational contrast, not a causal effect. The direction is large,
consistent across every reason, and has an obvious mechanism, so it is safe to build on. The **magnitude** is
not safe to promise. Section 7 keeps a control group specifically so we can price this properly rather than
quoting a backtest — the same lesson as the first-attempt engine, where backtests overstated by ~2×.

---

## 3. Target flow

```
customer submits card
        │
        ├─►  POST /api/v1/payments/route          (LIVE today)
        │      { bin, tag, product_id, email } → { gateway_id }
        │
        ├─►  CCW creates the order in Sticky with forceGatewayId
        │
        │    approved ──────────────────────────────────► done
        │    declined
        │        │
        └─►  POST /api/v1/payments/reprocess       (NEW)
               { order_id, decline_reason }
                     │
                     │  attempt 1  order_reprocess       + forceGatewayId  (no new order row)
                     │  attempt 2  order_reprocess       + forceGatewayId  (no new order row)
                     │  attempt 3  new_order_card_on_file + forceGatewayId + cascade_override=1
                     │
                     └─► { status: approved | declined | scheduled, order_id, gateway_id, attempts[] }
                                                    │
                                  scheduled ────────┴─► worker fires at T+5/15/30m ─► callback to CCW
```

CCW makes **one** call. Everything inside it is ours.

### Why two `order_reprocess` before one `new_order_card_on_file`

`order_reprocess` retries on the same order record and creates **no new order row**. That keeps Sticky clean,
keeps the customer on one order id for fulfilment, and avoids the lineage problem
(`new_order_card_on_file` produces an order whose only link to the original is a free-text note —
`ancestor_id` and `parent_id` are useless, confirmed by test). `new_order_card_on_file` earns its place as the
third rung because it is the only path that can **re-price**, the only one that accepts `cascade_override`, and
the only one available when the order is no longer in a reprocessable state (`374 – cannot reprocess
non-declined orders`).

Attempt count is a **policy value per position and reason class**, not a constant. The data supports 3 rungs
comfortably and a 4th at ~24%; P3/P4 have too little volume to justify more than 2.

### The seeded ladder, and the split test riding on it

`{15, 35, 40}` — rungs at **+15s, +50s, +90s** from the decline. That is the ~90 second budget agreed on
28 Aug, and per §2 it buys the 5–60s band on rungs 1–2 and just touches 1–5m on rung 3.

Rung 1 carries a **same-MID vs different-MID split test** (`same_mid_split_pct`, seeded at 40% same). It is a
split rather than a decision because history cannot settle it: only **2.7%** of CCW's own 1→2 retries ever
switched MID, so the different-MID arm at that step rests on ~1,000 attempts against 36,637. Any fixed number
would be an extrapolation dressed as a decision.

Where history *is* conclusive the split is forced to 0 and the generator may not override it:

| | same-MID | evidence |
|---|---|---|
| Insufficient Funds | **0.1%** | 15,541 same-MID duplicate attempts |
| Invalid CVV | **0.0%** | n=986 at step 1→2 |

Those are not low rates, they are zero with a large denominator. Rungs 2 and 3 always move MID regardless of
the assignment — the split is about the *first* retry only.

---

## 4. Choosing the retry MID

**Reuse the routing engine. Do not write a second selector.** `services/routingEngine/index.js` already walks
BIN → bank → card_class → card_type → default, filters members by live MID state (eligibility, alias markers,
`status = 'live'`, pacing pauses, kills, manual overrides, disabled rules), and draws by weight. Every guardrail
the first-attempt path respects must apply identically here, and the only way to guarantee that is to run the
same code.

What the retry path adds is **context** — a set of constraints passed into `route()`:

| Constraint | Source | Effect |
|---|---|---|
| `excludeMids[]` | our ledger + the order's `systemNotes` | never re-offer a MID this order already touched |
| `excludeAcquirers[]` | the declining MID's `lender` | prefer a different acquirer (worth +3.8 pts over same-acquirer switch) |
| `requireNoCvvEnforcement` | reason class = CVV Mismatch | drops MIDs whose CVV-decline share exceeds a threshold |
| `deadCellBlocks[]` | issuer × acquirer detector | drops Capital One × Paysafe and anything else measuring ~0% |
| `attempt_no`, `reprocess_id` | request | logged on the decision row for attribution |

`route()` today takes a single `excludeMid` and already accepts `declineReason` (recorded, never used in
selection). Widening one to a set and acting on the other is a small, contained change to a function that is
already on the live checkout path — it needs the same care as any change there, but it is not new machinery.

**Constraints degrade, they never empty the pool.** The existing rung-walk already climbs when a rule's members
are all unavailable; the same rule applies here — if excluding an acquirer leaves nothing, the exclusion is
dropped before the attempt is abandoned. An attempt on an imperfect MID beats no attempt.

### Reason → policy, generated not hardcoded

New table `routing_engine.decline_policy`, one row per (client, processor family, normalised reason), holding:
`retry_allowed`, `same_mid_allowed`, `require_switch_acquirer`, `require_no_cvv_mid`, `max_attempts`,
`delay_seconds`, `allow_reprice`. Generated per client from its own measured recovery by a job in
`routing-engine/clients/<id>/jobs/`, proposed as a draft, reviewed and activated exactly like a ruleset — same
lineage, same approval, same rollback. This is deliberate: per-client, data-driven policy is the standing rule
on this project, and a hand-written reason table would be a global constant wearing a client's name.

Starting classes, from the measured data:

| Class | Reasons | v1 policy |
|---|---|---|
| `switch_mid` | Pick Up Card - SF, Do Not Honor, Issuer Declined, Insufficient Funds | up to 3 attempts, different acquirer each time |
| `cvv` | Invalid CVV, Card Type Verification Error | switch MID **and** require a non-CVV-enforcing MID |
| `card_dead` | Account Closed, No account, Invalid card number, Expired Card | **one** attempt on a different acquirer, then stop (12–21% — real, but thin) |
| `mid_broken` | Invalid merchant ID, Terminal not programmed, Cannot process, Activity limit exceeded | switch MID immediately **and** feed the existing kill/alert detector |
| `fraud` | Pick Up Card - L, Pick Up Card - S, Security violation, `is_fraud`, `is_blacklisted` | **no retry** (0.7% of declines — cheap to honour) |
| `duplicate` | Duplicate Transaction | **no retry** — the risk is a double charge, not a lost sale |
| `3ds` | response_code 101 | no server-side retry; needs the customer at a browser |
| `time_only` | (candidate: NSF where a same-session switch fails) | schedule T+N minutes, callback |

`Activity limit exceeded` is the one reason where same-MID beat different-MID (24.6% vs 18.0%, n=245/50). Small
sample, plausible mechanism (a velocity window that has since rolled), and worth a deliberate test rather than a
guess.

---

## 5. What gets built

### 5.1 The API

`POST /api/v1/payments/reprocess` — same `apiAuth` (`x-client-id` / `x-client-secret`), same
`{ code, message, data }` envelope, same "HTTP 200 whether or not we act" discipline as the route endpoint.

**Request**

| Field | Req | Notes |
|---|---|---|
| `order_id` | **yes** | the declined Sticky order |
| `decline_reason` | no | as Sticky returned it; we normalise. If absent we read it from `order_view` |
| `routing_id` | no | the `routing_id` from the original route call — ties the whole chain to one decision lineage |
| `bin`, `product_id`, `price_point`, `campaign_id`, `email` | no | **send them if you have them** — supplying these lets us skip an `order_view` round trip, which is real latency inside the sync budget |

**Response**

```json
{ "code": 200, "message": "Recovered on attempt 2",
  "data": {
    "status": "approved",
    "reprocess_id": "019fe…",
    "original_order_id": "4948284",
    "order_id": "4948284",
    "gateway_id": 415,
    "attempts": [
      { "n": 1, "api": "order_reprocess", "gateway_id": 391, "result": "declined",
        "decline_reason": "Do Not Honor", "ms": 2140 },
      { "n": 2, "api": "order_reprocess", "gateway_id": 415, "result": "approved", "ms": 1980 }
    ]
  } }
```

`status` is one of `approved` · `declined` · `scheduled` · `not_attempted` (fraud / duplicate / 3DS / policy
off / outside coverage). `order_id` differs from `original_order_id` only when the third rung created one.
For `scheduled`, `next_attempt_at` is included and the outcome arrives by callback.

**Callback** (delayed attempts only): signed `POST` to a CCW-supplied URL carrying the same `data` block, with
retries and a dead-letter. They must be able to receive it before delayed retries are switched on; until then
the scheduler stays dark and the API answers `declined` at the end of the sync ladder.

### 5.2 Where it runs — this needs a decision

`beast-insights-backend` is deployed on Vercel (`vercel.json`, `@vercel/node`, no `maxDuration` set). A handler
that holds a request for 30–60 seconds while making four sequential external calls is a poor fit for serverless
and is not safe at the default function timeout. The first-attempt route endpoint is fine there — it answers in
under 200 ms — but this one is a different shape.

**What was built:** the endpoint lives in `beast-insights-backend` alongside the routing engine, so it reuses
`apiAuth`, the rate limiter, the connection pool, the snapshot cache and `route()` itself. That is a plain
Express handler, which means it can be served **either** from Vercel or from the Linux box — building it there
was a deliberate choice not to fork the engine, not a decision about where it runs.

**The deployment decision is still open and it matters.** A ~90 second handler on Vercel needs an explicit
`maxDuration` and is a poor fit regardless; concurrency is low (~900 declines/day across P1–P4, roughly one
every 90 seconds) so it would probably work, but "probably works" is a bad property for a charge path. The
worker is not optional either way — `reprocessWorker.js` must run as its own process on the Linux box, because
a scheduled charge cannot depend on a web request still being open and would not survive a serverless
function returning.

| Concern | Vercel | Linux box |
|---|---|---|
| 90s handler | needs `maxDuration`, still serverless | fine |
| Scheduled rungs + callback | impossible | the worker already assumes this |
| Deploy path | existing | new service, nginx, TLS |

### 5.3 Database

Migrations written: [`routing-engine/clients/10070/sql/68_reprocess_schema.sql`](./routing-engine/clients/10070/sql/68_reprocess_schema.sql)
and [`69_scope_first_attempt.sql`](./routing-engine/clients/10070/sql/69_scope_first_attempt.sql). Both are
additive and idempotent — no DROPs, no data change, nothing that alters what the live API answers today.
Postgres 16.14, so `ADD COLUMN … NOT NULL DEFAULT <constant>` is metadata-only and does not rewrite the 56 MB
`decision` table. There is no staging database, so these land on prod and are written to be safe there.

#### The one decision this schema encodes: a retry is a routing decision

Retry attempts go into **`decision`**, discriminated by a new `attempt_no`, and the new tables hold only what
`decision` cannot. A self-contained `reprocess_attempt` carrying its own `rule_id`, `arm` and candidate set was
the obvious alternative and it is wrong twice:

- It fails the `rule_event` test that this schema already follows — *does another table already record this
  fact with a real timestamp?* It does. Two records of one event can disagree, and the console,
  `v_rule_performance` and every attribution query would need a second code path to see half the decisions the
  engine makes.
- **It would break P2 inheritance on every recovered order.** `db.js getPositionMid` resolves P2's inherited
  MID from the newest P1 decision. If P1 declines on MID A and we recover it on MID B, a decision log without
  retries hands P2 the MID that just failed. Measured on 30 days of 10070: where P2 followed the declined first
  MID it approved at **3.8%** (n=26) against **81.4%** (n=3,248) where it followed the approving one. It is
  rare today only because CCW's loop picks the retry MID and carries it forward — the moment we own the
  cascade, that 78-point cliff becomes systematic.

**Changes to existing objects** (`68`):

| Object | Change | Why it is safe |
|---|---|---|
| `decision` | `+ attempt_no smallint NOT NULL DEFAULT 1` | metadata-only on PG 16; every existing row is a first attempt and reads as one |
| `decision` | `+ reprocess_id uuid`, `+ parent_decision_id uuid` | nullable; no existing row has either |
| `decision` | `+ decision_parent_fk`, `+ decision_attempt_ck`, `+ decision_reprocess_ck` | added `NOT VALID` then `VALIDATE` separately — no scan under an exclusive lock on a checkout-path table |
| `decision_reason_ck` | widened with 7 `reprocess_*` reason codes | existing 8 values unchanged; existing rows already satisfy it |
| `decision_outcome_method_ck` | `+ 'api_response'` | a cascade outcome is *told* to us synchronously, not guessed at ±5s — it must not be mixed into the matcher's confidence figures |
| indexes | 3 on `decision`, all `CONCURRENTLY` | no write lock; `decision_attempt_time_idx` matters because retries will outnumber first attempts ~3:1 |

**New tables** (`68`), all in `routing_engine`, tenanted on `client_id`, FK'd to `client_config`:

| Table | Holds |
|---|---|
| `reprocess_request` | one row per call from the CRM: order, resolved position/BIN/product, raw reason, normalised class, policy used, coverage draw, final status, latency |
| `reprocess_attempt` | one row per Sticky API call: rung, endpoint, MID chosen, exclusions applied, request/response, result, resulting order id, and `landed_gateway_id` parsed from systemNotes. **Written before the call fires** |
| `reprocess_schedule` | delayed attempts: `fire_at`, lease, worker-retry count, callback state |
| `decline_reason_map` | raw CRM string × **processor family** → reason class, with the evidence behind the mapping |
| `decline_policy` | what to do with a class, per position — versioned and status-gated like `ruleset`, with a partial unique index enforcing one active row |
| `dead_cell` | issuer × acquirer combinations measuring ~0%, with the attempts/approvals that put them there and a release path |

Two details worth not re-deriving:

- **Ledger-first is not bookkeeping pedantry.** No Sticky endpoint has an idempotency key. A timeout on
  `new_order_card_on_file` leaves us unable to say whether a card was charged, and a blind retry creates a real
  duplicate *order*. A row with `fired_at` set and `responded_at` still null is the only signal that a charge
  may exist that we never observed — `reprocess_attempt_unresolved_idx` exists to make that worklist cheap.
- **`landed_gateway_id` is stored separately from `gateway_id` on purpose.** Sticky's campaign cascade can fire
  on top of a forced attempt, and `order_view.gateway_id` then reports the cascade's pick rather than ours. If
  those two columns ever differ, our MID selection did not decide that outcome — and `gateway_id` alone would
  never say so.

#### Blast radius — everything that reads `decision` today

Retry rows in `decision` change the answer to every unfiltered query against it. All of these are first-attempt
assumptions that were true when written and are nowhere stated in SQL:

| Reader | Effect if unscoped | Fix |
|---|---|---|
| `v_coverage` | coverage = routed/all; retries are always routed, so it climbs toward 100% regardless of `coverage_pct` — and coverage is how we prove a control group exists | `69` |
| `v_arm_performance` | blends 46%-approving first attempts with ~25% cascade rungs inside **both** arms; the champion-vs-challenger comparison becomes a measure of decline share | `69` |
| `v_rule_performance` | credits a rule's `actual_ar` with retry outcomes; a rule chosen often for retries looks worse than the identical rule that is not | `69` |
| `lib/signals.py` degradation detector | drop-of-15-points-over-200-decisions would see the retry mix and **auto-pause or kill healthy MIDs** | code: `attempt_no = 1` |
| `jobs/match_outcomes.py` | would try to ±5s match retry decisions to orders that `order_reprocess` never created, and walk forward onto someone else's order — the exact failure fixed on 26 Aug | code: exclude `attempt_no > 1`; those outcomes are written directly as `api_response` |
| `db.js getRecentDecision` (dedup) | returns the newest retry MID instead of the original — **desirable**, but it is a live behaviour change and should be a deliberate one | verify, then keep |
| `db.js getPositionMid` (P2 inherit) | currently returns the newest decision regardless of outcome | change to prefer the attempt that **approved**, else newest — this is the 3.8%-vs-81.4% fix above |
| `lib/facts.py`, `refresh_mid_state` | §5.4 — the warehouse loses cascade attempts entirely | union `reprocess_attempt` |

#### Runbook order — this part is not flexible

1. `68_reprocess_schema.sql` — columns, constraints, tables. Safe against the code running today, which never
   mentions `attempt_no`.
2. `69_scope_first_attempt.sql` — scopes the three views to `attempt_no = 1` and adds the two retry views.
   Also safe today: every existing row has `attempt_no = 1`, so the filtered views return exactly what they
   returned before.
3. Deploy the scoped `signals.py` / `match_outcomes.py` and the `getPositionMid` change.
4. **Only then** deploy anything that writes a retry decision.

Running 4 before 2 or 3 does not error. It quietly changes what four dashboards and one auto-kill detector
mean, which is the failure mode that cost us a day when a challenger policy row landed before the arm-aware
code (`CHALLENGER-HANDOVER.md` §1). Same shape, same fix: order it and say why.

### 5.4 The measurement problem — the biggest hidden cost

> **Corrected 2026-08-28 by the live test.** The first version of this section said the warehouse loses
> cascade attempts entirely. It does not. `order_reprocess` creates no new *order*, but the CRM records each
> **transaction**, and the ETL emits a row per transaction — so both rungs of a cascade appear in
> `orders_enriched` under one `order_id`:
>
> ```
> 4958933  gateway 299  is_approved=0  attempt=4
> 4958933  gateway 428  is_approved=1  attempt=4   <- the recovery
> ```
>
> So per-MID facts and MID health DO see cascade attempts, and the alarm below was overstated. What remains
> true is subtler and still matters: **both rows carry the same `attempt` value**, so the rungs are
> indistinguishable by attempt number, and anything deduplicating on `order_id` collapses them. The fact
> loader filters `attempt_sort = 1`, so cascade rungs are excluded from it either way — which is correct for
> first-attempt rule generation and wrong for anything trying to measure the cascade.

`order_reprocess` does not create an order row. A P1 order cascaded across three MIDs therefore has no new
order id — but it does have a row per transaction, sharing that id.

That breaks things that currently work:

- `lib/facts.load_facts` builds the rule generator's fact base from `attempt_sort = 1` rows keyed on
  `gateway_id`. Post-change, a MID's measured approval rate becomes *"approval rate given it was the last MID
  tried"* — survivorship, pointed the wrong way.
- `mid_state` health, the consecutive-decline detectors and the hard-decline kill all count declines per MID
  from the warehouse. Cascade declines would stop being counted, and MID 370's "148 Invalid merchant ID in a
  row" would have been invisible.
- `decision_outcome` matching pairs a decision to an order within ±5s. Three decisions now map to one order.
- **Reported organic AR will jump for definitional reasons.** Retries stop creating attempt-2 rows, so the same
  underlying performance reads higher. The agreed baseline in `client_config.settings`
  (`agreed_baseline_ar: 41.00`, `agreed_baseline_net_ar: 65.00`, window 2026-06-12 → 2026-08-11) becomes
  non-comparable on the old definition. This is real and expected — it is also exactly the kind of number that
  causes an argument six weeks later if it is not written down now.

**Therefore:** `reprocess_attempt` becomes the system of record for cascade attempts, and the fact loader, the
MID health refresh and the outcome matcher must all union it. This work is not optional and it is not small —
it should be scoped in the same phase as the API, not after it.

The existing `systemNotes` → warehouse parser is the other half of the answer and it already works:
`orders_enriched_10070` carries `routing_decision_id`, `routing_rule`, `routing_arm`, `routing_mid`,
`routing_type`, `routing_first_gateway`, `routing_cascaded`, `routing_mid_honoured` — 23,566 of 26,321 P1 rows
in the last 7 days carry a decision id. Extending that parser to reconstruct the full cascade chain from the
notes gives us an independent check on our own ledger.

### 5.5 Scheduler

Neither backend has a queue (no Bull, BullMQ, node-cron or agenda in either `package.json`). Options were
weighed and the recommendation is a **Postgres-backed queue** (`reprocess_schedule`) drained by a lease-taking
worker loop on the routing-engine host — same deploy path, same `.env`, same operational surface as the
existing jobs, and it survives a restart because the state is in the table rather than in memory. Jenkins cron
gives one-minute granularity at best and would make a 5-minute retry a 5-to-6-minute retry; Vercel cron has the
same granularity plus the timeout problem.

---

## 6. Guardrails carried over unchanged

Everything the first-attempt engine enforces applies to every rung of the cascade, by construction (§4):

- MID eligibility: active in `public.payments`, alias free of `closed` / `keep off` / `do not use` / `dry run` /
  `sandbox` / `test payment`
- `mid_state.status = 'live'` — capped, killed, blocked, paused and manually-disabled MIDs are never offered
- Cap pacing: per-MID windowed budgets, 90% hard kill, auto pause/resume
- `override` / `disabledRules` from the internal console
- Weighted draw from rule members; rung climb on exhaustion; failsafe pool if the DB is unreachable

Two additions specific to the cascade:

- **Never re-offer a MID this order already touched** — from our ledger, not from the warehouse (the warehouse
  will not know, see §5.4)
- **Cap consumption is unaffected by extra attempts** — cap is gross *approved* revenue, so declines cost
  nothing. Recovered approvals do consume it, and pacing already handles that.

---

## 7. Rollout

**Phase 0 — unblock.** None of this can be tested without these:
1. ~~`order_reprocess` permission~~ — **DONE 28 Aug.** All three write methods verified permitted by probing
   against a nonexistent order id (no charge possible): `order_reprocess` → 350, `new_order_card_on_file` →
   10331, `subscription_order_update` → 906.
2. ~~Confirm which Sticky account~~ — **DONE.** `crm_credentials` for 10070 resolves to
   `franklinllcrm.sticky.io`, the **Franklin** account. No offers configured, so
   `new_order_card_on_file` uses the legacy `productId` payload, which is the shape already implemented.
3. Confirm the campaign's cascade profile is off, and that CCW's 6-attempt loop is off.
4. Establish whether Sticky's own decline salvage / initial-dunning is running on these campaigns
   (open question 4 in the reference doc — **still unanswered**, and running both double-charges).

**Phase 1 — build and shadow.** API, ledger, policy generator, selector constraints, measurement union.
Shadow mode: CCW calls the endpoint, we decide and log a full attempt plan, and fire **nothing**. Compare our
chosen MIDs against what actually happened to that order. Cheap, and it catches a wrong exclusion set before it
costs a real charge.

**Phase 2 — live behind a coverage dial.** The reprocess endpoint gets its own dial
(`client_config.settings.reprocess_coverage_pct`, seeded at 90). **This is the only way to get an unbiased
number** for what the cascade is worth — the observational contrasts in §2 will overstate it, and the
first-attempt engine already taught us that lesson at ~2×.

The holdout is **not** "no reprocess". CCW's own loop is being switched off, so a request we refused would
simply never be retried — that is not a control, it is a worse product for one customer in ten. The 10% holdout
instead **cascades on the same MID throughout**, which is what their loop did anyway. Treatment and control then
differ in exactly one respect: whether the MID moves. Recorded as `mid_strategy = 'control'`, and deliberately
kept out of the split-test denominator — a held-out request is not a drawn `same`, and pooling them would put
holdout traffic in the test's denominator and flatten the contrast the test exists to measure.

**Phase 3 — delayed retries.** Enable the scheduler and callbacks once CCW can receive them, starting with the
reason classes that measure as time-responsive rather than MID-responsive.

**Phase 4 — generalise.** The policy table, the dead-cell detector and the CVV-enforcement signal are all
client-agnostic. Sabre (10039, Konnektive) and Medibloom are the next candidates, and Konnektive's
`forceMerchantId` gives the same lever. Keep the API client-gated until then.

---

## 7a. What exists right now, and how to bring it up

Built 28 Aug. Nothing has been run and nothing is deployed.

| File | What it is |
|---|---|
| `sql/68_reprocess_schema.sql` | columns on `decision` + `mid_state`, widened CHECKs, 5 new tables |
| `sql/69_scope_first_attempt.sql` | scopes the 3 existing views to `attempt_no = 1`; adds `v_reprocess_performance`, `v_reprocess_split_test`, `v_reprocess_funnel` |
| `sql/70_seed_10070_reprocess_policy.sql` | reason map (3 processor families, ~55 strings), 10 policy rows, the Capital One × Paysafe dead cell, `reprocess_coverage_pct = 90` |
| `services/routingEngine/sticky.js` | the three CRM calls, response interpretation, systemNotes gateway-chain parser |
| `services/routingEngine/reprocessDb.js` | policy cache, classification, ledger writes |
| `services/routingEngine/reprocess.js` | the ladder, the split test, the holdout, `resumeScheduled` |
| `services/routingEngine/reprocessWorker.js` | leases and drains `reprocess_schedule`, posts callbacks |
| `index.js`, `db.js`, controller, routes | retry constraints in `route()`, 3 columns on the decision write, `POST /api/v1/payments/reprocess` |

| `sql/71_reprocess_dry_run.sql` | the three gates; ships **enabled but dry**, and excludes rehearsal rows from every reporting view |
| `services/routingEngine/reprocessReplay.js` | replays real declined orders through the whole engine without charging |

**Bring-up order** — 68 → 69 → deploy the scoped `signals.py` / `match_outcomes.py` / `getPositionMid` changes
→ 70 → 71 → deploy the API → start the worker. Running the API before 69 is the failure described in §5.3.

**Rehearsal, which is what unblocks this weekend.** `order_reprocess` is still not permissioned, so nothing can
charge — but everything upstream of the charge can be exercised against real declined orders now:

```bash
node services/routingEngine/reprocessReplay.js --client 10070 --limit 50
psql -c "SELECT * FROM routing_engine.v_reprocess_dry_run WHERE client_id = 10070"
```

A rehearsal cannot tell us whether a retry would have approved. It answers the things that can still be wrong
and are cheap to fix: whether the reason classifies at all (if `processor_family` resolves wrongly the whole
book falls to `unknown`), whether the selector finds a candidate or exhausts, whether the chosen MID is
genuinely different from the one that declined, and whether the acquirer switch survives or is relaxed away
every time. `reprocess_dry_run: false` makes the replay script refuse to run — a bulk replay against a live
cascade would be several hundred real charges on cards that already declined once.

Going live is then one flag: `reprocess_dry_run → false`, once `order_reprocess` is permissioned **and** CCW
has switched off both their retry loop and the campaign cascade.

### Run `refresh_mid_state` before going live — this one is not optional

`payments.cvv_required` was loaded on 28 Aug and **one of its cells is wrong in the dangerous direction**:

| MID | declared | measured CVV-decline share | attempts |
|---|---|---|---|
| **321** | `cvv_required = false` | **55.1%** | 1,544 |
| 252 | `cvv_required = true` | 1.2% | 1,553 |
| 308 | `cvv_required = true` | 1.0% | 1,611 |

252 and 308 are harmless — declared strict, measurably lenient, so we merely avoid them for no reason. **321 is
the opposite and it is the exact failure the flag exists to prevent**: declared CVV-optional while rejecting
more than half its traffic on CVV, at a 7.9% approval rate. Trusting that cell would make 321 a *preferred*
landing place for CVV declines.

`cvvSafety()` now lets a measured share overrule a declared-safe flag (never the reverse — avoiding a MID we
were told to avoid costs a few points; trusting a wrong "safe" costs the whole retry). **But the measurement
only exists once `sync_mid_attributes` has run.** Until then `cvv_decline_share` is NULL for every MID and 321
is treated as safe on the strength of the bad cell alone.

So: run the refresh job, confirm the warning it now logs for 321, and get the sheet corrected. Either fixes it;
neither has happened yet.

```bash
# env, per client
STICKY_API_URL_10070=https://<app_key>.sticky.io
STICKY_API_USR_10070=...   STICKY_API_PWD_10070=...
REPROCESS_CLIENTS=10070          # deliberately NOT inherited from ROUTING_ENGINE_CLIENTS

# the worker, on the Linux box
node -e "require('./services/routingEngine/reprocessWorker').run()"
```

**Still to build:** the generator that derives `decline_reason_map` / `decline_policy` from each client's own
history (the seed is hand-set from measurements), the `dead_cell` detector, `mid_state.acquiring_bin` /
`cvv_decline_share` population in `refresh_mid_state`, and the ETL change that attributes cascade attempts to
**net** approval rate. Until `cvv_decline_share` is populated, `require_no_cvv_mid` matches nothing and the CVV
class behaves like any other forced switch — the single largest effect in §2 is inert until that job runs.

---

## 7b. Live test, 28 Aug 2026

Run from a laptop against prod, `reprocess_dry_run: false`, `coverage: 100`, one order at a time. CCW's own
retry loop and the sticky campaign cascade were **still running** throughout, so these numbers are a floor:
some orders were being recovered by their loop in parallel.

**10 cascades attempted · 3 recovered · 3 refused as blacklisted · 0 unresolved charges · 0 errors.**

| order | class | declined on | chain | result |
|---|---|---|---|---|
| 4958933 | issuer_soft | 294 | 299 → **428** | approved |
| 4958951 | issuer_soft | 430 | **430** (same-MID arm) | approved |
| 4959148 | cvv | 321 | **415** | approved |
| 4958913 | cvv | 245 | 401 → 307 → 243 | declined |
| 4959016 | cvv | 245 | 407 → 393 → 391 | declined |
| 4959353 | issuer_soft | 415 | 415 (same arm) → 407 → 391 | declined |
| 4959363 | nsf | 430 | 345 → 428 → 427 | declined |
| 4958955 / 4958963 / 4958965 | fraud | — | — | not attempted (blacklisted) |

### What the live run settled

**`forceGatewayId` works on `order_reprocess`, and overrides `preserve_gateway = 1`.** Both were open
questions. Order 4958933 was pinned to its gateway and the forced MID still won; the CRM confirms
`gateway_id 428`, `is_cascaded 0`, no duplicate order.

**Blacklist refusal is right, and now has a number.** 6% of declines are blacklisted. They *do* recover today
at 27.6% — but recovered blacklisted orders run **16.41% chargebacks and 48.37% refunds**, against 0.07% and
8.75% for clean ones. Refusing them gives up ~$32K/month that arrives with a 234× chargeback rate attached.

**MID 299 was dead and nobody knew.** The cascade spent a rung on it, which is how we found it: 30 attempts
over three days, **0% approval, 100% "Attempted to perform an unauthorized operation"**. That string was
classified `issuer_soft`; it is a MID-level credential failure and is now `mid_broken`. It was in 21 active
rules taking live first-attempt traffic.

**Timing overhead is 3–5s per rung** (planned 35s → actual 38–40s), almost all of it the CRM round trip
(2.3–3.5s). So the ladder's real wall clock is ~100s, not 90s, and production co-located with the database
will not be materially faster.

**The same-MID arm is genuinely unsettled**, which is why it is a split rather than a decision: 4958951
recovered on the same MID at 15s, 4959353 did not. Two contradictory single observations is what "we cannot
answer this from history" looks like in practice.

### Three bugs the live run found that dry runs could not

1. **`writeDecision` was fire-and-forget**, so `reprocess_attempt`'s FK referenced a decision row that was not
   yet committed. Every write path now awaits and reports.
2. **The retry path never fell back to the default pool.** `poolFallback` defaults false and only the
   first-attempt endpoint passed true — so the path needing candidates most was the one denied them. Cascades
   were stopping at two MIDs while a 12-MID pool sat unused.
3. **Relaxing a soft constraint inside a thin rule.** Order 4958913's rungs 3 and 4 each had exactly ONE
   candidate, both CVV-required, so the ranking relaxed and rung 4 declined with *"A card security code has
   never been passed for this account"* — while 28 CVV-optional MIDs were live and unused. The walk now
   climbs before relaxing.

---

## 8. Risks

| Risk | Mitigation |
|---|---|
| **Cascade interference.** A forced `order_reprocess` still triggers the campaign cascade on top — one call becomes two gateway attempts and `order_view.gateway_id` records the cascade's MID, not ours. Confirmed by live test. | Campaign cascade profile off (agreed). Test whether `order_reprocess` accepts `cascade_override` as a second line of defence. Verify from `systemNotes`, never from `gateway_id`. |
| **Double charge on timeout.** No idempotency key on any Sticky endpoint. | Ledger row before every call; reconcile via `order_view.transaction_id`; never blind-retry `new_order_card_on_file`. |
| **Both engines running.** Their loop, Sticky dunning, or our cascade overlapping. | Phase 0 gate. Nothing fires until each is verified off. |
| **Card-network reattempt limits.** Card networks cap retries of a declined authorisation per card per window. Our 3–4 in-session attempts plus any scheduled ones need to stay inside them. | Confirm the current Visa/Mastercard limits before Phase 2; enforce a per-PAN attempt counter in the ledger. |
| **Measurement silently degrades.** §5.4. | Ledger union scoped in the same phase, not deferred. |
| **Alert rate on recovered orders.** 1.6–2× baseline on Pick-Up-Card and CVV classes. | Track alert rate per reason class as a first-class metric of the programme, not just approvals. |
| **Blast radius.** This sits in the checkout path of a live client. | Coverage dial, per-position enable, client-gated, and a kill switch in `client_config`. |

---

## 9. Open questions

1. **Sticky decline salvage / initial-dunning** — running on these campaigns or not? Two undocumented endpoints
   (`GET /initial-dunning/pending-orders`, `DELETE /initial-dunning/remove-orders`) suggest a product that could
   double up with us.
2. **Does `order_reprocess` accept `cascade_override` and `preserve_force_gateway`?** Untested. Both would help.
3. **Callback endpoint** — URL, auth scheme, and whether CCW will fulfil on a callback-driven approval the same
   way they fulfil an inline one.
4. **Re-pricing on the third rung.** `new_order_card_on_file` can salvage at a different price. Is a downsell
   allowed, and at what price points?
5. **P2 upsell tension.** P2's whole design is "reuse the MID that just approved P1". On a P2 decline the data
   says switch anyway (16.8% vs 11.4%) — a much weaker signal than P1's, and it argues against inheritance in a
   way worth thinking through rather than patching.
6. **`Activity limit exceeded`** — the one reason where the same MID measured better. Test deliberately.
7. **Does the dead-cell detector belong in the first-attempt engine now?** Capital One × Paysafe is losing
   real money on first attempts today, independently of anything in this document.

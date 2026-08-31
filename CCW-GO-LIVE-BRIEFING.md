# CCW (10070) — Decline Reprocessing Go-Live Briefing

**Prepared:** 2026-08-31
**For:** the go-live call with A.J. Elliott / Omid / Michael (CCW · 713Online)
**Client-facing doc already sent:** https://beastinsights.notion.site/decline-reprocessing-setup-guide
**Results workbook:** `~/Downloads/CCW-10070-reprocessing-results.xlsx`

Sources: Granola — CCW call 2026-08-25, internal BIN-routing reviews 08-24 / 08-25 / 08-27,
Mith's retry-timing analysis 08-28, approval-rate methodology 08-28. Live data — weekend
test 08-28 → 08-31 on `routing_engine` + `reporting.orders_enriched_10070`.

---

## 1. Where we are

| | Status |
|---|---|
| API | Deployed. `POST /api/v1/payments/sticky/reprocess` returns 401 on prod (routed, auth enforced) |
| Backend | `feature/routing-engine` @ `8e79ad4` |
| routing-engine | `challenger-arm` @ `16c68da` |
| Migrations 68–78 | Applied to prod (`db.beastinsights.com`). No staging exists |
| `reprocess_dry_run` | **true** — armed but not charging |
| `reprocess_coverage_pct` | 100 (see §6) |
| Retry loops / schedulers | All stopped |

**Nothing charges until `reprocess_dry_run` goes false.** That flip is the go-live moment;
it takes effect in 60s with no deploy or restart, and reverses the same way.

---

## 2. Results of the weekend test

103 orders recovered, **$9,785** gross, across 08-28 → 08-31.

### Net approval rate — Core (3047), 2026-08-30, CCW-local

| | first attempts | net approved | net AR |
|---|---|---|---|
| without our retries | 1,153 | 821 | **71.21%** |
| with our retries | 1,161 | 899 | **77.43%** |
| | | | **+6.23 pts** |

78 chains rescued that would otherwise have ended as declines.

Measured the way Shalin established with A.J. on 08-25: **net AR on first attempts**
(`final_order_status`), not organic. A.J. accepted that the true baseline is **net 65%**,
not the 41% organic figure. Organic on 08-30 was 48.41%.

Two adjustments make the counterfactual honest rather than flattering:
- **9 of 82** customers we approved had another approval anyway — excluded from the lift.
- **8 of our recoveries are themselves `attempt_sort = 1` rows** (`new_order_card_on_file`
  creates a genuinely new order). Without the engine those orders would not exist, so they
  come out of *both* sides. Subtracting them only from the numerator would have inflated the lift.

Aug 30 is not fully settled — chains starting late on the 30th can still move
`final_order_status`. Re-run in a few days before quoting the number externally.

### Overnight loop, 9 cycles, 321 orders

| decline class | tried | recovered | rate |
|---|---|---|---|
| issuer_soft | 216 | 44 | **20.4%** |
| cvv | 10 | 3 | **30.0%** |
| unknown | 33 | 1 | 3.0% |
| nsf | 53 | 1 | **1.9%** |
| mid_broken | 7 | 0 | 0% |
| card_dead | 2 | 0 | 0% |

**39 of 49 approvals came on rung 1** — the first MID switch. Rungs 2 and 3 together added
10 recoveries across 272 declines. The value is in switching, not in grinding.

`nsf` at 1.9% independently confirms Mith's 08-28 finding that insufficient-funds retries on
the same MID approve at ~0.1%. Those attempts are pure processing fees and MID damage —
exactly Jesus's argument for cutting six attempts to two or three.

---

## 2b. CCW's answers (Telegram, 2026-08-28, after the Notion doc)

Both questions in the doc are **answered**. Two of the three switch-offs are **not**.

### Q1 — checkout hold time → **~30 seconds, not 3 minutes**

> A.J.: "We can technically hold the customer indefinitely as long as their browser tab is
> still open; but realistically we wouldn't want to have the customer wait an unreasonable
> amount of time... I think your 3 minutes allocated is an eternity for a customer to wait
> for a card to process. It feels like **less than 30 seconds** would be ideal but open to
> discussion and what you've seen work elsewhere."

This is the single biggest change to the plan. Our ladder is `{15, 35, 40}` second delays and
runs 108–121s end to end. **It does not fit in 30 seconds.**

### What a shorter window actually costs — measured on all 103 recoveries

| winning rung | recoveries | median | p90 | max |
|---|---|---|---|---|
| 1 | **81** (79%) | 26s | 30s | 31s |
| 2 | 13 (13%) | 68s | 71s | 76s |
| 3 | 9 (9%) | 113s | 119s | 121s |

| cut off at | recoveries kept | share |
|---|---|---|
| 20s | 1 | 1% |
| **30s** | **71** | **69%** |
| **45s** | **81** | **79%** |
| 60s | 81 | 79% |
| 90s | 94 | 91% |
| 180s (today) | 103 | 100% |

Rung 1 does the work: **81 of 103**. Its median *execution* time is only **11s** (p90 15s) —
the rest is the 15-second delay we put in front of it.

### Recommendation — and proof it fits

**Internal decision already taken (today 12:02 IST): tell CCW 60 seconds, and finish all
three retries inside 30.** The measured budget says that works.

The 108–121s we run today is almost entirely *waiting*, not working:

| component | median | p90 |
|---|---|---|
| setup before first charge (order_view, guard, routing, writes) | 2.6s | 4.9s |
| each Sticky charge — `order_reprocess` | 2.9s | 3.8s |
| each Sticky charge — `new_order_card_on_file` | 2.7s | 3.8s |
| **configured delays `{15, 35, 40}`** | **90s** | **90s** |

Three attempts back-to-back cost only **8.7s median / 11.3s p90** of actual work.

**Re-time `delay_seconds` to `{5, 5}`:**

| | median | p90 |
|---|---|---|
| 3 full rungs, `{5,5}` delays | **21.3s** | **26.3s** |

Comfortably inside 30s at p90, with all three rungs intact — so we keep the full ladder and
**lose none of the 103 recoveries**, rather than the 69% a naive 30s cut-off would leave.

`{8,8}` would give median 27.3s but p90 32.3s — over budget. **`{5,5}` is the setting.**

**Caveat, from Mith's 08-28 analysis:** retries fired **within 5 seconds** of the decline
approve materially worse than the **5–60 second** band. `{5,5}` sits exactly on that boundary,
so it is defensible but should be watched — if rung-2/3 yield drops after the change, the delay
is the first thing to suspect, not the routing.

### Q2 — what triggers Initials → **the success response itself**

> A.J.: "It is the **success response from the final retry attempt** of the core (P1) order
> that triggers the initial (P2) transaction. If P1 approves in any capacity AND P2 was checked
> at time of checkout, then it will be attempted."

This is the answer we wanted: P2 fires off the response, so **our** `approved` response will
trigger it once they integrate. No webhook or status-watcher to accommodate.

### Also asked

> A.J.: "So will you basically be calling the NewOrderCardOnFile method from your end?"
> Bhautik: "It depends — either order_reprocess or NewOrderCardOnFile."

Correct answer, and worth keeping vague (see §7). It does mean they must handle a **changed
`order_id`**, which only happens on the `NewOrderCardOnFile` path.

### Still unanswered from CCW

- The **source of the six retries** (§4.1) — no reply.
- **Sticky cascade profile** and **decline salvage** — no confirmation either way.
- Only the retry loop was implicitly acknowledged, via the Q2 answer describing it.

---

## 2c. Decisions already taken internally (today, 2026-08-31 12:02 IST)

From the Neo1 sync. These are settled on our side and shape what we say on the call.

### Response contract
- **Tell CCW: 60-second timeout.** We answer inside it; all three retries complete within ~30s
  (§2b). If we do not answer in 60s, they proceed exactly as they do today.
- **Two-call shape under consideration:** first call does the cascade only (customer waiting);
  a second call places a new order if the cascade fails. Simpler alternative is one call that
  finishes inside the 60s window — decide before the call, since it changes their integration.

### P2 backfill for the recovered orders — action for today
The team reached the same conclusion the data showed (§3), from the other direction: **P2 never
fired because CCW never received a success response** — a success from outside their system
cannot trigger their trigger.

> "eni system ni bahar thi kyathi success gayu to trigger nai thay"

**Agreed action: send CCW the list of ~90 recovered orders so their P2 can be attempted.**
Note a direct P2 API call is not available — it has to be `new_order_card_on_file` with the
**Initials `product_id`**, and the **campaign changes** for Initials, so both fields must be
set correctly. This is real revenue currently sitting unclaimed (~$2,586, §3).

### Scheduled retries need a webhook from them
Night approval rates are structurally worse, so some declines should be retried in the morning
rather than in-session. That cannot use the response path.

- **Ask CCW for a webhook endpoint.** Then P2 has two triggers: our synchronous API response,
  and our webhook for scheduled retries.
- Real-time first; scheduled retries switch on once real-time is stable.
- Their own six retries already include long gaps (10–12 hours, some orders months apart), so
  scheduled retries match behaviour they already have.

### Cascade profile — ANSWERED FROM DATA: do NOT disable it globally

The assumption on the call was that Initials does not cascade. **It does — and P3/P4 cascade at
a higher rate than Core.** 30 days, `is_cascaded` on `orders_enriched_10070`:

| position | cascaded | recovered | **cascade AR** | revenue / 30d |
|---|---|---|---|---|
| P1 Core | 5,383 | 270 | **5.0%** | $26,635 |
| P2 Rebills | 389 | 10 | 2.6% | $270 |
| **P3 Upsell** | 206 | 105 | **51.0%** | **$3,045** |
| **P4 Upsell** | 107 | 56 | **52.3%** | **$2,380** |
| P2 Initials | 113 | 17 | 15.0% | $765 |

The cascade is **nearly useless where we are replacing it (P1, 5%) and highly effective where we
are not (P3/P4, ~51%)**. That is coherent rather than surprising: by P3/P4 the card has already
been approved twice, so moving a *proven* card to another MID works; at P1 the card is unproven.

**Switching the profile off globally would destroy ~$6,190 / 30 days (~$74K/yr) of upsell
recovery in order to remove a 5% cascade at Core that we are replacing anyway.**

Consistent across all six weeks in the window, so this is steady-state, not a recent change.

**Position for the call:**
1. **Ask whether the cascade profile can be scoped** — off for Core (3047) only, left on for
   Initials, Rebills, P3 and P4.
2. **If it cannot be scoped**, revisit the per-order `cascade: false` flag on the Core order —
   set aside earlier, but it becomes the only way to disable it exactly where we need to.
3. **Do not accept "switch it off entirely"** as the go-live condition. The Notion guide asks
   for the profile to be off; that instruction needs narrowing to Core before they act on it.

### Also agreed
- **Fewer retries is better** — more attempts drag the overall rate down. Three is the target;
  five or six is the ceiling, not the goal.
- **Internal reporting:** add **net** approval rate to the chart (currently organic only), and
  hourly granularity for a chosen day, to explain night-time dips.
- **Night vs day routing profiles** worth building — night performance is consistently worse.
- Shubham picks up the CCW changes; a replica client is being prepared for testing.

### One thing to confirm on the call
Their checkout appears to **complete before order processing finishes** — the six retries
include orders with gaps of months, which no checkout page waits for. If confirmed, the 30s
constraint is about perceived experience, not a hard technical limit, and the scheduled-retry
plan becomes much easier to sell.

---

## 3. The Initials gap — and what A.J.'s answer changes about it

**Correction to an earlier reading.** I first recorded this as "CCW never calls Initials on a
recovered order" — implying a defect in their funnel. A.J.'s Q2 answer says otherwise: P2 fires
from the **success response** of the core order. Our weekend recoveries were run **out of band**
— manual batches straight against Sticky, with CCW never called and never given a response.
So of course no P2 fired. The measurement is real; the original interpretation was wrong.

The corrected reading is more useful:

| Core approvals | got an Initials row within 24h |
|---|---|
| organic | **62.8%** (541 of 861) |
| our out-of-band recoveries | **0 of 92** |

This is a **clean measurement of what a recovery is worth with the funnel disconnected** — and
therefore of exactly what the integration buys:

- **$28.11** of Initials revenue per recovered Core
- **~$2,586** across 92 recoveries
- **30%** on top of each $95 Core

A.J. predicted this on 08-25, before it was measured:

> "we would probably be missing out an opportunity to upsell them on any of the post checkout
> upsells if it's taking two minutes."

**How to use it on the call:** not as a complaint, but as the proof that the wiring matters. A
recovery that does not feed P2 is worth 30% less. It also raises the stakes on the timing
question in §2b — the longer we hold, the more likely the customer has gone, and the more of
that 30% is at risk even once wired correctly.

### Two consequences to confirm with them

1. **Does P2 still fire when the P1 success arrives 15–40 seconds later than usual?** A.J. says
   the trigger is the response, and it reads as server-side ("if P1 approves in any capacity AND
   P2 was checked at time of checkout"). Worth confirming explicitly that nothing about it
   depends on the browser tab still being open.
2. **P2 must use the `gateway_id` we return**, and the `order_id` we return — which differs on
   the `NewOrderCardOnFile` path. Both are already in the doc; both need saying out loud.

### The same root cause: CCW re-charging customers we recovered

1 of 49 overnight recoveries. We approved order 4970990 at 16:21 CCW-local; CCW charged the
same card again at 16:43 on a different MID (order 4971417), which is not in our ledger.

Also an artifact of out-of-band testing — they had no way to know we had succeeded — and it
resolves the same way. Worth mentioning as evidence that **running both systems at once double
charges**, which is precisely why the three switch-offs in §4.3 are not negotiable.

## 4. THE CALL AGENDA — six items

Everything below expands in §4.1–4.7. This is the list to work through.

| # | Item | Our readiness |
|---|---|---|
| 1 | **How long will the customer wait on checkout?** | Ready — see note below |
| 2 | **Webhook endpoint**, so scheduled retries can still trigger Initials | Ready to spec, not built |
| 3 | **Call us on Initials failures too**, not just Core | Ready — P2 live-tested |
| 4 | **Turn cascading off entirely** — we retry P1/P2/P3/P4 | **NOT READY — see §4.8** |
| 5 | **Refund the 7**, and check the other ~90 for missing Initials | Ready — list below |
| 6 | **Carry our returned MID from Core into Initials** | Ready — shipped |
| 7 | **How long should we keep retrying a transaction on a schedule?** | Open question — data below |

---

### 1 · Checkout wait time — the premise needs correcting

The working assumption was "if under 60s we drop to 2 retries under 40s and schedule the rest."
**The measurement says we do not have to.** At `{5,5}` delays:

| | median | p90 |
|---|---|---|
| **3 full rungs** | **21.3s** | **26.3s** |

Three retries already fit under 40s, and under 30s at p90. Dropping to two would give up rungs
2 and 3 — 22 of 103 recoveries (21%) — for no gain.

**Only if they demand under ~20s** does the arithmetic force two rungs. So:

- **They say 60s** → 3 rungs, `{5,5}`. Nothing scheduled. Simplest outcome.
- **They say 30s** → 3 rungs, `{5,5}` still fits at p90 26.3s. Tight but real.
- **They say under 20s** → 2 rungs inline, rung 3 scheduled → needs the webhook (item 2).

Scheduling is a consequence of *their* number, not a design choice. Get the number first.

### 2 · Webhook for scheduled retries
Full detail in §4.6. The key condition: a webhook `approved` must be handled **exactly** like an
API `approved` — fulfil on the returned `order_id`, fire Initials with the returned
`gateway_id`. Otherwise scheduled recoveries silently reproduce the Initials gap (§3).

### 3 · Call us on Initials failures too
Same endpoint, same contract, no change on their side beyond calling it. **P2 is live-tested:**
23 requests, 10 recovered. The Notion guide currently scopes the first release to Core only and
says Initials/Upsells are "coming next" — that framing changes if we ask for P2 now.

### 4 · Turn cascading off entirely
This is the item we are not ready for. **See §4.8 before committing to it on the call.**

### 5 · The 7 refunds, and Initials on the other ~90
Refund list is in the workbook. The Initials check is done — results below in §4.9.

### 6 · Carry the MID from Core into Initials
Already shipped: `gateway_id` is returned on `approved` and null otherwise, and the Help Center
page documents forcing it on the Initials charge and going back to Payment Routing for the two
Upsells. Nothing to build; this is a confirmation, not a request.

### 7 · How long should a declined transaction stay retryable?

Once scheduled retries exist (item 2), a decline needs an expiry: hours, days, or weeks. This is
a **commercial** decision as much as a technical one — a charge landing a week after checkout is
a customer who may not recognise it, which is chargeback risk, not just recovered revenue.

Approval rate by gap from the customer's first attempt — Core, 60 days:

| gap from first decline | later attempts | approved | rate |
|---|---|---|---|
| under 1 min | 60,292 | 4,511 | **7.48%** |
| 1–10 min | 47,931 | 7,086 | 14.78% |
| 10–60 min | 5,549 | 877 | 15.80% |
| 1–6 h | 1,890 | 373 | 19.74% |
| 6–24 h | 1,936 | 314 | 16.22% |
| 1–3 days | 2,558 | 475 | 18.57% |
| 3–7 days | 2,353 | 450 | 19.12% |
| over 7 days | 8,869 | 1,981 | **22.34%** |

**Read this carefully — it does not say "wait longer, earn more."** The long buckets cannot be
separated from **customer-initiated returns**: someone coming back three days later with a
different card is a repeat purchase, not a recovery, and those naturally convert well. There is
no attempt-chain key in `orders_enriched`, so a system retry and a customer's second visit are
indistinguishable here.

What the data **does** support, robustly, because the volume is overwhelmingly machine-driven:

- **Sub-minute retries are the worst thing in the table — 7.48%**, against 14.78% at 1–10
  minutes. 60,292 attempts at the lowest yield on the book. This is the same effect Mith found
  on 08-28 (retries inside 5 seconds underperform the 5–60 second band), and it is the strongest
  argument for replacing their loop rather than merely shortening it.
- Waiting beyond ~10 minutes has **no proven incremental value** we can demonstrate. It may
  exist; this data cannot show it.

**Questions for them:**
1. **What is the latest we may charge a card after checkout** before they consider it a
   chargeback or customer-service risk? Their answer sets the horizon, not ours.
2. Do they have a view from the Second Swipe arrangement — that partner works older declines, so
   they may already know where the yield stops.
3. Their own retries include gaps of **months** (some of the six-attempt chains span 4–5 months).
   Is that deliberate, or a side effect of whatever is generating them (§4.1)?

**Our recommendation until we can measure it properly:** start scheduled retries at **24 hours
maximum**, and only for classes where in-session recovery is weak but the card is not dead
(`nsf` at 1.9% in-session is the obvious candidate — an empty account may not refill in 90
seconds, but it might by tomorrow). Then measure with a clean attempt chain before extending.

---

## 4. What we must ask them

### 4.1 Blocking — the six retries

On 08-25 A.J., Michael and Omid all said they have no such logic:
- `dicer` (their API layer) only performs the first attempt
- the Sticky cascade profile carries about half their MIDs, a limited set of decline codes,
  and is "only supposed to attempt once"
- no logic in Michael's checkout code beyond forcing the gateway we return

Yet the data shows six attempts on the same MID within a minute, each with its own order id.
They committed to auditing it and answering "by the time you wake up."

> **Has that been answered?** Everything downstream assumes they can switch the loop off, and
> nobody can switch off a mechanism they cannot locate. If it is still unidentified, this is
> the first item on the call — offer to work it with Sticky support jointly.

### 4.2 Answered — now decisions, not questions

Both doc questions came back on 08-28 (§2b). What is left is agreeing the consequences:

1. **Agree the response-time budget.** Propose **45s with a re-timed `{5, 10}` ladder** (~91%
   of recoveries) rather than defending 3 minutes. If they insist on 30s, `{5}` or `{5, 8}`
   keeps ~79%. Say plainly what each costs — this is a revenue trade, and it is theirs to make.
2. **Confirm P2 is server-side**, not dependent on the browser tab still being open (§3).
3. **Update the Notion doc and Help Center** once the number is agreed: the guide currently
   says 3 minutes and a 180s client timeout. Both change.

### 4.3 The three switch-offs — one of them now NARROWED

Any one left on means two systems retrying the same decline, which is a double charge on a real
card. But the second item is no longer "switch it off":

- [ ] **CCW's own retry loop** — off entirely. This is the block we replace.
- [ ] **Sticky campaign cascade profile — off for CORE (3047) ONLY.** Leave it running on
      Initials, Rebills, P3 and P4. Switching it off globally would cost ~$74K/yr (§2c). The
      Notion guide currently asks for it off wholesale and **must be corrected before they act
      on it.**
- [ ] **Sticky decline salvage / initial dunning** — off on these campaigns.

### 4.4 Integration commitments

- [ ] Will they read `order_id` from our response rather than assuming it is the one they sent?
      (For `new_order_card_on_file` recoveries it is a **different** order — that is the order
      that holds the charge.)
- [ ] Will they force our returned `gateway_id` on the **Initials** charge, and go back to
      Payment Routing for both Upsells?
- [ ] Any proxy/load balancer between us confirmed not to close the connection before **60s**?

### 4.5 Sequencing after us

Agreed 08-25: our declines → **Second Swipe** → **HelpGrid** as the last stop. Phase 1 hands
Second Swipe only "code declines" — customers who were never theirs — to keep chargeback risk
off their own book. Fraudulent decline reasons go nowhere at all.

> Open: at what point do we hand off, and does `not_attempted` go to Second Swipe or nowhere?
> Our position: `not_attempted` means retrying is the *wrong* thing to do — it should not be
> handed on.

### 4.6 Webhook endpoint — needed for scheduled retries

Some declines should not be retried in-session at all. Night-time approval rates are
structurally worse, so a decline at 03:00 is better retried at 09:00 than three times in
thirty seconds. That cannot use the synchronous response path.

**Ask them for a webhook endpoint we can POST the outcome to.** It gives P2 two triggers:

| trigger | when |
|---|---|
| our synchronous API response | in-session recovery (the launch scope) |
| **our webhook** | scheduled / next-morning retries (phase 2) |

Points worth making:
- It matches behaviour they **already have** — their own six retries include gaps of 10–12
  hours, and some orders months apart, so a delayed retry is not a new concept to their funnel.
- The payload is the same four fields as the synchronous response, so nothing new to parse.
- **They must treat a webhook `approved` exactly like an API `approved`** — fulfil against the
  returned `order_id` and fire P2 with the returned `gateway_id`. Otherwise scheduled recoveries
  reproduce the Initials gap in §3 by design.
- Not needed for launch. Ask now so it is scoped, switch it on once real-time is stable.

### 4.7 What we do NOT have — the ask list

Everything here is a dependency on CCW. Grouped by whether it blocks go-live.

#### Blocks go-live

| # | What we need | Status | Why it blocks |
|---|---|---|---|
| 1 | **Source of the six retries** | Asked 08-25, no reply | We cannot switch off a mechanism nobody has located. Two systems retrying = double charge. |
| 2 | **Cascade profile confirmed off — for Core only** | Not confirmed | See §2c: off globally costs ~$74K/yr at P3/P4. Needs narrowing, not switching off. |
| 3 | **Decline salvage / dunning confirmed off** | Not confirmed | Same double-charge risk. |
| 4 | **Agreed response budget** | They said ~30s, we propose 60s timeout | Determines the ladder re-timing (§2b). Cannot ship the delays without it. |
| 5 | **Proxy / load balancer confirmed not to close early** | Not confirmed | A 30s idle timeout upstream silently truncates every recovery. |

#### Needed, but not blocking

| # | What we need | Status | Why |
|---|---|---|---|
| 6 | **Can the cascade profile be scoped per campaign/product?** | Unknown | If not, we need the per-order `cascade:false` flag instead (#7). |
| 7 | **Does Sticky support a per-order "do not cascade" flag on NewOrder?** | Unknown | The only fallback if #6 is no. |
| 8 | **Webhook endpoint URL** | Not requested yet | Phase 2 scheduled retries (§4.6). |
| 9 | **Confirmation P2 trigger is server-side** | Implied, not stated | If it depends on the browser tab, a 30s wait loses P2 anyway (§3). |
| 10 | **Refund the 7 duplicate charges ($710)** | Not actioned | Ours to own, theirs to execute. Next-day now, so refunds not voids. |
| 11 | **P2 backfill for the ~90 recovered orders** | Not sent | ~$2,586 unclaimed (§2c). Either they trigger it, or we need the Initials `product_id` + campaign mapping to do it. |
| 12 | **Everflow pixel audit on attempts 3–6** | Promised 08-25, partially answered | A.J. confirmed dedupe by click id and payout only on approval; the specific audit of approved-on-attempt-3+ orders was never reported back. |
| 13 | **Notification when new MIDs are added** | Agreed verbally 08-25 | A.J. offered; make it a standing habit, not a favour. |

#### Things we should confirm we already have

| # | Item | Status |
|---|---|---|
| 14 | Sticky API permissions — `order_reprocess`, `new_order_card_on_file`, `order_view`, `order_find` | Confirmed in use |
| 15 | API credentials for the reprocess endpoint | They use the Payment Routing pair |
| 16 | Permission to **void/refund** on their account | **Unverified** — relevant to #10 |

### 4.8 Item 4 readiness — we are NOT ready to ask for a full cascade shutdown

Asking them to disable cascading entirely means we take over P1, P2, P3 and P4. The mechanics
are there for all four; the evidence is not.

| position | products | position mapped | policy | MID selection | live-tested |
|---|---|---|---|---|---|
| P1 Core | 3047 | yes | wildcard | 57 rulesets, 2 active | **513 requests, 93 recovered** |
| P2 Initials | 3034/3143/3145 | yes | wildcard | inherits P1's winning MID | **23 requests, 10 recovered** |
| P3 Upsell | 3116 | yes | wildcard | **1 active ruleset** | **never — 0 requests** |
| P4 Upsell | 3087 | yes | wildcard | **no rules of its own; reads P3's** | **never — 0 requests** |

Mechanically it works everywhere: `positionFor` resolves 3116→P3 and 3087→P4 from
`position_map`, the ten `decline_policy` rows are position-agnostic, and the salvage rung builds
its payload from the **declined order's own** campaign, product and price via `order_view` —
not from a Core-shaped assumption. Upsells were considered (there is an explicit carve-out for
products with no `shipping_id`).

**What is missing is evidence, and the stakes are the wrong way round.** Their cascade currently
recovers **51% at P3 and 52% at P4** — better than anything we have demonstrated anywhere (our
best is 25% on `issuer_soft` at Core). It works because by P3/P4 the card has already been
approved twice; a proven card moved to another MID converts. Switching that off in exchange for
an untested path risks ~$5,400/month of their best-converting recovery.

**Recommended position for the call:** ask for the full shutdown as the *agreed direction*, but
stage it —

1. **Now:** cascade off for **Core (3047)**. Proven, and the part that conflicts with us.
2. **Next:** Initials, once we have run a P2 batch at volume (23 requests is not enough).
3. **Then:** P3/P4, once we have numbers to compare against their 51%.

We can get those numbers **without CCW changing anything** — P3/P4 declines already exist and
can be reprocessed out of band exactly as the 103 Core orders were this weekend. A few hundred
attempts answers it before we ask them to switch off something that is working.

If the decision is to ask for the full shutdown today anyway, that is defensible — it is one
conversation instead of three — but **say plainly that P3/P4 is untested**, so a weak first week
is an expected risk rather than a surprise.

### 4.9 Item 5 — Initials never attempted on 91 of 93 recovered orders

| | orders |
|---|---|
| Core orders we recovered | 93 |
| **no Initials attempt at all** | **91** |
| Initials attempted but declined | **0** |
| Initials approved | 2 (both from a *new* checkout session, not ours) |

`initials_tried_but_declined = 0` is the important cell: it was **never attempted**, not
attempted and failed. Consistent with A.J.'s answer — no success response reached them, so
nothing fired.

**~$2,558** at their organic attach rate (62.8% × $44.76). Campaigns 339, 347, 359, 401, 430.

Full list in the workbook, sheet **"Initials not attempted (91)"**. The 7 refund cases are
flagged red there — **do not attempt Initials on those**; they are being refunded.

---

## 5. What we must tell them

1. **The Initials gap (§3)** — with the numbers, and the fix.
2. **They re-charge customers we recover (§3)** — one case observed.
3. **Results (§2)** on the net-65% benchmark they already accepted.
4. **We caused 7 duplicate charges ($710) on 08-30.** Found and fixed the same day; both guard
   layers now run before any charge. They need refunding — listed in the workbook. Better this
   comes from us today than from a customer complaint.
5. **We have aligned our CVV handling with their own cascade profile.** Their profile fires on
   four strings; ours now recognises all four, including `Issuer Declined MCC` (526 declines /
   30 days), which we had been treating as a MID problem rather than a CVV one. Their read of
   their own book, adopted. Migration 79, live.
6. **`nsf` is not worth retrying** — our 1.9% matches their ~0.1%. We will stop spending
   attempts there.

---

## 6. Internal decision to settle first

On 08-27 Jesus explicitly asked for a control group that does **not** move the gateway:

> "we should have this control group, maybe like 10%, whatever, where we don't change the mid
> where we can compare. If it's lower, perfect, we lower it or we replace it with a different
> challenger."

`reprocess_coverage_pct = 100` removes exactly that group. Bhautik's call is 100 at launch.

The trade is worth stating plainly: at 100 we maximise recovered revenue from day one, but we
lose the unbiased measure of what MID-switching is worth versus same-MID retry — and days
launched at 100 cannot be reconstructed later. Jesus is likely to ask for that comparison.
Worth aligning with him **before** the call, not after.

(The `same_mid_split_pct = 40` split on `issuer_soft` rung 1 is a different knob and still
gives a within-cascade same-MID comparison — but only on that one reason class and rung.)

Also outstanding from 08-27: Shalin asked for a written workflow tracker for the CCW project
(reprocessing, CVV/AVS removal, rebill/closure handling) rather than tracking it verbally.

---

## 7. Deliberate design notes (internal only — not for CCW)

- **Cascade vs new order is a black box on purpose.** `order_reprocess` hides the MID chain in
  Sticky's order notes; `new_order_card_on_file` exposes it. The mix is dynamic per order so
  the pattern cannot be inferred. Client-facing material therefore omits gateway ids, attempt
  counts, rungs and our decline taxonomy.
- **Reporting attribution (Shalin, 08-27):** cascade attempts would otherwise credit the
  *first* gateway with a success that a *later* gateway earned. Internal routing analysis
  attributes to the gateway that actually approved; the **client-facing definition does not
  change**. Two models: first-attempt (MID parameters) and reprocessing (MID parameters +
  decline reason).
- **Acquiring BIN** carries more predictive weight than the acquirer — one acquirer can issue
  several BINs with different performance. Already carried on `mid_state`.
- **Cap ladder:** 90 → 95 → 97 → 99, never 100; at 100 rebills start declining.

---

## 8. Go-live sequence

1. CCW confirms the three switch-offs (§4.3) and the source of the six retries (§4.1).
2. Agree the response-time budget (§2b), then re-time `decline_policy.delay_seconds` from
   `{15,35,40}` to fit it — `{5,10}` for a 45s window, `{5}` or `{5,8}` for 30s. This is a data
   change, no deploy:
   ```sql
   UPDATE routing_engine.decline_policy
      SET delay_seconds = '{5,10}', max_attempts = 2
    WHERE client_id = 10070 AND status = 'active'
      AND reason_class IN ('issuer_soft','cvv','unknown');
   ```
   `nsf` should go to `retry_allowed = false` at the same time (§2, 1.9%).
3. Update the Notion doc and Help Center to the agreed timeout (currently 3 min / 180s).
4. Verify ClickHouse env on the deployed API host — **without it the different-card guard
   silently degrades to card-only** and stops catching ~145 orders per 30 days, with no error
   in the logs:
   ```
   node -e 'const ch=require("./services/clickhouseClient");
     console.log("enabled:", ch.ENABLED);
     ch.query("SELECT count() n FROM reporting.orders_enriched_10070 WHERE date>=today()-1")
       .then(r=>console.log("guard layer 2 OK", JSON.stringify((r.rows||r)[0])))
       .catch(e=>console.error("GUARD LAYER 2 DEAD:", e.message));'
   ```
5. Dry-run smoke test against prod with a recently declined Core order — expect
   `status: "dry_run"` and three rungs on three different MIDs, none matching the declined MID.
   **Note: the already-paid guard does not execute in dry run**, so this proves routing, not
   duplicate protection.
6. Publish the Help Center API reference (10070 only) — promised "Monday" in the Notion doc.
7. Flip `reprocess_dry_run` to false.

**Rollback** — 60 seconds, no deploy:
```sql
UPDATE routing_engine.client_config
   SET settings = settings || '{"reprocess_dry_run": true}'::jsonb, updated_at = now()
 WHERE client_id = 10070;
```

---

## 9. Still open

- [ ] Refund the 7 duplicate charges ($710) — next-day now, so `order_void` will likely fail
- [ ] Help Center API reference for 10070
- [ ] Reconciliation sweeper for stuck `in_progress` rows (4 exist from 08-28/29; two approved
      but never marked complete, so `final_order_id` is missing — outcome is recorded in
      `reprocess_attempt`, and they are excluded from retry)
- [ ] ETL change to attribute cascade attempts to net AR internally (§7)
- [ ] Workflow tracker document for the CCW project (Shalin, 08-27)
- [ ] Extend reprocessing to Initials / Upsell positions — flagged as "coming next" in the
      Notion doc, so CCW should plan the same one-call pattern for those declines

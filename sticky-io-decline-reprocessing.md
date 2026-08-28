# sticky.io — Decline Reprocessing Reference

**Last updated:** 2026-08-26
**Scope:** Everything established while investigating how to reprocess declined orders on sticky.io, force a specific MID, and handle cascading.

---

## 1. Summary

| Goal | Answer |
|---|---|
| Retry a declined order | `order_reprocess` — same order record, no payload |
| Force a MID on that retry | `order_reprocess` accepts **`forceGatewayId`** (undocumented, confirmed by sticky support 26 Aug) |
| Retry at a different price | `new_order_card_on_file` — creates a **new** order |
| Cascade across multiple MIDs | **Not possible via API.** No endpoint accepts a MID array. Cascade = a routing profile configured in the CRM |
| Change next rebill date | `subscription_order_update` → `new_recurring_date` |
| Move rebills between MIDs in bulk | `PUT /api/v2/subscriptions` (Force Next Gateway by Order IDs) |

---

## 2. The three API surfaces

| API | Base | Docs |
|---|---|---|
| JSON API v1 | `https://{app_key}.sticky.io/api/v1/` | developer-prod.sticky.io |
| Restful API v2 | `https://{app_key}.sticky.io/api/v2/` | developer-v2.sticky.io |
| Services | `https://{sub_domain}.sticky.io/api/` | developer-services.sticky.io |
| Legacy | — | developer-legacy-prod.sticky.io → **404, removed** |

Auth: HTTP Basic. Rate limit 120 req/min on v1 methods, except `subscription_order_update` at **60/min**.

Doc sites are Postman-published collections. Full JSON can be pulled without auth:
```
https://documenter.gw.postman.com/api/collections/3906964/RWThUgGX   # v1  (65 requests)
https://documenter.gw.postman.com/api/collections/4811546/TWDamFCD   # v2  (264 requests)
https://documenter.gw.postman.com/api/collections/4811546/Tzz4SKjv   # Services (53)
```

---

## 3. Retry paths

### `order_reprocess` — the primary path
```json
POST /api/v1/order_reprocess
{ "order_id": "11279", "forceGatewayId": "417" }
```
- Retries a **declined** order on the same record — no duplicate order created
- No customer/card payload required
- `forceGatewayId` is **not in the published docs** but works (confirmed by sticky support, and by live test)
- Rejects non-declined orders → `374 – Cannot reprocess non-declined orders`
- Runs as an **MIT**, so the gateway profile must not require CVV or first-order retries will fail

### `new_order_card_on_file` — salvage / re-price
```json
POST /api/v1/new_order_card_on_file
{
  "previousOrderId": "4948284",
  "campaignId": "339",
  "shippingId": "3",
  "productId": "3047",
  "price": "95.00",
  "forceGatewayId": "417",
  "preserve_force_gateway": "0",
  "cascade_override": "1"
}
```
- Creates a **new order id** — must be linked back to the original for reporting
- Two payload shapes depending on the account (see §6)
- Only path that can retry at a **different price**
- Also accepts `promoCode`, `notes`, `ipAddress`, `temp_customer_id`

### `order_force_bill` — NOT a decline path
```
380 – "You cannot use force bill on a decline, please use reprocess function instead."
```
- It bills a subscription **immediately** — "customer wants to rebill now" (per sticky support)
- Accepts `forceGatewayId` + `preserve_force_gateway` + `product_id`
- If the main product is not recurring, `product_id` is **required** or it errors
- If all products share one recurring date, `order_id` alone bills the whole order

### v2 equivalent
```
POST /api/v2/subscriptions/{subscription_id}/bill_now      # empty body
PUT  /api/v2/subscriptions                                  # force next gateway first
```

---

## 4. Forcing a MID

| Field | Where | Effect |
|---|---|---|
| `forceGatewayId` | `order_reprocess`, `order_force_bill`, `new_order`, `new_order_card_on_file`, `new_order_with_prospect`, `new_upsell`, `authorize_payment`, `three_d_redirect` | The MID. Blank = campaign routing |
| `preserve_force_gateway` | same (except `order_reprocess` — untested) | `1` = pin this MID for all future rebills |
| `gateway_id` + `is_preserve` | `PUT /api/v2/subscriptions` | v2 equivalent, batched over `keys[]` of order ids |

Invalid MID → `702 – Invalid gateway Id` (no silent fallback).

**Bulk move (e.g. a MID closure):**
```json
PUT /api/v2/subscriptions
{ "view": "order", "gateway_id": 391, "is_preserve": 1, "keys": [29500, 29501] }
```
Response is only `{"status":"SUCCESS"}` — no per-order breakdown, so chunk batches to keep failures bisectable. Sticky fixed a bug here in Q3 2024 where it forced orders multiple times; test a small batch first.

**Finding the subscriptions to move is the hard part.** No endpoint filters by gateway:
- `order_find` — `gateway_id` is a response field, not a criterion
- `GET /api/v2/subscriptions/filter/{filter}` — only `order` and `email`
- `GET /api/v2/customers/{id}/subscriptions` — only is_recurring / is_archived / is_hold / recur_at / next_product_id / billing_model_id

Workaround: `order_find` with `criteria: {recurring:1, preserve_gateway:1}` + `return_type: "order_view"`, then filter client-side on `gateway_id`. Or pull the ids from the warehouse.

---

## 5. Cascading

**You cannot pass an array of MIDs.** Checked all 382 endpoints across the three collections — nothing accepts a gateway list.

Cascade = a **Payment Routing Profile** built in the CRM and attached to the campaign.

| Field / endpoint | Effect |
|---|---|
| `cascade_enabled` (authorize_payment) | `0/1` — allow the auth to follow a cascade profile |
| `cascade_override` (new_order*, three_d_redirect) | `1` = override the campaign's cascade for this call |
| `is_cascaded` (order_view / order_find) | Read-only: did this order cascade? |
| `payment_router_view` (v1) | Routing profile + its gateways |
| `GET /api/v2/payment_router/{id}/gateways` | `gateway_status` (Active / Inactive / Gateway Decline Limit Met / Maxed Out), `declines_count` |

Routing methods (UI only): Weighted (default), Round Robin, **User Defined Priority** (true cascade — fill top, then fall through). Plus reserve gateways, decline limits, MID group routing, preserve billing.

`forceGatewayId` and cascade are **mutually exclusive per transaction** — forcing a MID bypasses the router.

### The interference problem
On `order_reprocess` with `forceGatewayId`, the campaign's cascade profile still fires **on top** of the forced attempt:

```
Beast Insights - Reprocess attempt #1, previous gateway id was 245
sticky.io - Order attempted to process on gateway (417) and declined ...
sticky.io - Cascade profile (10) [INTELLIGENT] initiating attempt #1 on gateway (245)
sticky.io - Cascade profile (10) ... no eligible gateway was available to cascade to
```

Consequences:
1. One reprocess call = two gateway attempts
2. `order_view.gateway_id` ends up as the **cascade's** MID (245), not the forced one (417) — so verifying via `gateway_id` alone gives the wrong answer. The systemNotes are the reliable evidence. Confirmed live on order 4948284: `gateway_id` reads 245 while its notes show the attempt was made on 410 and the cascade moved it. `is_cascaded` is `"1"` there, so interference is at least detectable without parsing.
3. Extra declines land on MIDs you didn't choose, feeding their `declines_count`

`new_order_card_on_file` accepts `cascade_override: 1` to suppress this. `order_reprocess` — untested, worth trying.

**Note:** a campaign has *two* routing objects. Campaign 339 shows `payment_router_id: 42` while the notes reference `Cascade profile (10)`. Different IDs = different mechanisms. Disabling the cascade profile does not stop the router from picking the initial MID.

---

## 6. Account differences

| | Optimus (`optimus`) | Franklin (`franklinllcrm`) |
|---|---|---|
| Offers model | Yes — orders carry `offer` + `billing_model` objects | **No offers at all** (`GET /api/v2/offers` → `total: 0`) |
| `new_order_card_on_file` payload | `offers[]` with `offer_id`, `product_id`, `billing_model_id`, `quantity`, `price` | Legacy: `productId` + `campaignId` + `shippingId` (+ `price`) — **undocumented but works** |
| `offer_view` by campaign | Works | `909 – No match found` |

The legacy `productId` form is not in the current v1 spec and the Legacy API doc site is gone — but it's confirmed working on Franklin (order created 26 Aug).

---

## 7. Test log

| Test | Result |
|---|---|
| `order_force_bill` on declined initial (Optimus 503149) | `380 – cannot use force bill on a decline, use reprocess` |
| `order_reprocess` + `forceGatewayId` (test acct, gw 1) | `100 – reprocessed successfully` |
| `order_reprocess` + `forceGatewayId: 417` (Franklin 4948284) | Attempted on 417 per systemNotes; cascade then re-tried 245 |
| `offer_view` by campaign 339 (Franklin) | `909 – No match found` |
| `GET /api/v2/offers` (Franklin) | `total: 0` |
| `new_order_card_on_file` with `offers[]` (Franklin) | Blocked — no offer ids exist |
| `new_order_card_on_file` with `productId` (Franklin) | **Worked** |

---

## 8. Sticky support answers (26 Aug 2026)

- `new_order_card_on_file` creates a new order — it does **not** reprocess an existing one. `forceGatewayId` selects the gateway.
- **`order_reprocess` does accept force gateway.** Param is `forceGatewayId`. Absent from the docs — *"probably was just assumed"* — support said they'd flag it to the team.
- Next recurring date → **`subscription_order_update`**, not the v2 `recur_at` endpoint.
- Decline salvage and retry handling are "all automated"; the only other lever mentioned was `order_force_bill`.
- `order_force_bill` is for **immediate** billing, not scheduling.
- For an initial order with no prior approved transaction → **use `order_reprocess`**, but ensure **CVV is not required** on the first order, since the retry is an MIT.

---

## 8b. Response shapes — verified live, 28 Aug 2026

**The approved and declined bodies share almost no field names.** Reading the wrong one fails silently.

Approved `order_reprocess`:
```json
{ "error_found": "0", "response_code": "100", "transaction_id": "12485091577",
  "authorization_id": "054154", "response_message": "The order was reprocessed successfully" }
```

Declined:
```json
{ "response_code": 10800, "error_found": "1", "status": "ERROR", "order_id": "4959014",
  "gateway_id": "243", "gateway_alias": "JASO-quickprocessccw-Q7403",
  "orderTotalAmount": "85.00", "transId": "0", "authId": "",
  "decline_reason": "A card security code has never been passed for this account REFID:176009519",
  "error_message": "...same...", "cvv_response": "", "avs_response": "",
  "provider_name": "NETWORK MERCHANT INC", "gateway_code": "300", "decline_level": "0",
  "resp_code": "999999", "provider_response_code": "300" }
```

Consequences, each of which cost us a bug:

| you want | approved | declined |
|---|---|---|
| amount | **absent** — read the order | `orderTotalAmount` (NOT `order_total`) |
| transaction | `transaction_id` | `transId`, and it is `"0"` on a decline |
| gateway | **absent** — read the order | `gateway_id` |
| auth code | `authorization_id` | `authId` |
| CVV/AVS verdict | absent | `cvv_response` / `avs_response` |

`order_total` exists in neither. Reading it left `amount` null on every attempt and made recovered revenue
unmeasurable. And because the approved shape names no gateway, confirming that the forced MID actually took
the charge — rather than the campaign cascade moving it — requires a follow-up `order_view` on success.

**Decline strings carry a per-transaction `REFID`.** `Amount exceeds the maximum ticket allowed
REFID:175926161` — 2,327 occurrences over 60 days produced 2,327 distinct strings, so no exact-match table can
ever hit one. Strip a trailing ` REFID:<token>` before matching. One of them matters for safety rather than
optimisation: `Duplicate transaction REFID:N` means the charge may already have succeeded.

**Raw strings differ from the warehouse's normalised ones.** The API sends `CVV2 Mismatch`,
`CID Verification error`, `Declined for CVV failure`, `Incorrect CVV`, `Verification Data Failed` — five
distinct strings the warehouse all normalises to `Invalid CVV`. A map built from warehouse values catches 7%
of CVV declines.

---

## 9. Gotchas

**No idempotency key** on any of these endpoints. A timeout leaves you unsure whether the card was charged — and with `new_order_card_on_file` a blind retry creates a real duplicate *order*. Write the attempt row before the call; reconcile via `order_view.transaction_id`.

**Retry lineage: `child_id` IS structured — corrected 28 Aug.** `ancestor_id` and `parent_id` do both point back at the order itself, but `child_id` holds the salvage order created by `new_order_card_on_file`. Verified on order 4948284, whose `child_id` is `4949249`, matching its own note *"Card On File Billing Failed with declined order id 4949249"*. Still track lineage in our ledger — `child_id` holds one child, not a chain — but it is a real join key, not a string to parse.

**Our decision token lands in `employeeNotes`, not `systemNotes`.** Order 4948284 carries `bi:01a03ef8-133c-70ed-a535-bbdd92100b43` there. `systemNotes` is sticky's own log; anything we write via the API shows up as an employee note.

**`systemNotes` is append-only across the order's whole life, and gateway numbers appear in three different phrasings:**

```
Order attempted to process on gateway (410) and declined ... cascade gateway id (245) also declined
Cascade profile (10) [INTELLIGENT] initiating attempt #1 on gateway (245).
Declined by cascade gateway: (245) Issuer Declined MCC (1st attempt)
```

A pattern requiring `gateway(` matches only the first and silently misses the two that name the CASCADE's MID. And because the array spans every session — 4948284 has attempts from 12:47 and 13:57 in one list — parsing it whole after a reprocess returns `[410, 245, 417, 245]`, every gateway ever tried. Record the note count before the call and parse only what follows it; for 4948284 that isolates `[417, 245]`: 417 forced, 245 where the cascade actually landed it.

**`gateway_active` is unreliable.** On Franklin, dozens of MIDs have `CLOSED` in the alias but `gateway_active: "1"` — including GW370 (`CLOSED? 8/25 ...`, 47,681 MTD). Closure lives in a hand-typed alias string with inconsistent conventions (`CLOSED`, `CLOSED?`, `Keep off until`, `NO INITIALS`, `PAUSED`).

**Exclude gateway 420 `Dry Run Testing`** from any routing — active, cap 0, would fake-approve.

**Capacity is the real constraint.** As of 26 Aug, 19+ Franklin MIDs sit above 90% of monthly cap, including 245 at 94%. That's what "no eligible gateway was available to cascade to" means in practice.

**Pass `price` explicitly** on `new_order_card_on_file`. It's optional and falls back to the product default, which may not match the original order's custom price.

**`preserve_force_gateway: 1` is a commitment** — it pins the subscription to that MID for every future rebill. Default to `0` while cascading; promote deliberately.

**Don't retry NSF immediately.** "Insufficient funds" is a wait-and-retry-later class; a different MID doesn't create money, and the attempts feed `declines_count`.

**3DS (`response_code 101`)** needs the customer at a browser via `three_d_redirect` — not a server-side retry.

---

## 10. Useful supporting endpoints

| Endpoint | Use |
|---|---|
| `order_view` | Everything about a declined order from the order_id alone. Takes an **array** |
| `order_find` | `criteria: {declines:1, billing_cycle:{"=":"0"}}` for initials; `return_type: "order_view"` returns full bodies |
| `order_find_overdue` | `{"days": N}` — "orders that have been declined" |
| `order_find_updated` | Poll changes since a timestamp |
| `gateway_view` | `{"gateway_id":"all"}` — active flag, caps, MTD sales, descriptor, mid_group |
| `campaign_view` | Campaign's gateway list, `payment_router_id`, products, shipping |
| `validate_credentials` | Confirm API access before anything else |
| `GET /api/v2/orders/{id}/histories` | Order history notes — where cascade/gateway events are logged |
| `GET /api/v2/subscriptions/{id}/history` | Confirms whether a forced gateway took |
| `PUT /api/v2/subscriptions/{id}/recur_at` | v2 date change (support recommends v1 `subscription_order_update` instead) |
| `order_void` / `order_refund` | Undo a duplicate charge |

### `order_view` fields that matter
`decline_reason`, `decline_reason_details`, `response_code`, `gateway_id`, `gateway_descriptor`, `processor_id`, `retry_attempt`, `retry_date`, `is_cascaded`, `preserve_gateway`, `billing_cycle` (0 = initial), `is_recurring`, `cc_first_6`, `cc_last_4`, `cc_type`, `cc_expires`, `is_blacklisted`, `is_fraud`, `prepaid_match`, `on_hold`, `hold_date`, `recurring_date`, `systemNotes`, `employeeNotes`

### `order_find` criteria worth knowing
`declines`, `success`, `recurring`, `billing_cycle` (`{"=":"0"}` = initials only), `preserve_gateway` / `no_preserve_gateway`, `is_cascaded`, `hold_type` (`hard_decline` / `decline_salvage`), `date_type` (`create` / `hold` / `next_rebill` / `rma` / `return` / `chargeback`), `first_6_cc`, `last_4_cc`, `is_test`

---

## 11. Response codes

| Code | Meaning |
|---|---|
| `100` | Success |
| `101` | 3DS required → `three_d_redirect` |
| `200` | Invalid login credentials |
| `350` | Invalid order Id |
| `374` | Cannot reprocess non-declined orders |
| `380` | Order is not valid for force bill (incl. any decline) |
| `702` | Invalid gateway Id |
| `800` | Declined — read `decline_reason` |
| `909` | No match found |
| `10500/10501/10502` | Temp customer save failed / invalid / expired |
| `10600` | Invalid product Id |
| `11004` | `is_cascaded` invalid (must be 0 or 1) |

---

## 12. Curl reference

```bash
APP=franklinllcrm
U="$STICKY_API_USER"; P="$STICKY_API_PASS"
H="Content-Type: application/json"
B="https://$APP.sticky.io"

# Read a declined order
curl -sS -X POST "$B/api/v1/order_view" -u "$U:$P" -H "$H" \
  -d '{"order_id":["4948284"],"return_variants":1}'

# Retry on a chosen MID (same order record)
curl -sS -X POST "$B/api/v1/order_reprocess" -u "$U:$P" -H "$H" \
  -d '{"order_id":"4948284","forceGatewayId":"417"}'

# Salvage as a new order — Franklin (legacy productId form)
curl -sS -X POST "$B/api/v1/new_order_card_on_file" -u "$U:$P" -H "$H" \
  -d '{"previousOrderId":"4948284","campaignId":"339","shippingId":"3",
       "productId":"3047","price":"95.00","forceGatewayId":"417",
       "preserve_force_gateway":"0","cascade_override":"1"}'

# Salvage as a new order — Optimus (offers form)
curl -sS -X POST "https://optimus.sticky.io/api/v1/new_order_card_on_file" -u "$U:$P" -H "$H" \
  -d '{"previousOrderId":"503149","campaignId":"172","shippingId":"5",
       "offers":[{"offer_id":7,"product_id":143,"billing_model_id":2,"quantity":1,"price":"69.93"}],
       "forceGatewayId":"<MID>","preserve_force_gateway":"0"}'

# List MIDs with capacity
curl -sS -X POST "$B/api/v1/gateway_view" -u "$U:$P" -H "$H" \
  -d '{"gateway_id":"all"}' \
| jq '.data | to_entries[]
      | select(.value.gateway_active=="1")
      | select(.value.gateway_alias | test("CLOSED|Keep off|PAUSED"; "i") | not)
      | {id:.key, alias:.value.gateway_alias,
         used:((.value.monthly_sales|tonumber) / (.value.global_monthly_cap|tonumber) * 100 | floor)}'

# Find declined initials
curl -sS -X POST "$B/api/v1/order_find" -u "$U:$P" -H "$H" \
  -d '{"campaign_id":"all","start_date":"08/01/2026","end_date":"08/26/2026",
       "criteria":{"declines":1,"billing_cycle":{"=":"0"}},
       "search_type":"all","return_type":"order_view"}'

# Change next rebill date
curl -sS -X POST "$B/api/v1/subscription_order_update" -u "$U:$P" -H "$H" \
  -d '{"order_id":"4948284","product_id":3047,"new_recurring_date":"09/15/2026"}'

# Bulk move subscriptions to another MID
curl -sS -X PUT "$B/api/v2/subscriptions" -u "$U:$P" -H "$H" \
  -d '{"view":"order","gateway_id":391,"is_preserve":1,"keys":[29500,29501]}'
```

---

## 13. API permissions needed

Currently enabled on `beast_api`: `NewOrderCardOnFile`, `campaign_view`, `campaign_find_active`, `gateway_view`, `offer_view`, `order_find`, `order_find_updated`, `order_force_bill`, `order_view`, `product_index`, and v2 GETs for campaigns / customers / custom_fields / offers / orders / products.

**GRANTED 2026-08-28** — verified by probing each method against a nonexistent order id,
so no charge was possible:

| method | probe response | meaning |
|---|---|---|
| `order_reprocess` | `350` invalid order id | permitted |
| `new_order_card_on_file` | `10331` Missing previousOrderId | permitted |
| `subscription_order_update` | `906` Order Id must have a minimum value of (1) | permitted |

A permission failure looks different from all three — this is the method running and
rejecting the argument. Credentials come from `beast_insights_v2.crm_credentials`
(client 10070, crm_id 1) and resolve to `franklinllcrm.sticky.io`, confirming CCW is the
**Franklin** account: no offers configured, so `new_order_card_on_file` must use the
legacy `productId` payload.

Nice to have:
- `payment_router_view` — routing profile visibility
- `order_void` / `order_refund` — undo duplicates
- `validate_credentials`
- v2 `GET providers` — gateway profiles incl. CVV config

### Undocumented endpoints spotted in the permissions UI
- `GET /initial-dunning/pending-orders`
- `DELETE /initial-dunning/remove-orders`

Neither appears in any published collection. Likely the API surface for sticky's Smart Dunning / Recovery product — worth asking support about before building anything that could double up with it.

---

## 14. Open questions

1. Does `order_reprocess` accept `cascade_override`? (would stop the cascade interference)
2. Does `order_reprocess` accept `preserve_force_gateway`?
3. What are the `initial-dunning` endpoints?
4. Is sticky's built-in decline salvage running on these campaigns? If so, who owns retries — it or us? Running both double-charges.
5. Is the legacy `productId` form on `new_order_card_on_file` supported long-term, or deprecated?

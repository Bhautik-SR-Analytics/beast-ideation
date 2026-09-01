## What we need from CCW

### 1. One call on a P1 decline

Send us the `order_id` when a Core order declines. We work it and return the final outcome, so your own retry loop can be switched off. If both run, the same decline gets worked twice — that is a double charge on a real card.

### 2. Set a 60-second timeout

Nothing in between should close the connection earlier. If we have not answered in that window, carry on as you do today.

### 3. Act on the response

- `status` — `approved` means paid. `declined` or `not_attempted` should be treated as declined.
- `order_id` — key fulfilment on this, not on the id you sent. A recovery sometimes places a new order, and when the two differ this is the one that holds the charge.
- `gateway_id` — returned only on `approved`. Please force it on the P2 charge. For P3 and P4, call Payment Routing as you do today.

### 4. Trigger P2 from our success response

P2 currently fires from the P1 success response. It needs to fire from ours instead, through two paths:

- Real time — our API response
- Scheduled — our webhook, for attempts we delay on purpose

We will need a webhook endpoint from you for the second path. Same fields as the API response. A webhook `approved` must be treated exactly like an API `approved`, or scheduled recoveries lose their Initials and Upsells.

Two things to confirm:

- That the P2 trigger is server-side, not dependent on the browser tab staying open.
- How long after checkout you are comfortable having a card charged. That sets how far ahead we can schedule.

### 5. Switch off the other retry paths

- Your own retry loop
- The Sticky cascade profile, for Core (3047) only — please leave it on for Initials, Rebills, P3 and P4
- Sticky decline salvage and initial dunning on these campaigns

### 6. Extend to Initials later

Once P1 is stable, send us P2 declines the same way. Same request, same response, nothing to rework.

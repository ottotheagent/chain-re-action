# Duffel Flights API — Compiled Chain Config (blind compile from public docs)

Created: 2026-07-27
Last Updated: 2026-07-27
Compiled against: SPEC.md **Draft v0.3.1 (pre-implementation)**, SKILL.md (chain-compile, 2026-07-27), CONFORMANCE.md (applies to v0.3.1)

Evidence regime: **public documentation only** (https://duffel.com/docs + changelog.duffel.com).
No API access was available, so per the task rules the `observed` tag is unused:
- `documented` — stated on a fetched Duffel doc page (URL cited per claim)
- `documented(search-snippet)` — surfaced in a search-result summary attributed to a
  Duffel page but NOT verified on a fetched page; treated as weaker than `documented`
  and never load-bearing (routed to `known_unmatched` where it names a signal)
- `assumed` — my inference; each instance appears in §12 UNKNOWNs
- `spec-example` — would mark anything taken from SPEC.md's worked examples; **none used**

Scopes compiled (one document, per SKILL multi-scope guidance — shared elicitation log
and verdict-table core; per-scope handle graphs, actions, domain rows, gates,
walkthroughs; cross-chain notes in §11):

1. **duffel_instant_order** — search → offers → seat maps/services → instant-paid order,
   with the quote-then-confirm cancellation flow as its compensation sub-chain
2. **duffel_hold_order** — search → hold order (type=hold, no payment) → pay within
   `payment_required_by` window
3. **duffel_order_change** — order change request → change offers → pending change →
   confirm (in-place mutation of the order, `mutates`)

---

## 1. Chain summaries

**Scope 1 — Instant order.** Create an offer request (the search; offers returned inline
or listed), model/user selects one offer, refresh it via Get-single-offer (offers expire,
`expires_at`, "typically within 30 minutes"), optionally fetch seat maps and available
services (both may legitimately be empty), then create an order with `type: instant` and
a `payments` block — the commit. Duffel offers **no idempotency key**; docs explicitly
warn that retrying order creation "may result in a duplicate order or booking", and
provide the recovery machinery instead: `order.created` / `order.creation_failed`
webhooks carrying `offer_id` + `order_id`, a List-Orders filter by `offer_id`, and a
documented 502 `order_not_created` that guarantees no side effects. Compensation is the
documented two-step cancellation: create a pending cancellation (a refund quote with its
own `expires_at`), user reviews `refund_amount`/`refund_to` at a gate, then confirm —
confirming a stale/expired quote returns `order_cancellation_stale`.

**Scope 2 — Hold order.** Same search/selection prefix; create an order with
`type: hold` and **no** `payments` key. The order comes back with
`payment_status.payment_required_by` (hard deadline — airline releases the space after
it) and `price_guarantee_expires_at` (soft price lock — after it lapses the price can
drift; "the order will be available to complete until payment_required_by"). Paying is a
second, separate commit (`POST /air/payments`) whose amount must match the order's
*current* total (`price_changed` on mismatch). This scope is the mint-vs-commit rubric's
stress test: the analysis in §3 Q-H1 concludes the unpaid hold is a **commit**, not a
mint, despite self-expiry.

**Scope 3 — Order change.** Create an order change request naming slices to remove and
search criteria for slices to add; receive order change offers (each with `expires_at`
and `change_total_amount` / `penalty_total_amount`); select one; create a pending order
change (its own `expires_at`, and its price can drift — Get-single-order-change returns
the latest); confirm, providing payment if `change_total_amount > 0`. Confirmation
**updates the existing order in place** ("The booking with the airline will be updated
with the new slice you previously chose") — so the commit carries `mutates: order_id`,
content-based confirmation, `compensation: none`, and an unskippable `entry_gate` for
penalty/price consent.

---

## 2. Fit-test verdict (SKILL Step 0)

A chain is the right abstraction for all three scopes. Properties that fired:

- **Expiring handles at multiple hops**: offers (`expires_at`, ~30 min typical —
  https://duffel.com/docs/api/offers), cancellation quotes (`expires_at` +
  `order_cancellation_stale` — https://duffel.com/docs/guides/cancelling-an-order),
  change offers and pending changes (`expires_at` —
  https://duffel.com/docs/api/order-change-offers,
  https://duffel.com/docs/api/order-changes), hold-order payment/price windows
  (https://duffel.com/docs/guides/holding-orders-and-paying-later).
- **Unkeyed commits**: no idempotency mechanism anywhere in the request docs
  (https://duffel.com/docs/api/overview/making-requests — absence), plus an explicit
  do-not-retry warning for order creation
  (https://duffel.com/docs/api/overview/response-handling/order-and-booking-creation).
- **Async/ambiguous finality**: documented `202 Accepted` outcome on order creation,
  `order.created` webhook, "might take a couple of hours to show up" (same URL).
- **Empty is an answer**: zero offers is a valid search result (test scenario PVD→RAI —
  https://duffel.com/docs/api/overview/test-your-integration); seat maps "will not
  always be returned" (https://duffel.com/docs/api/seat-maps); change offers "will be
  empty if no offers are available"
  (https://duffel.com/docs/api/order-change-requests).
- **Compensation is a business action**: cancellation is quote-then-confirm with its own
  expiry, penalties, and refund destinations — a textbook `compensation.chain`.

None of the Step-0 refusal conditions hold. Q2/Q3 are answerable for every commit
(§3), so the compile is **not** BLOCKED.

---

## 3. Elicitation log (shared across scopes)

Source key: `[D]` = documented (URL follows), `[D~]` = documented(search-snippet),
`[A]` = assumed.

### Sources index (fetched pages)

- MR — https://duffel.com/docs/api/overview/making-requests
- ERR — https://duffel.com/docs/api/overview/errors
- ORQ — https://duffel.com/docs/api/offer-requests
- OFF — https://duffel.com/docs/api/offers
- ORD — https://duffel.com/docs/api/orders
- SM — https://duffel.com/docs/api/seat-maps
- PAY — https://duffel.com/docs/api/payments
- OC — https://duffel.com/docs/api/order-cancellations
- OCR — https://duffel.com/docs/api/order-change-requests
- OCO — https://duffel.com/docs/api/order-change-offers
- OCH — https://duffel.com/docs/api/order-changes
- HOLD — https://duffel.com/docs/guides/holding-orders-and-paying-later
- CXL — https://duffel.com/docs/guides/cancelling-an-order
- CHG — https://duffel.com/docs/guides/changing-an-order
- QS — https://duffel.com/docs/guides/quick-start
- GSF — https://duffel.com/docs/guides/getting-started-with-flights
- WH — https://duffel.com/docs/guides/receiving-webhooks
- AIC — https://duffel.com/docs/api/airline-initiated-changes
- RH-O — https://duffel.com/docs/api/overview/response-handling/order-and-booking-creation
- RH-R — https://duffel.com/docs/api/overview/response-handling/rate-limiting
- TEST — https://duffel.com/docs/api/overview/test-your-integration
- CL-ONC — https://changelog.duffel.com/announcements/we-ve-added-a-new-error-code-for-when-an-order-is-not-created

### 3.1 The create-order commit — Q1–Q5 answered explicitly (task requirement)

**Q1 — double-call behavior (same `selected_offers`, called twice)?**
`[D]` Duffel does **not** dedupe: "you should not retry the request as this may result
in a duplicate order or booking" (RH-O). So two successful calls with the same
`offer_id` can yield **two orders / two bookings**. The provider does not enforce
single-use of an offer — the single-use discipline must be client-side (see handle
`offer_id`, `single_use: true [A]`). Whether a second call *sometimes* fails (offer
consumed at the airline) is not documented → the safe compile treats the duplicate as
possible. Effect class: **commit**, no natural dedup.

**Q2 — mid-call death (request sent, response never read)?**
`[D]` An order may exist: the 202 path says "The request was accepted and we are working
to confirm its outcome … do not retry", and "The resource might take a couple of hours
to show up on the Duffel API" (RH-O). It costs money (payment `type: balance` is drawn
from the wallet at creation for instant orders — QS/PAY) and is user-visible (a PNR /
`booking_reference` at the airline — ORD). Q2b: creates a NEW durable object (`ord_…`),
does not mutate — no `mutates`.

**Q3 — confirmation probe when the response was lost?**
Three documented channels; the one that works with a lost response is (b):
- (a) **Get single order** — useless here: it needs `order_id`, which was in the lost
  response. `[D]` ORD.
- (b) **List orders filtered by `offer_id`** — works: `offer_id` is ours (from the
  request we sent) and is a documented List-Orders filter. `[D]` ORD. This is the
  probe. The correlation record (P3) persisted before dispatch stores `offer_id`.
- (c) **Webhooks** — `order.created` "will contain the `offer_id` and the `order_id`"
  `[D]` RH-O; the failure counterpart `order.creation_failed` exists (TEST: "202
  response followed by `order.creation_failed` webhook"). Webhooks are at-least-once
  with an `idempotency_key` for dedup `[D]` WH.
- Lag: "a couple of hours" worst case `[D]` RH-O → the probe must tolerate eventual
  consistency; `async.deadline` set to 6h, sweep beyond.

**Q4 — idempotency key availability + semantics?**
`[D]` (by omission + corroborating warning) **None.** The making-requests page
documents every request header and contains no idempotency mechanism (MR); RH-O's
do-not-retry warning confirms there is no safe replay. `idempotency.mode: none` →
per SPEC this **forces attempts=1** and requires a `confirmation` probe. (The only
"idempotency_key" in Duffel's docs is on *inbound webhook events*, WH — unrelated.)

**Q5 — natural idempotency?**
`[D]` No — Q1's documented duplicate-order risk is the direct disproof. Not assumed
away: `mode: none`, correlation record + cancellation shield mandatory (P3).

### 3.2 Remaining elicitation, per step

**create_offer_request** (POST /air/offer_requests)
- Q1: two calls → two `orq_…` resources, each a cached search result set with offers
  hanging off it. No cost, no inventory, not user-visible. `[D]` ORQ. Per SPEC §2.1
  ("Reads may return handles to cached result state … and remain reads"): **read**.
- Q6: mints `offer_request_id` (`orq_`), and the offers themselves (`off_…` ids)
  either inline (`return_offers: true`, default) or via List Offers. `[D]` ORQ.
- Q6 expiry: offer-request expiry is **undocumented** (ORQ: "no explicit statement");
  what expires is each **offer** (`expires_at`, "typically within 30 minutes", OFF).
  → `offer_request_id.staleness.detect: []` + UNKNOWN (§12); offers carry the detect.
- Q9: empty result is valid — test route PVD→RAI: "No offers will be returned" `[D]`
  TEST. Route: repair (user can change dates/cabin/airports).
- Q14: `supplier_timeout` 2–60s, default 20s; "recommends setting it lower than your
  HTTP request timeout" `[D]` ORQ.

**select_offer** (pseudo-step)
- Downstream results couple to the pick: seat maps are fetched **by offer_id** (SM),
  services belong to the offer (OFF), the order is created from the offer (ORD) →
  qualifies as `effect: select` per SPEC granularity rule (derived lineage exists).
- Q8b rematch key: provider offer ids are minted per search — fresh `orq_` mints fresh
  `off_` ids `[A]` (docs never claim id stability; the id embeds the request lineage).
  Intent-level identity: marketing carrier + flight numbers + departure datetimes per
  segment + cabin class + total_amount tolerance. `[A]` — flagged in §12.

**get_offer** (GET /air/offers/{id}?return_available_services=true)
- Q1: pure read; refreshes payload. "does not guarantee that the offer will be
  available at the time of booking"; "Each time you make this API call there's a
  possibility that some of the service information will have changed" `[D]` OFF.
- Purpose in-chain: docs prescribe it — "It is recommended to always get the offer
  before booking to get its latest state" `[D]` (TEST/offers guidance, surfaced via
  search over duffel.com; also GSF: "you should retrieve the offer to get the most up
  to date version"). Price drift is surfaced here (test scenario "Offer Price Change",
  LHR→STN, TEST) → precondition → consent gate.
- Q10: behavior of GET on an *expired* offer (404? payload with past `expires_at`?) is
  undocumented → UNKNOWN (§12); past `expires_at` in the payload is treated as a
  payload-predicate staleness signal `[A]`.

**get_seat_maps** (GET /air/seat_maps?offer_id=…)
- Q1: read. Q9: empty valid — "This is not available for all airlines or flights, so
  seat maps will not always be returned" `[D]` SM. Route: `ok` (proceed seatless,
  degraded, traced).
- Seats are services: `available_services` entries with `id`, `passenger_id`,
  `total_amount`; "the total amount charged will be the cost of the offer and any
  selected services"; "Each passenger can only select one seat per segment" `[D]` SM.
- Seat/service **picks are intent fields, not a select pseudo-step**: nothing
  downstream derives from the pick — the service ids ride the create-order request.
  Per SPEC §2.1 granularity rule and the §2.6 carve-out (gate/payload-validated opaque
  provider values entering intent are legitimately intent). Validation: chosen ids
  must appear in the presented seat-map/services payload.
- Q6: seat-map expiry undocumented `[D]` SM ("No expiry mentioned") — the handle's
  validity is tied to its offer; staleness surfaces on the offer instead.

**create_order** — Q1–Q5 in §3.1. Additional:
- Q10 documented signals: `offer_no_longer_available` ("Your integration should handle
  the error code offer_no_longer_available … while creating an order" — duffel.com
  offers/test docs via search + TEST scenario LHR→STN `[D]`); `insufficient_balance`
  (TEST, LGW→STN `[D]`); `order_not_created` (HTTP 502, "when we encounter an unknown
  error during order creation and the airline was unable to create your order", with
  the guarantee "no negative side effects have occurred during the process – for
  example an order being created or payment taken" `[D]` CL-ONC); HTTP 503 "no booking
  was created in the supplier systems" — safe to retry `[D]` RH-R; 202 Accepted
  (ambiguous success) `[D]` RH-O; `offer_expired` `[D~]` (search snippet attributed to
  Duffel help/guides; NOT verified on a fetched page → `known_unmatched`).
- Q11: mid-chain human decisions: final price/conditions consent before the commit
  (product rule; price can drift between list and get — TEST "Offer Price Change") →
  `entry_gate: book_consent`. Money consent → `audience: user`, never model.
- Q12 compensation: the two-step cancellation flow (OC/CXL), compiled as
  `compensation.chain` — see actions. Cost changes with time: `void_window_ends_at`
  on the order "indicates cancellation eligibility for refunds within this timeframe"
  `[D]` ORD; refund may go to `airline_credits` instead of money (TEST LTN→SYD,
  CXL) — the gate shows `refund_to`.
- Q13: replace/rebook ordering — not applicable inside this chain (no rebook step),
  stated for the planner in §11.
- Q14/Q15: "Airline and Accommodation APIs can occasionally be slow, taking up to
  120s. You must set a HTTP client timeout of at least 130s" `[D]` RH-O/RH-R.
  Finality: `order.created` webhook; up to hours `[D]` RH-O.

**create_order_cancellation** (POST /air/order_cancellations)
- Q1: creates a **pending refund quote**; "This is a review step—the order is not yet
  cancelled" `[D]` OC. Two calls → two quotes; "Only the most recent quote can be
  confirmed; stale quotes trigger an `order_cancellation_stale` error" `[D]` CXL.
  Quotes self-expire (`expires_at`: "The ISO 8601 datetime by which this cancellation
  must be confirmed" `[D]` OC; "if this timestamp is in the past you will not be able
  to confirm your cancellation and will need to re-request a cancellation quote" `[D]`
  CXL). Not billed, not airline-visible `[A]` (docs silent on side effects on the
  order — §12). → **mint**.
- Q9: payload carries `refund_amount`, `refund_to` ∈ {arc_bsp_cash, balance, card,
  voucher, awaiting_payment, airline_credits, original_form_of_payment} `[D]` OC.
- Q11: user must review the quote ("Make sure that you are happy with the prospective
  cancellation" `[D]` OC) → gate `cancel_quote_consent`.
- Precondition upstream: the order must list `"cancel"` in `available_actions`, else
  "manual requests are required through the dashboard" `[D]` CXL.

**confirm_order_cancellation** (POST /air/order_cancellations/{id}/actions/confirm)
- Q1: commit-class compensator; "The booking with the airline will be cancelled, and
  the `refund_amount` will be returned to the original payment method" `[D]` OC.
  Mutates the existing order (no new durable handle) → `mutates: order_id`.
- Q2: mid-call death → cancellation may have landed; Q3 probe: re-GET the order
  cancellation (`GET /air/order_cancellations/{id}` `[D]` OC) and the order; the
  content signal is the cancellation resource reaching its confirmed state / the order
  no longer cancellable — exact field names for the confirmed state are undocumented
  `[A]` §12. `airline_credits[].credit_code` "remains null until cancellation
  confirmation, then becomes available" `[D]` CXL — a documented content signal for
  the credits path.
- Q4: no key → unkeyed, attempts=1, P3 mechanics.
- Q10: `order_cancellation_stale` on stale/expired quote `[D]` CXL → re-quote.

**create_hold_order** (create_order with `type: hold`)
- **Q-H1 — the mint-vs-commit rubric stress test.** Facts: "Omit the `payments` key,
  as no payment takes place at the time of booking" `[D]` HOLD → never billed at
  creation. It self-expires: "If you don't pay for a flight before the time indicated
  in `payment_required_by`, the space will be released by the airline" `[D]` HOLD.
  But: it **holds real airline inventory** ("hold space on a flight", the airline
  releases "the space") and is **user-visible** (a real order with a
  `booking_reference` PNR, ORD). SPEC §2.1: "If a repeated artifact IS consequential
  to the user … it is a commit, however short-lived: mint status comes from
  inconsequence, not from self-expiry." Two hold calls → two PNRs holding seats,
  airline-visible, duplicate-booking territory (RH-O's warning does not distinguish
  hold from instant). → **commit** (unkeyed, attempts=1, full P3 mechanics), with the
  documented self-expiry recorded as a *compensation note* (lapse is a valid, free
  undo), not as a mint justification. Fees for holding: none documented (HOLD: "No
  information provided about whether holding incurs fees") → UNKNOWN §12.
- Expiry consequence: "the space will be released by the airline and the
  `awaiting_payment` status of the order will be set to `false`" `[D]` HOLD.
  ⚠ Surprising: `awaiting_payment: false` is also the *paid* state's value — the flag
  alone cannot distinguish paid from lapsed; probes must combine it with
  `payment_required_by` in the past and/or ticket `documents` presence (§12).
- Price guarantee: "Even as the price guarantee expires, the order will be available
  to complete until `payment_required_by`"; after expiry the system re-prices — the
  field "will either update to a new timestamp or become `null` if unguaranteed";
  "You can always get the most up-to-date price by retrieving the order" `[D]` HOLD.
- Eligibility: only offers with `payment_requirements.requires_instant_payment ==
  false` can be held `[D]` OFF (+ TEST scenario JFK→EWR returns such offers) →
  precondition on the hold path.

**pay_order** (POST /air/payments)
- Q1: two successful pays of one order → second returns invalid state `already_paid`:
  "The order you're paying for has already been paid for." `[D]` PAY. So a duplicate
  *successful* charge is prevented provider-side by order state — but this is a
  same-order guard, not an idempotency key; an ambiguous first attempt still requires
  probe-not-retry (a blind second dispatch that returns `already_paid` proves landed
  but burns nothing — still routed through `reconcile`, never a retry).
- Q2: mid-call death → payment may have been taken; user-visible (tickets issue on
  payment `[A]` — see §12); costs money. → commit, unkeyed (Q4: no key, MR),
  attempts=1.
- Q3 probe: `get_order` — content signal: `payment_status.awaiting_payment == false`
  AND ticket `documents` present (the AND is required because of the lapsed-hold flag
  collision above) `[D]`+`[A]` ORD/HOLD.
- Q10 documented signals `[D]` PAY: validation `payment_amount_does_not_match_order_amount`,
  `payment_currency_does_not_match_order_currency`, `price_changed` ("If the price for
  an order has changed from the time of booking and you pass in the old price…");
  invalid states `already_paid`, `already_cancelled`, `past_payment_required_by_date`
  ("The order's `payment_required_by` date has elapsed."), `schedule_changed` ("You
  can't pay for this order because it has been changed in some way on the airline's
  side."); `order_type_not_eligible_for_payment` (not a hold order — config bug).
- Payment types: `balance`, `arc_bsp_cash`, `card`, `airline_credit` `[D]` PAY. This
  compile pins `type: balance` as a `request_invariant`; the card/3DS path is a
  different chain shape (out of scope, §12).

**create_change_request** (POST /air/order_change_requests)
- Q1: read-class search over change space (two calls → two `ocr_` result sets;
  inconsequential) `[A]`-leaning-`[D]` (OCR documents it as a search returning offers;
  no side effect on the order is documented). Requires `"change"` in the order's
  `available_actions` `[D]` CHG.
- Output: `ocr_` id + inline `order_change_offers` (each `oco_` with `expires_at`)
  `[D]` OCR; "will be empty if no offers are available" `[D]` OCR → empty valid.
- Q9 route: repair (user changes the new-slice search criteria: dates, cabin).

**select_change_offer** (pseudo-step) — change offers expire and the pending change +
confirmed mutation derive from the pick → `effect: select`. Rematch key: new-slice
content (carrier + flight numbers + departure datetimes) + `change_total_amount`
tolerance `[A]` §12.

**create_pending_order_change** (POST /air/order_changes)
- Q1: mint — creates a pending change from `selected_order_change_offer` `[D]` OCH;
  quote-like, has its own `expires_at` ("The ISO 8601 datetime by which this change
  must be confirmed" `[D]` OCH); side effects on the order undocumented → `[A]` mint
  (§12). Its price can drift: "The price of a pending change order can change over
  time. You should let your customers review the final price before confirming …
  use Get a single order change endpoint to obtain the latest price" `[D~]`/`[D]` OCH.

**confirm_order_change** (POST /air/order_changes/{id}/actions/confirm)
- Q1/Q2b: commit that **mutates the order in place**: "The booking with the airline
  will be updated with the new slice you previously chosen, and the
  `change_total_amount` will be charged to your specific payment type" `[D]` OCH →
  `mutates: order_id`; existence proves nothing; confirmation must be content-based.
- Payment: "If change_total_amount is greater than zero, you will need to provide
  payment details when confirming the change." Negative → "returned to the `refund_to`
  method (e.g. your Duffel balance)" `[D]` OCH.
- Q3 probe: `get_order` — content: the order's slices now match the change offer's
  `added` slices (slice ids may be new — AIC docs document new ids after changes,
  AIC); plus `get_order_change` status if one exists (statuses undocumented → §12).
- Q4: no key → unkeyed, attempts=1, P3 mechanics.
- Q10: stale/expired-pending-change error code is **undocumented** (no
  `order_change_stale` analog found) → falls to default rows; §12.
- Q12: compensation **none** — no documented undo of a confirmed change (the only
  recourse is another change or a cancellation, both new business actions). Per SPEC
  rule, `mutates` + `compensation: none` ⇒ the pre-commit safety mechanism is named in
  `ordering_note`: the `entry_gate change_confirm_consent`.

**Cross-cutting**
- Errors envelope: `errors[]` with `type`, `code`, `title`, `message`,
  `documentation_url`, `source{field,pointer}` (validation only), `meta.request_id`;
  types: `authentication_error, airline_error, invalid_state_error, rate_limit_error,
  validation_error, invalid_request_error, api_error`; 422 carries validation /
  invalid-state / airline errors; 429 `rate_limit_exceeded`; 500–504 `api_error` `[D]`
  ERR.
- Rate limiting: 429 + `ratelimit-limit` / `ratelimit-remaining` / `ratelimit-reset`
  headers; "retry your request after the time specified in the `ratelimit-reset`
  header" `[D]` RH-R. Whether a 429 on order-creation is rejected **before** any
  supplier dispatch is NOT documented → no `rejected_before_execution` declaration for
  429; on unkeyed commits it classifies `reconcile` (SPEC §3 row 5 default). §12.
- Webhooks: `order.created`, `order.updated`, `order.airline_initiated_change_detected`,
  `ping.triggered` `[D]` WH; `order.creation_failed` `[D]` TEST. At-least-once,
  dedupe by event `idempotency_key`, HMAC-SHA256 signatures `[D]` WH.
- Airline-initiated changes (external mutation risk): AIC resources (`aic_` prefix)
  with `action_taken ∈ {accepted, cancelled, changed}`, `added`/`removed` slices where
  "slices and their segments may have a new ID"; endpoints to accept/update `[D]` AIC;
  detection webhook `order.airline_initiated_change_detected` `[D]` WH. In-chain
  manifestation: `schedule_changed` invalid state on pay `[D]` PAY. See §11.

---

## 4–7. Per-scope configs

## Scope 1 — `duffel_instant_order`

### 4.1 Handle graph

```yaml
handles:
  offer_request_id:
    minted_by: create_offer_request
    derived_from: []
    staleness: {detect: [], ttl_hint: unknown, refresh: create_offer_request}
    # UNKNOWN: offer-request expiry undocumented (ORQ). Empty detect → consumers'
    # stale cases fall to default rows (fail closed). Offers carry the real expiry.
  offer_id:
    minted_by: create_offer_request        # offers returned inline / via List Offers (ORQ)
    derived_from: [offer_request_id]
    single_use: true                        # ASSUMED, client-side discipline: Duffel does
                                            # NOT enforce single-use (duplicate order
                                            # documented possible, RH-O). After any commit
                                            # attempt with unknown outcome, never
                                            # re-present this offer (I4).
    staleness:
      detect: [code:offer_no_longer_available,           # documented (TEST, offers docs)
               payload:"expires_at < now on get_offer"]  # assumed payload predicate
      ttl_hint: ~30m                        # "typically within 30 minutes" (OFF)
      refresh: create_offer_request         # expired offers cannot be revived; re-search
  selected_offer:                           # selection pseudo-handle (I3)
    minted_by: select_offer
    derived_from: [offer_id]
    rematch:
      key: [marketing_carrier, flight_numbers, segment_departure_datetimes,
            cabin_class, total_amount_tolerance]          # ASSUMED (§12) — off_ ids are
      on_ambiguous: model                                 # not stable across re-search
    staleness: {detect: [], ttl_hint: unknown, refresh: select_offer}
  seat_map_id:
    minted_by: get_seat_maps
    derived_from: [offer_id, selected_offer]
    staleness: {detect: [], ttl_hint: unknown, refresh: get_seat_maps}
    # No expiry documented (SM); validity rides the offer — offer staleness sweeps
    # this away via I1.
  order_id:                                 # durable output of the commit
    minted_by: create_order
    derived_from: [offer_id]                # via the consumed offer
    staleness: {detect: [], ttl_hint: n/a, refresh: n/a}
  cancellation_quote_id:                    # pending cancellation (compensation sub-chain)
    minted_by: create_order_cancellation
    derived_from: [order_id]
    single_use: true                        # spent by confirm; only the MOST RECENT quote
                                            # is confirmable (order_cancellation_stale, CXL)
    staleness:
      detect: [code:order_cancellation_stale]             # documented (CXL)
      ttl_hint: unknown                     # expires_at returned per-quote (OC); duration
                                            # not documented as a constant
      refresh: create_order_cancellation
```

```
create_offer_request ──> offer_request_id
                          └─> offer_id ──> selected_offer (pseudo, I3)
                                │               │
                                │               └─> seat_map_id (also <- offer_id)
                                └─────────────> order_id  (commit; consumes offer_id)
                                                  └─> cancellation_quote_id (comp. sub-chain)
```

### 4.2 Actions

```yaml
actions:
  - id: create_offer_request
    description: Flight search; distributes to airlines, returns offers.
    effect: read            # double-call test: another cached result set; no consequence.
                            # A handle to cached results does not make a mint (SPEC §2.1).
    input:
      intent:
        slices:      {type: object, required: true}   # origin/destination/departure_date
        passengers:  {type: object, required: true}   # docs recommend AGE not type, to
                                                      # avoid offer_no_longer_available
                                                      # mismatches at order time (search-
                                                      # verified Duffel guidance)
        cabin_class: {type: enum[economy, premium_economy, business, first], required: false}
    output:
      payload: offers[]                      # inline with return_offers: true (default)
      handles: [offer_request_id, offer_id]
      empty: {valid: true, detect: "offers.length == 0",
              route: repair(to: create_offer_request,
                            fields: [slices, cabin_class, passengers])}
    request_invariants:
      return_offers: true
      supplier_timeout: 20000               # ms; must stay below our HTTP timeout (ORQ)
    idempotency: {mode: natural}
    timeout: 60s                            # supplier_timeout 20s + fan-out headroom
    latency_hint: 20s

  - id: select_offer
    description: Model/user picks one offer from the results.
    effect: select
    input: {handles: [offer_id]}
    output: {handles: [selected_offer]}

  - id: get_offer
    description: Refresh the selected offer (latest price, available services).
    effect: read
    input: {handles: [offer_id, selected_offer]}
    output: {payload: {offer: object, total_amount: money, expires_at: datetime,
                       available_services: services[],
                       passenger_identity_documents_required: bool,
                       payment_requirements: object}}
    preconditions:
      - when: "total_amount != selection_time_amount"
        verdict: gate(price_review)          # terms changed pre-commit (P8-analog)
        reason: "offer price drifted between list and get (documented test scenario)"
      - when: "passenger_identity_documents_required == true && intent lacks passports"
        verdict: repair(to: create_order, fields: [passengers])
        reason: "documented: must provide a passport document for every passenger (OFF)"
    request_invariants: {return_available_services: true}
    idempotency: {mode: natural}
    timeout: 30s
    latency_hint: 3s

  - id: get_seat_maps
    description: Seat maps for the selected offer; seats are priced services.
    effect: read
    input: {handles: [offer_id, selected_offer]}
    output:
      payload: seat_maps[]                   # seat picks become intent (service ids),
      handles: [seat_map_id]                 # validated against this payload — NOT a
      empty: {valid: true,                   # select pseudo-step (no derived lineage)
              detect: "seat_maps.length == 0",
              route: ok}                     # documented: not all airlines/flights (SM);
                                             # proceed seatless, degraded, traced
    idempotency: {mode: natural}
    timeout: 30s
    latency_hint: 3s

  - id: create_order                         # THE COMMIT
    description: Book + pay instantly from the selected offer.
    effect: commit
    input:
      intent:
        passengers: {type: object, required: true}    # names, DOBs, docs if required
        services:   {type: object, required: false}   # seat/bag service ids chosen from
                                                      # seat-map/offer payloads (validated)
        payment:    {type: object, required: true}    # {type: balance, amount, currency}
      handles: [offer_id]
    output:
      payload: {order: object, booking_reference: string, documents: tickets[]}
      handles: [order_id]
    request_invariants:
      type: instant
      payments[0].type: balance              # card/3DS is a different chain (out of scope)
    idempotency: {mode: none}                # Q4: no key exists → attempts=1 FORCED
    entry_gate: book_consent                 # unskippable final-price consent; re-fires
                                             # on every arrival incl. post-rewind replays
    confirmation:
      probe: list_orders_by_offer            # GET /air/orders?offer_id=… — works with a
                                             # lost response because offer_id is ours (Q3)
      signal: "orders[] contains an order whose offer_id matches the correlation record"
      async: {channel: webhook, deadline: 6h}    # order.created carries offer_id+order_id
                                                 # (RH-O); 'couple of hours' lag documented
      sweep: {interval: 30m, escalate_after: 24h}
    compensation:
      chain: [create_order_cancellation, confirm_order_cancellation]
      window: "refund per airline policy; order.void_window_ends_at bounds the free-void
               period (ORD); refund_to may be airline_credits rather than money (CXL)"
      ordering_note: "if a planner replaces this booking, land the replacement BEFORE
                      running this compensation (never leave the user with nothing)"
    timeout: 150s                            # documented: supplier calls up to 120s;
                                             # client timeout MUST be ≥130s (RH-O)
    latency_hint: 30s
    # P3 mechanics (unkeyed): correlation record {run_id, offer_id, intent snapshot,
    # status=PENDING} persisted to durable storage BEFORE dispatch; cancellation shield
    # on the in-flight attempt. The record's offer_id is what probe + webhook match.

  - id: list_orders_by_offer                 # probe (read-only)
    description: Confirmation probe for create_order.
    effect: read
    input: {handles: [offer_id]}
    output: {payload: orders[],
             empty: {valid: true, detect: "orders.length == 0", route: ok}}
             # empty = not-landed-yet; the reconcile procedure owns the loop, not
             # this action's route
    idempotency: {mode: natural}
    timeout: 30s

  - id: get_order
    description: Read one order (also the probe body for cancellation/pay/change).
    effect: read
    input: {handles: [order_id]}
    output: {payload: {order: object, available_actions: string[],
                       payment_status: object, documents: tickets[],
                       void_window_ends_at: datetime}}
    idempotency: {mode: natural}
    timeout: 30s

  # ---- compensation sub-chain (runs under state=compensating) ----

  - id: create_order_cancellation
    description: Mint a pending cancellation quote (refund_amount/refund_to/expires_at).
    effect: mint                             # review step; order not yet cancelled (OC)
    input: {handles: [order_id]}
    output:
      payload: {refund_amount: money, refund_currency: string,
                refund_to: enum[arc_bsp_cash, balance, card, voucher, awaiting_payment,
                                airline_credits, original_form_of_payment],
                expires_at: datetime, airline_credits: object[]}
      handles: [cancellation_quote_id]
    preconditions:
      - when: "quote returned"               # always gate the user on the quote
        verdict: gate(cancel_quote_consent)
        reason: "documented review step: 'Make sure that you are happy with the
                 prospective cancellation' (OC)"
    idempotency: {mode: natural}             # re-quoting supersedes; only most recent
                                             # quote is confirmable (CXL)
    timeout: 60s
    latency_hint: 10s

  - id: confirm_order_cancellation
    description: Execute the cancellation; airline booking cancelled, refund issued.
    effect: compensate                       # commit-class
    mutates: order_id                        # no new durable handle; the order flips state
    input: {handles: [cancellation_quote_id, order_id]}
    output: {payload: {cancellation: object}}
    idempotency: {mode: none}                # no key → attempts=1, P3 mechanics
    confirmation:
      probe: get_order_cancellation          # GET /air/order_cancellations/{id} (OC)
      signal: "cancellation resource shows confirmed state (exact field UNDOCUMENTED,
               §12); corroborate via get_order: 'cancel' gone from available_actions;
               airline_credits[].credit_code becomes non-null on confirmation (CXL)"
      async: {channel: poll, deadline: 6h}
      sweep: {interval: 1h, escalate_after: 24h}
    compensation: {action: none,
                   ordering_note: "cancellation is itself the undo; consent sits at
                                   cancel_quote_consent BEFORE this commit"}
    timeout: 150s

  - id: get_order_cancellation               # probe (read-only)
    effect: read
    input: {handles: [cancellation_quote_id]}
    output: {payload: cancellation}
    idempotency: {mode: natural}
    timeout: 30s
```

### 4.3 Domain verdict rows (Scope 1; evaluated before the generic skeleton §3 rows)

| # | Signal | Step scope | Verdict | Source |
|---|---|---|---|---|
| A1 | `code:offer_no_longer_available` (422) | create_order, get_offer | `rewind(to: create_offer_request)` — offer dead; re-search; selection re-established via rematch | documented (TEST LHR→STN; Duffel offers guidance) |
| A2 | payload: `expires_at < now` on get_offer | get_offer | `rewind(to: create_offer_request)` | assumed predicate (expiry documented OFF; GET-on-expired behavior undocumented) |
| A3 | precondition: `total_amount` drifted | get_offer | `gate(price_review)` | documented drift scenario (TEST) |
| A4 | `code:insufficient_balance` (422) | create_order | `dead_end(reason: insufficient_balance, permanent: false, retry_after_hint: "after wallet top-up")` — operator problem, not user intent | documented (TEST LGW→STN) |
| A5 | `http:502 code:order_not_created` | create_order | `dead_end(reason: order_not_created, permanent: false, retry_after_hint: minutes)` — documented guarantee of NO side effects (no order, no payment), so this is terminal-for-this-run but safely re-plannable; conservatively NOT declared `rejected_before_execution` → no fresh dispatch inside the run (see §14 friction) | documented (CL-ONC) |
| A6 | `http:503` | create_order | `retry` — the ONE signal declared `rejected_before_execution: true` on this unkeyed commit: docs guarantee "no booking was created in the supplier systems" | documented (RH-R) |
| A7 | `http:202` (accepted, outcome pending) | create_order | `reconcile` — documented "do not retry … may result in a duplicate order"; resolve via probe + order.created / order.creation_failed webhook | documented (RH-O, TEST) |
| A8 | webhook `order.creation_failed` observed during reconcile | create_order (reconcile resolution) | not-landed with cause → re-feed ONCE per §2.3; absent a finer code → `dead_end(permanent: false)` | documented event (TEST); payload codes undocumented |
| A9 | timeout / conn drop / `http:500` / ambiguous 5xx | create_order, confirm_order_cancellation | `reconcile` (unkeyed commit; NEVER retry) | documented risk (RH-O) |
| A10 | `validation_error` with `source.field` on passengers/services/payment | create_order | `repair(to: create_order, fields: [passengers, services, payment])` — user fixes data | documented envelope (ERR) |
| A11 | `code:order_cancellation_stale` | confirm_order_cancellation | `rewind(to: create_order_cancellation)` — re-quote; gate re-fires on the new quote | documented (CXL) |
| A12 | quote `expires_at` past (surfaced as stale on confirm) | confirm_order_cancellation | `rewind(to: create_order_cancellation)` | documented (CXL) |
| A13 | precondition: `"cancel" ∉ available_actions` | get_order (pre-compensation) | `dead_end(reason: cancel_not_available_via_api, permanent: true)` — manual/dashboard path; operator | documented (CXL) |

### 4.4 Gates (Scope 1)

```yaml
gates:
  book_consent:                       # entry_gate of create_order
    audience: user                    # money — never model-answerable
    payload: {total_amount: money, currency: string, conditions: object,
              services_total: money, expires_at: datetime}
    outcomes:
      approve: ok                     # proceed to dispatch
      decline: dead_end               # planner decides what's next, outside the chain
    timeout: {after: 20m, verdict: dead_end}   # offer ttl_hint ~30m; don't outlive it

  price_review:                       # raised by get_offer precondition on drift
    audience: user
    payload: {old: money, new: money, currency: string}
    outcomes:
      accept:  ok                     # get_offer's output stands; advance
      decline: reselect(to: select_offer)   # pick a different offer from live results
    timeout: {after: 20m, verdict: dead_end}

  cancel_quote_consent:               # inside compensation sub-chain (state=compensating)
    audience: user
    payload: {refund_amount: money, refund_currency: string, refund_to: enum,
              airline_credits: object[], expires_at: datetime}
    outcomes:
      confirm: ok                     # proceed to confirm_order_cancellation
      keep_order: dead_end            # abandon the unwind; commit stands; escalate to
                                      # operator if this was a dead_end-driven unwind
    timeout: {after: 72h, verdict: dead_end}   # → operator per C8.5: unwind stuck
```

---

## Scope 2 — `duffel_hold_order`

Shares `create_offer_request → select_offer → get_offer → get_seat_maps` and the
cancellation sub-chain with Scope 1 (same handles, same rows A1–A3, A11–A13).

### 5.1 Handles (additions/overrides)

```yaml
handles:
  hold_order_id:                      # durable; distinct id space use from scope 1 only
    minted_by: create_hold_order      # in that payment state machinery hangs off it
    derived_from: [offer_id]
    staleness: {detect: [], ttl_hint: n/a, refresh: n/a}   # durable object; its PAYABILITY
                                       # expires (payment_required_by) but the handle does
                                       # not — payability is payload state read via
                                       # get_order, enforced by provider signals, not timers
  # payment_required_by / price_guarantee_expires_at are PAYLOAD fields on the order,
  # not handles: they gate the pay_order business action and are re-read via get_order.
  # Per P2, they are planning hints (ttl_hint-like) — enforcement is the provider's
  # documented signals (past_payment_required_by_date, price_changed).
```

### 5.2 Actions (additions)

```yaml
actions:
  - id: create_hold_order
    description: Book with type=hold, no payment; airline holds space until
                 payment_required_by.
    effect: commit                    # RUBRIC (elicitation Q-H1): unpaid hold costs
                                      # nothing and self-expires, BUT holds real airline
                                      # inventory and is user-visible (PNR) — repeating
                                      # it is consequential ⇒ COMMIT, not mint
                                      # (SPEC §2.1: inconsequence, not self-expiry).
    input:
      intent:
        passengers: {type: object, required: true}
        services:   {type: object, required: false}
      handles: [offer_id]
    output:
      payload: {order: object, booking_reference: string,
                payment_status: {awaiting_payment: bool,        # true on creation
                                 payment_required_by: datetime,
                                 price_guarantee_expires_at: datetime | null}}
      handles: [hold_order_id]
    request_invariants: {type: hold}   # and NO payments key (HOLD)
    preconditions:
      # evaluated on the selected offer BEFORE this step dispatches (via get_offer output)
      - when: "payment_requirements.requires_instant_payment == true"
        verdict: reselect(to: select_offer)
        reason: "offer cannot be held (documented flag, OFF); pick a holdable offer"
    idempotency: {mode: none}          # same absence of keys as scope 1
    entry_gate: hold_consent
    confirmation:
      probe: list_orders_by_offer
      signal: "orders[] contains order matching correlation record's offer_id"
      async: {channel: webhook, deadline: 6h}
      sweep: {interval: 30m, escalate_after: 24h}
    compensation:
      chain: [create_order_cancellation, confirm_order_cancellation]
      window: "unpaid hold ALSO self-expires at payment_required_by — the airline
               releases the space and awaiting_payment flips to false (HOLD). Lapse is
               a valid zero-cost undo for unpaid holds; explicit cancellation is the
               deterministic one. refund_to: awaiting_payment exists for the
               nothing-was-paid case (OC)."
      ordering_note: "for replace flows, land the replacement before compensating"
    timeout: 150s
    latency_hint: 30s

  - id: get_current_order_state        # the re-price read; refresh target for pay
    description: Re-read the hold order for current total, payment window, guarantee.
    effect: read
    input: {handles: [hold_order_id]}
    output: {payload: {total_amount: money, payment_status: object,
                       available_actions: string[], documents: tickets[]}}
    preconditions:
      - when: "payment_status.awaiting_payment == false && documents.empty
               && payment_required_by < now"
        verdict: dead_end              # hold lapsed: space released by the airline
        reason: "documented lapse semantics (HOLD); permanent for THIS order —
                 planner starts a new run/search. ⚠ awaiting_payment==false alone is
                 ambiguous (same value when paid) — hence the compound predicate (§12)"
      - when: "total_amount != consented_amount"
        verdict: gate(pay_consent)     # price drifted after guarantee expiry; re-consent
        reason: "documented: price re-fetched after guarantee expiry; 'get the most
                 up-to-date price by retrieving the order' (HOLD)"
    idempotency: {mode: natural}
    timeout: 30s

  - id: pay_order                      # THE second commit
    description: Pay for the hold order within the payment window.
    effect: commit
    input:
      intent:
        amount:   {type: money,  required: true}   # code-derived from
        currency: {type: string, required: true}   # get_current_order_state payload
      handles: [hold_order_id]
    output: {payload: {payment: object}}
    request_invariants: {payment.type: balance}
    idempotency: {mode: none}          # no key (MR); attempts=1 forced
    entry_gate: pay_consent
    confirmation:
      probe: get_current_order_state
      signal: "payment_status.awaiting_payment == false AND documents (tickets)
               present — the AND guards against the documented lapse case where
               awaiting_payment also flips false (HOLD); ticket-issuance-on-payment
               is ASSUMED (§12)"
      async: {channel: webhook, deadline: 6h}      # order.updated expected; payload
                                                    # semantics undocumented (§12)
      sweep: {interval: 30m, escalate_after: 24h}
    compensation:
      chain: [create_order_cancellation, confirm_order_cancellation]
      window: "post-payment cancellation refunds per airline rules; void window per
               void_window_ends_at (ORD)"
    timeout: 150s
    latency_hint: 20s
```

### 5.3 Domain verdict rows (Scope 2; in addition to shared A-rows)

| # | Signal | Step scope | Verdict | Source |
|---|---|---|---|---|
| H1 | `code:price_changed` (422 validation on amount) | pay_order | `rewind(to: get_current_order_state)` — re-read current total; `pay_consent` entry gate re-fires with the new amount (new price ⇒ new consent, by construction) | documented (PAY, HOLD) |
| H2 | `code:past_payment_required_by_date` (invalid state) | pay_order | `dead_end(reason: hold_lapsed, permanent: true_for_this_order)` — airline released the space; planner may start a NEW run (fresh search) | documented (PAY, HOLD) |
| H3 | `code:already_paid` | pay_order | `reconcile` — this signal is strong evidence a previous ambiguous attempt LANDED; the probe (not the signal alone) resolves to ok | documented (PAY) |
| H4 | `code:already_cancelled` | pay_order | `dead_end(reason: order_cancelled, permanent: true)` | documented (PAY) |
| H5 | `code:schedule_changed` | pay_order | `gate(schedule_change_review)` — airline-side mutation (AIC); human decides accept-and-continue vs abandon | documented (PAY) |
| H6 | `code:payment_amount_does_not_match_order_amount` / `payment_currency_does_not_match_order_currency` | pay_order | `rewind(to: get_current_order_state)` — amount/currency are code-derived from the order read; a mismatch means our read is stale, not that the user must act | documented (PAY) |
| H7 | `code:order_type_not_eligible_for_payment` | pay_order | `dead_end(reason: config_bug, permanent: true)` — paying a non-hold order is a programming error; surface loudly | documented (PAY) |
| H8 | timeout / conn drop / ambiguous 5xx | create_hold_order, pay_order | `reconcile` | documented risk (RH-O) |
| H9 | `http:503` | create_hold_order, pay_order | `retry` (`rejected_before_execution: true`, documented no-side-effect guarantee) | documented (RH-R) — the guarantee is stated for booking creation; its applicability to /air/payments is ASSUMED-adjacent, see §12 |

### 5.4 Gates (Scope 2)

```yaml
gates:
  hold_consent:                        # entry_gate of create_hold_order
    audience: user
    payload: {total_amount: money, payment_required_by: datetime,
              price_guarantee_expires_at: datetime | null, conditions: object}
    outcomes: {approve: ok, decline: dead_end}
    timeout: {after: 20m, verdict: dead_end}

  pay_consent:                         # entry_gate of pay_order; re-fires after every
    audience: user                     # rewind through get_current_order_state, so a
    payload: {current_total: money, currency: string,                # re-priced hold is
              price_guarantee_expires_at: datetime | null,          # ALWAYS re-consented
              payment_required_by: datetime}
    outcomes:
      approve: ok
      decline: dead_end                # unpaid hold self-expires at payment_required_by
                                       # (documented) — declining is cheap
    timeout: {after: 12h, verdict: dead_end}
    # NOTE: gate parking is wall-clock-exempt (SPEC §2.4); payment_required_by is a
    # PROVIDER deadline that keeps running while parked. The 12h cap is a product
    # choice to resurface the decision well inside typical windows; the provider
    # signal H2 remains the enforcement (P2: no local timer kills the run).

  schedule_change_review:
    audience: user
    payload: {aic: object}             # added/removed slices from the AIC resource
    outcomes:
      accept_and_pay: rewind(to: get_current_order_state)   # re-read, re-consent, pay
      abandon: dead_end                # planner may route to cancellation chain
    timeout: {after: 24h, verdict: dead_end}
```

---

## Scope 3 — `duffel_order_change`

Input: an existing `order_id` (from either booking scope — see §11).

### 6.1 Handles

```yaml
handles:
  order_id:                            # imported, durable (see §11 cross-chain notes)
    minted_by: external                # minted by create_order / create_hold_order+pay
    derived_from: []
    staleness: {detect: [], ttl_hint: n/a, refresh: n/a}
  change_request_id:
    minted_by: create_change_request
    derived_from: [order_id]
    staleness: {detect: [], ttl_hint: unknown, refresh: create_change_request}
    # ocr_ expiry undocumented; its OFFERS expire individually (OCR/OCO). Fail-closed.
  change_offer_id:
    minted_by: create_change_request   # offers returned inline (OCR)
    derived_from: [change_request_id]
    staleness:
      detect: []                       # UNDOCUMENTED error code for using an expired
                                       # change offer (§12) → default rows (fail closed)
      ttl_hint: unknown                # expires_at returned per offer (OCO); no typical
                                       # duration documented
      refresh: create_change_request
  selected_change_offer:               # selection pseudo-handle (I3)
    minted_by: select_change_offer
    derived_from: [change_offer_id]
    rematch:
      key: [added_slice_carrier_flight_numbers, added_slice_departure_datetimes,
            change_total_amount_tolerance]           # ASSUMED (§12)
      on_ambiguous: gate               # money differs across change offers → user re-picks
    staleness: {detect: [], ttl_hint: unknown, refresh: select_change_offer}
  pending_change_id:
    minted_by: create_pending_order_change
    derived_from: [selected_change_offer]
    single_use: true                   # spent by confirm (I4); after an ambiguous confirm
                                       # never re-present — re-mint instead
    staleness:
      detect: []                       # stale/expired-confirm error code UNDOCUMENTED
                                       # (no order_change_stale analog found) → default
                                       # rows (fail closed); expires_at documented (OCH)
      ttl_hint: unknown
      refresh: create_pending_order_change
```

```
order_id (external) ──> change_request_id ──> change_offer_id
                                                └─> selected_change_offer (pseudo)
                                                      └─> pending_change_id
                                                            └─(confirm mutates order_id)
```

### 6.2 Actions

```yaml
actions:
  - id: create_change_request
    description: Search the change space (slices to remove + criteria to add).
    effect: read                       # double-call: another ocr_ result set; no
                                       # documented side effect on the order (leaning
                                       # documented; residual assumption noted §12)
    input:
      intent:
        remove_slice_ids: {type: object, required: true}   # slice ids from get_order —
                                       # provider values entering intent via a validated
                                       # presented payload (SPEC §2.6 carve-out)
        add_slices: {type: object, required: true}         # origin/destination/
        cabin_class: {type: enum[...], required: false}    # departure_date
      handles: [order_id]
    output:
      payload: order_change_offers[]
      handles: [change_request_id, change_offer_id]
      empty: {valid: true, detect: "order_change_offers.length == 0",
              route: repair(to: create_change_request, fields: [add_slices, cabin_class])}
              # documented: "will be empty if no offers are available" (OCR)
    preconditions:
      - when: "'change' ∉ order.available_actions"      # checked on the get_order read
        verdict: dead_end
        reason: "order not changeable via API (CHG); manual/dashboard path — permanent"
    idempotency: {mode: natural}
    timeout: 60s
    latency_hint: 15s

  - id: select_change_offer
    effect: select
    input: {handles: [change_offer_id]}
    output: {handles: [selected_change_offer]}

  - id: create_pending_order_change
    description: Turn the selected change offer into a confirmable pending change.
    effect: mint                       # quote-like, self-expiring, not airline-visible
                                       # (ASSUMED on the last point, §12)
    input: {handles: [selected_change_offer]}
    output:
      payload: {order_change: object, expires_at: datetime,
                change_total_amount: money, penalty_total_amount: money,
                refund_to: enum[original_form_of_payment, airline_credits, voucher]}
      handles: [pending_change_id]
    idempotency: {mode: natural}       # re-minting supersedes (assumed parallel to the
                                       # documented cancellation most-recent-quote rule)
    timeout: 60s
    latency_hint: 10s

  - id: get_order_change               # re-price read; refresh of the CONSENT basis
    description: Latest price of the pending change (documented drift).
    effect: read
    input: {handles: [pending_change_id]}
    output: {payload: {change_total_amount: money, penalty_total_amount: money,
                       expires_at: datetime}}
    preconditions:
      - when: "change_total_amount != consented_amount"
        verdict: gate(change_confirm_consent)   # re-consent on drift
        reason: "documented: pending change price can change over time; review final
                 price before confirming (OCH)"
      - when: "expires_at < now"
        verdict: rewind(to: create_pending_order_change)
        reason: "pending change must be confirmed by expires_at (OCH); payload-predicate
                 detection since no error code is documented"
    idempotency: {mode: natural}
    timeout: 30s

  - id: confirm_order_change           # THE COMMIT — in-place mutation
    description: Execute the change; airline booking updated with the new slices.
    effect: commit
    mutates: order_id                  # documented: "The booking with the airline will
                                       # be updated with the new slice" (OCH) — no new
                                       # order; existence proves nothing
    input:
      intent:
        payment: {type: object, required: false}   # required iff change_total_amount > 0
                                                   # (documented, OCH); type: balance
      handles: [pending_change_id, order_id]
    output: {payload: {order_change: object}}
    request_invariants: {payment.type: balance}
    idempotency: {mode: none}          # no key → attempts=1, P3 mechanics
    entry_gate: change_confirm_consent # penalty + delta consent; re-fires on every
                                       # arrival incl. replays (SPEC entry_gate)
    confirmation:
      # mutates ⇒ CONTENT-based: compare, never check existence
      probe: get_order
      signal: "order.slices now contain the change offer's added slices and no longer
               contain the removed slices (slice ids may be NEW after a change — match
               by content: carrier+flight numbers+times; AIC docs document id churn);
               corroborate via get_order_change status (status enum UNDOCUMENTED, §12)"
      async: {channel: webhook, deadline: 6h}      # order.updated presumed relevant;
                                                    # payload semantics undocumented (§12)
      sweep: {interval: 30m, escalate_after: 24h}
    compensation:
      action: none
      ordering_note: "no documented undo of a confirmed change — recourse is a NEW
                      change or cancellation, which are new business decisions, not
                      compensation. Per SPEC rule, protection sits BEFORE the commit:
                      entry_gate change_confirm_consent is the named pre-commit safety
                      mechanism and re-fires on every recovery replay."
    timeout: 150s
    latency_hint: 30s
```

### 6.3 Domain verdict rows (Scope 3)

| # | Signal | Step scope | Verdict | Source |
|---|---|---|---|---|
| X1 | `ok(empty)` change offers | create_change_request | `repair(to: create_change_request, fields: [add_slices, cabin_class])` | documented empty case (OCR) |
| X2 | payload: pending change `expires_at < now` | get_order_change (pre-confirm) | `rewind(to: create_pending_order_change)`; if the change offer also expired the replay hits X3 and climbs | documented expiry (OCH); predicate detection assumed |
| X3 | change-offer-expired reject at create_pending_order_change (exact code UNDOCUMENTED) | create_pending_order_change | falls to default rows → non-commit step → `dead_end`… **overridden by config**: any 422 `invalid_state_error`/`airline_error` on this step → `rewind(to: create_change_request)` (fresh offers) | assumed mapping (§12) — conservative alternative is fail-closed default |
| X4 | precondition: `change_total_amount` drifted | get_order_change | `gate(change_confirm_consent)` (re-consent) | documented drift (OCH) |
| X5 | precondition: `'change' ∉ available_actions` | create_change_request | `dead_end(reason: change_not_available_via_api, permanent: true)` — dashboard/support | documented (CHG) |
| X6 | timeout / conn drop / ambiguous 5xx / unknown 4xx-5xx code | confirm_order_change | `reconcile` — probe compares order content (mutates) | documented risk pattern (RH-O); content-compare per SPEC `mutates` |
| X7 | `validation_error` on payment amount at confirm | confirm_order_change | `rewind(to: get_order_change)` — re-read latest delta, re-consent | documented drift + payment-on-confirm (OCH) |
| X8 | webhook `order.airline_initiated_change_detected` arrives mid-chain for this order | any step, this run | `gate(schedule_change_review_change_chain)` at next cursor evaluation — the change space was computed against a mutated order; cached change offers are suspect | documented event (WH/AIC); routing is a config decision, see §11 |

### 6.4 Gates (Scope 3)

```yaml
gates:
  change_confirm_consent:              # entry_gate of confirm_order_change
    audience: user                     # money (penalty + delta) — never model
    payload: {change_total_amount: money, penalty_total_amount: money,
              new_total_amount: money, refund_to: enum, added_slices: object,
              removed_slices: object, expires_at: datetime}
    outcomes:
      approve: ok
      decline: dead_end                # order stands unchanged; nothing to unwind
    timeout: {after: 2h, verdict: dead_end}   # pending change expires provider-side
                                              # anyway; provider signal is enforcement

  schedule_change_review_change_chain: # X8: AIC landed mid-change
    audience: user
    payload: {aic: object}
    outcomes:
      restart_change: rewind(to: create_change_request)   # recompute against new order
      abandon: dead_end
    timeout: {after: 24h, verdict: dead_end}
```

---

## 8. Shared verdict-table core + known_unmatched

Ordering per SPEC §3: domain rows (A/H/X above) first, then:

| # | Signal | Applies to | Verdict | Source |
|---|---|---|---|---|
| G1 | handle staleness per `staleness.detect` (incl. inside 2xx payloads) | consumers | `rewind(to: refresh)` — shallowest reported-dead wins; unattributable staleness escalates deepest-first (SPEC §3) | per-handle rows above |
| G2 | transport success + `empty.detect` | empty-valid actions | `ok(empty)` → route | documented empties (TEST/SM/OCR) |
| G3 | transport success (fallthrough after ALL payload rows) | any | `ok` | — |
| G4 | `http:429 rate_limit_exceeded` | read / mint | `retry` after `ratelimit-reset` | documented (RH-R) |
| G5 | `http:429` | unkeyed commits (create_order, create_hold_order, pay_order, confirm_order_change, confirm_order_cancellation) | `reconcile` — NOT declared `rejected_before_execution`: pre-side-effect rejection is plausible for an edge limiter but UNDOCUMENTED (§12) | conservative per SPEC row 5 |
| G6 | `http:5xx` / timeout / conn reset | read / mint | `retry` | documented guidance (RH-R) |
| G7 | `http:401/403` auth errors | any | `dead_end(reason: auth, permanent: true)` — config/credentials bug | documented (ERR) |
| G8 | `validation_error` not matched by a domain repair row | any | `dead_end(permanent: true)` — request-shape bug | documented (ERR) |
| G9 | unmatched signal, commit in flight | commits | `reconcile` | SPEC fixed row |
| G10 | unmatched signal, otherwise | any | `dead_end` | SPEC fixed row |

### known_unmatched (consciously routed to G9/G10 defaults)

- `offer_expired` — `documented(search-snippet)` only (attributed to Duffel
  getting-started/help content; not verified on any fetched page). If real, it belongs
  with A1 (`rewind(to: create_offer_request)`). Until verified: default rows.
- `airline_error` codes generally — ERR documents the *type* but no code enumeration;
  airline passthrough errors are open-ended. Default rows (commit → reconcile is the
  safe posture for a mid-commit airline burp).
- The full `invalid_request_error` / `validation_error` code list (`bad_request`,
  `missing_content_type_header`, `missing_data_param`, `not_found`,
  `unsupported_format`, `access_token_not_found`, `expired_access_token`,
  `insufficient_permissions`) — G7/G8/G10 handle these adequately.
- Any stale/expired error code for change offers and pending order changes (no
  documented analog of `order_cancellation_stale`) — X3's config override for 422s on
  the mint is the only carve-out; everything else defaults.
- `order.creation_failed` webhook payload error codes — undocumented; A8 handles the
  event coarsely.
- Card-payment paths: the documented `200 partial` outcome ("booking exists in the
  supplier system but complete information wasn't retrievable" — card payments only,
  RH-O) and 3DS session codes are OUT OF SCOPE (balance pinned via
  `request_invariants`); listed here so the omission is conscious.

---

## 9. Policy

```yaml
policy:
  per_step:
    read:  {attempts: 3, backoff: {base: 500ms, factor: 2, jitter: full, max: 8s}}
    mint:  {attempts: 2, backoff: {base: 1s, factor: 2, jitter: full, max: 10s}}
    commit:
      unkeyed: {attempts: 1}          # ALL Duffel commits are unkeyed (Q4) — this row
                                      # is the whole ballgame. The single documented
                                      # exception path: http:503 (A6/H9) is
                                      # rejected_before_execution and may re-dispatch;
                                      # it does so within a bounded extra budget of 2
                                      # (see §14 friction on attempts=1 vs row-5 retry).
    compensate: {attempts: 1}         # confirm_order_cancellation is unkeyed too
    # overrides
    create_offer_request: {attempts: 2, timeout: 60s}   # search fan-out; retry once
    create_order:         {timeout: 150s}   # documented: client timeout MUST be ≥130s
    create_hold_order:    {timeout: 150s}
    pay_order:            {timeout: 150s}
    confirm_order_change: {timeout: 150s}
    confirm_order_cancellation: {timeout: 150s}
  per_chain:
    max_rewinds: 3
    max_repairs: 2
    compensation_order: reverse       # scope 2 lands ≤2 commits (hold, pay); unwinding
                                      # runs the cancellation sub-chain once — cancelling
                                      # the order supersedes unwinding the payment
                                      # (refund rides refund_to per the quote)
    wall_clock: 30m                   # active execution: 3× Σ latency_hints (~2m) is
                                      # dwarfed by 2× slowest timeout (300s) → padded to
                                      # 30m to cover reconcile PROBE cycles; the
                                      # hours-scale webhook wait lives in
                                      # state=reconciling under async.deadline (6h) +
                                      # sweep, NOT under wall_clock (see §14 friction —
                                      # SPEC is ambiguous here; this config treats
                                      # parked-reconciling like parked-at-gate)
    gate_timeout: 30m                 # default; per-gate values above take precedence
  escalation:
    - {audience: model,    may: [propose repair fields, choose among returned offers/
                                 change offers, rematch-ambiguity re-selection]}
    - {audience: user,     may: [answer book/hold/pay/change/cancel consent gates,
                                 approve repairs with cost]}
    - {audience: operator, may: [resolve reconcile deadline, unstick PENDING correlation
                                 records, handle cancel-not-available/insufficient-balance,
                                 dashboard-only manual actions]}
```

---

## 10. Invalidation walkthroughs (SPEC §4 applied by hand)

**W1 — Offer dies at the commit (Scope 1, the classic).**
`create_order` dispatch returns 422 `offer_no_longer_available` (A1). Verdict
`rewind(to: create_offer_request)`; rewind budget 3→2. I1: re-minting
`offer_request_id` invalidates `offer_id` → `selected_offer` (pseudo) → `seat_map_id`,
and every cached payload (offer list, seat maps, service prices). I2: the new offers
collection REPLACES the old (lineage-keyed; a seat service id from the old offer can
never ride the new create_order request — it died with `seat_map_id`). Replay:
cursor=create_offer_request → new offers → the selection pseudo-step re-establishes via
`rematch` (carrier+flight numbers+times) — exact match → code re-selects silently; no
match (schedule shifted) → `on_ambiguous: model` picks afresh (a NEW selection per I3).
`get_offer` re-runs; if the price moved, the precondition raises `gate(price_review)`
— and regardless, `create_order.entry_gate book_consent` re-fires before dispatch
(entry gates re-fire on EVERY arrival), so the user re-consents to the new price. Seat
intent fields (service ids) were invalidated with their payload; they are re-chosen
from the fresh seat map or dropped (degraded, traced). Note the interplay with I4:
`offer_id` is `single_use` — even had the rewind not fired, a consumed offer is never
re-presented after an ambiguous attempt.

**W2 — Ambiguous create_order death + duplicate-avoidance (Scope 1, P3 mechanics).**
Correlation record {run_id, offer_id, intent hash, PENDING} persisted; dispatch; the
process dies at t+90s (inside the documented ≤120s supplier window). Resume from
snapshot: state=reconciling (C1.2 — never re-dispatch). Probe
`list_orders_by_offer(offer_id)` → empty (documented lag: "might take a couple of
hours"). The run stays `reconciling`; sweep re-probes every 30m; the `order.created`
webhook, if it arrives, matches the record by `offer_id` and resolves → ok, handle
`order_id` minted from the webhook/probe payload. If `order.creation_failed` arrives
instead: not-landed with cause → re-fed through the table ONCE (A8) → dead_end
(permanent: false). If neither arrives by async.deadline (6h) → escalate(operator),
sweep continues to 24h. At no point is a second create_order dispatched — the
documented duplicate-order warning is exactly the double-booking bug the fixed rows
prevent. `offer_id` was spent (I4): any later rewind path re-mints via a fresh search.

**W3 — Hold re-price cascade (Scope 2).**
`price_guarantee_expires_at` lapses (no local timer fires — P2). `pay_order` is
dispatched with the stale consented amount → 422 `price_changed` (H1) →
`rewind(to: get_current_order_state)`; budget 3→2. I1: re-running the read replaces the
order-state payload (the old total is dead cache; `hold_order_id` itself — a durable
commit output — is untouched per I5, and `commits_landed` still lists
`create_hold_order`). Cursor returns to `pay_order`; its `entry_gate pay_consent`
re-fires with the NEW total (fresh consent by construction — C8.4 shape). User
declines → dead_end; compensation policy: `commits_landed` non-empty → the
cancellation sub-chain runs (state=compensating): quote → `cancel_quote_consent` gate
shows `refund_to: awaiting_payment` (nothing was paid) → confirm; OR product config may
elect the documented zero-cost lapse (space auto-released at `payment_required_by`) —
this compile keeps the explicit cancellation as the deterministic unwind and notes the
lapse alternative in the compensation window.

**W4 — Pending change expires under the consent gate (Scope 3).**
User parks at `change_confirm_consent` for 3h; pending change `expires_at` passes
provider-side. Gate resumes `approve` → cursor re-arrives at `confirm_order_change` —
but first `get_order_change` (the pre-confirm read) runs on replay and its
precondition `expires_at < now` fires → `rewind(to: create_pending_order_change)`;
budget decremented. I1: `pending_change_id` re-minted → nothing downstream of it
exists yet; the selection pseudo-handle `selected_change_offer` is UPSTREAM and
survives — but if `create_pending_order_change` rejects because the change OFFER also
died (X3), the rewind climbs to `create_change_request`, which invalidates
`change_offer_id` → `selected_change_offer` → re-selection per rematch
(`on_ambiguous: gate` — money differs, user re-picks). `entry_gate
change_confirm_consent` re-fires before any confirm dispatch, with the re-quoted
penalty/delta. The confirmed order is never touched until the single confirm dispatch
lands; because the commit `mutates: order_id`, its reconcile path (X6) compares slice
CONTENT, never existence.

---

## 11. Cross-chain notes

- **`order_id` is the shared durable handle.** Scope 1's `create_order` or Scope 2's
  `create_hold_order`+`pay_order` mint it; Scope 3 and the cancellation sub-chain
  import it (`minted_by: external`). Landed commits are facts (I5): no rewind in the
  change chain ever invalidates the order itself.
- **Cross-run serialization is out of scope** (SPEC §8 non-goal): a change run racing a
  cancellation run on one `order_id` needs deployment-level per-object locking. Flagged,
  not papered over.
- **Airline-initiated changes are the external-mutation risk.** Duffel documents AIC
  resources (`added`/`removed` slices, ids may change) and the
  `order.airline_initiated_change_detected` webhook. Consequences per scope:
  Scope 2 — `schedule_changed` blocks payment (H5, documented). Scope 3 — a mid-chain
  AIC silently invalidates the change-request search space; the config routes the
  webhook to a gate (X8). Handling the AIC itself (accept / update endpoints, AIC) is
  a **separate chain** (accept is another unkeyed in-place commit) — deliberately not
  compiled here.
- **Webhook infrastructure is shared**: at-least-once delivery, event
  `idempotency_key` dedup, HMAC signatures (WH). Reconcile resolution for all three
  scopes' commits rides the same consumer; correlation keys: `offer_id` (order
  creation), `order_id` (payment/change/cancellation observations).
- **Replace-before-compensate**: if a planner rebooks (new instant order replacing a
  cancelled one), the replacement `create_order` must land before the original's
  cancellation sub-chain confirms — stated in both commits' `ordering_note`s.

---

## 12. UNKNOWNs & assumptions (each needs a human answer before implementation)

1. **Offer single-use enforcement** — `[A]` Duffel does not document whether a
   successfully-booked offer is rejected on a second create_order. Config assumes NOT
   enforced (duplicate documented possible) and imposes client-side `single_use: true`.
   *Ask Duffel: can one offer ever yield two orders? Always? Sometimes?*
2. **429 on commits: rejected before side effects?** — `[A]` Plausible (platform edge
   limiter) but undocumented → G5 routes to `reconcile`, which is operationally noisy.
   *Ask Duffel: is `rate_limit_exceeded` ever returned after supplier dispatch has
   begun?* If provably pre-execution, flip G5 to `retry` via
   `rejected_before_execution: true`.
3. **GET on an expired offer** — behavior undocumented (404? stale payload?). A2's
   payload predicate (`expires_at < now`) is assumed detection.
4. **Rematch keys (both selection pseudo-handles)** — `[A]` intent-level identity
   (carrier + flight numbers + datetimes + amount tolerance) chosen by analogy, not
   documentation. Validate against real re-search behavior; tune amount tolerance.
5. **Pending cancellation side effects** — `[A]` assumed inert on the order (docs say
   "review step" only). *Does an unconfirmed cancellation quote block payment,
   changes, or other quotes?*
6. **Confirmed-cancellation content signal** — the exact field proving confirmation on
   `GET /air/order_cancellations/{id}` is undocumented (`confirmed_at`? a status
   enum?). The `credit_code` non-null signal is documented but only for the
   airline-credits path.
7. **Hold-order fee** — none documented; assumed free. Verify (some carriers charge
   for holds).
8. **`awaiting_payment: false` ambiguity** — documented oddity: the lapsed-unpaid hold
   sets the same flag value as the paid state (HOLD). The compound probe predicates
   (documents present / `payment_required_by` past) are assumed sufficient. *Ask: is
   there a dedicated lapsed/cancelled marker on an expired hold? Does the order get
   `cancelled_at`?*
9. **Ticket issuance timing on hold payment** — `[A]` `documents` assumed populated
   once payment settles; used in pay_order's confirmation signal.
10. **503 no-side-effect guarantee scope** — documented for booking creation (RH-R);
    its extension to `/air/payments` and `/air/order_changes/…/confirm` (H9, and by
    implication X-rows) is assumed. If unverifiable, drop `rejected_before_execution`
    on those steps → 503 becomes `reconcile`.
11. **Change-offer / pending-change stale error codes** — no `order_change_stale`
    analog documented. X3's 422→rewind override is assumed; the fail-closed
    alternative (pure default rows) is noted inline. Pending-change expiry is caught
    by the assumed payload predicate in get_order_change.
12. **Order-change statuses** — `get_order_change` status enum undocumented; the
    confirm probe therefore leans entirely on order-content comparison.
13. **`order.updated` webhook semantics** — event exists (WH) but payload/trigger
    conditions undocumented; async confirmation channels for pay/change assume it
    fires. If not, `channel: poll` (sweep) carries the load.
14. **`offer_expired` code** — search-snippet provenance only; in known_unmatched.
15. **Offer-request and change-request TTLs** — undocumented; both handles have empty
    `staleness.detect` → consumers fail closed by design (SKILL Step 2 rule).
16. **Where the correlation record lives** — this compile mandates a durable store
    (transactional DB row written before dispatch, per P3/C1.4) but the deployment
    target is an implementation decision outside this document.

---

## 13. Boundary recap (every model touchpoint, vs SPEC §5)

| Touchpoint | Where | SPEC §5 allowance |
|---|---|---|
| Pick an offer | select_offer (I3 pseudo-step) | "Choosing among returned options" — recorded as derivation |
| Pick a change offer | select_change_offer | same |
| Re-pick after rematch ambiguity | rematch `on_ambiguous: model` (Scope 1) | same — a NEW selection per I3; Scope 3 routes ambiguity to a **gate** instead (money differs) |
| Propose repair values | search fields (A/X empty routes), passenger/service fixes (A10), change-search criteria (X1) | "Proposing values for repair fields" — code validates against the listed intent fields |
| Seat/service choice | intent fields validated against presented seat-map payload | §2.6 carve-out: payload-validated opaque values entering intent |
| Phrase gate payloads & dead-end explanations | all gates; dead_end reporting | "Phrasing outcomes" — from machine-readable reasons |
| Decide what happens after dead_end | planner, above the chain | explicitly outside the spec |
| **Never**: handles, budgets, verdicts, retries of commits, gate answers about money | — | book/hold/pay/change/cancel consent gates are all `audience: user`; `verdict_source` ∈ {table, default} only |

---

## 14. Self-check + spec-friction notes

Self-check (SKILL Step 7): every commit/compensator declares `confirmation` (all are
probes — no keyed commit exists in Duffel, so `by_key_replay` is unused); all unkeyed
commits carry attempts=1 + correlation record + cancellation shield; both selection
pseudo-handles have `rematch`; both `mutates` commits (confirm_order_change,
confirm_order_cancellation) have content-based signals and explicit compensation
stances (`none` + named entry_gate / consent gate); every handle a commit consumes is
durable or has `staleness.detect` (or is explicitly UNKNOWN-fail-closed: change_offer_id
and pending_change_id, §12.11 — the pre-confirm read's payload predicate is the
declared detection); `single_use` handles (offer_id, cancellation_quote_id,
pending_change_id) each consumed by exactly one step; every documented error code is
mapped (A/H/X/G rows) or in known_unmatched; every gate has audience/outcomes/timeout;
empty routes exist for all Q9-yes steps; walkthroughs W1–W4 trace the derived_from
edges; no TTL is enforced by timer (payment_required_by / expires_at are provider-
enforced, locally planning-only); no budget lives in prompt text; compensation
ordering stated for replace flows.

**Friction encountered** (detail in the compile report): (a) SPEC row 5 permits
`retry` on an unkeyed commit for `rejected_before_execution` signals while §2.4 calls
`unkeyed: {attempts: 1}` non-negotiable — this config reads attempts=1 as "one attempt
per non-exempt signal" and grants 503 a bounded extra budget of 2, but the spec should
say which reading is intended. (b) `wall_clock` "covers active execution including
reconcile probes" collides with Duffel's documented hours-scale creation lag; this
config treats webhook-wait in state=reconciling as parked (wall-clock-exempt), like
gates — needs a spec ruling. (c) The evidence tagging scheme (`documented`/`observed`)
lacks a tier for "attributed to provider docs by a search engine but unverified" —
`documented(search-snippet)` was invented here and kept non-load-bearing.

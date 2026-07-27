# Booking.com Demand API — Accommodation chain compile

Created: 2026-07-27
Last Updated: 2026-07-27
Compiled against: `SPEC.md` Draft v0.3.2 (pre-implementation), via `SKILL.md` (chain-compile) end to end.
Evidence corpus: Booking.com public Demand API documentation at `https://developers.booking.com/demand/docs/…` (v3.1 stable + v3.2/Beta pages, fetched 2026-07-27 via the site's markdown renditions, e.g. `…/orders-api/order-preview-create.md`). Every claim below carries a source tag: `documented` (official docs page, URL cited) or `assumed` (my inference — listed again in §UNKNOWNs). No `observed` evidence exists (no live traffic was run). SPEC worked examples were treated as `spec-example` hearsay and used for nothing.

Scopes in this document (SKILL multi-scope compile — shared elicitation and generic table, per-scope handle graphs / actions / domain rows / gates / walkthroughs):

- **Scope A — Stay booking chain**: `accommodations/search → (select property) → accommodations/details → accommodations/availability → (select product) → orders/preview → orders/create → confirmation`, compensation = the cancel sub-chain.
- **Scope B — Post-booking chains**: `orders/modify` (in-place `mutates` commit, with `orders/modify/preview` Beta) and `orders/cancel` (fee schedule, free-cancellation deadline as a business-timescale boundary, cancel-for-less waiver).

---

## 1. Chain summary

**Scope A.** Book one accommodation stay for a traveller: search a location for dates+occupancy, pick a property, pull live per-room *products* (rate offers) for that property, pick product(s), mint a 15-minute `order_token` via `orders/preview` (which also returns the authoritative price breakdown, payment options and full cancellation schedule), then commit via `orders/create`, which processes payment and returns the durable `order` / `reservation` / `pincode` identifiers synchronously. There is **no client idempotency key on the accommodation commit**, so the chain leans entirely on P3 mechanics: a correlation record whose token rides the request's `accommodation.label` field, probed back out through the `/orders/details` created-window report. Compensation is the Scope-B cancel chain, whose cost is governed by the product's cancellation policy (free-until-deadline / partially-refundable / non-refundable).

**Scope B.** Two independent chains against a landed booking: (1) **modify** — change card details (pay-at-property only), stay dates, or room-level details, one modification type per request, previewable in Beta; the commit mutates the reservation in place and has **no documented undo**, so consent is entry-gated by construction; (2) **cancel** — synchronous for accommodations; fee determined by the per-product cancellation schedule; the free-cancellation deadline is a business-timescale boundary exercised through a fresh-eligibility read + fee-consent entry gate, with the optional `request_property_approval` (cancel-for-less) waiver.

---

## 2. Fit-test verdict (SKILL Step 0)

A chain is the right abstraction for Scope A. The disqualifying conditions all fail:

- **Not a single call**: five provider calls plus two selections before the commit.
- **Not idempotent CRUD**: `orders/create` has no documented idempotency key for accommodations (see elicitation Q4), and inventory expires under you (`order_unavailable`, `documented`).
- **Handles + expiry + commit all present**: `order_token` is a provider-documented 15-minute expiring handle consumed by the commit; product ids die when inventory sells; `orders/create` moves money and creates a reservation.
- **Provider does not run the state machine**: Booking.com gives you no "submit intent, poll status" for creation — creation is synchronous, and *you* own recovery for lost responses (no partner webhook exists; the Notifications API page is an empty stub — `documented`, https://developers.booking.com/demand/docs/additional-services/notifications/about-notifications).
- **Compensation exists** and is a real business action with fees and windows (cancellation policies — `documented`).

Scope B modify is a smaller chain (read → optional preview-read → gated in-place commit) — it clears the fit test on the commit + no-undo + business-blocker properties. Scope B cancel is likewise read → gate → commit-class action with its own confirmation; it is also Scope A's compensation sub-chain, compiled once here.

**Not BLOCKED**: elicitation Q2/Q3 are answerable for every commit (each has a documented observation path), so the compile can finalize — with the UNKNOWNs in §10 flagged for a human before implementation.

---

## 3. Elicitation log (SKILL Step 1) — shared across scopes

Transport facts (apply everywhere): all Demand API v3 endpoints are **POST-only** with JSON bodies; no query parameters accepted (`documented`, https://developers.booking.com/demand/docs/support/error-handling/http-4xx-scenarios). Auth = `Authorization: Bearer <key>` + `X-Affiliate-Id: <aid>` headers on every request (`documented`, https://developers.booking.com/demand/docs/development-guide/authentication). Error envelope = `{request_id, errors[]: {id, message}}`; only the first error is returned per response (`documented`, https://developers.booking.com/demand/docs/support/error-handling/about-errors). Rate limits are per partner account per minute; on 429 all requests from the API user are rejected for ~1 minute, docs prescribe exponential backoff (`documented`, https://developers.booking.com/demand/docs/development-guide/rate-limiting).

### Q1 — double-call test, per step

- `accommodations/search`, `accommodations/details`, `accommodations/availability`, `orders/details`, `orders/details/accommodations`, `orders/modify/preview`: repeating leaves nothing new behind — pure reads. (`documented` behavior shape: these endpoints are described purely as retrieval/validation, e.g. modify/preview "is consultive and does not apply any changes" — https://developers.booking.com/demand/docs/orders-api/3.2/order-modify-preview)
- `orders/preview`: each call returns a fresh `order_token` valid 15 minutes; docs' remedy for expiry is simply "repeat /orders/preview" — repeated calls create replaceable, self-expiring artifacts with no user-visible consequence → **mint** (`documented`, https://developers.booking.com/demand/docs/orders-api/order-preview-create ; https://developers.booking.com/demand/docs/orders-api/orders-faqs). No charge, no hold is placed at preview (`documented`: create "processes payment", preview only "validates" — same pages).
- `orders/create`: two calls with the same inputs ⇒ two paid reservations, as far as the docs show for **accommodations** — the documented duplicate detection (`409 duplicate_request`, "treat as successful, do not retry") is explicitly scoped "**for car rentals only**" (`documented`, https://developers.booking.com/demand/docs/support/error-handling/http-4xx-scenarios#409---conflict). → **commit**, exactly-once required. Whether replaying the *same* `order_token` dedupes server-side is nowhere documented (`assumed` not — see UNKNOWN-1).
- `orders/modify`: mutates an existing reservation in place; a repeated no-op modify returns an error ("Number of guests is already set to the requested value" — `documented`, https://developers.booking.com/demand/docs/orders-api/order-modify). → **commit with `mutates`**.
- `orders/cancel` (accommodation): second call against an already-cancelled reservation returns `409 order_not_cancellable` "The order has already been cancelled" — deterministic, distinguishable, cannot create a second cancellation (`documented`, https://developers.booking.com/demand/docs/orders-api/cancel/error-handling). → **compensate/commit-class, naturally idempotent in effect** (see Q5).

### Q2 — process dies mid-call: what might exist?

- `orders/create`: a paid, user-visible reservation may exist (for `pay_online_now` the card is charged at create — `documented`, https://developers.booking.com/demand/docs/payments/payments-timings). Q3 mandatory. 2b: creates a NEW durable object (order/reservation) — not `mutates`.
- `orders/modify`: the reservation may already be changed (dates modifications repriced — response carries `new_price`; "Once a modification succeeds, the system automatically updates the booking" — `documented`, https://developers.booking.com/demand/docs/orders-api/order-modify). 2b: **MUTATES in place** → content-based confirmation required; **no undo documented** → pre-commit safety gate required (SPEC §2.1 rule).
- `orders/cancel`: the reservation may already be cancelled (synchronous — `documented`, https://developers.booking.com/demand/docs/orders-api/cancel-order: "Accommodation cancellations are processed synchronously. A successful response confirms that the reservation has already been cancelled"). 2b: mutates the reservation to a terminal state; no undo (cannot un-cancel — `documented` by omission + "If you need to adjust dates [on a cancelled reservation], consider creating a new one", https://developers.booking.com/demand/docs/orders-api/order-modify).
- `orders/preview`: at most an orphan `order_token` that self-expires in 15 min — inconsequential (`documented` TTL, `assumed` inconsequence; consistent with "repeat preview" remedy).

### Q3 — how to find out whether an ambiguous attempt landed (the correlation-record question, answered explicitly)

This is the load-bearing answer for the whole compile, so it is spelled out against the docs:

- **Q3.a — Which read endpoint?** `/orders/details` — "Retrieve order information created or updated within a specified time period", supporting two mutually-exclusive filter families: **identifiers** (`orders`, `reservations`, max 100) or **date windows** (`created`, `updated`, `start`, `end`, each ≤ 7 days, 12-month lookback) (`documented`, https://developers.booking.com/demand/docs/orders-api/order-details). If the `orders/create` response was lost, you have **no identifier**, so the probe MUST be the date-window form: `{"created": {"from": <dispatch_t0 - skew>, "to": <now>}, "currency": "...", "services": ["accommodations"]}`.
- **Q3.b — Filtered/matched how?** The `/orders/details` response returns per order: `label` — "Custom partner string for tracking and attribution" / "Label provided when the order was created" (`documented`, https://developers.booking.com/demand/docs/orders-api/order-details and https://developers.booking.com/demand/docs/orders-api/order-details-accommodations). `accommodation.label` on the create request is a free-text partner-defined string with "no enforced format" (`documented`, https://developers.booking.com/demand/docs/orders-api/labels-attributions). **So the correlation record's token is embedded in `accommodation.label` at dispatch time, and the probe matches `label == correlation_token` within the created window.** Secondary match keys (defense against label loss/collision): property id + `checkin`/`checkout` + booker email, all returned by the details endpoints (`documented`, same pages). Note the API offers no server-side *filter by label* — the probe retrieves the window and matches client-side (`documented` filter list is identifiers/dates only).
- **Q3.c — Lag before queryable?** For the date-window report: "Updates may take **up to 2 hours** to become available" (`documented`, https://developers.booking.com/demand/docs/orders-api/order-details). A negative probe is therefore inconclusive for up to ~2h — the reconcile deadline below is sized to this. (Lag on identifier-based `/orders/details/accommodations` immediately after create: not documented — UNKNOWN-7.)
- **Q3.d — Webhook?** None. No partner-facing order webhook exists in the Demand API docs; the Notifications page is a TBD stub (`documented`, https://developers.booking.com/demand/docs/additional-services/notifications/about-notifications). Finality is probe-only.
- **Q3.e — Where does the correlation record live?** In the integrator's own durable store (Postgres), persisted **before** dispatch with `{run_id, correlation_token (=label value), intent snapshot (property, dates, occupancy, booker email hash), status=PENDING}` — survives process death; the sweep (§ policies) re-probes stuck PENDING rows. This is P3 verbatim; the docs give us the label/report machinery to make it work.

For `orders/cancel`: probe = `/orders/details/accommodations` by known `reservation` id; landed signal = `status ∈ {cancelled_by_guest, cancelled, cancelled_by_accommodation}` + `cancellation_details` populated (`documented`, https://developers.booking.com/demand/docs/orders-api/cancel-order). For `orders/modify`: probe = same endpoint; landed signal is **content-based** (new `checkin`/`checkout`, or new `allocation`/`guests` on the `room_reservation`) since the object pre-exists (`documented` response fields, https://developers.booking.com/demand/docs/orders-api/order-details-accommodations).

### Q4 — client idempotency key?

**No.** The exhaustively documented `orders/create` request schemas for v3.1 and v3.2 (field tables + full examples) contain no idempotency/request-key field (`documented`, https://developers.booking.com/demand/docs/orders-api/order-preview-create ; https://developers.booking.com/demand/docs/migration-guide/v3.2/orders/create — the v3.2 breaking-changes table adds no such field). Same for `orders/modify` and `orders/cancel`. → `idempotency.mode: none` on all three; accommodation create gets `attempts: 1` forced. The cars-only `duplicate_request` behavior is recorded in `known_unmatched` as a signal we must NOT rely on for accommodations.

### Q5 — natural idempotency where no key exists?

- `orders/create` (accommodation): **not proven** natural — assume duplicating. (`assumed`; UNKNOWN-1.)
- `orders/cancel` (accommodation): **effectively natural** — a repeat cannot create a second cancellation; it deterministically returns `409 order_not_cancellable` ("already been cancelled"), and the docs *explicitly instruct retrying* 500/504 on cancel ("Retry the request after a short delay" — `documented`, https://developers.booking.com/demand/docs/orders-api/cancel/error-handling). Modeled as `idempotency: natural` with that justification.
- `orders/modify`: not natural (a dates modify re-applied against changed state could re-price differently; no-op detection exists only for some fields) → `none`, `attempts: 1`.

### Q6 — handles returned, exact staleness signals, observed lifetimes

| Handle | Minted by | Staleness signal ON USE | Lifetime |
|---|---|---|---|
| `next_page` pagination token | search/details reads | `400 expired_token` "Token 'page' has expired." / `invalid_token` / `token_endpoint_mismatch` (`documented`, https://developers.booking.com/demand/docs/support/error-handling/error-codes) | 3 hours, endpoint-bound (`documented`, same page) |
| product id (`products[].id`, e.g. `1050736003_405384551_0_42_0_1122794`) | availability (or search `extras:["products"]`) | `409 order_unavailable` — "Order has unavailable items: product '<id>' … is unavailable", raised at **orders/preview and orders/create** (`documented`, https://developers.booking.com/demand/docs/support/error-handling/http-4xx-scenarios#409---conflict) | none documented; "Availability can change very quickly … the `recommendation` field will be `null` … Always handle this state" (`documented`, https://developers.booking.com/demand/docs/accommodations/accommodation-tutorial) |
| `order_token` | orders/preview | error on `orders/create` — "If you try to make an /orders/create call with an expired order_token, it will return an error" (`documented`, https://developers.booking.com/demand/docs/orders-api/orders-faqs) — **exact error id/status not documented** (UNKNOWN-2) | **15 minutes, provider-documented** ("The order_token expires after 15 minutes"; "Repeat /orders/preview if expired" — `documented`, https://developers.booking.com/demand/docs/orders-api/order-preview-create) |
| `order` id (v3.2 `data.order`; v3.1 `accommodation.order`), `reservation`, `pincode` | orders/create | durable — no expiry | lifecycle via `status` enum (`documented`, https://developers.booking.com/demand/docs/orders-api/orders-faqs) |
| `room_reservation` ids | orders/details/accommodations (`products[]`) | durable per reservation lifecycle (`documented`, https://developers.booking.com/demand/docs/orders-api/order-modify — "required for all room-level modifications") | — |

Rare treat: the `order_token` TTL is actually contractual in the docs. Per SPEC P2 it is still recorded as `ttl_hint` only and enforced by detection, but the hint is unusually trustworthy and drives planning ("don't park a consent gate longer than ~12m without planning a re-preview").

### Q7 — cheapest re-mint per handle

`next_page` → re-run the originating search. product id → re-run `accommodations/availability` (same property, dates, guests). `order_token` → re-run `orders/preview` (docs' own prescription). Durable ids → n/a.

### Q8 / Q8b — coupling and selection identity

- Availability products are priced **relative to the request**: property id, `checkin`/`checkout`, `guests` (rooms/adults/children ages), booker `country`+`platform` ("Ensures platform-specific prices and deals are returned correctly"; child ages affect rates; `documented`, https://developers.booking.com/demand/docs/accommodations/search-for-available-properties , https://developers.booking.com/demand/docs/accommodations/occupancy-allocation). The product id itself encodes an occupancy-scoped rate (`documented` shape; opaque to us). ⇒ `derived_from` edges: products ← {selected_property} plus intent {dates, guests, currency, booker}.
- `orders/preview` must be built by **copying** `booker.platform`, `booker.country`, `currency`, accommodation `id`, `checkin/checkout`, `products[].id`, `products[].allocation` from the availability step ("Use the values … from the preceding response" — `documented`, https://developers.booking.com/demand/docs/orders-api/order-preview-create). ⇒ `order_token` ← {selected_product} + same intent.
- `orders/create` `accommodation.products.id` "must match product IDs from /orders/preview. This is a key validation step" (`documented`, same page). ⇒ create ← {order_token, selected_product}.
- **Q8b rematch keys.** Property pick: `accommodation.id` is a stable global property identifier reused across search/details/availability/orders (`documented` across all pages) — rematch key is the id itself. Product pick: product ids are opaque and NOT documented stable across re-search → rematch by intent-level content: `room` id (stable room identifier, `documented`) + `policies.cancellation.type` + `policies.meal_plan.plan` + payment `timings` + `price.total` (tolerance 0 — any price drift must surface, it is money). `on_ambiguous: gate` (money-adjacent).

### Q9 / Q9b — valid empties and prescribed fallbacks

- `accommodations/search` can return an empty `data` array — valid: "A search may return no accommodations if: the property is closed …; search parameters do not match property policies (e.g., children at 'Adults only' properties)" (`documented`, https://developers.booking.com/demand/docs/accommodations/search-for-available-properties). Repairable intent: dates, location, guests.
- `accommodations/availability`: `recommendation: null` when the matching product sold out between calls — "Always handle this state" (`documented`, https://developers.booking.com/demand/docs/accommodations/accommodation-tutorial); `products` can be empty for the dates/occupancy. Routes: different property (reselect) or different dates (repair).
- `/orders/details` probe: "**Empty 200 OK response** — No orders match the request filters" is an expected outcome, not an error (`documented`, https://developers.booking.com/demand/docs/orders-api/order-details) — this is what a not-landed probe looks like.
- Q9b: no provider-prescribed deterministic fallback ("retry without filters") is documented for accommodations → no `auto_repairs` compiled. (`documented` by absence.)

### Q10 — documented failure taxonomy (consolidated; each code lands in a domain row or `known_unmatched`)

- Request-shape 400s: `missing_parameter`, `invalid_parameter`, `conflicting_parameters`, `malformed_request`, `unknown_parameter`, `invalid_request` (incl. the adults-only-with-children preview reject), `invalid_token`/`expired_token`/`token_endpoint_mismatch` (page tokens) (`documented`, https://developers.booking.com/demand/docs/support/error-handling/error-codes).
- 401/403/404/405/406/415 transport-adjacent client errors (`documented`, https://developers.booking.com/demand/docs/support/error-handling/http-4xx-scenarios). 403 on cancel has a special documented nuance: "If the reservation was created recently, wait a few minutes before retrying" (`documented`, https://developers.booking.com/demand/docs/orders-api/cancel/error-handling).
- 409: `order_unavailable` (two documented meanings: cancel-ineligible state, and product-no-longer-available during preview/create), `order_not_cancellable`, `duplicate_request` (cars only) (`documented`, http-4xx page + cancel error page).
- 422 payment family (all at create): `payment_refused` (generic), `payment_refused_book_process_failed` (PSP/fraud/`email_blocklist`), `payment_refused_insufficient_funds` ("Subsequent attempts at a later time may succeed"), `payment_refused_budget_overflow`, `payment_refused_card_is_expired`, `payment_refused_card_lost`, `payment_refused_invalid_card_number`, `payment_refused_invalid_amount` (authorised vs captured mismatch — "rounding issues or configuration bugs; a mismatch between the price shown in orders/preview and the final price"), `payment_refused_invalid_payment_method`, `payment_refused_issuer_cvc_check_failed`, `payment_refused_withdrawal_amount_exceeded`, `payment_refused_withdrawal_count_exceeded`, `payment_refused_psp_card_is_blocked`, `payment_refused_sca_required` (`documented`, https://developers.booking.com/demand/docs/support/error-handling/payment-errors).
- 429 rate limit (account-wide, ~1 min lockout) (`documented`, rate-limiting page). 5xx/504: docs prescribe retry with backoff for reads and — notably — for cancel (`documented`, about-errors + cancel error page).
- Modify errors: `incorrect_parameters` with message-discriminated variants — "Hotel does not accept card type", "Reservation not found", "Reservation is cancelled", "No valid change parameter specified", "Room reservation not found", "Cannot change number of guests when rate-level occupancy is in effect", "Number of guests exceeds the allowed limit", "Invalid number of guests specified", "Number of guests is already set to the requested value", "Cannot change smoking preference…", "Invalid smoking preference value" (`documented`, https://developers.booking.com/demand/docs/orders-api/order-modify). Plus the FAQ-documented `"status": "failed"` response when modifying prepaid online-payment orders (`documented`, https://developers.booking.com/demand/docs/orders-api/orders-faqs).

### Q11 — human-decision events and payload-level business blockers

- **Money consent before create** (total price, payment timing + instalment `dates` schedule, cancellation schedule incl. non-refundable). Docs require the partner to surface exactly this before confirming ("Key things to check: price, payment policy and cancellation policy" — `documented`, orders-faqs; FTC/DSA compliance pages require full price display — `documented`, https://developers.booking.com/demand/docs/compliance/ftc-compliance). → `entry_gate` on the commit.
- `occupancy_mismatch` in the preview payload — call succeeded, business blocker: "returned when the requested guest allocation exceeds the product capacity … allows you to review and adjust the booking before confirming"; `null` when fine (`documented`, https://developers.booking.com/demand/docs/accommodations/occupancy-allocation). → precondition → gate.
- `booker_address_required=true` from `accommodations/details` makes `booker.address` mandatory at create (`documented`, order-preview-create) → completeness precondition on the details read feeding intent validation.
- Cancel fee consent when past the free window: fee amounts from `cancellation_details` / `policies.cancellation` schedule; "Display any applicable cancellation fee before the traveller confirms"; "Refresh cancellation information immediately before submitting the cancellation request, as eligibility and fees may change over time" (`documented`, https://developers.booking.com/demand/docs/orders-api/cancellation-policies). → fresh read + entry gate.
- Modify consent: dates modify re-prices ("Modifying dates may change the total price"; preview returns `price.current`/`price.new` — `documented`, order-modify + order-modify-preview). → entry gate on modify.
- SCA: 3DS happens **on the partner's platform before create** (partner collects `authentication_value`/`eci`/`transaction` from their PSP and sends them IN the create request — `documented`, https://developers.booking.com/demand/docs/payments/models/booking-collects). So the SCA gate in this chain is "go run the challenge with the traveller, then repair the payment intent fields" — not a provider-driven mid-chain redirect.

### Q12 / Q13 — compensation

- Undo of a landed create = the cancel chain. Cost varies with time per the per-product cancellation schedule: free until `free_cancellation_until` (flexible), always-fee (partially_refundable / `special_conditions` in v3.1), full-fee-from-`now` (non_refundable) (`documented`, https://developers.booking.com/demand/docs/orders-api/cancellation-policies). Multi-room orders can carry **different policies per product** but cancel is per-reservation (`documented`, cancellation-policies + cancel-order: `accommodation.reservation` is the cancel unit; orders-faqs: multiroom returns a single order id) — see UNKNOWN-13.
- Accommodation cancel is synchronous — the compensator confirms in-band, with a probe declared anyway (Q3).
- Rebook flows ("I would like to book another property instead" is the docs' own example reason — `documented`, cancel-for-less page): **commit the replacement before compensating the original** (Q13 default yes).
- Modify has **no compensator** (nothing documented un-does a modification; the docs' remedy for unwanted state is cancel + rebook — `documented`, order-modify "If the price is not acceptable, cancel and create a new order"). → `compensation: none` + entry gate, per SPEC rule.

### Q14 — latency

Not documented for any endpoint (UNKNOWN-6). Timeouts below are set from generic API behavior (`assumed`), with the commit given the most headroom per SKILL Step 5.

### Q15 — finality after the final commit

`orders/create` success is synchronous and final for booking existence (status `booked`); cancellation by Booking.com can still occur later "for operational or risk-related reasons, such as suspected fraud or an invalid payment card" (`documented`, https://developers.booking.com/demand/docs/orders-api/cancel-order) — i.e. the booking is **landed-but-revocable at business timescale**; the `updated`-window sync job (30–60 min rolling windows, `documented`, order-details) is the watch mechanism, outside this chain's wall clock. Reporting visibility lags ≤ 2h (Q3.c). Hand-off to a human: reconcile deadline 6h → operator (below).

---

## 4. Scope A — Stay booking chain

### 4.1 Handle graph

```yaml
handles:
  search_results:                    # code-held results collection (no provider token);
    minted_by: search                # exists so selections have lineage (I3)
    derived_from: []
    staleness: {detect: [], ttl_hint: unknown, refresh: search}
    # UNKNOWN: staleness undetectable directly — data-only collection; product-level
    # death surfaces downstream (order_unavailable). Fail-closed per SKILL Step 2.

  selected_property:                 # pseudo-handle (I3): the property pick
    minted_by: select_property
    derived_from: [search_results]
    rematch: {key: [accommodation_id],   # stable global property id (documented)
              on_ambiguous: model}
    staleness: {detect: [], ttl_hint: n/a, refresh: select_property}

  availability_products:             # live rate offers for property+dates+occupancy
    minted_by: check_availability
    derived_from: [selected_property]    # + intent: checkin, checkout, guests,
    staleness:                            #   currency, booker.country, booker.platform (I6)
      detect: []                     # no direct signal; consumption signal is
                                     # 409 order_unavailable at preview/create (documented)
      ttl_hint: unknown              # "availability can change very quickly" (documented)
      refresh: check_availability

  selected_product:                  # pseudo-handle (I3): the room/rate pick(s)
    minted_by: select_product
    derived_from: [availability_products]
    rematch:
      key: [room_id, cancellation_policy_type, meal_plan, payment_timings, total_price]
      on_ambiguous: gate             # price differences are money → user re-confirms
    staleness: {detect: [], ttl_hint: unknown, refresh: select_product}

  order_token:
    minted_by: preview_order
    derived_from: [selected_product]
    single_use: true                 # ASSUMED (UNKNOWN-1): reuse semantics undocumented;
                                     # spent on any create attempt per I4 — always
                                     # re-preview after an attempt of unknown outcome
    staleness:
      detect: [create_error_naming_order_token]   # exact id UNDOCUMENTED (UNKNOWN-2);
                                     # classifier matches error message referencing the
                                     # token, else falls to default rows (fail closed)
      ttl_hint: 15m                  # PROVIDER-DOCUMENTED; planning only (P2)
      refresh: preview_order

  order_id:                          # data.order (v3.2) / accommodation.order (v3.1)
    minted_by: create_order          # + reservation_id + pincode, durable
    derived_from: [order_token]
    staleness: {detect: [], ttl_hint: n/a, refresh: n/a}
```

```
search ──▶ search_results ──▶ (select_property) ──▶ selected_property
                                                        │
                              intent: dates,guests ─────┤
                                                        ▼
                                    check_availability ──▶ availability_products
                                                              │
                                    (select_product) ─────────┴─▶ selected_product
                                                                      │
                                                   preview_order ─────┴─▶ order_token
                                                                              │
                                                        create_order ─────────┴─▶ order_id
                                                                                   (durable)
```

`accommodations/details` (get_details) hangs off `selected_property` as a read that feeds intent validation (`booker_address_required`, adults-only, bed configurations, `refuses_free_cancellation_requests`); it mints no handle.

### 4.2 Actions

```yaml
actions:
  - id: search
    description: Find properties with ≥1 available product for location+dates+occupancy.
    effect: read                     # double-call test: nothing new exists after (Q1)
    input:
      intent: {location: {type: object, required: true},   # exactly one of city/region/
                                                           # district/airport/landmark/coords
               checkin: {type: date, required: true},
               checkout: {type: date, required: true},
               guests: {type: object, required: true},     # number_of_adults/rooms,
                                                           # children ages, allocation
               booker_country: {type: string, required: true},
               booker_platform: {type: enum[desktop,mobile,tablet], required: true},
               currency: {type: string, required: false}}
    output:
      payload: accommodations[]      # id, price, products[] (best rate), urls
      handles: [search_results]
      empty:
        valid: true
        detect: "data.length == 0"
        route: repair(to: search, fields: [location, checkin, checkout, guests])
    idempotency: {mode: natural}
    timeout: 20s

  - id: select_property              # pseudo-step, no dispatch (I3)
    effect: select
    input: {handles: [search_results]}
    output: {handles: [selected_property]}

  - id: get_details
    description: Property content + booking prerequisites for the picked property.
    effect: read
    input: {handles: [selected_property],
            intent: {languages: {type: object, required: false}}}
    output: {payload: property_details}   # booker_address_required, adults-only flag,
                                          # rooms/bed_options, accepted pay-at-property
                                          # methods, refuses_free_cancellation_requests
    preconditions:
      - when: "property is adults-only AND intent.guests includes children"
        verdict: reselect(to: select_property)
        reason: "documented preview reject would follow; fail early (error-codes page)"
      - when: "booker_address_required == true AND booker address not collectable"
        verdict: gate(booker_details)
        reason: "address becomes mandatory at create (order-preview-create)"
    idempotency: {mode: natural}
    timeout: 20s

  - id: check_availability
    description: Live products (room+rate offers) for property+dates+occupancy.
    effect: read
    input: {handles: [selected_property],
            intent: {checkin: {type: date, required: true},
                     checkout: {type: date, required: true},
                     guests: {type: object, required: true},
                     currency: {type: string, required: false},
                     booker_country: {type: string, required: true},
                     booker_platform: {type: string, required: true}}}
    output:
      payload: {products: product[],           # id, room, price, policies.cancellation
                recommendation: allocation?}   # (type, free_cancellation_until, schedule),
                                               # payment timings, maximum_occupancy,
                                               # number_available_at_this_price
      handles: [availability_products]
      empty:
        valid: true
        detect: "products.length == 0 || recommendation == null"
        # "recommendation will be null … Always handle this state" (documented, tutorial)
        route: reselect(to: select_property)   # or the planner repairs dates as a new
                                               # proposal; reselect is the in-chain route
    idempotency: {mode: natural}
    timeout: 20s

  - id: select_product               # pseudo-step, no dispatch (I3)
    effect: select
    input: {handles: [availability_products]}
    output: {handles: [selected_product]}      # may carry multiple products (multi-room);
                                               # all products share one lineage

  - id: preview_order
    description: Validate order, get authoritative price/policies, mint order_token.
    effect: mint                     # repeatable; old tokens self-expire (Q1, documented)
    input:
      handles: [selected_product]
      intent: {allocation: {type: object, required: true},   # per-product adults/children
               currency: {type: string, required: true},
               booker_country: {type: string, required: true},
               booker_platform: {type: string, required: true}}
    output:
      payload: {price_breakdown: object,               # products[].price (total/base/charges)
                cancellation_schedule: object,         # products[].policies.cancellation —
                                                       # AUTHORITATIVE pre-booking (documented)
                payment_options: object,               # general_policies.payment: timings,
                                                       # methods.cards, instalment dates[] —
                                                       # intersection across products
                occupancy_mismatch: object|null}
      handles: [order_token]
    preconditions:
      - when: "occupancy_mismatch != null"
        verdict: gate(occupancy_review)
        reason: "guests exceed product capacity; adjust before commit (documented)"
      - when: "general_policies.payment lacks any timing the product intent requires"
        verdict: gate(payment_terms_review)
        reason: "preview returns only timings supported by ALL selected products (documented)"
    idempotency: {mode: natural}     # re-preview mints a fresh token; no dedup needed
    timeout: 30s
    latency_hint: unknown

  - id: create_order                 # THE COMMIT
    description: Confirm booking and process payment with the order_token.
    effect: commit                   # durable paid reservation (Q1/Q2)
    entry_gate: booking_consent      # re-fires on EVERY arrival incl. post-rewind replays:
                                     # a re-preview may carry a new price → new consent
    input:
      handles: [order_token, selected_product]   # products[].id echoed by code, must
                                                 # match preview (documented validation)
      intent: {booker_contact: {type: object, required: true},  # email, name, telephone,
                                                                # address iff required,
                                                                # company?, language?
               guests: {type: object, required: true},          # per product: name+email
               payment: {type: object, required: true},         # method, timing, card{...},
                                                                # card.authentication
                                                                # (3DS/riskified/exemption)
               bed_configuration: {type: string, required: false},
               remarks: {type: object, required: false}}
    request_invariants:
      # correlation token: code writes accommodation.label = correlation record token
      # before EVERY dispatch (P3). Not model-visible, not intent, not a handle.
      accommodation.label: "<correlation_token(run_id)>"
    output:
      payload: {order: string, reservation: string, pincode: string,
                receipt_url: string?, authorisation_form_url: string?}
      handles: [order_id]
    idempotency: {mode: none}        # Q4: no key documented for accommodations →
                                     # attempts: 1 FORCED; P3 mechanics mandatory:
                                     # correlation record persisted (integrator Postgres)
                                     # BEFORE dispatch + cancellation shield on the
                                     # in-flight attempt
    confirmation:
      probe: find_order_by_label     # /orders/details created-window + label match (Q3)
      signal: "an accommodations order exists in the created window with
               label == correlation_token AND property/checkin/booker-email match"
      async: {channel: poll, deadline: 6h}   # report lag ≤2h documented; 3× headroom,
                                             # then escalate(operator)
      sweep: {interval: 30m, escalate_after: 24h}
    compensation:
      chain: [read_cancel_eligibility, cancel_order]   # Scope B cancel chain (§5.2)
      window: "per product policies.cancellation schedule: free until
               free_cancellation_until (flexible); fee always applies
               (partially_refundable); full fee from booking time (non_refundable)"
      ordering_note: "for rebook/replace flows, commit the replacement booking BEFORE
        cancelling the original (Q13). For non_refundable products compensation exists
        mechanically but refunds nothing — economically compensation:none; the
        pre-commit protection is booking_consent, whose payload MUST show the
        non-refundable schedule (SPEC §2.1 no-undo rule, applied at the economic level)."
    timeout: 90s                     # ASSUMED (UNKNOWN-6); generous per SKILL Step 5 —
                                     # a tight timeout manufactures reconciles
    latency_hint: unknown

  - id: find_order_by_label          # the confirmation probe
    description: Created-window order report matched by correlation label.
    effect: read
    input: {intent: {created_from: {type: date, required: true},
                     created_to: {type: date, required: true},
                     currency: {type: string, required: true}}}
    output:
      payload: orders[]              # id, label, status, accommodations.reservation,
                                     # booker, price, created/updated
      empty: {valid: true, detect: "data.length == 0",
              route: ok}             # "no match yet" is an answer; reconcile loop
                                     # interprets it (≤2h lag) — never an error
    idempotency: {mode: natural}
    timeout: 30s

  - id: get_order_details_accom      # shared read; also Scope B's first step
    description: Accommodation order details by order/reservation id.
    effect: read
    input: {handles: [order_id],
            intent: {extras: {type: object, required: false}}}   # policies, extra_charges,
                                                                 # accommodation_details
    output:
      payload: {status: enum[booked, cancelled, cancelled_by_accommodation,
                             cancelled_by_guest, stayed, no_show],
                cancellation_details: object|null,   # at, fee (null fee = free)
                products: product_reservation[],     # room_reservation ids, per-product
                                                     # status + policies.cancellation
                checkin: date, checkout: date, label: string, pin_code: string}
    idempotency: {mode: natural}
    timeout: 20s
```

### 4.3 Verdict table — Scope A domain rows (before SPEC §3 generic skeleton)

Ordered; first match wins. Payload/staleness rows precede transport success per SPEC §3.

| # | Signal | Step scope | Verdict | Evidence |
|---|---|---|---|---|
| A1 | create-time error whose `errors[].message` names the `order_token` (expired/invalid) | create_order | `rewind(to: preview_order)` — re-mint token; entry gate re-consents on replay | `documented` expiry + "repeat preview" remedy (order-preview-create, orders-faqs); exact id UNKNOWN-2 → anything not matching this predicate falls to defaults |
| A2 | `409` + `order_unavailable` "Order has unavailable items: product '<id>'" | preview_order | `reselect(to: select_product)` — the pick is dead; source collection may live; failed option excluded (I3/C3.5). Config note: an implementation MAY rewind to `check_availability` first for fresh data (docs recommend re-checking availability) — it must still exclude the failed option | `documented`, http-4xx-scenarios §409 scenario 2 |
| A3 | `409` + `order_unavailable` (same, at commit time) | create_order | `reselect(to: select_product)` — definitive rejection, no booking created ("product … for this order is unavailable **at the time of the request**"); order_token spent (I4), re-preview happens on replay path | `documented`, same page. Deterministic-response reject → routes via table; the single uncertain-dispatch attempt was not spent on ambiguity |
| A4 | `422 payment_refused_sca_required` | create_order | `gate(sca_challenge)` → traveller completes 3DS on partner platform → `repair(to: create_order, fields: [payment])` with fresh `card.authentication.3d_secure` values | `documented`, payment-errors; 3DS data collected partner-side (booking-collects) |
| A5 | `422` ∈ {`payment_refused_insufficient_funds`, `payment_refused_card_is_expired`, `payment_refused_card_lost`, `payment_refused_invalid_card_number`, `payment_refused_issuer_cvc_check_failed`, `payment_refused_invalid_payment_method`, `payment_refused_psp_card_is_blocked`, `payment_refused_budget_overflow`, `payment_refused_withdrawal_amount_exceeded`, `payment_refused_withdrawal_count_exceeded`, `payment_refused` (generic)} | create_order | `repair(to: create_order, fields: [payment])` — the domain's valid-empty ("card declined") arriving on an error transport; user supplies another instrument. ASSUMPTION: a 422 means no reservation was created (UNKNOWN-4; the confirmation probe is the backstop) | `documented` codes + per-code remediation, payment-errors |
| A6 | `422 payment_refused_book_process_failed` (PSP fraud / email_blocklist) | create_order | `dead_end(reason: psp_fraud_reject, permanent: false, retry_after_hint: hours)` — docs: "wait before retrying"; identity-swap recovery is a planner/human decision, never an in-run repair loop | `documented`, payment-errors |
| A7 | `422 payment_refused_invalid_amount` | create_order | `dead_end(reason: amount_mismatch_config_bug, permanent: true)` — "rounding issues or configuration bugs"; surface loudly, do not retry | `documented`, payment-errors |
| A8 | `400 invalid_request` "Children are not allowed in this 'Adults only' property" | preview_order | `reselect(to: select_property)` | `documented`, error-codes |
| A9 | payload precondition: `occupancy_mismatch != null` on 2xx preview | preview_order | `gate(occupancy_review)` | `documented`, occupancy-allocation |
| A10 | `400` ∈ {`expired_token`, `invalid_token`, `token_endpoint_mismatch`} on a `page` token | search / find_order_by_label pagination | `rewind(to: <originating read>)` — re-run to mint a fresh `next_page` | `documented`, error-codes (3h TTL, endpoint-bound) |
| A11 | `429` | search, get_details, check_availability, preview_order, probes | `retry` (backoff per policy; docs prescribe exponential backoff) | `documented`, rate-limiting + http-4xx |
| A12 | `429` | create_order | `reconcile` — NOT declared `rejected_before_execution`: the account-gate reading of the rate limiter suggests pre-execution rejection, but the docs never state "no booking was created" (UNKNOWN-5). Fail safe | SPEC §3 row 5 default; `documented` 429 semantics insufficient for the flag |
| A13 | timeout / conn reset / `5xx` / `504` | search, get_details, check_availability, preview_order, probes | `retry` | `documented` retry guidance, about-errors |
| A14 | timeout / conn drop / `5xx` / ambiguous transport | create_order | `reconcile` — run `find_order_by_label`; landed → ok (order id recovered from probe payload into the correlation record); not-landed after lag window → `rewind(to: preview_order)` (token spent, I4) if budget, else dead_end | P3; probe machinery `documented` (Q3) |
| A15 | `401` / `403` | any | `dead_end(reason: auth_or_permission, permanent: true)` — credentials/agreement problem; operator fixes config | `documented`, http-4xx |
| — | …then generic rows 1–12 from SPEC §3 | | | |

**known_unmatched** (consciously left to the fixed default rows — reconcile if a commit is in flight, else dead_end):
`conflicting_parameters`, `invalid_parameter`, `missing_parameter`, `unknown_parameter`, `malformed_request`, `unsupported_media_type`, `404` endpoint errors, `405`, `406` — all integration/config bugs; a well-tested client should never emit them, and routing them anywhere softer would mask defects. `409 duplicate_request` — **documented for car rentals only**; deliberately NOT compiled as an accommodation dedup signal (if it ever appears on an accommodation create it hits row 11 → `reconcile`, and the probe resolves it — which is exactly the right behavior if Booking.com silently extends dedup to accommodations). Any create-time 400 that does NOT name the order_token (A1's predicate) also falls through — fail closed.

### 4.4 Gates — Scope A

```yaml
gates:
  booking_consent:                   # entry gate on create_order; re-fires on every
    audience: user                   # arrival — incl. replays after A1/A2/A3/A14 paths
    payload: {total_price: money, price_breakdown: object,
              payment_timing: enum, payment_schedule_dates: object,   # instalments
              cancellation_schedule: object,                         # per product; MUST
              non_refundable_warning: bool}                          # flag type=non_refundable
    outcomes:
      approve: ok                    # proceed to dispatch
      decline: dead_end              # reason: user_declined, permanent for this run
    timeout: {after: 12m, verdict: dead_end}
    # 12m < the documented 15m order_token ttl_hint: a consent parked longer would
    # dispatch against a near-dead token. Planning use of ttl_hint only (P2) — if the
    # user approves at 12m+network and the token IS dead, row A1 rewinds to preview
    # and this gate re-fires with the fresh price. The system stays correct either way.

  occupancy_review:
    audience: user
    payload: {allocated: object, unallocated: object, maximum_occupancy: object}
    outcomes:
      reallocate: {params: {allocation: object}, bind: [allocation],
                   verdict: repair(to: preview_order, fields: [allocation])}
      choose_other_product: reselect(to: select_product)
      cancel: dead_end
    timeout: {after: 30m, verdict: dead_end}

  payment_terms_review:              # preview returned no acceptable timing/method
    audience: user
    payload: {available_timings: object, available_card_types: object}
    outcomes:
      accept_available: {params: {payment: object}, bind: [payment], verdict: ok}
      choose_other_product: reselect(to: select_product)
      cancel: dead_end
    timeout: {after: 30m, verdict: dead_end}

  sca_challenge:                     # raised by row A4; challenge runs on partner platform
    audience: user
    payload: {amount: money, instrument_hint: string}
    outcomes:
      completed: {params: {payment: object},        # fresh 3d_secure values from PSP
                  bind: [payment],
                  verdict: repair(to: create_order, fields: [payment])}
      abandoned: dead_end
    timeout: {after: 15m, verdict: dead_end}

  booker_details:                    # raised by get_details precondition
    audience: user
    payload: {required_fields: object}               # address subfields all-mandatory
    outcomes:
      provided: {params: {booker_contact: object}, bind: [booker_contact], verdict: ok}
      cancel: dead_end
    timeout: {after: 30m, verdict: dead_end}
```

### 4.5 Policy — Scope A

```yaml
policy:
  per_step:
    read:  {attempts: 3, backoff: {base: 500ms, factor: 2, jitter: full, max: 8s}}
    mint:  {attempts: 2, backoff: {base: 1s, factor: 2, jitter: full, max: 10s}}
    commit:
      unkeyed: {attempts: 1,                 # forced: idempotency none (Q4)
                safe_reject_redispatch: 0}   # NOTHING is declared
                                             # rejected_before_execution: the docs never
                                             # certify "no booking/charge was created"
                                             # for any create-time signal (UNKNOWN-5).
                                             # Definitive-response rejects (A2..A7) route
                                             # via the table, which is unaffected by this.
    create_order: {timeout: 90s}             # assumed; slowest step, headroom on purpose
    find_order_by_label: {attempts: 3}       # probe follows read policy; reconcile loop
                                             # spaces re-probes via sweep, not attempts
  per_chain:
    max_rewinds: 3
    max_repairs: 2
    compensation_order: reverse              # single commit here; stated for form
    wall_clock: 20m                          # ≈ max(3× Σ latency_hints(unknown→est 3m),
                                             # 2× 90s slowest timeout) rounded up for two
                                             # gate round-trips' processing (parking time
                                             # excluded per SPEC §2.4); reconcile past the
                                             # first probe cycle parks state=reconciling
                                             # (long-lived, §2.5) governed by
                                             # async.deadline/sweep, not wall_clock
    gate_timeout: 30m
  escalation:
    - {audience: model,    may: [propose repair fields (dates, location, guests,
                                 allocation, payment hint), choose among returned options]}
    - {audience: user,     may: [answer booking_consent/occupancy/payment/SCA gates,
                                 approve repairs with cost]}
    - {audience: operator, may: [resolve reconcile deadline, unstick PENDING
                                 correlation records past sweep escalate_after]}
```

### 4.6 Invalidation walkthroughs — Scope A (SPEC §4 by hand)

**Walkthrough A-1: consent parked past token life (the everyday killer).**
User reaches `create_order`; `booking_consent` parks the run (state=at_gate; wall clock excluded). They approve after 14 minutes; dispatch hits a dead `order_token` → row A1 → `rewind(to: preview_order)` (budget 3→2). Invalidation per I1: `order_token` is re-minted, so the only descendants — the dead token itself and its cached preview payload (price breakdown, payment options) — are replaced (I2: the old preview payload is not mergeable with the new; collections keyed by token lineage). `selected_product`, `availability_products`, `selected_property`, `search_results` all survive: none is downstream of `order_token` (edges point the other way). Replay: preview re-runs (possibly returning a NEW price), cursor re-arrives at `create_order`, `entry_gate` **re-fires by construction** and shows the fresh price — stale consent cannot carry over (SPEC §2.1 entry_gate; C8.4 shape). If preview now throws `order_unavailable` → A2 → reselect. Nothing about this path touches the model's classification rights.

**Walkthrough A-2: product sells out at the commit.**
`create_order` dispatch returns `409 order_unavailable` naming the product → row A3 → `reselect(to: select_product)` (budget 3→2). Per I3/I1: the `selected_product` pseudo-handle is invalidated with descendants — `order_token` dies (also already spent per I4: it was consumed by a commit attempt, outcome known-rejected), preview payload dies. The failed product id is recorded **excluded** (C3.5) so neither rematch nor a fresh pick can re-choose it. `availability_products` survives (it is the selection's *source*, not descendant) — the fresh pick is collected from it; a config that prefers fresh inventory data may instead compile A3 as `rewind(to: check_availability)`+exclusion, which additionally replaces `availability_products` (I2: the products collection is replaced, never appended — two lineages of rate offers must never co-exist, their prices answer different questions). Replay: select_product (fresh pick, gate `on_ambiguous` if prices moved) → preview (new token) → create entry gate re-consents. Two rewind-budget units and zero double-charge exposure.

**Walkthrough A-3 (repair scope check): user changes dates mid-chain.**
Planner proposes `repair(to: check_availability, fields: [checkin, checkout])`. Per I6: handles minted by the target step and downstream die — `availability_products`, `selected_product`, `order_token`. `selected_property` survives (its minting inputs — search results + the pick — did not include the repaired fields... `search_results` technically derives from the original dates at *search* level; the config keeps `selected_property`'s rematch key = accommodation_id, which is date-independent, so the property pick legitimately survives; if the property has nothing for the new dates, `check_availability`'s empty route sends us back to `select_property` anyway). This is the boundary between "re-shop the same hotel" and "re-shop the world", and the `derived_from` edges encode it.

---

## 5. Scope B — Order modify + cancel chains

Both chains consume `order_id` / `reservation_id` (and for room modifications `room_reservation` ids) minted by Scope A — see §6 cross-chain notes.

### 5.1 Modify chain

Steps: `get_order_details_accom → preview_modification (Beta, optional but compiled in) → modify_order`.

**Constraint matrix** (`documented`, https://developers.booking.com/demand/docs/orders-api/order-modify): one modification type per request (card | dates | room); card changes supported only for pay-at-property orders; date changes supported for pay-at-property (stable) and online payments (Beta); room-level changes supported for both. Prepaid online orders may return `"status": "failed"` on modify — booking remains valid; the documented remedy is cancel+rebook (`documented`, orders-faqs). "Allow at least 5 minutes between modification requests to the same order" (`documented`, order-modify) — compiled as a deployment-level per-object pacing rule (cross-run serialization on a shared durable object is a SPEC §8 non-goal; noted, not modeled).

#### Handles

```yaml
handles:
  order_id:            {minted_by: external(scope A), derived_from: [], staleness: {detect: [], ttl_hint: n/a, refresh: n/a}}
  room_reservation_id: # per-room ids needed for type=room changes
    minted_by: get_order_details_accom
    derived_from: [order_id]
    staleness: {detect: [], ttl_hint: n/a, refresh: get_order_details_accom}
  modify_preview_result:             # cached eligibility+price payload; NOT a provider token
    minted_by: preview_modification
    derived_from: [order_id]         # + intent: the proposed change (I6)
    staleness: {detect: [], ttl_hint: unknown, refresh: preview_modification}
    # eligibility/fees "may change over time" (documented) — no signal, so the entry
    # gate + fresh-read discipline below carries the risk, not a TTL
```

No selection pseudo-steps: which room to modify is a validated user choice from the details payload → it enters intent through the `modification_review` gate's params (the sanctioned gate-params path, SPEC §2.6), not as a selection pseudo-handle — nothing downstream *derives* from it except the single commit.

#### Actions

```yaml
actions:
  - id: get_order_details_accom      # as defined in §4.2; preconditions added for this chain
    preconditions:
      - {when: "status != 'booked'",
         verdict: dead_end,          # reason: not_modifiable_state, permanent: true —
                                     # "Reservation is cancelled" is deterministic (documented)
         reason: "only active reservations are modifiable"}
      - {when: "modification.type == 'card' AND payment.timing != 'pay_at_the_property'",
         verdict: dead_end,          # documented support matrix
         reason: "card modification unsupported for online payments"}

  - id: preview_modification         # Beta /orders/modify/preview
    description: Consultive check; applies nothing (documented).
    effect: read
    input: {handles: [order_id],
            intent: {modification: {type: object, required: true}}}  # type + change payload
    output:
      payload: {modifiable: bool, reason: string?,
                price: {current: money, new: money}?}   # price only for date changes
    preconditions:
      - {when: "modifiable == false",
         verdict: dead_end,          # reason from payload, permanent: true for this intent;
         reason: "provider states the modification cannot be applied"}   # planner may
                                     # propose different intent as a NEW run
    idempotency: {mode: natural}
    timeout: 20s
    # v3.1 deployments without Beta access skip this step; modify_consent then warns the
    # user that the exact new price is unknown pre-commit (documented gap — UNKNOWN-9b)

  - id: modify_order                 # THE COMMIT (in place)
    description: Apply one modification (card | dates | room) to the reservation.
    effect: commit
    mutates: order_id                # modifies the existing durable object; output.handles
                                     # empty → confirmation MUST be content-based
    entry_gate: modify_consent       # re-fires on every arrival; dates changes are money
    input:
      handles: [order_id]            # + room_reservation echoed by code for type=room
      intent: {modification: {type: object, required: true}}
        # type=card  → change{number, cvc, cardholder, expiry_date}
        # type=dates → change{checkin, checkout}
        # type=room  → change{room_reservation(from gate params), allocation, guests,
        #                     smoking_preference}
    output: {payload: {status: enum[successful, failed], new_price: money?}}
    idempotency: {mode: none}        # no key documented; not naturally idempotent
                                     # → attempts: 1; correlation record persisted before
                                     # dispatch (order_id + intended change + PENDING) +
                                     # cancellation shield
    confirmation:                    # content-based (mutates): existence proves nothing
      probe: get_order_details_accom
      signal: "details reflect the intended change: checkin/checkout equal the requested
               dates (type=dates), or the room_reservation's allocation/guests/smoking
               preference equal the request (type=room). type=card: NOT observable in any
               documented read → UNKNOWN-9c; rely on the synchronous 'successful' status,
               and treat a lost response as reconcile→operator (no probe signal exists)"
      async: {channel: poll, deadline: 2h}
      sweep: {interval: 15m, escalate_after: 12h}
    compensation:
      action: none
      ordering_note: "No documented undo. Pre-commit protection = modify_consent entry
        gate (unskippable, re-fires on replays) + preview_modification eligibility/price
        check. A 'reverse modify' would be a NEW business action with fresh availability
        risk — the docs' own remedy for unwanted outcomes is cancel + rebook."
    timeout: 60s                     # assumed (UNKNOWN-6)
```

#### Verdict table — modify domain rows

All `incorrect_parameters` variants are message-discriminated (the API reuses one error id — the classifier keys on documented message shapes; anything off-pattern falls to defaults):

| # | Signal | Step scope | Verdict | Evidence |
|---|---|---|---|---|
| M1 | `incorrect_parameters` "Reservation not found" / "Room reservation not found" | modify_order | `dead_end(reason: bad_reference, permanent: true)` — re-read details before any new run | `documented`, order-modify |
| M2 | `incorrect_parameters` "Reservation is cancelled" | modify_order | `dead_end(reason: not_modifiable_state, permanent: true)` | `documented` |
| M3 | `incorrect_parameters` "Hotel does not accept card type …" | modify_order (type=card) | `repair(to: modify_order, fields: [modification])` — user picks a card type from `methods.cards` (preview) / accepted set | `documented` |
| M4 | `incorrect_parameters` "Cannot change number of guests when rate-level occupancy is in effect" | modify_order (type=room) | `dead_end(reason: rate_locked_occupancy, permanent: true)` — docs: cancel + rebook | `documented` |
| M5 | `incorrect_parameters` "Number of guests exceeds the allowed limit" / "Invalid number of guests specified" | modify_order (type=room) | `repair(to: modify_order, fields: [modification])` — fix allocation within `maximum_occupancy` | `documented` |
| M6 | `incorrect_parameters` "Number of guests is already set to the requested value" | modify_order | `ok` — desired state already holds; content-based confirmation verifies; traced as degraded-noop | `documented` (docs: avoid no-op submissions) |
| M7 | `incorrect_parameters` "Cannot change smoking preference…" / "Invalid smoking preference value" | modify_order (type=room) | `repair(to: modify_order, fields: [modification])` or `dead_end` if room policy fixed — first occurrence repairs, second dead_ends (budget-capped anyway) | `documented` |
| M8 | 2xx payload `modification.*.status == "failed"` (prepaid online order) | modify_order | `dead_end(reason: online_prepaid_not_modifiable, permanent: true)` — booking remains valid; planner offers cancel+rebook | `documented`, orders-faqs. A 2xx carrying a failure — MUST precede the transport-success fallthrough row |
| M9 | timeout / conn drop / 5xx | modify_order | `reconcile` (unkeyed in-place commit; probe per confirmation block) | P3 |
| M10 | timeout / 5xx / 429 | reads | `retry` | `documented` |

`known_unmatched` (modify): any `incorrect_parameters` message not matching M1–M7 patterns; `modifiable=false` reasons beyond the documented example — default rows apply.

#### Gate — modify

```yaml
gates:
  modify_consent:                    # entry gate on modify_order
    audience: user
    payload: {modification: object, price_current: money?, price_new: money?,
              price_known: bool}     # false on v3.1 (no preview) — warn: price may change
    outcomes:
      approve: ok
      decline: dead_end
    timeout: {after: 30m, verdict: dead_end}
```

#### Policy — modify

```yaml
policy:
  per_step:
    read: {attempts: 3, backoff: {base: 500ms, factor: 2, jitter: full, max: 8s}}
    modify_order: {attempts: 1, timeout: 60s}
  per_chain: {max_rewinds: 2, max_repairs: 2, wall_clock: 10m, gate_timeout: 30m}
  # Deployment note (outside spec, per SPEC §8 non-goals): serialize modify/cancel per
  # order_id and enforce the documented ≥5-minute spacing between modification commits
  # against the same order.
```

### 5.2 Cancel chain (also Scope A's compensation sub-chain)

Steps: `read_cancel_eligibility → cancel_order → (confirmation probe)`. When invoked as Scope A compensation, it runs under the parent run's policy with `state=compensating` at its gate (SPEC §2.1 compensation.chain).

The **free-cancellation deadline is a business-timescale boundary**, and the machinery treats it structurally, not as a failure: before the deadline the chain is fee-free; after it, a *fee-consent gate* (with a validated fresh fee) interposes; after check-in it is a **permanent structured dead end** (`order_not_cancellable` "Check-in date was yyyy-mm-dd" — `documented`). Because fees flip at a wall-clock instant that can pass *while the run is parked*, the eligibility read's preconditions are re-evaluated on every replay and the entry gate re-fires on every arrival — the SPEC's replay semantics are exactly what the docs demand ("Refresh cancellation information immediately before submitting the cancellation request").

#### Handles

```yaml
handles:
  reservation_id: {minted_by: external(scope A create_order output / details read),
                   derived_from: [order_id], staleness: {detect: [], ttl_hint: n/a, refresh: get_order_details_accom}}
  cancel_quote:                      # cached {fee, deadline, waiver_available} payload
    minted_by: read_cancel_eligibility
    derived_from: [reservation_id]
    staleness:
      detect: []                     # fee drift has NO signal — cancel does not reject on
                                     # "fee changed"; protection = fresh read + entry gate
                                     # on every arrival (documented refresh-before-submit)
      ttl_hint: unknown              # bounded by distance to the next schedule breakpoint
      refresh: read_cancel_eligibility
```

#### Actions

```yaml
actions:
  - id: read_cancel_eligibility
    description: Fresh status + fee schedule + waiver eligibility for the reservation.
    effect: read                     # /orders/details/accommodations (+ optionally
                                     # accommodations/details extras:
                                     # refuses_free_cancellation_requests — documented)
    input: {handles: [reservation_id]}
    output:
      payload: {status: enum, cancellation_details: object|null,
                fee_now: money|null,             # derived by code from policies.cancellation
                                                 # schedule vs now; null = free
                free_until: datetime|null,
                waiver_available: bool}           # !refuses_free_cancellation_requests
      handles: [cancel_quote]
    preconditions:
      - {when: "status in [cancelled, cancelled_by_guest, cancelled_by_accommodation]",
         verdict: ok,                # ALREADY CANCELLED: as compensation, the goal state
                                     # holds — advance degraded (traced); as a standalone
                                     # cancel run, report success-noop
         reason: "goal state already reached"}
      - {when: "status in [stayed, no_show] OR checkin date has passed",
         verdict: dead_end,          # reason: past_boundary, permanent: true —
         reason: "documented order_not_cancellable condition; fail before dispatch"}
    idempotency: {mode: natural}
    timeout: 20s

  - id: cancel_order
    description: Cancel the accommodation reservation (synchronous).
    effect: compensate               # commit-class
    entry_gate: cancel_fee_consent   # re-fires every arrival: the fee shown is re-derived
                                     # from a FRESH cancel_quote each time (I1: a rewind
                                     # of the read replaces the quote; the gate payload is
                                     # lineage-keyed to it)
    input:
      handles: [reservation_id, cancel_quote]
      intent: {reason: {type: string, required: true},
               request_property_approval: {type: enum[true,false], required: false}}
               # cancel-for-less waiver flag; v3.2, accommodations only (documented)
    output: {payload: {status: enum[successful]}}
    idempotency: {mode: natural}     # justification (Q5): re-dispatch cannot create a
                                     # second cancellation; repeat yields deterministic
                                     # 409 order_not_cancellable "already been cancelled";
                                     # docs explicitly sanction retrying 500/504 here
    confirmation:
      probe: get_order_details_accom
      signal: "status in [cancelled_by_guest, cancelled] AND cancellation_details
               present (at, fee — fee null when free)"
      async: {channel: poll, deadline: 1h}
      sweep: {interval: 15m, escalate_after: 12h}
    compensation:
      action: none
      ordering_note: "Un-cancelling does not exist. Pre-commit protection =
        cancel_fee_consent entry gate showing the validated fee. In rebook flows the
        replacement booking must land BEFORE this compensator runs (C8.2)."
    timeout: 60s                     # assumed (UNKNOWN-6)
```

#### Verdict table — cancel domain rows

| # | Signal | Step scope | Verdict | Evidence |
|---|---|---|---|---|
| C1 | `409 order_not_cancellable` "The order has already been cancelled" | cancel_order | `reconcile` — probe; status cancelled ⇒ treat as landed/ok (goal state holds; who cancelled is in `status`). Never an error to the user before the probe reads the truth | `documented`, cancel/error-handling |
| C2 | `409 order_not_cancellable` "Check-in date was yyyy-mm-dd" / `order_unavailable` "current state (e.g. stayed)" | cancel_order | `dead_end(reason: past_boundary, permanent: true)` — the business-timescale boundary crossed for good | `documented`, cancel/error-handling + http-4xx |
| C3 | `403` on a recently created reservation | cancel_order | `retry` with extended backoff — docs: "If the reservation was created recently, wait a few minutes before retrying" (safe: idempotency natural) | `documented`, cancel/error-handling |
| C4 | `500` / `504` / timeout | cancel_order | `retry` (same inputs) — explicitly documented as retryable; natural idempotency makes this safe where it would be forbidden on create | `documented`, cancel/error-handling |
| C5 | `404` | cancel_order | `dead_end(reason: bad_reference, permanent: true)` — verify identifiers via details | `documented` |
| C6 | timeout / 5xx / 429 | read_cancel_eligibility, probe | `retry` | `documented` |
| — | generic rows 1–12 | | | |

`known_unmatched` (cancel): `400` request-shape errors on cancel (config bugs → default dead_end); waiver-approval outcomes — "orders/details do not present whether the request is approved or not. This feature will be activated in further versions" (`documented`, cancel-for-less) — there is no signal to classify, see UNKNOWN-10.

#### Gate — cancel

```yaml
gates:
  cancel_fee_consent:                # entry gate on cancel_order; while running as
    audience: user                   # Scope A compensation the run parks state=compensating
    payload: {fee_now: money|null, free_until: datetime|null,
              waiver_available: bool, policy_type: enum}
    outcomes:
      proceed: ok                    # cancel, paying fee_now (possibly zero)
      request_waiver: {params: {request_property_approval: enum[true]},
                       bind: [request_property_approval],
                       verdict: ok}  # cancel-for-less: flag rides the same dispatch;
                                     # approval is at the property's discretion and NOT
                                     # observable via API today (UNKNOWN-10) → operator
                                     # follow-up recorded on the run
      abort: dead_end                # keep the booking (as compensation: escalate —
                                     # commit stands uncompensated, C8.3/C8.5 reporting)
    timeout: {after: 72h, verdict: dead_end}   # business decision window; on timeout
                                     # during compensation → escalate(operator): landed
                                     # commit with stuck unwind
```

#### Policy — cancel

```yaml
policy:
  per_step:
    read: {attempts: 3, backoff: {base: 500ms, factor: 2, jitter: full, max: 8s}}
    cancel_order: {attempts: 3, backoff: {base: 2s, factor: 2, jitter: full, max: 30s},
                   timeout: 60s}     # retry sanctioned by natural idempotency + docs
  per_chain: {max_rewinds: 2, max_repairs: 1, wall_clock: 10m, gate_timeout: 72h}
```

### 5.3 Invalidation walkthroughs — Scope B

**Walkthrough B-1: the deadline crosses while the user thinks.**
`read_cancel_eligibility` at T0 shows `fee_now: null`, `free_until: T0+40m`. `cancel_fee_consent` parks the run. The user answers `proceed` at T0+3h — past the deadline. Naive design charges them a fee they consented to as "free". Here: the gate's payload is lineage-keyed to `cancel_quote`; on resume the runtime re-arrives at `cancel_order`, whose **entry gate re-fires on every arrival** — but the fee it must show has to be *fresh*, so the conforming compile makes staleness of the quote structural: resume applies `rewind(to: read_cancel_eligibility)` when `free_until` in the quoted payload has passed at answer-validation time (a code-owned check against the gate's own presented payload — the answer references a quote whose stated validity expired, rejected like any invalid proposal, C6.3). The re-read replaces `cancel_quote` (I1/I2 — the old fee figure is unreadable now), preconditions re-evaluate (maybe it's `stayed` now → structured dead_end), and the gate re-fires showing the real fee. The free-cancellation deadline thus never silently converts a free cancel into a paid one. Budget: one rewind.

**Walkthrough B-2: modify guest-limit bounce.**
`modify_order` (type=room, 3 adults) → M5 "Number of guests exceeds the allowed limit" → `repair(to: modify_order, fields: [modification])` (repair budget 2→1); the user (via the modify_consent replay + model proposal validated against `maximum_occupancy` from details) resubmits 2 adults. No handles die: `order_id` and `room_reservation_id` are durable (I5 — landed commits/objects are facts, not cache; the failed modify landed nothing). If instead the answer is M4 (rate-level occupancy lock) the run dead-ends permanently and the planner offers the cancel+rebook composite — which is Scope B cancel + a fresh Scope A run, ordered replacement-first (C8.2), NOT an intra-run recovery.

---

## 6. Cross-chain notes

- `order_id` / `reservation_id` / `pincode` minted by Scope A `create_order` are the durable inputs to both Scope B chains and to the confirmation probes. The correlation record maps `run_id → label → order_id` once the probe or the create response supplies it; Scope B runs then reference orders by id only (never by label).
- Scope A's `compensation.chain` IS Scope B's cancel chain — one compile, two invocation modes (standalone run vs `state=compensating` sub-chain). The `cancel_fee_consent` timeout escalates differently in compensation mode (stuck unwind → operator, C8.5).
- The cancellation schedule captured at Scope A preview time (`order_token` payload) is *display provenance* only; Scope B always re-reads (`documented`: post-booking `/orders/details/*` is the source of truth for current eligibility). No handle carries policy data across chains.
- Concurrent Scope-B runs against one order (modify racing cancel; two modifies inside the documented 5-minute spacing) are cross-run serialization on a shared durable object — a SPEC §8 non-goal; enforce per-`order_id` locking at deployment level.
- Fan-out (I7): comparing candidate properties/products in parallel runs sharing the `search_results` read prefix is supported by lineage keying; product collections are keyed by (property, dates, occupancy) lineage so branch A's prices can never be read under branch B.

---

## 7. UNKNOWNs & assumptions (each with the question a human must answer)

1. **Accommodation `orders/create` double-submission behavior.** No idempotency key exists; `duplicate_request` dedup is documented cars-only; `order_token` reuse semantics undocumented. *Ask Booking.com: does re-POSTing the same `order_token` create a second reservation, error, or return the original?* Until answered: `attempts: 1`, token `single_use: true`, reconcile-only. (`assumed` on the conservative side.)
2. **Exact signal for expired/invalid `order_token` at create** (HTTP status + `errors[].id`). Row A1 matches on message content; anything else falls to defaults. *Ask for the error id; then pin A1 to it.*
3. **Does the `order_token` lock the price?** Docs say the token "encapsulates all order information" and exists "to reduce the error % caused by property changing prices" (`documented`, orders-faqs) — implying the create charges the preview price within 15 minutes — but no page states "the price cannot change at create". No price-changed-at-create signal is documented (only `order_unavailable`). *Confirm: can create succeed at a different amount than preview showed? If yes, what signal?* The `payment_refused_invalid_amount` row suggests amount mismatches surface as failures, not silent drift — weak evidence for the lock.
4. **Does a 422 `payment_refused_*` guarantee no reservation was created?** Assumed yes (payment is processed as part of create); the confirmation probe backstops the assumption. *Verify with provider; if a declined-payment reservation can exist in any state, add a probe-on-422 step.*
5. **Is 429 on `orders/create` `rejected_before_execution`?** The rate limiter reads as an account-level pre-execution gate, but "no booking/charge was created" is never stated. Currently NOT declared; 429-on-create → reconcile (row A12), `safe_reject_redispatch: 0`. *Provider confirmation would upgrade this to a bounded safe re-dispatch.*
6. **Latencies/timeouts** for create/modify/cancel are undocumented; 90s/60s/60s are `assumed`. Calibrate from sandbox traffic (test hotel 10507360 — charges are real, auto-refunded Mondays; `documented`, orders-faqs).
7. **Identifier-based details lag right after create** (is a just-created order immediately readable via `/orders/details/accommodations` by reservation id?). The 2h lag is documented only for the date-window report. Affects how quickly a reconcile can resolve via the id-path once the id is known.
8. **Product id stability across availability refreshes.** Ids look like deterministic composites but are not documented stable → rematch is by content (room + policy + price). *If provider confirms stability, rematch can key on the id and `on_ambiguous` churn drops.*
9. **Modify gaps**: (a) no undo — confirmed by omission but worth an explicit provider statement; (b) v3.1 has no modify preview → price-blind consent for date changes outside Beta; (c) card modification success is not observable in any documented read (no card fingerprint in details) → content-based confirmation impossible for type=card; lost-response card modifies escalate to operator.
10. **Cancel-for-less approval is unobservable** — "orders/details do not present whether the request is approved or not. This feature will be activated in further versions" (`documented`). The waiver outcome parks a manual follow-up; no verdict row can exist for approval/denial yet.
11. **`search_results` / `availability_products` have empty `staleness.detect`** — flagged per SKILL Step 2: staleness is only discoverable downstream (`order_unavailable`) or via valid-empty re-reads. Consumers' stale-handle rows fall to defaults by design.
12. **`payment.timing = pay_online_later` operational risk**: Booking.com charges the VCC/card per schedule later; "ensure the VCC has sufficient funds at least 2 days prior to the pay-later collection date" (`documented`, partner-collects). Post-chain obligation — outside this chain, needs a scheduled job; flagged so nobody thinks the chain covers it.
13. **Multi-room orders return a single order id + pincode** (`documented`, orders-faqs) and cancel takes a `reservation` — whether individual rooms of one accommodation order can be cancelled separately via the API is not documented (statuses can differ per product, implying it happens on Booking.com's side). *Ask before building partial-cancel UX.*
14. **Compensation `window` enforcement**: the schedule is data (per-product, per-booking); the config expresses it as the gate payload + structured dead ends, never as a local timer (P2-consistent). No open question — recorded as a design note.

---

## 8. Boundary recap (SPEC §5 applied to these chains)

| Model touchpoint | Where | Inside SPEC §5 allowance? |
|---|---|---|
| Pick a property from search results | `select_property` | ✅ choosing among returned options; recorded as I3 derivation with rematch key `accommodation_id` |
| Pick product(s)/rooms from availability | `select_product` | ✅ same; rematch by content; `on_ambiguous: gate` because re-picks move money |
| Propose new dates/location/guests on empty search or availability | `repair` fields listed in `empty.route` / gate outcomes | ✅ intent-field repair proposals; code validates allow-list |
| Propose reallocation on `occupancy_mismatch` | `occupancy_review` gate params → repair | ✅ gate-carried params bind only `allocation` |
| Suggest an alternative payment instrument after a decline | row A5 repair proposal | ✅ intent field `payment`; card data handled by code/sealed store; model sees instrument aliases only |
| Phrase gate payloads / dead-end explanations | all gates, all dead_ends | ✅ phrasing from machine-readable reasons |
| Decide what to do after `dead_end` (rebook elsewhere, cancel+rebook composite) | planner, above the chain | ✅ explicitly outside the spec |
| — Never: touch `order_token`/product ids/order ids, extend the 1-attempt create budget, reclassify a 409/422, answer `booking_consent`/`cancel_fee_consent`/`modify_consent` (money gates, `audience: user`), or re-fire a commit after reconcile | — | enforced by config: no gate grants `audience: model`; verdict table has no model rows; label injection is a `request_invariant` |

---

## 9. Self-check (SKILL Step 7)

- [x] Every commit/compensator declares `confirmation` — create (probe find_order_by_label), modify (content-based probe; type=card gap flagged UNKNOWN-9c rather than papered over), cancel (probe details status). No `by_key_replay` anywhere (no keys exist).
- [x] Unkeyed commits have `attempts: 1` (create, modify); cancel justified as `natural` with documented evidence.
- [x] Correlation record persisted before dispatch + cancellation shield stated for create and modify; record lives in integrator's durable store; token rides `accommodation.label`.
- [x] Every selection pseudo-handle has a `rematch` spec (`selected_property`, `selected_product`).
- [x] In-place commit (`modify_order`) has `mutates`, content-based confirmation, explicit `compensation: none` + entry-gate ordering_note.
- [x] Handles consumed by commits: `order_token` has staleness.detect (weak, flagged) + documented ttl_hint; `order_id`/`reservation_id` durable.
- [x] `single_use` (`order_token`) consumed by exactly one step (create_order).
- [x] Model touchpoints = selections, intent repairs, gate phrasing only (§8); no handle is model-repairable.
- [x] Every documented error code is mapped in a domain row or listed in `known_unmatched` (per scope).
- [x] Every gate has audience, outcome→verdict mapping, timeout.
- [x] Empty-result routes exist for search, availability, and the probe (empty probe = valid "not landed yet").
- [x] Invalidation walkthroughs (A-1/A-2/A-3, B-1/B-2) traced against the `derived_from` edges by hand.
- [x] Payload business blockers are `preconditions` (occupancy_mismatch, adults-only, status≠booked, modifiable=false, address-required), not overloaded `empty`.
- [x] No TTL used as an enforcement timer — the documented 15m token TTL is planning-only (gate timeout 12m is a *product* choice informed by it; expiry is still detected on use).
- [x] No budget or retry limit lives in prompt text — all in the policy blocks.
- [x] Compensation ordering stated for rebook flows (replacement-first) in both scopes.

*Deliverable ends. Implementation (client code, runtime, per-object locking, the pay-later funding job) is a separate task consuming this document.*

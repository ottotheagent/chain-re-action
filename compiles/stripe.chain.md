# Stripe PaymentIntents — Chain Compile

Created: 2026-07-27
Last Updated: 2026-07-27
Compiled against: SPEC.md Draft v0.3.1 (pre-implementation), SKILL.md chain-compile, CONFORMANCE.md (SPEC v0.3.1)

Evidence corpus: Stripe public documentation (docs.stripe.com) only. Tags:
`documented` = official Stripe reference/guide (URL cited per claim);
`assumed` = inference, needs human confirmation; `observed` unused (no API
access in this compile). Per the SKILL `spec-example` rule, **nothing** from
SPEC.md worked example B (card-payment chain) is used as evidence — every
fact below is re-derived from docs.stripe.com, and §12 lists where the real
Stripe API contradicts or refines example B.

Scopes compiled (one document, shared elicitation + verdict core per SKILL
Step 6 multi-scope guidance):

1. **Chain M** — manual-capture payment: `create_customer → attach_payment_method
   → create_payment_intent(capture_method=manual) → confirm_payment_intent →
   [sca_challenge gate] → verify_authorization → capture`; compensators
   `cancel_payment_intent` (pre-capture) and the refund chain (post-capture).
2. **Idempotency semantics** — woven into Chain M's elicitation (Q4/Q5) and
   verdict rows; decides every keyed-commit retry row.
3. **Chain R** — refund: `create_refund → async finality (webhook/probe)`,
   with asynchronous post-success failure modeled.

---

## 1. Chain summaries

**Chain M.** Place an authorization hold on a customer's card (manual
capture), survive 3DS/SCA challenges and declines, then capture some or all
of the held amount. The hold is a *consequential* reservation (ties up the
cardholder's credit line), self-expiring on a network-dependent window
(default 7 days online; the uncaptured PaymentIntent is then auto-canceled).
Partial capture releases the remainder automatically. Compensation before
capture = cancel the PaymentIntent (releases the hold); after capture =
Chain R.

**Chain R.** Refund a captured charge, fully or partially (multiple partial
refunds allowed up to the original amount). Creation usually succeeds
synchronously into `pending`/`succeeded`, but a refund **can fail
asynchronously up to ~30 days later** (closed account, expired card); the
failed amount returns to the merchant's Stripe balance and Stripe emits
`refund.failed`. Finality is therefore webhook/probe-driven, and terminal
failure requires a human to arrange an alternative refund.

---

## 2. Fit-test verdict (SKILL Step 0)

A chain is the right abstraction. Properties that fired:

- **Handles + expiry**: the authorization hold is server-side state consumed
  by a later step (capture) and expires on a discovered, network-dependent
  window ("If the authorization expires before you capture the funds, the
  funds are released and the payment status changes to `canceled`" —
  https://docs.stripe.com/payments/place-a-hold-on-a-payment-method). P2 applies.
- **Commits with compensation**: confirm (hold), capture (money moves),
  refund — each a durable externally visible mutation with its own
  business-level undo (cancel / refund), fees and windows.
- **Mid-chain mandatory human gate**: 3DS/SCA `requires_action` is a routine
  path, not an error (https://docs.stripe.com/payments/paymentintents/lifecycle).
- **Valid-empty**: card declined is a documented non-error outcome with
  machine-readable `decline_code` feeding repair routing
  (https://docs.stripe.com/declines/codes).
- **Async finality**: capture/refund finality confirmed by webhooks with **no
  ordering guarantee and possible duplicates** (https://docs.stripe.com/webhooks).

None of the refusal conditions hold: multiple steps, non-idempotent-by-nature
money movement, expiring intermediate state, provider does *not* run the
recovery state machine for you (it cancels expired holds but does not retry,
re-authorize, or arrange failed-refund alternatives). Q2/Q3 are answerable
for every commit (below), so no BLOCKED verdict.

One honest note: because Stripe keys **every** POST
(https://docs.stripe.com/api/idempotent_requests), the `reconcile`-heavy
machinery is less load-bearing than in the spec's origin domain — but it is
NOT vestigial: replayed cached 500s and the 24h key-retention cliff (§3 Q4)
route straight back into `reconcile`, and Chain R's async failure needs the
`confirmation.async` + sweep machinery outright.

---

## 3. Elicitation log (SKILL Step 1)

Every answer tagged. URLs are the evidence; quotes are exact.

### 3.1 Step: `create_customer` (POST /v1/customers)

- **Q1 double-call**: two distinct Customer objects exist (no server-side
  dedup by email is documented anywhere in the Customers API — absence of a
  dedup claim is itself the evidence). Durable but inconsequential: never
  billed, not cardholder-visible. `assumed`: Stripe does not dedupe customers
  by email; duplicates are harmless orphans.
- **Q2 death mid-call**: a Customer may exist. Costs nothing, holds nothing,
  not user-visible → by the §2.1 consequence rubric this *could* be a mint,
  but the artifact is durable (never self-expires), so classified
  `commit` with `compensation: none` (justified: orphan customer records are
  inert; GC out of band). `assumed` classification judgment; facts `documented`.
- **Q3 how to confirm**: keyed synchronous — key replay returns the original
  response ("Stripe saves the resulting status code and body of the first
  request… Subsequent requests with the same key return the same result" —
  https://docs.stripe.com/api/idempotent_requests). `documented`. Belt: stamp
  `metadata.run_ref` at create so a sweep can find orphans. `assumed` design.
- **Q4 idempotency**: all POSTs accept `Idempotency-Key` (≤255 chars).
  `documented` (same URL). See §3.6 for the full idempotency findings.

### 3.2 Step: `attach_payment_method` (POST /v1/payment_methods/:id/attach)

- **Q1 double-call**: re-attaching a PM already attached to the same customer
  is **not documented** (https://docs.stripe.com/api/payment_methods/attach
  documents neither the error nor idempotence) → UNKNOWN-1. `documented`
  (that it's undocumented).
- **Q2**: attachment may exist; user-visible (saved card) → `commit`,
  in-place (`mutates` the PaymentMethod's `customer` field — no new object id
  minted). `assumed` classification; endpoint shape `documented`.
- **Q2b/Q3**: in-place ⇒ content-based confirmation: GET the PaymentMethod
  and assert `customer == customer_id` (existence proves nothing). `assumed`
  probe design over `documented` read endpoint.
- **Note**: Stripe recommends SetupIntent or PaymentIntent
  `setup_future_usage` instead of bare attach: "Using the
  `/v1/payment_methods/:id/attach` endpoint without first using a SetupIntent…
  does not optimize the PaymentMethod for future use, which makes later
  declines and payment friction more likely."
  (https://docs.stripe.com/api/payment_methods/attach) `documented`. This
  compile keeps bare attach for scope but flags the recommendation (UNKNOWN-2).
- **Q12 compensation**: `detach` endpoint exists (Payment Methods API);
  `assumed` harmless.

### 3.3 Step: `create_payment_intent` (POST /v1/payment_intents, capture_method=manual)

- **Q1 double-call**: two PaymentIntents in `requires_payment_method` (the
  initial status — https://docs.stripe.com/payments/paymentintents/lifecycle).
  No hold, no money, not cardholder-visible; inert. By the mint-vs-commit
  consequence rubric: **mint** — the consequence, not the self-expiry,
  decides, and an unconfirmed PI is inconsequential. Caveat honestly noted:
  unconfirmed PIs are *durable* (no documented self-expiry), so they are
  orphaned state that does NOT self-expire — a mild deviation from the spec's
  mint description ("artifacts that self-expire"); still mint because
  inconsequence is the operative test. `documented` facts, `assumed`
  classification.
- **Q4**: keyed anyway (defense in depth; all POSTs). `documented`.
- **request_invariants**: `capture_method: manual` — a wrong/omitted value
  yields a **well-formed response that silently auto-captures**, invisible to
  any verdict row. Exactly the spec's request-invariant slot. `documented`
  param (https://docs.stripe.com/payments/place-a-hold-on-a-payment-method).

### 3.4 Step: `confirm_payment_intent` (POST /v1/payment_intents/:id/confirm) — the HOLD commit

- **Q1 double-call**: a successful confirm places an authorization hold
  (status → `requires_capture`) that "ties up" the cardholder's credit line
  for days. Re-dispatch without a key from a fresh PI would create a second
  consequential hold. **Per the §2.1 rubric: COMMIT, however short-lived —
  mint status comes from inconsequence, not self-expiry.** A second confirm
  of the *same* PI in `requires_capture` errors
  (`payment_intent_unexpected_state`: "The PaymentIntent's state was
  incompatible with the operation" — https://docs.stripe.com/error-codes).
  `documented`.
- **Q2**: process death mid-confirm → a hold may exist (cardholder-visible on
  their available credit). Commit; Q3 mandatory.
- **Q3 confirm landed?**: GET /v1/payment_intents/:id → status ∈
  {`requires_capture`, `requires_action`, `processing`} = landed (in some
  form); `requires_payment_method` + `last_payment_error` = attempt failed
  ("If the payment attempt fails… the PaymentIntent's status returns to
  `requires_payment_method` so that the payment can be retried" —
  https://docs.stripe.com/payments/paymentintents/lifecycle). Webhooks:
  `payment_intent.amount_capturable_updated` ("Occurs when a PaymentIntent
  has funds to be captured"), `payment_intent.requires_action`,
  `payment_intent.payment_failed`
  (https://docs.stripe.com/api/events/types). Queryable lag: none documented
  for card confirms (`assumed`: probe authoritative immediately). Webhook
  worst-case: retried "for up to three days with an exponential back off"
  (https://docs.stripe.com/webhooks). `documented` except where tagged.
- **Q6 hold lifetime (network-dependent — the task's rubric investigation)**:
  `documented`, all from
  https://docs.stripe.com/payments/place-a-hold-on-a-payment-method:
  - Online (card-not-present), customer-initiated: **7 days** (Visa, MC,
    Amex, Discover). Visa merchant-initiated: **5 days** ("The exact
    authorization window is 4 days and 18 hours, to allow time for clearing
    processes.").
  - Card-present: Visa 5 days (4d18h exact); **Mastercard/Amex/Discover 2
    days**.
  - Japan accounts, JPY: up to 30 days (Visa/MC/JCB/Diners/Discover).
  - Non-card: Affirm 30d, Afterpay/Clearpay 13d, Cash App Pay 7d, Klarna
    "by midnight of the 28th calendar day", PayPal 10d + auto-extend 10d.
  - Extended authorizations (IC+ pricing, eligible networks/merchant
    categories): up to **30 days** (Visa exact window 29d18h; Amex lodging &
    vehicle rental only; Discover specific categories). Request via
    `payment_method_options[card][request_extended_authorization]=if_available`;
    verify via `charge.payment_method_details.card.extended_authorization.status`
    and — normative Stripe guidance — "Rely on the `capture_before` field to
    confirm the validity window for a given payment because these rules can
    change without prior notice."
    (https://docs.stripe.com/payments/extended-authorization.md?platform=web&ui=elements)
  - **Expiry consequence**: "If the authorization expires before you capture
    the funds, the funds are released and the payment status changes to
    `canceled`." Plus API-ref: "Uncaptured PaymentIntents are cancelled
    automatically 7 days after creation"
    (https://docs.stripe.com/api/payment_intents/capture). Event:
    `charge.expired` "Occurs whenever an uncaptured charge expires"
    (https://docs.stripe.com/api/events/types). So expiry is *discovered* via
    `payment_intent.canceled`/`charge.expired` webhooks, probe status
    `canceled`, or capture error `charge_expired_for_capture` ("Authorization
    and capture charges must be captured within a set number of days (7 by
    default)" — https://docs.stripe.com/error-codes). `capture_before` is the
    per-payment ttl_hint. P2 fully satisfied: hint for planning, signals for
    detection.
- **Q7 cheapest re-mint**: an expired hold means the PI is **canceled —
  a terminal state** ("Cancellation… can't be undone",
  https://docs.stripe.com/payments/paymentintents/lifecycle,
  https://docs.stripe.com/api/payment_intents/cancel). You cannot re-confirm
  a canceled PI; the refresh target is `create_payment_intent` (new PI).
  `documented`.
- **Q9 empty-is-answer (declines)**: yes — decline is the domain's
  valid-empty. Transport shape: HTTP **402** `card_error`, `code:
  card_declined`, with `decline_code` ("When a card is declined, the error
  returned also includes the `decline_code` attribute" —
  https://docs.stripe.com/error-codes); PI returns to
  `requires_payment_method` and **remains reusable** for a retry with a
  different instrument (lifecycle URL above). Decline codes and Stripe's
  prescribed routing (https://docs.stripe.com/declines/codes), all `documented`:
  - Never reveal / hard stop: `stolen_card`, `lost_card`, `fraudulent`
    ("Don't report more detailed information. Instead, present as
    `generic_decline`"), `merchant_blacklist`.
  - Use another payment method: `insufficient_funds`,
    `withdrawal_count_limit_exceeded`, `expired_card`.
  - Fix and retry same card: `incorrect_cvc`, `incorrect_number`.
  - Contact issuer: `do_not_honor`, `generic_decline`, `call_issuer`,
    `card_velocity_exceeded`, `pickup_card`, `restricted_card`, etc.
  - Retryable: `processing_error` ("Payment needs to be attempted again"),
    `try_again_later` (deprecated).
  - `authentication_required`: soft decline → 3DS ("If requested by the
    issuer with a soft decline, we automatically reattempt and continue as if
    required" — https://docs.stripe.com/payments/3d-secure/authentication-flow).
  - Radar block: `outcome.type: "blocked"`, `advice_code: "do_not_try_again"`
    (https://docs.stripe.com/declines).
- **Q11 human-decision events (3DS/SCA gate)**: `requires_action` with
  `next_action` (`redirect_to_url` | `use_stripe_sdk`); the **customer**
  completes the challenge in their client (Stripe.js `confirmCardPayment` /
  native SDKs). Outcomes (`documented`,
  https://docs.stripe.com/payments/3d-secure/authentication-flow):
  - "Authenticated: Stripe attempts the charge and the PaymentIntent
    transitions to a status of `processing`" (then `requires_capture` for
    manual capture).
  - "Failure: The PaymentIntent transitions to a status of
    `requires_payment_method`" — retry different PM or reconfirm. Error code
    `payment_intent_authentication_failure` ("The provided payment method has
    failed authentication. Provide a new payment method…" —
    https://docs.stripe.com/error-codes).
  - With `confirmation_method: manual`: "the PaymentIntent will return to the
    `requires_confirmation` state after those actions are completed. Your
    server needs to then explicitly re-confirm"
    (https://docs.stripe.com/api/payment_intents/confirm).
  - **What expires**: no API-level TTL for the challenge session or for how
    long a PI may sit in `requires_action` is documented → UNKNOWN-3. Only
    signal: iOS SDK `authenticationTimeout` "must be at least 5 minutes"
    (3DS flow URL above). Gate timeout is therefore a product decision, not a
    documented provider constant.
  - Confirm-attempt ceiling: "There is a variable upper limit on how many
    times a PaymentIntent can be confirmed. After this limit is reached, any
    further calls… transition the PaymentIntent to the `canceled` state"
    (https://docs.stripe.com/api/payment_intents/confirm). Value undocumented
    → bounded by our own `max_repairs` anyway; UNKNOWN-4.

### 3.5 Step: `capture` (POST /v1/payment_intents/:id/capture) — **the money commit; Q1–Q5 answered explicitly per task**

- **Q1 — double-call, same inputs**: the first capture moves money once
  (status → `succeeded`); the second call **errors deterministically** —
  `charge_already_captured`: "The charge you're attempting to capture has
  already been captured." (https://docs.stripe.com/error-codes), and the
  endpoint errors "if the PaymentIntent isn't capturable"
  (https://docs.stripe.com/api/payment_intents/capture). With the SAME
  idempotency key, the second call doesn't execute at all: "Stripe saves the
  resulting status code and body of the first request… Subsequent requests
  with the same key return the same result"
  (https://docs.stripe.com/api/idempotent_requests). One capture per
  authorization: "If you partially capture a payment, you can't perform
  another capture for the difference" (unless multicapture; excluded here by
  request_invariant `final_capture` defaulting true)
  (https://docs.stripe.com/payments/place-a-hold-on-a-payment-method).
  → effect: **commit**. `documented`.
- **Q2 — process dies mid-call**: the capture may have executed — money
  moved, cardholder-visible, settles to the merchant. Commit confirmed.
  **Q2b**: capture MUTATES the existing PaymentIntent/charge in place
  (uncaptured charge → captured; event `charge.captured` "Occurs whenever a
  previously uncaptured charge is captured" —
  https://docs.stripe.com/api/events/types). No new durable object id is
  minted (`latest_charge` exists from confirm). → `mutates: auth_hold`,
  confirmation must be **content-based** (observe `status == succeeded` /
  `amount_received`), existence proves nothing. `documented` behavior,
  `assumed` modeling.
- **Q3 — how to learn whether an ambiguous attempt landed**: probe
  GET /v1/payment_intents/:id; landed ⇔ `status == succeeded` (capture
  "Returns PaymentIntent object with status=succeeded if capturable" —
  https://docs.stripe.com/api/payment_intents/capture) with
  `amount_received` equal to the captured amount; not-landed ⇔ still
  `requires_capture` (re-dispatch safe) or `canceled` (hold expired
  meanwhile). Queryable lag: none documented for card captures (`assumed`
  none — capture is synchronous). Webhooks `payment_intent.succeeded` /
  `charge.captured`; worst-case webhook delay: retried "for up to three days
  with an exponential back off in live mode"
  (https://docs.stripe.com/webhooks). `documented` except the lag assumption.
- **Q4 — idempotency key**: yes; every POST accepts `Idempotency-Key` (≤255
  chars, V4-UUID recommended). Scope: account-global per key. **Retention:
  keys "can be removed from the system automatically after they're at least
  24 hours old"** — after pruning, "reusing it generates a new request".
  **Replay**: "Stripe saves the resulting status code and body of the first
  request… regardless of success or failure. Subsequent requests with the
  same key return the same result, including `500` errors." Results saved
  only "after the execution of an endpoint begins"; NOT saved when params
  fail validation or when "the request conflicts with another request
  executing concurrently" (those are retryable). **Key reuse with different
  params**: the layer "compares incoming parameters to those of the original
  request and errors if they're not the same" — error type
  `idempotency_error` ("occur when an `Idempotency-Key` is re-used on a
  request that does not match the first request's API endpoint and
  parameters" — https://docs.stripe.com/api/errors); HTTP 409 "Conflict —
  the request conflicts with another request (perhaps due to using the same
  idempotent key)" with code `idempotency_key_in_use` ("currently being used
  in another request… duplicate requests simultaneously" —
  https://docs.stripe.com/error-codes) for the concurrent case.
  (https://docs.stripe.com/api/idempotent_requests). All `documented`.
  Consequences compiled into policy (§8) and verdict rows K1–K4 (§6):
  same-key retry is *safe* (never double-executes) but a replayed cached 500
  is *uninformative* → `reconcile`; and **no same-key re-dispatch after 24h**
  — beyond retention a "retry" is a brand-new execution (double-capture).
- **Q5 — natural idempotency without key**: yes in effect for full capture —
  a second capture deterministically errors (`charge_already_captured`), no
  double money movement — but we key it anyway; the error-on-replay shape is
  why the confirmation probe, not the error, is the source of truth.
  `documented`.
- **amount_to_capture semantics** (task investigation): "Defaults to the full
  `amount_capturable` if it's not provided"; "must be less than or equal to
  the original amount" — **no overcapture**; partial capture: "A partial
  capture automatically releases the remaining amount"; retaining the
  remainder requires `final_capture=false`, only "when multicapture is
  available" (https://docs.stripe.com/api/payment_intents/capture,
  https://docs.stripe.com/payments/place-a-hold-on-a-payment-method).
  Errors: `amount_too_large` / `amount_too_small`
  (https://docs.stripe.com/error-codes). The capturable amount is announced
  by `payment_intent.amount_capturable_updated` ("You may capture the
  PaymentIntent with an `amount_to_capture` value up to the specified
  amount" — https://docs.stripe.com/api/events/types). All `documented`.
- **Q12 compensation**: before capture → `cancel_payment_intent`; after →
  Chain R refund. Cost over time: refund after settlement is a real refund
  (5–10 business days to the customer; ARN up to 7 business days —
  https://docs.stripe.com/refunds). `documented`.

### 3.6 Step: `cancel_payment_intent` (POST /v1/payment_intents/:id/cancel) — hold release / compensator

- Cancellable from `requires_payment_method`, `requires_capture`,
  `requires_confirmation`, `requires_action`, and "in rare cases" `processing`;
  "For PaymentIntents with a `status` of `requires_capture`, the remaining
  `amount_capturable` is automatically refunded"; "Returns an error if the
  PaymentIntent is already canceled or isn't in a cancelable state";
  cancellation "can't be undone". `cancellation_reason` ∈ {duplicate,
  fraudulent, requested_by_customer, abandoned}.
  (https://docs.stripe.com/api/payment_intents/cancel,
  https://docs.stripe.com/payments/paymentintents/lifecycle). `documented`.
- **NOT a silent no-op when repeated** — the already-canceled error must be
  classified as "compensation already landed" by the confirmation probe
  (contrast worked example B, §12 delta 6). Confirmation: probe GET PI,
  signal `status == canceled` (+ webhook `payment_intent.canceled`).

### 3.7 Chain R: `create_refund` (POST /v1/refunds)

- **Q1**: two unkeyed create_refund calls with `amount` = two refunds (Stripe
  explicitly supports multiple partial refunds: "You can issue more than one
  refund against a charge, but you can't refund a total greater than the
  original charge amount" — https://docs.stripe.com/refunds). So **NOT
  naturally idempotent — double refund is real money**; the idempotency key
  is load-bearing. A full-amount duplicate errors
  `charge_already_refunded` (https://docs.stripe.com/error-codes). `documented`.
- **Q2**: death mid-call → refund may exist and money may be moving. Commit.
- **Q3**: probe GET /v1/refunds/:id (or list refunds on the PI, matched via
  `metadata.run_ref` correlation); webhooks `refund.created`,
  `refund.updated`, `refund.failed`, `charge.refunded` ("Stripe recommends
  that you listen for the `refund.created` event" —
  https://docs.stripe.com/refunds). `documented`.
- **Statuses** (https://docs.stripe.com/api/refunds/object,
  https://docs.stripe.com/refunds): `pending` (submitted to bank/issuer,
  with `pending_reason` ∈ {processing, insufficient_funds, charge_pending}),
  `succeeded`, `failed`, `canceled`, `requires_action` (non-card methods:
  customer must supply bank details; `next_action.display_details.expires_at`
  is a documented expiry timestamp). `documented`.
- **Async failure after successful creation** (task focus): "A refund can
  fail if the customer's bank or card issuer can't process it. For example, a
  closed bank account or a problem with the card… the bank returns the
  refunded amount to us and we add it back to your Stripe account balance.
  This process can take up to 30 days from the post date." "In the rare
  instance that a refund fails, we notify you using the `refund.failed`
  event. If this occurs, you need to arrange an alternative way to provide
  your customer with a refund." `failure_reason` ∈
  {`lost_or_stolen_card`, `expired_or_canceled_card`,
  `charge_for_pending_refund_disputed`, `insufficient_funds`, `declined`,
  `merchant_request`, `unknown`}; `failure_balance_transaction` records the
  reversal; some card refunds are cancellable for a short window (Dashboard
  only). (https://docs.stripe.com/refunds,
  https://docs.stripe.com/api/refunds/object). All `documented`.
- **Q12**: the compensator's compensator is a HUMAN ("arrange an alternative
  way") → operator gate, no API action. `documented`.

### 3.8 Cross-cutting: webhooks, test clocks (Q14/Q15 + task items)

- **Webhook guarantees** (`documented`, https://docs.stripe.com/webhooks):
  "Stripe doesn't guarantee the delivery of events in the order that they're
  generated… Make sure that your event destination isn't dependent on
  receiving events in a specific order." Duplicates possible: "Webhook
  endpoints might occasionally receive the same event more than once… guard
  against duplicated event receipts by logging the event IDs." Retries: "up
  to three days with an exponential back off in live mode." Missed events:
  retrieve objects via API. ⇒ Config consequence: webhooks are *hints* that
  wake the reconciler; the **probe (fresh GET of the object) is always the
  authoritative confirmation signal**; correlation records are matched by our
  `metadata.run_ref`, and event dedup is by event id.
- **Test clocks** (conformance-testing note, `documented`,
  https://docs.stripe.com/billing/testing/test-clocks,
  https://docs.stripe.com/billing/testing/test-clocks/api-advanced-usage):
  test clocks simulate time for **Billing resources only** (customers,
  subscriptions/schedules, invoices, quotes; limits: 3 customers, 3
  subscriptions/customer, 10 unattached quotes; advance ≤2 service periods).
  **PaymentIntents and authorization-hold expiry are NOT mentioned as
  supported** — there is no documented sandbox mechanism to fast-forward a
  hold to expiry. ⇒ CONFORMANCE C2.1/C2.2 tests for this chain must inject
  the *signals* (`charge.expired`, `payment_intent.canceled`,
  `charge_expired_for_capture`) via HTTP mock, not real time travel. This is
  consistent with P2 (expiry is discovered, so tests inject the discovery).
- **Latency (Q14)**: no p95 latencies documented; `assumed` seconds-scale
  synchronous card operations; slowest steps are confirm and capture (issuer
  round-trip). Timeouts set with headroom (§8) so a tight timeout doesn't
  manufacture ambiguity.
- **Finality (Q15)**: card capture is synchronous-authoritative on the probe;
  refunds are final only at `succeeded` (up to days; failure up to 30 days).

---

## 4. Handle graph (SKILL Step 2)

```
customer_id ──► pi_id ──► auth_hold ──► (capture mutates auth_hold) ──► refund_id
   (durable)     (mint)     (commit;                                     (chain R,
                            expires per                                  derived from
                            capture_before)                              pi_id)
instrument (INTENT field, gate-validated pm_... — not a handle; see note)
```

```yaml
handles:
  customer_id:
    minted_by: create_customer
    derived_from: []
    staleness: {detect: [code:resource_missing], ttl_hint: n/a, refresh: create_customer}
    # durable; resource_missing only if deleted out-of-band
    # (https://docs.stripe.com/error-codes)

  pi_id:
    minted_by: create_payment_intent
    derived_from: [customer_id]
    staleness:
      detect: [code:payment_intent_unexpected_state]   # observed on use in a
                                                       # terminal/incompatible state
      ttl_hint: unknown          # no documented expiry for unconfirmed PIs
      refresh: create_payment_intent

  auth_hold:                     # the authorization; server state ON the PI,
    minted_by: confirm_payment_intent      # not a separate token
    derived_from: [pi_id]
    single_use: true             # spent by capture (one capture only:
                                 # "you can't perform another capture for the
                                 # difference"); partial capture releases the
                                 # remainder (final_capture defaults true)
    staleness:
      detect: [code:charge_expired_for_capture,        # at capture
               event:charge.expired,                   # webhook
               event:payment_intent.canceled,
               probe:"status == 'canceled'"]
      ttl_hint: charge.payment_method_details.card.capture_before   # per-payment,
                                 # network-dependent (7d online default; Visa MIT
                                 # 4d18h; card-present MC/Amex/Discover 2d; extended
                                 # auth up to 30d) — PLANNING ONLY (P2). Stripe
                                 # itself says rely on capture_before.
      refresh: create_payment_intent   # expiry cancels the PI (terminal) →
                                       # cheapest re-mint is a NEW PI + confirm

  refund_id:                     # chain R
    minted_by: create_refund
    derived_from: [pi_id]        # refund created against the captured PI
    staleness: {detect: [], ttl_hint: n/a, refresh: n/a}   # durable record
```

Notes:
- **`instrument` (pm_…) is intent, not a handle.** It is a provider-minted
  opaque value that enters via the one sanctioned path (SPEC §2.6): the user
  chooses/creates it in their client and it arrives through a gate outcome
  (`collect_instrument` / `new_instrument`), validated as a well-formed
  PaymentMethod reference. Declines repair it. No downstream results couple
  to it via lineage that intent-invalidation (I6) doesn't already cover.
- **No selection pseudo-handles in either chain** — no step returns an
  options collection the model picks from with downstream coupling. The
  `rematch` machinery is unused (see §13 friction).
- **The 3DS challenge is not a chain handle.** `next_action` +
  `client_secret` are consumed by the *customer's client* inside the
  `sca_challenge` gate, never by a later chain step; nothing downstream
  derives from it. (Contrast example B's `client_challenge` handle — §12
  delta 4.)
- Missing-edge check by contradiction: if `customer_id` were re-minted, is
  `pi_id` still valid? No — the PI references the customer → edge exists. If
  `pi_id` re-minted, is `auth_hold` valid? No — the hold lives on the PI →
  edge exists. `refund_id` after `pi_id` re-mint? A landed refund is a fact
  (I5), unwound only by business action — but it *references* the old PI;
  the edge is for creation-lineage only; landed refunds are never
  invalidated (I5).

---

## 5. Actions (SKILL Step 3)

```yaml
actions:
  # ───────────────────────── Chain M ─────────────────────────
  - id: create_customer
    description: Create the Stripe Customer that will own the payment method.
    effect: commit                # durable record; keyed → retry-safe.
                                  # Consequence rubric: inert & harmless, but
                                  # durable (never self-expires) → commit-with-
                                  # no-compensator rather than mint.
    input:
      intent: {email: {type: string, required: true},
               name:  {type: string, required: false}}
    output: {payload: customer, handles: [customer_id]}
    idempotency:
      mode: keyed
      key_scope: run              # NOT intent: Stripe prunes keys after ≥24h
                                  # (docs.stripe.com/api/idempotent_requests) —
                                  # intent-scoped cross-run dedup silently stops
                                  # deduping after a day. Cross-run customer
                                  # dedup must be an app-side lookup, not the
                                  # idempotency layer. (§12 delta 8)
    confirmation:
      by_key_replay: {note: "keyed synchronous; replay returns the original
                      status+body (docs.stripe.com/api/idempotent_requests).
                      Sound here because retries occur within seconds — far
                      inside the 24h retention floor. Belt: metadata.run_ref
                      stamped for the sweep."}
    compensation:
      action: none
      ordering_note: "orphan Customer records are inert and never billed;
                      GC/dedup out of band (assumed — UNKNOWN-5)"
    timeout: 15s
    latency_hint: 1s

  - id: attach_payment_method
    description: Attach the user-provided PaymentMethod to the customer for reuse.
    effect: commit
    mutates: customer_id          # in-place: sets pm.customer; no new object id.
                                  # Confirmation is content-based per SPEC §2.1.
    input:
      intent: {instrument: {type: string, required: true}}   # pm_…, gate-validated
      handles: [customer_id]
    output: {payload: payment_method, handles: []}
    idempotency: {mode: keyed, key_scope: run}
    confirmation:
      probe: get_payment_method
      signal: "payment_method.customer == customer_id"   # content, not existence
    compensation:
      action: detach_payment_method
      window: "any time; detaching a saved card is fee-free"
    timeout: 15s
    latency_hint: 1s
    # UNKNOWN-1: behavior of re-attach when already attached is undocumented.
    # UNKNOWN-2: Stripe recommends SetupIntent / setup_future_usage instead of
    # bare attach ("does not optimize the PaymentMethod for future use…").

  - id: create_payment_intent
    description: Create the PaymentIntent shell with manual capture. No hold yet.
    effect: mint                  # double-call → two inert unconfirmed PIs;
                                  # never billed, not cardholder-visible.
                                  # Consequence rubric → mint. Deviation noted:
                                  # orphans are durable, not self-expiring.
    input:
      intent: {amount:   {type: money, required: true},
               currency: {type: enum[iso4217], required: true}}
      handles: [customer_id]
    output: {payload: payment_intent, handles: [pi_id]}
    request_invariants:
      capture_method: manual      # wrong value ⇒ well-formed response that
                                  # SILENTLY AUTO-CAPTURES — invisible to every
                                  # verdict row. The canonical request_invariant.
    preconditions:
      - when: "payment_intent.capture_method != 'manual'"
        verdict: dead_end
        reason: "completeness assertion backing the invariant: never proceed
                 toward confirm on an auto-capture PI"
    idempotency: {mode: keyed, key_scope: run}
    timeout: 15s
    latency_hint: 1s

  - id: confirm_payment_intent
    description: Confirm with the instrument — PLACES THE AUTHORIZATION HOLD.
    effect: commit                # the hold ties up the cardholder's credit
                                  # line for days and a repeat creates a second
                                  # consequential hold → COMMIT by the §2.1
                                  # rubric, despite self-expiry (mint =
                                  # inconsequence, not short life).
    entry_gate: authorize_consent # re-fires on EVERY arrival, incl. the
                                  # rewind-after-expiry replay: a NEW hold on
                                  # the user's card needs fresh consent by
                                  # construction (SPEC §2.1 entry_gate).
    input:
      intent: {instrument: {type: string, required: true}}
      handles: [pi_id]
    output:
      payload: {status: enum[requires_capture, requires_action, processing,
                             requires_payment_method],
                last_payment_error: object?, next_action: object?,
                capture_before: timestamp?}
      handles: [auth_hold]
      empty:
        valid: true
        detect: "signal is http:402 card_error code:card_declined"
        route: gate               # → gate(new_instrument); decline is an
                                  # answer (P4). NOTE spec strain: Stripe
                                  # transports its valid-empty as an HTTP 402
                                  # error object, not a 2xx payload — `detect`
                                  # here names a signal, not a payload
                                  # predicate; rows S1–S5 implement it. (§13)
    preconditions:
      - when: "status == 'processing'"
        verdict: ok               # async settle toward requires_capture;
                                  # verify_authorization polls it out
        reason: "asynchronous methods pass through processing (lifecycle doc)"
    idempotency: {mode: keyed, key_scope: run}
    confirmation:
      probe: get_payment_intent
      signal: "status in ['requires_capture','requires_action','processing']
               → landed; status=='requires_payment_method' with
               last_payment_error → attempt failed (classifiable cause,
               re-fed once per SPEC reconcile transition)"
      async: {channel: webhook, deadline: 1h}
        # payment_intent.amount_capturable_updated / requires_action /
        # payment_failed; webhooks unordered+duplicated → probe authoritative
      sweep: {interval: 10m, escalate_after: 24h}
    compensation:
      action: cancel_payment_intent
      window: "any time before capture; releases remaining amount_capturable
               ('automatically refunded' — api/payment_intents/cancel); hold
               self-expires per capture_before anyway (7d online default)"
    timeout: 60s                  # issuer round-trip; generous so a tight
                                  # timeout doesn't manufacture reconciles
    latency_hint: 3s

  - id: verify_authorization
    description: Post-gate / post-processing read; the single place the run
                 asserts the hold is live and learns amount_capturable.
    effect: read
    input: {handles: [pi_id, auth_hold]}
    output:
      payload: {status: enum, amount_capturable: money, capture_before: timestamp?}
      handles: []
    preconditions:
      - when: "status == 'requires_capture'"
        verdict: ok
        reason: "hold live; proceed to capture"
      - when: "status == 'processing'"
        verdict: retry            # poll until requires_capture (read policy)
        reason: "async settle in progress"
      - when: "status == 'requires_payment_method'"
        verdict: gate(new_instrument)
        reason: "3DS failed or declined post-action (lifecycle:
                 payment_intent_authentication_failure)"
      - when: "status == 'requires_confirmation'"
        verdict: rewind(to: confirm_payment_intent)
        reason: "manual confirmation_method: server must re-confirm after
                 next_actions complete (api/payment_intents/confirm)"
      - when: "status == 'canceled'"
        verdict: rewind(to: create_payment_intent)
        reason: "hold expired or confirm-limit cancel; PI terminal"
      - when: "status == 'succeeded'"
        verdict: dead_end
        reason: "capture_method invariant breached — auto-captured; permanent
                 config alarm, needs operator (refund to undo)"
    idempotency: {mode: natural}
    timeout: 15s
    latency_hint: 500ms

  - id: capture
    description: Capture the authorized funds (full or partial). THE money commit.
    effect: commit
    mutates: auth_hold            # in-place: uncaptured charge → captured; no
                                  # new object minted (charge.captured event).
                                  # Confirmation is content-based.
    input:
      intent: {amount_to_capture: {type: money, required: false}}
        # default = full amount_capturable; must be ≤ authorized (no
        # overcapture); partial ⇒ remainder AUTOMATICALLY RELEASED
        # (api/payment_intents/capture)
      handles: [pi_id, auth_hold]   # auth_hold single_use: spent even if
                                    # the outcome is unknown (I4)
    output: {payload: {status: 'succeeded', amount_received: money}, handles: []}
    request_invariants:
      final_capture: true         # default; false requires multicapture and
                                  # changes remainder semantics — pinned out
    idempotency: {mode: keyed, key_scope: run}
    confirmation:
      probe: get_payment_intent
      signal: "status == 'succeeded' && amount_received == amount_to_capture"
      async: {channel: webhook, deadline: 24h}   # payment_intent.succeeded /
                                                 # charge.captured as wake-ups
      sweep: {interval: 1h, escalate_after: 72h}
    compensation:
      action: create_refund       # chain R; full or partial
      window: "refundable any time up to original amount; customer sees credit
               ~5–10 business days; a refund can itself FAIL async ≤30d
               (docs.stripe.com/refunds) — see chain R"
    timeout: 60s
    latency_hint: 3s

  - id: cancel_payment_intent
    description: Release the hold (compensator for confirm_payment_intent).
    effect: compensate
    input:
      intent: {cancellation_reason: {type: enum[duplicate, fraudulent,
               requested_by_customer, abandoned], required: false}}
      handles: [pi_id]
    output: {payload: {status: 'canceled', canceled_at: timestamp}, handles: []}
    idempotency: {mode: keyed, key_scope: run}
      # NOT natural: repeat errors ("Returns an error if the PaymentIntent is
      # already canceled") — the error is classified as already-landed via the
      # probe, not swallowed. (§12 delta 6)
    confirmation:
      probe: get_payment_intent
      signal: "status == 'canceled'"
    timeout: 30s
    latency_hint: 1s

  # ───────── shared read probes (referenced by confirmations) ─────────
  - id: get_payment_intent
    effect: read
    input: {handles: [pi_id]}
    output: {payload: payment_intent, handles: []}
    idempotency: {mode: natural}
    timeout: 15s
    latency_hint: 500ms

  - id: get_payment_method
    effect: read
    input: {intent: {instrument: {type: string, required: true}}}
    output: {payload: payment_method, handles: []}
    idempotency: {mode: natural}
    timeout: 15s
    latency_hint: 500ms

  - id: detach_payment_method
    effect: compensate
    input: {intent: {instrument: {type: string, required: true}},
            handles: [customer_id]}
    output: {payload: payment_method, handles: []}
    idempotency: {mode: keyed, key_scope: run}
    confirmation: {probe: get_payment_method, signal: "payment_method.customer == null"}
    timeout: 15s
    latency_hint: 500ms

  # ───────────────────────── Chain R ─────────────────────────
  - id: create_refund
    description: Refund the captured charge, full or partial.
    effect: commit
    input:
      intent: {amount: {type: money, required: false},   # omitted = full
               reason: {type: enum[duplicate, fraudulent,
                        requested_by_customer], required: false}}
      handles: [pi_id]
    output:
      payload: {refund_id: string,
                status: enum[pending, succeeded, requires_action],
                pending_reason: enum[processing, insufficient_funds,
                                     charge_pending]?}
      handles: [refund_id]
    idempotency:
      mode: keyed, key_scope: run
      # LOAD-BEARING: multiple partial refunds are legal ("You can issue more
      # than one refund against a charge") — an unkeyed duplicate is a REAL
      # double refund, not an error. The key is the only dedup.
    confirmation:
      probe: get_refund
      signal: "refund.status == 'succeeded'"     # pending is NOT final
      async: {channel: webhook, deadline: 14d}
        # refund.updated / refund.failed / charge.refunded wake the
        # reconciler; failure can arrive up to ~30d ("can take up to 30 days
        # from the post date") — deadline is a product choice; past it →
        # escalate(operator) while the sweep continues. UNKNOWN-8.
      sweep: {interval: 24h, escalate_after: 30d}
    compensation:
      action: none
      ordering_note: "no API-level undo of a succeeded refund; a FAILED refund
        auto-returns funds to our Stripe balance (failure_balance_transaction)
        and requires a HUMAN alternative-refund arrangement per Stripe docs —
        gate(alternate_refund, audience: operator). Pre-commit protection:
        create_refund is itself the compensation path of capture; its
        entry protection is the calling policy."
    timeout: 30s
    latency_hint: 1s

  - id: get_refund
    effect: read
    input: {handles: [refund_id]}
    output: {payload: refund, handles: []}
    idempotency: {mode: natural}
    timeout: 15s
    latency_hint: 500ms
```

P3 mechanics: all commits here are keyed, so the spec does not *force*
correlation records — but they are cheap and the webhook/sweep channels need
them, so **every commit persists a correlation record (run_id, step,
intent snapshot hash, `metadata.run_ref` echoed into the Stripe object,
status=PENDING) before dispatch**, and every dispatched attempt runs under a
cancellation shield. This also covers the 24h key-pruning cliff: past
retention, the record + probe are the only safe reconciliation.

---

## 6. Verdict table (SKILL Step 4)

Domain rows first; then the generic skeleton (SPEC §3 rows 1–12) applies.
Ordering note honored: staleness/payload rows precede unconditional
transport success; a 200 carrying `status: canceled` can never classify ok.

### Idempotency rows (scope 2 — apply to every keyed step)

| # | Signal | Scope | Verdict | Evidence |
|---|---|---|---|---|
| K1 | `http:409` + `code:idempotency_key_in_use` | any keyed | `retry` (same key; the conflicting result "is not saved", so a later attempt executes or returns the winner) | docs.stripe.com/error-codes, /api/idempotent_requests |
| K2 | `http:400` + `type:idempotency_error` (key reused, params differ) | any keyed | `dead_end(permanent: true, reason: idempotency_config_bug)` — key derivation broke; surface loudly | docs.stripe.com/api/errors |
| K3 | Same-key retry returns a **replayed cached 5xx** (identical body twice) | keyed commit | `reconcile` — "Subsequent requests with the same key return the same result, including 500 errors"; the replay is a recording, not a fresh attempt; only the probe can resolve whether side effects exist | docs.stripe.com/api/idempotent_requests |
| K4 | Any commit re-dispatch where the original dispatch is **> 24h old** | keyed commit | `reconcile` (probe only; NEVER re-send the old key — retention is "at least 24 hours", after which the same key executes a NEW request = double charge) | docs.stripe.com/api/idempotent_requests |

K3 note for implementers: Stripe gives no client-visible marker
distinguishing a replayed 500 from a fresh 500; the deterministic rule is
attempt-count-based — first 5xx on a keyed commit → `retry` (same key, safe:
either replays or executes once); second identical 5xx → K3 `reconcile`.
This is the calibrated form of the generic row 6 for this provider.

### Chain M rows

| # | Signal | Step scope | Verdict | Evidence |
|---|---|---|---|---|
| S1 | `http:402 card_error card_declined`, decline_code ∈ {stolen_card, lost_card, fraudulent, merchant_blacklist} | confirm_payment_intent | `dead_end(permanent: true, present_as: generic_decline)` — Stripe: "Don't report more detailed information" | docs.stripe.com/declines/codes |
| S2 | same, decline_code ∈ {insufficient_funds, withdrawal_count_limit_exceeded, expired_card, incorrect_number, incorrect_cvc, invalid_account, card_not_supported, currency_not_supported, do_not_honor, generic_decline, call_issuer, card_velocity_exceeded, new_account_information_available, restricted_card, not_permitted, pickup_card} | confirm_payment_intent | `ok(empty)` → `gate(new_instrument)` — decline is an answer (P4); decline_code drives the gate's user-facing routing (another card vs fix CVC vs contact bank) | docs.stripe.com/declines/codes |
| S3 | same, decline_code = processing_error | confirm_payment_intent | `retry` (same key; "Payment needs to be attempted again") | docs.stripe.com/declines/codes |
| S4 | same, decline_code = authentication_required | confirm_payment_intent | `gate(sca_challenge)` — soft decline; issuer demands 3DS | docs.stripe.com/declines/codes, /payments/3d-secure/authentication-flow |
| S5 | `status == 'requires_action'` (2xx payload) | confirm_payment_intent | `gate(sca_challenge)` | docs.stripe.com/payments/paymentintents/lifecycle |
| S6 | Radar block: `outcome.type == 'blocked'` / advice `do_not_try_again` | confirm_payment_intent | `dead_end(permanent: true, present_as: generic_decline)` | docs.stripe.com/declines |
| S7 | `code:charge_expired_for_capture` | capture | `rewind(to: create_payment_intent)` — hold expired; PI is canceled (terminal), whole payment lineage re-mints; entry_gate re-consents | docs.stripe.com/error-codes, /payments/place-a-hold-on-a-payment-method |
| S8 | probe/webhook: `status == 'canceled'` or `charge.expired` or `payment_intent.canceled` | any consumer of auth_hold | `rewind(to: create_payment_intent)` (same rationale as S7) | docs.stripe.com/api/events/types |
| S9 | `code:charge_already_captured` | capture | `reconcile` — probe shows `succeeded` → treat original as landed (this is the double-call answer surfacing) | docs.stripe.com/error-codes |
| S10 | `code:payment_intent_unexpected_state` | confirm / capture / cancel | `reconcile` — the probe observes the actual state and the transition re-feeds it once (already captured → ok; canceled → rewind; etc.) | docs.stripe.com/error-codes |
| S11 | `code:amount_too_large` \| `amount_too_small` | capture | `gate(amount_consent)` — capturing a different amount than requested is a money decision, never auto-clamped | docs.stripe.com/error-codes, /api/payment_intents/capture |
| S12 | `code:payment_intent_authentication_failure` | confirm / verify_authorization | `gate(new_instrument)` — "Provide a new payment method to attempt to fulfill this PaymentIntent again" | docs.stripe.com/error-codes |
| S13 | `code:lock_timeout` | any | `retry` — "If you see this error intermittently, retry the request" (keyed steps: same key) | docs.stripe.com/error-codes |
| S14 | `http:429` / `code:rate_limit` | any (all commits keyed) | `retry` w/ exponential backoff ("We recommend an exponential backoff") — the unkeyed-commit 429 dilemma (SPEC row 5) never arises in this domain | docs.stripe.com/api/errors |
| S15 | cancel on already-canceled PI ("already canceled / isn't in a cancelable state") | cancel_payment_intent | `reconcile` → probe `status=='canceled'` → landed → ok | docs.stripe.com/api/payment_intents/cancel |
| S16 | `code:resource_missing` | any | `dead_end(permanent: true)` — wrong id; config/lineage bug | docs.stripe.com/error-codes |
| S17 | confirm-limit cancel: confirm returns canceled ("further calls… transition the PaymentIntent to the canceled state") | confirm_payment_intent | `dead_end(permanent: false, retry_after_hint: new_run)` — budget-analog on provider side; planner may start a fresh run/PI | docs.stripe.com/api/payment_intents/confirm |

### Chain R rows

| # | Signal | Step scope | Verdict | Evidence |
|---|---|---|---|---|
| R1 | `code:charge_already_refunded` | create_refund | `reconcile` — probe/list refunds matched via metadata.run_ref: ours → landed/ok; someone else's full refund → `dead_end(permanent: true, reason: already_refunded_elsewhere)` | docs.stripe.com/error-codes |
| R2 | refund payload `status == 'pending'` (incl. pending_reason: insufficient_funds) | create_refund | `ok` — advance; finality rides confirmation.async (pending is not failure) | docs.stripe.com/api/refunds/object |
| R3 | async observation (webhook/probe/sweep): `refund.status == 'failed'`, failure_reason ∈ {expired_or_canceled_card, lost_or_stolen_card, declined, insufficient_funds, unknown, merchant_request} | create_refund confirmation | `gate(alternate_refund)` (audience: operator) — funds are back in our balance (failure_balance_transaction); "you need to arrange an alternative way to provide your customer with a refund" | docs.stripe.com/refunds |
| R4 | async observation: failure_reason = charge_for_pending_refund_disputed | create_refund confirmation | `gate(alternate_refund)` with dispute annotation — Stripe: handle the dispute instead, avoid double reimbursement | docs.stripe.com/refunds |
| R5 | refund `status == 'requires_action'` | create_refund | `gate(refund_customer_action)` — customer must supply bank details; `next_action.display_details.expires_at` bounds the park | docs.stripe.com/api/refunds/object |
| R6 | `amount` > remaining refundable ("can't refund a total greater than the original charge amount") | create_refund | `repair(to: create_refund, fields: [amount])` — model may propose remaining amount; user approves via gate if product requires | docs.stripe.com/refunds |

…then the generic skeleton rows 1–12 (SPEC §3) close the table.

### known_unmatched

Consciously left to the fixed default rows (row 11 `reconcile` if a commit
was in flight, else row 12 `dead_end`):

- Auth/config-class: `api_key_expired`, `secret_key_required`,
  `testmode_charges_only`, `platform_api_key_expired`, 401/403/404 shapes not
  matched by S16.
- Decline codes not enumerated in S1–S4 (e.g. `try_again_later` (deprecated),
  `duplicate_transaction`, `issuer_not_available`, `reenter_transaction`,
  `security_violation`, `service_not_allowed`, `transaction_not_allowed`,
  `card_decline_rate_limit_exceeded`) — fall to default; promote to S2/S3
  rows only with production evidence per class.
- `payment_intent_incompatible_payment_method`, `setup_intent_*` family,
  `balance_insufficient` (Connect transfers — out of scope), `postal_code_invalid`.
- Refund `canceled` status arriving without our cancellation (Dashboard
  cancel by a human) — default rows; operator escalation via sweep.
- SPEC-example hearsay held out of the table per the `spec-example` rule:
  example B's `code:authorization_expired`, `code:challenge_expired`, and
  capture-time `code:insufficient_funds` were NOT corroborated by any Stripe
  doc (real signals are S7/S12/S11) and are therefore absent above.

---

## 7. Gates (SKILL Step 6.7)

```yaml
gates:
  authorize_consent:              # entry_gate on confirm_payment_intent
    audience: user
    payload: {amount: money, currency: string, instrument_summary: string,
              hold_days_hint: string}   # from capture_before once known
    outcomes:
      approve: ok
      decline: dead_end
    timeout: {after: 30m, verdict: dead_end}
    # Re-fires on EVERY arrival incl. the S7/S8 rewind replay: a fresh hold
    # needs fresh consent by construction (SPEC entry_gate; CONFORMANCE C8.4
    # analog — here the commit HAS a compensator, the gate is for the replay).

  new_instrument:                 # decline repair; also 3DS-failure repair
    audience: user
    payload: {decline_presentation: string,   # decline_code mapped per Stripe
                                              # rules — stolen/lost/fraudulent
                                              # presented as generic_decline
              suggestion: enum[use_other_card, fix_details, contact_bank]}
    outcomes:
      provide_new: {params: {payment_method_id: string}, bind: [instrument],
                    verdict: repair(to: confirm_payment_intent, fields: [instrument])}
        # pm_… is a provider-minted opaque value entering intent via the one
        # sanctioned gate path (SPEC §2.6): created by the user in their
        # client, validated as a well-formed unattached PaymentMethod ref.
      retry_same:  {verdict: rewind(to: confirm_payment_intent)}
        # only offered for incorrect_cvc / incorrect_number-class declines
        # (customer re-enters details → client re-tokenizes)
      give_up: dead_end
    timeout: {after: 30m, verdict: dead_end}

  sca_challenge:                  # 3DS: customer completes next_action in client
    audience: user
    payload: {next_action_type: enum[use_stripe_sdk, redirect_to_url]}
      # client_secret/next_action go to the CLIENT; never model-visible
    outcomes:
      completed: ok               # raising step's success path stands; run
                                  # advances to verify_authorization which
                                  # asserts the real post-challenge status
                                  # (authenticated → processing/requires_capture;
                                  # failure → requires_payment_method — the
                                  # precondition routes it; C6.4-safe)
      abandoned: dead_end
    timeout: {after: 30m, verdict: dead_end}
      # PRODUCT choice — no documented challenge-session TTL exists
      # (UNKNOWN-3); only floor: iOS SDK authenticationTimeout ≥ 5 minutes.

  amount_consent:                 # capture amount mismatch (S11)
    audience: user
    payload: {requested: money, amount_capturable: money}
    outcomes:
      capture_available: {set: {amount_to_capture: amount_capturable},
                          verdict: repair(to: capture, fields: [amount_to_capture])}
      abort: dead_end             # dead_end unwinds: cancel_payment_intent
                                  # releases the hold per compensation order
    timeout: {after: 30m, outcome: abort}

  alternate_refund:               # chain R terminal-failure gate
    audience: operator
    payload: {refund_id_alias: string, failure_reason: string,
              amount: money, failure_balance_transaction_alias: string}
    outcomes:
      resolved_out_of_band: ok    # operator arranged bank transfer/credit etc.
      write_off: dead_end
    timeout: {after: 7d, verdict: dead_end}   # escalation stays visible in
                                              # state=reconciling until answered

  refund_customer_action:         # non-card refunds needing bank details (R5)
    audience: user
    payload: {email_sent_to: string, expires_at: timestamp}
    outcomes:
      details_provided: ok        # Stripe transitions refund to pending itself
      expired: dead_end           # docs: customer no-response → refund failed
    timeout: {after: next_action.display_details.expires_at, outcome: expired}
```

All money/consent gates are `audience: user`/`operator` — never model (SPEC
§5, CONFORMANCE C6.5).

---

## 8. Policy (SKILL Step 5)

```yaml
policy:
  per_step:
    read: {attempts: 3, backoff: {base: 500ms, factor: 2, jitter: full, max: 8s}}
    mint: {attempts: 2, backoff: {base: 1s, factor: 2, jitter: full, max: 10s}}
    commit:
      keyed: {attempts: 3, backoff: {base: 2s, factor: 2, jitter: full, max: 30s}}
      # all Stripe commits are keyed; the unkeyed row is unreachable in this
      # domain. Constraint layered on top (K3/K4): a 2nd identical cached 5xx
      # short-circuits remaining attempts → reconcile; and no dispatch may
      # ever reuse a key whose first dispatch is >24h old (retention floor —
      # docs.stripe.com/api/idempotent_requests). Backoff ceiling 30s keeps
      # the whole retry envelope minutes-scale, safely inside retention.
    verify_authorization: {attempts: 6, backoff: {base: 2s, factor: 2,
                           jitter: full, max: 30s}}
      # doubles as the 'processing' poll; async methods can take longer —
      # exhaustion escalates per read rules (rewind if budget, else dead_end)
    compensate: {attempts: 3, backoff: {base: 2s, factor: 2, jitter: full, max: 30s}}
  per_chain:
    max_rewinds: 3
    max_repairs: 3            # raised from default 2 with justification:
                              # instrument repairs (declines) are the domain's
                              # dominant recovery; 2 gives users one bad card +
                              # one typo before dead_end — too tight for a
                              # payment product. Numbers remain OPEN per SPEC
                              # §2.4 — calibrate on the production decline mix.
    compensation_order: reverse   # capture → (refund); confirm → cancel;
                                  # attach → detach; create_customer → none
    wall_clock: 10m
      # heuristic max(3×Σ latency_hints≈36s, 2×60s slowest timeout)=2m, then
      # widened to 10m for reconcile probes + verify_authorization polling of
      # 'processing'; time parked at gates (3DS, consent) excluded per SPEC.
    gate_timeout: 30m
  escalation:
    - {audience: model,    may: [propose repair fields (amount_to_capture,
                                 refund amount), phrase decline/dead-end
                                 messaging from machine-readable reasons]}
    - {audience: user,     may: [answer authorize_consent, new_instrument,
                                 sca_challenge, amount_consent,
                                 refund_customer_action]}
    - {audience: operator, may: [resolve reconcile deadlines, alternate_refund,
                                 stuck-PENDING sweeps]}
```

Chain R runs under the same shape; its `create_refund` confirmation deadline
(14d) + sweep (30d) intentionally dwarf `wall_clock` — the run sits in
`state=reconciling`, which SPEC §2.5 permits for days; wall_clock covers
active execution only.

---

## 9. Invalidation walkthroughs (SKILL Step 6.9)

### W1 — Hold expiry discovered at capture (the dangerous rewind)

Setup: consent given, 3DS passed, run parked overnight at `amount_consent`;
next morning the operator answers; capture dispatches. Card network window
had elapsed → Stripe already canceled the PI.

1. `capture` attempt-1 → signal `code:charge_expired_for_capture` (or the
   `payment_intent.canceled` / `charge.expired` webhook raced us; either
   way rows S7/S8 match before generic transport-success — a 4xx here can
   never be mistaken for a config bug).
2. Verdict `rewind(to: create_payment_intent)`; rewind budget 3→2.
3. Invalidation (I1, atomic): `auth_hold` is dead (it reported); its minting
   ancestor `pi_id` is *also* dead by the same signal (the PI is canceled —
   terminal), and `rewind.to` re-mints it. Dying set: {pi_id, auth_hold} +
   every cached payload derived from them (`verify_authorization`'s
   `amount_capturable`, `capture_before`). Survivors: `customer_id` (upstream,
   inputs unchanged — I6), `instrument` (intent), all consent *history* in
   the trace (facts), but NOT consent *validity*: —
4. Cursor = `create_payment_intent` → mint fresh `pi_id` → cursor reaches
   `confirm_payment_intent` whose **entry_gate `authorize_consent` re-fires
   by construction** (a brand-new hold on the user's card; stale consent
   never carries over a replay — SPEC §2.1). C8.4-style assertion holds on
   this path.
5. Issuer may demand 3DS afresh (S5 → `sca_challenge` again) — routine, not
   exceptional.
6. `verify_authorization` re-reads the NEW `amount_capturable`; the old
   `amount_consent` answer was bound to intent `amount_to_capture`, which
   survives as intent (I6: repair target was not touched) — but capture
   re-validates it against the new hold via S11 if it no longer fits.
7. I2: any stored authorization payloads are replaced, keyed by the new
   `pi_id` lineage — a probe of the old PI can never be read as the new one.

I4 check: `auth_hold` is `single_use` — it was consumed by the capture
attempt whose outcome was known-failed; the replay re-mints it; the old PI id
is never re-presented to capture.

### W2 — Decline at confirm → instrument repair (I6 scope precision)

1. `confirm_payment_intent` attempt-1 → `http:402`, decline_code
   `insufficient_funds` → row S2 → `ok(empty)` → `gate(new_instrument)`.
2. User provides `pm_new…` → outcome `provide_new` binds `instrument` →
   `repair(to: confirm_payment_intent, fields: [instrument])`; budgets:
   rewinds 3→2, repairs 3→2.
3. I6: the repair invalidates handles minted by the target step and
   downstream: `auth_hold` (nothing landed — the decline meant no hold, but
   the *slot* and any cached confirm payload die). **`pi_id` SURVIVES**: its
   minting step `create_payment_intent` does not take `instrument` in its
   `input.intent` — statically provable from the config. This matches
   Stripe's own model exactly: "the PaymentIntent's status returns to
   `requires_payment_method` so that the payment can be retried" — same PI,
   new payment method (docs.stripe.com/payments/paymentintents/lifecycle).
4. Cursor = `confirm_payment_intent`; entry_gate `authorize_consent` re-fires
   (new instrument = new consent context); confirm dispatches with the SAME
   `pi_id`, new instrument, and a **fresh idempotency key** (new attempt
   identity — params changed, so reusing the old key would 400
   `idempotency_error` by K2; key derivation is `run_id + step + intent_hash
   + attempt_lineage`, code-owned).
5. `refund_id` / landed commits: none yet; I5 vacuously holds.

### W3 (chain R) — refund fails asynchronously after `ok`

1. `create_refund` → 200, `status: pending` → R2 → `ok`; correlation record
   PENDING; run enters `state=reconciling` under `confirmation.async`.
2. Day 6: `refund.failed` webhook (or the 24h sweep probe) observes
   `status == failed`, `failure_reason: expired_or_canceled_card`,
   `failure_balance_transaction` present (funds returned to our balance).
3. The observation re-feeds the cause signal through the table **once**
   (SPEC reconcile transition / C1.7 analog): row R3 → `gate(alternate_refund,
   audience: operator)`; run parks; no invalidation — the failed refund is a
   landed *fact* (I5), its record stays in `commits_landed` annotated failed.
4. Operator resolves out of band → `ok` → run done; or `write_off` →
   `dead_end` with machine-readable reason. No API re-dispatch of the refund
   ever happens from this path (a NEW refund would be a NEW run).

---

## 10. UNKNOWNs & assumptions (SKILL Step 6.10)

| # | Item | Question for a human / follow-up |
|---|---|---|
| UNKNOWN-1 | `attach_payment_method` double-call behavior undocumented (already-attached PM: error? no-op?) | Empirically test in sandbox; adjust idempotency note. Tag stays `assumed` until observed. |
| UNKNOWN-2 | Bare attach vs SetupIntent: Stripe recommends SetupIntent/`setup_future_usage` for future-use optimization | Product decision: adopt SetupIntent (adds its own requires_action gate) or accept degraded reuse? |
| UNKNOWN-3 | 3DS challenge-session TTL and max `requires_action` park duration are not documented at API level (only iOS SDK floor: `authenticationTimeout` ≥ 5 min) | Gate timeout 30m is a product choice, not provider fact. Validate empirically; keep as gate timeout, never a staleness timer (P2). |
| UNKNOWN-4 | Confirm-attempt ceiling is documented to exist but "variable"; value unknown | S17 handles the cancel outcome; confirm our `max_repairs` keeps us under it in practice. |
| UNKNOWN-5 | Customer dedup: assumed Stripe never dedupes by email; orphan customers assumed harmless | Confirm no billing/compliance cost to orphan Customers; decide app-side dedup lookup. |
| UNKNOWN-6 | Probe queryable-lag after capture/confirm assumed zero (synchronous card rails) | Confirm no read-after-write lag on GET /payment_intents in production. |
| UNKNOWN-7 | Replayed-500 vs fresh-500 indistinguishable client-side; K3 uses attempt-count heuristic | Ask Stripe support whether any header marks idempotent replays (e.g. `Idempotent-Replayed`) — if one exists, sharpen K3. |
| UNKNOWN-8 | Chain R `async.deadline` 14d and sweep 30d are product placeholders bracketing Stripe's "up to 30 days" failure window | Calibrate against real refund-failure latency distribution. |
| UNKNOWN-9 | No sandbox mechanism to fast-forward auth-hold expiry (test clocks are Billing-only) | Conformance C2.x tests inject `charge.expired` / `charge_expired_for_capture` via HTTP mock; accept that real-expiry e2e is untestable pre-prod. |
| UNKNOWN-10 | `pi_id` staleness beyond terminal-state detection: unconfirmed PIs have no documented expiry | Treat as durable-until-canceled; empty additional `detect` accepted → falls to default rows (fail closed) as SKILL Step 2 prescribes. |
| UNKNOWN-11 | Extended-authorization eligibility (IC+ pricing, merchant category) for THIS account | If eligible, add `request_extended_authorization: if_available` as a request_invariant variant and read `capture_before` as the ttl_hint source. |

---

## 11. Boundary recap (SKILL Step 6.11)

Every model touchpoint in these chains, checked against SPEC §5:

| Touchpoint | SPEC §5 allowance | Note |
|---|---|---|
| Phrase decline outcomes to the user (mapping decline_code → presentation, incl. the mandated generic_decline masking for stolen/lost/fraudulent) | "Phrasing outcomes" | Presentation only; classification stayed in the table (S1/S2). |
| Propose `amount` for R6 refund repair (remaining refundable) | "Proposing values for repair fields" | Code validates against remaining refundable; user gate if product requires. |
| Propose retry-vs-give-up *suggestion text* inside `new_instrument` gate payload | "Phrasing outcomes" | The verdict space is fixed by the gate's declared outcomes. |
| Decide what to do AFTER dead_end (e.g. offer invoice instead of card) | "model (planner), outside this spec" | New chain; budgets do not carry. |
| — Selections: **none** | I3/`select` unused | No options-collection step exists in either chain; no rematch specs needed. |
| — Gate answers: none by model | Money/consent gates user/operator only | authorize_consent, new_instrument, sca_challenge, amount_consent, alternate_refund all human. |
| — Handles: never | Handles code-owned | `pm_…` enters intent solely via the sanctioned gate-params path (§2.6 carve-out), validated. |

No handle field is model-repairable; no signal routes to a model decision;
budgets live only in §8 config.

---

## 12. Worked example B vs real Stripe — deltas (deliverable)

Every row of SPEC §7 was treated as `spec-example` hearsay and re-derived.
Where the real API contradicts or refines example B:

1. **`status: declined` is not a real PaymentIntent status, and declines are
   not a 2xx payload.** B's `authorize` outputs
   `status: enum[…, declined]` with `empty.detect: "status == 'declined'"`.
   Real Stripe: a decline surfaces as an **HTTP 402 `card_error`** with
   `code: card_declined` + `decline_code`, and the PI transitions to
   `requires_payment_method` (docs.stripe.com/error-codes,
   /payments/paymentintents/lifecycle). The P4 valid-empty *concept* holds;
   its transport contradicts B and strains the `empty.detect`
   payload-predicate grammar (§13).
2. **The decline-repair target is wrong in B.** B repairs
   `to: authorize`, implying a fresh intent/hold mint. Real Stripe keeps the
   SAME PaymentIntent alive and reusable ("status returns to
   `requires_payment_method` so that the payment can be retried") — the
   repair replays `confirm` on the surviving `pi_id` (W2). B's shape
   discards a durable object the provider deliberately preserves.
3. **Hold expiry: "~7d" is an oversimplification with money consequences.**
   Real windows are network- and channel-dependent: Visa online
   merchant-initiated is effectively **4 days 18 hours**; card-present
   Mastercard/Amex/Discover are **2 days**; extended auth reaches ~30 days
   (Visa 29d18h); Japan/JPY 30 days; non-card methods range 7–30 days.
   Stripe's own guidance: "Rely on the `capture_before` field… because these
   rules can change without prior notice." Also missing from B: **expiry
   auto-cancels the PaymentIntent** (terminal), so the refresh target is PI
   *creation*, not re-authorize-in-place.
4. **B's staleness codes don't exist.** `code:authorization_expired` and
   `code:challenge_expired` appear in no Stripe error-code reference. Real
   signals: `charge_expired_for_capture`, `charge.expired` /
   `payment_intent.canceled` events, probe `status == 'canceled'`, and 3DS
   failure = `payment_intent_authentication_failure` + return to
   `requires_payment_method`. B's `client_challenge` handle is also
   mis-modeled: the challenge artifact is consumed by the customer's client
   inside the gate, never by a later chain step — it isn't a chain handle.
5. **B's P6 capture row is fabricated for Stripe.** "insufficient_funds at
   capture (capture > auth or funds moved)" — real Stripe forbids overcapture
   outright (`amount_to_capture` "must be less than or equal to the original
   amount"; `amount_too_large` 400, deterministic), and capturing ≤ the
   authorized amount does not hit insufficient_funds. The amount-consent gate
   survives (S11) but hangs off a validation error, not a funds decline.
6. **`release_hold` is not a natural no-op.** B: "releasing a released hold:
   no-op". Real: canceling an already-canceled PI **returns an error**
   ("Returns an error if the PaymentIntent is already canceled") — the
   repeat must be classified as already-landed via the probe (S15), not
   assumed silent.
7. **"Ambiguous failure = plain retry; reconcile nearly vestigial" is too
   strong.** Same-key retry is indeed safe — but Stripe **replays cached
   errors including 500s**, so a same-key retry after an executed-but-500
   attempt returns the *recording*, resolving nothing (K3 → reconcile), and
   keys are **pruned after ~24h**, after which a same-key "retry" is a fresh
   execution — the canonical double-charge (K4 → probe only). Reconcile is
   thinner here than in example A, but it is load-bearing at exactly the two
   places B waves off.
8. **B's `create_customer` `key_scope: intent` "dedupes across runs" is
   unsound.** Idempotency keys are retained "at least 24 hours" then pruned —
   intent-scoped keys silently stop deduping after a day. Cross-run dedup
   must be an application-side lookup; this compile uses `key_scope: run`.
9. **Refund modeling in B is materially incomplete.** B: probe `succeeded`,
   webhook 72h, note "funds settle T+2, refund before settlement ≈ void" —
   the T+2/void claim appears nowhere in Stripe's refund docs (uncorroborated
   hearsay, dropped). Missing entirely from B: refunds **fail asynchronously
   up to ~30 days** after successful creation (`refund.failed`,
   `failure_reason`, `failure_balance_transaction`, funds auto-returned),
   multiple partial refunds are legal (making the key load-bearing, not
   belt-and-braces), `requires_action`/`pending_reason` states, and the
   terminal remedy is a human arranging an out-of-band alternative
   (operator gate), not an API action.
10. **B's `sca_challenge` 15m timeout rationale ("challenge session dies
    anyway") is undocumented.** No API-level challenge TTL exists; the only
    documented figure is the iOS SDK's ≥5-minute floor. The timeout is a
    product decision (UNKNOWN-3), and post-challenge re-entry differs by
    `confirmation_method` (`manual` bounces to `requires_confirmation` and
    needs an explicit server re-confirm — absent from B).
11. **Refinement, not contradiction: B's fraud row.** Real Stripe splits it:
    issuer `fraudulent`/`stolen_card`/`lost_card` declines AND Radar
    `blocked` outcomes, with a documented *presentation* mandate (mask as
    generic_decline) that belongs in the gate payload/phrasing layer, which B
    doesn't capture.

---

## 13. Spec/skill friction notes (this domain)

- **Valid-empty transported as an HTTP error.** `empty.detect` is defined as
  a *payload predicate* on a successful call; Stripe's canonical valid-empty
  (card declined) arrives as HTTP 402 with an error object. This compile
  declares the `empty` block for documentation intent but implements routing
  via domain rows S1–S5. The spec could allow `empty.detect` to name a
  signal, or bless "error-transported valid-empty" explicitly.
- **Post-ok revocation (async refund failure).** `confirmation.async` +
  sweep + the reconcile cause-re-feed absorb it (W3), but the fit is
  imperfect: the spec's `reconcile` is defined for *lost responses*, while
  here the response was received and `ok`; the run occupies
  `state=reconciling` for up to 30 days by stretching "finality arrives
  out-of-band". A first-class "landed-but-revocable commit" notion (finality
  window on the commit, revocation → cause re-feed) would name what this
  config does by convention.
- **Keyed-everything provider inverts the spec's emphasis.** The heavy P3
  machinery (correlation records for unkeyed commits) is never *forced*, yet
  the compile still wants it — for webhook correlation and for the 24h
  retention cliff. Conversely, the spec has no vocabulary for **idempotency
  retention windows**: `idempotency.mode: keyed` is treated as timeless, but
  "keyed for 24h, unkeyed thereafter" is the real contract and drives K4.
  A `key_retention` field under `idempotency` would make this declarable
  instead of a policy footnote.
- **Replayed cached errors.** The spec's retry row assumes a keyed retry
  re-executes idempotently; Stripe replays the *recording* of the first
  outcome, including 500s. The retry/reconcile boundary therefore needs an
  attempt-count heuristic (K3) the spec doesn't anticipate.
- **Rich documentation flips the evidence regime.** The spec (born from an
  empirically-noted provider) leans on `observed`; here nearly everything is
  `documented` and the scarce commodity is *absence* markers (undocumented
  double-call behavior, undocumented TTLs). The SKILL's tag set handles it,
  but a `documented-by-omission` tag would be more honest than `assumed` for
  UNKNOWN-1/-3-class facts.
- **No selections.** Both chains have zero `select` pseudo-steps; rematch,
  reselect, and I3 are dead weight here — evidence that the machinery is
  optional-by-config rather than ceremony, which is the right outcome.
- **Test-clock gap vs conformance.** C2.x wants expiry induced; the provider
  offers no time-travel for holds (test clocks are Billing-only), so
  conformance runs against mocks injecting the documented signals — worth a
  CONFORMANCE note that "induce expiry" may legitimately mean "inject the
  documented expiry signal".

---

## 14. Self-check (SKILL Step 7)

- [x] Every commit/compensator declares `confirmation` — create_customer
      (by_key_replay + justification), attach (content probe), confirm
      (probe+async+sweep), capture (content probe+async+sweep), cancel
      (probe), detach (probe), create_refund (probe+async+sweep). No unkeyed
      commits exist; attempts=1 rule not triggered.
- [x] P3 correlation record + cancellation shield adopted on ALL commits
      (voluntarily; required for webhook/sweep correlation + 24h key cliff).
- [x] Selection pseudo-handles: none exist; no rematch specs needed (checked,
      not skipped).
- [x] In-place commits (`mutates`: attach, capture) have content-based
      confirmation signals and explicit compensation stances.
- [x] Every handle a commit consumes is durable (`customer_id`, `pi_id`
      terminal-state-detected) or has `staleness.detect` (`auth_hold`).
- [x] `single_use`: `auth_hold` consumed by exactly one step (capture).
- [x] Model touchpoints = phrasing + repair-field proposals only (§11); no
      handle model-repairable; no signal routed to model.
- [x] Every documented error code encountered in the corpus is mapped
      (K1–K4, S1–S17, R1–R6) or consciously listed in `known_unmatched`.
- [x] Every gate has audience, outcome→verdict map, timeout (§7).
- [x] Empty-result routes exist where Q9 said valid (decline → gate;
      transport strain documented).
- [x] Invalidation walkthroughs traced by hand against `derived_from` (§9),
      including the I6 survival proof for `pi_id` in W2.
- [x] Payload business blockers are `preconditions`
      (verify_authorization status routing; capture_method completeness
      assertion), not overloaded `empty`.
- [x] No TTL used as an enforcement timer — `capture_before` is planning-only;
      expiry is discovered via S7/S8 signals (P2).
- [x] No budget or retry limit in prompt text — all in §8 config.
- [x] Compensation ordering stated (`reverse`; refund-failure remedy is an
      operator gate; no replace/rebook flow exists in scope).

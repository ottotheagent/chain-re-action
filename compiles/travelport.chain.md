# Travelport TripServices JSON APIs (Flights) — Chain Compile

Created: 2026-07-27
Last Updated: 2026-07-27
Compiled against: `SPEC.md` **Draft v0.3.2 (pre-implementation)** + `SKILL.md` (chain-compile) + `CONFORMANCE.md` (v0.3.2)
Status: **DRAFT FOR REVIEW — one BLOCKED-adjacent gap flagged** (see UNKNOWN-1: no documented
locator-free probe for a lost `commit_workbench` response; compile proceeds because a probe
*procedure* exists — correlation record + workbench-state inference + operator sweep — but a human
must confirm the operator path before implementation).

Evidence corpus: Travelport public documentation only —
`https://developer.travelport.com` (TripServices Flights guides + references) and the legacy
webhelp at `https://support.travelport.com/webhelp/JSONAPIs/` / `.../webhelp/TripServices/`.
Tags: `documented` = stated in official guide/reference (URL cited per claim);
`assumed` = my inference, listed in §10. `observed` unused (no API access in this exercise).
SPEC worked examples are `spec-example` hearsay and were **not** used as evidence anywhere in
this file; where SPEC example A's domain overlaps (air booking), every row here is independently
sourced from Travelport pages.

---

## 1. Chain summary (multi-scope)

Travelport TripServices Flights is a **workbench-session** API: all mutation happens by minting a
server-side workbench, loading it (offer, travelers, payment), and committing it. The full
purchase is a **two-stage commit**: commit #1 (workbench commit) creates a **held booking** — a
real airline reservation (PNR) with **no payment taken** and a **business TTL** (ticketing time
limit, "usually within 24 hours"); commit #2 (a *second* workbench built from the reservation
locator, plus payment, committed) **issues the ticket** and takes the money. Post-ticketing
servicing (void / refund / exchange) also runs through post-commit workbenches, with sharply
different rails for GDS vs NDC content.

Three scopes, one shared verdict-table core and shared handles:

- **Chain A — book + ticket** (the two-stage purchase):
  `search → select → air_price → create_workbench → add_offer → add_traveler →
  commit_workbench [COMMIT #1 → held booking] → create_postcommit_workbench →
  get_reservation → add_fop → add_payment → commit_ticketing [COMMIT #2 → ticket]`
- **Chain B — void / refund** (compensation-shaped; also runs standalone when the user cancels
  later): GDS `list_tickets → gate → void_ticket`; NDC `create_postcommit_workbench →
  refund_quote → cancel_ticket`.
- **Chain C — exchange** (GDS rail modeled; NDC deltas noted):
  `exchange_eligibility → create_postcommit_workbench → exchange_search → select_exchange →
  add_modify_offer → [add_fop_x → add_payment_x] → commit_exchange [COMMIT, mutates the PNR
  in place]`.

Rail note: GDS and NDC are compile-time **variants** (different endpoints, FOP timing, refund
support, Instant Pay). This document writes the GDS rail as the primary config and boxes NDC
deltas; an implementation ships two configs generated from one template. This is itself a
finding — see §12 friction notes.

---

## 2. Fit-test verdict

A chain is the right abstraction; every trigger property fires:

- **Handles that expire**: search results cached 12 min (GDS) / 34 min (NDC)
  (`documented` — Booking Guide: "Travelport retains a search from any of the search APIs in
  cache for 12 minutes for GDS content and 34 minutes for NDC content",
  https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/Book/BookingGuide.htm);
  workbench "valid for 30 minutes and must be committed within that time window or it expires"
  (`documented` — https://developer.travelport.com/docs/flights/guides/ticketing-guide).
- **Commits**: held booking (airline-confirmed reservation) and ticket issuance (payment) —
  durable, user-visible, money-adjacent. Two of them, dependent.
- **No idempotency key anywhere** (elicitation Q4): unkeyed commits ⇒ P3 machinery is
  load-bearing.
- **Empty is an answer**: "NO OFFERS FOUND FOR API CONTENT" is a documented result
  (https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/General/ErrorList.htm).
- **Compensation is a business action**: void window ("generally... within the same day it was
  issued, usually up to midnight local agency time") vs refund-with-penalty vs
  refund-not-supported-on-GDS — fees and windows vary with time and rail
  (https://developer.travelport.com/docs/flights/guides/exchange-refund-and-void-guide).
- **Mid-chain human decisions by business rule**: two-step commit price/schedule-change warnings,
  payment consent, remedy choice.

None of the Step-0 refusal conditions hold (multiple dependent mutating steps; provider does not
run recovery for you; not a poll-one-status-field API).

---

## 3. Elicitation log

Source tags: `documented(URL)` / `assumed`. Answers keyed to SKILL Step-1 questions.

### 3.1 Endpoints inventory (context for everything below)

`documented` — endpoints list
(https://developer.travelport.com/docs/flights/general/flights-api-endpoints and
https://support.travelport.com/webhelp/jsonapis/airv11/content/air11/General/AirEndpointsList11.htm),
base `https://api.travelport.net/11/air/`:

| Step | Method + path |
|---|---|
| Search | `POST catalog/search/catalogproductofferings` |
| Next Leg Search | `POST catalog/search/catalogproductofferings/buildnext` |
| AirPrice (reference) | `POST price/offers/buildfromcatalogproductofferings` |
| Create workbench | `POST book/session/reservationworkbench` |
| Retrieve workbench | `GET book/session/reservationworkbench/{workbenchID}` |
| Discard workbench | `DELETE book/session/reservationworkbench/{workbenchID}` |
| Add offer (reference) | `POST book/airoffer/reservationworkbench/{workbenchID}/offers/buildfromcatalogofferings` |
| Add traveler | `POST book/traveler/reservationworkbench/{workbenchID}/travelers` |
| Add form of payment | `POST payment/reservationworkbench/{workbenchID}/formofpayment` |
| Add payment | `POST paymentoffer/reservationworkbench/{workbenchID}/payments` |
| **Commit workbench** | `POST book/reservation/reservations/{workbenchID}` |
| Post-commit workbench | `POST book/session/reservationworkbench/buildfromlocator` |
| Reservation retrieve | `GET book/reservation/reservations/{LocatorCode}` |
| Reservation cancel (GDS, held) | `POST receipt/reservations/{LocatorCode}/receipts` |
| Ticket list | `GET receipt/reservations/{LocatorCode}/receipts` |
| Ticket retrieve | `POST ticket/tickets/getbylocator` / `GET ticket/tickets/{ticketID}` |
| Ticket void (GDS, single) | `PUT ticket/tickets/updatestatus/{ticketID}` |
| Batch void | `POST documents/void` |
| Refund Quote (NDC) | `POST book/airoffer/reservationworkbench/{workbenchID}/offers/canceloffer` |
| Cancel (NDC) | `POST receipt/reservation/{workbenchID}/receipts?OfferIdentifier={OfferID}` |
| Exchange eligibility (GDS) | `eligibility/ticketchangeeligibilities` (method: endpoints list says POST; the API-reference page reads as GET — see UNKNOWN-11) |
| Exchange search (GDS) | `POST exchangesearch/catalogofferingsairchange` |
| Add/Modify offer (exchange) | `POST book/offer/reservationworkbench/{workbenchID}/offers/buildfromcatalogofferings` |
| Reshop / Reprice / Modify (NDC) | `POST change/catalogofferingsairchange` / `POST reprice/reservationworkbench/{workbenchID}/offers/buildfromcatalogofferings` / `POST modify/reservations/{LocatorCode}` |
| Standalone Reprice (NDC) | `POST reprice/reservationworkbench/{workbenchID}/offers/buildfromoffer` |

### 3.2 Q1–Q5 for COMMIT #1 — `commit_workbench` (create held booking)

**Q1 — double call, same inputs, what exists after?**
The commit consumes the workbench: for the ticketing variant the guide states commit "issues the
ticket..., **discards the workbench**, and returns a ticket number" (`documented` —
https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/Ticketing/TicketingGuide.htm);
the workbench-not-found error class exists ("RESERVATION WORKBENCH ID DOES NOT EXIST",
SourceCode 4100, `documented` —
https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/General/ErrorList.htm).
So a literal second `POST book/reservation/reservations/{workbenchID}` after a *successful*
first commit fails on 4100 — the workbench id is structurally single-use. BUT re-running the
chain (new workbench, same intent) creates a **second held booking**: a real airline reservation,
inventory-holding and user-visible ("The booking is confirmed by the airline but not yet
ticketed", `documented` —
https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/Book/BookingGuide.htm).
⇒ effect = **commit**; the single-use workbench gives replay protection only *within* one
workbench lineage, not across re-mints (`assumed` for the "commit on a discarded workbench
returns 4100 specifically" — the exact code on this path is not stated; see UNKNOWN-6).

**Q2 — process dies mid-call: what might exist? cost/inventory/user-visible?**
A held booking may exist: airline-confirmed PNR holding inventory, visible to the traveler via
the carrier. No payment is taken at this stage (GDS FOP optional at booking; "Adding FOP for NDC
is not supported in booking, only at ticketing or in the Instant Pay workflow", `documented` —
BookingGuide.htm above). ⇒ commit, Q3 mandatory. Mitigation of blast radius: the held booking
self-expires — "will expire if ticketing doesn't take place within a certain time, usually
within 24 hours of booking" (`documented` — BookingGuide.htm). An orphaned held booking is
therefore consequential-but-self-limiting; it is still a commit (SPEC §2.1: consequence, not
self-expiry, decides).

**Q3 — how to find out whether an ambiguous attempt landed?**
- If the response was received: locator in `Receipt/Confirmation/Locator` (`documented` —
  BookingGuide.htm) → probe = `GET book/reservation/reservations/{LocatorCode}` (`documented` —
  Reservation Retrieve,
  https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/Book/ReservationRetrieveGuide.htm).
- If the response was **lost** (the P3 case): there is **no documented
  search-reservations-by-passenger/date endpoint** in the Flights API set. Documented partial
  probe: `GET book/session/reservationworkbench/{workbenchID}` (Retrieve Workbench,
  `documented` endpoint — AirEndpointsList11.htm). Inference (`assumed`): workbench still
  retrievable with contents intact ⇒ commit did not land (safe to re-commit the SAME
  workbench); workbench gone (4100) ⇒ ambiguous between "committed successfully" and
  "expired at the 30-min boundary" ⇒ operator escalation with the correlation record. This
  ambiguity is the design's weakest point — UNKNOWN-1.
- Eventual-consistency lag: for ticketless carriers reservation content "may be delayed for
  several hours after booking" (`documented` — ReservationRetrieveGuide.htm); commit itself may
  return "Carrier locator code not returned by carrier/provider. Please try to retrieve later"
  (`documented` — BookingGuide.htm) ⇒ probe retries must tolerate lag.
- Webhook: none documented for TripServices Flights booking finality (`assumed` absent) ⇒
  `async.channel: poll`.

**Q4 — client idempotency key?**
**No.** No idempotency-key header/field appears in any TripServices request reference consulted.
`transactionId` is response-side only: "This unique system-generated ID is generated by the
TripServices APIs and returned automatically", "The same value is returned for all API calls
that are connected in a workflow"; purpose is troubleshooting/tracking (also surfaced as the
`E2EtrackingId` response header; requests may send a custom `traceID` header "to serve as their
own tracking number") (`documented` —
https://developer.travelport.com/docs/flights/guides/flights-general-guide). It is **tracing,
not dedup**: it cannot be supplied on a dispatch to collapse duplicates. ⇒
`idempotency.mode: none` on every commit in this file; `attempts: 1` forced.

**Q5 — naturally idempotent without a key?**
Within one workbench: effectively yes-by-construction (workbench consumed/discarded ⇒ replay
fails closed on 4100) — but that is fail-closed, not replay-the-original-outcome, so it does
NOT qualify as `mode: natural` for retry purposes. Across workbenches: no — same intent through
a fresh workbench books again. ⇒ `mode: none`.

### 3.3 Q1–Q5 for COMMIT #2 — `commit_ticketing` (issue ticket, take payment)

**Q1 — double call?** Same workbench-consumption mechanics: commit "issues the ticket (and EMD
if applicable), discards the workbench, and returns a ticket number" (`documented` —
TicketingGuide.htm webhelp). Second call on same workbench id → 4100-class failure. Re-running
the whole ticket sub-chain (fresh post-commit workbench on the same locator, payment, commit)
against an already-ticketed booking: behavior not documented (`assumed`: rejected by
carrier/host — but NOT relied upon; the chain re-probes before any re-approach). ⇒ commit.

**Q2 — process dies mid-call?** A ticket may exist and the FOP may be charged — money moved,
maximally consequential. ⇒ commit, Q3 mandatory.

**Q3 — probe?** Strong and fully documented, because the **locator is known before dispatch**
(the post-commit workbench was built from it):
- Reservation Retrieve: "If the reservation has been ticketed, returns ticket number/s in the
  Document/Number object" (`documented` —
  https://developer.travelport.com/docs/flights/guides/ticketing-guide and
  TicketingGuide.htm webhelp).
- Ticket List: `GET receipt/reservations/{LocatorCode}/receipts` — "Returns a list of all ticket
  numbers on a reservation" (`documented` — same pages).
- Ticket Retrieve for detail (`documented`).
Signal: a Document/Number (ticket) exists whose itinerary matches the held booking's offer.
Content-based (this commit `mutates` the reservation): presence of ticket receipts on the SAME
locator, not mere reservation existence.

**Q4 — idempotency key?** None, as §3.2 Q4. `transactionId` tracing-only. ⇒ `mode: none`,
`attempts: 1`.

**Q5 — natural idempotence?** No (fresh workbench re-issues / double-charges risk). ⇒
`mode: none`.

### 3.4 Handles & staleness (Q6–Q8b)

- **Search results**: response returns offer/product/brand/T&C identifiers
  (`CatalogProductOffering/id` "o1", `ProductRef` "p0", ...) consumed by Next Leg Search,
  Flight Specific Search, AirPrice-reference and Add Offer-reference (`documented` —
  https://developer.travelport.com/docs/flights/guides/flights-search-guide). Caching is opt-in:
  "If a journey-based Search request does not send `offersPerPage`, any subsequent reference
  payload requests fail" (`documented` — same) ⇒ `request_invariants: offersPerPage`. Lifetime:
  12 min GDS / 34 min NDC cache (`documented` — BookingGuide.htm) ⇒ `ttl_hint`. Staleness signal
  on use: "OFFER ID AND/OR PRODUCT ID DOES NOT EXIST" (StatusCode 200, SourceCode 4101,
  `documented` — ErrorList.htm). Cheapest re-mint: re-run `search`.
- **Selection**: downstream results couple to the pick (AirPrice prices *that* offer; Next Leg
  Search "sends a payload with several identifiers from the preceding Search response... for
  the offer and product you want to select", `documented` — flights-search-guide) ⇒ selection
  pseudo-handle with `derived_from: [search_results]`. Q8b rematch key: provider offer ids are
  request-scoped ("o1", "p0") and certainly not stable across re-search ⇒ intent-level key =
  carrier + flight numbers + departure/arrival datetimes + brand name + cabin + total price
  (`assumed` — composition is my choice; constituent fields are documented response content).
- **Priced offer** (AirPrice response): returns an "Identifier for the response, and an OfferID
  object" (`documented` —
  https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/Price/AirPricingGuide.htm);
  carries `TermsAndConditions` incl. "last date and time to ticket the offer" (`documented` —
  same). Expiry signal on use: none dedicated documented; 4101-class reference failure
  (`assumed` mapping) ⇒ falls to default rows when unattributable. Refresh: re-run `air_price`.
- **Workbench**: "Identifier" (UUID) that "must be sent in all subsequent requests for this
  workbench session" (`documented` —
  https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/Workbench/APIRef_WorkbenchNewCreate.htm).
  Lifetime: 30 min, then expires (`documented` — ticketing-guide dev portal + BookingGuide.htm).
  Staleness signals: 4100 "RESERVATION WORKBENCH ID DOES NOT EXIST", 4367 "HOST SESSION HAS
  EXPIRED. IGNORE AND REINITIATE WORKBENCH AND RETRY", 4434 "RESERVATION WORKBENCH SESSION MUST
  BE ACTIVE. REINITIATE SESSION" (`documented` — ErrorList.htm; note 4367's message *prescribes*
  the recovery: re-initiate + retry ⇒ rewind). Disposal: `DELETE .../reservationworkbench/{id}`
  ("Discard workbench... explicitly removing uncommitted data", `documented` — BookingGuide.htm).
  Uncommitted workbench external effect: none indicated ("no indication that uncommitted
  workbenches affect external systems") — `assumed` inert ⇒ mint, not commit (UNKNOWN-5).
- **Reservation locator**: "A successful booking returns a unique six-digit alphanumeric number
  called the reservation locator in Receipt/Confirmation/Locator" (`documented` —
  BookingGuide.htm). Durable handle (the PNR). NDC returns two locators (carrier + Travelport
  passive PNR) (`documented` — BookingGuide.htm). The *handle* never expires; the **held
  booking's ticketability** does: `TermsAndConditionsFull/ExpiryDate` and `PaymentTimeLimit`
  returned in the commit response (`documented` — BookingGuide.htm; example
  `"PaymentTimeLimit": "2023-02-02T23:59:00Z"` in
  https://developer.travelport.com/docs/flights/guides/flights-general-guide). Modeled as a
  business precondition + structured dead_end, NOT handle staleness — see §12 friction.
- **Coupling (Q8)**: price valid only for the selected offer; workbench contents valid only for
  that workbench; ticketing workbench valid only for the locator; exchange offers valid only
  against the existing ticket/locator. All drawn as `derived_from` edges in §4.

### 3.5 Empty vs error (Q9, Q9b)

- Search: "NO OFFERS FOUND FOR API CONTENT" (200 / 1576, `documented` — ErrorList.htm) ⇒ valid
  empty; user-repairable fields: dates, cabin, airports. No doc-prescribed automatic fallback ⇒
  no `auto_repairs` (Q9b: none documented).
- AirPrice: "If no brand content exists for the itinerary, the AirPrice response prices the
  itinerary and returns a message that no brands are available" (`documented` —
  AirPricingGuide.htm) ⇒ warning-degraded ok, not empty. "UNABLE TO FARE QUOTE" (200 / 9010,
  `documented` — ErrorList.htm) ⇒ the chosen option can't be fared ⇒ `reselect`.
- Commit #1: "No Fare Found" — "If any pricing modifier... does not have any fares associated
  with it, the commit fails" (`documented` — BookingGuide.htm) ⇒ fare gone at commit ⇒
  `reselect` (source results may still live).
- Exchange search: empty = no viable change options ⇒ valid empty, repair (dates) or dead_end
  per gate.
- Errors ride HTTP 200: "the StatusCode is 200, the Message is COMMUNICATION ERROR, and the
  SourceCode is 2599" (`documented` —
  https://developer.travelport.com/docs/flights/general/error-messaging). The verdict table MUST
  therefore be payload-predicate-first (SPEC §3 row-3 rule) — transport 200 proves nothing.

### 3.6 Failure taxonomy (Q10, Q11)

Documented codes (all from ErrorList.htm unless noted; StatusCode 200 unless noted):

| Code | Message | Class |
|---|---|---|
| 4100 | RESERVATION WORKBENCH ID DOES NOT EXIST | workbench stale/consumed |
| 4367 | HOST SESSION HAS EXPIRED. IGNORE AND REINITIATE WORKBENCH AND RETRY | workbench stale (recovery prescribed) |
| 4434 | RESERVATION WORKBENCH SESSION MUST BE ACTIVE. REINITIATE SESSION | workbench stale |
| 4101 | OFFER ID AND/OR PRODUCT ID DOES NOT EXIST | search-reference stale |
| 1576 | NO OFFERS FOUND FOR API CONTENT | valid empty |
| 9010 | UNABLE TO FARE QUOTE | option not fareable |
| 4012 | AIR SEGMENTS CANNOT BE BOOKED | availability gone (interp. `assumed`) |
| 4132 | RESERVATION DOES NOT EXIST OR CANNOT BE FOUND | probe: not found |
| 4188 | RESERVATION CANNOT BE RETRIEVED. RETRY | transient read fault (retry prescribed) |
| 2599 | COMMUNICATION ERROR | supplier unreachable — ambiguous |
| 2560 | COMMUNICATION ERROR. TRANSACTION TIMED OUT | ambiguous |
| 2595 | COMMUNICATION ERROR. RETRY | transient (retry prescribed) — but see verdict D12: on unkeyed commits still `reconcile` |
| 4357 | DOCUMENT ISSUANCE FAILED. FORM OF PAYMENT IS MISSING | request wrong → repair |
| 4227 | ELECTRONIC TICKETING NOT AUTHORISED FOR YOUR AGENCY | deterministic reject |
| 4565 | OFFER IS NOT VALID FOR EXCHANGE | deterministic reject (exchange) |
| 401/1568–1570, 2500 | NOT AUTHORIZED... / AUTHORIZATION ERROR | deterministic reject (HTTP 401) |
| — | "No Fare Found" (commit) | fare gone → reselect (BookingGuide.htm) |
| — | Two-step commit: `Result/status = "Not Processed"`, Offer `@type OfferModify` with `priceUpdatedInd=true` / `scheduleChangeInd=true`, `ModifyPrice` diff, `StatusAir "Rejected"` | terms changed → gate (`documented` — https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/Workbench/APIRef_WorkbenchCommit.htm) |
| — | NDC (BA): "When ticketing a held booking, the workbench commit returns an error message if the price has changed" | reprice needed (`documented` — APIRef_WorkbenchCommit.htm) |
| — | "OFFER CANNOT BE CANCELED WHEN REFUND AMOUNT DOES NOT EQUAL OFFER PRICE. PERFORM A REFUND QUOTE AND TRY AGAIN." | stale/missing refund quote → rewind (`documented` — https://developer.travelport.com/docs/flights/guides/exchange-refund-and-void-guide) |
| — | "Carrier locator code not returned by carrier/provider. Please try to retrieve later" | warning, degraded ok (BookingGuide.htm) |
| — | "Split Ticketing is not supported and has been ignored." | warning, degraded ok (flights-search-guide) |

Q11 human-decision events (gates, not errors): two-step-commit price/schedule change; booking
consent; ticketing payment consent; remedy choice (void vs refund vs keep); exchange consent.
Q11 payload business blockers (preconditions): `exchangeable: "None"` from Exchange Eligibility
("Values are All, Some, or None", `documented` —
https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/GDSExchangesRefunds/APIRef_ExchangeEligibility.htm);
`automationNotSupportedInd` (same page); held-booking `PaymentTimeLimit`/`ExpiryDate` passed
(retrieve payload); ticket already present on locator (probe payload).

### 3.7 Compensation (Q12, Q13)

- **Held booking (commit #1)**: undo = GDS Reservation Cancel
  (`POST receipt/reservations/{LocatorCode}/receipts`, response `@type: "CancellationHold"`,
  held/unticketed only — `documented` —
  https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/Book/APIRef_ReservationCancelGDS.htm);
  NDC: post-commit workbench → Cancel (`documented` — exchange-refund-and-void-guide). Cost: no
  payment was taken; cancellation fees for held bookings not documented (`assumed` free).
  Passive alternative: let it lapse at PaymentTimeLimit — but active cancel is the polite/clean
  path and releases inventory immediately (`assumed`).
- **Ticket (commit #2)**: time-tiered —
  - within void period: full refund. "Generally a ticket can be voided within the same day it
    was issued, usually up to midnight local agency time. In some instances it may be voided up
    to midnight of the day after ticketing, or special holiday policies may apply"
    (`documented` — exchange-refund-and-void-guide). GDS single void:
    `PUT ticket/tickets/updatestatus/{ticketID}`, "Supported only for ARC users, not BSP",
    "Not supported for NDC"; success returns `"status": "Complete"`; "AirTicketing takes into
    account tickets issued past midnight and on weekends and holidays in determining the
    allowable void period" (`documented` —
    https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/Ticketing/APIRef_TicketVoidGDS.htm).
    BSP/multiple: Batch Void `POST documents/void` (`documented` — endpoints list).
  - after void period: **GDS automated refund "Not supported (under development)"**
    (`documented` —
    https://support.travelport.com/webhelp/JSONAPIs/Airv11/Content/Air11/General/ExchangeRefundGuide.htm)
    ⇒ operator escalation on the GDS rail. NDC: "Post-commit workbench → Refund Quote → cancel";
    "Refund Quote is mandatory when canceling a ticket outside the void period and there is any
    difference between a refund due and the purchase price"; refund may be "full, partial, or
    no refund" per fare rules (`documented` — exchange-refund-and-void-guide).
  - compensator async/idempotency: void response is synchronous (`status: Complete`); voiding an
    already-voided ticket undocumented (`assumed` errors; treated as unkeyed commit-class with
    probe).
- **Q13 replace-before-compensate**: exchanges NEVER decompose into cancel+rebook here — the
  exchange commit atomically swaps the offer on the existing PNR (old offer returned as
  `@type=Offer`, new as `@type=OfferModify` in the commit response; later retrieves show the new
  offer only — `documented` — ExchangeRefundGuide.htm). Where a true rebook (new PNR) replaces
  an old booking, the new `commit_workbench` must land before `cancel_held_booking`/void of the
  old — declared in `ordering_note`.

### 3.8 Latency & finality (Q14, Q15)

- Per-step latency: **not documented** anywhere consulted (`assumed` values in §8; the commits
  and exchange search are the slow steps by nature of host/carrier round-trips). Timeouts set
  from assumption, flagged UNKNOWN-9.
- Finality after commit #2: ticket numbers appear in retrieve/receipts; ticketless-carrier data
  "may be delayed for several hours after booking" (`documented` —
  ReservationRetrieveGuide.htm) ⇒ probe deadline hours-scale, then operator. No webhook
  documented ⇒ `async: {channel: poll}` + `sweep`.

---

## 4. Handle graph

```yaml
handles:
  search_results:                    # CatalogProductOfferings identifiers (offer/product/brand ids)
    minted_by: search
    derived_from: []
    staleness:
      detect: [code:4101]            # "OFFER ID AND/OR PRODUCT ID DOES NOT EXIST" on any reference use
      ttl_hint: 12m (GDS) / 34m (NDC)   # documented cache windows — planning only
      refresh: search

  selected_offer:                    # pseudo-handle (I3): the model/user pick from search_results
    minted_by: select
    derived_from: [search_results]
    rematch:
      key: [carrier, flight_numbers, departure_datetimes, arrival_datetimes, cabin, brand_name, total_price]
      on_ambiguous: model            # near-misses (price drift) → model re-picks; money delta resurfaces at gates anyway
    staleness: {detect: [], ttl_hint: unknown, refresh: select}

  price_offer:                       # AirPrice response Identifier + OfferID (priced quote handle)
    minted_by: air_price
    derived_from: [search_results, selected_offer]
    staleness:
      detect: []                     # no dedicated expiry code documented — UNKNOWN-7; falls to
                                     # unattributable-staleness handling / default rows
      ttl_hint: unknown              # TermsAndConditions may carry last-date-to-ticket; per-offer
      refresh: air_price

  workbench_id:                      # booking workbench session (UUID)
    minted_by: create_workbench
    derived_from: []
    single_use: true                 # consumed by commit_workbench (commit discards workbench); I4:
                                     # spent even when the commit outcome is unknown
    staleness:
      detect: [code:4100, code:4367, code:4434]
      ttl_hint: 30m                  # documented "valid for 30 minutes" — planning only, never a timer
      refresh: create_workbench

  workbench_offer:                   # the offer as loaded into THIS workbench
    minted_by: add_offer
    derived_from: [workbench_id, price_offer]
    staleness: {detect: [code:4101], ttl_hint: unknown, refresh: add_offer}

  traveler_data:                     # travelers as loaded into THIS workbench
    minted_by: add_traveler
    derived_from: [workbench_id]
    staleness: {detect: [], ttl_hint: n/a (dies with workbench), refresh: add_traveler}

  reservation_locator:               # THE PNR — durable output of commit #1; crosses into chains B/C
    minted_by: commit_workbench
    derived_from: [workbench_offer, traveler_data]
    staleness: {detect: [], ttl_hint: n/a, refresh: n/a}
    # NOTE: the handle is durable; the HELD BOOKING'S TICKETABILITY expires at
    # PaymentTimeLimit/ExpiryDate. That is a business precondition + structured dead_end
    # (permanent: false), NOT handle staleness — a PNR cannot be "re-minted" by rewind (I5).

  workbench2_id:                     # ticketing workbench (post-commit, built from locator)
    minted_by: create_postcommit_workbench
    derived_from: [reservation_locator]
    single_use: true                 # consumed by commit_ticketing
    staleness:
      detect: [code:4100, code:4367, code:4434]
      ttl_hint: 30m
      refresh: create_postcommit_workbench

  fop_id:                            # form-of-payment record in the ticketing workbench
    minted_by: add_fop
    derived_from: [workbench2_id]
    staleness: {detect: [], ttl_hint: n/a (dies with workbench), refresh: add_fop}

  payment_link:                      # payment linking fop → offer, inside the workbench
    minted_by: add_payment
    derived_from: [workbench2_id, fop_id]
    staleness: {detect: [], ttl_hint: n/a (dies with workbench), refresh: add_payment}

  ticket_numbers:                    # durable output of commit #2 (Document/Number)
    minted_by: commit_ticketing
    derived_from: [reservation_locator]
    staleness: {detect: [], ttl_hint: n/a, refresh: n/a}

  # ---- Chain C (exchange, GDS rail) ----
  workbench3_id:                     # exchange workbench (post-commit, built from locator)
    minted_by: create_postcommit_workbench_x
    derived_from: [reservation_locator]
    single_use: true                 # consumed by commit_exchange
    staleness: {detect: [code:4100, code:4367, code:4434], ttl_hint: 30m,
                refresh: create_postcommit_workbench_x}

  exchange_results:                  # ExchangeSearch offers — valid only against this ticket/workbench
    minted_by: exchange_search
    derived_from: [workbench3_id, ticket_numbers]
    staleness: {detect: [code:4101], ttl_hint: unknown, refresh: exchange_search}

  selected_exchange_offer:           # pseudo-handle (I3)
    minted_by: select_exchange
    derived_from: [exchange_results]
    rematch:
      key: [carrier, flight_numbers, departure_datetimes, arrival_datetimes, cabin, brand_name, change_fee, add_collect_total]
      on_ambiguous: gate             # money moved: ambiguity goes to the user, not the model
    staleness: {detect: [], ttl_hint: unknown, refresh: select_exchange}

  exchange_offer_in_wb:              # Add/Modify Offer result inside the exchange workbench
    minted_by: add_modify_offer
    derived_from: [workbench3_id, selected_exchange_offer]
    staleness: {detect: [code:4101], ttl_hint: unknown, refresh: add_modify_offer}
```

Derivation sketch (`derived_from` edges point up):

```
                          search_results
                               |
                         selected_offer (I3)
                               |
        workbench_id      price_offer
             \             /
              workbench_offer     traveler_data ── workbench_id
                        \            /
                     [COMMIT #1] commit_workbench
                               |
                      reservation_locator  (durable; business TTL = PaymentTimeLimit)
                     /         |          \
        workbench2_id    (chain B: void/refund)   workbench3_id ── ticket_numbers
             |                                        |         (exchange_results needs both)
           fop_id                               exchange_results
             |                                        |
        payment_link                        selected_exchange_offer (I3)
             |                                        |
   [COMMIT #2] commit_ticketing               exchange_offer_in_wb
             |                                        |
       ticket_numbers                     [COMMIT] commit_exchange (mutates reservation_locator)
```

Cross-chain notes: `reservation_locator` and `ticket_numbers` are the only handles that cross
scopes (A → B, A → C). They are durable commit outputs (I5): chains B and C consume them but no
rewind in B/C can ever re-mint them — their "refresh" is a whole new Chain-A run, decided by the
planner above this spec.

---

## 5. Actions

Abridged to decision-relevant fields; all per SPEC §2.1. GDS rail primary; NDC deltas boxed at
the end (§5.4).

### 5.1 Chain A — book + ticket

```yaml
actions:
  - id: search
    description: Air shop; returns catalog product offerings for the itinerary
    effect: read                     # double-call test: no external consequence; the identifiers
                                     # reference a cached result set (cache ≠ session state)
    input:
      intent: {origin: airport, destination: airport, dates: date_range,
               cabin: enum, travelers: pax_spec}
    output:
      payload: catalog_product_offerings[]
      handles: [search_results]
      empty:
        valid: true
        detect: "code:1576 NO OFFERS FOUND FOR API CONTENT (or offerings empty)"
        route: repair(to: search, fields: [dates, cabin, origin, destination])
    request_invariants:
      offersPerPage: <set>           # documented: omitting it breaks ALL later reference
                                     # payloads (Flight Specific Search, AirPrice, Add Offer) —
                                     # a silently-degrading parameter, exactly what this field is for
    idempotency: {mode: natural}
    timeout: 30s
    latency_hint: 8s                 # assumed — not documented (UNKNOWN-9)

  - id: select                       # selection pseudo-step: no dispatch
    effect: select
    input: {handles: [search_results]}
    output: {handles: [selected_offer]}

  - id: air_price
    description: AirPrice reference payload — confirm/refresh pricing on the selected offer
    effect: mint                     # produces a priced-offer quote handle prepared for Add Offer
                                     # (SPEC 2.1: "a quote handle" = mint); repeat = orphan quote,
                                     # self-expiring, never billed
    input: {handles: [search_results, selected_offer]}
    output:
      payload: {priced_offer: OfferID, terms: TermsAndConditions,   # incl. last date to ticket
                total_price: money}
      handles: [price_offer]
    preconditions:
      - when: "total_price != selected_offer.total_price"
        verdict: gate(price_review)  # assumed product choice: surface pre-book drift to user
        reason: "price drifted between search and AirPrice"
      - when: "warning: no brands available"
        verdict: ok                  # documented degraded success — proceed brandless, traced
        reason: "carrier returned pricing without brand content"
    idempotency: {mode: natural}
    timeout: 45s
    latency_hint: 10s                # assumed

  - id: create_workbench
    description: Mint booking workbench session
    effect: mint                     # dedicated mutable session state for a later commit —
                                     # the definitional mint; discardable via DELETE; assumed inert
                                     # while uncommitted (UNKNOWN-5)
    input: {}
    output: {payload: identifier, handles: [workbench_id]}
    idempotency: {mode: natural}     # repeat = orphan workbench, self-expires in ~30m
    timeout: 15s
    latency_hint: 2s                 # assumed

  - id: add_offer
    description: Load the priced offer into the workbench (reference payload)
    effect: mint                     # mutates only workbench-scoped state
    input: {handles: [workbench_id, price_offer]}
    output: {payload: offer_in_workbench, handles: [workbench_offer]}
    idempotency: {mode: natural}     # re-add into a FRESH workbench after rewind; duplicate add
                                     # into same workbench never attempted (I2 replace = new workbench)
    timeout: 45s
    latency_hint: 8s                 # assumed

  - id: add_traveler
    description: Add traveler details to the workbench
    effect: mint
    input:
      intent: {travelers: traveler_details}   # names, DOB, contact, loyalty — model/user-repairable
      handles: [workbench_id]
    output: {payload: traveler_ack, handles: [traveler_data]}
    idempotency: {mode: natural}
    timeout: 30s
    latency_hint: 3s                 # assumed

  - id: commit_workbench             # ========== COMMIT #1 — held booking ==========
    description: Commit the workbench → airline-confirmed HELD booking (no payment taken)
    effect: commit
    entry_gate: booking_consent      # re-fires on EVERY arrival, incl. post-rewind replays
    input:
      intent: {accept_changes: bool = false}   # two-step-commit acknowledgement; set only via
                                               # the price_schedule_change gate outcome
      handles: [workbench_id, workbench_offer, traveler_data]
    output:
      payload: {locator: string, offer_status: StatusAir,           # "HK" = confirmed
                expiry: TermsAndConditionsFull/ExpiryDate, payment_time_limit: PaymentTimeLimit}
      handles: [reservation_locator]
    request_invariants:
      enableTwoStepCommitInd: "true when accept_changes == false; MUST be false/omitted on the
                               post-consent redispatch (documented: resending true returns an
                               error and discards the workbench)"   # GDS only
      errorWhenPriceChangesInd: "mirrors enableTwoStepCommitInd"
      errorWhenScheduleChangesInd: "mirrors enableTwoStepCommitInd"
    idempotency: {mode: none}        # no client key exists (elicitation Q4) → attempts: 1 forced
    confirmation:
      probe: get_reservation         # GET book/reservation/reservations/{LocatorCode} when locator
                                     # known; locator-LOST path: probe_workbench_state
                                     # (GET .../reservationworkbench/{workbenchID}) — workbench
                                     # retrievable+intact ⇒ not landed; workbench gone ⇒ AMBIGUOUS
                                     # (committed vs expired) ⇒ hold at reconciling, sweep, operator.
                                     # UNKNOWN-1: no documented locator-free reservation search.
      signal: "reservation exists at locator with itinerary matching workbench_offer and
               travelers matching traveler_data"
      async: {channel: poll, deadline: 6h}      # carrier-locator lag documented ("retrieve later");
                                                # hours-scale for ticketless carriers
      sweep: {interval: 30m, escalate_after: 12h}
    compensation:
      chain: [cancel_held_booking]
      window: "any time before ticketing; the held booking also self-expires at PaymentTimeLimit
               ('usually within 24 hours of booking' — documented). No payment was taken at this
               commit, so unwinding costs nothing (fee-free: assumed, UNKNOWN-8)."
      ordering_note: "if this run replaces an existing booking, land THIS commit before
                      compensating the old one — never leave the traveler with nothing"
    timeout: 200s                    # assumed generous: host+carrier round trip; a tight timeout
                                     # manufactures ambiguous commits (UNKNOWN-9)
    latency_hint: 30s                # assumed
    # P3 mechanics (unkeyed commit): correlation record {run_id, intent snapshot incl. traveler
    # name/date/route, workbench_id, custom traceID header value, status=PENDING} persisted in
    # the run store (durable DB, survives process death) BEFORE dispatch; cancellation shield on
    # the in-flight attempt. The custom traceID request header (documented) is stamped with the
    # run_id + step so Travelport-side support can correlate — tracing, not dedup.

  # ---------- ticket sub-chain (COMMIT #2) ----------

  - id: create_postcommit_workbench
    description: Mint ticketing workbench from the reservation locator
    effect: mint
    input: {handles: [reservation_locator]}
    output: {payload: identifier, handles: [workbench2_id]}
    idempotency: {mode: natural}
    timeout: 15s
    latency_hint: 3s                 # assumed

  - id: get_reservation
    description: Retrieve the held booking (also the confirmation probe for both commits)
    effect: read
    input: {handles: [reservation_locator]}
    output:
      payload: {reservation, offer_status, payment_time_limit, expiry_date,
                receipts: [Confirmation | Payment/Document...]}
    preconditions:
      - when: "receipts contain ticket Document/Number matching this offer"
        verdict: ok                  # already ticketed (e.g. resumed run): skip to done, degraded-
        reason: "tickets already issued for this booking"          # advance recorded in trace
      - when: "now > PaymentTimeLimit or now > ExpiryDate"
        verdict: dead_end            # reason: held_booking_expired, permanent: false —
        reason: "ticketing time limit passed; airline may have cancelled the unticketed booking;
                 planner must start a NEW booking run (a PNR cannot be re-minted by rewind)"
      - when: "offer_status not confirmed (not HK)"
        verdict: gate(booking_state_review)
        reason: "booking present but not confirmed (UC/UN etc.) — human reviews"
    idempotency: {mode: natural}
    timeout: 30s
    latency_hint: 3s                 # assumed

  - id: add_fop
    description: Add form of payment to the ticketing workbench
    effect: mint
    input:
      intent: {payment_instrument: fop_ref}   # user-repairable
      handles: [workbench2_id]
    output: {payload: fop_ack, handles: [fop_id]}
    idempotency: {mode: natural}
    timeout: 30s
    latency_hint: 3s                 # assumed

  - id: add_payment
    description: Link FOP to the offer(s) being ticketed
    effect: mint                     # no charge occurs here; money moves at commit (documented:
                                     # "at workbench commit, that payment is applied and ticket/s issued")
    input: {handles: [workbench2_id, fop_id]}
    output: {payload: payment_ack, handles: [payment_link]}
    idempotency: {mode: natural}
    timeout: 30s
    latency_hint: 3s                 # assumed

  - id: commit_ticketing             # ========== COMMIT #2 — issue ticket, take payment ==========
    description: Commit ticketing workbench → applies payment, issues ticket(s)/EMD(s)
    effect: commit
    mutates: reservation_locator     # in-place: the SAME reservation transitions held → ticketed;
                                     # ticket_numbers minted as durable output alongside
    entry_gate: ticketing_payment_consent     # money gate; re-fires on every arrival incl.
                                              # post-reprice replays (new price ⇒ new consent)
    input: {handles: [workbench2_id, payment_link], intent: {}}
    output:
      payload: {receipt_payment: Document/Number, settlement: ...}
      handles: [ticket_numbers]
    idempotency: {mode: none}        # no client key (Q4) → attempts: 1 forced
    confirmation:                    # content-based (mutates): observe the mutated state —
      probe: get_reservation         # ticket receipts on the SAME locator; reservation existence
      signal: "Reservation Retrieve / Ticket List returns Document/Number (ticket) whose
               itinerary matches the held booking's offer"          # proves nothing by itself
      async: {channel: poll, deadline: 6h}    # ticketless-carrier data may lag hours (documented)
      sweep: {interval: 30m, escalate_after: 12h}
    compensation:
      chain: [list_tickets, void_ticket]      # GDS rail; gate cancel_remedy_choice fires via
                                              # list_tickets precondition (state=compensating)
      window: "void: generally same day as issuance, usually up to midnight local agency time
               (sometimes midnight next day / holiday policies — documented). OUTSIDE the void
               window on GDS: automated refund NOT supported ('under development' — documented)
               ⇒ compensator dead-ends to operator. NDC rail: refund_quote → cancel_ticket."
      ordering_note: "never void the old ticket before a replacement need is settled; for
                      rebook-replacements land the new commit first"
    timeout: 200s                    # assumed (UNKNOWN-9)
    latency_hint: 30s                # assumed
    # P3: correlation record {run_id, locator, offer snapshot, fop alias, status=PENDING}
    # persisted before dispatch; cancellation shield. Probe here is STRONG (locator known
    # before dispatch) — unlike commit #1.
```

### 5.2 Chain B — void / refund (compensators; also standalone servicing chain)

```yaml
  - id: list_tickets
    description: List ticket receipts on the reservation (GET receipt/reservations/{Locator}/receipts)
    effect: read
    input: {handles: [reservation_locator]}
    output: {payload: tickets[]}
    preconditions:
      - when: "tickets.length > 0 and unwind in progress"
        verdict: gate(cancel_remedy_choice)   # user picks: void now (full refund) / operator
        reason: "human picks the remedy; void is time-boxed"        #  refund path / keep ticket
      - when: "tickets.length == 0"
        verdict: ok                  # nothing ticketed → nothing to void; compensation of
        reason: "no tickets on PNR"  # commit #2 degenerates to no-op (traced)
    idempotency: {mode: natural}
    timeout: 30s

  - id: void_ticket                  # GDS single void — PUT ticket/tickets/updatestatus/{ticketID}
    description: Void a ticket within the void period → full refund
    effect: compensate               # commit-class; mutates ticket status in place
    mutates: ticket_numbers
    input: {handles: [ticket_numbers], intent: {ticket_id: string}}  # ticket_id chosen at the
                                     # remedy gate — gate-validated opaque value entering intent
                                     # (the one sanctioned path, SPEC §2.6)
    output: {payload: {status: "Complete", settlement_authorization: string?}}
    idempotency: {mode: none}        # voiding a voided ticket: undocumented (UNKNOWN-4) —
                                     # treated unkeyed → attempts: 1 + probe
    confirmation:
      probe: list_tickets            # content-based: ticket status/receipt reflects void
      signal: "ticket receipt shows voided/cancelled state for {ticket_id}"   # exact field
                                     # UNDOCUMENTED in pages consulted — assumed; UNKNOWN-4
      async: {channel: poll, deadline: 2h}
      sweep: {interval: 30m, escalate_after: 24h}
    compensation: {action: none,
                   ordering_note: "no un-void documented; protection = cancel_remedy_choice gate
                                   BEFORE dispatch (money/consent gate, audience user) — void is
                                   only reachable through it"}
    timeout: 90s                     # assumed
    # Constraints (documented, APIRef_TicketVoidGDS.htm): ARC only (not BSP — BSP uses Batch
    # Void POST documents/void); not supported for NDC; void window per §3.7.

  - id: refund_quote                 # NDC rail — POST .../offers/canceloffer (in workbench)
    description: Quote the refund for cancelling a ticketed NDC offer
    effect: mint                     # produces a quote inside the workbench; documented as
                                     # MANDATORY before cancel when refund != purchase price
    input: {handles: [workbench2_or_3_id, ticket_numbers]}
    output: {payload: {refund_amount: money, penalty: money, breakdown}}
    preconditions:
      - when: "refund quote returned"
        verdict: gate(refund_consent)
        reason: "user must accept the (possibly partial) refund before the cancel commit"
    idempotency: {mode: natural}
    timeout: 60s

  - id: cancel_ticket                # NDC rail — POST receipt/reservation/{workbenchID}/receipts
    description: Cancel the ticketed NDC offer; refund credited to original FOP
    effect: compensate
    mutates: reservation_locator
    input: {handles: [workbench2_or_3_id, ticket_numbers], intent: {}}
    output: {payload: cancellation_receipt}
    idempotency: {mode: none}
    confirmation:
      probe: get_reservation
      signal: "reservation/offer shows cancelled state and cancellation receipt present"
      async: {channel: poll, deadline: 6h}
      sweep: {interval: 1h, escalate_after: 48h}
    compensation: {action: none,
                   ordering_note: "no un-cancel; protection = refund_consent gate before dispatch"}
    timeout: 120s                    # assumed

  - id: cancel_held_booking          # compensator for COMMIT #1 (GDS: POST receipt/reservations/
    description: Cancel a held (unticketed) reservation                    # {Locator}/receipts)
    effect: compensate
    mutates: reservation_locator
    input: {handles: [reservation_locator], intent: {}}
    output: {payload: {receipt: CancellationHold, locator: string}}       # documented @type
    idempotency: {mode: none}        # re-cancel of cancelled PNR undocumented (UNKNOWN-8) —
                                     # treated unkeyed; probe disambiguates
    confirmation:
      probe: get_reservation
      signal: "reservation shows cancelled state / CancellationHold receipt present"
      async: {channel: poll, deadline: 2h}
      sweep: {interval: 30m, escalate_after: 24h}
    compensation: {action: none,
                   ordering_note: "held booking was free and self-expiring; no undo needed —
                                   a fresh Chain-A run rebooks"}
    timeout: 90s                     # assumed
```

### 5.3 Chain C — exchange (GDS rail)

```yaml
  - id: exchange_eligibility
    description: Optional pre-check — does the ticket have exchange/refund value?
    effect: read
    input: {handles: [ticket_numbers, reservation_locator]}
    output: {payload: {exchangeable: enum[All, Some, None], refundable: enum[All, Some, None],
                       automationNotSupportedInd: bool, penalties: {change: money_range}}}
    preconditions:
      - when: "exchangeable == 'None'"
        verdict: dead_end            # reason: not_exchangeable, permanent: true (for this ticket)
        reason: "carrier fare rules forbid exchange"
      - when: "automationNotSupportedInd == true"
        verdict: dead_end            # reason: manual_processing_required, permanent: true;
        reason: "documented flag: manual processing required — operator handles offline"
      - when: "penalties.change.max > 0"
        verdict: gate(exchange_fee_ack)
        reason: "user acknowledges the fee range before shopping"
    idempotency: {mode: natural}
    timeout: 45s

  - id: create_postcommit_workbench_x
    description: Mint exchange workbench from locator (retrieves the existing booking into it)
    effect: mint
    input: {handles: [reservation_locator]}
    output: {payload: identifier, handles: [workbench3_id]}
    idempotency: {mode: natural}
    timeout: 15s

  - id: exchange_search
    description: Shop changed itineraries against the existing ticket (leg-scoped or full)
    effect: read
    input:
      intent: {changed_legs: leg_selector, new_dates: date_range?, cabin: enum?}
      handles: [workbench3_id, ticket_numbers]
    output:
      payload: exchange_offers[]     # incl. change fee + add-collect per option
      handles: [exchange_results]
      empty:
        valid: true
        detect: "exchange_offers.length == 0"
        route: repair(to: exchange_search, fields: [new_dates, cabin, changed_legs])
    idempotency: {mode: natural}
    timeout: 90s                     # assumed slow (host repricing)
    latency_hint: 20s                # assumed

  - id: select_exchange              # pseudo-step
    effect: select
    input: {handles: [exchange_results]}
    output: {handles: [selected_exchange_offer]}

  - id: add_modify_offer
    description: Load the selected exchange offer into the workbench, replacing the itinerary
    effect: mint                     # workbench-scoped until commit
    input: {handles: [workbench3_id, selected_exchange_offer]}
    output: {payload: offer_modify_preview, handles: [exchange_offer_in_wb]}
    idempotency: {mode: natural}
    timeout: 60s

  # add_fop_x / add_payment_x: same shapes as add_fop / add_payment, workbench3-scoped.
  # Documented: FOP required only for add-collect, but "Payment: required even if you will not
  # ticket the reservation at commit".

  - id: commit_exchange              # ========== COMMIT — in-place PNR modification ==========
    description: Commit the exchange; reissues ticket now (payLaterInd=false) or holds changes
    effect: commit
    mutates: reservation_locator     # documented: response carries old offer (@type=Offer) and
                                     # new offer (@type=OfferModify); later retrieves show the
                                     # new offer ONLY — the PNR is modified in place
    entry_gate: exchange_consent     # money gate (change fee + add-collect); re-fires on replays
    input:
      intent: {pay_later: bool}      # payLaterInd — user-chosen at exchange_consent
      handles: [workbench3_id, exchange_offer_in_wb]   # + payment_link_x when add-collect
    output:
      payload: {receipt, new_ticket: Document/Number?, modify_price: ModifyPrice}
      handles: [ticket_numbers]      # REPLACES the prior ticket_numbers value for pay-now;
                                     # old ticket is reissued/absorbed by the exchange
    idempotency: {mode: none}        # attempts: 1
    confirmation:                    # content-based (mutates): existence proves nothing —
      probe: get_reservation         # compare CONTENT against the selected exchange offer
      signal: "retrieve shows the NEW itinerary (OfferModify content) and, for pay-now, a new
               ticket Document/Number; old itinerary absent"
      async: {channel: poll, deadline: 6h}
      sweep: {interval: 30m, escalate_after: 12h}
    compensation:
      action: none                   # no documented un-exchange; the newly issued ticket has its
                                     # own void window (assumed applicable — UNKNOWN-10), which the
                                     # planner may exploit as a NEW Chain-B run, not as automatic
                                     # compensation here
      ordering_note: "protection sits BEFORE the commit: exchange_consent entry gate (user,
                      money) on every arrival — required by SPEC rule for mutates+no-undo"
    timeout: 200s                    # assumed
    # P3: correlation record {run_id, locator, old ticket numbers, selected offer snapshot,
    # status=PENDING} persisted before dispatch; cancellation shield.
```

### 5.4 NDC rail deltas (compile-time variant)

`documented` — exchange-refund-and-void-guide + BookingGuide.htm + NDC workflows page
(https://developer.travelport.com/docs/flights/workflows/exchanges/ndc-exchange-and-refund-workflows):

- **Booking**: FOP cannot be added at booking; Instant Pay books+tickets in ONE workbench
  (single commit — the two-stage shape collapses to one commit whose compensation is
  void/refund directly). Ticketless carriers: "the booking is issued and completed in the
  commit step; there is no separate ticketing step". Two locators returned (carrier + passive
  PNR); two-step commit indicators are **GDS only** — the NDC `commit_workbench` variant drops
  the `enableTwoStepCommitInd` invariants and the D5 row routes to `known_unmatched`.
- **Ticketing price change**: documented (BA) error when price changed at ticketing → D8 row
  routes to `standalone_reprice` (POST reprice/.../buildfromoffer) then `reprice_consent` gate,
  then re-approach `commit_ticketing` (entry gate re-fires — new price, new consent).
- **Exchange**: Reshop → Reprice → [FOP/Payment] → Modify (`POST modify/reservations/{Locator}`);
  "If any refund is due, new tickets are issued at Modify and the refund is credited to the
  original form of payment".
- **Void**: no Ticket Void API — post-commit workbench → Cancel inside the void period issues
  full refund.
- **Refund**: Refund Quote → Cancel supported (unlike GDS).

---

## 6. Verdict table

Domain rows first (top-down, first match wins), then SPEC §3 generic skeleton rows 1–12.
All Travelport business errors arrive with **HTTP 200 + Result/Error payload** (documented,
error-messaging page) — every row below is a payload predicate, honoring the rule that
payload-level rows precede unconditional transport success.

| # | Signal | Step scope | Verdict | Evidence/rationale |
|---|---|---|---|---|
| D1 | `code:4367` / `code:4434` / `code:4100` (workbench gone/expired/inactive) | any consumer of `workbench_id`/`workbench2_id`/`workbench3_id` **except an in-flight commit's own response** (see D2) | `rewind(to: <that workbench's refresh>)` | documented codes; 4367's own message prescribes "REINITIATE WORKBENCH AND RETRY" (ErrorList.htm) |
| D2 | `code:4100`-class on the response of `commit_workbench` / `commit_ticketing` / `commit_exchange` itself | those commits | `reconcile` | cannot distinguish "expired before dispatch" from "already consumed by a landed commit" — fail to probe, never re-dispatch (UNKNOWN-6) |
| D3 | `code:4101` OFFER/PRODUCT ID DOES NOT EXIST | air_price, add_offer, add_modify_offer, exchange consumers | `rewind(to: search)` (Chain A) / `rewind(to: exchange_search)` (Chain C) — replay re-crosses `select` via rematch | documented (ErrorList.htm); search reference dead ⇒ re-shop is the refresh |
| D4 | `code:1576` NO OFFERS FOUND / empty offerings | search, exchange_search | `ok(empty)` → step's `empty.route` (repair: dates/cabin/airports) | documented; P4 |
| D5 | Two-step commit: `Result/status == "Not Processed"` + `OfferModify.priceUpdatedInd` or `.scheduleChangeInd` (+ MCT warning variant) | commit_workbench (GDS) | `gate(price_schedule_change)` — signal declared `rejected_before_execution: true` | documented (APIRef_WorkbenchCommit.htm): warning returned "instead of proceeding", StatusAir "Rejected", booking not created; redispatch (with indicators false) rides `safe_reject_redispatch`, not the exactly-once attempt |
| D6 | "No Fare Found" | commit_workbench | `reselect(to: select)` — declared `rejected_before_execution: true` | documented: "the commit fails" — fare gone, offer-level; source results may still live; failure-before-booking wording (BookingGuide.htm); redispatch after fresh selection uses safe-reject budget |
| D7 | `code:9010` UNABLE TO FARE QUOTE | air_price | `reselect(to: select)` | documented code; chosen option unfareable, siblings may price |
| D8 | NDC price-changed error at ticketing commit | commit_ticketing (NDC) | `rewind(to: standalone_reprice)` → reprice mints fresh priced offer → `gate(reprice_consent)` → replay commit (entry gate re-fires) | documented (BA note, APIRef_WorkbenchCommit.htm; Standalone Reprice endpoint documented). Treated as pre-execution reject (`rejected_before_execution: true` — error *instead of* ticketing; `assumed` no partial issuance, UNKNOWN-3) |
| D9 | `code:4357` FOP MISSING | commit_ticketing | `repair(to: add_fop, fields: [payment_instrument])` — declared `rejected_before_execution: true` (validation reject; `assumed` no issuance) | documented code; user fixes instrument |
| D10 | `code:4012` AIR SEGMENTS CANNOT BE BOOKED | commit_workbench | `reselect(to: select)` (interpretation `assumed` — could be deterministic reject; see known_unmatched note) declared NOT safe-reject ⇒ consumes the attempt ⇒ effectively: probe first if any doubt | documented code, undocumented semantics |
| D11 | "OFFER CANNOT BE CANCELED WHEN REFUND AMOUNT DOES NOT EQUAL OFFER PRICE..." | cancel_ticket | `rewind(to: refund_quote)` | documented; the message prescribes exactly this recovery |
| D12 | `code:2595` COMMUNICATION ERROR. RETRY / `code:2560` timeout / `code:2599` COMMUNICATION ERROR / transport timeout / conn drop / ambiguous 5xx | read + mint steps | `retry` | 2595 documented "Temporary"; reads/mints safe to repeat |
| D13 | same signals as D12 | commit_workbench, commit_ticketing, commit_exchange, void_ticket, cancel_ticket, cancel_held_booking | `reconcile` | unkeyed commits: supplier-unreachable-after-dispatch is ambiguous; Travelport does NOT document 2595/2599 as no-side-effect ⇒ they do NOT qualify for `rejected_before_execution` (P3; the canonical double-booking row) |
| D14 | `code:4188` RESERVATION CANNOT BE RETRIEVED. RETRY | get_reservation, list_tickets | `retry` | documented, retry prescribed |
| D15 | `code:4132` RESERVATION DOES NOT EXIST OR CANNOT BE FOUND | get_reservation as confirmation probe | probe evidence "not landed" → per reconcile procedure (rewind to re-mint consumed workbench if budget, else dead_end) | documented code |
| D16 | HTTP 401 / codes 1568–1570, 2500 | any | `dead_end(reason: not_authorized, permanent: true)` | documented; config/provisioning problem, identical on retry |
| D17 | `code:4227` E-TICKETING NOT AUTHORISED | commit_ticketing | `dead_end(reason: agency_not_authorized, permanent: true)` | documented |
| D18 | `code:4565` OFFER IS NOT VALID FOR EXCHANGE | add_modify_offer, commit_exchange | `dead_end(reason: not_exchangeable, permanent: true)` | documented |
| D19 | void rejected: outside void period (message text undocumented — matched via void failure + local void-window calculation as *planning* signal) | void_ticket | `dead_end(reason: void_window_closed, permanent: true /*for void*/, retry_after_hint: none)` — planner routes to refund path (NDC) or operator (GDS) | guide documents the window; exact error text UNKNOWN-4; falls to default rows until captured, this row documents intent |
| D20 | warning "Carrier locator code not returned by carrier/provider. Please try to retrieve later" | commit_workbench (success path) | `ok` degraded (precondition-style; carrier locator backfilled by later probe) | documented (BookingGuide.htm) |
| —  | then SPEC §3 generic rows 1–12 (staleness, empty, ok-fallthrough, rate-limit, transport, terms-changed, deterministic reject, degrade, unmatched→reconcile/dead_end) | | | |

**`known_unmatched`** (consciously left to generic rows 11–12 — commit-in-flight → `reconcile`,
else `dead_end` — until independently captured):
`1170 RESERVATION DATA IS INVALID`, `3002 ENTRY IS NOT VALID`, `3003 FLIGHT SEGMENTS MUST BE
LESS THAN 9 SEGMENTS`, `3159 FLIGHT NOT FOUND`, `9000 REQUEST MUST CONTAIN VALID INFORMATION`,
`8149 TICKET NUMBER IS REQUIRED`, the un-enumerated remainder of the 1000/2000/3000/4000/8000/
9000/20000/21000 SourceCode ranges (the error-messaging page documents the ranges but the full
message list was not exhaustively crawled), all NDC carrier-specific (`SourceID = carrier code`)
error bodies, HTTP 429 / rate-limit behavior (never documented in pages consulted), and the
exact error/status shape for: committing an expired workbench, voiding a voided ticket,
cancelling a cancelled reservation, ticketing an already-ticketed booking. Also `4012` semantics
(D10 interpretation is assumed). Split-ticketing warning → ok degraded (documented, traced).

---

## 7. Gates

```yaml
gates:
  booking_consent:                   # entry gate on commit_workbench
    audience: user
    payload: {itinerary, total_price: money, payment_time_limit_note: string}
    outcomes:
      approve: ok
      decline: dead_end              # reason: user_declined, permanent: true (this run)
    timeout: {after: 30m, verdict: dead_end}

  price_review:                      # pre-book drift surfaced at air_price (assumed product choice)
    audience: user
    payload: {searched: money, priced: money}
    outcomes:
      accept:  ok
      decline: reselect(to: select)
    timeout: {after: 30m, verdict: dead_end}

  price_schedule_change:             # two-step commit warning (GDS)
    audience: user
    payload: {modify_price: ModifyPrice, price_updated: bool, schedule_changed: bool, mct_warning: bool}
    outcomes:
      accept:  {set: {accept_changes: true},
                verdict: repair(to: commit_workbench, fields: [accept_changes])}
                                     # redispatch with two-step indicators FALSE (documented:
                                     # resending true errors + discards workbench); rides
                                     # safe_reject_redispatch, not the exactly-once attempt
      decline_pick_other: reselect(to: select)
      decline_abandon:    dead_end
    timeout: {after: 30m, verdict: dead_end}   # workbench dies at ~30m anyway (hint, not timer):
                                               # timing the gate near it avoids a doomed redispatch

  ticketing_payment_consent:         # entry gate on commit_ticketing — MONEY, never model
    audience: user
    payload: {itinerary, total_price: money, instrument: fop_ref, payment_time_limit: datetime}
    outcomes:
      approve: ok
      change_instrument: {params: {payment_instrument: fop_ref}, bind: [payment_instrument],
                          verdict: repair(to: add_fop, fields: [payment_instrument])}
      decline: dead_end              # held booking left to self-expire or planner cancels it
    timeout: {after: 12h, verdict: dead_end}   # product choice: well inside typical 24h
                                               # PaymentTimeLimit; the limit itself stays a
                                               # per-booking payload value, never a timer

  reprice_consent:                   # NDC: after standalone_reprice on held booking
    audience: user
    payload: {old_price: money, new_price: money}
    outcomes:
      accept:  ok                    # advance to commit_ticketing (its entry gate still re-fires)
      decline: dead_end              # reason: user_declined_reprice; planner may cancel held booking
    timeout: {after: 12h, verdict: dead_end}

  booking_state_review:              # held booking in unexpected status at retrieve
    audience: operator
    payload: {reservation, offer_status}
    outcomes:
      proceed: ok
      abort:   dead_end
    timeout: {after: 24h, verdict: dead_end}

  cancel_remedy_choice:              # inside compensation of commit_ticketing (state=compensating)
    audience: user
    payload: {tickets[], void_window_estimate: datetime, refund_supported: bool /*rail*/}
    outcomes:
      void_now: {params: {ticket_id: string}, bind: [ticket_id], verdict: ok}   # proceed to void_ticket
      keep:     dead_end             # reason: user_kept_ticket — unwind intentionally stops; traced
    timeout: {after: 4h, verdict: dead_end}    # void window is same-day: short timeout, then
                                               # operator per C8.5 (dead_end w/ landed commits
                                               # escalates, never silently abandons)

  refund_consent:                    # NDC refund path (state=compensating when unwinding)
    audience: user
    payload: {refund_amount: money, penalty: money, breakdown}
    outcomes:
      accept:  ok                    # proceed to cancel_ticket
      decline: dead_end              # keep the ticket
    timeout: {after: 24h, verdict: dead_end}

  exchange_fee_ack:                  # eligibility showed a fee range
    audience: user
    payload: {change_fee_range: money_range}
    outcomes: {acknowledge: ok, abandon: dead_end}
    timeout: {after: 24h, verdict: dead_end}

  exchange_consent:                  # entry gate on commit_exchange — MONEY, never model
    audience: user
    payload: {old_itinerary, new_itinerary, change_fee: money, add_collect: money,
              pay_later_option: bool}
    outcomes:
      approve_pay_now:   {set: {pay_later: false}, verdict: repair(to: commit_exchange, fields: [pay_later])}
      approve_pay_later: {set: {pay_later: true},  verdict: repair(to: commit_exchange, fields: [pay_later])}
      decline: dead_end
    timeout: {after: 30m, verdict: dead_end}
    # note: both approve outcomes are repairs carrying the declared constant into intent before
    # the (re-)arrival at the commit; on arrival the entry gate has been satisfied this pass.
```

---

## 8. Policy

```yaml
policy:
  per_step:
    read:  {attempts: 3, backoff: {base: 500ms, factor: 2, jitter: full, max: 8s}}
    mint:  {attempts: 2, backoff: {base: 1s, factor: 2, jitter: full, max: 10s}}
    commit:
      unkeyed:
        attempts: 1                  # ALL Travelport commits are unkeyed (no client key exists)
        safe_reject_redispatch: 2    # consumed only by rows declared rejected_before_execution:
                                     # D5 (two-step "Not Processed"), D6 (No Fare Found),
                                     # D8/D9 (validation rejects at ticketing, assumed-flagged)
    compensate: {attempts: 1, safe_reject_redispatch: 1}
    # step overrides
    commit_workbench:  {timeout: 200s}
    commit_ticketing:  {timeout: 200s}
    commit_exchange:   {timeout: 200s}
    exchange_search:   {timeout: 90s}
  per_chain:
    chain_A: {max_rewinds: 4, max_repairs: 2, wall_clock: 12m, gate_timeout: 30m}
      # wall_clock = max(3× Σ latency_hints ≈ 3×110s ≈ 5.5m, 2× slowest timeout = 400s ≈ 6.7m)
      # → 12m headroom for one full rewind cycle; excludes gate parking and reconcile parking
    chain_B: {max_rewinds: 2, max_repairs: 1, wall_clock: 8m, gate_timeout: 4h}
    chain_C: {max_rewinds: 3, max_repairs: 2, wall_clock: 12m, gate_timeout: 30m}
    compensation_order: reverse      # commit_ticketing unwinds before commit_workbench:
                                     # void/refund the ticket FIRST, then (and only then) the
                                     # now-unticketed booking is cancellable via cancel_held_booking
  escalation:
    - {audience: model,    may: [choose among returned offers/exchange options,
                                 propose repair values for dates/cabin/airports/travelers]}
    - {audience: user,     may: [answer all money/consent gates, approve repairs with cost,
                                 pick remedy (void/refund/keep), supply payment instrument]}
    - {audience: operator, may: [resolve reconcile deadlines, GDS post-void refunds (API
                                 unsupported), unstick PENDING correlation records,
                                 booking_state_review]}
```

All numbers are config to be tuned (SPEC §2.4 OPEN note); the *shape* — `attempts: 1` on every
commit here — is non-negotiable given `idempotency.mode: none` throughout.

External-mode note: if executed by a planner loop, `restart_required` carries
`{rewind_to, repair_fields?, reason, budgets_remaining}` and continuations decrement the same
budgets from the durable snapshot (SPEC §2.3); no budget lives in prompt text.

---

## 9. Invalidation walkthroughs

### 9.1 Workbench expiry mid-booking (the everyday failure)

Setup: Chain A has run `search → select → air_price → create_workbench → add_offer`. The user
dawdles; cursor reaches `add_traveler`; dispatch returns 200 + `code:4367 HOST SESSION HAS
EXPIRED`.

1. Row D1 matches → `rewind(to: create_workbench)`. Budget: rewinds 4→3.
2. I1 transitive invalidation, atomic: `workbench_id` re-mint pending ⇒ descendants
   `workbench_offer`, `traveler_data` die. `price_offer` has NO `derived_from` path through
   `workbench_id` (edges: [search_results, selected_offer]) ⇒ **survives**. `search_results`,
   `selected_offer` survive. `commits_landed` empty — nothing to compensate.
3. Replay: `create_workbench` mints fresh `workbench_id` (old UUID never re-presented — it was
   also `single_use`-tracked); `add_offer` re-consumes the SURVIVING `price_offer`; I2: the new
   `workbench_offer` **replaces** the old in run state (keyed by lineage — old workbench lineage
   is unreadable).
4. Nested failure: if `add_offer` now returns `code:4101` (search cache — 12 min GDS — died
   during the dawdle), row D3 → `rewind(to: search)` (3→2). I1 kills `search_results`,
   `selected_offer` (pseudo), `price_offer`, and the fresh workbench lineage. Replay re-crosses
   `select`: `rematch.key` (carrier+flights+times+brand+cabin+price) re-selects silently on
   exact match; drifted price ⇒ `on_ambiguous: model` picks fresh (new I3 selection), and any
   price delta resurfaces deterministically at `price_review`/`booking_consent` — the consent
   gates, not the rematch, are the money guard.
5. Cursor proceeds to `commit_workbench`; `entry_gate: booking_consent` fires again by
   construction (arrival after rewind) — stale consent never carries over.

### 9.2 The two-stage strain: price change / expiry on the HELD booking at ticketing

Setup: commit #1 landed yesterday (`commits_landed = [commit_workbench]`,
`reservation_locator = ABC123`, PaymentTimeLimit tomorrow). Chain resumes the ticket sub-chain:
`create_postcommit_workbench → get_reservation → add_fop → add_payment → commit_ticketing`.
NDC rail; commit returns the documented price-changed error.

1. Row D8 matches → `rewind(to: standalone_reprice)`, flagged `rejected_before_execution` (error
   *instead of* ticketing — assumed no partial issuance, UNKNOWN-3), so the exactly-once attempt
   for `commit_ticketing` is NOT consumed; redispatch budget `safe_reject_redispatch: 2`.
2. I5 — **commits are facts**: `reservation_locator` and the landed `commit_workbench` are
   untouched by invalidation. What dies: nothing upstream — `standalone_reprice` mints a fresh
   priced offer INSIDE `workbench2_id` (or a fresh workbench if the old one 4367'd meanwhile —
   then D1 handles it and `fop_id`/`payment_link` die with it and are re-minted; I4: if
   `workbench2_id` had been spent by the ambiguous commit attempt it is never re-committed —
   fresh post-commit workbench instead).
3. `reprice_consent` gate parks the run (audience user): new price shown. Accept → cursor
   re-approaches `commit_ticketing`; its `entry_gate: ticketing_payment_consent` **re-fires**
   (new price ⇒ new money consent — C8.4 shape). Decline → `dead_end(user_declined_reprice)`;
   `commits_landed` non-empty ⇒ compensation per config: `cancel_held_booking` (or planner
   elects to let it lapse at PaymentTimeLimit — the compensator notes both).
4. Contrast the harder sibling: `get_reservation` precondition finds `now > PaymentTimeLimit`.
   No rewind target exists — a PNR is not re-mintable (I5), and the "refresh" of
   `reservation_locator` is a whole new Chain-A run. Verdict:
   `dead_end(reason: held_booking_expired, permanent: false)` — a *business-timescale* dead end;
   the planner starts a NEW booking run at `search`. Compensation: `cancel_held_booking` fires
   per config but degenerates gracefully if the airline already auto-cancelled (probe shows
   cancelled ⇒ compensator's confirmation satisfied without dispatch... strictly: probe-first
   compensation is an implementation nicety; config-level the compensator runs and its D-row/
   probe tolerate "already cancelled" — exact already-cancelled signal is UNKNOWN-8).

Both walkthroughs were traced against the §4 edges; every death above follows an explicit
`derived_from` path (or I4/I5 rule), none is asserted.

---

## 10. UNKNOWNs & assumptions (human answers needed before implementation)

1. **UNKNOWN-1 (the big one) — locator-free probe for a lost commit #1 response.** No documented
   endpoint searches reservations by passenger/date/agency queue in the TripServices Flights
   set. Our probe procedure (retrieve-workbench state inference + operator) has a genuinely
   ambiguous branch: workbench gone = committed OR expired. **Q to Travelport/API owner: is
   there a queue/list/search API (or support procedure) to find a just-created PNR without its
   locator? Does the custom `traceID` header let support locate the transaction?** Until
   answered, the sweep escalates to operator at 12h and the run must not rebook automatically.
2. **UNKNOWN-2 — idempotency:** confirmed absent from docs, but verify with Travelport that no
   gateway-level replay protection exists (e.g. duplicate-PNR detection by host).
3. **UNKNOWN-3 — `rejected_before_execution` claims.** D5 is well-supported (docs: warning
   returned "instead of" booking, StatusAir "Rejected"). D6 ("the commit fails" — no booking
   implied), D8 (error "instead of" ticketing), D9 (FOP-missing validation) are *inferred*
   no-side-effect rejects. **Q: does Travelport guarantee no PNR/ticket/charge exists after
   each of these errors?** Any "no" ⇒ that row becomes `reconcile` and consumes the attempt.
4. **UNKNOWN-4 — void semantics:** exact payload field proving a ticket is voided (probe
   signal); error text/code for out-of-window void; whether void also cancels the itinerary
   segments or leaves the (now unticketed) booking alive — determines whether Chain B must
   chain `cancel_held_booking` after `void_ticket`; behavior of voiding an already-voided
   ticket; voidability of exchange-reissued tickets.
5. **UNKNOWN-5 — uncommitted workbench inertness:** docs show discard + 30-min self-expiry and
   never mention external effects, but do NOT explicitly say "no host PNR/segment activity
   occurs before commit". If any carrier holds inventory at Add Offer, `add_offer` reclassifies
   toward commit and this config changes materially.
6. **UNKNOWN-6 — exact signal when committing an expired/consumed workbench** (assumed
   4100/4367 family). Needed to keep D2's reconcile routing honest.
7. **UNKNOWN-7 — AirPrice offer expiry signal:** no dedicated "price offer expired" code found;
   currently falls to unattributable-staleness handling (deepest-candidate rewind) / default
   rows. Capture the real code in the error corpus.
8. **UNKNOWN-8 — cancel_held_booking:** fees (assumed none), idempotency on re-cancel, the
   "already cancelled" signal, and whether GDS Reservation Cancel works when the airline
   already auto-cancelled at ticketing-time-limit.
9. **UNKNOWN-9 — all latency/timeout numbers are assumed** (nothing documented). Calibrate
   against real traffic; the 200s commit timeouts especially (too tight ⇒ manufactured
   reconciles).
10. **UNKNOWN-10 — exchange compensation:** is a reissued ticket voidable in a fresh void
    window (assumed yes per general void rules)? Any documented un-exchange?
11. **UNKNOWN-11 — minor doc conflicts to verify on current versions:** (a) GDS automated
    refunds "Not supported (under development)" per webhelp ExchangeRefundGuide.htm — confirm
    current status on the developer portal before wiring the GDS refund dead-end; (b) Exchange
    Eligibility method GET vs POST (endpoints list vs API-ref extraction); (c) Add Offer path
    spelling `offers/buildfromcatalogofferings` (endpoints list) vs `buildfromcatalogproductofferings`
    (some guide prose).
12. **Assumed handle-graph choices:** rematch key composition (§3.4); `price_review` gate is a
    product choice not a provider mandate; seat/ancillary/EMD steps deliberately OUT OF SCOPE
    for this compile (documented to exist; add as optional mints later — they ride the same
    workbench and inherit D1/D3 rows).
13. **Empty `staleness.detect` handles:** `price_offer` (UNKNOWN-7), `fop_id`/`payment_link`/
    `traveler_data` (die with their workbench — detection rides the workbench codes),
    `reservation_locator`/`ticket_numbers` (durable by design), selection pseudo-handles
    (refresh = re-select). Each is either durable or covered by a parent's detect + the
    unattributable-staleness rule; none silently undetectable beyond UNKNOWN-7.

---

## 11. Boundary recap (model touchpoints — SPEC §5 audit)

| Touchpoint | Where | SPEC §5 allowance |
|---|---|---|
| Pick an offer from search results | `select` (I3 pseudo-step) | "Choosing among returned options" — recorded as derivation |
| Pick an exchange option | `select_exchange` | same; `on_ambiguous: gate` sends money-ambiguity to the user |
| Propose new dates/cabin/airports on empty search | `empty.route` repair fields | repair proposals on listed intent fields only; code validates |
| Propose traveler-detail fixes | repair to `add_traveler` fields | intent fields only |
| Phrase gate payloads & dead-end explanations | all gates / dead_ends | phrasing from machine-readable reason |
| — | — | Model may NOT: answer any gate above (all are `user`/`operator`); touch `workbench_id`, locators, ticket numbers, OfferIDs (opaque aliases only); extend budgets; reclassify D-rows; trigger void/refund/exchange commits |
| `ticket_id` / instrument entering intent | `cancel_remedy_choice`, `ticketing_payment_consent` outcomes | gate-validated params bound to declared fields — the one sanctioned opaque-value path (SPEC §2.6) |
| `accept_changes` / `pay_later` constants | gate `set` assignments | config-declared constants riding declared repairs, `proposal_source: code` |

No signal routes to a model decision anywhere in §6 (checked row by row).

---

## 12. Compiler's notes — where the spec strained (for the spec authors)

1. **Held booking with a business TTL.** The spec's expiry machinery (staleness.detect +
   refresh) is built for *handles*; a held booking is a **landed commit whose usefulness
   expires**. I5 forbids invalidating it; `refresh` cannot point at "re-run the whole previous
   chain". The honest encoding — precondition on the downstream chain + `dead_end(permanent:
   false)` + planner-level rebook — works but lives in three places at once (precondition on
   `get_reservation`, D-row intent note, compensator note). A first-class "commit output with
   declared business expiry → structured dead_end + planner hint" would collapse that. The
   `ttl_hint` concept ALMOST fits (PaymentTimeLimit is even per-object, like Stripe's
   `capture_before`) but there is no legal place to hang it on a durable handle whose staleness
   must never trigger a rewind.
2. **Two dependent commits in one chain worked — but only because `compensation_order:
   reverse` happens to be business-correct here** (void ticket, then cancel booking). The
   deeper coupling — compensating commit #2 (void) *re-enables* the compensator of commit #1
   (a ticketed booking can't be Reservation-Cancelled; a voided one presumably can, UNKNOWN-4)
   — is inexpressible: compensator preconditions can gate, but there's no way to declare
   "compensator B's viability depends on compensator A's outcome". Reverse order encodes it
   implicitly; a comment carries the load.
3. **Rail variants (GDS vs NDC).** Same business chain, different endpoints/rows/gates per
   content source, decided at *selection time* (which offer the user picked). The spec has no
   variant/parameterization concept; two configs from one template is the workaround. Fine for
   two rails; would smell at five.
4. **Two-step commit fit was a pleasant surprise**: `rejected_before_execution` +
   `safe_reject_redispatch` + a gate whose outcome `set`s an intent flag that a
   `request_invariants` rule translates into changed dispatch params — all pre-existing
   primitives — express Travelport's "commit returns Not Processed, then commit again with the
   indicator off" exactly, including the trap that redispatching with the indicator still true
   discards the workbench. The strain: `request_invariants` is documented as "declared
   constants... never model- or runtime-varied", and here it's intent-conditional
   (`accept_changes`). I stretched it; the spec may want an explicit "intent-derived dispatch
   parameter map" rather than my conditional invariant.
5. **Probe asymmetry across the two commits** is the design's center of gravity: commit #2's
   probe is strong (locator known pre-dispatch), commit #1's is weak (locator IS the output).
   The spec handles this via correlation records + sweep + operator, but nothing in the schema
   *forces* the compiler to notice that a commit's confirmation depends on an output that the
   ambiguous case doesn't have. A lint — "probe input must be derivable without the commit's
   own output; else require correlation-record keys sufficient for out-of-band lookup" — would
   have caught it mechanically instead of by elicitation discipline.
6. **Compensation sub-chains can't branch.** "Void if within window else refund-quote else
   operator" needed: a gate inside the sub-chain + rail-specific chains + structured dead_end
   reasons. Expressible, but scattered; a declarative window→compensator ladder (the
   `window` field is prose today) would centralize the single most safety-relevant timing rule
   in ticketing (same-day-void).

---

## 13. Self-check (SKILL Step 7)

- [x] Every commit/compensator declares `confirmation` — all unkeyed ⇒ all have probes
      (`commit_workbench`, `commit_ticketing`, `commit_exchange`, `void_ticket`,
      `cancel_ticket`, `cancel_held_booking`); no `by_key_replay` anywhere (no keys exist).
      All unkeyed ⇒ `attempts: 1` (§8).
- [x] Correlation record persisted before dispatch + cancellation shield on every unkeyed
      commit; record lives in the durable run store; `traceID` request header stamped for
      out-of-band correlation (P3).
- [x] Both selection pseudo-handles carry `rematch` specs (`selected_offer`,
      `selected_exchange_offer`).
- [x] In-place commits (`commit_ticketing`, `commit_exchange`, `void_ticket`, `cancel_ticket`,
      `cancel_held_booking`) have content-based confirmation signals + explicit compensation
      stance (`none` cases name their pre-commit gates in `ordering_note`).
- [x] Handles consumed by commits: `workbench*_id` have `staleness.detect` (4100/4367/4434);
      `reservation_locator`/`ticket_numbers` durable; `payment_link` dies with its workbench
      (parent detect covers) — residual gaps declared in §10.13.
- [x] `single_use`: `workbench_id`→`commit_workbench`, `workbench2_id`→`commit_ticketing`,
      `workbench3_id`→`commit_exchange` — exactly one consumer each (I4 honored in §9.2).
- [x] Model touchpoints: selections, listed-field repair proposals, phrasing only (§11); no
      handle is model-repairable; no verdict is model-decided.
- [x] Every documented error code mapped (§6 D-rows) or in `known_unmatched`.
- [x] Every gate: audience + all outcomes mapped to verdicts + timeout (§7).
- [x] Empty routes exist where Q9 said valid-empty (search, exchange_search; AirPrice
      no-brands = documented degraded ok, not empty).
- [x] Invalidation walkthroughs traced by hand against §4 edges (§9).
- [x] Payload business blockers are `preconditions` (eligibility None, PaymentTimeLimit passed,
      already-ticketed, non-HK status), not overloaded `empty`.
- [x] No TTL used as a timer: 30m workbench / 12m-34m cache / PaymentTimeLimit all hints or
      payload-checked preconditions; expiry is discovered via 4367/4101/commit errors (P2).
- [x] No budget in prompt text; external mode rides the snapshot (§8).
- [x] Compensation ordering stated: `reverse` (ticket before booking) + replace-before-
      compensate ordering_notes on both A-chain commits.

*Deliverable ends. Implementation (runtime + client code) is a separate task consuming this
document — starting with the UNKNOWN-1 and UNKNOWN-3 answers, which change verdict rows if they
come back adverse.*

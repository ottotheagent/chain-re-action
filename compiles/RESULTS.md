# Blind-Compile Results — Four Public APIs

Created: 2026-07-28
Last Updated: 2026-07-28

The spec and skill in this repo are tested by **blind compiles**: an agent is
given SPEC.md + SKILL.md + CONFORMANCE.md and a target provider's **public
documentation only** — nothing else, no reference implementation, no prior
compiles — and must produce a full chain config by following the skill end to
end. The four configs in this directory are those artifacts, unedited. Every
claim in them carries an evidence tag and a doc URL; everything the docs
don't answer is an explicit UNKNOWN, not a guess.

The exercise is scored on three things: does the compile reproduce the
safety core (exactly-once commits, no unknown-outcome replays, lineage
invalidation, budgets in code)? Is the UNKNOWN list *right* — genuine
epistemic humility rather than missed elicitation? And what friction does
the domain expose in the spec itself — each compile's complaints became the
next spec version's changelog.

## The four runs

| Provider | Compiled against | Scope | Result |
|---|---|---|---|
| **Duffel** (flights) | v0.3.1 | instant order, hold order, order change | 1,340 lines, 32 cited doc pages, 16 UNKNOWNs |
| **Stripe** (payments) | v0.3.1 | manual capture, idempotency semantics, refunds | 1,316 lines, 92 citations, 11 UNKNOWNs |
| **Travelport** (GDS flights) | v0.3.2 | search→price→workbench→held booking→ticket; void/refund; exchange | 3 chains, ~15 cited pages |
| **Booking.com Demand** (stays) | v0.3.2 | search→availability→preview→create; cancel/modify | 2 chains, 14 UNKNOWNs |

### Duffel — the correlation-record vindication

Duffel has **zero idempotency keys** and documents "do not retry — may
result in a duplicate order" on its create call. The spec's P3 mechanics
(persist a correlation record before dispatching an unkeyed commit; recover
lost responses out-of-band) mapped almost verbatim onto Duffel's own
documented recovery machinery (`offer_id` + order webhooks + a list-orders
filter). The compile also classified Duffel's **hold order** as a commit —
it holds real airline space and mints a user-visible PNR — even though it
self-expires, exercising the rubric that consequence, not lifetime, decides
effect class. Cancellation compiled as a quote→consent-gate→confirm
`compensation.chain`.

### Stripe — the example-slayer

SPEC's worked example B is a card-payment chain, so the skill's provenance
rule ("worked examples are never evidence") got its hardest test: the
compiler had to re-derive everything from docs.stripe.com. It found **11
material errors in the worked example** — declines arrive as HTTP 402 error
objects (not a 2xx status field); hold expiry varies ~2 days to ~30 days by
network and *terminally cancels* the payment intent; several error codes in
the example simply didn't exist; cross-run key dedup is unsound past
Stripe's ~24h key retention; the async-refund-failure story was missing
entirely. Example B was rewritten from this compile, and `key_retention` /
`replay_semantics` / full-response `empty.detect` entered the spec because
of it. Stripe's compile also used zero selection pseudo-steps — evidence
that the selection machinery is config-optional, not ceremony.

### Travelport — the structural stress test

The hardest public shape attempted: booking is a **workbench session**
(30-minute single-use mint) whose commit produces a **held booking** — an
airline-confirmed PNR with no payment and a ~24h business lifetime — and
ticketing is a *second*, dependent commit that mutates the PNR in place.
Two findings shaped v0.4: landed commits whose *usefulness* expires needed a
first-class home (`output.business_expiry`), and the compile proposed the
**probe-input rule** — a confirmation probe that requires the commit's own
output (a get-by-locator read) cannot run in exactly the lost-response case
it exists for. Bonus: Travelport's native two-step-commit protocol mapped
exactly onto the spec's `no_side_effect` + `safe_reject_redispatch` + gate
`set` machinery, and nearly all of its business errors ride HTTP 200
payloads — vindicating the rule that payload predicates outrank transport
success.

### Booking.com — the production comparison

Booking.com is the one provider where we could also score the compile
against a production integration we operate. The comparison: **6 MATCH / 2
PARTIAL / 3 DIVERGE** — and the matches were the hard-won parts (the blind
compile independently derived detection-on-use token expiry with code-owned
rewind, pay-later avoidance, replacement-before-cancel, content-based
re-matching after re-search). Two of the diverges were defects in the
production client, not the compile: an unkeyed create call that a shared
HTTP client was silently auto-retrying on 5xx/timeouts (the canonical
double-booking vector — **found by this exercise, since fixed, with a
regression test**), and an assume-failed lost-response path. The
comparison's verdict: the blind compile was "materially safer than the
production client it was tested against."

Domain surprises worth reading the compile for: a **contractual** 15-minute
`order_token` TTL (most providers document none); the provider shipping its
own correlation mechanism (a partner-defined `label` echoed in reporting —
now what the production fix uses); and *different idempotency regimes per
vertical* under one endpoint family (cars get request dedup, accommodations
don't).

## What the exercise proved

1. **The abstractions generalize.** Two idempotency regimes (Duffel and
   Travelport: no keys anywhere; Stripe: keyed everything with retention
   cliffs), two evidence regimes (empirical notes vs rich public docs), four
   failure shapes — the same primitives compiled all of them, and the safety
   core reproduced unchanged in every run.
2. **UNKNOWN discipline is the product.** Across all four compiles, not one
   headline UNKNOWN was a guess dressed up as fact; several (e.g. "is 429
   rejected before processing?") turned out to be exactly the questions the
   provider or production experience later answered.
3. **The spec is made of its own findings.** Every version from v0.2.2 to
   v0.4 is traceable to a compile complaint: see SPEC.md's changelog. The
   compiles in this directory are the receipts.
4. **It pays for itself.** One of these compiles found a real
   double-booking vector in a production system that had been running for
   months — by knowing what a safe client *must* look like before reading a
   line of the unsafe one.

## Reproducing

Give an agent SPEC.md, SKILL.md, CONFORMANCE.md, and a provider's public
docs; forbid everything else; ask for the full skill pipeline (fit test →
elicitation log with per-answer evidence tags → handle graph → actions →
verdict table + known_unmatched → gates → policy → invalidation
walkthroughs → UNKNOWNs → boundary recap → self-check). Compare what comes
back against these four for calibration. The compiles here were produced
against the spec versions noted in their headers; the current spec is newer
— a fresh compile should find *less* friction than these did, and anything
new it finds is a contribution.

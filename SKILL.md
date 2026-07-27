---
name: chain-compile
description: Compile a chained-action config (per chain-re-action SPEC.md) for a multi-step stateful API before writing any client code. Use when building or reviewing an API client / business layer for flows like search→price→book, authorize→capture, or provision→attach→activate — where later steps mutate external state, intermediate handles expire, partial failure needs compensation, or "nothing available" is a valid answer. Triggers on "chain config", "booking flow", "multi-step API client", "saga", "compensation", "idempotency design".
---

# chain-compile — produce a chain config for a new domain

Created: 2026-07-27
Last Updated: 2026-07-27

You are COMPILING a chain, not improvising one at runtime. The output is a
config (per `SPEC.md` in this repo) plus design notes — authored once,
reviewed by a human, and then implemented/executed deterministically.
Everything you cannot answer from documentation or empirical evidence becomes
an explicit `UNKNOWN` in the output, never a guess.

Read `SPEC.md` fully before starting. The spec's principles P1–P6 are
normative; your config must not violate them.

## Step 0 — Fit test (refuse early)

A chain is the WRONG abstraction when any of these hold. Say so, explain
which condition fired, and recommend the simpler alternative instead of
producing a config:

- **Single call.** One request/response, even a slow one → a retry wrapper
  with backoff. No chain.
- **Fully idempotent CRUD.** Every step is keyed or naturally idempotent,
  nothing expires, order doesn't matter → plain retries; a chain adds
  ceremony without safety.
- **No handles, no expiry, no commit.** If no step mints server-side state
  that a later step consumes, there is nothing to invalidate or rewind to.
- **The provider already runs the state machine.** If the API is "submit
  intent, poll one status field" (many provisioning APIs), the provider owns
  recovery; you need a poller with a deadline, not a chain.
- **A human drives every step interactively** with minutes between steps and
  no automated recovery expectation → workflow/UI problem, not a chain.
- **True distributed transaction across providers** where partial states are
  unacceptable and no compensation exists → this spec can't save you; needs
  escrow/2PC at the business level. Flag it; do not paper over with a chain.

Also refuse to finalize (deliver a draft marked BLOCKED instead) when
elicitation Q2/Q3 below cannot be answered for a commit step: a commit with
unknown double-call behavior AND no way to observe whether it landed is
unimplementable safely (SPEC §2.1 makes that an invalid config).

## Step 1 — Elicitation

Interview the API owner, the provider docs, and any empirical-behavior notes.
Docs lie by omission; prefer empirical evidence, and record the source of
each answer (`documented` / `observed` / `assumed`). A claim that appears
ONLY in SPEC.md's worked examples — including when your target domain IS an
example's domain — is tagged `spec-example`: never evidence; route its
signals to `known_unmatched` until independently sourced. When the corpus is
itself an empirical-notes document, tag restated API shapes (endpoints,
fields, enums) as `documented` and behavioral claims (TTLs, timing, failure
behavior) as `observed`. Two more tags for public-docs corpora:
`documented-by-omission` — an otherwise-complete reference that nowhere
mentions a feature (no idempotency key documented anywhere) is evidence of
absence, stronger than `assumed`; and treat search-snippet-attributed claims
as unverified until the page itself is fetched (else downgrade to
`assumed`). Public docs also arrive per PAGE, not per step — elicit per page
and reassemble; the per-step log is the output format, not the reading
order. Ask, per candidate step:

**Effect classification**
1. If this call runs twice with the same inputs, what exists afterwards?
   (nothing new → read; two expiring server-side artifacts → mint; two
   durable/billable artifacts → commit)
2. If the process dies mid-call — request sent, response never read — what
   might exist? Can it cost money, hold inventory, or be user-visible?
   (yes → commit, and Q3 becomes mandatory)
   2b. Does this commit create a NEW durable object, or MUTATE an existing
   one in place? In-place → set `mutates`; confirmation must compare CONTENT
   (existence proves nothing); and if no undo exists, name the pre-commit
   safety gate.
3. How would you find out whether an ambiguous attempt landed? Which read
   endpoint, filtered how? Is there a lag before it's queryable? Is there a
   webhook, and what's its worst-case delay?

**Idempotency**
4. Does the provider accept a client idempotency key on this call? What's
   its scope and retention window? What happens on key-replay with different
   params?
5. If no key: is the call naturally idempotent (same input → same result, no
   duplicate)? Prove it, don't assume it.

**Handles & staleness**
6. What tokens/IDs does this call return that later calls consume? For each:
   what EXACT signal appears when a downstream call uses it after expiry
   (HTTP status, error code, message substring)? What's the observed
   lifetime? (Record as `ttl_hint` only — SPEC P2.)
7. Which earlier step is the cheapest way to re-mint this handle?
8. Is this output valid only relative to some other choice or handle
   (coupling)? E.g. prices valid only for a selected option, quotes tied to a
   session. These become `derived_from` edges — probe hard, this is where
   silent corruption lives.
   8b. For every point where the model/user selects among returned options:
   which intent-level fields stably identify "the same option" across a
   re-search? (Provider option ids usually are NOT stable.) These become the
   selection pseudo-handle's `rematch.key`.

**Empty vs error**
9. Can this call succeed with "nothing available / declined / no capacity"?
   How does that look in the payload (NOT the status code)? What should the
   run do next — which intent fields could a user plausibly change?
   9b. Do the docs prescribe a deterministic fallback ("retry without filters
   before reporting no availability")? These compile to `auto_repairs`
   (proposal_source: code), not model repairs.

**Failure taxonomy**
10. List every documented error code for this endpoint. For each: is it
    identical on every attempt (deterministic) or can it clear on its own
    (transient)? Any codes that mean "your handle is stale" vs "your request
    is wrong" vs "the provider is down"?
11. Are there mid-chain events that require a HUMAN decision by business
    rule, not by failure — price/terms changed, challenge required, approval
    needed? These become gates, never error handling. Also: which read
    payloads encode business blockers (eligibility flags, status fields) even
    though the call succeeded? These become `preconditions` — not `empty`,
    not error handling.

**Compensation**
12. For each commit: what business action undoes it? What does undoing cost,
    and does the cost change with time (void window vs refund penalty)? Is
    the compensator itself async or non-idempotent?
13. If this chain replaces an existing thing (rebook, re-provision): must the
    new commit land BEFORE the old one is compensated? (Default yes — never
    leave the user with nothing.)

**Latency & finality**
14. Per step: expected and worst-case latency? Which step is slowest?
15. After the final commit: when is the result truly final vs eventually
    consistent? What confirms it (webhook, poll), and after what deadline do
    you hand off to a human? Can a SUCCEEDED result be revoked later (a
    refund that fails days after creation succeeded, an airline-initiated
    change)? → `revocation` watch.

**Integrator-mandated steps**
16. Which NON-provider steps does your own stack mandate before a commit —
    fraud screening, payment tokenization proxies, compliance checks? These
    compile as actions/gates in the chain like any provider step, with their
    own failure taxonomy — never as invisible middleware.

## Step 2 — Build the handle graph

- One `handle` entry per token, plus a **pseudo-handle for every selection
  that downstream results couple to** (SPEC I3) — the pick needs
  `derived_from` edges (an itinerary choice that returns/seat maps derive
  from); such selections are derivations and drive invalidation exactly like
  tokens. A pick that is merely a request parameter with no derived lineage
  (a seat preference) is an intent field, not a pseudo-step.
- Draw `derived_from` edges from Q8 answers. Then verify by contradiction:
  for each handle pair (A, B) with no edge, ask "if A is re-minted, is B
  really still valid?" Missing edges are the classic bug.
- Mark `single_use` handles (consumed/spent by a commit — SPEC I4).
- Every selection pseudo-handle needs a `rematch` spec (Q8b): the composite
  intent-level key code uses to re-establish the selection after a rewind,
  plus `on_ambiguous` routing. Without it, rewind-through-a-selection is
  unimplementable.
- Every handle gets `staleness.detect` from Q6. A handle with an empty
  `detect` list must be either durable (never expires) or flagged `UNKNOWN:
  staleness undetectable` — which makes every consumer's stale-handle row
  fall to the table's default rows (fail closed).

## Step 3 — Classify actions

For each step, emit the full Action record (SPEC §2.1). Non-negotiables:
- effect from Q1/Q2 evidence, not from the endpoint's name. ("checkout",
  "validate", "initiate" are usually mints; "create", "book", "capture",
  "provision" are usually commits — but verify, some "create" calls mint.)
  Self-expiry does not make a mint: if a repeated artifact is consequential
  to the user (a credit hold, a reservation that blocks inventory), it is a
  commit however short-lived.
- Selection points compile as `effect: select` pseudo-steps: no dispatch,
  input = the results handle they pick from, output = the selection
  pseudo-handle (whose `rematch` spec comes from Q8b).
- Pre-commit consent that must survive recovery replays (new price ⇒ new
  consent) goes on the commit's `entry_gate`, not on a prior step's
  precondition — entry gates re-fire on every arrival by construction.
- Split inputs into `intent` (model-repairable) vs `handles` (code-owned).
  When in doubt, a field is a handle: the cost of wrongly letting the model
  repair a token is far higher than a round-trip to re-mint it.
- EVERY commit and compensator declares `confirmation`. Unkeyed or
  async-finality → a probe (+ signal + async deadline) is mandatory; if Q3
  had no answer, stop: BLOCKED. Keyed synchronous → a probe, or an explicit
  `by_key_replay` note stating why key replay suffices.
- `empty` block on every step where Q9 said yes, with a concrete
  `route` — usually `repair` listing the intent fields from Q9.
- In-place commits (Q2b): set `mutates`, a content-based confirmation signal,
  and an explicit compensation stance (`none` requires the pre-commit gate
  named in `ordering_note`).
- Unkeyed commits inherit the P3 mechanics: correlation record persisted
  before dispatch + cancellation shield. Say WHERE the record lives — it must
  survive process death.
- **Name the enforcement point for every budget.** attempts=1 is only real
  if the transport layer's DEFAULT retry policy is pinned to zero for commit
  dispatches — shared HTTP clients often carry a default that silently
  applies when per-call options are omitted, and that exact mechanism
  produced a real production double-booking vector (the pin existed for one
  provider's client and was never applied to another's). The config states
  the mechanism (per-call option, dedicated client, middleware), and the
  conformance test must exercise the transport boundary, not a mock above it.
- Confirmation probes must be runnable without the commit's own output
  (SPEC §2.1 probe rule); if only get-by-locator reads exist, the
  correlation record carries the lookup keys, or the compile is BLOCKED.
- Q11 business blockers → `preconditions`; Q9b prescribed fallbacks →
  `auto_repairs`.

## Step 4 — Author the verdict table

Start from the generic skeleton (SPEC §3) and prepend domain rows:
- Every error code from Q10 appears exactly once, mapped to a verdict. Codes
  you consciously leave unmapped go in a `known_unmatched` list — they'll hit
  the fixed default rows (reconcile-if-commit-in-flight, else dead_end).
- Stale-handle signals (Q6) → `rewind(to: <handle>.refresh)`. Prefer the
  cheapest rewind: expired downstream handle with live upstream → re-mint
  just the dead one. Order matters: staleness and payload-level rows must
  precede unconditional transport success (a 200 can carry an expiry code).
- "The chosen option is gone but its source results may live" (fare bucket
  sold out, plan no longer offered) → `reselect(to: <selection>)`, never a
  `repair` with empty fields.
- Human-decision events (Q11) → `gate(...)` with declared outcomes, audience,
  and timeout verdict. Consent about money is `audience: user`, never model.
- Deterministic provider rejects → `dead_end`. Resist the urge to retry
  things that are documented to fail identically. Structure every `dead_end`
  reason: `permanent: bool`, and a `retry_after_hint` for business-timescale
  blocks (airline control, in-progress ticketing) so the planner can
  re-attempt as a NEW run later.
- Payload preconditions (Q11) get their own rows evaluated on ok results —
  don't fold eligibility blocks into `empty`.
- Ambiguous transport failure on a commit: `retry` iff keyed (same key),
  `reconcile` iff unkeyed. This single row is where double-charges come from;
  get it right.
- You may NOT route any signal to a model decision. If you feel the need to,
  what you actually want is either a `gate` (human decides) or a `repair`
  (model proposes intent values, code executes) — pick one.

## Step 5 — Set policy

Defaults (override with justification):

```yaml
per_step:
  read:  {attempts: 3, backoff: {base: 500ms, factor: 2, jitter: full, max: 8s}}
  mint:  {attempts: 2, backoff: {base: 1s,   factor: 2, jitter: full, max: 10s}}
  commit: {keyed: {attempts: 3}, unkeyed: {attempts: 1}}   # unkeyed=1 is not overridable
per_chain: {max_rewinds: 3, max_repairs: 2,
            wall_clock: <max(3× Σ latency_hints, 2× slowest step timeout)>,
            gate_timeout: <product decision>}
```

Set `timeout` per step from Q14 worst-case, not average. The slowest step is
usually the commit; give it headroom rather than letting a tight timeout
manufacture ambiguous outcomes.

`wall_clock` covers active execution including reconcile probe attempts and
compensation sub-chains; time parked at gates or in
reconciling-awaiting-async is excluded (SPEC §2.4) — provider finality can
take hours and must not blow the run clock.

Budgets are config, enforced by code — never prompt text. If the executor is
a planner loop (external mode, SPEC §2.3), budgets ride in the durable run
snapshot and continuation runs decrement the same counters.

## Step 6 — Emit

Produce ONE document with these sections, in order. For multi-chain compiles
(e.g. booking + round-trip + exchange for one provider): still one document —
share the elicitation log and the verdict-table core; give each chain its own
handle graph, actions, domain rows, gates, and walkthrough subsections; add a
cross-chain notes section for shared handles and interactions (e.g. a
booking's `pnr_id` feeding the exchange chain).

1. **Chain summary** — the business outcome, the step list, one paragraph.
2. **Fit-test verdict** — why a chain is the right abstraction here (which
   properties fired: handles? expiry? commit? compensation?).
3. **Elicitation log** — every Step-1 answer with its source tag
   (`documented` / `observed` / `assumed`). This is the evidence trail
   reviewers audit; a config without it is an assertion, not a compile.
4. **Handle graph** — YAML per SPEC §2.2 + a one-screen ASCII sketch of the
   `derived_from` edges.
5. **Actions** — YAML per SPEC §2.1, full field-level.
6. **Verdict table** — domain rows, then a pointer to the generic skeleton;
   `known_unmatched` list.
7. **Gates** — YAML per SPEC §2.6.
8. **Policy** — YAML per SPEC §2.4.
9. **Invalidation walkthrough** — pick the two most dangerous rewind/repair
   scenarios and trace exactly which handles/collections die (SPEC §4 rules
   applied by hand). If you can't produce this section, your `derived_from`
   edges are wrong.
10. **UNKNOWNs & assumptions** — every `assumed`-sourced answer from Step 1,
    every empty `staleness.detect`, every `compensation: none`. Each with the
    question a human must answer before implementation.
11. **Boundary recap** — a table of every point where the model participates
    in THIS chain (selections, repair proposals, gate phrasing) proving each
    one is inside SPEC §5's allowances.

## Step 7 — Self-check before delivering

- [ ] Every commit/compensator declares `confirmation` (probe, or a
      `by_key_replay` justification on keyed synchronous commits); unkeyed
      commits additionally have `attempts: 1`.
- [ ] Every unkeyed commit has a correlation record persisted before dispatch
      and a cancellation shield on the in-flight attempt (SPEC P3).
- [ ] Every selection pseudo-handle has a `rematch` spec.
- [ ] Every in-place commit (`mutates`) has a content-based confirmation
      signal and an explicit compensation stance.
- [ ] Every handle a commit consumes: either durable or has `staleness.detect`.
- [ ] Every `single_use` handle is consumed by exactly one step.
- [ ] Every model touchpoint is a selection, an intent-field repair proposal,
      or gate phrasing — nothing else. No handle field is model-repairable.
- [ ] Every documented error code is mapped or listed in `known_unmatched`.
- [ ] Every gate has an audience, all outcomes mapped to verdicts, a timeout.
- [ ] Empty-result routes exist wherever Q9 said empty is valid.
- [ ] The invalidation walkthrough (§9 of your output) is consistent with the
      `derived_from` edges — trace it, don't assert it.
- [ ] Payload business blockers are `preconditions`, not overloaded `empty`.
- [ ] No TTL is used as an enforcement timer anywhere (hints only).
- [ ] No budget or retry limit lives in prompt text.
- [ ] Every budget names its enforcement point; transport default retries
      are pinned to zero for commit dispatches, and the test exercises the
      transport boundary.
- [ ] Every confirmation probe is runnable without the commit's own output.
- [ ] Landed-commit business deadlines (`business_expiry`) and post-ok
      revocation paths (`revocation`) are declared where the domain has them.
- [ ] A variant axis is declared when the provider runs parallel rails or
      per-vertical regimes.
- [ ] Compensation ordering stated for any replace/rebook flow.

Deliver the config. Implementation (runtime, client code) is a separate task
that consumes this document.

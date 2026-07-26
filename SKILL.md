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
each answer (`documented` / `observed` / `assumed`). Ask, per candidate step:

**Effect classification**
1. If this call runs twice with the same inputs, what exists afterwards?
   (nothing new → read; two expiring server-side artifacts → mint; two
   durable/billable artifacts → commit)
2. If the process dies mid-call — request sent, response never read — what
   might exist? Can it cost money, hold inventory, or be user-visible?
   (yes → commit, and Q3 becomes mandatory)
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

**Empty vs error**
9. Can this call succeed with "nothing available / declined / no capacity"?
   How does that look in the payload (NOT the status code)? What should the
   run do next — which intent fields could a user plausibly change?

**Failure taxonomy**
10. List every documented error code for this endpoint. For each: is it
    identical on every attempt (deterministic) or can it clear on its own
    (transient)? Any codes that mean "your handle is stale" vs "your request
    is wrong" vs "the provider is down"?
11. Are there mid-chain events that require a HUMAN decision by business
    rule, not by failure — price/terms changed, challenge required, approval
    needed? These become gates, never error handling.

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
    you hand off to a human?

## Step 2 — Build the handle graph

- One `handle` entry per token, plus a **pseudo-handle for every model/user
  selection** among returned options (SPEC I3) — selections are derivations
  and drive invalidation exactly like tokens.
- Draw `derived_from` edges from Q8 answers. Then verify by contradiction:
  for each handle pair (A, B) with no edge, ask "if A is re-minted, is B
  really still valid?" Missing edges are the classic bug.
- Mark `single_use` handles (consumed/spent by a commit — SPEC I4).
- Every handle gets `staleness.detect` from Q6. A handle with an empty
  `detect` list must be either durable (never expires) or flagged `UNKNOWN:
  staleness undetectable` — which makes every consumer's stale-handle row
  fall to the table's default rows (fail closed).

## Step 3 — Classify actions

For each step, emit the full Action record (SPEC §2.1). Non-negotiables:
- effect from Q1/Q2 evidence, not from the endpoint's name. ("checkout",
  "validate", "initiate" are usually mints; "create", "book", "capture",
  "provision" are usually commits — but verify, some "create" calls mint.)
- Split inputs into `intent` (model-repairable) vs `handles` (code-owned).
  When in doubt, a field is a handle: the cost of wrongly letting the model
  repair a token is far higher than a round-trip to re-mint it.
- `idempotency.mode: none` + `effect: commit` → `confirmation` is REQUIRED
  (probe + signal + async deadline). If Q3 had no answer, stop: BLOCKED.
- `empty` block on every step where Q9 said yes, with a concrete
  `route` — usually `repair` listing the intent fields from Q9.

## Step 4 — Author the verdict table

Start from the generic skeleton (SPEC §3) and prepend domain rows:
- Every error code from Q10 appears exactly once, mapped to a verdict. Codes
  you consciously leave unmapped go in a `known_unmatched` list — they'll hit
  the fixed default rows (reconcile-if-commit-in-flight, else dead_end).
- Stale-handle signals (Q6) → `rewind(to: <handle>.refresh)`. Prefer the
  cheapest rewind: expired downstream handle with live upstream → re-mint
  just the dead one.
- Human-decision events (Q11) → `gate(...)` with declared outcomes, audience,
  and timeout verdict. Consent about money is `audience: user`, never model.
- Deterministic provider rejects → `dead_end`. Resist the urge to retry
  things that are documented to fail identically.
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
per_chain: {max_rewinds: 3, max_repairs: 2, wall_clock: <~3× sum of latency_hints>, gate_timeout: <product decision>}
```

Set `timeout` per step from Q14 worst-case, not average. The slowest step is
usually the commit; give it headroom rather than letting a tight timeout
manufacture ambiguous outcomes.

## Step 6 — Emit

Produce ONE document with these sections, in order:

1. **Chain summary** — the business outcome, the step list, one paragraph.
2. **Fit-test verdict** — why a chain is the right abstraction here (which
   properties fired: handles? expiry? commit? compensation?).
3. **Handle graph** — YAML per SPEC §2.2 + a one-screen ASCII sketch of the
   `derived_from` edges.
4. **Actions** — YAML per SPEC §2.1, full field-level.
5. **Verdict table** — domain rows, then a pointer to the generic skeleton;
   `known_unmatched` list.
6. **Gates** — YAML per SPEC §6.3 shape.
7. **Policy** — YAML per SPEC §2.4.
8. **Invalidation walkthrough** — pick the two most dangerous rewind/repair
   scenarios and trace exactly which handles/collections die (SPEC §4 rules
   applied by hand). If you can't produce this section, your `derived_from`
   edges are wrong.
9. **UNKNOWNs & assumptions** — every `assumed`-sourced answer from Step 1,
   every empty `staleness.detect`, every `compensation: none`. Each with the
   question a human must answer before implementation.
10. **Boundary recap** — a table of every point where the model participates
    in THIS chain (selections, repair proposals, gate phrasing) proving each
    one is inside SPEC §5's allowances.

## Step 7 — Self-check before delivering

- [ ] Every `commit` is keyed, or has `attempts: 1` AND a confirmation probe.
- [ ] Every handle a commit consumes: either durable or has `staleness.detect`.
- [ ] Every `single_use` handle is consumed by exactly one step.
- [ ] Every model touchpoint is a selection, an intent-field repair proposal,
      or gate phrasing — nothing else. No handle field is model-repairable.
- [ ] Every documented error code is mapped or listed in `known_unmatched`.
- [ ] Every gate has an audience, all outcomes mapped to verdicts, a timeout.
- [ ] Empty-result routes exist wherever Q9 said empty is valid.
- [ ] The invalidation walkthrough (§8 of your output) is consistent with the
      `derived_from` edges — trace it, don't assert it.
- [ ] No TTL is used as an enforcement timer anywhere (hints only).
- [ ] Compensation ordering stated for any replace/rebook flow.

Deliver the config. Implementation (runtime, client code) is a separate task
that consumes this document.

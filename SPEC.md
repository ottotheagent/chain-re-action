# Chained Actions — Specification

Created: 2026-07-27
Last Updated: 2026-07-27
Status: Draft v0.2.1 (pre-implementation)

A domain-agnostic model for multi-step API chains where later steps mutate
external state, intermediate results expire, and "nothing available" is a valid
answer rather than an error. The goal: make such chains **recoverable**
(partial failure has a defined path back), **bounded** (deterministic budgets,
never model-decided), and **replayable** (every attempt is a typed trace
record; a run can resume across process restarts).

This spec defines the data model and semantics. It does not define a runtime.
It is language-agnostic; examples use YAML for configs and JSON for records.

---

## 0. Vocabulary

- **Chain** — an ordered sequence of Actions toward one business outcome
  (e.g. "book this flight"). One execution of a chain is a **Run**.
- **Action** — one typed call against an external system.
- **Handle** — an opaque token minted by an Action and consumed by later
  Actions (`search_id`, `checkoutResponseId`, `payment_intent_id`). Handles
  are code-owned; the model never reads, writes, or fabricates them.
- **Intent** — the human-meaningful inputs to the chain (route, dates, cabin,
  payment instrument). Intent fields are the only inputs the model may repair.
- **Verdict** — the classified outcome of one attempt of one Action.
- **Gate** — a defined pause point where the chain must wait for an external
  decision (user consent, 3DS challenge, operator approval). A gate is not a
  failure.
- **Signal** — the raw observable from an attempt: HTTP status, structured
  error code, timeout, payload predicate.

## 1. Design principles

These are normative. A conforming implementation MUST preserve them.

**P1 — The deterministic line.** The model may (a) choose among options an
Action returned, (b) propose repairs to *intent* fields, (c) answer gates it
is authorized to answer, (d) phrase outcomes to the user. Deterministic code
owns: step sequencing, attempt budgets, backoff, stop conditions, signal →
verdict classification, handle values, invalidation, idempotency enforcement,
and compensation execution. The model can never extend a budget, re-classify
a signal the verdict table already matched, retry a commit, or touch a handle.
Budgets live in config and are enforced by code — a budget stated in prompt
text is not a budget. Config-declared deterministic fallbacks (`auto_repairs`,
§2.1) count as code decisions even though they change intent fields.

**P2 — Expiry is discovered, not scheduled.** Providers rarely expose handle
TTLs, and observed lifetimes drift. Staleness is modeled as detection signals
on use (e.g. HTTP 410, `ITINERARY_FARE_EXPIRED`), plus an optional
`ttl_hint` used ONLY for planning (e.g. "don't start a commit with a
14-minute-old handle"). Implementations MUST NOT hard-fail on a timer.

**P3 — Unknown outcome is first-class.** A commit whose response was lost
(timeout, connection drop) may have succeeded. The only safe next step is a
**confirmation probe**, never a retry and never silent failure. Every
commit-class Action MUST declare how to find out whether it landed.
Two mechanics follow: (a) **correlation record** — before dispatching an
unkeyed commit, persist a durable record of the attempt (run id, intent
snapshot, status=PENDING) so out-of-band confirmation (webhook, sweep,
operator) can correlate the provider's result back to this run even if the
process dies; (b) **cancellation shield** — once a commit request is
dispatched, the attempt runs to completion; caller timeouts or cancellation
yield `reconcile`, never an abandoned in-flight commit.

**P4 — Empty is an answer.** "No inventory", "no capacity", "card declined"
are valid results with their own routing, distinct from transport or provider
errors.

**P5 — Compensation is a business action, not a rollback.** A compensator has
its own fees, windows, failure modes, and may itself be async. Compensation
ordering is a config decision (e.g. book-new-before-cancel-old), never an
implicit stack unwind. Most non-commit steps need no compensator: expiring
handles are self-compensating.

**P6 — Invalidation is a graph, not a field.** Handles declare what they were
derived from. Re-minting a handle (or repairing intent) transitively
invalidates everything downstream, and invalidated collections are
**replaced, never appended**.

---

## 2. Data model

### 2.1 Action

```yaml
action:
  id: string                      # unique within the chain
  description: string             # for humans and for the model's situational awareness

  effect: read | mint | commit | compensate
  # Classify by the DOUBLE-CALL TEST (what exists after calling twice) — never
  # by the endpoint's name, and never by whether the call returns a handle.
  # Reads may return handles to cached result state (a search_id) and remain
  # reads; what makes a mint is dedicated mutable session state prepared for a
  # later commit (a fare session, a hold).
  # read       — repeating has no external consequence; free to repeat.
  # mint       — repeating creates distinct expiring server-side artifacts you
  #              must track (a session, a hold, a quote handle). Repeat-safe:
  #              worst case is orphaned state that self-expires. Never billed,
  #              never user-visible.
  # commit     — durable, externally visible mutation. Exactly-once semantics
  #              required. May CREATE a new object (PNR minted, payment
  #              captured, VM created) or MUTATE an existing one in place
  #              (modify-booking) — declare `mutates` for the latter.
  # compensate — business-level undo of a prior commit. Is itself
  #              commit-class: non-idempotent unless keyed, needs its own
  #              confirmation.

  mutates: handle_id              # commit only, optional: this commit modifies
                                  # an existing durable object in place instead
                                  # of minting a new handle. output.handles may
                                  # be empty; the confirmation signal MUST then
                                  # be content-based (observe the mutated
                                  # state), because existence proves nothing.

  input:
    intent:                       # model/user-repairable fields
      <field>: {type: string|int|date|money|enum[...]|object, required: bool}
    handles: [handle_id, ...]     # code-owned; resolved by the runtime from
                                  # the run's handle store, never by the model

  output:
    payload: <type>               # decision-relevant data returned to caller/model
    handles:                      # handles this action mints (see 2.2)
      - handle_id
    empty:                        # only for actions where empty is valid
      valid: bool
      detect: <payload predicate> # e.g. "results.length == 0"
      route: <verdict>            # ok | gate | repair | dead_end — what
                                  # ok(empty) maps to; `ok` = proceed to the
                                  # next step degraded (e.g. no seat map →
                                  # book seatless), recorded in the trace

  preconditions:                  # payload-level business blockers on ok results
    - when: <payload predicate>   # e.g. "no ticket with EXCHANGEABLE_BY_OBT"
      verdict: <ok|gate|dead_end|repair(...)>   # ok = matched but proceed
                                  # degraded (skip the dependent capability),
                                  # recorded in the trace
      reason: string
  # Evaluated after ok, before advancing. Distinct from `empty`: the call
  # succeeded and returned data — the data says the business action is blocked.

  auto_repairs:                   # config-declared deterministic fallbacks
    - trigger: <signal or payload predicate>
      fields: <intent transform>  # e.g. "drop optional filters"
      once: bool                  # default true: fire at most once per run
  # When a trigger matches, the runtime issues repair(proposal_source: code)
  # with the declared transform — no model round-trip. Counts against
  # max_repairs. Use for provider-prescribed fallbacks ("retry without
  # filters before reporting no availability").

  idempotency:
    mode: keyed | natural | none
    # keyed   — provider accepts a client idempotency key; safe to retry on
    #           ambiguous failure with the SAME key.
    # natural — the call is naturally idempotent (PUT-with-id, read).
    # none    — no dedup mechanism exists. For commit actions this FORCES
    #           per-step attempts=1 and REQUIRES `confirmation`.
    key_scope: run | intent       # keyed only: key derived from run_id+step
                                  # (retry-safe) or from intent hash (also
                                  # dedupes across runs)

  confirmation:                   # REQUIRED if effect=commit and mode!=keyed;
                                  # RECOMMENDED for all commits
    probe: action_id              # a read-class action that can observe the result
    signal: <payload predicate>   # how the probe shows the commit landed
    async:                        # if finality arrives out-of-band
      channel: webhook | poll
      deadline: duration          # after which → escalate(operator)
    sweep:                        # background reconciler for correlation
      interval: duration          # records stuck PENDING (P3): re-probe each
      escalate_after: duration    # interval; past escalate_after → operator

  compensation:                   # only meaningful for effect=commit
    action: action_id | none      # `none` must be explicit + justified in notes
    window: <constraint>          # e.g. "within 24h of ticketing → void, no
                                  # penalty; after → refund w/ penalty"
    ordering_note: string         # e.g. "commit replacement BEFORE compensating
                                  # original" (never leave user with nothing)

  timeout: duration               # per-attempt wall clock
  latency_hint: duration          # expected p95, for planning only
```

Rules:
- A `commit` with `idempotency.mode: none` and no `confirmation` is an
  **invalid config**. There is no safe behavior for its ambiguous failures.
- `input.handles` values never appear in model-visible payloads except as
  opaque aliases (e.g. `flt_x7a2`), and the model can only echo aliases back.
- Unkeyed commits REQUIRE the P3 mechanics: a correlation record persisted
  before dispatch, and a cancellation shield on the in-flight attempt.
- A commit with `mutates` and `compensation.action: none` MUST name its
  pre-commit safety mechanism (usually a consent gate) in `ordering_note` —
  when no undo exists, the protection has to sit before the commit.

### 2.2 Handle

```yaml
handle:
  id: string
  minted_by: action_id
  derived_from: [handle_id, ...]  # dependency edges; drives invalidation (§4)
  single_use: bool                # true if consuming it in a commit spends it
  rematch:                        # selection pseudo-handles only (I3): how the
    key: [intent-level fields]    # runtime re-establishes the selection after a
                                  # rewind re-mints what it was derived from.
                                  # Provider option ids are NOT stable across
                                  # re-search; the key must be intent-level
                                  # (carrier + flight numbers + times + fare
                                  # basis — not itinerary_id).
    on_ambiguous: model | gate    # exact key match → code re-selects silently;
                                  # no/multiple matches → model re-selects (a
                                  # new selection per I3), or gate to the user
  staleness:
    detect: [signal, ...]         # e.g. [http:410, code:ITINERARY_FARE_EXPIRED]
                                  # observed ON USE of this handle downstream
    ttl_hint: duration | unknown  # planning hint ONLY; never enforced by timer
    refresh: action_id            # cheapest step that re-mints this handle;
                                  # the rewind target when it dies
```

### 2.3 Verdict

Exactly one verdict per attempt. Produced by the deterministic classifier
(§3); recorded in the trace with its source.

| Verdict | Meaning | Who decides next input |
|---|---|---|
| `ok` | Attempt succeeded; payload available. `ok(empty)` flags a valid empty result and follows the action's `empty.route`. | — |
| `retry` | Same step, same inputs, after deterministic backoff. Only for transient transport/provider faults on non-commit steps (or keyed commits). | code |
| `rewind(to: action_id)` | A handle died; re-execute from the cheapest step that re-mints it, then replay downstream. Inputs unchanged. | code |
| `repair(to: action_id, fields: [...])` | Rewind that additionally requires a proposed change to named *intent* fields (new date, other payment instrument). The proposal comes from model, user, or a config-declared `auto_repair` (`proposal_source: code`); the rewind mechanics stay code-owned. | model/user/config propose, code executes |
| `gate(id)` | Pause for an external decision (consent, challenge, approval). Resumes with one of the gate's declared outcomes. | user/operator |
| `reconcile` | Outcome unknown (lost response on a commit). Run the action's confirmation probe procedure. Resolves to `ok`, `rewind`, `dead_end`, or `escalate(operator)` on deadline. | code |
| `dead_end(reason)` | Terminal for this run. Provider rejected deterministically, or budgets exhausted. `reason` is machine-readable and carries `permanent: bool` plus optional `retry_after_hint` — an airline-control hold or in-progress ticketing is a *temporal* dead end the planner may re-attempt as a NEW run hours later; a fraud decline is permanent. Report; run compensation per config if partial commits exist. | code |

Notes:
- Your classic `replan` is split: intra-chain replan = `rewind`/`repair`;
  abandoning the chain for a different plan is above this spec — the chain
  returns `dead_end` with a machine-readable reason and the planner (model)
  decides what to do *outside* the chain.
- `reconcile` is deliberately not collapsible into `retry`: retrying an
  unconfirmed commit is the canonical double-booking bug.

**Execution modes.** `rewind`/`repair` may be executed in two conforming ways:
- *Internal*: the runtime moves the cursor and replays within the same process.
- *External (planner-executed)*: the run terminates with a typed outcome
  `restart_required {rewind_to, repair_fields?, reason, budgets_remaining}` and
  the caller (typically an LLM planner loop) starts a continuation run.
  Conforming ONLY if: the signal is typed and machine-readable (never prose
  the model must interpret), the continuation decrements the SAME budgets via
  the durable run snapshot, and the continuation's trace links to the original
  (`continuation_of`) so the logical run stays replayable. A retry limit
  stated in prompt text is not a budget.

### 2.4 Policy

All values are static config. None may be model-decided (P1).

> **OPEN — deliberately unsettled.** The default attempt counts below and the
> retry→rewind escalation threshold (when same-step retry stops being worth it
> and the run should rewind/replan instead) are provisional placeholders. They
> must be calibrated against a real production error corpus — which signal
> classes actually recover on same-step retry vs require rewind — not decided
> in the abstract. Treat the *shape* of the policy as normative and the
> *numbers* as unreviewed defaults. See §8.

```yaml
policy:
  per_step:                       # overridable per action
    read:    {attempts: 3, backoff: {base: 500ms, factor: 2, jitter: full, max: 8s}}
    mint:    {attempts: 2, backoff: {base: 1s, factor: 2, jitter: full, max: 10s}}
    commit:
      keyed:   {attempts: 3, backoff: {base: 2s, factor: 2, jitter: full, max: 30s}}
      unkeyed: {attempts: 1}      # NON-NEGOTIABLE for idempotency.mode=none
    compensate: <same shape as commit>
  per_chain:
    max_rewinds: int              # total rewind+repair budget for the run
    max_repairs: int              # subset cap: repairs need model/user round-trips
    wall_clock: duration          # whole-run budget, excludes time parked at gates
    gate_timeout: duration        # default for gates that omit their own
                                  # timeout; a gate's timeout.after takes
                                  # precedence over this chain-level value
  escalation:                     # who may be asked what, in order
    - {audience: model,    may: [propose repair fields, choose among returned options]}
    - {audience: user,     may: [answer consent gates, approve repairs with cost]}
    - {audience: operator, may: [resolve reconcile deadline, unstick pending commits]}
```

### 2.5 Trace

One record per attempt, append-only, plus a resumable run snapshot. Traces are
eval-grade: they must let you re-derive every decision without the original
process.

> **OPEN — deliberately unsettled.** The *field semantics* below are
> normative: what must be captured, and the invariants at the end of this
> section. The *exact serialization* — JSON shape, field naming, encoding,
> storage layout — is intentionally unpinned; the examples are illustrative
> only. Pin it when the eval tooling and the real error corpus exist to
> validate it against, not before. See §8.

```json
// attempt record
{
  "run_id": "…", "chain_id": "…", "seq": 17,
  "step_id": "revalidate", "attempt": 1,
  "t_start": "…", "t_end": "…",
  "input": {
    "intent_snapshot_hash": "…",       // full snapshot stored once per repair
    "handles": {"checkout": "cko_9f2…"}// aliases; raw tokens in a sealed store
  },
  "signal": {"transport": "http:200|http:410|timeout|…",
              "provider_code": "FARE_PRICE_CHANGED|null",
              "payload_predicates": ["empty:false"]},
  "verdict": {"type": "gate", "arg": "price_change_consent"},
  "verdict_source": "table",           // "table" | "default" — never "model"
  "handles_minted": ["booking_id"],
  "handles_invalidated": [],
  "budget_after": {"step_attempts_left": 0, "rewinds_left": 2, "wall_ms_left": 412000}
}
```

```json
// run snapshot (rewritten after every attempt; enables resume after restart)
{
  "run_id": "…", "chain_id": "…", "state": "at_gate|running|reconciling|done|dead_end",
  "cursor": "create_pnr",
  "intent": { … },                     // current, post-repairs
  "live_handles": {"search_id": {"alias": "…", "minted_at": "…", "spent": false}},
  "commits_landed": ["create_pnr"],    // what compensation would need to unwind
  "pending_gate": null,
  "budgets": { … }
}
```

Requirements:
- Long-lived states (`reconciling`, `at_gate`) may persist for **days**;
  snapshot + attempt log MUST be sufficient to resume with zero in-memory state.
- Raw handle values and payment data live in a sealed store keyed by alias;
  the trace itself is safe to feed to evals and to the model.

### 2.6 Gate

```yaml
gate:
  id: string
  audience: user | operator | model   # model only if the config explicitly
                                      # grants it; gates about money or consent
                                      # are never model-answerable
  payload: <type>                     # what the audience is shown to decide
  outcomes:
    <name>:
      params: {field: type}           # optional: values the answer carries
                                      # (e.g. option_id of the chosen refund)
      bind: [intent_field, ...]       # params are written into these intent
                                      # fields before the verdict executes
      verdict: ok | repair(...) | rewind(...) | dead_end
  timeout: {after: duration, verdict: <verdict>}
```

An outcome with `params` is how a gate answer feeds the chain: the user's
choice (a cancellation option, an alternate seat) binds into intent fields
and typically rides a `repair`. `outcome_name: verdict` is a valid shorthand
when there are no params.

An outcome verdict of `ok` means the raising step's original success path
stands: the run advances past the step that raised the gate, using that
step's already-produced output. It never re-executes the raising step — use
`rewind` in the outcome if re-execution is what you mean.

---

## 3. Verdict decision table

The classifier is an **ordered list of matchers**, first match wins, evaluated
by deterministic code. The generic skeleton every domain table starts from:

| # | Signal | Applies to | Verdict | Rationale |
|---|---|---|---|---|
| 1 | Transport success + `empty.detect` true | actions with `empty.valid` | `ok(empty)` → `empty.route` | empty is an answer (P4) |
| 2 | Transport success | any | `ok` | |
| 3 | Handle-staleness signal (per handle `staleness.detect`) | any consumer of that handle | `rewind(to: handle.refresh)` — shallowest dead handle wins | cheapest re-mint (P2) |
| 4 | Rate limit (`http:429`, provider throttle code) | any | `retry` | transient by contract |
| 5 | Transport fault (5xx, timeout, conn reset) | `read` / `mint` / keyed `commit` | `retry` | safe to repeat |
| 6 | Transport fault (timeout, conn drop, ambiguous 5xx) | unkeyed `commit` | `reconcile` | may have landed (P3) |
| 7 | Provider "terms changed" (price, schedule) | per config | `gate(consent)` | user decision, not failure |
| 8 | Provider deterministic reject (validation, policy, fraud, unsupported) | any | `dead_end` | identical on every attempt |
| 9 | Sub-resource unavailable but chain can proceed degraded (seat gone, ancillary failed) | per config | `gate(degrade_consent)` or auto-degrade + `ok` | config decides |
| 10 | **Unmatched** | commit in flight this run | `reconcile` | safe default |
| 11 | **Unmatched** | otherwise | `dead_end` | fail closed; NEVER model-classified |

Rules:
- Every provider-documented error code MUST appear in the domain table or be
  consciously left to rows 10–11 (list them in a `known_unmatched` note).
- Rows 10–11 are fixed. A config may not route unmatched signals to `retry`,
  `repair`, or a model decision.
- When multiple handles report stale in one signal, rewind to the refresh
  target of the **shallowest** (earliest-minted) dead handle; its re-mint
  invalidates the rest anyway (§4).

### Verdict → next-state transitions

```
ok           → evaluate `preconditions`; first match → its verdict; else
               advance cursor to next step; if last step → done
ok(empty)    → follow empty.route (ok | gate | repair | dead_end;
               ok = advance degraded)
retry        → same step after backoff; attempts exhausted → escalate per
               effect class: read/mint → rewind(refresh of newest input
               handle) if rewinds remain, else dead_end; keyed commit → dead_end
               (the exhaustion→rewind threshold is OPEN — calibrate against
               the production error corpus, see §2.4 and §8)
rewind(to)   → decrement rewind budget; invalidate per §4; cursor = to;
               when replay re-crosses a selection pseudo-step, apply its
               `rematch` spec (exact key match → code re-selects; else per
               on_ambiguous)
repair(to,f) → decrement rewind+repair budgets; obtain proposal (matching
               `auto_repair` → apply its declared transform, no round-trip;
               else model or user per escalation ladder); validate proposal
               touches ONLY listed intent fields; apply; invalidate per §4;
               cursor = to
gate(id)     → park run (state=at_gate); resume with a declared outcome
               (each outcome maps to ok | repair | dead_end); timeout →
               gate's timeout verdict
reconcile    → mark correlation record RECONCILING; run confirmation probe
               (probe follows read policy; results matched via the record):
               landed   → treat original attempt as ok, advance
               not-landed → rewind(refresh of consumed handle) if budget, else dead_end
               still unknown at async.deadline → escalate(operator),
               state=reconciling; `sweep` keeps re-probing until escalate_after
dead_end     → if commits_landed non-empty → execute compensation per config
               (each compensator is itself a commit-class action with its own
               confirmation); report machine-readable reason; state=dead_end
```

---

## 4. State invalidation rules

**I1 — Transitive invalidation.** When handle H is re-minted (rewind) or its
minting step's intent inputs change (repair), every handle with a
`derived_from` path to H is invalidated, plus every cached payload derived
from those handles. Invalidation is atomic with the rewind: no step may
observe a mix of old and new lineage.

**I2 — Replace, never append.** Re-executing a step replaces its prior
outputs in the run state. Appending mixed-lineage results (e.g. return
flights coupled to two different outbound selections) silently corrupts
pricing and MUST be structurally impossible: run state keys collections by
the lineage of the handles that produced them.

**I3 — Selection is derivation.** When the model/user selects among returned
options, the selection is recorded as a derived pseudo-handle
(`selected_outbound`, derived_from: [search_id]). Downstream results coupled
to a selection (coupled return search, seat map for an itinerary) declare it
in `derived_from`, so changing the selection invalidates them via I1. On
replay after a rewind, the selection is re-established per its `rematch` spec
— never by re-presenting the old provider option id, which is not stable
across re-search.

**I4 — Spent handles.** A `single_use` handle consumed by a commit attempt is
spent even if the attempt's outcome is unknown; a subsequent rewind must
re-mint it. Never re-present a spent handle to the provider.

**I5 — Commits don't invalidate.** Landed commits are facts, not cache. They
are unwound only by explicit compensation, never by I1. `commits_landed` in
the snapshot is the authoritative list of what compensation must consider.

**I6 — Intent repair scope.** A repair to intent fields invalidates all
handles minted by the repair's target step and everything downstream of it
(via I1). Handles minted upstream survive iff their minting inputs didn't
include the repaired fields — the config's `derived_from` edges plus each
action's `input.intent` list determine this statically.

**I7 — Fan-out lineage.** Concurrent runs may share a read prefix (one
search, N candidate selections compared in parallel). Shared handles are
stored once and referenced by alias from each run; every result collection is
keyed by the full lineage that produced it, so branches cannot
cross-contaminate — a return list coupled to branch A's outbound can never be
read under branch B, even when the flights look near-identical (they differ
in price). The join/comparison across branches happens in the planner, above
this spec.

---

## 5. The LLM / deterministic boundary

| Concern | Owner | Notes |
|---|---|---|
| Step sequencing, cursor | code | chain config is the program |
| Executing rewinds/restarts | code, or planner in external mode | conforming only via typed `restart_required` signal + snapshot-carried budgets (§2.3) |
| Attempt budget, backoff, wall clock | code | policy (§2.4); model can't extend; never encoded in prompt text |
| Signal → verdict classification | code | table (§3); unmatched rows fixed |
| Handle values, storage, lineage | code | model sees opaque aliases only |
| Invalidation | code | §4, atomic with rewind |
| Idempotency keys, attempts=1 on unkeyed commits | code | |
| Compensation execution + ordering | code | per config |
| Choosing among returned options | model | selection recorded as derivation (I3) |
| Proposing values for `repair` fields | model or user | code validates: listed intent fields only |
| Answering gates | user/operator (model only if gate explicitly grants it) | money/consent gates are never model-answerable |
| Phrasing outcomes, dead-end explanation | model | from machine-readable reason |
| Deciding what to do AFTER dead_end | model (planner) | outside this spec; new chain or give up |

Enforcement, not convention: proposals arriving from the model are validated
against the schema (field allow-list, types) before the runtime applies them.
A model message can never *be* a state transition; it can only be an input to
one.

---

## 6. Worked example A — Spotnana air booking chain

Failure shape: expiring opaque handles at every hop, no idempotency key on
the commit, price can change mid-chain, coupled sub-searches (round-trip),
async finality via webhook. Values below are empirically observed, not
contractual (P2).

### 6.1 Handles

```yaml
handles:
  search_id:
    minted_by: search
    derived_from: []
    staleness: {detect: [http:410, code:SEARCH_EXPIRED], ttl_hint: ">25m", refresh: search}
  selected_itinerary:              # pseudo-handle (I3): the model's pick
    minted_by: select              # a model-selection step, not an API call
    derived_from: [search_id]
    rematch: {key: [carrier, flight_numbers, departure_times, fare_basis],
              on_ambiguous: model} # provider itinerary ids are NOT stable
                                   # across re-search; re-match by content
    staleness: {detect: [], ttl_hint: unknown, refresh: select}
  return_results:                  # round-trip only: returns coupled to outbound pick
    minted_by: coupled_return_search
    derived_from: [search_id, selected_itinerary]
    staleness: {detect: [http:410, code:SEARCH_EXPIRED], ttl_hint: ">25m", refresh: coupled_return_search}
  checkout_id:
    minted_by: checkout
    derived_from: [search_id, selected_itinerary]
    staleness: {detect: [http:410, code:ITINERARY_FARE_EXPIRED], ttl_hint: 10-15m, refresh: checkout}
  seat_map_id:
    minted_by: seat_map
    derived_from: [search_id, selected_itinerary]
    staleness: {detect: [http:410], ttl_hint: ~5m, refresh: seat_map}
  initiate_booking_id:
    minted_by: initiate_booking
    derived_from: [checkout_id, seat_map_id]
    staleness: {detect: [http:410], ttl_hint: unknown, refresh: initiate_booking}
  booking_id:
    minted_by: revalidate
    derived_from: [checkout_id, initiate_booking_id]
    single_use: true               # spent by create_pnr (I4)
    staleness: {detect: [http:410], ttl_hint: unknown, refresh: revalidate}
  pnr_id:
    minted_by: create_pnr          # durable output of the commit, not expiring
    derived_from: [booking_id]
    staleness: {detect: [], ttl_hint: n/a, refresh: n/a}
```

### 6.2 Actions (abridged to the decision-relevant fields)

```yaml
actions:
  - id: search
    effect: read                   # double-call test: repeating has no external
                                   # consequence; search_id merely references a
                                   # cached result set (§2.1 rubric — a handle
                                   # does not make this a mint)
    input: {intent: {origin: airport, destination: airport, dates: date_range,
                      cabin: enum, travelers: pax_spec}}
    output:
      payload: itineraries[]
      handles: [search_id]
      empty: {valid: true, detect: "itineraries.length == 0",
              route: repair(to: search, fields: [dates, cabin, origin, destination])}
    idempotency: {mode: natural}

  - id: coupled_return_search      # round-trip only. NOT a cross-join:
    effect: read                   # returns are priced against the selected
    input: {intent: {}, handles: [search_id, selected_itinerary]}   # outbound
    output:
      payload: return_itineraries[]
      handles: [return_results]
      empty: {valid: true, detect: "return_itineraries.length == 0",
              route: repair(to: select, fields: [])}   # pick different outbound
    idempotency: {mode: natural}

  - id: checkout                   # prepares fare; does NOT lock the price
    effect: mint
    input: {handles: [search_id, selected_itinerary]}
    output: {payload: fare_summary, handles: [checkout_id]}
    idempotency: {mode: natural}   # re-checkout just re-mints; old handle self-expires

  - id: seat_map                   # optional step
    effect: read
    input: {handles: [search_id, selected_itinerary]}
    output: {payload: seat_maps[], handles: [seat_map_id],
             empty: {valid: true, detect: "seat_maps.length == 0",
                     route: ok}}   # degrade: proceed without seats
    idempotency: {mode: natural}

  - id: initiate_booking
    effect: mint
    input: {intent: {travelers: traveler_details, payment: payment_ref,
                      seats: seat_selection?},
            handles: [checkout_id, seat_map_id]}
    output: {payload: booking_prep, handles: [initiate_booking_id]}
    idempotency: {mode: natural}

  - id: revalidate                 # THE price lock; where drift surfaces
    effect: mint
    input: {handles: [checkout_id, initiate_booking_id]}
    output: {payload: {final_price: money, price_changed: bool, old: money?, new: money?},
             handles: [booking_id]}
    idempotency: {mode: natural}

  - id: create_pnr                 # the commit
    effect: commit
    input: {handles: [booking_id], intent: {trip_ref: string}}
    output: {payload: {pnr: string, confirmations: string[]}, handles: [pnr_id]}
    idempotency: {mode: none}      # provider offers NO key → attempts=1 forced
    confirmation:
      probe: get_pnr_by_trip       # read: query PNRs on the trip; the
                                   # correlation record (P3) persisted BEFORE
                                   # dispatch is what probe/webhook match against
      signal: "pnr exists for this trip with matching itinerary"
      async: {channel: webhook, deadline: 24h}   # provider webhook confirms booking;
                                                 # probe retries tolerate eventual
                                                 # consistency (PNR queryable lag)
      sweep: {interval: 12h, escalate_after: 72h}  # stuck-PENDING records → operator
    compensation:
      action: cancel_pnr
      window: "within ~24h of ticketing → void (no penalty); after → refund per fare rules"
      ordering_note: "for replace-booking flows, commit the replacement BEFORE compensating the original"

  - id: cancel_pnr                 # compensator; itself commit-class
    effect: compensate
    input: {handles: [pnr_id], intent: {option_id: string}}   # option chosen at a gate
    output: {payload: {status: enum[CANCELLED, AGENT_TASK_CREATED],
                       refund: money?, credits: money?}}
    idempotency: {mode: natural}   # cancelling a cancelled PNR is a no-op reject
    confirmation:
      probe: get_pnr_by_trip
      signal: "pnr status == CANCELLED"
      async: {channel: webhook, deadline: 72h}   # AGENT_TASK_CREATED path is async
```

### 6.3 Verdict table (domain rows before generic skeleton rows)

| # | Signal | Step scope | Verdict |
|---|---|---|---|
| D1 | `http:410` / `SEARCH_EXPIRED` | any consumer of `search_id` | `rewind(to: search)` |
| D2 | `ITINERARY_FARE_EXPIRED` (search_id still live) | revalidate, initiate_booking | `rewind(to: checkout)` — cheap transparent re-mint |
| D3 | `FARE_UNAVAILABLE` | checkout, revalidate | `repair(to: select, fields: [])` — fare bucket gone; pick another itinerary (flight may still exist) |
| D4 | `FARE_PRICE_CHANGED` (payload has old+new) | revalidate | `gate(price_change_consent)` |
| D5 | `INVALID_SEAT_OPTION` (seat sold / cabin mismatch) | initiate_booking | `gate(seat_degrade)` — outcomes: proceed seatless → `rewind(to: initiate_booking)` with seats stripped; pick another → `repair(to: initiate_booking, fields: [seats])` |
| D6 | `http:502 BOOKING_FAILED` (codeshare/split-ticket deterministic reject) | create_pnr | `dead_end` |
| D7 | `http:424 NON_RETRYABLE_THIRDPARTY_ERROR` | create_pnr | `dead_end` |
| D8 | fraud check declined (pre-commit) | pre-create_pnr guard | `dead_end` (support-only recovery) |
| D9 | traveler/payment pre-send validation fails | initiate_booking | `repair(to: initiate_booking, fields: [travelers, payment])` — user fixes profile |
| D10 | timeout / conn drop / ambiguous 5xx | create_pnr | `reconcile` (NEVER retry: unkeyed commit) |
| D11 | provider "trip already completed" | revalidate | `repair(to: revalidate, fields: [trip_ref])` via `auto_repair` (proposal_source: code): mint a fresh trip ref, no model round-trip |
| —  | …then generic rows 1–11 from §3 | | |

Gates:

```yaml
gates:
  price_change_consent:
    audience: user
    payload: {old: money, new: money}
    outcomes:
      accept:  ok            # proceed with new price (revalidate output stands)
      decline: repair(to: select, fields: [])   # or dead_end per product choice
    timeout: {after: 30m, verdict: dead_end}
  seat_degrade:
    audience: user
    outcomes: {proceed_seatless: rewind(to: initiate_booking),
               choose_other: repair(to: initiate_booking, fields: [seats])}
    timeout: {after: 15m, verdict: rewind(to: initiate_booking)}  # default: seatless
```

### 6.4 What the invalidation graph buys (round-trip case)

User changes outbound pick → `selected_itinerary` re-minted → I1 invalidates
`return_results`, `checkout_id`, `seat_map_id`, `initiate_booking_id`,
`booking_id` transitively; I2 forces the return list to be replaced (mixing
returns coupled to two different outbounds silently corrupts prices, because
the itinerary-level fare is the coupled round-trip total). A per-output TTL
cannot express any of this; the `derived_from` edges make it automatic.

### 6.5 Policy

```yaml
policy:
  per_step:
    create_pnr: {attempts: 1, timeout: 200s}     # slowest endpoint; unkeyed commit
    seat_map:   {attempts: 3}                    # flaky-tolerant read
  per_chain: {max_rewinds: 3, max_repairs: 2, wall_clock: 20m, gate_timeout: 30m}
```

Execution-mode note: the reference implementation of this chain runs in
*external* mode — a fare expiry mid-booking terminates the run with a typed
restart signal and the planner re-enters at `search`, re-establishing the
selection via `rematch`. Conforming per §2.3 iff the rewind budget rides in
the durable run snapshot, not in prompt text.

---

## 7. Worked example B — Card payment capture chain

Chosen for a deliberately different failure shape: the provider **offers
idempotency keys** (so ambiguous commit failures become `retry`, not
`reconcile`), the reservation step (**authorization hold**) has a real
compensator and a long self-expiry (~7 days), a mandatory user gate can occur
mid-chain (3DS/SCA), and the most common "failure" — card declined — is a
valid answer repairable at intent level.

Chain: `create_customer → authorize → [3DS gate] → capture`, compensators
`release_hold` and `refund`.

### 7.1 Handles

```yaml
handles:
  customer_id:
    minted_by: create_customer
    derived_from: []
    staleness: {detect: [], ttl_hint: n/a, refresh: n/a}   # durable, not expiring
  intent_id:                        # payment intent / authorization hold
    minted_by: authorize
    derived_from: [customer_id]
    staleness: {detect: [code:authorization_expired], ttl_hint: ~7d, refresh: authorize}
  client_challenge:                 # 3DS challenge session, short-lived
    minted_by: authorize            # emitted only when SCA required
    derived_from: [intent_id]
    staleness: {detect: [code:challenge_expired], ttl_hint: ~15m, refresh: authorize}
  charge_id:
    minted_by: capture
    derived_from: [intent_id]
    staleness: {detect: [], ttl_hint: n/a, refresh: n/a}
```

### 7.2 Actions

```yaml
actions:
  - id: create_customer
    effect: commit                  # durable record, but keyed → retry-safe
    input: {intent: {email: string, name: string}}
    output: {payload: customer, handles: [customer_id]}
    idempotency: {mode: keyed, key_scope: intent}   # same email+name dedupes across runs
    compensation: {action: none, ordering_note: "orphan customer records are harmless; GC out of band"}

  - id: authorize                   # mint: places a HOLD, not a charge
    effect: mint                    # repeat-safe: un-captured holds self-expire;
                                    # NOTE: unlike Spotnana mints, this one has a
                                    # real compensator (release) because holds
                                    # tie up the customer's credit limit for days
    input: {intent: {amount: money, instrument: payment_method_ref},
            handles: [customer_id]}
    output:
      payload: {status: enum[requires_capture, requires_action, declined], decline_code: string?}
      handles: [intent_id, client_challenge?]
      empty: {valid: true, detect: "status == 'declined'",
              route: repair(to: authorize, fields: [instrument])}   # decline = answer, not error (P4)
    idempotency: {mode: keyed, key_scope: run}
    compensation:
      action: release_hold
      window: "any time before capture; hold self-expires ~7d anyway"

  - id: capture                     # the commit
    effect: commit
    input: {handles: [intent_id], intent: {amount: money}}   # partial capture allowed
    output: {payload: charge, handles: [charge_id]}
    idempotency: {mode: keyed, key_scope: run}   # keys exist → ambiguous
                                                 # failure = retry SAME key,
                                                 # reconcile rarely needed
    confirmation:                    # still declared: finality is webhook-async
      probe: get_intent
      signal: "intent.status == 'succeeded'"
      async: {channel: webhook, deadline: 24h}
    compensation:
      action: refund
      window: "refund any time; funds settle T+2, refund before settlement ≈ void"

  - id: release_hold
    effect: compensate
    input: {handles: [intent_id]}
    idempotency: {mode: natural}     # releasing a released hold: no-op

  - id: refund
    effect: compensate
    input: {handles: [charge_id], intent: {amount: money}}
    idempotency: {mode: keyed, key_scope: run}
    confirmation: {probe: get_refund, signal: "status == 'succeeded'",
                   async: {channel: webhook, deadline: 72h}}
```

### 7.3 Verdict table (domain rows)

| # | Signal | Step scope | Verdict |
|---|---|---|---|
| P1 | `status == declined` + decline_code | authorize | `ok(empty)` → `repair(to: authorize, fields: [instrument])` |
| P2 | `status == requires_action` | authorize | `gate(sca_challenge)` |
| P3 | `code:challenge_expired` | capture (post-gate) | `rewind(to: authorize)` |
| P4 | `code:authorization_expired` | capture | `rewind(to: authorize)` |
| P5 | timeout / conn drop | capture | `retry` — SAME idempotency key makes this safe (contrast Spotnana D10) |
| P6 | `code:insufficient_funds` at capture (auth ok, capture > auth or funds moved) | capture | `gate(amount_consent)` outcomes: capture auth'd amount → `repair(to: capture, fields: [amount])`; abort → `dead_end` |
| P7 | fraud/risk block | authorize | `dead_end` |
| P8 | idempotency key conflict (replay with different params) | any keyed | `dead_end` — config bug, surface loudly |

```yaml
gates:
  sca_challenge:
    audience: user                  # user completes 3DS in their client
    outcomes: {completed: ok, failed: repair(to: authorize, fields: [instrument]),
               abandoned: dead_end}
    timeout: {after: 15m, verdict: dead_end}   # challenge session dies anyway
```

### 7.4 Shape contrast vs example A (why generality holds)

| Dimension | Spotnana air | Card payment |
|---|---|---|
| Idempotency on commit | none → attempts=1, `reconcile` path is load-bearing | keyed → ambiguous failure is plain `retry`; `reconcile` nearly vestigial |
| Reservation step | mint handles are free garbage (self-expire, no cost) | hold ties up real credit → mint has a genuine compensator |
| Mid-chain user gate | price-change consent (exceptional path) | 3DS (routine path, must be designed for) |
| "Nothing available" | empty search results | card declined (with machine-readable reason feeding the repair) |
| Finality | webhook after commit + eventual-consistency probe | webhook after capture; probe is authoritative immediately |

The primitives don't change across the two; only the config does. That is the
test of the abstraction.

---

## 8. Non-goals and open questions

Non-goals (v0.2): DAG/parallel step execution inside one run (fan-out =
multiple runs sharing a read prefix, with lineage keying per I7; the
comparison/join happens in the planner above this spec); cross-chain
distributed transactions; provider rate-limit budgeting across concurrent
runs; streaming/partial results.

Open questions:
- **OPEN (held back deliberately): the retry-vs-replan threshold.** Default
  attempt counts and the retry-exhaustion→rewind escalation rule are
  placeholders. Settle them against the real production error corpus (per
  signal class: observed recovery rate on same-step retry vs after rewind),
  not in the abstract. Until then, implementations treat §2.4's numbers as
  config to be tuned, and conformance tests parameterize over them.
- **OPEN (held back deliberately): trace serialization.** §2.5's field
  semantics and invariants are normative; the exact wire format is not.
  Decide it when eval tooling consumes real traces, so the format is
  validated by use rather than speculation.
- Should `gate` outcomes be allowed to carry model-proposed defaults ("auto-
  accept price increases under $5")? Current answer: no — thresholded
  auto-consent is a *policy* field if a product wants it, still deterministic.
- Compensation of multi-commit runs: strict reverse order vs config-declared
  order. Current answer: config-declared, default reverse.
- Trace retention/PII policy is deployment-specific; the alias/sealed-store
  split is the mechanism, not the policy.

---

## Changelog

- **v0.2.1 (2026-07-27)** — round-2 grammar fixes: `ok` allowed in
  `empty.route` and `preconditions.verdict` (proceed-degraded, resolving a
  self-contradiction with example A's seat_map); gate outcome `ok` semantics
  stated (raising step's success path stands, never re-executes); gate
  timeout precedence stated (per-gate wins over `per_chain.gate_timeout`).
  Deliberately marked OPEN, to be settled against the real production error
  corpus rather than in the abstract: the retry-vs-replan escalation
  threshold (§2.4) and the trace serialization format (§2.5). Added
  `CONFORMANCE.md` — implementation-agnostic behavioral requirements.
- **v0.2 (2026-07-27)** — amendments driven by the blind-compile verification
  (compile the Spotnana air chain from spec+skill+API docs only, compare
  against a production implementation; verification artifacts cite production
  internals and are kept out of this public repo):
  P3 correlation-record + cancellation-shield mechanics; external
  (planner-executed) rewind mode with typed `restart_required` signals;
  effect-class rubric (double-call test; returning a handle ≠ mint);
  `mutates` for in-place commits with content-based confirmation;
  `preconditions` for payload-level business blockers; `auto_repairs` for
  config-declared deterministic fallbacks; `rematch` spec for selection
  replay; structured `dead_end(reason)` with permanent vs business-timescale;
  formal Gate schema (§2.6) with value-carrying outcomes; I7 fan-out lineage;
  `sweep` reconciler on confirmations.
- **v0.1 (2026-07-27)** — initial draft.

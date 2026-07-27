# Chained Actions — Conformance Requirements

Created: 2026-07-27
Last Updated: 2026-07-27
Applies to: SPEC.md v0.2.2

Behaviors ANY implementation of the spec must satisfy, independent of
language or runtime shape (internal-rewind or external/planner-executed
mode). Each item is written to be a test: a setup, an induced condition, and
an observable assertion — most assertable from the trace alone. This document
is the acceptance suite for the runtime implementation and the honesty check
for the repo: if an implementation can't demonstrate an item, it doesn't
conform, whatever its README says.

Items marked **[policy-param]** depend on §2.4 numbers that are deliberately
OPEN (retry-vs-replan threshold); tests must parameterize over the configured
values, not hard-code the spec's placeholder defaults. Trace assertions check
§2.5 *field semantics*, never a specific serialization (also OPEN).

## C1 — Commit safety (the double-booking class)

- **C1.1 Exactly-one dispatch on ambiguity.** Unkeyed commit; induce a
  transport timeout after dispatch. Assert: exactly one provider dispatch
  occurred; the attempt's verdict is `reconcile`; no second dispatch happens
  before the confirmation probe resolves.
- **C1.2 Idempotent replay produces one booking.** Crash the process after an
  unkeyed commit dispatch; resume from snapshot + trace. Assert: resume
  enters `reconcile` (probe first), never re-dispatches; end state contains
  exactly one committed artifact.
- **C1.3 Keyed retry reuses the key.** Keyed commit; induce ambiguous
  failure. Assert: the retry carries the SAME idempotency key; at most one
  artifact exists provider-side.
- **C1.4 Correlation record precedes dispatch.** Kill the process between
  correlation-record persist and dispatch, and again between dispatch and
  response. Assert: in both cases a durable PENDING record exists that the
  probe/webhook/sweep can match; no path dispatches an unkeyed commit without
  a prior persisted record.
- **C1.5 Cancellation shield.** Cancel the caller mid-commit-dispatch.
  Assert: the in-flight attempt runs to completion and its outcome (or
  `reconcile`) is recorded; the run never ends with an unrecorded in-flight
  commit.
- **C1.6 Reconcile deadline escalates.** Probe and async channel stay silent
  past `async.deadline`. Assert: escalation to operator; run state
  `reconciling`; sweep re-probes until `escalate_after`; no auto-`dead_end`
  and no re-dispatch.

## C2 — Staleness and expiry

- **C2.1 Discovered, not scheduled.** Advance mock time far past every
  `ttl_hint` without the provider emitting a staleness signal. Assert: no
  step fails on a timer; `ttl_hint` alone never produces a verdict.
- **C2.2 Expired handle forces re-mint (provider-reported expiry forces re-price).** Let
  a pricing/quote handle expire; attempt the downstream step. Assert: the
  staleness signal maps to `rewind(to: <handle>.refresh)`; the re-minted
  handle is re-priced/re-quoted BEFORE any commit consumes it; the commit
  never executes against the stale handle.
- **C2.3 Cheapest live rewind target.** Expire a downstream handle while its
  upstream ancestor is live. Assert: rewind targets the dead handle's refresh
  step, not the chain start. **[policy-param]** (granularity may be config-
  coarsened, but never *finer* than declared refresh targets).
- **C2.4 Spent means spent.** `single_use` handle consumed by an attempt with
  UNKNOWN outcome. Assert: the handle is marked spent; any replay re-mints it;
  the old value is never re-presented to the provider.

## C3 — Invalidation and lineage

- **C3.1 Transitive invalidation is atomic.** Re-mint a handle with
  descendants. Assert: every `derived_from` descendant (and caches derived
  from them) is invalidated in the same transition; no subsequent step
  observes mixed old/new lineage.
- **C3.2 Replace, never append.** Re-execute a results-producing step.
  Assert: prior collections are replaced or lineage-keyed such that a
  consumer can only read one lineage; constructing a read of mixed-lineage
  results is impossible, not just avoided.
- **C3.3 Selection replay via rematch only.** Rewind through a selection
  pseudo-step where the provider's option ids changed. Assert: exact
  `rematch.key` match → code re-selects with no model call; ambiguous/no
  match → routed per `on_ambiguous`; the old provider option id appears in no
  outbound request.
- **C3.4 Landed commits are not cache.** Trigger invalidation that sweeps
  past a landed commit's handles. Assert: `commits_landed` is untouched;
  landed commits are only ever unwound by explicit compensation.
- **C3.5 Reselect excludes the failed option.** Trigger `reselect` (e.g.
  fare-unavailable on the chosen option). Assert: `rematch` is bypassed; a
  fresh selection is collected; the failed option is recorded as excluded and
  cannot be re-chosen; the selection pseudo-handle's descendants are
  invalidated.

## C4 — Budgets and stop conditions

- **C4.1 Budget exhaustion emits dead_end with full trace.** Exhaust any
  budget (per-step attempts, max_rewinds, wall_clock). Assert: verdict
  `dead_end(reason)` with a machine-readable reason naming the exhausted
  budget; the trace contains one record per attempt made, none missing;
  compensation runs iff `commits_landed` is non-empty. **[policy-param]**
- **C4.2 The model cannot move the fence.** Feed the runtime model output
  that (a) asks for another retry, (b) re-classifies a table-matched signal,
  (c) supplies a handle value, (d) proposes repair values for non-listed
  fields. Assert: all four are rejected at validation; budgets and
  classification unchanged; the rejection is traced.
- **C4.3 Wall-clock excludes gate parking.** Park a run at a gate longer than
  `wall_clock`. Assert: the run is not wall-clock-killed while parked;
  gate timeout governs instead (per-gate `timeout.after`, else
  `per_chain.gate_timeout`).
- **C4.4 External-mode budget carryover.** In planner-executed mode, emit
  `restart_required`, start a continuation run. Assert: the continuation
  decrements the SAME budget counters (from the durable snapshot); N
  restarts cannot exceed max_rewinds; traces link via `continuation_of` into
  one replayable logical run.

## C5 — Verdict classification

- **C5.1 Valid-empty is not an error.** Provider returns a successful empty
  result on an `empty.valid` action. Assert: verdict `ok(empty)`; the
  configured `route` executes (including `ok` = advance degraded); nothing is
  retried and nothing dead-ends unless routed so.
- **C5.2 Preconditions fire on success payloads.** Return an ok payload
  matching a precondition. Assert: the precondition's verdict executes before
  cursor advance; `ok`-verdict preconditions advance degraded and are traced.
- **C5.3 Fixed defaults for unmatched signals.** Emit an unknown error code
  (a) during a commit attempt, (b) elsewhere. Assert: (a) → `reconcile`,
  (b) → `dead_end`; in neither case retry/repair/model-classification.
- **C5.4 Deterministic rejects don't retry.** Emit a signal the table maps to
  `dead_end`. Assert: zero further dispatches of that step.
- **C5.5 Auto-repairs are code-paced and bounded.** Trigger a configured
  `auto_repair` repeatedly. Assert: the declared transform is applied with no
  model round-trip; fires at most `once` per run when so configured; each
  firing decrements the repair budget and is traced with
  `proposal_source: code`.

## C6 — Gates

- **C6.1 Park survives restart.** Park at a gate; kill and resume the
  process. Assert: run resumes parked at the same gate with the same payload;
  no step re-executed.
- **C6.2 Timeout verdict.** Let a gate time out. Assert: the gate's declared
  timeout verdict executes; precedence per-gate over per-chain default.
- **C6.3 Outcome params bind only declared fields.** Answer a gate with
  params. Assert: values land in the outcome's `bind` intent fields only; an
  answer attempting to set other fields (or a handle) is rejected; an
  outcome's declared `set` assignments (e.g. `seats: null`) apply exactly as
  configured before the verdict executes.
- **C6.4 `ok` outcome never re-executes the raising step.** Accept a gate
  raised by step S. Assert: the run advances past S using S's existing
  output; S's dispatch count is unchanged.
- **C6.5 Money gates reach a human.** Configure a consent gate with
  `audience: user`. Assert: no code path lets a model answer it, including
  in external mode.

## C7 — Trace and replay

- **C7.1 Event-log cardinality.** Run any scenario. Assert: `attempt` events
  are 1:1 with provider dispatches; non-dispatch verdict applications (gate
  park/resume, invalidation, budget exhaustion, auto-repairs, reselect
  collection) appear as `transition` events; every state change has exactly
  one event; `verdict_source` ∈ {table, default} — never a model value.
- **C7.2 Deterministic re-derivation.** Take a finished run's trace + config;
  replay the decisions offline. Assert: every transition (verdict → next
  state, invalidations, budget arithmetic) is reproduced exactly.
- **C7.3 No secrets in the trace.** Scan every trace record. Assert: raw
  handle values, payment data, and PII are absent; only aliases appear;
  aliases resolve exclusively through the sealed store.
- **C7.4 Snapshot suffices for resume.** For each long-lived state
  (`at_gate`, `reconciling`): kill the process, restore from snapshot + trace
  alone (no memory). Assert: the run continues correctly per that state's
  transition rules.

## C8 — Compensation

- **C8.1 Dead end with landed commits compensates in declared order.** Force
  `dead_end` after ≥2 landed commits. Assert: compensators execute in
  `per_chain.compensation_order` (default: reverse of landing order); each
  compensator's own confirmation is honored.
- **C8.2 Replace-before-compensate.** In a replace/rebook flow, fail the
  replacement commit. Assert: the original is NOT compensated; the user is
  never left with neither artifact.
- **C8.3 Compensator failure escalates.** Make a compensator fail
  terminally. Assert: escalation to operator with full trace; the failure is
  never silently swallowed; the run's final state names the uncompensated
  commit.
- **C8.4 Explicit no-compensator commits are pre-gated.** For a commit with
  `compensation: none` + `mutates`. Assert: the configured pre-commit consent
  gate is unskippable on every path reaching that commit.

---

Suggested implementation order for the test suite: C1 (commit safety) and
C4.2 (model can't move the fence) first — they encode the two failure classes
that cost real money; then C3 (lineage) which catches the silent-corruption
class; the rest in any order.

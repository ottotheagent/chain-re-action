# chain-re-action

Created: 2026-07-27
Last Updated: 2026-07-27

A spec + skill for making multi-step stateful API chains (search → price →
hold → commit) recoverable, bounded, and replayable — and for teaching coding
agents to COMPILE a chain config for a new domain instead of improvising
retry/recovery logic at runtime.

Ground truth: Otto's Spotnana air booking integration (expiring opaque
handles, no idempotency key on the commit, coupled round-trip sub-searches,
webhook finality) and Booking.com-style hotel flows.

- `SPEC.md` — the data model (Action, Handle, Verdict, Policy, Gate, Trace),
  the verdict decision table, state-invalidation rules, and two worked
  examples (Spotnana air; card payment capture).
- `SKILL.md` — the `chain-compile` skill: elicitation questions, compile
  procedure, output format, self-check, and refusal conditions.

The runtime implementation is intentionally out of scope; the spec is written
to be implemented from, language-agnostic.

The spec is validated by blind-compile verification: an agent seeing only
SPEC+SKILL and a target API's documentation compiles a chain config, which is
then compared against a production implementation of the same chain. Those
verification artifacts cite production internals and are kept out of this
public repo (gitignored).

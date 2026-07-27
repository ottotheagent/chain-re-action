# chain-re-action

Created: 2026-07-27
Last Updated: 2026-07-27

A spec + skill for making multi-step stateful API chains (search → price →
hold → commit) **recoverable, bounded, and replayable** — and for teaching
coding agents to COMPILE a chain config for a new domain instead of
improvising retry/recovery logic at runtime.

## Where this comes from

This distills the learning experience of building
[Otto](https://ottotheagent.com), an AI business travel agent, whose core is
exactly this problem: long chains of supplier API calls where later steps
mutate real-world state (tickets, payments), intermediate results expire
underneath you, "nothing available" is a valid answer rather than an error,
and the most dangerous failure is the one whose outcome you never learned
(a commit whose response was lost). The hard-won details are baked into the
spec as first-class concepts:

- expiring opaque handles at every hop, where expiry is *discovered on use*,
  never scheduled by timer
- commits with **no idempotency key**, forcing attempts=1 plus
  confirm-don't-guess reconciliation
- results that are valid only relative to an earlier selection (coupled
  sub-searches), where mixing lineages silently corrupts data
- compensation that is a business action with fees and windows, not a rollback
- the LLM/deterministic boundary: the model chooses among options and proposes
  intent repairs; code owns budgets, backoff, sequencing, handles, and stop
  conditions

If you're building an agentic application on top of any stateful supplier API
— travel, payments, provisioning, logistics — these are the same problems,
and this repo is meant to save you the incidents we learned them from.

## What's here

- `SPEC.md` — the data model (Action, Handle, Verdict, Policy, Gate, Trace),
  the verdict decision table, state-invalidation rules, and two worked
  examples with deliberately different failure shapes (a flight booking
  chain; a card payment capture chain).
- `SKILL.md` — the `chain-compile` skill: the elicitation questions to ask
  about a target API, the compile procedure, the output format, a self-check,
  and refusal conditions (when a chain is the wrong abstraction).
- `CONFORMANCE.md` — behaviors any implementation must satisfy (idempotent
  replay produces one booking, provider-reported expiry forces re-price,
  budget exhaustion emits dead-end with full trace, …). This is the
  acceptance test suite for a runtime and the honesty check for the repo.
- `compiles/` — four real blind-compile artifacts against public APIs
  (Duffel, Stripe, Travelport, Booking.com Demand), unedited, plus
  [`RESULTS.md`](compiles/RESULTS.md) — what each run found, including the
  production double-booking vector one of them caught. Use them to
  calibrate what a compile of your own API should look like.

The runtime implementation is intentionally out of scope; the spec is
language-agnostic and written to be implemented from.

## How to use it

**1. Install the skill for your coding agent.** The skill uses the portable
`SKILL.md` format: agents with native Agent Skills support discover it
directly; other agents can consume it as project context or a task prompt.
Keep `SPEC.md` next to it (or vendored in-repo) — the skill references it.

- **Claude Code** — project or personal skills directory:

  ```
  .claude/skills/chain-compile/SKILL.md      # or ~/.claude/skills/...
  ```

- **OpenAI Codex** — Codex discovers `SKILL.md` skills the same way; invoke
  with `$chain-compile` or let it auto-activate on match:

  ```
  .codex/skills/chain-compile/SKILL.md       # or ~/.codex/skills/...
  ```

  Optionally add a line to your `AGENTS.md` telling Codex to compile a chain
  config before building any multi-step API client.

- **GitHub Copilot** — [officially supports Agent Skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills)
  across the coding agent, Copilot CLI, and VS Code agent mode:

  ```
  .github/skills/chain-compile/SKILL.md      # or ~/.copilot/skills/...
  ```

  Copilot also picks up `.claude/skills/` automatically, so the Claude Code
  placement above covers Copilot too.

- **Cursor** — add `SKILL.md` to the project and reference it from your
  Cursor rules (e.g. a rule saying "before building a multi-step API client,
  follow SKILL.md to compile a chain config"), or provide it directly as
  agent context for the task.

- **Anything else** (Devin, OpenHands, your own harness, …) — paste
  `SKILL.md` as the task prompt and attach `SPEC.md`; the compile procedure
  is self-contained.

**2. Compile before you code.** When you're about to build a client/business
layer for a multi-step API, invoke the skill with your API documentation (and
any empirical behavior notes — those are usually worth more than the official
docs). The agent interviews the docs with the skill's elicitation questions
and produces a **chain config**: handle graph, per-action effect
classification, verdict table, gates, policy, invalidation walkthroughs, and
— critically — an explicit UNKNOWNs list with every assumption tagged by
evidence source.

**3. Review the compile output as a design document.** The UNKNOWNs section
is the point: each item is a question to put to the API owner before
implementation (e.g. "is there an idempotency key?" — the single
highest-leverage question for any commit). A config with unanswered
commit-safety questions is marked BLOCKED by the skill, not silently guessed.

**4. Implement from the config.** Hand the reviewed config plus `SPEC.md`'s
semantics (§3 verdict transitions, §4 invalidation rules) to your
implementation — human or coding agent — as the contract, with
`CONFORMANCE.md` as the acceptance suite. The trace format (§2.5) doubles as
your eval/replay substrate.

**5. Respect the refusal conditions.** The skill refuses when a chain is
overkill (single call, fully idempotent CRUD, provider already runs the state
machine) — if it says "use a retry wrapper," believe it.

The spec is tested through blind-compile verification: an agent seeing only
SPEC+SKILL and a target API's documentation compiles a chain config, scored
on safety-core reproduction and UNKNOWN discipline — and, where we operate a
production integration of the same chain, compared against it
dimension-by-dimension. Four public-API compiles are published in
`compiles/` with a results write-up; the production-comparison artifacts
cite internals and are kept out of this public repo (gitignored).

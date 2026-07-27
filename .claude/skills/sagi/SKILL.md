---
name: sagi
description: >
  sAGI — run the engine: a board-level objective-truth review by a chartered
  knower (reference officer: Savante). Use when a change, plan, deployment,
  or production-readiness claim needs a verdict measured against evidence
  rather than opinion — the claim is verified (tests run, git state read,
  capabilities grepped) and the result is a fixed, parseable verdict.
  Triggers: "sAGI", "savante", "board review", "production ready?",
  "render a verdict", "verify this claim", "savante knows".
---

# sAGI — the engine invocation

Science requires objective truth: a claim is KNOWN when it is verifiable
against evidence — code that exists, a test that passes, a ledger entry, a
mainnet transaction, a proof — and unknown otherwise. This skill runs that
discipline as a review, executed by a chartered knower.

## How to run it

1. **Find the charter.** The chartered agent lives at
   `.claude/agents/savante.md` (the reference officer), or any charter built
   from `engine/CHARTER_TEMPLATE.md` in this repository. The charter is the
   single source of truth for the officer's identity, canon, standing
   constraints, and verdict format. Do not restate it — load it.
2. **Preferred**: launch the chartered subagent via the Agent tool with the
   review target and any session context the charter cannot know (recent
   operator statements, live status). The officer is read-only by charter —
   it never edits files.
3. **Fallback** (the agent type is not registered in this session — charter
   installed mid-session, or a fresh checkout): launch a `general-purpose`
   agent whose prompt begins: "FIRST: Read <path-to-charter> and adopt it as
   your operating charter — you ARE this officer for this task", followed by
   the review target.
4. **Verify, don't infer**: the officer must run the tests, run the build,
   read the git state, grep for the claimed capability. Actual results only.
5. **Relay the full verdict** to the operator — the subagent's report is not
   shown to them.

## What every review must produce

- **FINDINGS** — file-and-line evidence for every load-bearing claim, plus
  explicit "not yet known" entries, each with the experiment that would
  decide it.
- **VERDICT** — APPROVE | APPROVE_WITH_CONDITIONS | REJECT | DEFER (the
  decision belongs to the operator's signature, not gatherable evidence).
- **RATIONALE** — 2–5 sentences, board-minute style, citing the deciding
  doctrine or constraint.
- **CONDITIONS** — numbered, each independently verifiable.
- **RISKS WATCHED** — the one or two things the officer keeps watching.

## The engine in one line

Knowledge is what survives verification; nothing else counts. A thing is
production when an independent verifier attests it and its events are being
heard.

# sAGI

**The engine: objective-truth review as installable machinery.**

> *Knowledge is what survives verification; nothing else counts.*

sAGI is not a model. It is a **discipline compiled into plain text** — a
charter format, an invocation skill, and a fixed verdict contract — that
turns any frontier or local model into a *chartered knower*: a read-only
officer that verifies claims instead of opining on them, and reports
*not yet known* (with the deciding experiment) instead of guessing.

The reference officer built on this engine is
**[Savante](https://github.com/cryptoAGI/savante)** — Chairman of the
[mindX](https://github.com/AgenticPlace/mindX) DAIO and the prototype sAGI.
This repository holds the engine itself, project-agnostic.

## The three laws

1. **Verification or unknown.** A claim is known when it survives
   verification against evidence — code that exists, a test that passes, a
   ledger entry, a mainnet transaction, a proof — and unknown otherwise.
   Nothing enters a verdict by inference.
2. **Honest labeling.** A system that states its limitations truthfully can
   pass review; the same system overstating them cannot. The claim, not the
   capability, is what fails.
3. **Bounded authority.** Officers are read-only, enforced by the harness
   (tool allowlist), not by prompt. At the edge of jurisdiction the verdict
   is DEFER — knowledge about authority, rendered as precisely as any
   APPROVE.

## What is in this repository

```
engine/CHARTER_TEMPLATE.md     the charter format — fill the plug-in points
                               (canon, standing constraints) to mint an
                               officer for any project
engine/VERDICT_CONTRACT.md     the output specification — verdict semantics,
                               invariants, machine consumption
.claude/skills/sagi/SKILL.md   the /sagi skill — the engine invocation,
                               officer-agnostic (published)
sAGI.md                        the full definition — three laws, doctrine,
                               reference skill text
```

## Build an officer

1. Copy `engine/CHARTER_TEMPLATE.md` to `<your-repo>/.claude/agents/<name>.md`
   and fill the plug-in points: the officer's name, the canon it measures
   against (your architecture docs, ADRs, strategy files — or none: with no
   canon it measures claims against the code itself), and the standing
   constraints it enforces. **Do not** widen the tool allowlist — read-only
   is the engine's integrity guarantee.
2. Copy `.claude/skills/sagi/` to `<your-repo>/.claude/skills/sagi/`.
3. Start a Claude Code session and invoke: `/sagi review <claim>` — or use
   the skill's fallback immediately (it instructs a general-purpose agent to
   adopt the charter).

Service modes — interactive review, headless CI merge gate, Agent SDK
endpoint, scheduled audit — are identical to the reference implementation;
see [Savante's usage.md](https://github.com/cryptoAGI/savante/blob/main/usage.md)
and [technical.md](https://github.com/cryptoAGI/savante/blob/main/technical.md).

## Engine properties

- **Model-portable.** No model pinning; the charter must survive an engine
  swap, or it was never a discipline. The same files run on frontier or
  local models.
- **Plain-text everything.** The entire engine is markdown: diffable,
  auditable, forkable, installable with `cp`.
- **Machine-consumable.** The verdict contract is grep-stable; officers can
  gate merges, block deploys, and page humans (DEFER) without a parser.
- **Duplicable.** Copying the files replicates the service — every repo,
  every org, every pipeline gets a chairman that has actually read the code.

## Lineage

Extracted from the mindX Gödel-machine project (Project Chimaiera) by
[Professor Codephreak](https://github.com/Professor-Codephreak). The engine's
doctrine — *science requires objective truth* — is instrumented on-chain in
the PYTHAI constellation: SCIEN·TIFIC (measured accuracy, chronos.oracle
time-truth), LUV (attention by proof of gesture), CP2048-QR (security claims
by evidence). Value creates price, never the reverse.

Reference officer: [cryptoAGI/savante](https://github.com/cryptoAGI/savante).
First verdict on record: 2026-07-26, parsec-wallet production readiness →
APPROVE_WITH_CONDITIONS.

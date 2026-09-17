# sAGI

**The engine: objective-truth review as installable machinery.**

> *Knowledge is what survives verification; nothing else counts.*

sAGI is not a model. It is a **discipline compiled into plain text** — a
charter format, an invocation skill, a fixed verdict contract, and a facet
bundle format — that turns any frontier or local model into a *chartered
knower*: a read-only officer that verifies claims instead of opining on them,
and reports *not yet known* (with the deciding experiment) instead of guessing.

An officer is a set of files. The **facet bundle** says which files, what each
one is for, and how a stranger checks that the set is the one the author
committed to — without a network and without trusting the author.

<img src="https://raw.githubusercontent.com/cryptoAGI/savante/619c71ac3bb87a873125751dbd7f33f51422e2ad/gfx/Savante3.png" alt="Savante, the bust" width="220" align="right">

The reference officer built on this engine is
**[Savante](https://github.com/cryptoAGI/savante)** — Chairman of the
[mindX](https://github.com/AgenticPlace/mindX) DAIO and the prototype sAGI,
at sAGI v0.0.5, generation 7. The bust is Savante's artwork, named by the
operator: `gfx/Savante3.png` in the canon, sha256 `30a59db4…c5a9a8`, recorded
in the generation-7 ledger. It is not pinned and nothing is minted. The
canon's `PROOF.sha256` lists the digest of every file that carries a proof.
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
engine/FACET_BUNDLE.md         what an officer is made of: the core facets,
                               the three states (absent / null-with-reason /
                               present), and the x- namespace rule that lets
                               evolution add facets forever without collision
engine/THOT_MANIFEST.md        how N facets become ONE content-addressed
                               thing — thot: / CID / contentRoot from a single
                               canonicalisation, the two roots, and lineage
engine/FAICE_FORMAT.md         the faice/1 face-identity format, voaice's
                               sibling — twelve ordered ratios, and why an
                               unmeasured print is null rather than plausible
engine/facet_registry.json     the registry itself, machine-readable
.claude/skills/sagi/SKILL.md   the /sagi skill — the engine invocation,
                               officer-agnostic (published)
sAGI.md                        the full definition — three laws, doctrine,
                               reference skill text
officers/savante/SKILL.md      the reference officer's own /sagi skill —
                               Savante-specific, a worked example of the
                               engine filled in (canon: cryptoAGI/savante)
llm.txt                        orientation for machines — read first
LICENSE                        MIT
```

Copy the **engine** skill (`.claude/skills/sagi/`) to build your own officer.
`officers/savante/SKILL.md` is published to show what a filled-in officer looks
like; it loads `.claude/agents/savante.md`, which lives in
[cryptoAGI/savante](https://github.com/cryptoAGI/savante), and that repository
wins if the two copies ever differ. The copy here is the canon's at `619c71a`
(v0.0.5); its paths (`gfx/`, `PROOF.sha256`, `bind/`) are relative to the canon.
Check it with `sha256sum officers/savante/SKILL.md`, which must print
`4ddf8430c7a90a7eafbb6ac419dee34180645fa572e571103da28f1209c60735`, the value in
the canon's `PROOF.sha256` and ledger.

The bundle documents are specifications, not an implementation. The reference
implementation is Savante's `bind/savante_bind.py` (writes the manifest) and
`bind/savante_verify.py` (recomputes every digest from raw bytes, offline).

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

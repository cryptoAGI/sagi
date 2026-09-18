# The sAGI facet bundle

*An officer is a set of files. This document says which files, what each is for, and how a stranger
checks that the set is the one the author committed to. It is the third engine document, beside
`CHARTER_TEMPLATE.md` (what an officer is told) and `VERDICT_CONTRACT.md` (what an officer returns).*

The reference implementation is `bind/savante_bind.py` and `bind/savante_verify.py` in
[savante](https://github.com/cryptoAGI/savante); the first bundle is `sAGI` (Savante). This spec is
generic: nothing here names an officer.

As of 2026-09-14 that reference implementation (savante main, 1fcca89) predates the closed
algorithms refusal in `THOT_MANIFEST.md` §3a. Its verifier does not refuse a missing or incomplete
`algorithms` block, and `savante.thot.json` lacks `algorithms.doctrine_root`, so §3a rule 1(b) refuses
that manifest until savante re-binds. Where the implementation and this spec disagree, the spec wins.

References of the form `mindX/…`, `facets.py` and `persona_project.py` name files in mindX
(AgenticPlace/mindX, a private repository). They are **mindX-internal (private), not required to verify**: cited as
provenance, and not required to validate or verify a bundle.

---

## 1. The shape

A bundle is a set of sibling files sharing one **stem** — the officer's name — distinguished by
extension:

```
<stem>.persona   <stem>.model   <stem>.prompt   <stem>.agent
<stem>.tool      <stem>.skill   <stem>.voaice   <stem>.faice
```

The stem is the key. mindX keys its blockchain-agent class the same way
(`mindX/agents/blockchain/facets.py:32-33`, mindX-internal (private), not required to verify), so a bundle installs by copy and rename, and one officer never
collides with another.

A bundle is named and verified by its **THOT manifest** — see `THOT_MANIFEST.md`. The manifest is the
only place the set is enumerated; a file that is not in the manifest is not in the bundle.

## 2. Three states, never two

Every facet, and every measured field inside a facet, is in exactly one state:

| State | Meaning | How it appears |
|---|---|---|
| `present` | authored or derived, with content | the file exists and validates |
| `null_with_reason` | the file exists; a value in it is not yet known | the field is `null` **and** a sibling `*_reason` or `*_note` says why, naming the deciding experiment where one exists |
| `absent` | the facet does not exist | listed in the manifest's `absent[]` with a reason; never a stub |

**A plausible-looking value in place of an unmeasured one is a defect, not a placeholder.** This is
not a new rule; it is the house rule stated once for every facet. [`voaice/FORMAT.md`](https://github.com/cryptoAGI/voaice/blob/main/FORMAT.md) puts it as
*"Nulls mean nothing was measured"*, and an unmeasured voice ships `measured: null, vprint: null`
rather than a synthesised number.

The rule has teeth because an identity facet is exactly where a false value is most tempting and least
detectable: a faceprint or a voiceprint is a hash of *a measurement*, so a fabricated one is
indistinguishable from a real one until someone tries to reproduce it.

## 3. The core facets

Reserved extensions. This list is owned by the spec version at the top of `facet_registry.json`; a
fork may add facets (§4) but may never redefine one of these.

| Facet | Required | Format | What it is |
|---|---|---|---|
| `.persona` | **yes** | JSON, mindX `.persona v1` | identity: beliefs, desires, intentions, skills, safety, embodiment. The source of truth for everything derivable. Validator: `persona_project.py --check` (mindX-internal (private), not required to verify) |
| `.agent` | **yes** | plain text, CAPS spec | the class facet: implementation, domain, capabilities, knowledge domains. Renderer: `facets.py:88` (mindX-internal (private), not required to verify) |
| `.model` | **yes** | YAML | the inference *policy* — logical model, task class, and whether it is pinned. A policy, not weights |
| `.prompt` | no | text + YAML frontmatter | the system prompt. **Derived** where a charter exists (§5) |
| `.tool` | no | JSON | the capability surface: tool allowlist, forbidden set, grant mask, and for each row **what enforces it** (§6) |
| `.skill` | no | Markdown | the invocation surface — how a caller reaches the officer |
| `.voaice` | no | JSON, `voaice/1` | voice identity. Spec: [`voaice/FORMAT.md`](https://github.com/cryptoAGI/voaice/blob/main/FORMAT.md) |
| `.faice` | no | JSON, `faice/1` | face identity. Spec: `FAICE_FORMAT.md`, beside this file |
| `.attribute` | no | Markdown + YAML frontmatter | the traits a persona is built FROM — each a claim with a **value, a source, and a state** |
| `.verse.xml` | no | XML | the verse; its lines become the persona's `voice_examples` |
| `.aiml` | no | XML | model-free rendering — categories from `exchanges` + `task.battery` |
| `.reputation` | no | JSON | standing, earned rather than asserted (same source) |
| `.agency` | no | Markdown + YAML frontmatter | **permission, not ability** — what this actor may decide alone, what it may do with a witness, and what it must defer |

### `.agency` — why permission is a separate facet

A `.persona` says what an actor *is*; a `.tool` says what it *can reach*. Neither answers the question
that decides whether it may act unsupervised. `.agency` is that answer, and it is deliberately not
derivable from the other two: **capability and permission are different facts, and conflating them is
how an assistant becomes an actor nobody authorised.**

Five required sections — `may_decide`, `may_do_with_a_witness`, `must_defer`, `bounds`, `state` —
after a free-form preamble. Two rules make it safe to read quickly:

- **Deny by default.** Absence means not permitted, so a missing line is a refusal, never an oversight.
  Revocation is deleting a line.
- **Granted from outside.** An actor asking to widen its own agency is a request, not a change.

It joined the core set after the fact: two well-formed instances (`jaimla.agency`, `luvai.agency`)
already used the bare extension, which §4's namespace rule would have made a **hard error**. Renaming
correct files to protect a list would have been the wrong repair — the list was incomplete, not the
files.

Two enumerations existed before this spec and disagreed (both mindX-internal (private), not required to verify): six facets in
`mindX/agents/blockchain/facets.py:29` (`agent, model, persona, walletpublickey, bankon, iNFT`) and
seven in `mindX/faicey/FAICE.md` (`model, agent, prompt, persona, skill, attribute, reputation`). The
core set above is their **union**, minus three that are not authored at all:

**Mint outputs are not core facets.** `walletpublickey`, `bankon` and `iNFT` are written by a mint
pipeline as on-chain data lands (`facets.py:14-16`), never by an author. They are registered as
`x-mindx.walletpublickey`, `x-mindx.bankon`, `x-mindx.inft` and carry `written_by: "pipeline"`. Their
absence from a bundle means nothing is minted; their presence is evidence, not authorship.

## 4. Custom facets — how evolution adds a facet forever

Anything outside the core list is a custom facet and **must** be namespaced:

```
^x-[a-z0-9]+\.[a-z0-9_]+$        e.g.  x-mindx.bankon   x-savante.verdictlog
```

and **must** be declared in the manifest with `{owner, spec_url, media, added_in}` — `added_in` being
the generation at which it entered the bundle.

**An unnamespaced unknown extension is a hard error, not a warning.** That single rule is what lets
the core list grow later without ever colliding with somebody's private facet: the namespace is
reserved to the spec, everything else is reserved to its owner. A warning would not do — the failure
it prevents is silent, arriving years later when a core facet takes a name a fork already used.

Declaration is also what makes evolution auditable. A facet that appears without a manifest entry is
an undeclared change to the officer, which is exactly the thing the bundle exists to make visible.

## 5. Derivation and the authority ladder

Some facets are **derived**: generated from another file, byte-checked on every bind, never
hand-edited. Where a charter exists (a Claude Code subagent definition, per `CHARTER_TEMPLATE.md`),
the ladder is:

```
charter body          BINDING — the harness enforces its frontmatter; it wins
  └─ .prompt          DERIVED — byte-identical to the charter body; drift fails the bind, closed
       └─ persona /system_prompt   a COMPRESSED restatement — deliberately shorter, NOT byte-linked
```

The third rung is the one to state out loud: the persona's `system_prompt` is a précis used for
imprint training, and it is *supposed* to differ in length from the charter. Only the first two are
byte-linked. A reader who assumes all three agree will chase a difference that is intentional.

A derived facet carries its provenance in its own frontmatter — `derived_from`, `derivation`,
`authority` — so the file says what it is without needing this document.

### `.prompt` runs in **either** direction, and the bundle must say which

The ladder above is one legitimate arrangement, not the only one. Both of these are correct:

| authority | who wins on disagreement | first instance |
|---|---|---|
| **derived** | the charter. The prompt is regenerated; drift is a hard error. | `sAGI.prompt` |
| **source** | the prompt. The persona is stale and is regenerated from it. | `luvai.prompt` |

**Guessing inverts the repair** — regenerating a source prompt from a persona destroys the original,
and hand-editing a derived prompt creates a second source of truth. So a bundle *must* declare the
direction in the file's own frontmatter, and this spec deliberately does not fix it.

This registry originally hardcoded `derived_from: charter`, which would have declared the perfectly
correct `luvai.prompt` to be wrong. Corrected once a second instance existed — the same repair as
`.agency`: the list was incomplete, not the file.

### `.prompt` + `.attribute` → `.persona`

Where a bundle uses source-authority prompts, the persona is **produced**, not written: `.prompt`
supplies the body, `.attribute` supplies the traits, and `.persona` is generated from both. Authority
runs one way only — *the prompt never gets edited to match the persona.* Documented at
`mindX personas/README.md` §"which way authority runs" (mindX-internal (private), not required to verify); first instances
`luvai.prompt` and `luvai.attribute`, published in [cryptoAGI/luvai](https://github.com/cryptoAGI/luvai).

## 6. `.tool` declares what enforces it

A tool facet lists capability, and for every row an `enforced_by` whose only legal values are:

| Value | Meaning |
|---|---|
| `harness` | the runtime refuses the call — a real control |
| `nothing` | published expectation only; no code checks it |
| `executor_that_does_not_exist` | a control is specified, and the thing that would apply it has not been built |

A permission bitmap stored on chain but read by nobody is `nothing`. Writing that down is the point:
a capability surface that looks like a control, but is not, is worse than no surface at all, because
readers grant it trust it has not earned. The facet must carry the admission itself — not leave it to
an audit document a holder may never read.

## 7. Validation

A bundle validates when:

1. every required facet is `present`;
2. every facet validates against the format its registry row names;
3. every unknown extension is namespaced and declared (§4);
4. every derived facet reproduces byte-for-byte from its source (§5);
5. every `null` field has a reason beside it (§2);
6. the manifest's digests match the files, and the manifest's own identity matches its bytes
   (`THOT_MANIFEST.md`).

Checks 1–6 need no network and no trust in the author: every one is a recomputation from raw bytes.
That property is load-bearing — a bundle whose verification requires trusting a server has not
verified anything.

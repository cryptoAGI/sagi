# The THOT manifest

*How N facet files become one content-addressed thing that can be named, verified, and — if its owner
decides — bound to a token. Companion to `FACET_BUNDLE.md`, which says what the facets are.*

---

## 1. Why a manifest at all

A bundle of eight files has eight digests and no name. A holder asking *"is this the officer the
author committed to?"* cannot answer it from eight separate hashes; they need one value that changes
if any byte anywhere changes, and that they can recompute themselves.

The manifest is that value's preimage. It is a small canonical JSON document listing every facet with
its size and digest, plus the roots and the lineage. **Its own bytes are the bundle's identity.**

## 2. Two THOT vocabularies, one document

Two definitions of "THOT" existed before this spec and appeared to conflict:

| Source | Names | Identity |
|---|---|---|
| `mindX/mindx/godel/mindxtrain/inft_publish.py:39-47` | a *document* — one training generation | `thot-<cidv1>` |
| `mindX/docs/blockchain/THOTpaper.md:56-62`, `permanence/lib/ipld.js:98` | *bytes* — anything storable, on a rung ladder | `thot:<sha256>` plus a CID |

The conflict dissolves once you notice a manifest is a document **that is also bytes**. So both names
are emitted, from one canonicalisation, and they cannot drift because each is a function of the same
input:

```
canonical  = json.dumps(manifest_without_identity, sort_keys=True,
                        separators=(",", ":"), ensure_ascii=False).encode("utf-8")

thot       = "thot:" + sha256(canonical).hex()      # permanence identity
cid        = cid_v1_raw(canonical)                  # CIDv1, raw leaf, sha2-256, base32
name       = "thot-" + cid                          # inft_publish convention
contentRoot= keccak256(canonical)                   # the on-chain content root
```

### Two hashes, and why they differ

A manifest has **two** identities, and confusing them is the easiest mistake to make here:

| | Over what | Answers | Where it lives |
|---|---|---|---|
| `manifest.identity` | the **canonical form, without the `identity` block** | *which bundle is this?* | inside the manifest |
| the **file** identity | the **file as written** — pretty-printed, `identity` block included | *which stored object is this?* | the ledger, never the manifest |

They cannot be equal, and neither is wrong. The first excludes the identity block because a document
cannot contain a digest of itself; the second is what a storage tool actually hashes when you hand it
the file — `permanence/dapp.mjs identify <manifest>` reports exactly this one, and it is the name an
Arweave upload will carry.

So the file identity is recorded **in the ledger**, beside the manifest's own — it is the one value
about the manifest that cannot live inside it. A verifier checks the first by rebuilding the canonical
form, and the second by hashing the bytes on disk. Expect two different `thot:` values and be
suspicious of any tool that reports only one.

`cid_v1_raw` equals what `ipfs add --cid-version 1 --raw-leaves` gives a blob **up to 256 KiB**
(`inft_publish.py:40-43`). A manifest is a few kilobytes, so the name here is the name IPFS would give
it. Larger artifacts are referenced by sha256 and never by a locally guessed CID, because above that
bound the CID depends on chunking.

**No salt, no timestamp, no wall clock, anywhere in the canonical form.** A content root that is not
reproducible from the repository alone is not a content root; it is a random number that happens to
have been recorded. (The pre-existing mint pipeline derived its root from `os.urandom` — that is the
defect this rule exists to prevent.)

## 3. Shape

```jsonc
{
  "$comment": "DERIVED OUTPUT — regenerable, never hand-edited. Contains no digest of itself.",
  "schema": "sagi.thot_manifest/1",
  "bundle":  { "id": "<stem>", "officer": "<agent id>", "generation": 1, "parent": null },

  "facets":  [ { "facet": "persona", "path": "<stem>.persona", "bytes": 49644,
                 "sha256": "…", "cid": "bafkrei…", "state": "present",
                 "custom": false, "added_in": 1 } ],
  "absent":  [ { "facet": "attribute", "state": "absent", "reason": "not authored" } ],
  "custom":  [ { "facet": "x-savante.verdictlog", "owner": "savante",
                 "spec_url": "…", "media": "application/json", "added_in": 3 } ],

  "doctrine_root": "0x…",                     // the officer's immutable clauses, if it has one
  "bundle_root":   { "value": "0x…", "hash": "keccak256",
                     "construction": "concat over facets in registry order of ext+0x1f+sha256+0x1e" },
  "merkle":        { "leaves": 64, "leaf_rule": "keccak256(ext||0x1f||sha256)",
                     "padding": "keccak256(\"\")", "root": "0x…",
                     "ternary_head": "<persona leaf>", "ternary_head_index": 0 },

  "identity":      { "thot": "thot:…", "cid": "bafkrei…", "name": "thot-…", "contentRoot": "0x…" },
  "rung":          { "value": "referenced",
                     "evidence": { "locator": "<repo>@<commit>", "commitTx": null,
                                   "dataTx": null, "attestation": null } },
  "license": "MIT"
}
```

`identity` is computed over the document **without** the `identity` block — a document cannot contain
a digest of itself.

## 3a. Hash agility — the algorithms are data

A manifest is **immutable once anchored**. So the one thing it must never do is assume its own hash
functions will outlive it.

Every manifest carries an `algorithms` block inside the canonical bytes:

```jsonc
"algorithms": {
  "facet_digest": "sha256",
  "cid": "cidv1-raw-sha2-256-base32",
  "bundle_root": "keccak256",
  "merkle_leaf": "keccak256",
  "identity_thot": "sha256",
  "identity_content_root": "keccak256",
  "canonicalisation": "json.dumps(sort_keys=True, separators=(',',':'), ensure_ascii=False)"
}
```

**Why it is in-band.** Without it, every digest in every manifest ever written dies on the day sha256
or keccak256 falls, and the format has no way to name a successor — the documents would be
self-describing about everything *except* the one property their integrity rests on. Declaring the
algorithms costs a few hundred bytes today and cannot be retrofitted to an anchored manifest
tomorrow.

**Two rules follow, and both are refusals.**

1. **A verifier MUST refuse a manifest whose declared algorithms it does not implement**, rather than
   verify with the functions it happens to have. A digest checked with the wrong function is not a
   check; it is a green light with no evidence behind it.
2. **A manifest with no `algorithms` block is a finding, not a pass.** Its digests can only be checked
   under an assumption, and the report must say so.

### Migration, when an algorithm falls

Do **not** recompute an anchored manifest. It is immutable, and rewriting it would destroy the record
of what was actually committed to.

Instead, publish the **next generation**:

- `algorithms` names the successor function;
- `lineage.parent` carries the superseded manifest's CID;
- the old digests travel forward as **historical evidence** — what was committed to, under the
  algorithm believed sound at the time — and are never silently replaced.

The chain then reads honestly across the break: *these bytes were committed under sha256 in 2026, and
re-anchored under its successor in year N.* That is a stronger claim than a manifest which quietly
appears to have always used the new function.

## 4. Two roots, deliberately separate

| Root | Answers | Moves when |
|---|---|---|
| `doctrine_root` | *is the officer still the officer?* | one of its fixed clauses is edited |
| `bundle_root` | *is this the same bundle?* | any facet byte changes, or a facet is added |

Do not conflate them. If adding a facet moved the doctrine root, ordinary maintenance would ring the
tamper alarm, and **an alarm that rings on maintenance is one holders learn to ignore.** The doctrine
root is defined by the officer's own spec (a fixed, ordered list of JSON pointers); the bundle root is
defined here. They travel together in the manifest and never merge.

## 5. The Merkle tree, and why it is a real one

Where a bundle is bound to a commitment registry that expects a Merkle root over a fixed leaf count —
mindX's `THOTCommitmentRegistry.issueTHOT4096(root, ternaryHead, ternaryHeadIndex, cid, metadataURI)`
documents `root` as *"Merkle root computed off-chain from 64 leaves"* — build a genuine 64-leaf tree:

- **leaf** `i` = `keccak256(ext || 0x1f || sha256_of_facet_bytes)`, facets in registry order;
- **padding** to 64 leaves = `keccak256("")`, a documented constant, not a repeat of the last leaf
  (repetition invites second-preimage confusion);
- **ternaryHead** = the persona leaf, index 0 — the facet that most defines the officer.

Passing a flat digest into a slot the contract calls a Merkle root would be a semantic lie: it would
verify, and it would mean something other than what the contract says it means. If the registry is not
used, the `merkle` block is still emitted — it costs nothing and it is the same bytes either way.

## 6. Rung — derived, never asserted

The rungs are `referenced → committed → stored → attested`, and the value is **derived from evidence**
(`permanence/lib/rungs.js rungOf(evidence)`), never set by hand:

| Rung | Earned by |
|---|---|
| `referenced` | a locator exists — the manifest names where the bytes live |
| `committed` | a commitment transaction exists |
| `stored` | the bytes are actually stored, with a data transaction id |
| `attested` | a third party attests to that storage |

At authoring time the evidence block is empty apart from a locator, so the honest rung is
`referenced`. Writing `stored` before an upload receipt exists is the exact failure this ladder was
built to prevent — and it is checkable by anyone, which is the point.

## 7. Lineage — how a bundle evolves

Evolution adds facets; it never rewrites the genesis record.

- Each change to any facet — including adding a custom one — produces a **new generation**:
  `generation` increments and `parent` is set to the previous manifest's CID.
- The chain is walkable: generation *n* names *n−1*, back to `parent: null` at genesis.
- Where a bundle is bound to a token, the **genesis manifest stays attached forever** (an attach
  operation is typically one-shot per token, and a content root one-shot per contract). Later
  generations are reached through mutable metadata — a token URI or a registry's agent URI — never by
  altering what was attached.

That split is the design: the immutable record says *what was committed to*, the mutable pointer says
*where the current generation is*, and the parent chain proves the second descends from the first.

## 8. Verifying a manifest

Offline, trusting nothing the author wrote:

1. hash every listed facet from raw bytes; compare size, sha256 and CID;
2. rebuild the canonical form, byte-for-byte;
3. recompute `thot`, `cid`, `name` and `contentRoot`; compare all four;
4. recompute `bundle_root` and the Merkle root;
5. check every derived facet reproduces from its source;
6. check every `absent` entry has a reason and every custom facet is declared.

Then, and only then, anything on chain: compare the recorded root against the **first** root written
at registration, not merely the current one — an owner who edits, re-binds and publishes a fresh
internally-consistent root is caught by the first write and by nothing else.

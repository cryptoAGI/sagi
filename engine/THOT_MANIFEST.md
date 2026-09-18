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

The `mindX/…` paths in this document are in AgenticPlace/mindX, a private repository:
**mindX-internal (private), not required to verify**. The `permanence/…` paths are in a local tree
that is not published. Both are provenance only: verifying a manifest needs neither.

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
(`mindX/…/inft_publish.py:40-43`, mindX-internal (private), not required to verify). A manifest is a few kilobytes, so the name here is the name IPFS would give
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
  "algorithms": { … §3a … },
  "bundle":  { "id": "<stem>", "officer": "<agent id>", "generation": 1, "parent": null,
               "parent_reason": "genesis: no earlier manifest of this bundle" },
  "relations": [ … optional, §7 — a relation, never a parent … ],

  "facets":  [ { "facet": "persona", "path": "<stem>.persona", "bytes": 49644,
                 "sha256": "…", "cid": "bafkrei…", "state": "present",
                 "custom": false, "added_in": 1 } ],
  "absent":  [ { "facet": "attribute", "state": "absent", "reason": "not authored" } ],
  "custom":  [ { "facet": "x-savante.verdictlog", "owner": "savante",
                 "spec_url": "…", "media": "application/json", "added_in": 3 } ],

  "doctrine_root": "0x…",                     // the officer's immutable clauses, if it has one
  "bundle_root":   { "value": "0x…", "hash": "keccak256", "preimage_bytes": 571,
                     "order": [ "persona", "agent", … ],
                     "construction": "concat over present facets — core in registry order, then custom by name — of (facet_utf8 + 0x1f + sha256_hex_ascii + 0x1e)" },
  "merkle":        { "leaves": 64, "leaf_rule": "keccak256(facet_utf8 || 0x1f || sha256_hex_ascii), in bundle_root order",
                     "padding": "keccak256(b'')", "populated": 8, "root": "0x…",
                     "ternary_head": "persona", "ternary_head_index": 0 },

  "identity":      { "thot": "thot:…", "cid": "bafkrei…", "name": "thot-…", "contentRoot": "0x…" },
  "rung":          { "value": "referenced",
                     "evidence": { "locator": "<host>/<owner>/<repo>@<full commit id>",
                                   "locator_holds": [ "persona", "agent", … ],
                                   "locator_lacks": [ … ],
                                   "commitTx": null, "dataTx": null, "attestation": null } },
  "license": "MIT"
}
```

`identity` is computed over the document **without** the `identity` block — a document cannot contain
a digest of itself.

The `construction`, `leaf_rule`, `padding`, `why` and `note` strings are human-readable descriptions.
They are inside the canonical bytes, but a verifier recomputes every value from §2, §3a and §5 and
does not compare these strings to anything.

**The structural fields beside them are compared** (§8 step 4). Each is compared by exact JSON value.
"Integer" means a JSON number with an integer value, never a boolean: `true` is not `1`, and `false` is
not `0`.

| # | Field | Type | MUST equal |
|---|---|---|---|
| S1 | `bundle_root.value` | string | `"0x"` + 64 lowercase hex characters, the §5 `bundle_root` recomputed |
| S2 | `bundle_root.hash` | string | `algorithms.bundle_root` |
| S3 | `bundle_root.order` | array of strings | the §5 labels of the present facets, in §5 order |
| S4 | `bundle_root.preimage_bytes` | integer | the byte length of the §5 `bundle_root` preimage |
| S5 | `merkle.root` | string | `"0x"` + 64 lowercase hex characters, the §5 root recomputed |
| S6 | `merkle.leaves` | integer | `64` |
| S7 | `merkle.populated` | integer | `n`, the number of present facets |
| S8 | `merkle.ternary_head` | string | `"persona"` |
| S9 | `merkle.ternary_head_index` | integer | `0` |
| S10 | `rung.evidence.locator` | string | the form `<host>/<owner>/<repo>@<commit>`, where `<commit>` is 40 lowercase hex characters (§6) |
| S11 | `rung.evidence.locator_holds` | array of strings | a list of distinct labels, each naming a present facet, in `bundle_root.order` order (§6). An empty array is allowed. |

A verifier SHOULD report any other key in `bundle_root` or `merkle` as a finding. Where a facet entry
carries `at_locator`, the value MUST be `true` exactly when that facet is in `locator_holds`.
`rung.evidence.locator_lacks` and `referenced_elsewhere`, where present, are descriptive. S11 checks
the list's shape. Whether the listed bytes really are at the locator commit is the separate §6 check,
which needs a local clone.

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
  "merkle_pad": "keccak256",
  "doctrine_root": "keccak256",          // conditional: see the table
  "identity_thot": "sha256",
  "identity_content_root": "keccak256",
  "git_blob": "sha1 over b'blob <len>\\x00' + bytes (git's own object id; used only to name the referenced voice)",  // conditional: see the table
  "canonicalisation": "json.dumps(sort_keys=True, separators=(',',':'), ensure_ascii=False).encode('utf-8')",
  "note": "free text"
}
```

### The vocabulary — one, closed

For `schema: "sagi.thot_manifest/1"` these are the only keys, and each value is compared as an
**exact string** (after JSON decoding):

| Key | The one value v1 knows | Declared | What it names |
|---|---|---|---|
| `facet_digest` | `sha256` | always | every `facets[].sha256`, and the digest inside every §5 record |
| `cid` | `cidv1-raw-sha2-256-base32` | always | every `cid`, per facet and in `identity` |
| `bundle_root` | `keccak256` | always | `bundle_root.value` (§5) |
| `merkle_leaf` | `keccak256` | always | every Merkle leaf **and every inner node** (§5) |
| `merkle_pad` | `keccak256` | always | the function whose empty-input digest is the pad: pad = `keccak256(b"")` (§5) |
| `identity_thot` | `sha256` | always | `identity.thot` |
| `identity_content_root` | `keccak256` | always | `identity.contentRoot` |
| `canonicalisation` | `json.dumps(sort_keys=True, separators=(',',':'), ensure_ascii=False).encode('utf-8')` | always | the bytes every identity is computed over (§2) |
| `doctrine_root` | `keccak256` | **when the manifest carries a non-null top-level `doctrine_root`** | the doctrine root over the officer's pointer preimage (§4) |
| `git_blob` | `sha1 over b'blob <len>\x00' + bytes (git's own object id; used only to name the referenced voice)` | **when a `facets[]` or `custom[]` entry carries a git blob id field** (mechanical test below) | that blob id |
| `note` | any string | optional | free text. **Not an algorithm**; the rules below ignore it |

The `git_blob` value is written in the table as its decoded string; in JSON the backslash is escaped
(`\\x00`), as in the block above. The parenthesis is part of the value.

**The conditions are mechanical.**

- `doctrine_root` is required iff the manifest's top-level `doctrine_root` is present and not `null`.
- `git_blob` is required iff **G** holds:

  > **G.** Take each element of the top-level array `facets` and each element of the top-level array
  > `custom` (a missing array counts as empty). Walk every JSON object nested inside those elements,
  > the element itself included, through objects and arrays at any depth. G holds iff some object key
  > begins with the eight ASCII characters `git_blob`, compared case-sensitively (a prefix test,
  > `key.startswith("git_blob")`, never a substring test).

  Only object **keys** are tested, never string values. The key's value does not matter, not even
  `null`. Nothing outside those two arrays is walked: not `absent[]`, not `relations[]`, not `rung`, and
  not `algorithms`. So the `algorithms.git_blob` declaration cannot trigger itself.
- **Git commit ids never trigger it**, and neither does a 40-hex value alone. A verifier MUST NOT widen
  G by matching other key names (`git_object`, `oid`, `sha1`, `commit`), by matching 40-hex values,
  or by walking other top-level keys. A wider test would refuse, under rule 1(b), manifests that this
  spec accepts.
- A conditional key MAY be declared when its condition does not hold. Rule 1(d) still applies: its
  value must be the v1 value.

Examples of G:

| # | JSON location | G | Why |
|---|---|---|---|
| +1 | `facets[5].reference.git_blob_sha1` (jaimla @ 8b57ccf: the voaice held by reference) | holds | key begins `git_blob`, two levels inside a `facets[]` element |
| +2 | `facets[0].git_blob` | holds | the element's own key |
| +3 | `custom[0].source.git_blob_id` | holds | inside a `custom[]` element |
| +4 | `facets[2].renderings[1].git_blob_sha1` | holds | through an array, at any depth |
| −1 | `facets[5].reference.commit` = `"3f7412…6c76"` | does not | a commit id; the key does not begin `git_blob` |
| −2 | `rung.evidence.locator` = `"github.com/cryptoAGI/savante@368c73…24d4"` | does not | outside `facets[]`/`custom[]`, and a commit |
| −3 | `relations[0].manifest_commit` = `"8b57ccf"` (luvai @ 0c1eef7) | does not | outside `facets[]`/`custom[]` |
| −4 | `facets[1].reference.git_object` or `facets[1].oid` = 40 hex | does not | the key does not begin `git_blob` |
| −5 | `absent[0].git_blob_sha1` or a top-level `references[0].git_blob_sha1` | does not | not walked |
| −6 | `algorithms.git_blob` | does not | not walked |
| −7 | `facets[0].note` = `"git_blob_sha1 bfcc5e…"` | does not | string values are never tested |

Against the published manifests, G holds for jaimla @ 8b57ccf, which declares `git_blob`. It does not
hold for savante @ 1fcca89 or luvai @ 0c1eef7, and neither declares it.

**What is refused before the vocabulary is read.** These checks are part of §8 step 0, in this order:

- **Duplicate keys.** The manifest file MUST parse as exactly one JSON value. Any object at any depth
  that contains the same key twice (after JSON string unescaping) is refused. A parser that keeps the
  last or first duplicate silently MUST NOT be used without a duplicate check (in Python, an
  `object_pairs_hook` that refuses repeats). Two readers of one file must never see two documents.
- **Schema id.** The top-level `schema` MUST be the JSON string `"sagi.thot_manifest/1"`, exactly. A
  missing, non-string, or different value is refused. This vocabulary is defined for no other
  schema.
- **`note`.** If `algorithms.note` is present, it MUST be a JSON string. Any other type, including
  `null`, is refused. Its content is never compared.

**Why it is in-band.** Without it, every digest in every manifest ever written dies on the day sha256
or keccak256 falls, and the format has no way to name a successor — the documents would be
self-describing about everything *except* the one property their integrity rests on. Declaring the
algorithms costs a few hundred bytes today and cannot be retrofitted to an anchored manifest
tomorrow.

**The rules are refusals.** To refuse is to render no APPROVE and to exit non-zero.

1. **A verifier MUST refuse a manifest whose `algorithms` block**
   - (a) is missing, or is not a JSON object;
   - (b) lacks a key declared *always*, or lacks a conditional key whose condition holds;
   - (c) carries any key other than the ten algorithm keys above and `note`; or
   - (d) declares, for any key, a value other than the one this verifier implements; or
   - (e) belongs to a manifest that fails one of the pre-checks above (a duplicate key, a schema id
     other than `sagi.thot_manifest/1`, or a non-string `note`).

   It refuses rather than verify with the functions it happens to have. A digest checked with the
   wrong function is not a check; it is a green light with no evidence behind it. An absent key is an
   assumption, exactly as an absent block is. (An earlier text of this section called a missing block
   "a finding, not a pass"; it is now a refusal.)
2. **The algorithms are checked first**, before any digest is recomputed.
3. **Identity-only scope.** A tool that recomputes only a *related* manifest's identity (§7
   `relations[]`) needs only `cid`, `identity_thot`, `identity_content_root` and `canonicalisation` to
   be values it implements. It MUST refuse on those four, and MUST report every other declared key as
   not verified rather than as checked.
4. **A successor is a new value, or a new key, in a new schema version** — never an edit to this
   table. Until a verifier implements it, rule 1 refuses it, and that refusal is the design.

### Migration, when an algorithm falls

Do **not** recompute an anchored manifest. It is immutable, and rewriting it would destroy the record
of what was actually committed to.

Instead, publish the **next generation**. This applies whether or not the manifest is anchored: a
successor is always a new generation (§7 rule G3), even when no facet byte changed.

- `schema` names the new schema version, and `algorithms` names the successor function;
- `bundle.generation` increments and `bundle.parent` carries the superseded manifest's `identity.cid`
  (§7 — a persona's `lineage` is a separate relation and is never a THOT parent);
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
root is defined by the officer's own spec (a fixed, ordered list of JSON pointers), and its hash
function is declared as `algorithms.doctrine_root` (§3a); the bundle root is defined here. They
travel together in the manifest and never merge.

## 5. The Merkle tree, and why it is a real one

Where a bundle is bound to a commitment registry that expects a Merkle root over a fixed leaf count —
mindX's `THOTCommitmentRegistry.issueTHOT4096(root, ternaryHead, ternaryHeadIndex, cid, metadataURI)`
(mindX-internal (private), not required to verify)
documents `root` as *"Merkle root computed off-chain from 64 leaves"* — build a genuine 64-leaf tree.
`bundle_root` and the tree share one ordering and one record:

- **which facets** — the facets whose state is `present` (including a present facet held at a
  reference outside the repository). Absent facets take **no** slot.
- **order** — core facets in `facet_registry.json` `core` key order, then custom facets sorted by name
  in Unicode codepoint order. Slots are packed from index 0, so a facet's index differs between
  bundles (a bundle with no `skill` puts `voaice` one index earlier).
- **label** — the facet **name** as UTF-8: the core key (`persona`, `skill`, `verse.xml`) or the full
  custom name (`x-mindx.imprint`). For a core facet the name is the registry key, which is the
  `<stem>.<key>` extension the bundle convention uses. It is that key whatever the file is actually
  called: `skill` may live at `.claude/skills/sagi/SKILL.md`, and a custom facet may live at
  `<stem>.json`. The file path and its real suffix are never the label. A binder's `ext_utf8` means
  this key.
- **digest** — the facet's sha256 as **64 lowercase hex characters, ASCII-encoded**; not the 32 raw
  bytes.
- **record** `i` = `label || 0x1f || digest`.
- **bundle_root** = `keccak256(record_0 || 0x1e || record_1 || 0x1e || … || record_(n-1) || 0x1e)` —
  every record, the last included, followed by `0x1e`.
- **leaf** `i` = `keccak256(record_i)` for `i < n`. `n` MUST be at most 64: a binder with more
  present facets fails; it never truncates.
- **padding** — leaf `i` for `n ≤ i < 64` = `keccak256(b"")` =
  `0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`, a documented constant, not a
  repeat of the last leaf (repetition invites second-preimage confusion).
- **inner node** = `keccak256(left || right)`, the two children's 32 raw bytes in index order.
  Pairs are **not sorted** and **no domain-separation prefix** is added, to leaves or to nodes.
  Levels run 64 → 32 → 16 → 8 → 4 → 2 → 1; `merkle.root` is the last node as `0x` + 64 lowercase hex.
- **ternaryHead** = leaf 0, the persona — required, and first in core order, so always index 0: the
  facet that most defines the officer.

A tool built for another Merkle convention (sorted pairs, node or leaf prefixes, other padding)
computes a different root and MUST NOT be used to check this one. Changing any rule above is a new
`algorithms` value (§3a), never an edit.

**Test vectors.** Recomputed from each manifest's own `facets[].sha256` with only the rules above:

| Manifest | n | `bundle_root.value` | `merkle.root` |
|---|---|---|---|
| cryptoAGI/savante `savante.thot.json` @ 1fcca89 | 8 | `0x235da8e993dc8af2c077f50d698962446b872b17b1e5b033d5c5d976532b8880` | `0xdc1d80957cf831aee6638fd569e22cf0f6e5a1ec99ddde91cfecb5a15408fbe1` |
| cryptoAGI/jaimla `jaimla.thot.json` @ 8b57ccf (one custom facet; no `skill`) | 9 | `0x7f5bcc69dc9d50854189762dba0a97b3a48090ee546a0423aa9722917740e086` | `0xd7cb5363324645b4f6d0df19902fe9cd15dcb1c36ae925ee5ff2278315f6a258` |
| cryptoAGI/luvai `luvai.thot.json` @ 0c1eef7 (compound `verse.xml`) | 9 | `0x2feb7ecfcffe91c035951298e5d977d880ec475edb87fff5f3038efc8d44f94c` | `0xf594d1b2ab3f98b20ca203227e71a437ac86d82f6c4d4942227ecf8ee018309b` |

The savante row is a §5 root vector only. `savante.thot.json` @ 1fcca89 has a non-null top-level
`doctrine_root` but declares no `algorithms.doctrine_root`, so §3a rule 1(b) refuses that manifest
until savante re-binds. The jaimla and luvai manifests at the commits shown pass §3a.

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

### The locator is inside the identity

`rung.evidence.locator` is `<host>/<owner>/<repo>@<full commit id>`: the commit whose tree the binder
read, its git HEAD at bind time. It is in the canonical bytes, so **the manifest identity binds that
commit**. That is intended — a `referenced` rung is a claim about where the bytes are held, and a
claim is part of what was committed to. Three consequences:

- **Byte-idempotence means same inputs, including the same git HEAD, give byte-identical output.** It
  does not mean that the same facets give the same identity.
- The commit that records a bind (the ledger commit) necessarily follows the commit the manifest
  names, so a published manifest trails its repository's HEAD by that commit. That is expected, not
  drift. A verifier MUST NOT require the locator to equal HEAD. It checks the facet bytes against the
  manifest, and SHOULD check that every facet in `locator_holds` is byte-identical at the locator
  commit. A binder MUST NOT list a facet in `locator_holds` unless the facet's bytes at the locator
  commit hash to the manifest's `sha256` for it. Being present in that commit's tree is not enough:
  a facet edited but not committed is not held. Subtracting the facets that differ from HEAD, as
  jaimla's `components_differing_from_head` does, is the model.
- Re-binding at a later commit with no facet change and no successor moves `thot`, `cid`, `name` and
  `contentRoot`. `bundle.generation` and `bundle.parent` stay the same. That is allowed only while the
  manifest is unanchored (rung `referenced`), and it is §7 rule G4 exactly. A facet change is never
  such a re-bind: it is a new generation even while unanchored. An anchored manifest (rung
  `committed` or higher) is never re-bound, and any later change is a new generation (§7).

## 7. Lineage — how a bundle evolves

Evolution adds facets; it never rewrites the genesis record. Lineage is a **loop per bundle**, not a
chain across personas.

- **`bundle.parent` names the previous manifest of the same bundle, and nothing else.**
- The chain is walkable: generation *n* names *n−1*, back to `parent: null` at genesis. Every bundle is
  its own genesis.

### The generation rules

These are the rules a binder emits and a verifier checks. **P** is the superseded manifest: the most
recent manifest of this bundle (the same `bundle.id`) committed to the default branch of the bundle's
own repository. For generation 2 of the three published bundles, P is the genesis manifest at the
commits below. Its `identity.cid` is the value `bundle.parent` must carry:

| Bundle | P | P `identity.cid` |
|---|---|---|
| `sAGI` (cryptoAGI/savante) | `savante.thot.json` @ 1fcca89 | `bafkreieerglzkjrkmwrxhmrk53bf3tvhp3aut3qlatwpdtoutmspuxrh4i` |
| `jaimla` (cryptoAGI/jaimla) | `jaimla.thot.json` @ 8b57ccf | `bafkreidczapw72pldcf5pxyoz2hhdleauto7ajvjgun7w2iiq7ccwwhvku` |
| `luvai` (cryptoAGI/luvai) | `luvai.thot.json` @ 0c1eef7 | `bafkreiefvcfiw2rkrcjzblcj5ivrux7mtpsha4fqaulgvteycpr4zbwqtq` |

A **facet change** is any difference between P and the new manifest in the set of present facets, or
in any present facet's `sha256`: adding, removing or editing a facet, custom facets included.

- **G1: genesis.** `bundle.generation` is `1`, `bundle.parent` is `null`, and `bundle.parent_reason` is
  a non-empty string saying why there is no parent.
- **G2: a later generation.** For `bundle.generation` = *n* ≥ 2, the new manifest has
  `bundle.generation` = P's `bundle.generation` + 1, and `bundle.parent` = P's `identity.cid`, taken
  from the published P file and never recomputed from a local re-bind. `bundle.parent_reason` is
  **required** at every generation: a non-empty string saying what changed since P. Its content is
  descriptive and never compared. `bundle.parent` is never the manifest's own `identity.cid`, and it is
  never the CID of another bundle's manifest.
- **G3: what makes a new generation.** Every facet change is a new generation, anchored or not. Every
  change of `schema`, or of the value of any `algorithms` key, is also a new generation (an algorithm
  successor, §3a Migration), anchored or not. Under one schema version every algorithm value is fixed
  (§3a rule 1(d)), so in practice an algorithm successor always arrives with a `schema` change. When
  a facet change and a successor happen together, they make **one** new generation, not two. So the
  next manifest of any of the three bundles above that has a facet change is **generation 2**, with
  `bundle.parent` set to the P `identity.cid` in the table, whether or not it is anchored and however
  many facet edits it batches.
- **G4: what does not.** A re-bind with no facet change and no G3 change keeps `bundle.generation`
  and `bundle.parent`, and it moves the identity. That is allowed only while the rung is
  `referenced`. Such a re-bind may change the locator (§6), and it may change `relations[]`, absent
  reasons, derivation records or descriptive strings. It may also add or remove a conditional
  `algorithms` key carrying its v1 value (for example, adding `algorithms.doctrine_root`), or change
  `algorithms.note`. None of those changes `schema` or any declared algorithm value. An anchored
  manifest is never re-bound, so any change after anchoring is a new generation.
- **G5: never re-issue a genesis.** A facet change is never bound as generation 1. A binder MUST
  refuse to emit a facet change under an unchanged generation, rather than write a fresh genesis. A
  published manifest that did so before this rule (savante's generation-1 re-binds before 1fcca89)
  stays in the history as recorded. The next generation takes P as defined above.

For example, jaimla's next manifest after a facet change carries:

```jsonc
"bundle": { "id": "jaimla", "officer": "Jaimla", "generation": 2,
            "parent": "bafkreidczapw72pldcf5pxyoz2hhdleauto7ajvjgun7w2iiq7ccwwhvku",
            "parent_reason": "<what changed since generation 1>" }
```

**What a verifier checks** (§8 step 7), offline:

- V1. `bundle.generation` is an integer ≥ 1, never a boolean.
- V2. `generation` = 1 exactly when `bundle.parent` is `null`.
- V3. At generation ≥ 2, `bundle.parent` is a string matching `^bafkrei[a-z2-7]{52}$` (the §2 `cid`
  form, 59 characters), and it is not equal to `identity.cid`.
- V4. `bundle.parent_reason` is a non-empty string.
- V5. There is no top-level `lineage` key.
- V6. If the verifier is given P's bytes locally (a file path, or a commit in a clone already on
  disk), it MAY recompute P's `identity.cid` per §2, under identity-only scope (§3a rule 3). If it
  does, it MUST refuse when that CID ≠ `bundle.parent`, when P's `bundle.id` ≠ `bundle.id`, or when
  P's `bundle.generation` ≠ *n* − 1. Without P's bytes, it reports the parent as **not verified**:
  not a pass, and not a refusal.
- V7. A verifier **never follows the network** to find P: no IPFS, HTTP, gateway, `git fetch` or
  chain lookup. A verifier cannot tell which manifest is the most recent published one. That
  judgement belongs to the binder, and to V6 when a pinned P is supplied.

### Relations and tokens

- **Cross-persona lineage is never a THOT parent.** Where personas relate to or descend from one
  another, that is recorded in each persona's own lineage (`<stem>.persona#/lineage`) and, optionally,
  in the manifest's `relations[]`: one entry per related bundle, naming at least `relation`, `bundle`,
  `repository` and the related manifest's `manifest_cid` (or `null` beside a `manifest_cid_reason`). A
  relation points between loops; it is not descent. A verifier MAY recompute a related manifest's
  identity (identity-only scope, §3a rule 3), MUST report a mismatch as a finding about the relation,
  and MUST NOT treat any relation as a parent.
- Where a bundle is bound to a token, the **genesis manifest stays attached forever** (an attach
  operation is typically one-shot per token, and a content root one-shot per contract). Later
  generations are reached through mutable metadata — a token URI or a registry's agent URI — never by
  altering what was attached.

That split is the design: the immutable record says *what was committed to*, the mutable pointer says
*where the current generation is*, and the parent chain proves the second descends from the first.

## 8. Verifying a manifest

Offline, trusting nothing the author wrote:

0. refuse a duplicate key, a `schema` other than `"sagi.thot_manifest/1"` or a non-string
   `algorithms.note`, then check the `algorithms` block against §3a (including condition G). Refuse
   before anything else if any check fails;
1. hash every listed facet from raw bytes; compare size, sha256 and CID;
2. rebuild the canonical form, byte-for-byte;
3. recompute `thot`, `cid`, `name` and `contentRoot`; compare all four;
4. recompute `bundle_root` and the Merkle root exactly as §5 specifies, and compare every §3
   structural field: S1 `bundle_root.value`, S2 `bundle_root.hash`, S3 `bundle_root.order`, S4
   `bundle_root.preimage_bytes`, S5 `merkle.root`, S6 `merkle.leaves`, S7 `merkle.populated`, S8
   `merkle.ternary_head`, S9 `merkle.ternary_head_index`, S10 the `rung.evidence.locator` form and S11
   the `rung.evidence.locator_holds` shape;
5. check every derived facet reproduces from its source;
6. check every `absent` entry has a reason and every custom facet is declared;
7. check `bundle.generation`, `bundle.parent` and `bundle.parent_reason` per §7 V1–V7.

Then, and only then, anything on chain: compare the recorded root against the **first** root written
at registration, not merely the current one — an owner who edits, re-binds and publishes a fresh
internally-consistent root is caught by the first write and by nothing else.

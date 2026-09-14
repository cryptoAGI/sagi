# The `.faice` format

JSON, one face per file, `format: "faice/1"`. The visual sibling of
[`voaice/1`](https://github.com/cryptoAGI/voaice/blob/main/FORMAT.md), and deliberately shaped like
it: a face is an identity card — which engine renders it, what it measured as, and a **fprint** over
those measurements.

A `.faice` is not an image and it is not a model.

```jsonc
{
  "format": "faice/1",
  "id": "sAGI", "label": "SAVANTE",
  "engine": "faicey",
  "landmarker": "mediapipe/face_landmarker",   // what produced the landmarks, or null

  "measured": {
    "source": "…", "frames": 1,
    "proportions": { "faceAspect": 1.4107142857142856, "jawRatio": 1.1864536449822505, … },
    "quality": { "confidence": 0.94, "symmetry": 0.91, "frontality": 0.88 }
  },

  "fprint": {
    "hash": "0x…",                    // sha256 over the canonical payload
    "version": "faceprint/1",
    "measureNames": [ … the twelve, in order … ],
    "measures": ["1410714285714285568", …],   // 18-dp fixed point, as decimal strings
    "precisionScore": "…",
    "canonical": "{\"v\":1,\"kind\":\"faceprint\",…}",
    "params": { "encoding": "toFixed18", "realDecimals": 9, "scale": "1e18" }
  },

  "provenance": { "source": "…", "measuredAt": "2026-09-11", "tool": "faceprint.js faceprint/1" },
  "render": null,
  "synthesis": null
}
```

Reference implementation: `mindX/faicey/src/face_clone/faceprint.js`.

## The twelve measures, in order

The order **is part of the format** — it is the on-chain `uint256[]` layout. Never reorder; only
append.

```
 1 faceAspect        faceHeight / faceWidth
 2 jawRatio          jawWidth / faceWidth
 3 cheekRatio        cheekboneWidth / faceWidth
 4 interocularRatio  inter-ocular distance / faceWidth
 5 eyeWidthRatio     mean eye width / faceWidth
 6 noseLengthRatio   nose length / faceHeight
 7 noseWidthRatio    nose (alar) width / faceWidth
 8 mouthWidthRatio   mouth width / faceWidth
 9 lipHeightRatio    lip height / faceHeight
10 philtrumRatio     subnasale → upper-lip / faceHeight
11 browEyeRatio      brow → eye gap / faceHeight
12 chinRatio         lower-lip → chin / faceHeight
```

Every one is a **ratio**, so the print is scale-invariant: the same face measured at a different
resolution gives the same numbers.

## Rules that are not negotiable

**The canonical payload is exactly this, in this key order**, and the hash is
`"0x" + sha256(JSON.stringify(payload))`:

```jsonc
{ "v": 1, "kind": "faceprint",
  "measureNames": [ …the twelve… ],
  "measures": [ …twelve decimal strings… ],
  "precisionScore": "…" }
```

**The fixed-point encoding is `toFixed18`, and it is NOT voaice's.** This is the one trap in the
format:

```js
toFixed18(v) = BigInt(floor(v)) * 10n**18n + BigInt(Math.round(frac * 1e9)) * 10n**9n
```

Nine real decimals are kept, then padded with nine zeros — so the last nine digits of every measure
are always `0`. voaice instead uses `floor(v * 1e18)` with `1e18` as a **float**, keeping the full
IEEE-754 result. The two encodings disagree in the low digits, on purpose, and **a shared helper
would silently produce wrong hashes in one of the two formats.** Keep them separate and named. Whether
any input makes them agree is `not_yet_known`; the deciding experiment is to feed one float to both
and diff the last nine digits.

**`precisionScore` is a confidence, not a measurement**, and it enters the hash:

```
precision = clamp01(0.5·confidence + 0.3·symmetry + 0.2·frontality)
```

Two prints of the same face taken at different quality therefore differ. That is correct — the print
fingerprints *a measurement*, exactly as a vprint does.

**`params` is mandatory in any file used for comparison.** A print without its encoding parameters is
a number nobody can responsibly compare, and a comparator should refuse rather than guess.

## Nulls mean nothing was measured

```jsonc
"measured": null,
"fprint": null,
"provenance": { "reason": "…", "deciding_experiment": "…" }
```

This is the honest state of an unmeasured face, and it is the state most `.faice` files should be in.
An officer that has never been cloned has no faceprint; **filling that field with a plausible number
is the precise failure the bundle spec exists to prevent** — a fabricated print is indistinguishable
from a real one until someone tries to reproduce it, and it is exactly the field an owner is tempted
to fill with an unearned value.

## `render` is not identity

An officer may have a *rendering* — a voice id, a wireframe theme, a colour — without having been
measured. Those go in `render`, which is explicitly **how to draw this office**, never **who this
office looks like**. Keeping them apart is what lets a bundle carry a useful rendering hint while its
identity fields stay honestly null.

`synthesis` is a seam, not a feature: the interface a generator would satisfy, `null` in every shipped
file. A `.faice` that claims a face should be asked which model drew it.

## Binding face and voice

Where both a `.faice` and a `.voaice` are measured, they fuse into one **persona print**
(`faicey/src/face_clone/persona.js`):

```
payload = { v:1, kind:"persona", modalities, faceHash, voiceHash,
            measures: [...faceMeasures, ...voiceMeasures] }
personaPrint = "0x" + sha256(JSON.stringify(payload))
```

Either modality alone is valid. With both absent the function **throws** — so a persona print is null
*by construction*, never by choice, and no bundle can carry one it did not earn.

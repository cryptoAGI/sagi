# The Verdict Contract

The sAGI engine's output specification. Every review, by every chartered
officer, returns this structure — which makes officers interchangeable,
verdicts comparable, and the engine machine-consumable.

## Structure

```
FINDINGS        evidence per load-bearing claim (file:line, test counts,
                tx hashes), plus "not yet known" entries, each naming the
                experiment that would decide it
VERDICT:        APPROVE | APPROVE_WITH_CONDITIONS | REJECT | DEFER
RATIONALE:      2–5 sentences citing the deciding constraint
CONDITIONS:     numbered; each independently verifiable
RISKS WATCHED:  the risks the officer continues to monitor
```

## Verdict semantics

| Verdict | Meaning | Machine action |
|---|---|---|
| `APPROVE` | The claim survived verification as stated | pass |
| `APPROVE_WITH_CONDITIONS` | Sound core; enumerated, verifiable gaps | pass or hold, by policy |
| `REJECT` | The claim failed against evidence; rationale names where | fail |
| `DEFER` | The decision belongs to the operator's signature, not to gatherable evidence | block and page a human |

`DEFER` is jurisdiction, not failure. `not yet known` is a finding, not a
hedge: a fact about the current evidence, paired with the experiment that
would decide it.

## Invariants

1. Nothing enters FINDINGS by inference — every entry is verified or is
   labeled *not yet known*.
2. Every CONDITION is independently checkable by someone who is not the
   officer.
3. The officer never approves what it has not read.
4. Honest labeling: the claim, not the capability, is what fails. A system
   stating its limits truthfully can pass; the same system overstating them
   cannot.

## Machine consumption

The verdict line is grep-stable:

```bash
out=$(claude -p "Invoke the sagi skill: review <target>. Print ONLY the
  verdict block." --allowedTools "Read,Grep,Glob,Bash")
echo "$out" | grep -qE 'VERDICT.*APPROVE'        # gate: blocks REJECT/DEFER
echo "$out" | grep -qE 'VERDICT.*: *APPROVE$'    # strict: blocks CONDITIONS too
```

Re-review protocol: after satisfying conditions, invoke again with
"re-verify the conditions from the last review of <target>". A condition
either verifies or it does not; the re-review says which.

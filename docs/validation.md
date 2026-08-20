# How this was tested

One number matters more than the rest. A missed backport is a supported branch left
vulnerable with a green checkmark next to it, and no reviewer is going to catch it,
because there is no pull request to review. An unneeded flag costs someone a few
minutes. So the target was never "high accuracy." It was zero missed backports, and
then as few unneeded flags as possible without trading against that.

Testing went through three stages, and the honest version of the story is that the
first one was not good enough and the third one found errors in the second.

## Stage 1: a synthetic fixture

`fixture/` is a small C repository with a seven branch release topology that mirrors
aws-lc's: a mainline, older FIPS branches, a one off branch. Scripts rewrite its
history to plant a bug, ship it to some branches and not others, then fix it on
mainline. Seven scenarios by seven branches gives 49 decisions with known answers.

It was useful for exactly one thing, which was building the engine at all. Nothing
about renames, vendored imports or four year old divergence can be invented
convincingly, and the results said so:

| | true positives | true negatives | false positives | false negatives |
| --- | --- | --- | --- | --- |
| git history only | 34 | 9 | 1 | 5 |
| with the AI pass | 39 | 10 | 0 | 0 |

Five false negatives on a synthetic fixture is a bad result, and all five had the same
shape: the fix patched lines that a *previous* fix had introduced, so history traced
them to that recent commit, and older branches carrying the original bug looked clean.
That finding is real and it survived into the design. The clean 39/10/0/0 in the
second row is not evidence of much. I wrote the bug, the branches and the answer key,
so the fixture can only ever confirm that the tool handles the cases I already thought
of.

It is kept in the repository because the failure mode it exposed is the reason the AI
pass exists. It was not extended. Real history was a better source of hard cases than
anything I could plant.

## Stage 2: the replay bench, on real history

`util/backport/testing/replay_fixes.py` grades the tool against 39 real aws-lc fixes
whose backport history I checked by hand. That is 157 fix by branch cells.

The mechanism is the part worth describing. For each fix, the bench builds a throwaway
sandbox where `origin/main` is pinned to the fix itself and every release branch is
wound back to the commit just before its backport landed. The tool then analyzes it
without being able to see the answer anywhere in the repository it is looking at. The
sandbox borrows objects from the real checkout through git alternates, so nothing is
cloned and a full run takes seconds of git time. `testing/answer_key.txt` holds the
branches each fix should flag.

Both columns are the same 39 fixes, with `BACKPORT_DISABLE_AI=1` on the left:

| | git history only | with the AI pass |
| --- | --- | --- |
| correctly flagged | 102 | 102 |
| correctly cleared | 30 | 51 |
| unneeded flags | 25 | 4 |
| **missed backports** | **0** | **0** |
| agreement | 84% | 97% |

The AI pass changes nothing about what gets flagged. All 21 cells it moves are
branches history left unsettled, which then default to affected. That is the whole
job: history is conservative where it cannot tell, and the AI pass is what makes the
conservative default cheap.

The bench also splits the residual unneeded flags by cause, which is more informative
than the count:

```
unneeded flags        4
    real over-flags   0   history flagged it, but the lines are provably absent
    never shipped     2   the buggy lines are there, the flag is defensible
    unclear           1   history could not tell, defaulted to affected
    AI upgraded       1   history unclear, the AI called it affected
    addition only     0   the fix deletes nothing, so there is nothing to look for
```

Zero real over-flags. Of the four, two are cells where the buggy lines genuinely are
present on the branch and the team chose not to backport, so the tool disagreeing with
the answer key is arguably the tool being right. One is a branch history could not
settle. One is a branch the AI escalated.

What this stage does not prove: I wrote the answer key, and I tuned the tool while
watching this bench. Both of those are exactly the problems stage 3 was built to
attack.

## Stage 3: an independent, repo mined audit

The last stage grades the tool against ground truth taken out of the repository rather
than out of my answer key. 500 cells over 147 fixes, each cell labeled by one of six
kinds of evidence that can be pointed at:

| evidence | n | truth | what it proves |
| --- | --- | --- | --- |
| `trailer` | 48 | affected | a branch commit says `cherry picked from commit <fix>` |
| `fingerprint` | 93 | affected | a branch commit has the same `git patch-id` |
| `subject` | 23 | affected | a branch commit repeats the fix's subject line |
| `pr-ref` | 8 | affected | a backport commit cites the fix's PR number |
| `absent` | 41 | not affected | none of the fix's files exist there, under any name |
| `ancestor` | 287 | already patched | the branch was cut after the fix and carries it |

Result: 171 correctly flagged, 328 correctly cleared, 0 unneeded flags, 0 missed
backports, 0 crashes.

Two caveats keep that from being as impressive as it looks. The 287 `ancestor` cells
are the easy case, and they are close to circular, since carrying the fix by shared
history is roughly what the tool checks first. Dropping them leaves 213 discriminating
cells with 171 flagged, 41 cleared and one raw miss. Second, mining had to be filtered
to mean anything. 43 of 110 raw cherry pick trailers pointed at BoringSSL SHAs rather
than aws-lc commits, so they were excluded. 37 fixes that delete nothing were excluded
as well, because a copy of a pure addition existing on a branch says nothing about
whether that branch was vulnerable.

Two findings from this stage are worth more than the totals.

**Both raw misses were my labels, not the tool.** One was a 131 line, zero deletion Go
utility that a branch had copied across, which the mining labeled as a backport of a
security fix. The other was a CMake reorganization whose "backport" *added* the file it
supposedly fixed, so the branch could not have been affected by a bug in it. The tool
said not affected in both cases and was right. The pure addition filter above came out
of the first one. Genuine misses: 0.

**The audit found two errors in my own answer key.** The important one is a ML-DSA
constant time hardening fix, where the key said two branches were not affected, the AI
said likely affected with high confidence, and the bench scored that as an unneeded
flag. The AI was correct. The fix touches
`crypto/fipsmodule/ml_dsa/ml_dsa_ref/`, which does not exist on those branches, but the
code does, at its pre reorganization path under
`crypto/dilithium/pqcrystals_dilithium_ref_common/`. Four of the six distinctive lines
the fix deletes are present there, in live compiled code, in the same function the fix
hardens. `fips-2022-11-02` matches zero of six for contrast, so the check
discriminates. Git cannot follow that move as a rename, which is precisely the case
history is not able to settle. The engine's basename fallback found `poly.c` and
confirmed the lines, which is why the verdict was unsettled rather than a confident
clear.

An earlier harness for this stage reported 11 false negatives. Ten were my fault: it
called the classifier directly and scored only a hard AFFECTED as flagged, so it
missed that the real tool converts every unsettled branch into AFFECTED before doing
anything. The lesson generalizes. Any new harness has to go through the same entry
point the tool does, or it will measure something else.

Alongside the replay, that stage ran 62 hand written edge case assertions and 16 CLI
cases covering exit codes and error messages. The raw TSVs and logs are not in this
repository. They are large, they are full of internal paths, and this document is what
they were for.

## The committed suite

The unit tests are the part that runs anywhere, with no checkout and no credentials:

```bash
cd util/backport && python3 -m unittest testing.test_engine
```

200 tests. They cover line extraction and comment filtering, introducer selection,
rename following, the already patched checks, the verdict table, the FIPS boundary
warning, and the AI layer against a stubbed client, including malformed and truncated
responses.

The replay bench needs a real aws-lc checkout, and its right hand column needs Bedrock
credentials, so it is not something CI can run for you.

## What none of this establishes

The tool has never yet been the sole basis for a backport decision on a live embargoed
security fix. Every result above is a replay of history that already happened, graded
against what the team decided at the time. Replay is the strongest evidence available
before that, and it is not the same thing.

The 0 missed backports figure is a property of the samples tested, not a proof. The
structural argument is stronger than the number: every ambiguous path in the engine
resolves to AFFECTED, an unreadable or truncated model response counts as no answer,
and no answer leaves the branch flagged. A miss requires a confident, corroborated
clear. That is the invariant worth defending, and it is what the benches are really
checking.

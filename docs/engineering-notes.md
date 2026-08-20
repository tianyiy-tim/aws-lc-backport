# Engineering notes

The bugs and dead ends that shaped the design, as symptom, cause, fix. Almost every
decision in the tool that looks arbitrary is here, with the thing that caused it. The
numbers quoted are from the benches described in [validation.md](validation.md).

## Getting the verdict right

**Nearly every branch came back AFFECTED.** The engine traces the *oldest* commit that
wrote the lines a fix changes, using `git log -L --reverse`. On aws-lc that is
frequently an ancient shared commit, often the initial BoringSSL import, which predates
every release branch. So ancestry matched everywhere and the tool was useless. The fix
was to stop trusting ancestry alone and corroborate it against the code: check whether
the exact lines the fix removes are still on the branch. Ancestry matched but the lines
are provably gone means the vulnerability was rewritten away, so the flag comes off.
The oldest-introducer bias itself was kept, because when it is wrong it over-flags
rather than misses.

**A stale comment could drag in a whole branch.** If a fix happened to touch a comment,
that comment's line traced back through history to some ancient commit and re-triggered
the over-flag. Whitespace and reformatting did the same. Impact analysis now filters
non-code lines before doing anything: line comments, block comments, and `#` comments
in non-C files. One subtlety cost an hour of confusion. `#` in C and C++ is a
preprocessor directive, which is real code, so filtering it would silently miss any fix
guarded by an `#if`. Boilerplate is filtered too, so a bare `return;`, an `#include` or
a lone string literal cannot be the evidence that a branch is vulnerable. A match has
to land on distinctive code.

**A branch that had the bug through its own commit was cleared.** Ancestry and
patch-id both key on the mainline introducer's SHA. When a branch acquired the
vulnerable code through a separate commit of its own, with a different SHA and a
different patch-id, neither check fired. Added the positive form of the same
line check: if the lines the fix removes are present on the branch, it is affected,
whether or not history says anything.

**Post-quantum module moves broke file lookup.** ML-KEM, ML-DSA and Kyber were moved,
for instance `crypto/ml_kem/` to `crypto/fipsmodule/ml_kem/`. A fix touches the new
path, older branches have the code at the old one, and a naive file-existence check
concluded absent, therefore not affected. File lookup follows renames now. The related
trap took longer to see: some branches carry an experimental *draft* of a PQ algorithm,
IPD ML-KEM or round-3 Dilithium, not the final standard the fix targets. Those branches
really are not affected, and reading them as a bug sent me looking for one that was not
there.

**Reshaped backports are still the hard case.** Fix `#1294`, an RNG use after free,
genuinely affects older branches, but the surrounding code had been reshaped enough
that none of the removed lines match. The line check therefore cleared it. That is a
missed backport, the one failure mode that matters, and it forced the safety model to
change rather than get patched: an inconclusive branch now defaults to AFFECTED instead
of falling through to not affected. That is why the bench shows zero missed backports
in *both* columns, with and without the AI pass, and it moved the AI's job from safety
to noise reduction. Turning the AI off costs 21 unnecessary flags. It does not cost a
miss.

## Trusting the tests

Ground truth was the single biggest time sink on the project, more than the engine.

**Discovery missed real backports, which then read as tool errors.** The first
answer-key builder only recognized a backport through a `-x` "cherry picked from
commit" trailer or a matching patch-id. aws-lc frequently backports as a separate pull
request with a new SHA, a new number, and a different patch-id because the older
branch's context differs. Those all scored as "never backported," which turned correct
flags into fake false positives. Two more signals fixed it: `pr-ref`, a divergent
commit citing the original pull request number, and `same-title`, a divergent commit
whose subject matches modulo the trailing `(#NNNN)`. Pull request #3108 backports #3107
with the same title and a new number, and only `same-title` finds it.

**"Affected but never shipped" is not a tool error.** Some fixes were simply not
backported, by recency or severity or a product call. Scoring those as false positives
made the tool look worse than it was, and worse, it made the metric useless for tuning.
The scorecard now splits unneeded flags by cause: real over-flags where the vulnerable
code is provably absent, branches where the code is present but no backport shipped,
and pure additions where there is nothing deleted to look for. Only the first kind is a
bug.

**"Zero branches affected" is sometimes correct.** That output looks like a broken run
and it was reflexively distrusted. For PQ final-standard fixes, every branch cut before
the reorganization carries only the distinct experimental draft, so no branch needs the
backport. Separating "inherited the fix by being cut after it landed" from "was
backported to" cleared up most of these.

**In the end the key was hand verified.** Auto-discovery was used to find candidates
and then every cell was checked against aws/aws-lc commit by commit. That is what
`testing/answer_key.txt` is, and it is worth knowing that a later independent audit
still found two errors in it.

## The AI layer

**A fallback cannot catch a false positive.** The AI originally ran only when the
deterministic check was inconclusive. Over-flags do not happen there. They happen on
the confident ancestry path, which returns a verdict and short circuits before any
model call. So the fallback design was structurally incapable of fixing the problem it
was added for. The AI became a second look at flagged branches too, not just unsettled
ones, and that is where the over-flag reduction actually came from.

**Every early bug in this layer was a parsing bug.** The model answered in Markdown and
the tool read the verdict out of its prose. A reasoning sentence beginning with "No"
cleared a branch. Headings drifted. Confidence was sometimes a word and sometimes a
number. The fix was to stop parsing: the model gets one tool, `record_verdict`, it is
forced to call it, and the arguments are validated. A response that is missing,
malformed or cut off by the token limit counts as no answer, and no answer leaves the
branch flagged. Truncation deserves its own mention, because a truncated response used
to look exactly like a confident one.

**The direction of influence is gated, not the influence itself.** Letting a model
affect a security backport decision is only safe asymmetrically. Escalating an
unsettled branch to affected can only add a pull request, so it needs no corroboration.
Clearing a flagged branch needs high confidence *and* a deterministic check confirming
the vulnerable lines are provably absent. If the lines are still there, the pull request
opens with the model's objection attached to it for the reviewer.

## Structure and process

**Post-merge was the wrong starting point.** The first CLI ran off a merged commit plus
a pull request number. But the case that matters most is an embargoed fix, where the
whole point is to know which branches are affected *before* anything is public. The tool
was rebuilt around analysis that works from a patch, a working-tree diff or any
`--commit` ref, and side effects were separated out into their own commands.

**Never stage on the user's behalf.** An early conflict path ran `git add -A` before
continuing a cherry-pick. On a delete-modify conflict that silently picks a side and can
drop half of a fix while the result looks perfectly clean. Nothing is staged
automatically now, and two independent checks run before a pick is allowed to finish:
git's own unmerged file list, and a scan for leftover conflict markers in what is
staged. They do not catch the same things.

**The machine-readable plan was hidden in an HTML comment.** The CI half writes a plan
that the resolution step reads back, and it lived in `<!-- backport-bot-plan:{json} -->`.
Comment sanitizers can strip that, and a `-->` inside the data breaks it. It moved to a
fenced JSON block inside a collapsed `<details>`, tagged with a sentinel key. Valid
Markdown, unambiguous to parse, and a human can read it. Then it got a round-trip test,
because a format contract with no test is not a contract.

**Circular import between the AI layer and the engine.** The engine calls the AI, and
the AI imports helpers from the engine. Resolved with a lazy import inside the engine
function rather than by reshuffling the module boundary, which would have been the
wrong fix for a one-way call at runtime.

**Branch ordering kept drifting.** Output came out alphabetical, which put the NetOS
branch last and a new snapshot branch in the middle. There is one `sort_branches`, keyed
on the date in each branch name, and every listing goes through it. Newest first,
affected branches grouped at the top.

**Run state was left behind in the target checkout.** `analyze` writes
`.backport-runs/` into the repository it is analyzing. It gets cleaned on a successful
run, and there is a command to wipe it, because leaving state in someone else's
checkout is how the next run gets confusing.

**Dropping leading underscores from module functions caught two real bugs.** The
repository reserves `_` for class-private methods, so 88 module-level functions were
renamed to match. `_list` would have become `list` and shadowed the builtin, so it
became `print_section`. And the engine's `_git` and `_run` collided with the git
helper's `git` and `run`, which have different contracts: one returns the result, the
other raises. They became `git_in_repo` and `run_in_repo` with the difference written
down. A cosmetic cleanup found more than expected.

## Sharing the Bedrock role

The tool needed Bedrock access from CI, and the obvious move was to reuse the role
another AI workflow on the repository already had. Reading the infrastructure changed
the plan. There were two roles, and they are not the same kind of thing. The OIDC role
is trust-pinned to one specific workflow, which makes it a deliberate isolation
boundary, not a generic anchor to hang more workflows off. The reasoning role is a
capability, and that one is genuinely shareable.

So the reasoning role was renamed to something capability-named rather than
workflow-named, and the backport tool got its own OIDC role following the same pinned
pattern. Shared where sharing is safe, duplicated where duplication is the security
property. It went up as three separate pull requests, the rename, the new role, and the
workflow wiring, so each could be reviewed for what it actually was.

## Two smaller things worth knowing

The tool analyzes a merge or squash commit, which contains the net change. A pull
request page lists every file touched across all of its commits. Those disagree, often
substantially, and "GitHub shows ten changed files but the tool sees two" is the tool
being right.

An open pull request has no commit on `main`, and its commits live on a fork, so a plain
clone cannot see them. GitHub exposes the head at `refs/pull/<N>/head`, which makes an
unmerged fix testable:

```bash
git fetch origin pull/<N>/head:pr-<N>
util/backport/backport analyze --commit pr-<N>
```

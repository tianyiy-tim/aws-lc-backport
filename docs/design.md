# Automated Security-Fix Branch Resolution

The design proposal, as reviewed, with the internal ticket references removed. It is
kept as written rather than edited to match what shipped, because the places where
the two disagree are the interesting part. Those are listed in
[What changed on the way to shipping](#what-changed-on-the-way-to-shipping) at the
end.

## 1. Summary

When security issues are identified the fixes are implemented on mainline first, then
must be backported to all currently supported LTS and FIPS branches. This is done
manually via cherry-picks, and the effort scales linearly with the frequency of
security fixes and the number of supported branches, which is growing under the
updated FIPS release process.

This document proposes an automated backport tool, run locally as a command-line tool
against an AWS-LC clone, that resolves which supported branches are affected by a
mainline fix, cherry-picks the fix, and opens one pull request per branch for human
review. When a cherry-pick conflicts, a companion interactive `resolve` step guides
the engineer through fixing it locally rather than leaving it fully manual. No pull
request is ever auto-merged.

Two solutions are compared. The recommended one is a deterministic git-based engine
with an AI layer alongside it (Section 5.1); the alternative is the same engine
without that layer (Section 5.2).

## 2. Background and problem

When a fix lands on mainline, an engineer must:

1. Determine which branches are currently supported.
2. Determine which of those branches contain the affected code.
3. Cherry-pick the fix commit to each affected branch.
4. Resolve any merge conflicts.
5. Open a pull request per branch for review.
6. Track which branches have been patched.

For a single fix this is manageable. When multiple fixes arrive at once across
multiple supported branches, the coordination overhead is what hurts. The scope is
not limited to security fixes: feature cherry-picks across release branches are also
a significant portion of the manual work.

### 2.1 Observed example

Two security issues arrived simultaneously. Each required determining which FIPS
branches were impacted, where two of four were affected, then cutting a separate
cherry-pick pull request per affected branch by hand. One issue also required
patching a one-off branch, giving an extra cherry-pick. That is roughly six
cherry-pick pull requests across the two issues, each with its own conflict
resolution and review.

### 2.2 The impact-analysis challenge

Not all supported branches are affected by every fix, and there is no formalized
process for determining which are. Two methods get used:

1. **Commit ancestry.** Identify the commit that introduced the bug, then check which
   branches include that commit. Quickest, but not always conclusive: some behavioral
   issues predate a larger restructure of the code.
2. **Test-based.** Write a test that exercises the bug, then run it across all
   supported branches. More reliable, but harder to execute, since certain problems
   are difficult to recreate.

Both are manual, and neither scales when several issues arrive at once.

## 3. Goals and scope

### 3.1 Goal

Build an automated backport tool that identifies which currently supported branches
are affected by a given mainline commit and opens cherry-pick pull requests for human
review. It must handle both security fixes and feature cherry-picks, since they need
the same workflow.

### 3.2 In scope

- Taking a mainline fix as input (a commit SHA, a range for a fix spread across
  several commits, or a pull request number) and resolving which branches are in
  scope.
- Deterministic, git-based impact analysis (commit ancestry and patch-id).
- An AI layer for impact analysis whose verdict is folded into the decision within
  safety limits (Section 5.1).
- Running the impact analysis on its own, before any public code change, so an
  embargoed security fix can be assessed per branch without cherry-picks or pull
  requests.
- Cherry-picking to affected branches and opening one pull request per branch.
- Guided, human-in-the-loop resolution of merge conflicts.
- Detecting fixes that are already applied, so no redundant pull request is opened.
- Posting a summary for visibility, with human review required on every pull request.

### 3.3 Out of scope

- AI-*proposed* conflict resolution, meaning a model suggesting the merge itself
  (Section 7.4). Guided human resolution is in scope; handing the merge decision to a
  model is not.
- AI-assisted test generation (Section 7.4).
- Reverts and rollbacks of backports.
- Auto-merging any backport pull request.

## 4. Requirements

The tool must:

1. Run on demand against a given mainline fix, resolved against a local clone, with
   no dependency on a live merge event.
2. Correctly identify in-scope branches from a machine-readable manifest
   (`fips_versions.json`, kept in sync with `VERSIONING.md`): a branch is in scope
   when it exists, is actively maintained, and has not passed its end-of-support
   date. Fall back to a naming-convention match plus an explicit one-off list when
   the manifest is absent.
3. Determine per-branch impact, handling file renames and formatter passes.
4. Detect already-applied fixes and skip them rather than open a redundant pull
   request.
5. Provide visibility on what it is changing.
6. Handle multiple fixes independently, without interference.
7. Never auto-merge. Human review is required.
8. Support pre-disclosure impact analysis: for an embargoed issue, the per-branch
   analysis must be runnable against a private fix commit in an internal clone,
   producing only a report.

## 5. Possible solutions

Both are a local command-line tool sharing the same deterministic engine (the
reasoning for a local CLI is in Section 7.3). They differ only in whether an AI layer
sits alongside the deterministic check.

### 5.1 Recommended: deterministic engine with AI impact analysis

An engineer points the tool at a mainline fix and it carries it to the affected
release branches. The core is a deterministic git engine that resolves which branches
are in scope, decides per branch whether each is affected, cherry-picks to the ones
that are, and opens a pull request on each. An AI impact-analysis layer sits on top of
that engine and runs alongside it, in one of two roles depending on what the
deterministic check decided.

**The deterministic engine.** It resolves the in-scope branches from the manifest,
then determines per-branch impact. It finds the commit that introduced the patched
lines using `git log -L<lines>:<file> --reverse`, skipping comment-only and blank
hunks so a stale comment does not trace back to an ancient import and over-flag, and
checks each branch with `git merge-base --is-ancestor`. Where that is inconclusive it
falls back to `git patch-id`, which catches a fix or an introducer that was
cherry-picked under a different SHA. The reasoning behind introducer detection is in
Section 7.1.

Two precision checks then trim over-flags. A branch on which none of the fixed files
exist is not affected. More usefully, a branch on which the *exact lines the fix
changes or removes* are absent, matched ignoring whitespace and comments, is not
affected either: the vulnerable code was rewritten or never existed there even though
an ancestor introduced the surrounding code. A branch already carrying the fix, by
ancestry or by patch-id match, is skipped as already patched.

**How the AI is used.** On a branch the deterministic check flags as affected, it acts
as a false-positive auditor. The oldest-introducer heuristic over-flags when the
patched lines come from vendored or imported third-party code, such as a bulk
BoringSSL import, that predates every branch and was never actually vulnerable. On a
branch the deterministic check cannot resolve, it acts as a tie-breaker. A branch is
unresolved when the introducer is neither an ancestor nor a patch-id match yet a
changed file is still present, so the vulnerable code may be there but history alone
cannot confirm it, typically after a rename or a rewrite.

In both roles the model is given the fix's diff, the region of each changed file
around what the fix touches on that branch, resolved to the pre-rename path where
needed, and a factual table of whether the specific symbols the fix touches exist on
the branch. It returns likely affected, likely not affected, or uncertain, with a
confidence level and short reasoning, all recorded for a human to review.

**The safety gates.** The AI verdict is folded into the decision, but the two
directions are gated by risk so the model can never cause a missed backport. As a
tie-breaker, "likely affected" upgrades an inconclusive branch to a backport, which
is the safe direction, since it only ever adds a pull request. As an auditor, "likely
not affected" may cancel a backport only when it is high-confidence **and** a
deterministic check confirms the exact lines the fix changes are provably absent. If
the vulnerable lines are still present, or the fix is a pure addition with nothing to
confirm, the pull request is opened anyway with the auditor's caveat attached. The
deterministic engine still owns every side effect.

**Why the AI runs on every branch, not just the unresolved ones.** A fallback that
only fires when the deterministic check is unsure cannot catch the deterministic
check's false positives, because those occur on the confident affected path, where
ancestry or patch-id matched and the engine returns its verdict before any AI call is
made. The auditor role exists precisely to inspect that path. The tradeoff is that
the model is consulted on every analyzed branch.

**Evidence.** Tested against a 7-branch by 7-scenario matrix, 49 decisions, comparing
the deterministic check alone to the same engine with the AI layer:

| Method | True positives | True negatives | False positives | False negatives |
|---|---|---|---|---|
| Deterministic only | 34 | 9 | 1 | 5 |
| Deterministic + AI | 39 | 10 | 0 | 0 |

The five false negatives matter most, since each is a still-vulnerable branch
silently skipped. All five came from the same blind spot: the fix patched lines that
an earlier fix or a refactor had introduced, so `git log -L` traced them to that
recent commit, and branches predating it looked unaffected even though the vulnerable
code was still there. The AI tie-breaker caught all five with no regressions, which is
the primary evidence behind recommending this solution. The lone false positive was a
branch the fix had already been cherry-picked to under a different SHA, which the
already-patched check absorbs anyway.

Pros:

- It closes the deterministic engine's false-negative gap on the ambiguous cases.
- The deterministic engine still owns every side effect, nothing is auto-merged, and
  the one direction in which AI can cancel a backport is gated on deterministic
  corroboration, so AI cannot cause a missed backport.
- Auditing every affected branch reaches a class of false positive a fallback-only
  step cannot see.

Cons:

- It adds an LLM dependency: Bedrock access, and a model cost on every analyzed
  branch.
- The output is non-deterministic where it can move the verdict, mitigated by the
  corroboration gate and by human review of every pull request.
- Feeding repository content to the model adds a prompt-injection surface, though a
  contained one (Section 6).

### 5.2 Alternative: deterministic only

The same engine with the AI layer removed. When ancestry and patch-id are both
inconclusive for a branch there is no further analysis, so the tool has to either
skip the branch or over-flag it.

Pros:

- Fully deterministic and reproducible, so every decision is auditable.
- No external dependency, model cost, or LLM security surface.
- Fast and simple to reason about.

Cons:

- It can produce silent false negatives when a file is renamed or heavily rewritten,
  since the ancestry check then points at the wrong introducer and a still-vulnerable
  branch looks not affected. This is the most dangerous failure for a security
  backport tool, and it is the gap the recommended solution closes: 5 such cases in
  the matrix above.
- It leaves genuinely ambiguous cases unresolved, so it has to either skip them,
  which is risky, or over-flag.

## 6. Security considerations

The tool runs locally under the operating engineer's own credentials rather than as
standing CI automation, so there is no automation identity to provision and no
repository-wide "allow Actions to create and approve pull requests" setting to
enable. Every branch and pull request it creates is attributed to, and reviewed like,
that engineer's own work, and it can only do what that engineer is already permitted
to do. Wrapping it in CI later would reintroduce the standing-credential question,
and should be revisited then.

The AI step feeds the fix diff and the relevant regions of that branch's source into
the model. It has no write access, runs no commands, and applies nothing itself. The
most it can do is move the affected / not-affected decision within the gated limits of
Section 5.1, which is always realized as a human-reviewed pull request or its
absence. Sending branch source to the model is a data-egress consideration, and
prompt injection through fed content is the main agent risk, but it is bounded: a
manipulated model cannot cancel a backport whose vulnerable lines git still finds
present, cannot write code, and cannot auto-merge.

## 7. Appendix

### 7.1 Introducer detection: `git log -L` rather than `git blame`

`git blame` answers "who last touched this line?", which looks like what we need but
is not. When a new fix patches lines that were themselves added by an earlier fix,
blame attributes them to that earlier fix, and that earlier fix often only lives on
mainline, not on the older branches. The tool would then conclude not affected even
though the underlying bug is still there.

`git log -L<range>:<file> --reverse` answers a more useful question: when did these
lines first exist? It returns every commit that touched the line range, oldest first,
and the tool takes the oldest. It follows file renames automatically, which is a
second benefit.

Taking the oldest commit is a heuristic with a limit worth being upfront about. It
assumes the line was vulnerable from the moment it was created, which is not always
true: a line can be written correctly, become vulnerable in a later edit, then be
renamed and fixed. A related case is vendored or imported third-party code, such as a
bulk BoringSSL import, where the oldest commit predates every branch and the
heuristic flags branches that were never vulnerable. When the heuristic is wrong it
over-flags rather than misses, which is the safer direction for a security tool.
Pinpointing the commit that actually introduced the bug needs a test or semantic
analysis, which is the role of the AI step.

### 7.2 Conflict handling and the `resolve` command

Cherry-picks split into three outcomes. A **clean** apply opens a normal backport pull
request. A conflict confined to **test or generated files only**, where the source fix
applied cleanly and only a test hunk clashed, is auto-resolved: the branch keeps its
own tests, the source fix is committed, and the pull request notes it. A **real source
conflict** is aborted, so nothing is committed and no half-applied branch is left
behind, and reported in the summary, because deciding whether the vulnerability still
applies and how to adapt the fix requires human judgement.

Those real conflicts are then handled by an interactive `resolve` command rather than
left entirely to the engineer. Given the same fix, `resolve` targets exactly the
branches that conflicted and, for each, cherry-picks the fix and lets the engineer
resolve it in place, by default in their own checkout so their editor shows the
conflict live. `git rerere` is enabled, so a resolution recorded on one branch is
auto-applied to an identical conflict on a sibling branch, such as the FIPS twin
branches, and surfaced for verification rather than committed silently. `rerere`
matches conflicts byte for byte, so it reuses most on the near-identical twins;
branches that diverged differently present different conflicts and are resolved
individually.

### 7.3 Why a local CLI

Both solutions are packaged as a local command-line tool that runs against an AWS-LC
clone, rather than as a GitHub Action or a hosted service:

- No infrastructure and no standing automation. There are no hosted runners to manage
  and no long-lived bot identity or repository-wide permission to grant.
- It fits the embargoed-fix case naturally. The same tool can be pointed at a private
  fix commit in an internal clone to produce an analysis-only report before anything
  is public.
- It is easy to run ad hoc and easy to inspect, since everything happens in a local
  clone.
- It can still be wrapped in CI later if standing automation is wanted, which would
  reintroduce the credential question in Section 6.

### 7.4 Out of scope: further AI uses

The recommended solution only uses AI for impact analysis. Two further uses were
considered and are recorded here with their tradeoffs. The same guardrails would
apply to both, which is why they remain reasonable future work: no write access, no
commands run, nothing auto-applied or auto-merged, and a human reviews every result.

**AI conflict resolution.** When a cherry-pick fails, the model would be given the
conflict, the patch, and the conflicting files, and would propose a per-file
resolution. This targets the step that takes the most engineer time today and stays
advisory. The downside is that conflict resolution on security fixes is high-stakes,
so a wrong but plausible suggestion could mislead, and it widens the LLM threat
surface beyond read-only analysis.

**AI test generation.** The model would generate a test that exercises the bug, which
would make the test-based impact method from Section 2.2 practical to run
automatically. That makes the more reliable method scalable and is useful well beyond
backporting, but it needs a sandboxed execution environment that is harder to build
safely, and generated tests can be wrong or incomplete.

In general, AI closes real gaps in exactly the places a human currently has to step
in, and it also adds non-determinism and a security surface the deterministic pipeline
does not have. The recommended solution accepts that tradeoff only for impact
analysis, where the output is read-only and contained.

### 7.5 References

Manual-backport examples that motivated this project: aws-lc pull requests
[#3105](https://github.com/aws/aws-lc/pull/3105),
[#3106](https://github.com/aws/aws-lc/pull/3106),
[#3107](https://github.com/aws/aws-lc/pull/3107),
[#3108](https://github.com/aws/aws-lc/pull/3108),
[#3109](https://github.com/aws/aws-lc/pull/3109),
[#3110](https://github.com/aws/aws-lc/pull/3110),
[#3270](https://github.com/aws/aws-lc/pull/3270),
[#3272](https://github.com/aws/aws-lc/pull/3272),
[#3273](https://github.com/aws/aws-lc/pull/3273).

## What changed on the way to shipping

Four things in this document did not survive contact with the real repository. Each
is worth reading against the section above it.

**The AI no longer runs on every branch.** Section 5.1 argues for an always-on
auditor, on the grounds that a fallback cannot see false positives on the confident
path. That argument still holds, but the shipped tool narrows it: the AI runs on
branches git history cannot settle, plus a second look at branches that match only
part of a fix's history. The partial-match case is where the over-flags actually
turned out to live, so the auditor keeps its reach at a fraction of the model calls.

**The verdict is a forced tool schema, not prose.** Nothing in this document says how
the model's answer is read back, and every early version read it out of Markdown.
Every bug in that layer was a parsing bug, including a reasoning sentence beginning
with "No" clearing a branch. The shipped tool gives the model one tool,
`record_verdict`, forces it with `tool_choice`, and validates what comes back. A reply
that is unreadable, missing, or cut short by the token limit counts as no answer, and
no answer leaves the branch flagged.

**Test-only conflicts are no longer auto-resolved.** Section 7.2 auto-resolves a
conflict confined to test files. The shipped `resolve` stages nothing on the
engineer's behalf, because `git add -A` before continuing a cherry-pick takes an
arbitrary side of a delete conflict and can drop half the fix while looking
completely clean. Two independent checks now run before a pick is finished: git's own
unmerged list, and a scan for leftover conflict markers in staged files. Neither
finds what the other does.

**It grew a CI half after all.** Section 7.3 argues the deliverable is the CLI and
not a workflow. That is still the right default, and the tool is still local-first.
But a merged pull request labelled `needs-backport` is a natural trigger, so the
workflow exists, split into two jobs: the one that reaches the model has read-only
access to the repository, and the one that can write never reaches the model. The
verdict travels between them as an artifact. That split is the answer to the standing
credential question this section deferred.

# AWS-LC Backport

Works out which supported release branches a security fix still needs to reach,
before it merges, and carries it there.

This is the working repository behind [`util/backport`](https://github.com/aws/aws-lc/tree/main/util/backport)
in [aws/aws-lc](https://github.com/aws/aws-lc): the tool as it shipped, the design
record it came out of, and the synthetic multi-branch fixture it was first built
against.

## The problem

aws-lc keeps several long-lived FIPS and LTS release branches alongside `main`.
When a security fix lands, it has to be backported to every older branch that
**still carries the buggy code**, but not to branches that never had it, and not to
branches that already got it. Deciding that by hand means reading git history per
branch, and the work scales with fixes multiplied by branches. The branch count only
grows.

The interesting part is not the cherry-picking. It is deciding, for a branch whose
surrounding code has moved on by four years, whether the vulnerability is still
there.

## How it decides

Two passes, deterministic first.

**Git history.** Find the distinctive lines the fix deletes, blame the commits that
wrote them, and check whether those commits and those lines reached each branch.
Comments, whitespace and boilerplate are filtered out first, so a stale comment
cannot trace back to an ancient import and drag in the wrong branch. This settles
most branches on its own, and it settles them identically every run.

**AI.** Only for the branches history cannot settle, plus a second look at branches
that match part of a fix's history. It answers through a forced tool schema rather
than in prose, and a no-answer always leaves the branch flagged.

The bias is deliberate and one-directional. The only confident *not affected* is
"the buggy code is genuinely not here." Anything ambiguous becomes AFFECTED for
review. The tool can cost a reviewer a few minutes; it cannot silently drop a needed
security backport.

Then `apply` cherry-picks onto one local branch per affected branch, and `publish`
turns those into one pull request each. Nothing is auto-merged and nothing is a
draft.

## Results

The replay bench winds a throwaway sandbox back to just before each of 39 real
aws-lc fixes landed, so the tool cannot see its own answer, and grades it against a
hand-verified key. That is 157 fix by branch cells:

|                      | git history only | with the AI pass |
| ---                  | ---              | ---              |
| unneeded flags       | 25               | 4                |
| correctly cleared    | 30               | 51               |
| **missed backports** | **0**            | **0**            |
| agreement            | 84%              | 97%              |

Missed backports staying at 0 in both columns is the property worth protecting. A
separate 500-cell replay, built from evidence mined out of the repository rather
than from the answer key the tool was tuned on, also found 0. The methodology and
the caveats are in [docs/validation.md](docs/validation.md).

## What shipped

| PR | What | State |
| --- | --- | --- |
| [#3389](https://github.com/aws/aws-lc/pull/3389) | `analyze`: the engine, the AI pass, the FIPS boundary check | merged |
| [#3414](https://github.com/aws/aws-lc/pull/3414) | `apply`: cherry-pick per affected branch, in isolated worktrees | merged |
| [#3395](https://github.com/aws/aws-lc/pull/3395) | shared Bedrock model config between this tool and autofix | merged |
| [#3415](https://github.com/aws/aws-lc/pull/3415) | `publish`: one pull request per branch | in review |
| [#3416](https://github.com/aws/aws-lc/pull/3416) | the GitHub Actions half, and its OIDC role in CDK | in review |
| [#3417](https://github.com/aws/aws-lc/pull/3417) | `resolve`: guided conflict resolution | in review |

The copy here includes `publish` and the workflow, which are still in review.
`resolve` is not in it, since that PR is open against a later base.

## Layout

```
util/backport/          the tool, at the path it lives at upstream
  backport              entry point
  src/engine/           the verdict: which lines, which commits, which branches
  src/commands/         analyze, apply, publish
  testing/              unit tests, and the replay bench with its answer key
  README.md             the full command reference
ci/backport-bot.yml     the CI half, as a reference copy
docs/design.md          the design proposal, as reviewed
docs/validation.md      how it was tested, and what the numbers mean
docs/engineering-notes.md   the bugs that shaped the design
fixture/                the synthetic multi-branch repo it was first built against
```

`util/backport/` sits at the path it occupies in aws-lc on purpose. The tool pins
itself to the checkout two levels above itself, so this directory is a drop-in copy
rather than a fork of it.

The workflow is kept at `ci/backport-bot.yml` rather than under `.github/workflows/`
so that it does not try to run here. It assumes an OIDC role that only exists in the
aws-lc account.

## Running it

The tool operates on the checkout it lives in, so to run it for real, copy
`util/backport/` into an aws-lc checkout at the same path:

```bash
git clone https://github.com/aws/aws-lc.git
cp -R util/backport aws-lc/util/backport
cd aws-lc
git fetch origin                       # the release branches have to be present

util/backport/backport analyze --commit ac3aee310
util/backport/backport apply
```

The AI pass calls Claude on Amazon Bedrock and needs credentials for it. The
git-history pass alone runs with `BACKPORT_DISABLE_AI=1`. Full setup, every flag,
and troubleshooting are in [util/backport/README.md](util/backport/README.md).

The unit tests need neither a checkout nor credentials:

```bash
cd util/backport && python3 -m unittest testing.test_engine
```

## Notes

Built during an internship on the Amazon cryptographic libraries team, May to
August 2026. The synthetic fixture in `fixture/` came first, before the tool was
pointed at real history; `docs/validation.md` covers why it was replaced rather than
extended.

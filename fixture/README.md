# The synthetic fixture

A tiny C "crypto library" with a branch topology shaped like aws-lc's. This is what
the engine was built against before it was pointed at real history. It is kept for the
record, and because the failure mode it exposed is the reason the AI pass exists. See
[docs/validation.md](../docs/validation.md) for what it did and did not prove.

The interesting content is not the C. It is the commit graph the scripts build.

## What it contains

Seven long-lived branches: `main`, `NetOS` as the one-off, and `AWS-LC-FIPS-2020`
through `AWS-LC-FIPS-2025`. Each is cut at a different point in a history that grows
files over time, so an older branch genuinely lacks code that a newer one has, which is
the property the whole tool turns on.

Then a set of tagged fixes on `main`, each one a case the engine has to get right:

| tag | the case |
| --- | --- |
| `cve-buffer` | fix in the oldest file, so every branch has the code |
| `cve-handshake-postrefactor` | fix in a file that was moved after some branches were cut |
| `cve-record-multifile` | one fix spanning two files |
| `cve-pure-modification` | changes existing lines, adding and removing nothing |
| `cve-pure-deletion` | removes the vulnerable code and adds nothing |
| `cve-cross-era` | touches the oldest file and a recent one in the same fix |

The graph also plants the situations that break naive checks: a refactor that renames
files partway through, a branch that diverged with its own commits, and a branch that
already cherry-picked one of the fixes, so it should come back as already patched rather
than flagged.

`app.c`, `crypto/`, `tls/` and `utils/` are the working tree of whatever state the
scripts last left the repository in. They only mean anything in combination with the
history around them.

## The scripts rewrite history

`scripts/setup_test_fixture_v2.sh` builds the graph.
`setup_test_fixture_v3_extensions.sh` adds the last three cases and has to run after it.

Both wipe local refs, rebuild every branch and force push. They belong to a throwaway
test repository and nothing else. `v2` refuses to run unless `origin` looks like this
repository, which is a guard, not a guarantee.

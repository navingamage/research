# Top 3 hardest picks

These are the three ideas most likely to still trip up a strong run. I picked them after the 2026-11-04 pre-checks, where 9 hard problems were all solved in one go ([past-attempts.md](../past-attempts.md#max-difficulty-pre-checks-2026-11-04)).

| # | Task | Repo @ pin | Type | Why it's hard |
|---|---|---|---|---|
| 1 | [Loan prepayment penalty, wider scope](fineract-prepayment-penalty/) | apache/fineract @ `8a61e30a` | Feature | The 10-requirement version was finished in 96 min, but reading the result turned up about 20 gaps its tests didn't catch: the payoff quote for every caller, 1 of 30 replay paths hooked, undo disbursal, foreclosure. The wider scope covers those cases. |
| 2 | [Early-payment discount ("2/10 net 30")](ofbiz-early-payment-discount/) | apache/ofbiz-framework @ `f05a0c22` | Feature | The discount has to close an invoice with no cash moving. It goes through two different PAID checks and an apply path that triggers nothing, and failed postings drop silently into `ERROR_JOURNAL`. It has to work for both AR and AP. |
| 3 | [Review of PR #7931](typst-pr7931-review/) | typst/typst @ `1f0cd84a` (base `c98e9103`) | Review | The PR re-records 96 renders and 221 output hashes, so only one test fails. 2 of the 6 problems from the PR thread are in exporters the diff never touches. |

## What's in each folder
- `codebase.md` covers how the codebase works around the change, with `file:line` references at the pin. It also has build and test commands, repo conventions, known traps, and a list of what couldn't be confirmed by reading.
- `sections.md` has facts, numbers and open choices for the four write-ups: the ask, the goal and success checks, the decision points, and difficulty and codebase understanding. It's fragments only, and the write-ups themselves get written separately.

## How the notes were checked
- Everything comes from reading the code at the pin. Nothing was built or run for these notes. The one exception is section 17 of the Fineract notes, which records the earlier 96-minute run.
- Every path and line reference was checked against the pinned tree: about 2,100 for Fineract, 1,150 for OFBiz and 860 for Typst.
- Anything traced by reading but not run is marked "by reading", "not run" or "(unverified)".
- Re-checked on 2026-11-04:
  - Fineract `develop` at `06ce07b5` still has no prepayment charge time.
  - OFBiz trunk at `8cfe7b17` still never reads `FIN_PAYMENT_DISC`.
  - The Typst PR head is still `1f0cd84a`.

## Before a run
- **Fineract**
  - Integration tests need Postgres or MariaDB in Docker plus a running server (`codebase.md` §15).
  - The build version comes from git tags, so the clone needs its tags.
  - The next free charge time id at the pin is 17, which upstream later gave to `DOWN_PAYMENT`.
- **OFBiz**
  - Run `gradle/init-gradle-wrapper.sh` first, since the wrapper jar isn't committed.
  - The build needs packages.atlassian.com and artifacts.alfresco.com (for one dependency each), plus the Gradle plugin portal and npm on a cold cache. The first build confirms they're reachable.
  - No accounting test baseline exists at the pin yet.
- **Typst**
  - Fetch `pull/7931/head` and put `main` at `c98e9103`.
  - `cargo testit --base-revision c98e9103` compares against the base refs, so every re-recorded change shows up again.
  - `issue-6242-tight-list-attach-spacing` is expected to fail at the head (not run).

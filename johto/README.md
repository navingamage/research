# Johto: task ideas and repo research

Research notes for coding tasks set in real open-source applications. Every idea here targets **domain correctness that the repo doesn't spell out**, such as money, rounding, dates, reversals, timing and existing behaviour that has to keep working. Those are the places a strong model still gets things wrong.

## Ideas

| Idea | Repo @ pin | Size | Status |
|---|---|---|---|
| [Capital gains across stock splits](ideas/paisa-stock-splits.md) | ananthakumaran/paisa @ `18ecd7d5` | Personal-scale | Full run done |
| [Ask before a tool runs](ideas/oterm-tool-approval.md) | ggozad/oterm @ `de8604c6` | Personal-scale | Ready, next 3 #1, full notes in [next3/](next3/oterm-tool-approval/); shell_gpt is the backup |
| [Restarts and upgrades during a send](next3/listmonk-safe-restarts/) | knadh/listmonk @ `30846f84` | Personal-scale | Ready, next 3 #2 (operations, app code unchanged) |
| [Sleep timer for the player](next3/jellyfin-web-sleep-timer/) | jellyfin/jellyfin-web @ `0a580ce8` | Enterprise | Ready, next 3 #3 |
| [Tracked time per day and week on the list page](ideas/traggo-daily-totals.md) | traggo/server @ `8ca9bb6a` | Personal-scale | Vetted 2026-11-06, frontend candidate; fava is the backup |
| [Uneven bill splits](ideas/ihatemoney-uneven-splits.md) | spiral-project/ihatemoney @ `e66a7672` | Personal-scale | Backup (likely too easy) |
| [Loan prepayment penalty](ideas/fineract-prepayment-penalty.md) | apache/fineract @ `8a61e30a` | Enterprise | Picked, top 3 #1. A 10-requirement version was finished in 96 min (2026-11-04), so it needs a wider scope |
| [Early-payment discount](ideas/ofbiz-early-payment-discount.md) | apache/ofbiz-framework @ `f05a0c22` | Enterprise | Ready, top 3 #2 |
| [Order instalment payments](ideas/pretix-instalments.md) | pretix/pretix @ `e6f8f178` | Enterprise | Ready |
| [Review of an open layout PR](ideas/typst-pr7931-review.md) | typst/typst PR #7931 @ `1f0cd84a` (base `c98e9103`) | Enterprise | Not run yet, top 3 #3 |
| [Minimum invoice amount](past-attempts.md#kill-bill-minimum-invoice-amount) | killbill/killbill @ `52e2f55f`, rerun on `cb60779c` | Enterprise | Rejected |

- [top3/](top3/) has full codebase notes for the three hardest picks, plus the facts for each write-up.
- [next3/](next3/) has the same notes for the next three: oterm, listmonk and jellyfin-web.
- [repo-vetting.md](repo-vetting.md) has every repo checked, the checks run, and why each one was kept or rejected.
- [past-attempts.md](past-attempts.md) covers what happened when ideas were tried, including the ones that turned out too easy (9 more on 2026-11-04).

## What makes an idea worth running
- A real application, not a library or framework, pinned to a full commit SHA.
- No AI-assistant files (`CLAUDE.md`, `AGENTS.md`, `.claude/`, `.cursor`, Copilot instructions) and no AI co-authored commits up to the pin. If they appear later, pin to a commit before they started.
- It builds from public sources only: parent POMs, plugins and packages on Maven Central, the Gradle plugin portal, PyPI, npm or Go modules, with no logins.
- The feature is missing at the pin **and** upstream, and it isn't partly covered by something else (a setting, a plugin, or a catalog option).
- The difficulty is in the domain, not in following a public spec. Well-specified features in clean modern repos got finished in about 10 minutes.
- Don't reuse a repo for two ideas in a row, so the ideas stay distinct.

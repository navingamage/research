# Johto: task ideas and repo research

Research notes for coding tasks set in real open-source applications. Every idea here targets **domain correctness that the repo doesn't spell out**, such as money, rounding, dates, reversals and existing behaviour that has to keep working. Those are the places a strong model still gets things wrong.

## Ideas

| Idea | Repo @ pin | Size | Status |
|---|---|---|---|
| [Capital gains across stock splits](ideas/paisa-stock-splits.md) | ananthakumaran/paisa @ `18ecd7d5` | Personal-scale | Full run done |
| [Uneven bill splits](ideas/ihatemoney-uneven-splits.md) | spiral-project/ihatemoney @ `e66a7672` | Personal-scale | Backup (likely too easy) |
| [Loan prepayment penalty](ideas/fineract-prepayment-penalty.md) | apache/fineract @ `8a61e30a` | Enterprise | Ready |
| [Early-payment discount](ideas/ofbiz-early-payment-discount.md) | apache/ofbiz-framework @ `f05a0c22` | Enterprise | Ready |
| [Order instalment payments](ideas/pretix-instalments.md) | pretix/pretix @ `e6f8f178` | Enterprise | Ready |
| [Minimum invoice amount](past-attempts.md#kill-bill-minimum-invoice-amount) | killbill/killbill @ `52e2f55f` | Enterprise | Submitted |

- [repo-vetting.md](repo-vetting.md) has every repo checked, the checks run, and why each one was kept or rejected.
- [past-attempts.md](past-attempts.md) covers what happened when ideas were tried, including the ones that turned out too easy.

## What makes an idea worth running
- A real application, not a library or framework, pinned to a full commit SHA.
- No AI-assistant files (`CLAUDE.md`, `AGENTS.md`, `.claude/`, `.cursor`, Copilot instructions) and no AI co-authored commits up to the pin. If they appear later, pin to a commit before they started.
- It builds from public sources only: parent POMs, plugins and packages on Maven Central, the Gradle plugin portal or PyPI, with no logins.
- The feature is missing at the pin **and** upstream, and it isn't partly covered by something else (a setting, a plugin, or a catalog option).
- The difficulty is in the domain, not in following a public spec. Well-specified features in clean modern repos got finished in about 10 minutes.
- Don't reuse a repo for two ideas in a row, so the ideas stay distinct.

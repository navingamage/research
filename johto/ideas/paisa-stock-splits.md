# Paisa: capital gains across stock splits

- **Repo:** https://github.com/ananthakumaran/paisa/tree/18ecd7d502c91cd5695d18ee42ac51d35919a6dd
- **Stack:** Go 1.24 (gin, gorm/SQLite) backend, Svelte frontend, reads ledger / hledger / beancount journals
- **Domain:** Personal investing: tax lots, holding periods, capital gains tax (personal-scale codebase)
- **Status:** active. It replaces ihatemoney as the stronger personal-scale idea.

## Pre-flight (checked 2026-11-01)
- It's a real application: a self-hosted personal finance and investment tracker with a REST API and a web UI.
- **Size:** 93 Go files and 159 Svelte/TS files. The rest are fonts, docs and fixtures.
- **Tests:** 6 Go test files, plus a regression suite (`tests/regression.test.ts`) that replays fixture journals through the API and compares the JSON.
- **Clean history:** single maintainer, AGPL. No pricing in the repo or docs. Not one AI co-authored commit in its history, and no AI files.
- **Builds from public sources:** Go modules and npm. Tests need the `ledger`, `hledger` and `beancount` binaries, which all install from the distro or pip.
- **Pin:** HEAD of main, 2026-10-21.

## Gap
- Commit `873c3c1` ("handle stock split", 2023) added `service.IsStockSplit`. It only treats a transaction as a split when every posting is the same non-currency account, and it only filters split postings out of the **investment**, **networth** and **assets balance** views.
- The **capital gains** FIFO (`internal/server/capital_gain.go`) and **tax harvesting** (`internal/server/harvest.go`, which goes through `accounting.FIFO`) never check for splits, so:
  - the split's "remove old units" posting shows up as a **sale**, which pulls old lots out of the FIFO queue and adds a fake sale to that financial year's units and sale value
  - the "add new units" posting becomes a **new lot dated on the split date**, so the holding period starts again and long-term gains turn into short-term ones
  - lots bought before 2018-02-01 lose grandfathering
- No regression fixture covers capital gains or harvest at all. The fixtures don't include `capital_gain.json`.

## Where a solution can go wrong
- **Lots have to be restated, not dropped:**
  - units × ratio, cost per unit ÷ ratio
  - total cost unchanged
  - **original purchase dates kept**
  - FIFO order kept when buys and sells happen before and after the split
- **The easy wrong fix:** filtering split postings out of the FIFO, the way the investment view does. That leaves the old lots in pre-split units, so units don't add up and later sales run past what's available.
- **Grandfathering:** `taxation.Calculate` looks up the price on `EQUITY_GRANDFATHER_DATE`. That price is in pre-split units for any split after 2018, so it has to be scaled.
- **Price history** before the split is in old units. That affects the harvest page's "current price vs lot cost", and anything else that compares lot prices with market prices.
- **Edge cases:**
  - reverse splits (consolidation), and fractional units that come out of a ratio and get paid out as cash
  - several splits on the same commodity
  - the same commodity held in several accounts
  - a split in the same year as a sale
- **Three journal back ends:** ledger uses `lot_price`, hledger and beancount use `cost(position)`. Each one shows a split differently, and the right answer has to hold for all three.
- **Detection:** `IsStockSplit` is a heuristic. A split entered with a balancing equity posting, or in a different shape, gets missed.
- **Tax categories:** Equity, Equity65, Equity35, Debt and UnlistedEquity each have their own holding-period thresholds and indexation (CII). A holding period that resets wrongly changes which rule applies.
- **The rest of the app has to keep agreeing:** XIRR (`8942670` fixed capital gains in XIRR), gain, portfolio, and the doctor/diagnosis checks.
- **Existing regression fixtures** for every back end have to stay byte-for-byte identical when the journal has no splits.

## Verifying in a lane
- Build with `go build ./...`, then run `go test ./...`. Install `ledger`, `hledger` and `beancount` (apt/pip), and node, for `tests/regression.test.ts`.
- Expect the model to need its own fixture journal with a split, plus expected JSON for `/api/capital_gains` and `/api/harvest`. Neither exists yet.

## Check run (2026-11-02)
- One turn with no follow-ups. About 6.5 min and 40 tool calls, with a 13 KB diff across 5 files.
- **What it built:**
  - `ApplyStockSplit` in `internal/accounting/accounting.go` rescales lot units by (units after ÷ units before) and keeps each lot's cost and purchase date. The last lot absorbs rounding.
  - It handles both ways of writing a split ("remove old + add new" and "add only the extra units"), plus reverse splits.
  - `FIFO` is unchanged, and a new `FIFOWithStockSplit` sits beside it. `capital_gain.go` and `harvest.go` apply the split instead of treating it as a sale. Detection reuses `service.IsStockSplit`.
  - Unit tests and an in-memory DB server test. It **checked that the tests fail on the old code**. `go vet` and `gofmt` are clean.
- **Gaps:**
  1. **Grandfathering:** the 2018-01-31 cost price is still in pre-split units. The model said so and left it.
  2. **Splits with cash** (fractional shares paid out) aren't detected, because `IsStockSplit` rejects any transaction with a currency posting. Those quietly keep the old fake-sale behaviour. The model didn't mention this.
  3. **No end-to-end run:** it never installed `ledger`, `hledger` or `beancount`, never ran a real journal, and never ran the regression suite.
  4. **Narrow detection:** it only catches a split recorded in the exact shape the heuristic expects (same account, no other postings).
- **Outcome:** not finished in one go, but the core is correct. Next step is a full run, pushing only on gaps 1–3.

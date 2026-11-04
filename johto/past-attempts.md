# Past attempts

Every run used Claude Code with Opus 5.5 at High effort, inside a clean Docker lane at the pinned commit.

## Healthchecks: maintenance windows
- Repo: healthchecks/healthchecks @ `f6c46b594dec66c92bca7dd56db58fbd1e4ce9fe` (Django)
- Idea: plan quiet periods ahead of time, so checks that are expected to be down during patch or deploy nights don't page anyone.
- Result: **too easy.** One turn of about 9 min with 65 tool calls. It built a working feature with 34 new tests and ran the full suite (1,808 tests, all green, plus mypy). The only weak spots were a race at the end of a window and tag selection being fixed at the moment a window is created.
- Lesson: a well-specified feature in a clean, modern Django codebase gets done properly in one go.

## Miniflux: WebSub push
- Repo: miniflux/v2 @ `faecfa2fac61a91f4f69550bb1afd506c4b51749` (Go, Postgres)
- Idea: real-time feed updates through WebSub hubs instead of only polling on a schedule.
- Result: **too easy.** One turn of about 13 min with 59 tool calls and 24 files changed. It handled hub discovery, one shared subscription per feed (with an atomic claim tested under 20 concurrent requests), the HMAC signature check in constant time, leases and renewal, and protection against hub URLs pointing at private networks. Polling stayed as a fallback, and `go test -race ./...` was green. It also found and fixed its own lease-renewal bug.
- Lesson: following a public protocol spec is exactly what the model does best.

## Kill Bill: minimum invoice amount
- Repo: killbill/killbill @ `52e2f55f103baf0703ddee71e10a7036595fc851` (Java 21, Maven)
- Idea: stop collecting payment on tiny invoices that cost more to collect than they're worth, without losing the money.
- **Check run** (one turn, about 3.5 min): it only skipped the automatic charge on the payment side. It never compiled anything (it claimed there was no JDK, though SDKMAN was available), never rolled the small amount forward, left those invoices to go overdue, and used one threshold for every currency.
- **Full run** (about 83 min, 9 follow-ups): with pushback it moved to the invoice side and carried small balances forward. It also skipped overdue for deferred invoices and added per-currency and per-tenant thresholds. When pushed on customer impact it found 4 bugs in its own work: overdue counting money that was never attempted, refunded or charged-back amounts being charged again, partial-payment leftovers, and a wrong dry-run total. **It never got a single build or test to run.**
- Build blocker: this pin needs `killbill-oss-parent 0.147.36`, which only exists on Kill Bill's login-protected Cloudsmith repo. Maven Central stops at 0.147.21. The last pin that builds from public sources is `cb60779c` (parent 0.147.20).
- **Review 1 (2026-11-02): sent back.** The goal asked for more than the prompt did. The prompt read as "don't bother with tiny invoices", so neither collecting them later nor a per-account on/off switch followed from it. The reviewer asked for a new run with a prompt that says a minimum gets collected right away.
- **Rerun plan:** re-pin to `cb60779c171391be558cd7aebb1eafea60ad2b82` so it builds from public sources, use a new prompt, and write a goal that only asks for what that prompt implies.
- **Rerun check run** (2026-11-02, `cb60779c`, about 15 min, 67 tool calls, one clarifying question): built a per-currency minimum on the payment side. Held invoices stay open and are excluded from overdue, and the next bill charges everything as one payment, with the excess turned into account credit. **Misses:**
  - it changes core behaviour for every account (any overpayment becomes credit, even with no minimum set)
  - held amounts on a cancelled account are never collected
  - it never installed Java, so nothing was compiled or tested
  - its own recommended default was "leave it unpaid", which lets the account go overdue
- **Rerun full run** (2026-11-03, about 45 min, 3 substantive follow-ups): moved to an invoice-side carry-forward after the clarifying answer. It only added the cancellation flush after pushback, and only built and tested after being asked (Docker with a Maven image). Full build and about 930 module tests pass, and 51 targeted end-to-end tests pass. **Left open:**
  - the full end-to-end regression, and MySQL/PostgreSQL
  - invoice plugins skipped on the final invoice
  - the upcoming-invoice preview doesn't show carried amounts
- **Final outcome (2026-11-04): rejected.** The reviewer read the result as close to complete (3 or 4, not 2). The success criteria also still graded behaviour the short prompt never asked for: when held amounts get collected, flushing them on cancellation, credits, and overdue.
- **Lesson:** with a short casual prompt, only grade what the prompt says. Any rule that has to be inferred (when to collect, edge cases) gets read as a new requirement. Hidden domain rules need a prompt that states them.
- Lesson: domain correctness in a large, older Java codebase is where the model struggles. Before running anything, check that the pin builds from public repositories.

## Max-difficulty pre-checks (2026-11-04)
Sandboxed one-turn runs (Opus 5.5, High), not lane runs. Each workspace was the pinned tree with toolchains preinstalled. Hidden variants or a trap checklist were written before each run.

| Repo @ pin | Problem | Time | Result |
|---|---|---|---|
| itchyny/gojq @ `e7c14ec` | keep object key order like jq | 30 min | 1,740 differential cases identical to jq |
| postgres/postgres @ `b6178c94` | CREATE INDEX CONCURRENTLY on partitioned tables | 34 min | passed concurrent pgbench writes and amcheck |
| binji/binjgb @ `1662111` | mode 3 timing (15 mooneye failures) | 32 min | all test ROMs pass |
| OpenTTD/OpenTTD @ `611118d70` | #8424 trains crash through the back of a depot | 19 min | wrote a Squirrel AI to build the layout in 4 orientations; fixed in the collision check, not the radius change maintainers rejected; ctest 98/98 |
| idris-lang/Idris2 @ `708075dc` | #3800 coverage checker accepts a proof of Void | 24 min | also fixed all 7 of 15 hidden variants the clean compiler gets wrong |
| typst/typst @ `84cca406` | #5603 + #5314 footnote on a different page than its marker | 53 min | splits the breakable block above the marker line; 3,826 tests pass; 431-footnote stress test with none separated |
| pimutils/khal @ `3ba0fcef` | edit only this / this and following for recurring events | 19 min | RFC 5545 details right (UTC UNTIL, COUNT split, all-day RECURRENCE-ID, re-editing an override); fixed a hidden data-loss bug in edit-all; missed updating the weekday rule on a weekday move (disclosed) |
| beancount/fava @ `12d6fea6` | tick transactions, live sum, bulk tags/metadata (#2147) | 22 min | raw amounts instead of locale-formatted text, bottom-up writes for several entries in one file, all-or-nothing on plugin entries; UI placed where the maintainer suggested |
| apache/fineract @ `8a61e30a` | prepayment penalty from a ~700-word, 10-requirement spec | 96 min to green, 110 min total | 19 new integration tests green on a live server and Postgres; 1,140 unit and 184 related integration tests green; caught 2 of its own bugs |

- **Lesson:** a clear target plus any way to check it (a repro, a reference tool, a test suite, a spec) gets solved, whatever the language or codebase age. Size and slow verification only stretched the time: Fineract was the only run past 90 minutes.


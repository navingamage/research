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
- Lesson: domain correctness in a large, older Java codebase is where the model struggles. Before running anything, check that the pin builds from public repositories.

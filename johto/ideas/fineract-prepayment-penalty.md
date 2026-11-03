# Apache Fineract: loan prepayment penalty

- **Repo:** https://github.com/apache/fineract/tree/8a61e30a4521c77a59bc55f598e14c128ffd183a
- **Stack:** Java 21, Gradle
- **Domain:** Core banking and lending
- **Status:** ready

**Pre-flight (checked)**
- Pin 2026-02-14, just before Claude co-authored commits started (2026-02-17). `AGENTS.md` and `.claude/` weren't added until 2026-07-05. Zero AI commits up to the pin.
- Repositories are only Maven Central and the Gradle plugin portal. Both are public.
- 4,736 main and 930 test Java files, 266 integration tests. A real core-banking server (Apache, behind Mifos), which makes it Enterprise-sized.
- Gap: `ChargeTimeType` has no prepayment or early-repayment type, at the pin or at HEAD. No commit adds one. `EarlyPaymentLoanRepaymentScheduleTransactionProcessor` only sets allocation order and charges no fee.
- Rejected on the way: balloon loans (partly there through fixed-principal %, FINERACT-1348 and 2789).

**Facts about the gap**
- Loan charge times are disbursement, specified due date, instalment fee, overdue, and tranche. Nothing is tied to paying early.
- The prepay and foreclosure paths exist: `LoanTransactionsApiResource` `prepayLoan` template, and the foreclosure validators and events.
- Penalty income already has its own GL mapping (`INCOME_FROM_PENALTIES`).

**Where a solution can go wrong**
- What counts as a prepayment: paying principal ahead of schedule, versus paying amounts that are due or overdue. Partial prepayment versus full payoff.
- The base and the limits: a percent of the prepaid principal or of the outstanding principal, min and max caps, and a lock-in window (only in the first N months).
- Exclusions: the final scheduled instalment, reschedule or refinance, write-off and charge-off.
- About 10 transaction processors plus the advanced payment allocation. Allocation order decides whether the penalty takes money meant for principal.
- Backdated transactions replay the loan's whole history, so the penalty has to come out the same on every replay. Undoing a repayment must remove its penalty.
- With interest recalculation, a prepayment changes future interest and the schedule, so the order of the calculation matters.
- The payoff and prepay template amount must include the penalty.
- Accounting entries under both cash and accrual, plus COB accruals.
- There are two schedule engines (cumulative and progressive), and both have to work.

**Verification in the lane:** Gradle compile takes about 10 min. Unit tests in `fineract-loan` and `fineract-progressive-loan` need no DB. Integration tests need Postgres or MariaDB in Docker and are slow, so only part of them gets run in a time-boxed run.
- Small risk: the version comes from git tags (`me.qoomon.git-versioning`), and it falls back if the lane clone has no tags. Watch the first build.

## Re-check (2026-11-03)
- **Gap still open at upstream HEAD `71281f17` (2026-11-03):**
  - no prepayment or early-repayment `ChargeTimeType`
  - no code matches prepayment, early-repayment or foreclosure fee identifiers
  - the 2,535 commits since the pin only fix prepay amounts and allocation
- **At the pin:** 0 AI co-author trailers, and no AI files. Build sources are Maven Central and the Gradle plugin portal only.
- **Size:** 5,667 Java files (930 tests) and 67 Cucumber feature files.
- **Scope a full solution has to cover:**
  - 11 repayment transaction processors
  - 2 schedule engines (cumulative and progressive)
  - replaying backdated transactions
  - undoing and adjusting repayments
  - the prepay and foreclosure templates
  - four accounting modes
- **Closest existing pattern:** the `OVERDUE_INSTALLMENT` charge, used across about 21 files.

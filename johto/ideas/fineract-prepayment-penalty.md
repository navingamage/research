# Apache Fineract: loan prepayment penalty

- **Repo:** https://github.com/apache/fineract/tree/8a61e30a4521c77a59bc55f598e14c128ffd183a
- **Stack:** Java 21, Gradle
- **Domain:** Core banking and lending
- **Status:** picked (2026-11-03) for a written-spec, unattended run. A 10-requirement version was finished in 96 min on 2026-11-04, so the next run needs the wider scope.
- **Full notes:** [codebase notes and facts for the write-ups](../top3/fineract-prepayment-penalty/) (checked against the pin on 2026-11-04)

## Background: what Fineract is
- Apache Fineract is an open-source core-banking system. It started as Mifos (Grameen-style microfinance) and became an Apache top-level project in 2017.
- **Users:** microfinance institutions, credit unions, small banks and fintechs, often as the ledger behind their own apps. Companies sell hosting and support around it.
- **Shape:**
  - a multi-tenant REST API server (Spring Boot, MySQL, MariaDB or Postgres, Liquibase) with no UI of its own
  - covers clients, groups and centres, loans, savings, deposits, shares, charges and a full general ledger
  - a nightly close-of-business (COB) batch handles accruals and overdue
  - business events go out over Kafka or ActiveMQ

### How a loan works
- **Product and account:** a loan **product** is the template (rates, terms, charges, accounting rules). A loan **account** is created from it, then approved and disbursed.
- **Schedule:** disbursement builds a repayment schedule with one of two engines. **Cumulative** is the classic one. **Progressive** is the newer EMI-style one, which recalculates as payments arrive.
- **Allocation:** every repayment runs through a transaction processing strategy, which sets the allocation order (for example penalties, then fees, then interest, then principal). There are 9 of them, plus an advanced, configurable allocation, so 10 in all.
- **Charges** attach at fixed trigger points: disbursement, a specified date, per instalment, overdue, or per tranche.
- **Backdated activity replays the whole history:** a payment entered for an earlier date, or an undone payment, re-runs every transaction from the start. Anything computed during processing must come out the same on every replay. (The latest payment on a loan without interest recalculation is processed on its own.)
- **Accounting** posts journal entries in one of four modes: none, cash, periodic accrual or upfront accrual.

### The gap in one line
Borrowers can prepay (a normal `repayment`, priced by the `prepayLoan` quote) or close early (`foreclosure`), but no charge trigger exists for paying early, so lenders can't recover the interest they lose.

## Code map (at the pin)
- **Charge definitions:**
  - `fineract-core/.../charge/domain/ChargeTimeType.java` (17 values)
  - `fineract-charge/.../charge/domain/ChargeCalculationType.java` (FLAT, PERCENT_OF_AMOUNT, PERCENT_OF_AMOUNT_AND_INTEREST, PERCENT_OF_INTEREST, PERCENT_OF_DISBURSEMENT_AMOUNT)
- **Repayment processors:**
  - `AbstractLoanRepaymentScheduleTransactionProcessor`, plus 9 strategy subclasses (FineractStyle, HeavensFamily, Creocore, RBI, EarlyPayment, PrincipalInterestPenaltyFeesOrder, InterestPrincipalPenaltyFeesOrder, DuePenFeeIntPri..., DuePenIntPriFee...)
  - `AdvancedPaymentScheduleTransactionProcessor`
- **Schedule engines:** `AbstractCumulativeLoanScheduleGenerator` and `ProgressiveLoanScheduleGenerator`. Both implement `calculatePrepaymentAmount`.
- **Replay:** `ReprocessLoanTransactionsService`, `LoanTransactionProcessingService`.
- **Reversal:** `adjustLoanTransaction` (`LoanWritePlatformService`, `LoanRepaymentAdjustmentCommandHandler`).
- **Payoff quote:** `LoanReadPlatformServiceImpl.retrieveLoanPrePaymentTemplate`, and the `prepayLoan` and `foreclosure` templates (GET `transactions/template`) on `LoanTransactionsApiResource`. There's no POST `prepayLoan`: a prepayment is a `repayment`. `foreclosure` is also a POST command.
- **Closest pattern:** `OVERDUE_INSTALLMENT`, which shows up in `LoanChargeService`, `LoanBalanceService`, both generators, `LoanRescheduleRequestDataValidatorImpl`, `Loan`, `LoanCharge`, and the charge read and write services.
- **Tests:**
  - unit tests, mostly in `fineract-provider/src/test` (146 classes), plus `fineract-progressive-loan` (13) and `fineract-loan` (8)
  - integration tests in `integration-tests/` (ChargesTest, LoanWaiveChargeTest, LoanTransactionReverseReplayChargeOffTest, ...)
  - Cucumber e2e in `fineract-e2e-tests-*`
  - Docker Compose files for Postgres and MariaDB

## Open design questions a spec has to settle
- **Trigger:** principal paid beyond what's due, or any payment before maturity. Partial prepayments, full payoff or foreclosure, or both.
- **Amount:** flat or a percent. The base is either prepaid or outstanding principal. Optional min and max caps.
- **Lock-in:** only within the first N months or instalments.
- **Exclusions:** final instalment on its due date, reschedule or refinance, write-off or charge-off, waiver, down payment. A foreclosure collapses the schedule first, so every foreclosure lands in the final period.
- **Collection:** taken from the same payment, or added as a due charge. Where it sits in the allocation order.
- **Payoff and foreclosure quotes** include it.
- **Reversals and replay** never leave a duplicate or stale penalty.
- **Both schedule engines**, and all four accounting modes.
- **Existing products and loans without the charge stay unchanged.**

## Why this over the other Enterprise ideas
- **OFBiz early-payment discount:** still open at HEAD (re-checked 2026-11-04). But the build needs packages.atlassian.com and artifacts.alfresco.com for one dependency each, which couldn't be confirmed reachable ahead of time, and the scope is smaller.
- **Pretix instalments:** still open, and the repo is clean. But Python/Django with a fast test suite is where a one-shot model does best, and the feature leans on checkout and emails more than backend logic.

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
- 9 transaction processors plus the advanced payment allocation. Allocation order decides whether the penalty takes money meant for principal.
- Backdated transactions replay the loan's whole history, so the penalty has to come out the same on every replay. Undoing a repayment must remove its penalty.
- With interest recalculation, a prepayment changes future interest and the schedule, so the order of the calculation matters.
- The payoff and prepay template amount must include the penalty.
- Accounting entries under both cash and accrual, plus COB accruals.
- There are two schedule engines (cumulative and progressive), and both have to work.

**Verification in the lane:** Gradle compile time hasn't been measured. Most loan unit tests are in `fineract-provider` and use Mockito, so they need no DB. Integration tests need Postgres or MariaDB in Docker and are slow, so only part of them gets run in a time-boxed run.
- Small risk: the version comes from git tags (`me.qoomon.git-versioning`), and it falls back if the lane clone has no tags. Watch the first build.

## Re-check (2026-11-03)
- **Gap still open at upstream HEAD `71281f17` (2026-11-03):**
  - no prepayment or early-repayment `ChargeTimeType`
  - no code matches prepayment, early-repayment or foreclosure fee identifiers
  - the 2,535 commits since the pin only fix prepay amounts and allocation
- **At the pin:** 0 AI co-author trailers, and no AI files. Build sources are Maven Central and the Gradle plugin portal only.
- **Size:** 5,667 Java files (930 tests) and 67 Cucumber feature files.
- **Scope a full solution has to cover:**
  - 10 repayment transaction processors (9 plus the advanced one)
  - 2 schedule engines (cumulative and progressive)
  - replaying backdated transactions
  - undoing and adjusting repayments
  - the prepay and foreclosure templates
  - four accounting modes
- **Closest existing pattern:** the `OVERDUE_INSTALLMENT` charge, used across about 21 files of main code.

## Re-check (2026-11-04)
- **Gap still open on upstream `develop` at `06ce07b5`:** no prepayment `ChargeTimeType`, and 0 hits for prepaymentPenalty, earlyRepaymentFee or foreclosureFee.
- Upstream added one loan charge time after the pin, `DOWN_PAYMENT` with id 17. That's also the next free id at the pin, so a new value there would clash with upstream.
- Corrections to the notes above are listed in section 18 of the [codebase notes](../top3/fineract-prepayment-penalty/codebase.md).

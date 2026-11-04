# Fineract loan prepayment penalty: facts for the four write-ups

Facts, numbers and open choices only, as fragments. The write-ups themselves get written separately.

## 1. The ask

### Notation
- **Pin**: `8a61e30a4521c77a59bc55f598e14c128ffd183a` (2026-02-14); `File.java:line` at the pin; short file names as in the path index of `codebase.md` (§2.1)
- **(attempt)**: line in the tree of the one finished attempt at the narrow scope (`codebase.md` §17); **(unverified)**: static reading only
- **Tags**: [narrow] = the 10-requirement spec; [wider] = wider-scope addition; [decide] = open choice, options split by `|`

### Domain context
- **Fineract**: Apache core-banking server; REST API only, no UI in the repo; multi-tenant, one database per tenant; loans, savings, shares, clients, GL accounting
- **Loan lifecycle**: product (charges, accounting mode, allocation rules); application with a `charges[]` array in the request JSON; approval; disbursement in one or several tranches; repayments; closure as CLOSED_OBLIGATIONS_MET 600 or OVERPAID 700 (`LoanStatus.java:26-37`)
- **Charge definition vs loan charge**: `m_charge` (`Charge`) vs `m_loan_charge` (`LoanCharge`); amount, caps, penalty flag, time and calculation type copied at attach (`LoanChargeService.create` :479-521)
- **Loan charge times today**: DISBURSEMENT 1, SPECIFIED_DUE_DATE 2, INSTALMENT_FEE 8, OVERDUE_INSTALLMENT 9 (penalty only), TRANCHE_DISBURSEMENT 12 (`ChargeTimeType.java:57-61`); no prepayment value at the pin (17 constants with INVALID, `ChargeTimeType.java:23-39`); upstream `develop` at `06ce07b5` (2026-11-04): still none, 0 hits for prepaymentPenalty, earlyRepaymentFee or foreclosureFee; one loan time type added there after the pin, DOWN_PAYMENT with id 17
- **Calculation types**: FLAT 1, PERCENT_OF_AMOUNT 2, PERCENT_OF_AMOUNT_AND_INTEREST 3, PERCENT_OF_INTEREST 4, PERCENT_OF_DISBURSEMENT_AMOUNT 5 (`ChargeCalculationType.java:23-28`); no outstanding-principal or prepaid-principal base (`LoanChargeService.java:244-271`)
- **Only system-created loan penalty today**: OVERDUE_INSTALLMENT, created by a COB step or job from the product's charge, never attached at loan creation (`LoanChargeWritePlatformServiceImpl.java:795, :804-805`)
- **Early-exit penalty precedent**: savings only, pre-closure penal interest (`DepositPreClosureDetail.java:40-49`)
- **Two schedule types**: CUMULATIVE (9 legacy processors, fixed allocation orders, chosen by strategy code) and PROGRESSIVE (advanced processor only, rule-driven allocation over {PAST_DUE, DUE, IN_ADVANCE} x {PENALTY, FEE, PRINCIPAL, INTEREST}, `PaymentAllocationType.java:40-51`); pairing enforced (`LoanProductDataValidator.java:2810-2823`)
- **Business date**, not the system clock (`DateUtils.getBusinessLocalDate` :235-237); COB date for batch work
- **Replay**: a backdated, adjusted or undone transaction re-runs every transaction; changed ones reversed and replaced, linked REPLAYED (`AbstractLoanRepaymentScheduleTransactionProcessor.createNewTransaction` :497-507)
- **Payoff quote**: GET `/v1/loans/{loanId}/transactions/template?command=prepayLoan`; paid by a plain POST `command=repayment`; no prepay POST command (`LoanTransactionsApiResource.java:566-612`)
- **Foreclosure**: GET template `command=foreclosure` plus POST `command=foreclosure`; the POST collapses the schedule (`LoanAccountDomainServiceJpa.java:946-1014`)
- **Accounting modes**: NONE 1, CASH_BASED 2, ACCRUAL_PERIODIC 3, ACCRUAL_UPFRONT 4 (`AccountingRuleType.java:30-33`); penalty GL keys INCOME_FROM_PENALTIES 5, PENALTIES_RECEIVABLE 9 under accrual (`AccountingConstants.java:43, :101`)
- **COB**: per-loan close-of-business batch; 11 step beans, 6 seeded (overdue penalties, delinquency, due and overdue checks, arrears aging, periodic accrual)

### Requirement topics
- **[narrow] scope of the 10-requirement version** (topic labels only; R numbers are used as labels below)
  - R1 definition rules: new time type; penalty only; FLAT or % of prepaid principal; every other combination rejected with 400
  - R2 attach routes: product charge, or loan charge added before disbursement
  - R3 trigger and amount: prepaid principal of the repayment; flat or %; min and max caps; partial prepayment, full payoff, prepay and foreclosure
  - prepaid principal in that version: principal of installments due after the transaction date; due and overdue amounts excluded
  - R4 lock-in: months from disbursement
  - R5 last-installment-period exclusion
  - R6 prepay and foreclosure quotes: penalty included; quoted amount on the quoted date closes the loan
  - R7 undo, adjust, backdated replay: one penalty per trigger, recalculated; waived stays waived
  - R8 cumulative and progressive
  - R9 existing penalty accounting (cash, accrual, none); product penalty income account
  - R10 no change for loans and products without the charge
  - test topics there: cumulative (partial %, due-only, lock-in in and out, last period, quote plus closing payment, undo, backdated); progressive partial plus payoff; accrual journals; existing tests unchanged
  - out of scope there: savings, client charges, web app
- **[wider] additions**
  - interest recalculation: cumulative REDUCE_EMI_AMOUNT, REDUCE_NUMBER_OF_INSTALLMENTS, RESCHEDULE_NEXT_REPAYMENTS; progressive ADJUST_LAST_UNPAID_PERIOD; generator quotes; payoff detection; COB recalculation
  - multi-disbursement: lock-in anchor, undisbursed tranche principal, undo last tranche
  - reschedule, re-amortize, re-age: last period and prepaid base after schedule changes; charge re-derivation
  - charge-off (with write-off and recovery): add-charge guards, ACCELERATE_MATURITY, GL by transaction type
  - business and external events: create, recalculate, remove, balance change, accrual; seeding; startup validation
  - pulled in with the wider scope: more trigger types (MIR, payout, goodwill credit, IPW, down payment, recovery), more quote consumers (contract termination, top-up, futureSchedule), chargeback, holidays, delinquency, COB

### Open choices: definition
- [decide] [narrow] **time type**: new value 17 `chargeTimeType.prepayment` | flag on SPECIFIED_DUE_DATE (upstream later gave id 17 to DOWN_PAYMENT, so a pinned branch and upstream would clash on 17)
  - loan set `validLoanValues` {1, 2, 8, 9, 12} (`ChargeTimeType.java:57-61`); dropdown list hard-coded (`ChargeDropdownReadPlatformServiceImpl.java:85-88`); one label case per value (`ChargeEnumerations.java:38` on); e2e copy with 15 values (`fineract-e2e-tests-core/src/test/java/org/apache/fineract/test/data/ChargeTimeType.java`)
- [decide] [narrow] **percentage base**: new calculation type 6 (% of prepaid principal) | PERCENT_OF_AMOUNT read differently for this time type | outstanding-principal base
  - 8 calculation-type switches in `LoanChargeService.java` (:249, :356, :396, :447, :766, :782, :822, :884); `LoanChargeApiJsonValidator.java:267, :427`; `LoanChargeAssembler.java:267`; update range `inMinMaxRange(1, 5)` (`ChargeDefinitionCommandFromApiJsonDeserializer.java:412-416`)
- **Penalty only**: settled by R1; the overdue precedent re-checks the flag on update (`Charge.java:596-598`); flag editable on update (:561-566)
- **Definition error status**: settled by acceptance criterion 1 (400) against a 403 precedent; decision point D5
- [decide] [narrow] **runtime rejections** (loan-level add after disbursement, duplicate, add on a charged-off loan): 400 | 403
  - today 403: `LoanChargeCannotBeAddedException` (`LoanChargeCannotBeAddedException.java:26`) and `error.msg.loan.is.charged.off` (`LoanChargeWritePlatformServiceImpl.java:187-190`); attempt: 400
- [decide] [narrow] **lock-in**
  - unit settled by R4 (months)
  - storage: definition column read live | copied into the loan charge at attach
  - null and 0: null none, 0 rejected (attempt) | 0 none
  - end day `disbursement date + n months`: inside (attempt `isAfter`, `PrepaymentPenaltyHelper.java:140`) | outside
  - month ends: `plusMonths` clamps (loans disbursed 30 and 31 January both end a 1-month lock-in on the last day of February)
  - [wider] anchor: first tranche through `Loan.getDisbursementDate` (:973-979, earliest actual tranche, set at `LoanDisbursementService.java:127-129`) | each tranche (`m_loan_disbursement_detail` rows); restart on reschedule or re-age | never
- [decide] [narrow] **caps**
  - caps on FLAT: rejected | dropped without an error as today (stored only for % of amount or % of disbursement, `Charge.java:289-292`)
  - min > max: rejected at definition | stated precedence; no min <= max check today (`ChargeDefinitionCommandFromApiJsonDeserializer.java:279-286`); pin order min wins (`LoanCharge.minimumAndMaximumCap` :309-331); attempt max wins (`PrepaymentPenaltyHelper.java:214-219`)
  - rounding: caps returned unrounded, in-range values through `Money.of` (`LoanCharge.java:328`) | every amount rounded
  - tiny prepaid principal: full min cap for any positive amount (attempt `PrepaymentPenaltyHelper.java:189-191, :214-216`; a 0.01 spill into the next installment gives a min-cap penalty) | threshold
- [decide] [narrow] **definition edits while in use**: allowed as today (only deactivation and fee frequency or interval blocked, `ChargeWritePlatformServiceJpaRepositoryImpl.java:140-158`) | Prepayment fields locked
  - existing rules and penalties after an edit: attach-time values kept | the edited definition followed

### Open choices: attach
- [decide] [narrow] **product-level attach**: through `charges[]` like every product charge (`LoanChargeAssembler.fromParsedJson` :66-224; a client that omits it gets none) | auto-attached at creation (attempt `LoanAssemblerImpl.java:213`) | read from the product at run time (overdue pattern, `LoanChargeWritePlatformServiceImpl.java:804-805`)
- [decide] [narrow] **per-loan overrides in `charges[]`**: `chargeTimeType` and `chargeCalculationType` read per entry (`LoanChargeAssembler.java:108-118`), not checked against allowed values (`LoanChargeApiJsonValidator.java:353-363`): rejected | coerced (attempt); `amount` required and positive per entry (:390-393), its meaning for a % rule (attempt: rate override, 3% over a 2% definition)
- [decide] [narrow] **loan-level add**: POST `/v1/loans/{loanId}/charges` before disbursement (R2); after disbursement rejected | allowed (spec silent); same definition twice rejected | allowed

### Open choices: trigger and amount
- [decide] [narrow] **trigger types** ("a repayment" in R3), with routes
  - REPAYMENT: `LoanRepaymentCommandHandler.java:47`, `makeLoanRepayment`, `LoanAccountDomainServiceJpa.makeRepayment`, shared handler (:1020); the top-up repayment of the old loan takes the same route (`repayLoanWithTopup`, `LoanWritePlatformServiceJpaRepositoryImpl.java:1676`)
  - foreclosure REPAYMENT: `LoanAccountDomainServiceJpa.java:1026`, sub-status FORECLOSED
  - GOODWILL_CREDIT, manual DOWN_PAYMENT: same route (`LoanGoodwillCreditCommandHandler.java:47`, `LoanDownPaymentCommandHandler.java:45`); auto down payment `LoanDownPaymentHandlerServiceImpl.java:232`
  - MERCHANT_ISSUED_REFUND, PAYOUT_REFUND: own processing in `makeRefund` (`LoanAccountDomainServiceJpa.java:808-895`), not the shared handler
  - INTEREST_PAYMENT_WAIVER: own processing (`LoanWritePlatformServiceJpaRepositoryImpl.java:943-993`, called :1027), not the shared handler
  - [wider] RECOVERY_REPAYMENT: written-off loans only (`LoanDownPaymentTransactionValidator.java:173-175`), capped at the written-off total (`LoanDownPaymentHandlerServiceImpl.java:88-92`), allocated like a repayment (`AdvancedPaymentScheduleTransactionProcessor.java:480-482`; cumulative `AbstractLoanRepaymentScheduleTransactionProcessor.java:181, :267, :594`)
  - [wider] REPAYMENT after charge-off
  - exclusions to state: CHARGE_ADJUSTMENT (also an adjustment of the penalty itself), CHARGE_REFUND, INTEREST_REFUND (created RELATED with each MIR or payout, `LoanAccountDomainServiceJpa.java:827-838`), CAPITALIZED_INCOME_ADJUSTMENT
    - all four repayment-like (`LoanTransaction.java:646-650`), allocated through `handleRepayment` (`AdvancedPaymentScheduleTransactionProcessor.java:480-482`), each with its own allocation rule type (`PaymentAllocationTransactionType.java:39-46`)
  - code sets: `isRepaymentType` 7 types (`LoanTransactionType.java:182-185`); `isRepaymentLikeType` 10 types; RECOVERY_REPAYMENT in neither
  - attempt: REPAYMENT only, not reversed, non-zero (`LoanPrepaymentPenaltyService.java:200-202`)
- [decide] [narrow] **current installment paid early**: prepaid (spec wording, its due date is after the transaction date; attempt `PrepaymentPenaltyHelper.java:124`) | not prepaid; question for the product owner
  - reference loan (below, §2), 600 on 15 Jan: 12 (600 prepaid) | 7 (350 prepaid)
  - installment due on the transaction date: due, never prepaid (spec)
  - processors define "in advance" 3 ways: due date after the transaction date (`AbstractLoanRepaymentScheduleTransactionProcessor.java:348-352`); before the previous installment's due date (`HeavensFamilyLoanRepaymentScheduleTransactionProcessor.java:83-97`); DUE only on the exact due date, so one day early makes the whole current installment IN_ADVANCE (`AdvancedPaymentScheduleTransactionProcessor.java:2629-2637`)
- [decide] [narrow] **last-period rule**
  - "within the period": after the last installment's fromDate | on or after it (attempt: a payment on the fromDate is outside, `PrepaymentPenaltyHelper.java:158`; wrapper `isInPeriod` from exclusive, to inclusive, `LoanRepaymentScheduleProcessingWrapper.java:251-254`)
  - which installment is last: additional N+1, down-payment, re-aged rows counted | skipped (attempt skips additional and down-payment, not re-aged, `PrepaymentPenaltyHelper.java:148`)
  - foreclosure: pre-collapse schedule | collapsed schedule (every foreclosure in the last period)
  - [wider] schedule read: at trigger time | current schedule on replay (reschedule, re-age and REDUCE_NUMBER_OF_INSTALLMENTS move the last due date, `Loan.getLastRepaymentPeriodDueDate` :633-643)
- [decide] [narrow] **shape on the loan**: one linked charge per trigger | one aggregate charge | deduction from the payment
  - waived flag per `LoanCharge`, kept by `LoanCharge.resetPaidAmount` (:168-175); aggregate: one waiver covers every trigger, undo of one trigger needs a subtraction
- [decide] [narrow] **link**: `to_loan_charge_id` on `m_loan_transaction_relation` (`LoanTransactionRelation.java:47-49`) plus a relation type appended last | new column or table | date and amount matching
  - relation type stored by ORDINAL (:51-53), `fromInt` maps only 1-3 (`LoanTransactionRelationTypeEnum.java:47-59`), so a mid-enum insert shifts stored values
  - `LoanTransactionRelation.linkToCharge` (:72-75) does not add itself to the transaction's set; relations copied only on replacement (`LoanTransaction.copyLoanTransactionRelations` :982-987); adjust builds a new transaction without relations (`LoanAdjustmentServiceImpl.java:117-118`)
  - date matching: same-day triggers collide
- [decide] [narrow] **stored amount**: fixed amount with its base kept (FLAT, base in `amountPercentageAppliedTo`, attempt) | plain % charge
  - a plain % penalty is re-based on total scheduled principal by the wrapper (`LoanRepaymentScheduleProcessingWrapper.java:179-206`)
  - and touched at 6 more re-derivation sites: `LoanChargeService.recalculateLoanCharge` :85-108; reschedule approve `LoanRescheduleRequestWritePlatformServiceImpl.java:455`; regeneration `LoanScheduleService.java:112-123`; both generators; foreclosure `LoanAccountDomainServiceJpa.java:993-995`
  - FLAT keeps `amountOrPercentage`; cost of FLAT: `chargeCalculationType` in the charge event (`LoanChargeDataV1.avsc:48`) reports FLAT
- [decide] [narrow] **collection**: own charge due on the trigger date, paid by later payments | paid from the same payment's surplus (attempt) | taken before principal
- [decide] [narrow] **trigger pays its own penalty**: never | from its surplus only (attempt: cumulative `payPrepaymentPenaltyFromOverpayment`, advanced `processOverpaidTransactions`) | first, shrinking its own base
  - cumulative replay spreads every active charge before any transaction (`AbstractLoanRepaymentScheduleTransactionProcessor.java:110-111`); penalty-first orders (FineractStyle, Creocore, RBI on time; both `DuePen*` strategies in advance) would then pay it from its own trigger
- [decide] [narrow] **cascade**: an earlier outstanding penalty paid from a later trigger before principal | kept out of the later trigger's base
  - attempt: 600 on 20 Jan gives 12; after a backdated 300 on 10 Jan: 6 and 11.88 (6.00 of the 20 Jan payment pays the 10 Jan penalty; `LoanPrepaymentPenaltyIntegrationTest.java:366-387`, attempt)
  - test product strategy `due-penalty-fee-interest-principal-in-advance-principal-penalty-fee-interest-strategy` (`BaseLoanIntegrationTest.java:506-507`); outcome depends on the strategy order

### Open choices: quotes
- [decide] [narrow] **quote computation**: the trigger's own computation | a separate quote formula (attempt: schedule rows due after the date, `PrepaymentPenaltyHelper.java:239-248`; trigger: the transaction's installment mappings, :118-129)
- [decide] **callers that show the penalty**, one per consumer of `retrieveLoanPrePaymentTemplate` (`LoanReadPlatformServiceImpl.java:556-589`) or a quote
  - [narrow] prepayLoan template (`LoanTransactionsApiResource.java:668-677`, date param or business date): R6
  - [narrow] foreclosure template (`LoanTransactionsApiResource.java:682-689`, `retrieveLoanForeclosureTemplate`): R6
  - [narrow] MIR, payout, goodwill, IPW templates: same method at the business date (:634-649)
  - [narrow] contractTermination template: same branch as prepayLoan (:668-677); termination progressive only, not charged off (`LoanContractTerminationServiceImpl.java:186-215`); accelerates maturity when an installment is unpaid (`AdvancedPaymentScheduleTransactionProcessor.java:503-504`), leaving no principal due later
  - [narrow] top-up: net disbursal from the old loan's prepay amount (`LoanWritePlatformServiceJpaRepositoryImpl.java:879-886`), the same amount checked at validation (`LoanApplicationValidator.java:682, :1433, :2011`); old loan repaid by REPAYMENT (`LoanWritePlatformServiceJpaRepositoryImpl.java:1667-1679`), so a trigger
  - [wider] futureSchedule association: `LoanScheduleCalculationPlatformServiceImpl.updateFutureSchedule` :83-120 (call :116), from `LoansApiResource.java:1114-1117`; active interest-bearing recalc loans, not NPA, not charged off, interest-first processors only
  - dispatch: generator path only for interest-bearing recalc loans not charged off on the date and not contract-terminated (`LoanTransactionProcessingServiceImpl.java:157`); else the whole-schedule sum (:175-188); foreclosure `LoanBalanceService.fetchLoanForeclosureDetail` :214-221, row dated on the business date (:218-220)
- **Quote internals**
  - template class read-only (`@Transactional(readOnly = true)`, `LoanReadPlatformServiceImpl.java:163`); `adjustPrepayInstallmentCharge` always 0 (:591-604)
  - progressive quote (`ProgressiveLoanScheduleGenerator.java`): date from the pre-close strategy, NONE throws (:194-198); saved model required (:200-202); down-payment principal added (:210-217); fees and penalties of every installment summed (:228-236), so a penalty row already on an installment is quoted automatically
  - cumulative quote under TILL_REST_FREQUENCY_DATE runs interest to the next rest date (`AbstractCumulativeLoanScheduleGenerator.java:2795-2798`)
- [decide] [narrow] **foreclosure base**: before `updateInstallmentsPostDate` (detail :716, collapse :724) | after; on later replays frozen per foreclosure penalty | recomputed | frozen for the whole loan (attempt: `AbstractLoanRepaymentScheduleTransactionProcessor.java:265`, `AdvancedPaymentScheduleTransactionProcessor.java:277`)
- [decide] [wider] **payoff detection** (`LoanAccountDomainServiceJpaHelper.java:66-72`, recalc and TILL_PRE_CLOSURE_DATE only): penalty inside the generator moves the threshold | template-only addition leaves it; exceptions swallowed (:74-80); null inside batches with an enclosing transaction (:47-64)

### Open choices: reversal and replay
- [decide] [narrow] **undo of a trigger whose penalty was waived**: penalty removed and waiver reversed (attempt `LoanPrepaymentPenaltyService.java:231-247`) | penalty kept waived
- [decide] [narrow] **adjust of such a trigger to a larger amount**: new amount waived | only the old amount (`LoanCharge.waive` covers the amount outstanding at that moment, :181-203)
- [decide] [narrow] **removal**: deactivated (attempt `setActive(false)`) | deleted; its accruals, accrual adjustments and waivers reversed (attempt) | kept

### Open choices: accounting
- [decide] [narrow] **accrual source** (R9 "accrued when added"): own ACCRUAL at creation | COB periodic accrual (ADD_PERIODIC_ACCRUAL_ENTRIES)
  - convention: charge-applied ACCRUAL only for active NONE, CASH or UPFRONT loans, or overpaid or closed-obligations-met ones (`LoanChargeWritePlatformServiceImpl.java:1056-1064`); periodic loans left to COB
- [decide] [narrow] **accrual date**: trigger date | business date (backdated triggers); an accrual dated before its trigger can hit ACCOUNTING_CLOSED in a closed GL period (`AccountingProcessorHelper.java:540-548`), one dated on the trigger date adds no new failure
- [decide] [narrow] **progressive accrual**: `handleChargeAppliedTransaction` returns null for progressive loans (`LoanChargeService.java:155-162`): own accrual for this penalty | COB only
- [decide] [narrow] **upfront outcome**: one Dr PENALTIES_RECEIVABLE / Cr INCOME_FROM_PENALTIES at creation on the decided date, none from COB, reversed on undo | another stated outcome
- [decide] [narrow] **GL account**: product-level INCOME_FROM_PENALTIES | the product's per-charge mapping (`acc_product_mapping.charge_id`, `AccountingProcessorHelper.getLinkedGLAccountForLoanCharges` :1147-1167)
  - the accrual repayment branch is product-level only (`AccrualBasedAccountingProcessorForLoan.java:1736-1756`); the `Charge` income account is savings only (`AccountingProcessorHelper.java:1169-1197`)

### Open choices: engines
- [decide] [narrow] **processor coverage**: all 9 cumulative strategies plus advanced | advanced plus the default cumulative one; HeavensFamily in-advance rule kept | unified
  - HeavensFamily (`HeavensFamilyLoanRepaymentScheduleTransactionProcessor.java`): in advance only before the previous due date (:83-97); in-advance money to principal first, then the whole installment interest waived without a transaction once principal completes (:133-135); the whole-schedule quote still includes that interest (`LoanTransactionProcessingServiceImpl.java:175-188`)

### Open choices: wider scope
- [decide] [wider] **interest recalculation** (how, not whether): measured at allocation | from the schedule before the transaction | as the difference after regeneration; quote through the generators (TILL_PRE_CLOSURE_DATE, TILL_REST_FREQUENCY_DATE); "out" only as a reported scope cut
  - cumulative regeneration folds early-paid amounts into the current period's principal (`AbstractCumulativeLoanScheduleGenerator.java:679-700`; `applyEarlyPaymentStrategy` :1552-1575); effect on a mapping-based measure (unverified)
  - progressive rows rewritten after allocation (`AdvancedPaymentScheduleTransactionProcessor.java:2790`); under TILL_REST_FREQUENCY_DATE an in-advance payment counts as paid on the due date (:2873-2881)
  - cumulative recalc loans refuse repayments dated before the last user transaction (`LoanTransactionValidatorImpl.java:718-735`), so backdated checks run only on progressive or non-recalc loans
  - not excluded by the narrow spec; progressive products require ADJUST_LAST_UNPAID_PERIOD (`LoanProductDataValidator.java:1173-1178`)
- [decide] [wider] **fee compounding**: unpaid penalty in compounded arrears under FEE or INTEREST_AND_FEE (`AbstractCumulativeLoanScheduleGenerator.java:1375-1377`) | excluded
- [decide] [wider] **multi-disbursement base**: undisbursed tranche principal excluded | rows as they are (cumulative rows carry future tranches, `AbstractCumulativeLoanScheduleGenerator.java:1978-2012`)
  - repaid principal capped at disbursed amounts ("amount.exceeds.threshold", `LoanDownPaymentHandlerServiceImpl.java:174-184`); undo last tranche refused after a later repayment (`LoanWritePlatformServiceJpaRepositoryImpl.java:3404`); foreclosure refused with an undisbursed tranche expected by the date (:2661-2665)
- [decide] [wider] **reschedule, re-age, re-amortize**
  - reschedule approve re-derives every active charge (`LoanRescheduleRequestWritePlatformServiceImpl.java:455`) and replays (:456)
  - reschedule blocked while a penalty is due after the from-date (overdue charges do this, `LoanRescheduleRequestDataValidatorImpl.java:228-238`) | allowed
  - earlier penalties frozen | recomputed on the new schedule
  - arrears turned into future principal by re-age or re-amortize: counted as prepaid when caught up | excluded
  - re-age keeps an unpaid penalty on its installment under DEFAULT handling, spreads it only under the EQUAL_AMORTIZATION types (`AdvancedPaymentScheduleTransactionProcessor.java:3190-3207`; `liftOutstandingBalances` :3998-4073)
- [decide] [wider] **charge-off**
  - creation after charge-off: new path past the add-charge guards | none (`LoanChargeWritePlatformServiceImpl.java:187-190`; COB overdue path :800-803)
  - ACCELERATE_MATURITY makes the current installment the last principal installment (`AdvancedPaymentScheduleTransactionProcessor.java:2047-2164`), so base 0; REGULAR and ZERO_INTEREST keep due dates; behaviours only with the advanced strategy (`LoanProductDataValidator.java:897-906`)
  - GL after charge-off by type: REPAYMENT Cr INCOME_FROM_RECOVERY / Dr FUND_SOURCE; MIR and payout Cr INCOME_FROM_CHARGE_OFF_PENALTY; goodwill Cr INCOME_FROM_RECOVERY / Dr INCOME_FROM_GOODWILL_CREDIT_PENALTY (`CashBasedAccountingProcessorForLoan.java:645-671`; `AccrualBasedAccountingProcessorForLoan.java:1483-1517`) vs R9 "penalty income account mapped on the loan product": question for the product owner
  - backdated trigger before the charge-off date: normal penalty income (`JournalEntryWritePlatformServiceJpaRepositoryImpl.java:860-865`); an own journal path instead: accounts differ from charge-off reporting and from replays across the charge-off date
- [decide] [wider] **chargeback of a trigger**: penalty kept | reversed through the CHARGEBACK relation (`LoanWritePlatformServiceJpaRepositoryImpl.java:1316, :1352`); principal reopened on the next installment due after the date (cumulative `AbstractLoanRepaymentScheduleTransactionProcessor.processCreditTransaction` :530)
- [decide] [wider] **undo disbursal**: generated penalties deactivated | left as today (non-overdue charges `resetToOriginal` and still active, `LoanWritePlatformServiceJpaRepositoryImpl.java:3158-3171`)
- [decide] [wider] **events** for create, recalculate, remove, balance change, accrual
  - options: `LoanAddChargeBusinessEvent` (add path in `LoanChargeWritePlatformServiceImpl`, pre :242, post :289) | loan-level event as for overdue charges (`LoanApplyOverdueChargeBusinessEvent` pre :1117, post :1139, with `LoanBalanceChangedBusinessEvent` :1140) | new event class with a seed part and both test lists | `NoExternalEvent`
  - per replay | net change only
  - `LoanChargeDataV1` carries time and calculation type as EnumOptionDataV1 (`LoanChargeDataV1.avsc:32-36, :48`), no schema change for new enum values; seeds in `parts/0056_add_external_event_default_configuration.xml` (Add :55, Delete :59, Update :63, ApplyOverdue :183, BalanceChanged :191)
  - startup validation of every tenant whatever the external-events flag (`ExternalEventConfigurationValidationService.java:46-113`); integration tests run with external events on (`integration-tests/build.gradle:75`)
  - attempt: no event on create, recalculate or remove (`LoanPrepaymentPenaltyService.java:208-247`); only `LoanAccrualTransactionCreatedBusinessEvent` (:323) and the rule add's `LoanAddChargeBusinessEvent`
- [decide] [wider] **holidays**: moved due date | original due date; the holiday job moves due dates without a replay (`ApplyHolidaysToLoansTasklet.java:99-115`)
- [decide] [wider] **delinquency**: unpaid penalty counted (penalty outstanding in overdue days, `LoanDelinquencyDomainServiceImpl.java:55-118`) | excluded; COB overdue penalty on an installment open only for this penalty: allowed | blocked (selection skips only obligations-met installments, `LoanReadPlatformServiceImpl.java:1610`)
- **COB**: no new step needed (a user transaction triggers); LOAN_INTEREST_RECALCULATION and ADD_PERIODIC_ACCRUAL_ENTRIES have to leave penalty count and amount as they are

## 2. Goal and success checks

### Context a domain expert needs
- **Business date**: tests move it through the API (`BaseLoanIntegrationTest.runAt` :1334); code reads `DateUtils.getBusinessLocalDate`, never `LocalDate.now()`
- **Command pipeline**: `@CommandType(entity, action)` handlers; permission rows `ACTION_ENTITY` in `m_permission`; maker-checker; one transaction per command (`CommandSourceService.processCommand` :133-152)
- **Error mapping**: `PlatformApiDataValidationException` 400; `AbstractPlatformDomainRuleException` subclasses 403 (`PlatformDomainRuleExceptionMapper.java:53`)
- **Processors**: 9 cumulative strategies by code (`m_loan.loan_transaction_strategy_code`) with fixed orders; the advanced processor with rules per transaction type (14 `PaymentAllocationTransactionType` values, DEFAULT fallback, `AdvancedPaymentScheduleTransactionProcessor.getAllocationRule` :3395-3400)
- **Installment kinds**: down payment, additional N+1, re-aged, recalculated-interest component (`is_down_payment`, `is_additional`, `is_re_aged`)
- **Per-installment record of a payment**: `LoanTransactionToRepaymentScheduleMapping`, the only place showing which installment a transaction settled
- **Fast path vs replay**: a chronologically latest repayment on a loan without recalculation is processed alone (`LoanDownPaymentHandlerServiceImpl.java:135-139`; `LoanTransactionProcessingServiceImpl.java:72-74`); on cumulative recalc loans a prepayment fails the exact-installment test (:91-92) and goes through regeneration plus replay (`LoanDownPaymentHandlerServiceImpl.java:144-163`)
- **Replay mechanics**: paid amounts reset, waivers kept (`resetPaidAmount`); changed transactions reversed and replaced; relations copied only on replacement; adjust = reversal plus a new transaction without relations; undo = adjust with amount 0
- **Charge sets**: `loan.getActiveCharges()` (40 calls) vs raw `getLoanCharges()` and `getCharges()` (25 lines) that include inactive rows
- **Quotes**: read-only; 4 calculations; dispatch by loan state; payoff is a plain repayment of the template amount (`BaseLoanIntegrationTest.verifyPrepayAmountByRepayment` :301-314)
- **Closure rule**: all charges paid = inactive, amount <= 0, paid or waived (`DefaultLoanLifecycleStateMachine.java:303-304`); overpaid from `getTotalOverpaid` (:302)
- **Foreclosure**: collapse before posting; FORECLOSED sub-status; a closed foreclosed loan cannot be adjusted (`LoanWritePlatformServiceJpaRepositoryImpl.java:1189-1192`)
- **Accounting split**: cash credits penalty income per charge at payment (`CashBasedAccountingProcessorForLoan.java:783`); accrual credits product-level PENALTIES_RECEIVABLE at payment and posts per-charge accrual entries (`AccrualBasedAccountingProcessorForLoan.java:1736-1756, :2028-2037`)
- **External events**: startup validation; `NoExternalEvent` marker; seeds as new Liquibase parts
- **Liquibase**: applied parts never edited; next free core part 0210, loan module 1035
- **Tests**: integration tests need a running server and database (Cargo, or `-PcargoDisabled` with a running server); generated client from Swagger annotations; RestAssured helpers deprecated for removal; e2e enum copies separate
- **Overdue precedent**: system-created penalty; rule read from the product; loan-level event; never re-created by replay; deactivated by undo disbursal, undo last tranche and foreclosure

### Reference loan for the numbers
- cumulative, 1,000 EUR, 0% interest, no recalculation, 4 monthly installments of 250 due 01 Feb, 01 Mar, 01 Apr, 01 May 2024, disbursed 01 Jan 2024, NONE accounting
- rule 2% of prepaid principal unless stated; numbers below assume the spec definition of prepaid principal unless stated
- numbers below also assume a trigger never pays its own penalty and a later trigger pays no earlier penalty (the "trigger pays its own penalty" and "cascade" choices in §1); other choices change them

### Success checks: [narrow] definition
- POST `/v1/charges` Prepayment + penalty + FLAT: created
- Prepayment + penalty + % of prepaid principal: created
- Prepayment + non-penalty: 400, not 403
- Prepayment + calculation type 2, 3, 4 or 5: 400 each
- Prepayment for savings, client or shares: 400 each
- % of prepaid principal with any other time type: 400
- PUT on a Prepayment charge setting penalty false or a disallowed calculation type: 400
- `lockInPeriodInMonths` on another time type: 400; 0 or negative: 400 or accepted per the decided meaning
- `lockInPeriodInMonths` persisted and returned by GET `/v1/charges/{chargeId}`
- GET `/v1/charges/template`: new time type and calculation type in the loan options
- min > max: 400 or the stated precedence; caps on FLAT: 400 or dropped, per the decision

### Success checks: [narrow] attach
- product with the charge, loan created with it in `charges[]`: rule on the loan
- product with the charge, `charges[]` omitted: rule present or absent per the decision
- POST `/v1/loans/{loanId}/charges` before disbursement: accepted
- same after disbursement, or the same definition a second time: decided status, or accepted per the decision
- `charges[]` entry with a Prepayment definition sent as SPECIFIED_DUE_DATE, and a non-Prepayment definition sent with the Prepayment time type: 400 or coerced per the decision

### Success checks: [narrow] trigger and amount
- 600 on 15 Jan: one penalty of 12 (7 under "current installment not prepaid"), due 15 Jan, penalty flag true, linked to that repayment; total outstanding 412 when the trigger does not pay it
- 250 on 01 Feb (installment 1 on its due date): no penalty
- 250 on 10 Mar after installment 1 was paid (installment 2 overdue): no penalty
- installment 1 paid 01 Feb, then 250 on 26 Feb (installment 2 before its 01 Mar due date): 5 or none per the decision
- max cap 10, 600 on 15 Jan: 10
- min cap 15, 600 on 15 Jan: 15
- min > max: amount per the stated precedence
- FLAT 50: 50 per trigger whatever the prepaid amount
- min cap 15 and 0.01 prepaid (one cent beyond the due amount): 15 or none per the decided threshold

### Success checks: [narrow] lock-in and last period
- FLAT 50, lock-in 2 months: 500 on 15 Feb (250 overdue, 250 prepaid) gives 50; 300 on 15 Mar gives none
- same rule, prepayments on 01 Mar (disbursement + 2 months) and on 02 Mar: inside or outside per the decided end day
- loan disbursed 31 Jan 2024, lock-in 1 month: lock-in ends 29 Feb 2024; prepayments on 29 Feb and 01 Mar per the decided end day
- installments 1-3 paid on their due dates, 250 on 15 Apr: no penalty, CLOSED_OBLIGATIONS_MET
- installments 1-2 paid on their due dates, 500 on 01 Apr (installment 3 due, 250 of installment 4, last fromDate 01 Apr): 5 or none per the decided boundary; 500 on 02 Apr: none

### Success checks: [narrow] quotes
- prepay template on 15 Jan: amount 1,020, principal 1,000, penaltyChargesPortion 20
- POST repayment of 1,020 on 15 Jan: CLOSED_OBLIGATIONS_MET, total outstanding 0, nothing overpaid, one penalty of 20 fully paid
- after 600 on 15 Jan: template on 20 Jan 420 (400 principal, 12 + 8 penalty); paying it closes the loan
- foreclosure template on 15 Jan: 1,020 with penalty 20; POST foreclosure: CLOSED_OBLIGATIONS_MET, penalty 20 paid
- foreclosure after an earlier partial prepayment: earlier penalty unchanged; foreclosure penalty on its own pre-collapse base
- installments 1-3 paid: template on 15 Apr 250, no penalty
- MIR, payout, goodwill, IPW and contractTermination templates on a loan with the charge: penalty shown only for callers decided as triggers

### Success checks: [narrow] reversal and replay
- undo of the 600 trigger: no active outstanding penalty for it; 1,000 outstanding again
- adjust of 600 to 300: one penalty of 6, linked to the new transaction
- 600 on 20 Jan, then a backdated 300 on 10 Jan: two penalties, 6 and 12 or 6 and 11.88 per the cascade decision
- then a backdated 10 on 05 Jan: three penalties, one per trigger, amounts per the decided rules
- waived penalty: still waived, same waived amount, after a backdated replay
- undo, or adjust to a larger amount, of a trigger whose penalty was waived: per the decision
- the same trigger posted as the latest transaction (fast path) and reached by replay: same penalty count and amount

### Success checks: [narrow] progressive (R8)
- partial prepayment: one penalty equal to rate x prepaid principal
- template amount paid: CLOSED_OBLIGATIONS_MET, not OVERPAID
- undo: no active penalty
- backdated payment: one penalty per trigger at the recalculated amount
- payment one day before a due date: base per the decided current-installment rule

### Success checks: [narrow] accounting (R9)
- NONE: no journal entries for the loan
- CASH, payoff of 1,020: Dr fund source 1,020 / Cr loan portfolio 1,000 / Cr penalty income 20 (product account, or the product's mapping for this charge)
- ACCRUAL_PERIODIC: accrual Dr PENALTIES_RECEIVABLE / Cr INCOME_FROM_PENALTIES 12 on the decided date; payment of the penalty Cr PENALTIES_RECEIVABLE 12; COB ADD_PERIODIC_ACCRUAL_ENTRIES adds no second accrual; no negative receivable afterwards
- ACCRUAL_UPFRONT: the decided outcome (default in §1: one accrual at creation, none from COB, reversed on undo)
- undo of the trigger: penalty accrual and income net to 0
- per-charge mapping on the product: used for the cash credit and the accrual entries

### Success checks: [narrow] no change (R10)
- loan and product without the charge: same schedule, allocation, templates, journals and events as at the pin
- foreclosure of a loan without the charge: same transactions and schedule as at the pin
- no existing test expectation changed in the diff
- new tests pass on the final tree

### Success checks: [wider] interest recalculation
- cumulative recalc REDUCE_EMI_AMOUNT, X paid beyond the current installment: rate x (X + unpaid principal of the current installment) or rate x X per the decision; unchanged after a later unrelated replay
- cumulative recalc REDUCE_NUMBER_OF_INSTALLMENTS: penalty still active after regeneration; last-period rule on the decided schedule
- progressive ADJUST_LAST_UNPAID_PERIOD, payment one day before a due date: base per the decision
- TILL_PRE_CLOSURE_DATE loan paid the quoted amount: CLOSED_OBLIGATIONS_MET, 0 outstanding, no interest after the transaction date
- futureSchedule association on such a loan: penalty shown or not per the decision
- COB LOAN_INTEREST_RECALCULATION run: same penalty count and amount

### Success checks: [wider] multi-disbursement
- second tranche in month 5, lock-in 6 months, prepayment in month 7: none with a first-tranche anchor, penalty with a per-tranche anchor
- base without undisbursed tranche principal, or as decided
- undo of a tranche disbursed after the trigger: penalty count and amount unchanged, or refused as today when a later repayment exists

### Success checks: [wider] reschedule, re-age, re-amortize
- reschedule approved after a trigger: same penalty count; amount per the decided rule, never a % of the whole loan
- re-age and undo re-age after a penalty: no duplicate; last-period rule on the decided schedule
- re-amortize, then a catch-up payment of the former arrears: penalty or none per the decision

### Success checks: [wider] charge-off
- prepaying REPAYMENT after charge-off: penalty or none, decided against the add-charge guard
- penalty paid after charge-off: GL per trigger type as listed in §1, or the decided mapping, in cash and accrual modes
- ACCELERATE_MATURITY: no penalty for payments after charge-off
- backdated prepayment before the charge-off date: normal penalty income; charge-off journal re-posted (2 + 2 entry pattern)
- prepay template on a charged-off loan (whole-schedule sum): equal to what the payment creates

### Success checks: [wider] other triggers and consumers
- MIR settling future principal: penalty or none per the decision; MIR template consistent with it
- payout, goodwill credit, IPW, down payment, recovery repayment: penalty or none per the decision, GL per mode
- charge adjustment of the penalty, and the INTEREST_REFUND created with an MIR: no new penalty
- contract termination, then payment of the template amount: closed, not OVERPAID
- top-up: old loan closed, penalty paid or none per the decision
- chargeback of a trigger: penalty kept or reversed per the decision
- undo disbursal: no active generated penalty

### Success checks: [wider] events, delinquency, holidays
- server starts for every tenant; GET `/v1/externalevents/configuration` (`ExternalEventConfigurationApiResource.java:41`, `fineract-core/src/main/java/org/apache/fineract/infrastructure/event/external/api/`) lists every new type
- create, recalculate, remove: the decided events, plus `LoanBalanceChangedBusinessEvent` when the balance changes
- external events on: no `ExternalEventConfigurationNotFoundException` on any trigger
- unpaid penalty past its installment's due date plus the wait period: delinquent or not per the decision; COB overdue penalty stacked or not per the decision
- due date moved by a holiday, payment on the original due date: penalty or none per the decision

### Shortcuts that must not count
- penalty in the template only, not created by the closing payment
- template amount right while the loan ends OVERPAID or ACTIVE
- penalty created in the API or command layer only (lost or duplicated on replay)
- penalty matched to its trigger by date or amount instead of a link
- waived penalty re-created or recalculated after replay
- prepaid principal read from regenerated or rewritten installment rows
- % penalty re-based on total loan principal at replay
- penalty added by a test step instead of by the trigger
- fast path only (no backdated, undo, reschedule or COB recalculation case)
- cumulative only, or progressive hitting the `UnsupportedOperationException` overloads (`AdvancedPaymentScheduleTransactionProcessor.java:170-197`)
- edited expectations in existing tests
- 500 errors instead of 400
- hard-coded GL accounts, or INCOME_FROM_PENALTIES posted outside the accounting processors
- zero foreclosure penalty after the schedule collapse
- new event class marked `NoExternalEvent` only to pass the startup check
- e2e features using the copied `ChargeTimeType` enum without the new value

## 3. Decision points

### D1. Where penalties are created, recomputed and removed
- **Fork**: penalty lifecycle inside processing and replay, or on one command path
- **Paths**: (a) per-trigger lifecycle in the processors and the replay loops; (b) hook on the shared repayment handler plus foreclosure
- **Code**
  - 30 `reprocess*` calls in 10 main files: `LoanWritePlatformServiceJpaRepositoryImpl` 10, `LoanChargeWritePlatformServiceImpl` 5, `LoanScheduleService` 3, `LoanReAgingService` 3, `LoanReAmortizationService` 3, `LoanContractTerminationServiceImpl` 2, one each in `LoanDownPaymentHandlerServiceImpl`, `CapitalizedIncomeWritePlatformServiceImpl`, `LoanAccountDomainServiceJpa`, `LoanRescheduleRequestWritePlatformServiceImpl`
  - MIR and payout processed in `makeRefund` (`LoanAccountDomainServiceJpa.java:808-895`), IPW in `LoanWritePlatformServiceJpaRepositoryImpl.java:943-993`; neither reaches the shared handler
  - fast path for a chronologically latest repayment (`LoanDownPaymentHandlerServiceImpl.java:135-139`); replay loops cumulative `AbstractLoanRepaymentScheduleTransactionProcessor.java:90-232`, progressive `AdvancedPaymentScheduleTransactionProcessor.java:201-289`
  - adding a charge itself replays (`LoanChargeWritePlatformServiceImpl.java:253-280`; a charge due on the trigger date never skips it, :263-271), so creation through the add-charge path re-runs the trigger
  - attempt: create and remove only at `LoanDownPaymentHandlerServiceImpl.java:103, :167` and `LoanAccountDomainServiceJpa.java:739` (attempt); amounts recomputed inside both processors; up to 5 passes of remove, create and full reprocess (`LoanPrepaymentPenaltyService.java:66, :103-117`, attempt)
- **What (b) breaks**
  - replays from charge add or waive, reschedule, re-age, re-amortize, chargeback, write-off, charge-off and undo, capitalized income, contract termination, COB recalculation: no missing penalty created, no obsolete one removed
  - penalties stale after a backdated payment, duplicated or missing after replay, orphaned after undo
  - MIR, payout and IPW never trigger
  - accrued amount drifts from the charge amount until the next repayment (code reading, not run)
  - outer loop: up to 5 extra full replays per repayment; stops without an error at the cap while amounts still change
- **Tests**: scope; root cause vs symptom; honesty (silent cap)

### D2. One prepaid-principal computation for quote and trigger
- **Fork**: how the quote knows the penalty that the paying repayment will create
- **Paths**: (a) one computation shared by quote and trigger, shown only to callers that can trigger; (b) a separate quote formula in the shared template
- **Code**
  - one template method for prepayLoan and contractTermination (`LoanTransactionsApiResource.java:668-677`) and for MIR, payout, goodwill and IPW (:634-649)
  - 4 quote calculations: cumulative generator (`AbstractCumulativeLoanScheduleGenerator.java:2791-2823`), progressive generator (`ProgressiveLoanScheduleGenerator.java:184-239`), whole-schedule sum (`LoanTransactionProcessingServiceImpl.java:175-188`), foreclosure detail (`LoanBalanceService.java:214-221`); dispatch :157
  - payoff detection compares the payment with the generator's outstanding (`LoanAccountDomainServiceJpaHelper.java:66-72`); exceptions swallowed (:74-80)
  - quote read-only (`LoanReadPlatformServiceImpl.java:163`); no penalty row at quote time
  - rows stop showing what a payment prepaid under recalculation: cumulative folding (`AbstractCumulativeLoanScheduleGenerator.java:679-700`), progressive rewrite (`AdvancedPaymentScheduleTransactionProcessor.java:2790`)
  - contract termination accelerates maturity before the payment (`AdvancedPaymentScheduleTransactionProcessor.java:503-504`)
  - attempt: quote from rows due after the date (`PrepaymentPenaltyHelper.java:239-248`), trigger from the transaction's mappings (:118-129); quote added for every caller (`LoanReadPlatformServiceImpl.java:582-590`, attempt); trigger REPAYMENT only (`LoanPrepaymentPenaltyService.java:200-202`, attempt)
- **What (b) breaks**
  - MIR, payout, goodwill and IPW templates quote a penalty those transactions never create; an MIR for the quoted amount leaves the penalty part as excess (code reading, not run)
  - contract termination: a penalty quoted before termination that the payment never creates; loan ends OVERPAID
  - quote and trigger disagree wherever rows and mappings differ (recalculation, HeavensFamily interest waiver, rounding)
  - generator-based payoff detection threshold left unchanged by a template-only addition
- **Tests**: root cause; scope; following instructions (R6 closing payment)

### D3. Foreclosure base and freeze
- **Fork**: when the foreclosure penalty is measured and what later replays may change
- **Paths**: (a) base taken before the schedule collapse, frozen per foreclosure penalty; (b) base after the collapse, or one freeze for the whole loan
- **Code**
  - `foreCloseLoan` (`LoanAccountDomainServiceJpa.java:702-768`): `fetchLoanForeclosureDetail` :716, then `updateInstallmentsPostDate` :724
  - collapse merges every installment due on or after the date into one due on it (:952-986); later charges set inactive (:991-992); their waivers reversed (:999-1013); afterwards nothing is due after the date and the collapsed installment is the last one
  - posting through the shared handler with sub-status FORECLOSED, never the fast path (:1023-1027; `LoanDownPaymentHandlerServiceImpl.java:137`)
  - `LoanForeclosureValidator`: no interest-bearing recalc loans (:31-35), no future date (:37-40), not before the last user transaction (:42-46); row dated on the business date (`LoanBalanceService.java:218-220`)
  - attempt: payoff penalty computed before the collapse (`LoanAccountDomainServiceJpa.java:727`, attempt) and added to the payment amount; `createForeclosurePenalties` once the payment is built (:739, attempt)
  - attempt: recompute frozen for every penalty once `loan.isForeclosure()` (`AbstractLoanRepaymentScheduleTransactionProcessor.java:265`, `AdvancedPaymentScheduleTransactionProcessor.java:277`, attempt)
  - attempt: `updateInstallmentsPostDate` moved after the payment build for every loan (:742, attempt; pin :724)
- **What (b) breaks**
  - after the collapse: base 0 and every foreclosure inside the last period, so no penalty; template differs from the posted amount; a later replay recomputes the penalty to 0
  - loan-wide freeze: earlier partial-prepayment penalties stop following replays after any foreclosure
  - the shared path reordered for every loan touches R10; one foreclosure test (`testLoanForeclosure`) in the attempt's regression runs
- **Tests**: root cause vs symptom; scope (R10)

### D4. How the rule sits on the loan
- **Fork**: representation of the attached rule and its values on each loan
- **Paths**: (a) rule kept off `LoanCharge`, or every charge accessor audited; (b) placeholder `LoanCharge` rows filtered at one accessor
- **Code**
  - 40 `.getActiveCharges()` calls in 19 main files; 25 raw accessor lines (13 `.getLoanCharges()`, 12 `loan.getCharges()`); predicate calls `LoanAccrualsProcessingServiceImpl.java:437, :570`
  - accrual predicate evaluates `isChargeDue` before `isActive` (:569-575); :646 dereferences the due date when `charge-accrual-date` = submitted-date (`GlobalConfigurationConstants.java:73`)
  - raw readers include `InvestorBusinessEventSerializer.java:135`, the reschedule overdue check (`LoanRescheduleRequestDataValidatorImpl.java:232`) and the closure rule (`DefaultLoanLifecycleStateMachine.java:303`)
  - product charges reach loans only through `charges[]` (`LoanChargeAssembler.fromParsedJson` :66-224); the overdue precedent reads the product at run time (`LoanChargeWritePlatformServiceImpl.java:804-805`)
  - definitions editable while in use (`ChargeWritePlatformServiceJpaRepositoryImpl.java:140-158`); amount and caps copied into the loan charge (`LoanChargeService.java:488-489`)
  - attempt: rule rows with time 17, no due date, amount 0, hidden from `getActiveCharges` (`Loan.java:1260`, attempt) and from the schedule (`LoanScheduleAssembler.java:744`, attempt); lock-in read live (`PrepaymentPenaltyHelper.java:192`, attempt); generated penalties recognised from the live definition (`LoanCharge.java:253-255`, attempt)
- **What (b) breaks**
  - null-due-date rows at raw and predicate sites: NullPointerException path at `LoanAccrualsProcessingServiceImpl.java:646` (code reading, not run); rule rows in investor events and charge lists
  - snapshot and live reads mixed: a time-type edit on the definition orphans every generated penalty; lock-in edits act retroactively, amount and cap edits do not
  - auto-attach departs from the `charges[]` convention and from the overdue precedent; per-loan time and calculation overrides ignored
- **Tests**: repo conventions

### D5. Definition errors: spec status vs precedent
- **Fork**: HTTP status for an invalid Prepayment definition where the spec and the overdue precedent disagree
- **Paths**: (a) field-level 400 as the spec says, departure from the precedent named in the change notes; (b) a copy of the overdue 403 domain exception
- **Code**
  - overdue precedent: non-penalty OVERDUE_INSTALLMENT throws `ChargeMustBePenaltyException` in the `Charge` constructor (`Charge.java:278-280`; update :596-598)
  - `ChargeMustBePenaltyException` extends `AbstractPlatformDomainRuleException` (`ChargeMustBePenaltyException.java:27`), mapped to 403 (`PlatformDomainRuleExceptionMapper.java:53`)
  - the same 403 family for attaching an overdue charge, by manual add or in `charges[]` (`LoanChargeCannotBeAddedException.java:26`; `LoanChargeWritePlatformServiceImpl.java:1008-1011`; `LoanChargeApiJsonValidator.java:385-389`)
  - repo TODO on domain vs validation exception (`LoanChargeApiJsonValidator.java:382`, on the currency check)
  - spec acceptance criterion 1: HTTP 400 for every other combination, the non-penalty case included
  - field errors through `DataValidatorBuilder` and `PlatformApiDataValidationException` in `ChargeDefinitionCommandFromApiJsonDeserializer` (`validateForCreate` :91, `validateForUpdate` :296)
  - attempt: 400 for every definition error (`Charge.java:388-428`, attempt)
- **What (b) breaks**: acceptance criterion 1 for the non-penalty case; client handling keyed on 400
- **Tests**: following instructions vs repo conventions; clarifying (departure stated, not silent)

### D6. Verification of the final tree
- **Fork**: what counts as evidence that the feature works
- **Paths**: (a) the whole new IT class re-run after the last edit, with an exact report of what ran; (b) an earlier green run reported as final
- **Code and record**
  - integration tests need a running server and database (Cargo started before `test` unless `-PcargoDisabled`, `integration-tests/build.gradle:142-146`), external events on (:75)
  - attempt log times: it1 19 tests green at 09:29:39; IT file modified at 09:30:07; it2 only 2 tests at 09:30:32; regression runs it-reg1 (82 tests, 11 classes) and it-reg2 (102 tests, 6 classes) without the new class
  - not run: 255 of 272 existing integration-test classes, the Cucumber e2e suite; no unit tests for the 2 new main classes
  - behaviour spans JPA flush order (attempt `LoanPrepaymentPenaltyService.java:104-105`), Liquibase, REST error codes, journals and COB accruals
- **What (b) breaks**
  - the final IT file never ran in full; 17 of its 19 tests not re-run after its last edit
  - formula-only unit tests with mocked repositories miss duplicates on replay, overpaid closure, double accrual and wrong GL
  - ITs written but not run prove nothing
- **Tests**: verification; honesty

## 4. Difficulty and codebase understanding

### System
- **What**: Apache Fineract, core-banking REST server, multi-tenant; pin `8a61e30a` (2026-02-14)
- **Stack**: Java 21, Spring Boot 3.5.6, EclipseLink JPA with static weaving, Liquibase, MariaDB / MySQL / PostgreSQL, Gradle 8.14.3
- **Size**: 29 Gradle modules in `settings.gradle` (plus `:custom:docker` and 8 sample `custom/acme` modules); 5,667 tracked `.java` files (4,736 main, 930 test, 1 `misc/Main.java`)
- **Tests**: 490 JUnit classes by the CI rule (integration-tests 272, provider 146, core 14, progressive-loan 13, client-feign 12, investor 10, fineract-loan 8, 15 in 9 other modules); integration-tests 421 `.java`, 266 `*Test.java`; 67 `.feature` files (45 e2e with 2,177 scenarios, 20 provider, 2 custom)
- **Schema**: core Liquibase parts 0001-0209 (next 0210); loan module parts 1001-1034 (next 1035)
- **Test cost**: full run "often take[s] an hour or longer" (`CONTRIBUTING.md:80`); integration CI in 5 shards, 60-minute job timeout (`.github/workflows/build-postgresql.yml:11, :94`)

### What the task touches
- **Modules**: fineract-core (`ChargeTimeType`), fineract-charge (definition, validation, labels, Swagger), fineract-loan (domain, cumulative processors, replay, `LoanChargeService`), fineract-progressive-loan (advanced processor, progressive generator), fineract-provider (write and read services, quotes, foreclosure, accounting, accruals, Liquibase, bean config), integration-tests; e2e enum copies in fineract-e2e-tests-core; no Avro schema change for new enum values
- **Repayment call path**
  1. `LoanTransactionsApiResource` POST `command=repayment` (:566-567)
  2. `LoanRepaymentCommandHandler`
  3. `LoanWritePlatformServiceJpaRepositoryImpl.makeLoanRepayment` :936-941
  4. `LoanAccountDomainServiceJpa.makeRepayment` :216-331
  5. `LoanDownPaymentHandlerServiceImpl.handleRepaymentOrRecoveryOrWaiverTransaction` :82-185: fast path, or regenerate plus replay
  6. processor allocation (cumulative strategy order or advanced rules)
  7. journal posted from `LoanAccountDomainServiceJpa.java:289` (`JournalEntryWritePlatformServiceJpaRepositoryImpl.createJournalEntriesForLoanTransaction` :813-831); post events (`LoanAccountDomainServiceJpa.java:290-295`)
- **Other flows**: charge definition create and update; loan creation with `charges[]`; add charge (`LoanChargeWritePlatformServiceImpl.addLoanCharge` :182-300); adjust and undo (`LoanAdjustmentServiceImpl`); 4 quote calculations and their consumers; foreclosure (`LoanAccountDomainServiceJpa.foreCloseLoan` :702-768); cash and accrual processors; charge-applied and periodic accruals; event raise points, seeds, startup validation
- **Big files on the path (lines)**
  - over 2,000: `AdvancedPaymentScheduleTransactionProcessor` 4,114; `LoanWritePlatformServiceJpaRepositoryImpl` 3,651; `LoanProductDataValidator` 3,018; `AbstractCumulativeLoanScheduleGenerator` 2,859; `LoanReadPlatformServiceImpl` 2,322; `AccrualBasedAccountingProcessorForLoan` 2,197; `ProgressiveEMICalculator` 2,066
  - under 2,000: `Loan` 1,854; `LoanChargeWritePlatformServiceImpl` 1,449; `LoanAccrualsProcessingServiceImpl` 1,298; `LoanTransaction` 1,045; `LoanAccountDomainServiceJpa` 1,029; `CashBasedAccountingProcessorForLoan` 1,005; `AbstractLoanRepaymentScheduleTransactionProcessor` 995; `LoanChargeService` 964
- **Enum surface**: `ChargeTimeType` 17 constants (next 17); `ChargeCalculationType` 6 (next 6); `LoanTransactionType` 43 constants, ids 0-43, id 11 unused (next 44); `LoanTransactionRelationTypeEnum` 6 (by ordinal); `PaymentAllocationType` 12; `PaymentAllocationTransactionType` 14; `AccountingRuleType` 4; 7 repayment types, 10 repayment-like
- **Processing surface**
  - 10 processors (9 cumulative + advanced); 3 definitions of "in advance"; 30 reprocess calls in 10 files
  - 13 kinds of replay trigger: backdated transaction, undo or adjust, charge add, charge waive or undo waive, reschedule approve, re-age or re-amortize and undo, chargeback, write-off and undo, charge-off and undo, capitalized income, contract termination, tranche disbursement and undo, COB interest recalculation
  - 40 `getActiveCharges` calls plus 25 raw accessor lines; 8 calculation-type switches in `LoanChargeService`; 7 charge re-derivation sites
- **Quote surface**: 4 calculations; consumers prepayLoan, contractTermination, MIR / payout / goodwill / IPW templates, foreclosure template, top-up, futureSchedule, payoff detection
- **Event surface**: 122 event types in `ExternalEventConfigurationHelper` (size asserted, `ExternalEventConfigurationIntegrationTest.java:56`); 25 core parts insert into the configuration table (34 XML files with module parts and one custom file); 38 loan Avro schemas; 60 e2e event classes; 11 COB loan step beans, 6 seeded
- **Precedent**: OVERDUE_INSTALLMENT literal in 12 files (26 lines); `OVERDUE_INSTALLMENT|isOverdueInstallmentCharge` in 22 main files (21 real); broad regex 26 non-test files (25 real)
- **Test counts by pattern**: 33 IT files match `isInterestRecalculationEnabled`, 38 `*Test.java` match `InterestRecalculation`, 43 match `interestRecalculation` case-insensitive; 35 IT files with charge or penalty in the name (32 `*Test.java`)
- **Product matrix from the validator**
  - cumulative: 9 processors x recalc off or on; with recalc, 3 reschedule strategies (ADJUST_LAST_UNPAID_PERIOD rejected, `LoanProductDataValidator.java:1166-1171`) x pre-close 0-2 (:1310-1314)
  - progressive: 1 processor x HORIZONTAL or VERTICAL x recalc off or on (only ADJUST_LAST_UNPAID_PERIOD, :1173-1178) x 3 charge-off behaviours (advanced strategy only, :897-906)
  - recalc products also pick a compounding method 0-3 (:1131-1132) and a rest frequency type 1-4 (:1188-1189)
  - each x 4 accounting modes; single or multi-disbursement; down payment on or off

### Why it is hard
- **Per-trigger identity across replay**
  - changed transactions reversed and replaced, with a REPLAYED relation to the old one (`AbstractLoanRepaymentScheduleTransactionProcessor.createNewTransaction` :497-507; advanced override `AdvancedPaymentScheduleTransactionProcessor.java:1453-1490`)
  - relations copied only on replacement (`LoanTransaction.copyLoanTransactionRelations` :982-987, called at `AbstractLoanRepaymentScheduleTransactionProcessor.java:502` and `AdvancedPaymentScheduleTransactionProcessor.java:1422`)
  - adjust builds a fresh transaction without relations (`LoanAdjustmentServiceImpl.java:117-118`); undo leaves no transaction
  - each penalty must follow its trigger through all four
- **Two paths for one repayment**: fast path vs regenerate plus replay (`LoanDownPaymentHandlerServiceImpl.java:135-163`); same result needed on both
- **Self-referential allocation**: a penalty dated on its trigger date sits in that installment's period (`LoanRepaymentScheduleProcessingWrapper.isInPeriod` :251-254); cumulative replay spreads all active charges before transactions (`AbstractLoanRepaymentScheduleTransactionProcessor.java:110-111`); penalty-first orders pay it from its own trigger and shrink the base
- **Quote before the fact**: read-only quote; 4 calculations dispatched by loan state; payoff detection threshold; one template method behind 6 template commands (prepayLoan, contractTermination, MIR, payout, goodwill, IPW) and 4 top-up call sites (`LoanWritePlatformServiceJpaRepositoryImpl.java:884`; `LoanApplicationValidator.java:682, :1433, :2011`)
- **Foreclosure rewrites the schedule** before posting (`LoanAccountDomainServiceJpa.java:724`)
- **"In advance" defined 3 ways**, none equal to the spec definition
- **Generic % logic**: 7 re-derivation sites; only the overdue type keeps its base
- **Charge sets**: 40 + 25 accessor sites; placeholder rows leak
- **Interest recalculation rewrites rows**: cumulative folding, progressive rewrite; recalc loans cannot foreclose and refuse backdated repayments
- **Accounting**: 4 modes, 2 processors; cash per charge vs accrual product-level receivable; no charge-applied accrual for progressive loans; GL after charge-off by type; closed periods by transaction date
- **Events**: unseeded class stops the server for every tenant; seed parts plus 2 test lists
- **Attempt effort at the narrow scope**: reported 96 minutes (file times 07:52-09:42; what bounds the 96 minutes unverified); 26 tracked files +530 / -37 plus 4 new files of 1,280 lines; 8 build logs 08:05-08:32; 19 new ITs green on an earlier version of the IT file, 2 re-run on the final file; 184 regression ITs (82 + 102); 1,140 unit tests (42 + 177 + 921); checkstyle passed
- **Attempt traps**: one hook among 30 replay sites; quote and trigger computed two ways; template penalty for every caller; loan-wide foreclosure freeze; rule rows leak; silent 5-pass cap; no events; max wins over min; boundary days untested; final IT file not re-run in full

### Not settled from code (unverified)
- vertical allocation (`AdvancedPaymentScheduleTransactionProcessor.processPeriodsVertically` :2917-3058) and the interest-bearing progressive path (:2790-2853)
- effect of cumulative early-payment folding (`AbstractCumulativeLoanScheduleGenerator.java:679-700`) on a mapping-based measure
- product-level rejection of WEEKLY and MONTHLY rest frequency on progressive loans (only a run-time `IllegalStateException`, `AdvancedPaymentScheduleTransactionProcessor.java:2900-2901`)
- cumulative prepay template with undisbursed tranches
- progressive foreclosure without recalculation (no IT or feature found)
- GL of REAGE and REAMORTIZE transactions
- allocation of a manual down payment larger than the down-payment installment
- two concurrent repayments on one loan
- compile and test durations; what bounds the attempt's 96 minutes

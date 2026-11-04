# Fineract: codebase context for a loan prepayment penalty

https://github.com/apache/fineract/tree/8a61e30a4521c77a59bc55f598e14c128ffd183a (2026-02-14); Java 21, Spring Boot 3.5.6, EclipseLink JPA, Liquibase, MariaDB / MySQL / PostgreSQL, Gradle 8.14.3; 29 Gradle modules in `settings.gradle` (plus `:custom:docker` and 8 sample `custom/acme` modules), 5,667 tracked `.java` files (4,736 under `*/src/main/java`, 930 under `*/src/test`, 1 `misc/Main.java`)

## 1. What Fineract is and its runtime shape

- **Notation**
  - `File.java:line` = line at the pin; full paths for short names in the §2 path index
  - (attempt) = the finished attempt's working tree, described in §17; (unverified) = static reading only, not confirmed in code or by a run
  - counts from `git ls-files`, `git grep`, `wc -l` at the pin; nothing in the repository was built or run for these notes
- **What it is**
  - Apache Fineract: open-source core-banking server (loans, savings, shares, clients, GL accounting)
  - REST API only; no UI in the repository
  - pin commit message: "FINERACT-2486: Fix guarantor endpoint returning savingsAccount.id=0 for group savings (#5482)"
- **App**
  - Spring Boot 3.5.6 (`build.gradle:109`)
  - entry `fineract-provider/src/main/java/org/apache/fineract/ServerApplication.java`: extends `SpringBootServletInitializer`; imports `FineractWebApplicationConfiguration` and `FineractLiquibaseOnlyApplicationConfiguration`
  - component scan `org.apache.fineract.**` (`FineractWebApplicationConfiguration.java:52`); DataSource, Hibernate JPA, Liquibase and JdbcTemplate autoconfiguration excluded (:47-49)
  - JPA is EclipseLink with static weaving, not Hibernate (`STATIC_WEAVING.md`, `static-weaving.gradle`, one `persistence.xml` per module; `fineract-loan/dependencies.gradle:71-76` excludes hibernate)
  - packaging: boot JAR `./gradlew clean bootJar` (`README.md:88`); WAR for Tomcat `./gradlew :fineract-war:clean :fineract-war:war` (`README.md:112`)
  - HTTPS port 8443, context `/fineract-provider`, health `/fineract-provider/actuator/health`; default Basic auth `mifos` / `password` (`README.md`)
  - node roles `fineract.mode.read-enabled`, `write-enabled`, `batch-worker-enabled`, `batch-manager-enabled` (`fineract-provider/src/main/resources/application.properties:67-70`)
  - loan COB switch `fineract.job.loan-cob-enabled`, default true (`application.properties:81`)

### 1.1 Multi-tenancy
- **Tenant store** DB `fineract_tenants`: tables `tenants` and `tenant_server_connections`, joined on `oltp_Id` / `report_Id` (`TenantMapper.java:31-57`; `JdbcTenantDetailsService` in the same package)
- **Tenant data**: one DB per tenant (default `fineract_default`); properties `fineract.tenant.*` at `application.properties:44-65` (host localhost:3306, identifier `default`, timezone Asia/Kolkata, rounding mode 6)
- **Routing**: `RoutingDataSource`, `DataSourcePerTenantServiceFactory`, `TomcatJdbcDataSourcePerTenantService` (`fineract-core/src/main/java/org/apache/fineract/infrastructure/core/service/database/`); `TenantDataSourceFactory` (`.../core/service/migration/`)
- **Tenant resolution** in `TenantAwareBasicAuthenticationFilter`: header `Fineract-Platform-TenantId` (:71, read :114); fallback query parameter `tenantIdentifier` (:117); then `setTenant` (:131) and `setBusinessDates` (:133)
- **Request context** in ThreadLocals (`ThreadLocalContextUtil.java:35-40`)
- **Business dates**
  - `BusinessDateType`: BUSINESS_DATE(1), COB_DATE(2) (`BusinessDateType.java:28-29`)
  - `ActionContext`: DEFAULT uses BUSINESS_DATE, COB uses COB_DATE (`ActionContext.java:29-30`)
  - `ThreadLocalContextUtil.getBusinessDate()` picks the date for the current action context (:94); `DateUtils.getBusinessLocalDate()` reads it (`DateUtils.java:235-237`)
  - tests move the date through the business-date API (`BaseLoanIntegrationTest.runAt` :1334)

### 1.2 Databases
- `DatabaseType`: MYSQL, POSTGRESQL; the MariaDB driver maps to MYSQL (`DatabaseTypeResolver.java:31-33`)
- requirements: MariaDB >= 11.5.2 or PostgreSQL >= 17.0; Java >= 21 (`README.md:30-32`)
- CI images: postgres:17.4, mariadb:11.5.2, mysql:9.1 (`.github/workflows/build-postgresql.yml`, `build-mariadb.yml`, `build-mysql.yml`)
- default datasource `org.mariadb.jdbc.Driver`, `jdbc:mariadb://localhost:3306/fineract_tenants` (`application.properties:401-402`)
- dialect differences through `DatabaseSpecificSQLGenerator` (`escape`, `limit`, `subDate`, `currentBusinessDate`, `groupConcat`), used throughout `LoanReadPlatformServiceImpl`

### 1.3 Liquibase
- **Runner** `TenantDatabaseUpgradeService`: contexts `tenant_store_db`, `initial_switch`, `tenant_db`, `custom_changelog` (:58-61); `upgradeTenantStore` :104; `upgradeIndividualTenants` :138; tenant identifier passed as a context (:183-203)
- **Master** `fineract-provider/src/main/resources/db/changelog/db.changelog-master.xml`, include order (:30-43):
  1. tenant-store changelogs (:30-31)
  2. `initial-switch-changelog-tenant.xml`: parts 0001-0002 (:32)
  3. `tenant/changelog-tenant.xml`: 206 includes, 0003-0209 (:33); last include `0209_transaction_summary_with_asset_owner_and_from_asset_owner_id_for_asset_sales.xml` at `changelog-tenant.xml:230`
  4. modules, under the comment "Add new module to the end" (:34-41): loan, investor, savings, `includeAll db/custom-changelog` (context `custom_changelog`), progressiveloan, loanorigination, command
  5. `final-changelog-tenant.xml`: part 0146 (:43)
- **Core parts** `fineract-provider/src/main/resources/db/changelog/tenant/parts/`: 209 files; include line `<include file="parts/NNNN_x.xml" relativeToChangelogFile="true" />`
- **Module ranges**
  - loan `fineract-loan/src/main/resources/db/changelog/tenant/module/loan/parts/`: 34 files, 1001-1034 (`1001_add_audit_fields_to_loan_charge.xml` to `1034_loan_reamortization_parameters.xml`), master `module/loan/module-changelog-master.xml`
  - savings 2001-2004; progressiveloan 5001-5003; investor 0001-0022; loanorigination 0001-0004; command 0001-0002; tenant store 0001-0010 (11 files)
  - charge module master `fineract-charge/src/main/resources/jpa/charge/db/changelog/tenant/module/charge/module-changelog-master.xml` is empty ("starting from 6000") and not included; accounting, rates and branch masters not included either
- **Next free numbers**: core `parts/0210` plus one include line in `tenant/changelog-tenant.xml`; loan module `1035_*` plus one line in the loan master
- **Applied parts are never edited** (checksums): a new external event type gets its own part (pattern `parts/0191_add_LoanApprovedAmountChangedBusinessEvent.xml`; row format as in `parts/0056_add_external_event_default_configuration.xml:55`)

### 1.4 COB (close of business)
- **Spring Batch** constants (`LoanCOBConstant.java:25-35`): job `LOAN_COB`, step-config job name `LOAN_CLOSE_OF_BUSINESS`, inline job `INLINE_LOAN_COB`
- **Manager** (`LoanCOBManagerConfiguration.java:115-120`): `resolveCustomJobParametersStep`, partitioned `loanCOBStep`, `stayedLockedStep`
- **Worker** (`LoanCOBWorkerConfiguration.java`): remote partitioning (:87-89); flow initialisation, applyLock, `loanBusinessStep`, resetContext; `loanBusinessStep` = chunk of `LoanItemReader` / `LoanItemProcessor` / `LoanItemWriter` (:121-128)
- partition transport flags: spring-events true, jms false, kafka false (`application.properties:97-104`); ActiveMQ / Kafka remote partitioning described at `README.md:264`
- **Step runner** `COBBusinessStepServiceImpl`: configured steps in order through a TreeMap (:59-75); beans matched to rows by `getEnumStyledName` (:98-107)
- interfaces `fineract-cob/src/main/java/org/apache/fineract/cob/COBBusinessStep.java`; loan flavour `fineract-loan/src/main/java/org/apache/fineract/cob/loan/LoanCOBBusinessStep.java`
- **Config table** `m_batch_business_steps` (created in `parts/0022`); default seeds cover only 6 steps of `LOAN_CLOSE_OF_BUSINESS`:

| order | step | seed |
|---|---|---|
| 1 | APPLY_CHARGE_TO_OVERDUE_LOANS | `parts/0022_add_batch_business_step_configuration_table.xml:44` |
| 2 | LOAN_DELINQUENCY_CLASSIFICATION | `parts/0047_add_loan_delinquency_tags_business_step.xml:28` |
| 3 | CHECK_LOAN_REPAYMENT_DUE | `parts/0067_add_configurations_for_repayment_due_business_steps.xml:59` |
| 4 | CHECK_LOAN_REPAYMENT_OVERDUE | `parts/0067_...xml:64` |
| 5 | UPDATE_LOAN_ARREARS_AGING | `parts/0089_add_update_loan_arrears_aging_business_step.xml:28` |
| 6 | ADD_PERIODIC_ACCRUAL_ENTRIES | `parts/0092_add_periodic_accrual_entries_business_step.xml:28` |

- **Step beans**: 11 `*BusinessStep.java` implementing `LoanCOBBusinessStep` in `fineract-provider/src/main/java/org/apache/fineract/cob/loan/` (enum name line):
  - `ApplyChargeToOverdueLoansBusinessStep` :48; `SetLoanDelinquencyTagsBusinessStep` (LOAN_DELINQUENCY_CLASSIFICATION) :122; `CheckLoanRepaymentDueBusinessStep` :71; `CheckLoanRepaymentOverdueBusinessStep` :75; `CheckDueInstallmentsBusinessStep` :99
  - `UpdateLoanArrearsAgingBusinessStep` :42; `AddPeriodicAccrualEntriesBusinessStep` :51; `AccrualActivityPostingBusinessStep` (ACCRUAL_ACTIVITY_POSTING) :50; `CapitalizedIncomeAmortizationBusinessStep` :49; `BuyDownFeeAmortizationBusinessStep` :53; `LoanInterestRecalculationCOBBusinessStep` :66
  - outside provider: `fineract-investor/src/main/java/org/apache/fineract/investor/cob/loan/LoanAccountOwnerTransferBusinessStep.java` (EXTERNAL_ASSET_OWNER_TRANSFER :303); `custom/acme/loan/cob/src/main/java/com/acme/fineract/loan/cob/AcmeNoopBusinessStep.java` (ACME_LOAN_NOOP, seeded at order 5 by `custom/acme/loan/cob/src/main/resources/db/custom-changelog/0001_acme_loan_cob.xml:28`)
  - unseeded steps (ACCRUAL_ACTIVITY_POSTING, BUY_DOWN_FEE_AMORTIZATION, CAPITALIZED_INCOME_AMORTIZATION, CHECK_DUE_INSTALLMENTS, LOAN_INTEREST_RECALCULATION, EXTERNAL_ASSET_OWNER_TRANSFER) run only once configured through `ConfigureBusinessStepApiResource`
- **e2e configuration** sets 12 steps in this order (`WorkFlowJobHelper.java:46-57`): APPLY_CHARGE_TO_OVERDUE_LOANS, LOAN_DELINQUENCY_CLASSIFICATION, CHECK_LOAN_REPAYMENT_DUE, CHECK_LOAN_REPAYMENT_OVERDUE, CHECK_DUE_INSTALLMENTS, UPDATE_LOAN_ARREARS_AGING, ADD_PERIODIC_ACCRUAL_ENTRIES, ACCRUAL_ACTIVITY_POSTING, CAPITALIZED_INCOME_AMORTIZATION, BUY_DOWN_FEE_AMORTIZATION, LOAN_INTEREST_RECALCULATION, EXTERNAL_ASSET_OWNER_TRANSFER
- **Inline COB**
  - `LoanCOBApiFilter` (OncePerRequestFilter) runs COB inline on loan API calls when the loan is behind and the user is not a bypass user (:79-80)
  - `LoanCOBFilterHelper`: LOAN_PATH_PATTERN :83, `isLoanBehind` :196, `executeInlineCob` :274
  - related: `InlineLoanCOBExecutorServiceImpl`, `LoanCOBCatchUpApiResource`, locks (`LoanAccountLock`, `LoanLockingServiceImpl`)
- **Standalone scheduler jobs** (`JobName.java`): APPLY_CHARGE_TO_OVERDUE_LOAN_INSTALLMENT :30, ADD_PERIODIC_ACCRUAL_ENTRIES :36, LOAN_COB :55, ACCRUAL_ACTIVITY_POSTING :60

### 1.5 Business and external events (overview; detail in §12)
- **Business events**: interface `BusinessEvent`, marker `NoExternalEvent` (same package); loan events in `fineract-loan/src/main/java/org/apache/fineract/infrastructure/event/business/domain/loan/` (102 files)
- **Dispatcher** `BusinessEventNotifierService` (`fineract-core/src/main/java/org/apache/fineract/infrastructure/event/business/service/`): pre / post notify, listener registration, start / stop / reset of external-event recording
- **External event written** only when all three hold (`BusinessEventNotifierServiceImpl.java:96-103`):
  - the event is not `NoExternalEvent` (:96)
  - `fineract.events.external.enabled` is true (`application.properties:124`, default false; impl :101, :142-144)
  - the type is enabled in `m_external_event_configuration` (:103)
- recorded events flushed as `BulkBusinessEvent` (:153-176)
- in-process listeners via `addPostBusinessEventListener`; example `LoanAccrualEventService.java:39-70`: on `LoanCloseBusinessEvent` or `LoanBalanceChangedBusinessEvent` with status CLOSED_OBLIGATIONS_MET or OVERPAID it runs `processAccrualsOnLoanClosure` and `processAccrualActivityForLoanClosure`
- **Startup guard** (`ExternalEventConfigurationValidationService.java:46-113`): unconditional `@Service` / `InitializingBean`; validates every tenant at startup whatever the external-events flag says (§12.3)

### 1.6 Commands, maker-checker, permissions
- every loan and charge write goes through the legacy command pipeline (§3.1)
- **New `fineract-command` framework** (Command, CommandHandler, CommandPipeline, CommandRouter, sync / async / disruptor executors, `CommandEntity`; `starter/CommandAutoConfiguration` through `AutoConfiguration.imports`): used only by business date, cache, external-event config, currency and notes; not by loans or charges
- **Handler registry**: `@CommandType(entity, action)` on `NewCommandSourceHandler` beans, keyed `entity|action` (`CommandHandlerProvider.java:62-69`); 435 annotated handler classes in main code; 56 with entity LOAN; 57 files in `fineract-loan/src/main/java/org/apache/fineract/portfolio/loanaccount/handler/`
  - LOANCHARGE actions: CREATE, UPDATE, DELETE, WAIVE, PAY, ADJUSTMENT, DEACTIVATEOVERDUE
- **Permission code** `ACTION_ENTITY` (`CommandWrapper.java:85, :297`); seeded in `parts/0002_initial_data.xml`: `REPAYMENT_LOAN` :5171, `CREATE_LOANCHARGE` :4891, `WAIVE_LOANCHARGE` :4939, `FORECLOSURE_LOAN` :9187, `CREATE_CHARGE` :3915
  - table `m_permission(grouping, code, entity_name, action_name, can_maker_checker)` (`Permission.java:28-43`); later seeds as parts, e.g. `parts/0011_add_credit_balance_refund_permission.xml`, loan `1006_add_downpayment_transaction_enum_permissions.xml`
  - check `validateHasPermissionTo(wrapper.getTaskPermissionName())` (`PortfolioCommandSourceWritePlatformServiceImpl.java:67`)
- **Maker-checker**
  - needs global config `maker-checker` and the permission's `can_maker_checker` (`ConfigurationDomainServiceJpa.java:48-62`); with the global flag on, a command with no `m_permission` row throws `PermissionNotFoundException`
  - `CommandSourceService.processCommand` (:133-152): checker or superuser gets markAsChecked; anyone else gets markAsAwaitingApproval and `RollbackTransactionNotApprovedException`
  - `approveEntry` :82-100; same-maker rule `validateMakerCheckerTransaction` :117-135 (`PortfolioCommandSourceWritePlatformServiceImpl`); API `MakercheckersApiResource`
- **Domain-rule vs validation errors**
  - `PlatformApiDataValidationException` (list of `ApiParameterError`): HTTP 400
  - `AbstractPlatformDomainRuleException` subclasses: HTTP 403 (`PlatformDomainRuleExceptionMapper.java:53`, `Status.FORBIDDEN`)

## 2. Repo layout

- modules from `settings.gradle:50-78` (29 includes); plus `:custom:docker` (:80) and dynamic `custom/<company>/<category>/<module>` modules (:81-94)
- `fineract-db/` is not a Gradle module (SQL demo backups)

| module | purpose | main / test `.java` |
|---|---|---|
| fineract-core | tenancy, ThreadLocal context, commands, events, Money, DateUtils, config; also ChargeTimeType, LoanStatus, AccountingConstants | 760 / 14 |
| fineract-security | auth filters (tenant-aware basic auth) | 52 / 0 |
| fineract-cob | COB framework (COBBusinessStep, step service) | 44 / 0 |
| fineract-validation | Jakarta constraints (EnumValue, LocalDate, Locale) | 6 / 2 |
| fineract-command | new command framework | 28 / 21 |
| fineract-accounting | GL accounts, rules, product-to-GL mapping domain | 146 / 0 |
| fineract-provider | the app: API resources, most service impls, Spring config, core changelog | 2,452 / 186 |
| fineract-branch | tellers (`organisation/teller`) | 52 / 0 |
| fineract-document | document management | 34 / 0 |
| fineract-investor | external asset owners (loan sale / buyback), its COB step | 98 / 10 |
| fineract-rates | floating rates | 21 / 0 |
| fineract-charge | charge definitions: `Charge`, ChargeCalculationType, validation, `ChargesApiResource` | 35 / 0 |
| fineract-tax | tax groups / components | 27 / 0 |
| fineract-loan-origination | loan originators | 40 / 3 |
| fineract-loan | loan domain, handlers, cumulative generators, 9 processors, loan changelog | 656 / 8 |
| fineract-savings | savings and deposits | 168 / 1 |
| fineract-report | report service SPI | 4 / 0 |
| fineract-war | WAR packaging | 0 / 0 |
| integration-tests | tests against a running server (RestAssured + generated client) | 0 / 421 |
| twofactor-tests, oauth2-tests | auth-mode tests | 0 / 1 each |
| fineract-client | client generated from OpenAPI (Retrofit) | 9 / 3 |
| fineract-client-feign | client generated from OpenAPI (Feign) | 16 / 12 |
| fineract-doc | asciidoc docs (186 files) | 0 / 0 |
| fineract-avro-schemas | external-event schemas: 81 `.avsc` under `src/main/avro` (82 in the module, counting `src/main/resources/avro-templates/bigdecimal.avsc`) | 1 / 0 |
| fineract-e2e-tests-core | Cucumber step defs, factories, helpers | 0 / 205 |
| fineract-e2e-tests-runner | Cucumber runner, 45 features, initializers | 0 / 20 |
| fineract-progressive-loan | progressive generator, EMI calculator, advanced processor | 73 / 13 |
| fineract-progressive-loan-embeddable-schedule-generator | shadowed standalone progressive generator | 1 (+ `misc/Main.java`) / 1 |
| custom/acme/* | samples: event, loan cob / job / processor, note (8 modules) | 21 total |

- **Totals**: 5,667 `.java` = 4,736 under `*/src/main/java` (custom samples included) + 930 under `*/src/test` + 1 `fineract-progressive-loan-embeddable-schedule-generator/misc/Main.java`
- **Where loan code sits**
  - `fineract-loan`: 410 files in `portfolio/loanaccount`, 63 in `portfolio/loanproduct`, 103 in `infrastructure/event`, 54 in `portfolio/delinquency`
  - `fineract-provider`: 138 files in `portfolio/loanaccount`, plus `cob/loan` and `accounting/journalentry`
  - `fineract-progressive-loan`: 50 files in `portfolio/loanaccount`, 19 in `portfolio/loanproduct`
  - `fineract-core`: LoanStatus, ChargeTimeType, accounting constants
- **Module dependencies** (`*/dependencies.gradle`)
  - core: avro-schemas, command, validation
  - cob: avro-schemas, command, core, validation
  - charge: core, tax; accounting: avro-schemas, charge, core
  - loan: core, cob, accounting, charge, rates, tax (`fineract-loan/dependencies.gradle:27-32`; comment at :22 says never use `api`)
  - progressive-loan: core, accounting, charge, loan, avro-schemas (`fineract-progressive-loan/dependencies.gradle:27-34`)
  - embeddable generator: core, loan, progressive-loan
  - investor: accounting, avro-schemas, charge, cob, core, loan
  - provider: every runtime module; war: provider plus the domain modules
  - integration-tests: accounting, avro-schemas, charge, client, client-feign, core, investor, loan, progressive-loan, provider, rates, savings, tax
  - e2e-tests-runner: avro-schemas, client, client-feign, e2e-tests-core (test output)
- **Consequences**: interfaces and entities sit low (loan, core, charge), most service impls in provider; `fineract-loan` cannot see provider or progressive-loan classes; progressive-loan can see loan

### 2.1 Path index (short names used in this file)
- `fineract-core/src/main/java/org/apache/fineract/`
  - `portfolio/charge/domain/ChargeTimeType.java`; `portfolio/charge/data/ChargeData.java`; `portfolio/loanaccount/domain/LoanStatus.java`; `portfolio/savings/PreClosurePenalInterestOnType.java`
  - `commands/service/`: `CommandWrapperBuilder`, `PortfolioCommandSourceWritePlatformServiceImpl`, `CommandSourceService`, `SynchronousCommandProcessingService`; `commands/provider/CommandHandlerProvider`; `commands/domain/CommandWrapper`
  - `accounting/common/`: `AccountingRuleType`, `AccountingConstants`
  - `infrastructure/event/business/domain/`: `BusinessEvent`, `NoExternalEvent`; `infrastructure/event/business/service/BusinessEventNotifierServiceImpl`
  - `infrastructure/event/external/service/ExternalEventConfigurationValidationService`; `infrastructure/event/external/repository/CustomExternalEventConfigurationRepositoryImpl`; `infrastructure/event/external/repository/domain/ExternalEvent`; `infrastructure/event/external/service/serialization/mapper/support/AvroMapperConfig`
  - `infrastructure/core/exceptionmapper/PlatformDomainRuleExceptionMapper`; `infrastructure/core/exception/PlatformApiDataValidationException`
  - `infrastructure/core/service/`: `DateUtils`, `MathUtil`, `ExternalIdFactory`, `ThreadLocalContextUtil`; `.../tenant/TenantMapper`; `.../database/DatabaseType`, `DatabaseTypeResolver`
  - `infrastructure/core/data/`: `DataValidatorBuilder`, `ApiParameterError`, `EnumOptionData`; `infrastructure/core/api/JsonCommand`; `infrastructure/core/serialization/FromJsonHelper`; `infrastructure/core/domain/ExternalId`, `ActionContext`; `infrastructure/core/config/MapstructMapperConfig`
  - `infrastructure/businessdate/domain/BusinessDateType`; `infrastructure/jobs/service/JobName`; `infrastructure/configuration/api/GlobalConfigurationConstants`; `infrastructure/configuration/service/MoneyHelperInitializationService`
  - `organisation/monetary/domain/`: `Money`, `MoneyHelper`; `useradministration/domain/Permission`
- `fineract-charge/src/main/java/org/apache/fineract/portfolio/charge/`: `domain/` (`Charge`, `ChargeCalculationType`, `ChargeAppliesTo`, `ChargePaymentMode`); `serialization/ChargeDefinitionCommandFromApiJsonDeserializer`; `exception/` (`ChargeMustBePenaltyException`, `ChargeDueAtDisbursementCannotBePenaltyException`, `LoanChargeCannotBeAddedException`); `service/` (`ChargeDropdownReadPlatformServiceImpl`, `ChargeEnumerations`); `api/` (`ChargesApiResource`, `ChargesApiResourceSwagger`); `request/ChargeRequest`
- `fineract-loan/src/main/java/org/apache/fineract/portfolio/loanaccount/`
  - `domain/`: `Loan`, `LoanCharge`, `LoanTransaction`, `LoanTransactionType`, `LoanTransactionRelation`, `LoanTransactionRelationTypeEnum`, `ChangedTransactionDetail`, `LoanOverdueInstallmentCharge`, `LoanRepaymentScheduleInstallment`, `LoanRepaymentScheduleProcessingWrapper`, `SingleLoanChargeRepaymentScheduleProcessingWrapper`, `DefaultLoanLifecycleStateMachine`, `LoanTransactionToRepaymentScheduleMapping`, `LoanChargeOffBehaviour`, `LoanSubStatus`, `LoanEvent`, `LoanTransactionComparator`, `LoanRepaymentScheduleTransactionProcessorFactory`, `LoanTransactionRepository`, `LoanDisbursementDetails`, `LoanInterestRecalculationDetails`; `domain/reaging/LoanReAgeInterestHandlingType`
  - `domain/transactionprocessor/`: `LoanRepaymentScheduleTransactionProcessor`, `AbstractLoanRepaymentScheduleTransactionProcessor`; `impl/` (9 strategies, e.g. `HeavensFamilyLoanRepaymentScheduleTransactionProcessor`)
  - `service/`: `LoanChargeService`, `LoanDownPaymentHandlerServiceImpl`, `LoanBalanceService`, `LoanScheduleService`, `ReprocessLoanTransactionsService`, `LoanTransactionService`, `LoanReadPlatformService`, `ReplayedTransactionBusinessEventServiceImpl`
  - `serialization/`: `LoanChargeApiJsonValidator`, `LoanChargeValidator`, `LoanDownPaymentTransactionValidator`
  - `loanschedule/domain/`: `AbstractCumulativeLoanScheduleGenerator`, `LoanScheduleGenerator`, `LoanScheduleGeneratorFactory`, `LoanApplicationTerms`, `LoanScheduleType`, `LoanScheduleProcessingType`, `LoanRepaymentScheduleHistory`
  - `rescheduleloan/data/LoanRescheduleRequestDataValidatorImpl`; `rescheduleloan/api/RescheduleLoansApiResource`
  - `handler/`: `LoanRepaymentCommandHandler`, `LoanRepaymentAdjustmentCommandHandler`, `ForeClosureCommandHandler`, `AddLoanChargeCommandHandler`, `LoanGoodwillCreditCommandHandler`, `LoanDownPaymentCommandHandler`, `LoanMerchantIssuedRefundCommandHandler`, `LoanPayoutRefundCommandHandler`, `LoanRecoveryPaymentCommandHandler` (`LoanInterestPaymentWaiverCommandHandler` is in provider `portfolio/loanaccount/handler/`)
  - `data/`: `LoanAccountData`, `AccountingBridgeDataDTO`; `api/`: `LoanApiConstants`, `LoanTransactionsApiResourceSwagger`, `LoanChargesApiResourceSwagger`
- `fineract-loan/src/main/java/org/apache/fineract/portfolio/loanproduct/`: `LoanProductConstants`; `domain/` (`LoanProduct`, `LoanProductRelatedDetail`, `PaymentAllocationType`, `PaymentAllocationTransactionType`, `DueType`, `AllocationType`, `FutureInstallmentAllocationRule`, `CreditAllocationTransactionType`, `LoanPreCloseInterestCalculationStrategy`, `LoanRescheduleStrategyMethod`, `InterestRecalculationCompoundingMethod`, `LoanProductInterestRecalculationDetails`); `service/LoanEnumerations`
- `fineract-loan/src/main/java/org/apache/fineract/portfolio/delinquency/service/LoanDelinquencyDomainServiceImpl`; `fineract-loan/src/main/java/org/apache/fineract/accounting/productaccountmapping/service/LoanProductToGLAccountMappingHelper`
- `fineract-progressive-loan/src/main/java/org/apache/fineract/portfolio/`: `loanaccount/domain/transactionprocessor/impl/` (`AdvancedPaymentScheduleTransactionProcessor`, `ChangeOperation`); `loanaccount/loanschedule/domain/ProgressiveLoanScheduleGenerator`; `loanproduct/calc/` (`ProgressiveEMICalculator`, `EMICalculator`); `loanaccount/rescheduleloan/data/ProgressiveLoanRescheduleRequestDataValidator`; `loanaccount/service/` (`CapitalizedIncomeWritePlatformServiceImpl`, `BuyDownFeeWritePlatformServiceImpl`)
- `fineract-provider/src/main/java/org/apache/fineract/`
  - `portfolio/loanaccount/api/`: `LoanTransactionsApiResource`, `LoansApiResource`, `LoanChargesApiResource`, `LoansApiResourceSwagger`
  - `portfolio/loanaccount/service/`: `LoanWritePlatformServiceJpaRepositoryImpl`, `LoanReadPlatformServiceImpl`, `LoanChargeWritePlatformServiceImpl`, `LoanChargeReadPlatformServiceImpl`, `LoanChargeAssembler`, `LoanAssemblerImpl`, `LoanTransactionProcessingServiceImpl`, `ReprocessLoanTransactionsServiceImpl`, `LoanAccrualsProcessingServiceImpl`, `LoanAccrualActivityProcessingServiceImpl`, `LoanDisbursementService`, `LoanAccrualEventService`, `LoanJournalEntryPosterImpl`, `LoanAccountServiceImpl`, `LoanRepaymentScheduleService`, `LoanUtilService`; `adjustment/LoanAdjustmentServiceImpl`; `contracttermination/LoanContractTerminationServiceImpl`; `reaging/` (`LoanReAgingService`, `LoanReAgingValidator`); `reamortization/` (`LoanReAmortizationService`, `LoanReAmortizationValidator`)
  - `portfolio/loanaccount/domain/`: `LoanAccountDomainServiceJpa`, `LoanAccountDomainServiceJpaHelper`
  - `portfolio/loanaccount/serialization/`: `LoanForeclosureValidator`, `LoanTransactionValidatorImpl`, `LoanApplicationValidator`
  - `portfolio/loanaccount/loanschedule/service/`: `LoanScheduleAssembler`, `LoanScheduleCalculationPlatformServiceImpl`; `portfolio/loanaccount/loanschedule/domain/DefaultLoanScheduleGeneratorFactory`
  - `portfolio/loanaccount/rescheduleloan/service/LoanRescheduleRequestWritePlatformServiceImpl`
  - `portfolio/loanaccount/starter/`: `LoanAccountConfiguration`, `LoanAccountAutoStarter`
  - `portfolio/loanaccount/jobs/`: `applyholidaystoloans/ApplyHolidaysToLoansTasklet`, `applychargetooverdueloaninstallment/ApplyChargeToOverdueLoanInstallmentTasklet`
  - `portfolio/loanproduct/serialization/LoanProductDataValidator`; `portfolio/loanproduct/service/LoanProductWritePlatformServiceJpaRepositoryImpl`
  - `portfolio/charge/service/`: `ChargeWritePlatformServiceJpaRepositoryImpl`, `ChargeReadPlatformServiceImpl`; `portfolio/savings/service/DepositAccountReadPlatformServiceImpl`
  - `accounting/journalentry/service/`: `CashBasedAccountingProcessorForLoan`, `AccrualBasedAccountingProcessorForLoan`, `AccountingProcessorHelper`, `AccountingProcessorForLoanFactory`, `JournalEntryWritePlatformServiceJpaRepositoryImpl`
  - `cob/loan/`: the 11 `*BusinessStep` classes, `LoanCOBConstant`, `LoanCOBManagerConfiguration`, `LoanCOBWorkerConfiguration`
  - `infrastructure/core/boot/FineractWebApplicationConfiguration`; `infrastructure/core/service/migration/TenantDatabaseUpgradeService`; `infrastructure/configuration/domain/ConfigurationDomainServiceJpa`; `infrastructure/jobs/filter/` (`LoanCOBApiFilter`, `LoanCOBFilterHelper`); `infrastructure/event/external/service/serialization/serializer/loan/` (`LoanChargeBusinessEventSerializer`, `LoanBusinessEventSerializer`, `LoanTransactionBusinessEventSerializer`)
- `fineract-security/src/main/java/org/apache/fineract/infrastructure/security/filter/TenantAwareBasicAuthenticationFilter`; `fineract-cob/src/main/java/org/apache/fineract/cob/COBBusinessStepServiceImpl`; `fineract-savings/src/main/java/org/apache/fineract/portfolio/savings/domain/DepositPreClosureDetail`; `fineract-investor/src/main/java/org/apache/fineract/investor/service/serialization/serializer/investor/InvestorBusinessEventSerializer`
- tests: `integration-tests/src/test/java/org/apache/fineract/integrationtests/` (`BaseLoanIntegrationTest`, `ConfigProperties`, `ExternalEventConfigurationIntegrationTest`; `common/` `Utils`, `FineractClientHelper`, `ExternalEventConfigurationHelper`, `BusinessDateHelper`, `SchedulerJobHelper`, `GlobalConfigurationHelper`; `common/charges/ChargesHelper`; `common/loans/` `LoanTransactionHelper`, `LoanProductTestBuilder`, `LoanTestLifecycleExtension`; `common/accounting/JournalEntryHelper`); `fineract-e2e-tests-core/src/test/java/org/apache/fineract/test/helper/WorkFlowJobHelper`
- **Same file name twice** (tracked): `ChargeTimeType`, `ChargeCalculationType`, `LoanStatus` (e2e copies in `fineract-e2e-tests-core/src/test/java/org/apache/fineract/test/data/`), `LoanProduct` (e2e copy in `.../test/data/loanproduct/`), `InterestRecalculationCompoundingMethod` (e2e copy), `ExternalIdConverter` (investor and provider), `Utils` (integration-tests `common/` and e2e `helper/`); short names here mean the main-code file (integration-tests file for `Utils`)

## 3. Command write path and read path

### 3.1 Write path: a loan repayment
1. **API** `LoanTransactionsApiResource` (834 lines): `@Path("/v1/loans")` :87; POST `{loanId}/transactions?command=...` :283-309; private `executeTransaction` :558-621; `command=repayment` :566-567 builds `new CommandWrapperBuilder().loanRepaymentTransaction(loanId)`; `logCommandSource` :619; unknown command: `UnrecognizedQueryParamException` :616-617
2. **Wrapper** `CommandWrapperBuilder.loanRepaymentTransaction` :840-847: action `REPAYMENT`, entity `LOAN`, href `/loans/{id}/transactions/template?command=repayment`
3. **Command source** `PortfolioCommandSourceWritePlatformServiceImpl.logCommandSource` :54-79: permission check :67 (`REPAYMENT_LOAN`), `validateIsUpdateAllowed`, `JsonCommand.from`, `executeCommand`
4. **Processing** `SynchronousCommandProcessingService`: resilience4j retry :91; idempotency; `saveInitialNewTransaction` :128; `processCommand` :147; `findCommandHandler` :254-318 calls `commandHandlerProvider.getHandler(entity, action)` :315
5. **Transaction boundary** `CommandSourceService.processCommand` :133-152: `@Transactional`, `handler.processCommand`, maker-checker branch (§1.6)
6. **Handler** `LoanRepaymentCommandHandler` (`@Service`, `@CommandType(entity = "LOAN", action = "REPAYMENT")`): `writePlatformService.makeLoanRepayment(LoanTransactionType.REPAYMENT, loanId, command, false)`; maps `JpaSystemException` / `DataIntegrityViolationException` to "loan.repayment"
7. **Write service** `LoanWritePlatformServiceJpaRepositoryImpl` (3,651 lines): `makeLoanRepayment` :936-941 calls `makeLoanRepaymentWithChargeRefundChargeType` :1074-1119 (validators; `loanAssembler.assembleFrom` :1098; `loanAccountDomainService.makeRepayment` :1104; result builder :1110-1118); type check `LoanUtilService.validateRepaymentTransactionType` :301-307
8. **Domain service** `LoanAccountDomainServiceJpa.makeRepayment` :216-331 (public overload :166-172)
   - pre event :222-223; `LoanTransaction.repaymentType` :240 (transaction built :236-242); recalculateFrom :244-247; `ScheduleGeneratorDTO` :248-255; `calculateRecalculateTillDate` :251-252
   - holiday / non-working-day validation :257-263; `validateActivityNotBeforeLastTransactionDate` :265
   - private `makeRepayment` :269 (body :1017-1021) calls `loanDownPaymentHandlerService.handleRepaymentOrRecoveryOrWaiverTransaction` at :1020
   - interest-recalc accruals :271-274; save transaction :276 and loan :277; `processAccrualsOnInterestRecalculation` :284; delinquency tag :287; journal :289; post events `LoanBalanceChangedBusinessEvent` plus the type-specific one (e.g. `LoanTransactionMakeRepaymentPostBusinessEvent`, `LoanChargePaymentPostBusinessEvent`) :290-295; post-dated checks :301-328
   - no charge-off guard in this method; a repayment dated before the charge-off date is accepted and replayed (§13.4)
9. **Allocation** `LoanDownPaymentHandlerServiceImpl.handleRepaymentOrRecoveryOrWaiverTransaction` :82-185
   - recovery amount cap :88-92; transaction added when non-zero :99-101; type check :103-107; future date :111-114
   - fast-path flag :135-139 (§6.6); fast path `processLatestTransaction` with a `TransactionCtx` built from `loan.getActiveCharges()` :140-143
   - cumulative + interest recalculation after the fast path: replay anyway when the current installment is not fully paid, or the next installment has paid-in-advance > 0 (:144-154)
   - otherwise regenerate (cumulative + recalc: `regenerateRepaymentScheduleWithInterestRecalculation`; progressive: `regenerateRepaymentSchedule`) and `reprocessTransactions` (:156-163)
   - `determineAndTransition` (:168-172; recovery repayments only refresh the summary)
   - multi-disbursement check "amount.exceeds.threshold" (:174-184)
10. **Persistence** `LoanAccountServiceImpl.java:42-54`: `saveAndFlush` on `LoanTransactionRepository` and `LoanRepositoryWrapper`
11. **Journal** `LoanJournalEntryPoster` (interface in fineract-loan), `LoanJournalEntryPosterImpl` :36-40, then `JournalEntryWritePlatformServiceJpaRepositoryImpl.createJournalEntriesForLoanTransaction` :813-831 (§11)

- **All POST commands** of `executeTransaction` (:566-612), values as sent: `repayment`, `merchantIssuedRefund`, `payoutRefund`, `goodwillCredit`, `interestPaymentWaiver`, `chargeRefund`, `waiveinterest`, `writeoff`, `close-rescheduled`, `close`, `undowriteoff` (:586), `recoverypayment`, `refundByCash`, `foreclosure` (:592-593), `creditBalanceRefund`, `charge-off` / `undo-charge-off` (constants `LoanTransactionsApiResource.java:93-94`, dispatched :596-599), `downPayment`, `reAge`, `undoReAge`, `reAmortize`, `undoReAmortize`, `capitalizedIncome`, `buyDownFee`
  - `chargeOff()` / `undoChargeOff()` are only `CommandWrapperBuilder` method names
- **No POST `prepayLoan`**: a prepayment is POST `command=repayment`; `prepayLoan` exists only as a GET template (§8). Stale `fineract-provider/src/main/resources/static/legacy-docs/apiLive.htm:13068-13073` still documents POST `?command=prepayLoan` (the server rejects it with `UnrecognizedQueryParamException`); `CommandWrapperBuilder` has no prepay builder
- **Foreclosure command**: `CommandWrapperBuilder.loanForeclosure` :975; `ForeClosureCommandHandler` :31 / :38; `LoanWritePlatformServiceJpaRepositoryImpl.forecloseLoan` :2645; `LoanAccountDomainServiceJpa.foreCloseLoan` :702-768 (§8.4)
- **Charge commands**: `createLoanCharge` :774, `waiveLoanCharge` :791 (`CommandWrapperBuilder`); adjust `adjustTransaction` :948 (§10)

### 3.2 Read path
- **Pattern**: `*ApiResource` GET, then `*ReadPlatformService(Impl)`, then `JdbcTemplate` with inner `RowMapper` classes building SQL from `schema()` strings; some reads load JPA entities instead
- **GET loan** `LoansApiResource` (`@Path("/v1/loans")` :191; `{loanId}` :463-480; private `retrieveLoan` :995): `loanReadPlatformService.retrieveOne` :1000; associations: transactions :1079, charges through `LoanChargeReadPlatformService.retrieveLoanCharges` :1133, `futureSchedule` :1114-1117 (§8.2)
- **`LoanReadPlatformServiceImpl`** (2,322 lines; `@Transactional(readOnly = true)` :163; built as a `@Bean` in `LoanAccountConfiguration`)
  - `retrieveOne` :202-220: `LoanMapper` (inner `RowMapper<LoanAccountData>` :765), office-hierarchy filter, `jdbcTemplate.queryForObject`
  - `retrieveAll` :326 with `PaginationHelper.fetchPage` :403
  - `retrieveLoanTransactions` :285-317: `LoanTransactionsMapper` :1329; relations and charge-paid-by fetched separately
  - schedule `retrieveRepaymentSchedule` :275 through `LoanRepaymentScheduleService.findLoanScheduleData`
  - other mappers: `LoanCurrencyDataMapper` :738, `MusoniOverdueLoanScheduleMapper` :1284, `LoanDisbursementDetailMapper` :1660, `CurrencyMapper` :2102, `RepaymentTransactionTemplateMapper` :2117
  - interface `LoanReadPlatformService` (`retrieveLoanPrePaymentTemplate` :122)
- **Transaction templates**: GET `{loanId}/transactions/template?command=...` (`LoanTransactionsApiResource` :118-146, `retrieveTransactionTemplate` :623-720)
  - MIR, payout refund, goodwill credit, interest payment waiver templates call `retrieveLoanPrePaymentTemplate` at the business date (:634-649)
  - `prepayLoan` and `contractTermination` call `retrieveLoanPrePaymentTemplate(LoanTransactionType.REPAYMENT, loanId, date)` (:668-677; date param or business date)
  - `foreclosure` calls `retrieveLoanForeclosureTemplate` (:682-689)
  - other template commands: repayment, waiveinterest, writeoff, close-rescheduled, close, disburse, disburseToSavings, recoverypayment, refundbycash, refundbytransfer, creditBalanceRefund, charge-off, downPayment, capitalizedIncome (+ adjustment), buyDownFee (+ adjustment), interest-refund, reAge, reAmortization
- **Overdue installments** for the COB charge step: `retrieveAllOverdueInstallmentsForLoan` :1591-1630

## 4. Loan domain model

- **`Loan`** (1,854 lines): `@Table m_loan` :88; extends `AbstractAuditableWithUTCDateTimeCustom` :91; `@Getter`
  - columns: account_no :118, external_id :122, `loan_transaction_strategy_code` :159, approved_principal :325, is_npa :358, accrued_till :362, is_fraud :395, last_closed_business_date :410, is_charged_off :413, charged_off_on_date :420
  - collections: `paymentAllocationRules` :167, `creditAllocationRules` :172 (both TODO FINERACT-1932); `charges` :293; installments ordered by installmentNumber; transactions ordered by "dateOf, createdDate, id"
  - embedded `LoanProductRelatedDetail loanRepaymentScheduleDetail` :175; `LoanSummary`
  - status `loan_status_id` through `LoanStatusConverter` :188; `loan_sub_status_id` :389
  - methods
    - `getDisbursementDate` :973-979 (actual date when set; earliest tranche, §13.2); `hasChargesAffectedByBackdatedRepaymentLikeTransaction` :1258-1272 (§6.6)
    - `isInterestBearing` :1364-1369 (annual nominal rate > 0, or progressive with a positive interest-rate term variation); `isInterestBearingAndInterestRecalculationEnabled` :1371-1373; `isChargeOffOnDate` :1819-1822 (charge-off transaction dated on or before the date)
    - `getLoanCharges(predicate)` :1496-1498; `getLoanCharges()` :1659-1661; `getLastLoanRepaymentScheduleInstallment` :1747
    - `isUpfrontAccrualAccountingEnabledOnLoanProduct` :1149; `isNoneOrCashOrUpfrontAccrualAccountingEnabledOnLoanProduct` :1157
- **`LoanRepaymentScheduleInstallment`** (1,292 lines), `m_loan_repayment_schedule` :52
  - `installment` :60, `fromdate`, `duedate` :66
  - principal amount / completed / writtenoff; interest amount / completed / waived / writtenoff, `accrual_interest_derived`
  - fees `fee_charges_amount` / completed / writtenoff / waived, `accrual_fee_charges_derived`
  - penalties `penalty_charges_amount` :111, completed / writtenoff / waived, `accrual_penalty_charges_derived` :123
  - `total_paid_in_advance_derived` :126, `total_paid_late_derived`, `completed_derived` :132, `obligations_met_on_date`
  - flags `recalculated_interest_component`, `is_additional`, `is_down_payment`, `is_re_aged`; credits `credits_amount`, `credited_interest/fee/penalty`
  - methods `pay*Component`, `waive*Component`, `updateAccrualPortion`, `getPaymentFunction(AllocationType, PaymentAction)`
- **`LoanTransaction`** (1,045 lines), `m_loan_transaction` :61
  - `transaction_type_enum` :79, `transaction_date` :82, `amount` :89; `principal/interest/fee_charges/penalty_charges/overpayment_portion_derived` (penalty :101), `unrecognized_income_portion`
  - `is_reversed` :110, `external_id`, `reversal_external_id`, `reversed_on_date`, `outstanding_loan_balance_derived`, `manually_adjusted_or_reversed`, `charge_refund_charge_type`, `classification_cv_id`
  - collections: `loanChargesPaid` :123, schedule mappings :139, relations :142-143 (OneToMany mappedBy fromTransaction, cascade ALL, orphanRemoval)
  - factories `repaymentType`, `recoveryRepayment`, `accrueLoanCharge`, `interestRefund`; `copyTransactionProperties` :313-330 (no relations); `transactionAmountsMatch` :417-425; `isRepaymentLikeType` :646-650; `isTypeAllowedForChargeback` :652-655; `isPaymentTransaction` :969-972; `hasChargebackLoanTransactionRelations` :978-980; `copyLoanTransactionRelations` :982-987
- **`LoanTransactionType`** (`LoanTransactionType.java:23-78`): 43 constants including INVALID, ids 0-43, id 11 unused; next free id 44
  - REPAYMENT 2, WAIVE_CHARGES 9, ACCRUAL 10, CHARGE_PAYMENT 17, CHARGEBACK 25, CHARGE_ADJUSTMENT 26, CHARGE_OFF 27, DOWN_PAYMENT 28, REAGE 29, REAMORTIZE 30, ACCRUAL_ACTIVITY 32, ACCRUAL_ADJUSTMENT 34, CONTRACT_TERMINATION 38, BUY_DOWN_FEE* 40-43
  - `isRepaymentType` (:182-185), 7 types: REPAYMENT, MERCHANT_ISSUED_REFUND, PAYOUT_REFUND, GOODWILL_CREDIT, CHARGE_REFUND, DOWN_PAYMENT, INTEREST_PAYMENT_WAIVER
  - `LoanTransaction.isRepaymentLikeType` (:646-650), 10 types: the 7 above plus CHARGE_ADJUSTMENT, INTEREST_REFUND, CAPITALIZED_INCOME_ADJUSTMENT
  - RECOVERY_REPAYMENT is in neither set; only on written-off loans (`LoanDownPaymentTransactionValidator.java:173-175`)
- **`LoanCharge`** (728 lines), `m_loan_charge` :61, `@Getter @Setter` :58-59
  - `charge_time_enum` :72, `due_for_collection_as_of_date` :78, `charge_calculation_enum`, `charge_payment_mode_enum`, `calculation_percentage`, `calculation_on_amount` (`amountPercentageAppliedTo`), `charge_amount_or_percentage`, `amount` :96
  - paid / waived / writtenoff / outstanding derived (outstanding :108), `is_penalty` :111, `is_paid_derived`, `waived`, `min_cap` :122, `max_cap` :125, `is_active`, `external_id`, `submitted_on_date`
  - collections `loanInstallmentCharge` :128, `overdueInstallmentCharge` :137, `loanTrancheDisbursementCharge`, `loanChargePaidBySet` :143
  - `resetToOriginal` :156-166 (clears paid and waived); `resetPaidAmount` :168-175 (keeps waived); `waive` :181-203 (waives the amount outstanding at that moment); `undoWaive` :205; `isOverdueInstallmentCharge` :245-246
  - `percentageOf` :292-302 (uses `MoneyHelper.getMathContext()` at :297); `minimumAndMaximumCap` :309-331: min checked first and returned unrounded, then max returned unrounded, an in-range value rounded through `Money.of` (:328); with min > max the min wins
- **Charge links**
  - `LoanInstallmentCharge` (`m_loan_installment_charge`): loan_charge_id, loan_schedule_id, amount and paid / waived / writtenoff / outstanding derived, `amount_through_charge_payment`, `is_paid_derived`, `waived`
  - `LoanOverdueInstallmentCharge` (`m_loan_overdue_installment_charge`): overdue charge to installment
  - `LoanChargePaidBy` (`m_loan_charge_paid_by`): loan_transaction_id, loan_charge_id, amount, installment_number
  - `LoanTransactionToRepaymentScheduleMapping` (`m_loan_transaction_repayment_schedule_mapping`): transaction, installment, the four portions, amount; the only per-installment record of what a transaction settled
- **`LoanTransactionRelation`** (`m_loan_transaction_relation` :35)
  - from :38-40; to transaction :42-45; `to_loan_charge_id` :47-49 (a transaction can already point at a loan charge)
  - `relation_type_enum` stored `@Enumerated(EnumType.ORDINAL)` :51-53
  - `linkToTransaction` :65-70 adds itself to the transaction's relation set; `linkToCharge` :72-75 does not
  - `LoanTransactionRelationTypeEnum` (:23-28): INVALID 0, CHARGEBACK 1, CHARGE_ADJUSTMENT 2, REPLAYED 3, RELATED 4, ADJUSTMENT 5; ordinal equals value; `fromInt` (:47-59) maps only 1-3; a new value must be appended last
  - usages
    - CHARGEBACK `LoanTransaction.java:978-980`, `LoanWritePlatformServiceJpaRepositoryImpl.java:1316, :1352`
    - CHARGE_ADJUSTMENT `LoanChargeWritePlatformServiceImpl.java:869-870` (`linkToCharge`), `JournalEntryWritePlatformServiceJpaRepositoryImpl.java:988`
    - REPLAYED abstract processor :505, `LoanAccrualActivityProcessingServiceImpl.java:177, :361`, advanced :1465-1476
    - RELATED `LoanAccountDomainServiceJpa.java:837`, `LoanWritePlatformServiceJpaRepositoryImpl.java:2990`, `LoanAdjustmentServiceImpl.java:324`, `LoanBuyDownFeeAmortizationProcessingServiceImpl.java:193, :268`, `LoanCapitalizedIncomeAmortizationProcessingServiceImpl.java:174, :194`
    - ADJUSTMENT `BuyDownFeeWritePlatformServiceImpl.java:160`, `CapitalizedIncomeWritePlatformServiceImpl.java:150`, `LoanTransactionRepository.java:377`
  - indexes on the relation table, `to_loan_charge_id` included, already exist (core parts 0115, 0157)
- **`LoanProduct`** (771 lines), `m_product_loan` :86: strategy code and name; payment and credit allocation rules (EAGER); `charges` `@ManyToMany` EAGER through `m_product_loan_charge(product_loan_id, charge_id)` :120-122; rates; `LoanProductRelatedDetail`, `LoanProductMinMaxConstraints`; `accounting_type`; tranche and interest-recalculation details
- **`LoanProductRelatedDetail`** (`@Embeddable` :53, shared by Loan and LoanProduct): currency, principal, interest, repayment and amortization fields, days in month / year, `interest_recalculation_enabled`, down-payment fields, `loan_schedule_type` (STRING :151-153), `loan_schedule_processing_type` (STRING :155-157), `enable_accrual_activity_posting`, `charge_off_behaviour` (STRING :166-168), `interest_recognition_on_disbursement_date`, capitalized-income fields
- **Status lifecycle**
  - `LoanStatus` (`LoanStatus.java:26-37`): INVALID 0, SUBMITTED_AND_PENDING_APPROVAL 100, APPROVED 200, ACTIVE 300, TRANSFER_IN_PROGRESS 303, TRANSFER_ON_HOLD 304, WITHDRAWN_BY_CLIENT 400, REJECTED 500, CLOSED_OBLIGATIONS_MET 600, CLOSED_WRITTEN_OFF 601, CLOSED_RESCHEDULE_OUTSTANDING_AMOUNT 602, OVERPAID 700
  - `LoanSubStatus` (:25-27): INVALID 0, FORECLOSED 100, CONTRACT_TERMINATION 900; foreclosure sets FORECLOSED (`LoanAccountDomainServiceJpa.java:1023-1027`)
  - `LoanEvent` (:26-55, 30 values) incl. LOAN_REPAYMENT_OR_WAIVER, REPAID_IN_FULL, LOAN_FORECLOSURE, LOAN_CHARGE_ADDED, LOAN_CHARGE_PAYMENT, LOAN_ADJUST_TRANSACTION, LOAN_OVERPAYMENT
  - state machine `DefaultLoanLifecycleStateMachine` (383 lines): transitions :119-225; LOAN_CHARGE_ADDED can reopen CLOSED_OBLIGATIONS_MET (:215-219); `determineAndTransition` :55 calls `determineTransition` :299-370
  - closure rule: "all charges paid" = inactive, amount <= 0, paid or waived, over raw `getLoanCharges` (:303-304); overpaid flag from `loan.getTotalOverpaid()` (:302); result ACTIVE, CLOSED_OBLIGATIONS_MET or OVERPAID

## 5. Schedule generation

- **Interface** `LoanScheduleGenerator`: `generate`, `rescheduleNextInstallments` (2 overloads), `calculatePrepaymentAmount` (:48), `getPeriodInterestTillDate`
- **Factory** `LoanScheduleGeneratorFactory` (interface, fineract-loan); impl `DefaultLoanScheduleGeneratorFactory` (`@Component` :27)
  - CUMULATIVE: flat or declining-balance generator by `InterestMethod` (INVALID gives null)
  - PROGRESSIVE: always the progressive generator
- **Product choice** (both stored on `LoanProductRelatedDetail`)
  - `loanScheduleType` CUMULATIVE | PROGRESSIVE (`LoanScheduleType.java:31-32`)
  - `loanScheduleProcessingType` HORIZONTAL | VERTICAL (`LoanScheduleProcessingType.java:31-32`)
  - columns from loan parts `1010_introduce_loan_schedule_type_configuration` and `1012_introduce_loan_schedule_processing_type_configuration`
- **Pairing rule** `LoanProductDataValidator.validateLoanScheduleType` (:2800-2825; checks :2810-2823): PROGRESSIVE requires `advanced-payment-allocation-strategy` and that strategy requires PROGRESSIVE; cumulative loans use the 9 legacy processors, progressive loans only the advanced one (§6)
- **Cumulative** `AbstractCumulativeLoanScheduleGenerator` (2,859 lines, class :70)
  - subclasses `CumulativeFlatInterestLoanScheduleGenerator` (:34), `CumulativeDecliningBalanceInterestLoanScheduleGenerator` (:60)
  - with interest recalculation it replays real transactions through the processor's `handleRepaymentSchedule` while regenerating (:544, :888-889, :1448, :2592)
  - early payments folded into the current period: `updateAmountsBasedOnCurrentEarlyPayments` :679-700 (early-paid amount added to `principalForThisPeriod` at :695); then `applyEarlyPaymentStrategy` :1552-1575 (§13.1)
  - charges per period: `cumulativeFeeChargesDueWithin` :2085, `cumulativePenaltyChargesDueWithin` :2146-2175 (overdue special cases :2097, :2159)
  - tranche amounts: `getDisbursementAmount` :1979 (future undisbursed tranches in rows, §13.2)
  - `calculatePrepaymentAmount` :2791-2823 (§8.1); `getPeriodInterestTillDate` :2826 returns null for additional, down-payment and re-aged installments (:2827)
- **Progressive** `ProgressiveLoanScheduleGenerator` (506 lines, :61): `calculatePrepaymentAmount` :184-239 (§8.1); charges per period :385 and :418 (overdue % case :407); `EmbeddableProgressiveLoanScheduleGenerator` in the embeddable module
- **EMI** `EMICalculator` (interface), `ProgressiveEMICalculator` (2,066 lines, final, :72); model `ProgressiveLoanInterestScheduleModel` with `InterestPeriod` / `RepaymentPeriod` (`calc/data/`); saved per loan through `interestScheduleModelRepositoryWrapper` (`ReprocessLoanTransactionsServiceImpl.java:95`, :127)
- **`fixedLength`** is in repayment-frequency units, not days: last due date = start plus `fixedLength` (+ variation days) days, weeks, months or years by frequency (`LoanApplicationTerms.java:2229-2253`; `ProgressiveEMICalculator.java:896-900`; `DefaultScheduledDateGenerator.java:62, :108`)
- **Pre-close config** `LoanPreCloseInterestCalculationStrategy` NONE 0, TILL_PRE_CLOSURE_DATE 1, TILL_REST_FREQUENCY_DATE 2 (`LoanPreCloseInterestCalculationStrategy.java:26-28`; `calculateTillRestFrequencyEnabled` :83, `calculateTillPreClosureDateEnabled` :86-87); product value validated 0-2 (`LoanProductDataValidator.java:1310-1314`), so NONE is a valid product setting
- **Installment kinds** a penalty rule must treat explicitly: down payment (`is_down_payment`, due on the disbursement date), additional N+1 (`is_additional`, e.g. charge after maturity), re-aged (`is_re_aged`), recalculated-interest component; progressive replay deletes re-aged and additional rows and recreates them (`AdvancedPaymentScheduleTransactionProcessor.java:217-218`)

## 6. Repayment transaction processing

### 6.1 Interface, base class, wiring
- **Interface** `LoanRepaymentScheduleTransactionProcessor`; contexts `TransactionCtx`, `MoneyHolder` (same package); `ProgressiveTransactionCtx` (progressive-loan)
- **Base** `AbstractLoanRepaymentScheduleTransactionProcessor` (995 lines)
  - `accept` matches by code or name (:85-87)
  - `reprocessLoanTransactions` :90-232 (§9.2)
  - `processLatestTransaction` :235-260: WRITEOFF, REFUND_FOR_ACTIVE_LOAN, CHARGEBACK, CHARGE_OFF have own handlers (:237-240); every other type goes to `handleTransactionAndCharges`; leftover becomes overpayment (`onLoanOverpayment` :249), zeroed for waivers
  - `processTransaction` :627-658: loops installments in due-date order; for each one not fully paid picks the in-advance (:293), late (:315) or on-time (:304) handler; refund handler :338
  - `isTransactionInAdvanceOfInstallment` :348-352 (transaction date before the installment due date); `isTransactionALateRepaymentOnInstallment` :361-365 (after the due date)
  - `handleTransactionAndCharges` :591-625; `extractPenaltyCharges` :670; `updateChargesPaidAmountBy` :680-722 records paid amounts on `LoanCharge` rows, earliest unpaid first (`findEarliestUnpaidChargeFromUnOrderedSet` :736)
  - RECOVERY_REPAYMENT branches: :181, :267, :594
- **Wiring** `LoanAccountAutoStarter`
  - each processor is `@Bean @Conditional(<X>Condition)`; conditions in `fineract-loan/.../loanaccount/starter/*Condition.java`, advanced in progressive-loan
  - switches `fineract.loan.transactionprocessor.<x>.enabled` (`application.properties:165-174`) and `...error-not-found-fail` (:175)
  - factory bean `@ConditionalOnMissingBean` :128-133; default processor PrincipalInterestPenaltyFeesOrder
  - advanced bean :137-146 (EMICalculator, interest-refund service @Lazy, ExternalIdFactory, LoanScheduleComponent, LoanChargeValidator, LoanBalanceService, LoanChargeService @Lazy, ScheduledDateGenerator)
- **Factory** `LoanRepaymentScheduleTransactionProcessorFactory`: `determineProcessor` (throws `LoanTransactionProcessingStrategyNotFoundException` when fail-on-not-found is set), `getStrategies`; loan keeps the code in `m_loan.loan_transaction_strategy_code`

### 6.2 Every processor
- **Table key**: class names without the suffix `LoanRepaymentScheduleTransactionProcessor`; the 9 cumulative ones in `fineract-loan/.../domain/transactionprocessor/impl/`; "delegates" = handler calls the on-time handler

| # | class (lines) | code | in advance | late | on-time order |
|---|---|---|---|---|---|
| 1 | FineractStyle (190) | `mifos-standard-strategy` (:48) | delegates (:75) | delegates (:88) | penalty, fee, interest, principal (:132-141) |
| 2 | HeavensFamily (265) | `heavensfamily-strategy` (:50) | own rule and handler (below) | delegates (:72-80) | principal, interest, fee, penalty (:205-214) |
| 3 | Creocore (188) | `creocore-strategy` (:48) | delegates (:76) | delegates (:90) | penalty, fee, interest, principal (:132-141) |
| 4 | RBI (302) | `rbi-india-strategy` (:52) | delegates (:80) | own: penalty, fee, interest across overdue and current installments, then principal (:88-183) | penalty, fee, interest, principal (:235-244) |
| 5 | EarlyPayment (229) | `early-repayment-strategy` (:42) | own: principal only (:67-116, principal :105) | delegates (:128) | interest, principal, penalty, fee (:166-175) |
| 6 | PrincipalInterestPenaltyFeesOrder (194) | `principal-interest-penalties-fees-order-strategy` (:43) | delegates (:72) | delegates (:86) | principal, interest, penalty, fee (:130-139) |
| 7 | InterestPrincipalPenaltyFeesOrder (195) | `interest-principal-penalties-fees-order-strategy` (:43) | delegates (:72) | delegates (:86) | interest, principal, penalty, fee (:130-139) |
| 8 | DuePenFeeIntPriInAdvancePriPenFeeInt (294) | `due-penalty-fee-interest-principal-in-advance-principal-penalty-fee-interest-strategy` (:49) | own (:72-176, below) | delegates (:187) | penalty, fee, interest, principal (:232-244) |
| 9 | DuePenIntPriFeeInAdvancePenIntPriFee (294) | `due-penalty-interest-principal-fee-in-advance-penalty-interest-principal-fee-strategy` (:49) | own (:72-176) | delegates (:187) | penalty, interest, principal, fee (:232-244) |
| 10 | AdvancedPaymentScheduleTransactionProcessor (4,114, progressive-loan) | `advanced-payment-allocation-strategy` (:133), extends the base (:131) | configurable (§6.3) | configurable | configurable |

- **Charge-payment and waiver branches**: each on-time handler pays only the penalty or fee component for CHARGE_PAYMENT and only interest for interest waivers (e.g. FineractStyle :118-127)
- **HeavensFamily**
  - in advance = before the previous installment's due date (`isTransactionInAdvanceOfInstallment` override :83-97); a payment inside the current period is on time
  - in-advance handler :103-163: interest-waiver branch, charge-payment branch, else principal first; once installment principal completes it waives the whole installment interest (FIXME :133-135, no waiver transaction); the rest pays principal, interest, fee, penalty (:143-154)
- **`DuePen*` in-advance handlers**: first sum active pending charges with effective due date on or before the transaction date (:112-125); pay penalties and fees up to that sum, interest only from the due date, then principal; a second pass without the due-date check spreads any rest (:127-168)
- **Acme sample** `custom/acme/loan/processor/src/main/java/com/acme/fineract/loan/processor/AcmeLoanRepaymentScheduleTransactionProcessor.java` (11th class only when counting `custom/`)

### 6.3 Advanced allocation (line numbers in `AdvancedPaymentScheduleTransactionProcessor.java`)
- cumulative-signature overloads throw `UnsupportedOperationException` (:170-197)
- **`processLatestTransaction`** :466-495
  - repayment-like and not reversed: sets `prepayAttempt` (`calculateIsPrepayAttempt` :2855-2870: amount >= principal + interest outstanding till the current installment due date; false without recalculation details :2861-2863)
  - recalculates interest to the transaction date, then dispatches (:474-494)
  - `handleRepayment` (:1876) for REPAYMENT, MERCHANT_ISSUED_REFUND, PAYOUT_REFUND, GOODWILL_CREDIT, CHARGE_REFUND, CHARGE_ADJUSTMENT, DOWN_PAYMENT, WAIVE_INTEREST, RECOVERY_REPAYMENT, INTEREST_PAYMENT_WAIVER, CAPITALIZED_INCOME_ADJUSTMENT (:480-482)
  - INTEREST_REFUND to `handleInterestRefund` (:483; method :546-584, ends in `handleRepayment` :583)
  - CHARGE_PAYMENT to `handleChargePayment` (:2274-2322: pays the named charge on its due installment, rest through normal allocation); WAIVE_CHARGES log only; REAMORTIZE, REAGE, CAPITALIZED_INCOME, CONTRACT_TERMINATION own handlers; other types warn
- **Rule** `getAllocationRule` :3395-3400: loan rule for the transaction type, else the DEFAULT rule
- **Processing type** `processPeriods` :2574-2585: HORIZONTAL `processPeriodsHorizontally` :2587; VERTICAL `processPeriodsVertically` :2917 (vertical not read in detail)
- **Horizontal** `processAllocationsHorizontally` :2602-2790
  - allocation types grouped by DueType in rule order (:2590-2593)
  - in-advance filter :2612-2623 (recalc variant keeps zero-balance installments not yet skipped)
  - each loop: oldest past-due installment (transaction after due date), due installment (transaction on the due date), in-advance installments by `FutureInstallmentAllocationRule` (:2641-2673): REAMORTIZATION all future, NEXT_INSTALLMENT the next, LAST_INSTALLMENT the last, NEXT_LAST_INSTALLMENT the current period or else the last
  - then each type in order (:2676 on); interest-bearing recalc loans use `handlingPaymentAllocationForInterestBearingProgressiveLoan` :2790 (rewrites installment principal and interest, §13.1)
- **`processPaymentAllocation`** :1921-1958: PENALTY pays the installment penalty, then `LoanChargePaidBy` across that installment's penalty charges (:1934-1938); FEE the same for fees
- **Charges of an installment** `getLoanChargesOfInstallment` :2907-2915: instalment fees with a row for that installment, re-aged installments, or `isDueInPeriod(from, due, isFirst)`
- **Pay-date shift** under TILL_REST_FREQUENCY_DATE: in-advance payment treated as paid on the installment due date (`calculateNewPayDateInCaseOfInAdvancePayment` :2873-2881); WEEKLY, MONTHLY, INVALID rest frequency throw `IllegalStateException` (:2884-2902)
- **Enums** (`fineract-loan/.../portfolio/loanproduct/domain/`)
  - `PaymentAllocationType` :40-51: {PAST_DUE, DUE, IN_ADVANCE} x {PENALTY, FEE, PRINCIPAL, INTEREST}, 12 values (label typo "In advanced interest")
  - `DueType` PAST_DUE, DUE, IN_ADVANCE (:22-24); `AllocationType` PENALTY, FEE, PRINCIPAL, INTEREST
  - `FutureInstallmentAllocationRule` NEXT_INSTALLMENT, LAST_INSTALLMENT, NEXT_LAST_INSTALLMENT, REAMORTIZATION
  - `PaymentAllocationTransactionType` :33-46, 14 values: DEFAULT, REPAYMENT, DOWN_PAYMENT, MERCHANT_ISSUED_REFUND, PAYOUT_REFUND, GOODWILL_CREDIT, CHARGE_REFUND, CHARGE_ADJUSTMENT, WAIVE_INTEREST, CHARGE_PAYMENT, REFUND_FOR_ACTIVE_LOAN, INTEREST_PAYMENT_WAIVER, INTEREST_REFUND, CAPITALIZED_INCOME_ADJUSTMENT
  - `CreditAllocationTransactionType` CHARGEBACK only (:33)
  - product rules `LoanProductPaymentAllocationRule`, `LoanProductCreditAllocationRule`; loan copies `LoanPaymentAllocationRule`, `LoanCreditAllocationRule`; parsing and validation `AdvancedPaymentAllocationsJsonParser`, `AdvancedPaymentAllocationsValidator`, `CreditAllocationsValidator`; doc `fineract-doc/src/docs/en/chapters/architecture/advanced-payment-allocation.adoc`

### 6.4 Three definitions of "in advance"
- abstract cumulative: transaction date before the installment due date (`AbstractLoanRepaymentScheduleTransactionProcessor.java:348-352`)
- HeavensFamily: before the previous installment's due date (:83-97)
- advanced: PAST_DUE when after the due date, DUE only when on the due date, else IN_ADVANCE (:2629-2637); a payment one day before a due date makes the whole current installment IN_ADVANCE
- none of them matches the spec's "principal of installments due after the transaction date" for every case: an installment due on the transaction date is DUE (advanced) or on time (cumulative) and is not prepaid under the spec either; the current installment paid early is "in advance" in the processors but its principal is prepaid under the spec only when its due date is after the transaction date

### 6.5 How penalties get paid (all processors)
- **On the schedule**: `LoanRepaymentScheduleProcessingWrapper.reprocess` (:38-80) spreads every passed charge into `fee/penalty_charges_amount` per period, with waived and written-off amounts (:58-76); period rule `isInPeriod` :251-254 (from exclusive, to inclusive; first period inclusive at both ends), so a penalty dated on a transaction date sits on the installment whose period contains that date
- **Percent penalties in the wrapper** (:179-206): overdue % keeps `chargeAmount` (:189-190); every other % penalty is recomputed from total scheduled principal, principal + interest, or interest (:191-201); flat penalties use `amount` (:203)
- **Cumulative**: the repayment pays the installment penalty component in strategy order, then `updateChargesPaidAmountBy` spreads it over penalty `LoanCharge` rows (:680-722)
- **Advanced**: `processPaymentAllocation` PENALTY branch (:1934-1938)
- **Self-trigger risk**: during cumulative replay every active charge is on the schedule before any transaction runs (§9.2 step 4); a penalty dated on its trigger date is then paid by its own trigger in penalty-first strategies (FineractStyle, Creocore, RBI on time; both `DuePen*` strategies in advance pay charges due by the transaction date first, `DuePenFeeIntPriInAdvancePriPenFeeIntLoanRepaymentScheduleTransactionProcessor.java:112-138`), which lowers the principal that trigger prepays
- **CHARGE_PAYMENT** (`LoanChargesApiResource`, `command=pay`) targets one charge; WAIVE_CHARGES transactions are stored but not allocated by the advanced processor (:486)

### 6.6 Fast path vs full replay
- **Gate** `LoanDownPaymentHandlerServiceImpl.java:135-139`: `processLatestTransaction` only when all hold
  1. chronologically latest (`LoanTransactionService.isChronologicallyLatestRepaymentOrWaiver` :45-51)
  2. no adjusted transaction (not an adjust or undo)
  3. not a foreclosure (sub-status FORECLOSED set before the call, `LoanAccountDomainServiceJpa.java:1025`)
  4. not `loan.hasChargesAffectedByBackdatedRepaymentLikeTransaction` (`Loan.java:1258-1272`: progressive only; transaction before the business date; active SPECIFIED_DUE_DATE or OVERDUE_INSTALLMENT charges with min(due date, submitted date) after the transaction date)
  5. `canProcessLatestTransactionOnly` (`LoanTransactionProcessingServiceImpl.java:70-93`): true when the loan is not interest-bearing with recalculation; false when the transaction is not on the business date, when charges are affected by it, or for SAME_AS_REPAYMENT rest + TILL_PRE_CLOSURE_DATE or DAILY rest + TILL_REST_FREQUENCY_DATE; progressive needs a saved model valid for the date; cumulative needs amount = current installment total outstanding (:91-92)
- **Fast path** `processLatestTransaction` with `loan.getActiveCharges()` (:140-143); cumulative + recalc then still replays when the current installment is not fully paid or the next one has paid-in-advance > 0 (:144-154)
- **Else** regenerate (cumulative + recalc: `regenerateRepaymentScheduleWithInterestRecalculation`; progressive: `regenerateRepaymentSchedule`) and `reprocessTransactions` (:156-163)
- **For a prepayment penalty**: a same-day repayment on a loan without recalculation takes the fast path; logic placed only in the replay loop never runs for it, logic placed only after the fast path misses every replay

## 7. Charges

### 7.1 Enums
- **`ChargeTimeType`** (`ChargeTimeType.java:23-39`), 17 constants:

| v | name | for |
|---|---|---|
| 0 | INVALID | |
| 1 | DISBURSEMENT | loan |
| 2 | SPECIFIED_DUE_DATE | loan, savings, client |
| 3-7 | SAVINGS_ACTIVATION, SAVINGS_CLOSURE, WITHDRAWAL_FEE, ANNUAL_FEE, MONTHLY_FEE | savings |
| 8 | INSTALMENT_FEE | loan |
| 9 | OVERDUE_INSTALLMENT | loan (must be penalty) |
| 10, 11 | OVERDRAFT_FEE, WEEKLY_FEE | savings |
| 12 | TRANCHE_DISBURSEMENT | loan |
| 13-15 | SHAREACCOUNT_ACTIVATION, SHARE_PURCHASE, SHARE_REDEEM | shares |
| 16 | SAVINGS_NOACTIVITY_FEE | savings |

  - codes `chargeTimeType.<camelCase>` except `tranchedisbursement` (12), `activation` (13), `sharespurchase` (14), `sharesredeem` (15)
  - sets: `validLoanValues` :57-61 = {1, 2, 8, 9, 12}; `validLoanChargeValues` :63-66 = {1, 2, 8}; savings :68-73; client :75-77; shares :79-82
  - `fromInt` :84 (explicit switch :87); `isOverdueInstallment` :188-190; `isAllowedLoanChargeTime` :192-194; `isDisbursementOrTrancheDisbursementCharge` :225; next free value 17
  - loan time list hard-coded in `ChargeDropdownReadPlatformServiceImpl.java:85-88`
  - e2e copy `fineract-e2e-tests-core/.../test/data/ChargeTimeType.java` (15 values, no INVALID and no 4; 9 named `OVERDUE_FEES`); `ChargeGlobalInitializerStep` (239 lines) seeds e2e charges
- **`ChargeCalculationType`** (`ChargeCalculationType.java:23-28`, in fineract-charge), 6 constants: INVALID 0, FLAT 1, PERCENT_OF_AMOUNT 2, PERCENT_OF_AMOUNT_AND_INTEREST 3, PERCENT_OF_INTEREST 4, PERCENT_OF_DISBURSEMENT_AMOUNT 5
  - codes `chargeCalculationType.flat`, `.percent.of.amount`, `.percent.of.amount.and.interest`, `.percent.of.interest`, `.percent.of.disbursement.amount`
  - sets: `validValuesForLoan` :46-50 = {1..5}; savings {1, 2} :52; shares {1, 2} :56; clients {1} :60; share activation {1} :64; tranche {1, 5} :68-70
  - `fromInt` :72-81; `isPercentageOfAmount` :83; `isPercentageBased` :107-110; next free value 6
  - e2e copy uses other names (PERCENTAGE_AMOUNT, ...)
- **Switches that need a case for a new calculation type**: `LoanChargeService.java` :249, :356, :396, :447, :766, :782, :822, :884; `LoanChargeApiJsonValidator.java:267, :427`; `LoanChargeAssembler.java:267`; savings and client copies `SavingsAccountCharge.java:255/414/488`, `ClientCharge.java:166`
- **Switches on time type**: `LoanProductParamType.java:47`, `LoanChargeWritePlatformServiceImpl.java:895`
- **Other enums**: `ChargeAppliesTo` LOAN 1, SAVINGS 2, CLIENT 3, SHARES 4; `ChargePaymentMode` REGULAR 0, ACCOUNT_TRANSFER 1
- **Labels** `ChargeEnumerations` (217 lines): `chargeTimeType(int)` :34, `chargeTimeType(ChargeTimeType)` :38 (one case per value), OVERDUE_INSTALLMENT label "Overdue Fees" :77-79, `chargeAppliesTo` :113, `chargeCalculationType` :141, `chargePaymentMode` :176

### 7.2 Definition validation, penalty flag, caps
- **Deserializer** `ChargeDefinitionCommandFromApiJsonDeserializer` (487 lines)
  - `validateForCreate` :91: unsupported-parameter check :99; `DataValidatorBuilder(...).resource("charge")` :102
  - loan branch :158-183: time in `validLoanValues` (:164-165); payment mode required; calculation in `validValuesForLoan` (:176-179); `performChargeTimeNCalculationTypeValidation` :181-183
  - penalty `notNull` :270-271; caps only checked when present, positive only (:279-286); no min <= max check anywhere
  - `validateForUpdate` :296: time in the union of all sets (:386-400); calculation `inMinMaxRange(1, 5)` (:412-416); penalty :423-426; caps :432-439
  - `validateChargeTimeNCalculationType` :459-464; `performChargeTimeNCalculationTypeValidation` :466-480: share activation needs FLAT; tranche needs FLAT or % of disbursement; every other time rejects % of disbursement
  - errors thrown as `PlatformApiDataValidationException` (HTTP 400)
- **Entity** `Charge` (738 lines), `m_charge` with unique name (:61)
  - columns `is_penalty` :108-109, `min_cap` :117-119, `max_cap` :121-123 (precision 19, scale 6), GL income account, tax group
  - `fromJson` :164 (penalty :179, caps :183-184); constructor :211; loan rules :273-287
    - penalty due at DISBURSEMENT or TRANCHE_DISBURSEMENT: `ChargeDueAtDisbursementCannotBePenaltyException` (:275-277)
    - OVERDUE_INSTALLMENT that is not a penalty: `ChargeMustBePenaltyException` (:278-280)
    - other time not allowed for loans: "not.allowed.charge.time.for.loan" (:283-286)
  - both exceptions extend `AbstractPlatformDomainRuleException` (`ChargeMustBePenaltyException.java:27`), so HTTP 403 (`PlatformDomainRuleExceptionMapper.java:53`), not 400
  - caps stored only when `isPercentageOfDisbursementAmount() || isPercentageOfApprovedAmount()` (`isPercentageOfApprovedAmount` = PERCENT_OF_AMOUNT :343-345) (:289-292); caps sent with a FLAT charge are dropped without an error
  - `update` :371: penalty :561-566; caps only for PERCENT_OF_AMOUNT :575-590; disbursement recheck :593-598; `isOverdueInstallment` :672-673
- **Write service** `ChargeWritePlatformServiceJpaRepositoryImpl`: create :74; update :125-160; delete :213, blocked when used by a loan (:226); duplicate name :243
  - update blocks only deactivation while products use the charge (:140-151) and fee frequency / interval changes for loan charges (:153-158); time type, calculation type, amount and penalty flag stay editable while loans use the charge
- **Other**: `ChargesApiResource`, `ChargesApiResourceSwagger`, `ChargeRequest`, `ChargeData` (read-model flags :129-134); bean in `ChargeConfiguration`
- **Savings precedent for an early-exit penalty**: `DepositPreClosureDetail.java:40-49` (`pre_closure_penal_applicable`, `pre_closure_penal_interest`, `pre_closure_penal_interest_on_enum`); `PreClosurePenalInterestOnType` WHOLE_TERM 1, TILL_PREMATURE_WITHDRAWAL 2 (:28-30)

### 7.3 Caps and rounding on the loan side
- `LoanCharge.minimumAndMaximumCap` :309-331: min checked first and returned unrounded; then max returned unrounded; an in-range value rounded through `Money.of` (:328); with min > max the min wins
- `LoanCharge.percentageOf` :292-302 (uses `MoneyHelper.getMathContext()` :297)
- callers `LoanChargeService.java:382, :431, :804, :844`; caps copied from the definition in `LoanChargeService.create` (:488-489)
- money rounding: `Money` constructor strips zeros, rounds to `inMultiplesOf` for 0-decimal currencies, then `setScale(decimalPlaces, roundingMode)` (`Money.java:40-52`); tenant rounding mode from global config `rounding-mode` (default 6 = HALF_EVEN)

### 7.4 Product to loan
- **Product charges** `LoanProduct.charges` `@ManyToMany` EAGER through `m_product_loan_charge` (`LoanProduct.java:120-122`); assembled in `LoanProductWritePlatformServiceJpaRepositoryImpl.assembleListOfProductCharges` :333-363 (currency must match, :354-355); duplicate type message "Loan product may only have one charge of each type" on a data-integrity error (:410)
- **Template** pre-fills product charges except overdue: `LoanAccountData.withProductData` :364-373 (overdue skipped :370; `toLoanChargeData` :707-718); charge options exclude OVERDUE, and TRANCHE for single-disbursement loans (`LoanReadPlatformServiceImpl.java:1490-1497`; `LoansApiResource.java:1220, :1223`)
- **Server builds loan charges only from the request JSON `charges` array**: `LoanChargeAssembler.fromParsedJson` :66-224 (body :90-223): single disbursement :135-138; multi-disbursement :139-207; existing id updates :209-217; `createNewFromJson` :248-256 / :258-320; `createNewWithoutLoan` :325-330; a client that omits a product charge gets a loan without it
- **Per-loan overrides** in `charges[]`: `amount`, `chargeTimeType`, `chargeCalculationType`, `dueDate`, `chargePaymentMode`, `externalId` read per entry (`LoanChargeAssembler.java:108-118`)
  - `LoanChargeApiJsonValidator` resolves time and calculation overrides without checking them against allowed values (:353-363); the overdue rejection looks at the definition (`chargeDefinition.isOverdueInstallment()`, :385-389) and throws a 403 `LoanChargeCannotBeAddedException`; amount `notNull` and positive per entry (:390-393); due date not before expected disbursement (:395-405)
  - `LoanChargeService.create` :479-521: submitted date = business date (:486); penalty flag (:487); caps (:488-489); time override (:490); due date required for SPECIFIED_DUE_DATE and OVERDUE (:492-504); calculation override (:506); amount override (:508-511)
- **Overdue charges** are never attached at creation; COB reads the product charge (`LoanChargeWritePlatformServiceImpl.java:804-805`; `LoansApiResource.java:1261-1262`; `LoanReadPlatformServiceImpl.java:1598-1599`)
- **Modify application**: `LoanAssemblerImpl.updateFrom` builds the existing-charge map from `getActiveCharges`

### 7.5 Adding a charge to an existing loan
1. POST `/v1/loans/{loanId}/charges` (`LoanChargesApiResource.handleExecuteLoanCharge` :432-451), `@CommandType("LOANCHARGE", "CREATE")` `AddLoanChargeCommandHandler`
2. `LoanChargeWritePlatformServiceImpl.addLoanCharge` :182-300 (file 1,449 lines)
   - JSON validation :183; charged-off loans rejected with `GeneralPlatformDomainRuleException("error.msg.loan.is.charged.off")` (:187-190); disbursement charge on a disbursed loan :195-199; % of disbursement tranche loop :205-239; `createNewFromJson` :241; pre event :242
   - `validateAddLoanCharge` :244 (private :1007-1032): manual OVERDUE_INSTALLMENT rejected "overdue.charge" (:1008-1011); due date before the last transaction or disbursement rejected
   - `addCharge` :245 (:1034-1074): currency and account-transfer checks; `validateChargeAdditionForDisbursedLoan` :1049; specified date :1050; `loanChargeService.addLoanCharge` :1051; save :1052
   - charge-applied ACCRUAL when (active and NONE, CASH or UPFRONT) or overpaid or closed-obligations-met (:1056-1064), through `handleChargeAppliedTransaction`, which returns null for progressive loans (`LoanChargeService.java:155-162`); else post-maturity immediate accrual through `createChargeAppliedTransaction`, progressive included (:1065-1071); `createChargeAppliedTransaction` :164-207 (`LoanTransaction.accrueLoanCharge`)
3. Replay decision :253-280
   - overpaid loan with an unpaid charge: replay (:255)
   - interest-bearing + recalc: backdated + fee compounding runs `runScheduleRecalculation` instead (:256-259); `updateOriginalSchedule` :260
   - advanced processor: replay skipped only when the charge's effective due date is strictly after the last transaction date for reprocessing (:263-271); a penalty dated on its trigger date never skips
   - other processors: always replay; progressive regenerates first (:273-279)
4. Delinquency tag :287; post events `LoanAddChargeBusinessEvent` :289 and `LoanBalanceChangedBusinessEvent` :290
- `LoanChargeService.addLoanCharge` :209-242: single-charge wrapper reprocess (:237-238), transition LOAN_CHARGE_ADDED (:241); `updateOverdueScheduleInstallment` :537; `removeLoanCharge` :917
- charge due after maturity: `addInstallmentIfPenaltyAppliedAfterLastDueDate` adds a `recalculatedInterestComponent` installment (`LoanChargeWritePlatformServiceImpl.java:1146-1167`)
- **Validators** (`fineract-loan/.../loanaccount/serialization/`)
  - `LoanChargeApiJsonValidator` (458 lines): `validateAddLoanCharge` :74; `validateLoanCharges(Set)` :260-296; `validateLoanCharges(JsonElement, ...)` :298-421 (supported params :308-312); interest-recalculation restrictions :423-457 (% of amount: no instalment fee; % of amount + interest: no instalment fee, no specified date; % of interest: no specified date, no instalment fee on progressive)
  - `LoanChargeValidator` (107 lines): `validateLoanIsNotClosed` :40, `validateLoanChargeIsNotWaived` :51, `validateChargeHasValidSpecifiedDateIfApplicable` :59, `validateChargeAdditionForDisbursedLoan` :68, `validateRepaymentTypeTransactionNotBeforeAChargeRefund` :86, `validateChargePaymentNotInFuture` :100; no charge-off check in this class (the guard sits in `addLoanCharge`)

### 7.6 Percentage base and re-derivation
- `LoanChargeService.calculateAmountPercentageAppliedTo` :244-271: overdue charges keep their own `amountPercentageAppliedTo` (:245-247); PERCENT_OF_AMOUNT via `getDerivedAmountForCharge` (:250; method :896-915: loan principal, or tranches expected by the due date for SPECIFIED_DUE_DATE on multi-disbursement loans); PERCENT_OF_AMOUNT_AND_INTEREST principal + total interest; PERCENT_OF_INTEREST total interest; % of disbursement the tranche or loan principal; FLAT 0
- `LoanChargeAssembler.createNewFromJson(loan, charge, command, dueDate)` :258-318 takes the base from command params `principal` / `interest` when present (COB overdue path, write service :1121)
- no existing base is "outstanding principal" or "prepaid principal"
- **Re-derivation sites** (a stored base must survive each)
  1. charge-spreading wrapper: fee :84-120 (multi-disburse fee special case :107-113), penalty :179-206 (totals :43-46)
  2. `LoanChargeService.recalculateAllCharges` :68-75 and `recalculateLoanCharge` :85-108 (FLAT keeps `amountOrPercentage`; % re-based)
  3. reschedule approve `recalculateAllCharges` (`LoanRescheduleRequestWritePlatformServiceImpl.java:455`)
  4. `LoanScheduleService.regenerateRepaymentScheduleWithInterestRecalculation` :97-125: charges due on or after the last transaction date and not waived (or instalment fees) recalculated (:112-123); charges due after the new last repayment date set inactive for cumulative only (progressive always takes the recalc branch)
  5. cumulative generator :2085-2175
  6. progressive generator :385-450
  7. foreclosure: null-due-date charges recalculated (`LoanAccountDomainServiceJpa.java:993-995`)

### 7.7 Waive and undo waive
- `LoanChargeWritePlatformServiceImpl` waive :475-571: loan active; charge not already waived or paid; pre event `LoanWaiveChargeBusinessEvent` :498; instalment fee picks the installment; periodic accrual sums the accrued charge; WAIVE_CHARGES transaction, journal, post events :555-556
- `LoanCharge.waive` :181-203 waives the amount outstanding at that moment; `undoWaive` :205; `resetPaidAmount` :168-175 keeps waived; `resetToOriginal` :156-166 clears paid and waived
- undo waive `LoanTransactionsApiResource.undoWaiveCharge` (:425, :439); service `undoWaiveLoanCharge` :358 (charged-off guard :388-390); adjust of a charges-waiver transaction calls `undoWaive` (`LoanAdjustmentServiceImpl.java:132-141`)
- other LOANCHARGE operations: refund :304, update :423, delete :573, pay :606, adjustment :701 (`linkToCharge` CHARGE_ADJUSTMENT :869-870), deactivateOverdue :752, `applyChargeToOverdueLoanInstallment` :1076, `waiveLoanCharge(Loan, ...)` :1369

## 8. Prepay and foreclosure quotes

### 8.1 The four calculations
1. **Cumulative generator** `AbstractCumulativeLoanScheduleGenerator.calculatePrepaymentAmount` :2791-2823
   - till date = next rest date under TILL_REST_FREQUENCY_DATE (:2795-2798)
   - `rescheduleNextInstallments` on a copy (:2800-2801), replays all non-reversed transactions with `loan.getActiveCharges()` (:2802-2805)
   - sums principal, interest, fee, penalty outstanding of every not-fully-paid installment (:2810-2817)
2. **Progressive generator** `ProgressiveLoanScheduleGenerator.calculatePrepaymentAmount` :184-239
   - needs the advanced processor (:187-189)
   - date: TILL_PRE_CLOSURE_DATE the quote date; TILL_REST_FREQUENCY_DATE the current installment due date; NONE throws (:194-198)
   - saved model `orElseThrow` (:200-202); `getOutstandingAmountsTillDate` :203; TODO past-due amounts (:204)
   - adds down-payment installment principal (:210-217)
   - non-recalc deduction discarded: `result.principal().minus(paidPrincipal)` result not assigned (:224-225), and `paidPrincipal` sums scheduled principal, not paid principal (:222-223); unreachable through the dispatch below
   - adds additional-installment principal and interest, and fee and penalty outstanding of every installment (:228-236)
3. **Whole-schedule sum** `LoanTransactionProcessingServiceImpl.getTotalOutstandingOnLoan` :175-188 (outstanding of every installment)
4. **Foreclosure detail** `LoanBalanceService.fetchLoanForeclosureDetail` :214-221
   - `retrieveIncomeOutstandingTillDate` :248-295: installments due on or before the date: full interest, fee, penalty outstanding; current period: interest pro rata by days and charges due up to the date (`fetchInterestFeeAndPenaltyTillDate` :298-345, raw `getLoanCharges` with an active check); later installments: amounts already paid on them counted as `paidFromFutureInstallments`
   - principal = summary principal outstanding minus `paidFromFutureInstallments` (:216-217); returned row dated on the business date (:218-220)
- **Dispatch** `LoanTransactionProcessingServiceImpl.fetchPrepaymentDetail` :153-173: generator path only when `isInterestBearingAndInterestRecalculationEnabled`, not charged off on the date and not contract-terminated (:157); else the whole-schedule sum; a 0% loan with recalculation enabled takes the sum path
- `LoanScheduleAssembler.calculatePrepaymentAmount` :784-798 is the other caller of the generator method
- **Prepay template** `LoanReadPlatformServiceImpl.retrieveLoanPrePaymentTemplate` :556-589 (class `@Transactional(readOnly = true)` :163)
  - `validateRepaymentTransactionType` :560; `fetchPrepaymentDetail` :572-573
  - `adjustPrepayInstallmentCharge` always 0, body commented out (:579, :591-604); amount = total outstanding (:580); returned `date` is the business date even when a date is passed
- **Foreclosure template** `retrieveLoanForeclosureTemplate` :2069-2100: `validateForForeclosure` :2073; `fetchLoanForeclosureDetail` :2081-2082; amount :2089-2092
- **`LoanForeclosureValidator`** :30-47: interest-bearing + recalc loans cannot foreclose (:31-35); no future date (:37-40); not before the last user transaction (:42-46)
- no penalty row exists at quote time; the quote is read-only and a later repayment creates the penalty inside its own allocation

### 8.2 Consumers of the quote
- GET `{loanId}/transactions/template` with `command=prepayLoan` and `command=contractTermination`: `retrieveLoanPrePaymentTemplate(REPAYMENT, loanId, date)` (`LoanTransactionsApiResource.java:668-677`; date param or business date)
- MIR, payout refund, goodwill credit, interest payment waiver templates: same method at the business date (:634-649)
- foreclosure template (:682-689)
- top-up: approve / modify sets net disbursal from the old loan's prepay amount (`LoanWritePlatformServiceJpaRepositoryImpl.java:879-886`); disbursal validates and repays the old loan (`validateTopupLoan`, `LoanApplicationValidator.java:2011`; also :682, :1433); `disburseLoanToLoan` posts a REPAYMENT on the old loan (`LoanWritePlatformServiceJpaRepositoryImpl.java:383-388`, :1667-1679) through `repayLoanWithTopup` (:1676), which calls `LoanAccountDomainServiceJpa.makeRepayment`, so the shared handler
- `futureSchedule` association on GET loan (`LoansApiResource.java:1114-1117`): `LoanScheduleCalculationPlatformServiceImpl.updateFutureSchedule` :83-120 calls `loanScheduleAssembler.calculatePrepaymentAmount` (:116); guard :91-94: interest-bearing + recalc, not NPA, not charged off, active, interest-first processor only (`InterestPrincipalPenaltyFeesOrder...java:192-193`, `RBI...java:257-258`)
- contract termination: validation open, PROGRESSIVE only, not charged off, not already terminated (`LoanContractTerminationServiceImpl.java:186-215`); the advanced `handleContractTermination` calls `handleAccelerateMaturityDate` when an installment is unpaid (:503-504)
- integration-test helper `BaseLoanIntegrationTest.verifyPrepayAmountByRepayment` :301-314: GET template, POST `repayment` for that amount, assert CLOSED_OBLIGATIONS_MET; `getPrepayAmount` :297

### 8.3 Payoff detection
- `LoanAccountDomainServiceJpaHelper.calculateRecalculateTillDate` :42-82, called from `makeRepayment` (:251-252)
  - `REQUIRES_NEW`, `readOnly`; inside a batch with an enclosing transaction it returns null, with a "changes state of entities" comment (:47-64)
  - recalc + TILL_PRE_CLOSURE_DATE only (:66-67); payment >= prepay outstanding sets recalculate-till = transaction date (:70-72)
  - exceptions swallowed with `log.warn` (:74-80)
- a quote that includes a penalty compared with a payment that also covers it still detects payoff; a quote without the penalty compared with a payment that includes it also passes (payment >= outstanding); only a compared outstanding that includes the penalty while the payment does not misses; batch and exception cases fail silently

### 8.4 Foreclosure flow (`LoanAccountDomainServiceJpa.foreCloseLoan` :702-768)
1. charged-off loan: foreclosure date before the charge-off date rejected (:705-710)
2. `LoanForeClosurePreBusinessEvent` :711
3. `fetchLoanForeclosureDetail` :716 (§8.1 calculation 4); `processAccrualsOnLoanForeClosure` :718
4. payables read from the detail (:720-723)
5. `updateInstallmentsPostDate` :724 (method :946-1014)
   - installments due on or after the date removed; one new installment from the last remaining due date to the foreclosure date carries their principal (:952-986; undisbursed tranches subtracted :963-967; `updateLoanScheduleOnForeclosure` :986)
   - active charges due after the date set inactive (:991-992); null-due-date charges recalculated (:993-995)
   - waivers of later due-date charges and later instalment fees reversed (:999-1013)
6. one REPAYMENT for principal + interest + fee + penalty, dated on the foreclosure date (:728-735)
7. `validateForForeclosure` :738; `validateAccountStatus(LOAN_FORECLOSURE)` :740
8. `handleForeClosureTransactions` :741 (:1023-1027): sub-status FORECLOSED, then the shared repayment handler (always full replay, §6.6)
9. `reprocessExistingAccruals` :743; recalc income posting :744-746; save and journal per new transaction :748-753; `saveAndFlush` :757; post events `LoanBalanceChangedBusinessEvent` :765, `LoanForeClosurePostBusinessEvent` :766
- after the collapse nothing is due after the foreclosure date and the collapsed installment is the last one; a closed foreclosed loan cannot be adjusted (`LoanWritePlatformServiceJpaRepositoryImpl.java:1189-1192`)
- foreclosure is blocked when an undisbursed tranche is expected on or before the date (`LoanWritePlatformServiceJpaRepositoryImpl.java:2661-2665`)
- command path: `CommandWrapperBuilder.loanForeclosure` :975; `ForeClosureCommandHandler` :31 / :38; `forecloseLoan` :2645; client helpers `LoanTransactionHelper.forecloseLoan` :3052 / :3057 (:1377 is a deprecated RestAssured variant)

## 9. Reprocessing and replay

### 9.1 Service and call sites
- `ReprocessLoanTransactionsService` (:27-35); impl (173 lines): `reprocessTransactions(loan)` :64, `(loan, newTransactions)` :74, `reprocessTransactionsWithoutChecks` :82, `processLatestTransaction` :89-124 (progressive needs the saved model, :95), `handleChangedDetail` :130-156 (saves new transactions, journals, accrual events, reversal journals; replay events :155), `reprocessTransactionsAndFetchChangedTransactions` :158-172 (`getActiveCharges` :160-162; `recalculateAccrualActivityTransaction` :170)
- `LoanTransactionService.retrieveListOfTransactionsForReprocessing` :40-43; `isChronologicallyLatestRepaymentOrWaiver` :45-51
- 30 `reprocessLoanTransactionsService.reprocess*` calls (28 + 2 WithoutChecks) in 10 main files: `LoanWritePlatformServiceJpaRepositoryImpl` 10, `LoanChargeWritePlatformServiceImpl` 5, `LoanScheduleService` 3, `LoanReAgingService` 3, `LoanReAmortizationService` 3, `LoanContractTerminationServiceImpl` 2, `LoanDownPaymentHandlerServiceImpl` 1, `CapitalizedIncomeWritePlatformServiceImpl` 1, `LoanAccountDomainServiceJpa` 1, `LoanRescheduleRequestWritePlatformServiceImpl` 1
- replay triggers: backdated transaction, undo / adjust, charge add (§7.5), charge waive / undo waive, reschedule approve, re-age / re-amortize and undo, chargeback, write-off / undo, charge-off / undo, capitalized income, contract termination, tranche disbursement / undo, COB interest recalculation
- `LoanAdjustTransactionBusinessEvent` per replaced transaction (`ReplayedTransactionBusinessEventServiceImpl.java:53`)

### 9.2 Cumulative replay (`AbstractLoanRepaymentScheduleTransactionProcessor.reprocessLoanTransactions` :90-232)
1. reset paid amounts of non-disbursement charges with `resetPaidAmount` (keeps waived) (:94-100)
2. `addChargeOnlyRepaymentInstallmentIfRequired` (:101)
3. reset installment derived components and obligations (:103-106)
4. wrapper spreads every passed charge over the periods before any transaction runs (:110-111)
5. charge payments re-applied first, by due period (:115-170)
6. per repayment-like, interest-waiver or recovery transaction (:181)
   - new (no id): `processLatestTransaction` (:184-187)
   - existing: `copyTransactionProperties` (:193) and process the copy; copy reversed: reverse original (:204-206); amounts match: keep original, update its mappings (:207-209); else `createNewTransaction` (:210-212)
7. write-off, refund, credit balance refund, chargeback (with relation reprocessing), charge-off branches (:215-228)
8. `reprocessInstallments` (:230)
- `createNewTransaction` :497-507: charge-refund guard, reverse old, null its external id, copy relations to the new one, add REPLAYED from new to old, record the change
- relations are copied only on replacement (`LoanTransaction.copyLoanTransactionRelations` :982-987); `copyTransactionProperties` :313-330 copies no relations; `transactionAmountsMatch` :417-425

### 9.3 Progressive replay (`AdvancedPaymentScheduleTransactionProcessor.reprocessProgressiveLoanTransactions` :201-289)
1. reset non-disbursement charge paid amounts (:208-214)
2. remove re-aged and additional installments, recreated during replay (:217-218); reset balances (:220-223)
3. build the interest model (:230-231) and context (:234-235)
4. `createSortedChangeList` :237: transactions, charges and term variations in one ordered list
   - order `ChangeOperation.compareTo` :126-152 by effective date, then submitted date, then created time; accrual activity after others; disbursement before instalment fees
   - charge effective date `getEffectiveDate` :82-98: due date when backdated (due before submitted, `isBackdatedCharge` :77-80), else submitted date (instalment fees: due date before disbursement)
5. loop :240-271: term variation; transaction (`processSingleTransaction` :256; overpaid repayment-like ones kept, :259-261); charge (`processSingleCharge` :263-267, then `processOverpaidTransactions` :268-270 lets earlier overpayments pay it)
6. replacements through `createNewTransaction` (:275-282; method :1453-1490, REPLAYED :1474-1476); the replacement got the old relations earlier in `updateOrRegisterNewTransaction` (:1415-1426, copy :1422), unless the old transaction was kept (`useOldTransactionIfApplicable` :1417-1420)
7. `recalculateInterestForDate(targetDate)` :283; `reprocessInstallments`
- `reprocessLoanTransactions` uses the business date as target (:292-298)
- `ChangedTransactionDetail`: list of `TransactionChangeData(old, new)`; merge by ids; `addNewTransactionChangeBeforeExistingOne`

### 9.4 Charges during replay
- **Overdue charges**: persisted rows, re-applied and re-paid, never re-created by replay; created only by the COB step or job, dedupe by charge + installment + frequency number (`LoanChargeReadPlatformServiceImpl.java:250-258`)
- **Waived**: waived amount kept through `resetPaidAmount`; wrapper re-applies it (`LoanRepaymentScheduleProcessingWrapper.java:58-76`)
- **Percentage penalties** re-derived during replay (§6.5, §7.6); only the overdue type keeps its base
- **Charge set**: replay passes `loan.getActiveCharges()` (40 call sites in main code) to the processors; inactive charges drop out of allocation
- **Backdated charge flag** forces full replay on progressive loans (§6.6 step 4)

## 10. Undo and adjust

### 10.1 Adjust and undo path
1. **API** POST `/v1/loans/{loanId}/transactions/{transactionId}` (`LoanTransactionsApiResource.java:348-361`; external-id variants up to :415); private `adjustTransaction` :722-747 sends `command=chargeback`, capitalized-income adjustment, buy-down-fee adjustment and interest refund to their own builders; any other value becomes `CommandWrapperBuilder.adjustTransaction` :948 (action ADJUST, entity LOAN)
2. **Handler** `LoanRepaymentAdjustmentCommandHandler` (`@CommandType(entity = "LOAN", action = "ADJUST")` :32) calls `adjustLoanTransaction` (:40)
3. **Write service** `LoanWritePlatformServiceJpaRepositoryImpl.adjustLoanTransaction` :1182-1248
   - closed and FORECLOSED: rejected (:1189-1192)
   - pre event `LoanAdjustTransactionBusinessEvent` (:1195-1196)
   - rejected: account transfer (:1197-1200), written-off loan (:1201-1204), transaction with chargeback relations (:1206-1210), interest refund (:1212-1215)
   - adjust = `transactionAmount` > 0 (:1222); adjusting a non-editable type is rejected (:1223-1227); `LoanTransaction.isEditable` :765-767 excludes chargeback, goodwill credit, payout refund and MIR; an undo (amount 0) is allowed for every type
4. **Adjustment service** `LoanAdjustmentServiceImpl.adjustLoanTransaction` :107-296 (file 380 lines)
   - builds a new transaction of the same type, also for an undo with amount 0 (:117-118); interest-waiver variant (:119-131)
   - adjust of a charges-waiver transaction: `undoWaive` on each charge it waived (:132-141)
   - capitalized income, buy-down fee and their adjustments: own rules (:143-197)
   - recalc loans: recalculate from the earlier of the old and new dates (:199-204)
   - cumulative + recalc (or hold-guarantee-funds product): adjusting a transaction dated before the last user transaction is refused (:209-213; `LoanTransactionValidatorImpl.java:718-724`)
   - holiday and non-working-day checks on the new date (:215-218)
   - `adjustExistingTransaction` :220; `reprocessExistingAccruals` :222; income posting for recalc loans (:223-225); new transaction saved and journalled when non-zero (:227-233); post events `LoanBalanceChangedBusinessEvent` :284, `LoanAdjustTransactionBusinessEvent` :285
5. **`adjustExistingTransaction`** :298-353
   - allowed types: repayment-like, waiver, accrual-related, credit balance refund, deferred income, capitalized-income and buy-down-fee adjustments (:304-312)
   - charge-refund ordering check, then the old transaction is reversed and flagged manually adjusted (:314-317)
   - MIR or payout: their RELATED interest refunds reversed, journalled, one `LoanAdjustTransactionBusinessEvent` each (:319-335)
   - closed written off: the write-off reversed (:337-343)
   - repayment-like or waiver: `handleRepaymentOrRecoveryOrWaiverTransaction(loan, new, adjusted, dto)` (:345-348), also for an undo; the handler adds the new transaction only when non-zero (`LoanDownPaymentHandlerServiceImpl.java:99-101`) and never takes the fast path (adjusted transaction set, §6.6 step 2)
   - capitalized income: schedule regenerated with reprocessing (:350-352)
- **Undo = adjust with amount 0**; no separate undo command for repayments
- **No relation copy on adjust**: the new transaction starts with no relations; the service reads relations only to find RELATED refunds (:324); replay copies relations only when it replaces a transaction (`createNewTransaction`, §9.2)
- **For a prepayment penalty**
  - the adjusted repayment is reversed and a new transaction (no id, no relations) goes through the shared handler; a link from the old trigger to its penalty has to move to the new transaction, or the penalty looks orphaned
  - an undo creates no new transaction, so the trigger is gone; removal has to deal with a waived penalty (`LoanCharge.waive` covers only the amount outstanding at that moment, `LoanCharge.java:181-203`)
  - cumulative + recalc loans accept adjust or undo only on the latest user transaction, so backdated scenarios there come only from charge add, reschedule and COB

### 10.2 Undo disbursal and undo last tranche
- **Undo disbursal** `LoanWritePlatformServiceJpaRepositoryImpl.undoLoanDisbursal` :853-874; allowed on an open loan that is not a top-up (`LoanDownPaymentTransactionValidator.java:111-123`), also when repayments exist
  - `updateLoanToPreDisbursalState` :3158-3190: `reverseExistingTransactions` :3163 (method :3193-3204 reverses and journals every transaction)
  - active overdue charges `setActive(false)` (:3166-3168); every other active charge `resetToOriginal` (:3170), which clears paid and waived (`LoanCharge.java:156-166`); those charges stay active
  - wrapper re-spreads the active charges (:3186-3187)
- **Undo last tranche** `undoLastLoanDisbursal` :2602; private `undoLastDisbursal` :3391
  - refused when a repayment-like, waiver or charge-payment transaction is dated after the tranche (`UndoLastTrancheDisbursementException`, :3404)
  - `LoanBalanceService.updateLoanToLastDisbursalState` :181-194: overdue charges inactive (:183-184); that tranche's disbursement charge reset (:185-188)
  - `reverseExistingTransactionsTillLastDisbursal` :3450-3480: tranche charge `resetPaidAmount` and inactive (:3471-3472); reverses disbursement, accrual activity, repayment at disbursement and repayments on or after the tranche date (`LoanTransaction.isAllowTypeTransactionAtTheTimeOfLastUndo` :961-963)
  - then `regenerateScheduleWithReprocessingTransactions` (:3420)
- **For a prepayment penalty**: after undo disbursal every trigger is reversed while a generated penalty stays an active, unpaid, unwaived charge, since neither path knows the type; undo last tranche is refused once a later repayment exists (:3404), so it cannot orphan a later trigger

### 10.3 Chargeback
- POST `.../transactions/{transactionId}?command=chargeback` (`LoanApiConstants.CHARGEBACK_TRANSACTION_COMMAND` :182); only repayment-like types (`LoanTransaction.isTypeAllowedForChargeback` :652-655)
- new CHARGEBACK (25) transaction linked from the original by a CHARGEBACK relation (`LoanWritePlatformServiceJpaRepositoryImpl.java:1316, :1352`); the original stays and can no longer be adjusted (:1206-1210)
- cumulative `processCreditTransaction` :509-585: amount goes back onto the next regular installment due after the chargeback date (:530); else an existing additional installment (:543); else a new N+1 additional installment (:577); `handleChargeback` :790-792
- advanced `handleChargeback` :793-794, `processCreditTransaction` :1075 (EMI-calculator variant :985); credit allocation rule type CHARGEBACK (§6.3); installment columns `credits_amount`, `credited_principal/interest/fee/penalty` (§4)
- **For a prepayment penalty**: a chargeback re-opens principal without reversing the trigger, so the trigger's penalty stays unless new code follows the CHARGEBACK relation; the re-opened principal can land on an installment due after the date

## 11. Accounting and accruals

### 11.1 Modes, GL keys and mapping
- **Modes** `AccountingRuleType` NONE 1, CASH_BASED 2, ACCRUAL_PERIODIC 3, ACCRUAL_UPFRONT 4 (`AccountingRuleType.java:30-33`), per product in `m_product_loan.accounting_type`; loan helpers `Loan.java:1149` (upfront), :1153 (disabled), :1157 (none, cash or upfront), :1162 (periodic)
- **GL keys** (`AccountingConstants.java`)
  - `CashAccountsForLoan` :37-58: FUND_SOURCE 1, INCOME_FROM_PENALTIES 5 (:43), INCOME_FROM_RECOVERY 12, GOODWILL_CREDIT 13, INCOME_FROM_CHARGE_OFF_INTEREST 14, INCOME_FROM_CHARGE_OFF_FEES 15, CHARGE_OFF_EXPENSE 16, CHARGE_OFF_FRAUD_EXPENSE 17, INCOME_FROM_CHARGE_OFF_PENALTY 18, INCOME_FROM_GOODWILL_CREDIT_INTEREST / FEES / PENALTY 19-21
  - `AccrualAccountsForLoan` :91-117: same numbering up to 21 plus receivables INTEREST_RECEIVABLE 7, FEES_RECEIVABLE 8, PENALTIES_RECEIVABLE 9 (:101); INCOME_FROM_PENALTIES 5 (:97); 22-25 capitalization and buy-down accounts (cash 22 is CLASSIFICATION_INCOME)
  - `LoanProductAccountingParams`: `incomeFromPenaltyAccountId` :158, `receivablePenaltyAccountId` :164, `penaltyToIncomeAccountMappings` :169
- **Mapping table** `acc_product_mapping`: gl_account_id, product_id, product_type, payment_type, charge_id, financial_account_type, charge_off_reason_id, write_off_reason_id (`ProductToGLAccountMapping.java:42-80`); product save `LoanProductToGLAccountMappingHelper.saveChargesToIncomeAccountMappings` :136
- **Per-charge lookup for loans** `AccountingProcessorHelper.getLinkedGLAccountForLoanCharges` :1147-1167: product account, replaced by the product's row for that charge when one exists (:1159-1161)
  - the `Charge` entity's own income account (:1184) is read only in `getLinkedGLAccountForSavingsCharges` (:1169-1197), savings only
  - per-charge lookup used by the cash penalty credit (`CashBasedAccountingProcessorForLoan.java:783`) and by accrual entries (`createJournalEntriesForLoanCharges`, helper :377-395)
  - not used by the accrual repayment branch: penalty portion goes to product-level PENALTIES_RECEIVABLE, or product-level INCOME_FROM_PENALTIES for repayment at disbursement (`getLinkedGLAccountForLoanProduct`, `AccrualBasedAccountingProcessorForLoan.java:1736-1756`)
- **Test fixture**: `BaseLoanIntegrationTest.java:156-179` creates the GL accounts per test class (penalty receivable, penalty income, charge-off penalty, recovery and others)

### 11.2 From transaction to journal
1. `JournalEntryWritePlatformServiceJpaRepositoryImpl.createJournalEntriesForLoanTransaction` :813-831: returns at once when the product has no accounting (:817-820)
2. `createAccountingBridgeDataForSingleTransaction` :850 builds `AccountingBridgeDataDTO` with `LoanChargePaidByDTO` entries carrying `isPenalty`; `wasChargedOffAtTransactionTime` is false for a transaction dated before the charge-off date (:860-865)
3. `createJournalEntriesForLoan` :531-542 (NONE guard :536)
4. `AccountingProcessorForLoanFactory.java:28-50`: cash to `CashBasedAccountingProcessorForLoan`; upfront or periodic to `AccrualBasedAccountingProcessorForLoan`
5. `AccountingProcessorHelper` (1,386 lines) builds `LoanDTO`, `LoanTransactionDTO`, `ChargePaymentDTO`; splits fee and penalty payments (:120-152, `isPenalty` :143)
6. each processor first calls `checkForBranchClosures` (accrual :60, cash :62): `AccountingProcessorHelper.java:540-548` throws ACCOUNTING_CLOSED (:544) unless the latest closure date is before the transaction date; manual entries check the same (`JournalEntryWritePlatformServiceJpaRepositoryImpl.java:353`)
- a transaction in a closed period is refused through its own journal; an accrual dated on its trigger date adds no new failure, an accrual dated before its trigger could

### 11.3 Cash (`CashBasedAccountingProcessorForLoan`, 1,005 lines)
- dispatch from :49; repayments `createJournalEntriesForRepayments` :519-529: charged-off loans to `createJournalEntriesForChargeOffLoanRepayments` (:523-524), else `createJournalEntriesForLoanRepayments` :731 (:526)
- normal repayment: Dr FUND_SOURCE; penalty portion Cr INCOME_FROM_PENALTIES per charge through `createCreditJournalEntryForLoanCharges` (:783); goodwill credit debits its goodwill account (:784-788)
- charged-off penalty branch :645-671 (by transaction type)
  - MIR (:648-651), payout refund (:653-656): Cr INCOME_FROM_CHARGE_OFF_PENALTY / Dr FUND_SOURCE
  - goodwill credit (:658-660): Cr INCOME_FROM_RECOVERY / Dr INCOME_FROM_GOODWILL_CREDIT_PENALTY
  - REPAYMENT (:662-664): Cr INCOME_FROM_RECOVERY / Dr FUND_SOURCE
  - anything else (:666-669): Cr INCOME_FROM_PENALTIES / Dr FUND_SOURCE
- charge-off transaction: Cr INCOME_FROM_PENALTIES / Dr INCOME_FROM_CHARGE_OFF_PENALTY for the outstanding penalty (:174-175)

### 11.4 Accrual, periodic and upfront (`AccrualBasedAccountingProcessorForLoan`, 2,197 lines)
- reversed transactions (:63-67); ACCRUAL and ACCRUAL_ADJUSTMENT (:75-77); repayment types except charge adjustment, repayment at disbursement and charge payment to `createJournalEntriesForRepayments` (:82-84); `isIncomeFromFee` = repayment at disbursement
- dispatch :1336-1344: charged off to `createJournalEntriesForRepaymentWhenLoanIsChargedOff` :1355 (:1340); else `createJournalEntriesForLoanRepayments` :1662 (:1342)
- normal repayment penalty portion (:1736-1756): Cr PENALTIES_RECEIVABLE, or Cr INCOME_FROM_PENALTIES when `isIncomeFromFee` (:1739-1747), both product level
- charged-off penalty branch :1483-1517: MIR (:1486-1489) and payout (:1491-1494) Cr INCOME_FROM_CHARGE_OFF_PENALTY; goodwill (:1496-1499) Cr INCOME_FROM_RECOVERY / Dr INCOME_FROM_GOODWILL_CREDIT_PENALTY; REPAYMENT (:1501-1504) Cr INCOME_FROM_RECOVERY / Dr FUND_SOURCE; anything else (:1506-1515) Cr INCOME_FROM_PENALTIES when `isIncomeFromFee` (:1507), else PENALTIES_RECEIVABLE
- `createJournalEntriesForAccruals` :1989-2040, penalty part :2028-2037: ACCRUAL = Dr PENALTIES_RECEIVABLE / Cr INCOME_FROM_PENALTIES; ACCRUAL_ADJUSTMENT the reverse; per charge through `createJournalEntriesForLoanCharges` (helper :377-395, charge-specific credit and debit accounts :386, :388)
- charge-off transaction: Cr PENALTIES_RECEIVABLE / Dr INCOME_FROM_CHARGE_OFF_PENALTY (:909-910); fraud moves principal to CHARGE_OFF_FRAUD_EXPENSE (:888-890)
- argument order trap: `populateCreditDebitMaps(..., credit, debit, ...)` (:930-931)
- reason codes map to GL accounts for charge-off and write-off (`AccountingProcessorHelper.java:108-111, :174-182`)

### 11.5 Creating accruals (`LoanAccrualsProcessingServiceImpl`, 1,298 lines)
- **Periodic** (COB step ADD_PERIODIC_ACCRUAL_ENTRIES, job ADD_PERIODIC_ACCRUAL_ENTRIES): `addPeriodicAccruals` :147 (batch :125, job :160); core `addAccruals` :327-420, periodic accrual only; progressive loans merge into one transaction plus ACCRUAL_ADJUSTMENT
- **Charge accrual** `addChargeAccrual` :565-577: `loan.getLoanCharges(predicate)` with `isChargeDue` evaluated in the predicate before the `isActive` check (:569-575); amounts :579-630 (`findChargeAccrualAmount` :624)
- **`isChargeDue`** :641-649; line :646 is `chargeOnDueDate = chargeOnDueDate || loanCharge.getDueLocalDate().isBefore(loanCharge.getSubmittedOnDate());`, reached when global config `charge-accrual-date` is set to submitted-date: a non-instalment charge with a null due date (inactive ones included) throws NullPointerException there (code reading, not run)
- **`isChargeOnDueDate`** :1196-1199: config `charge-accrual-date`, default "due-date" (`parts/0107_add_configuration_charges_accrual_date.xml`; constant `GlobalConfigurationConstants.java:73`)
- **Other hooks**: `reprocessExistingAccruals` :207, `processAccrualsOnInterestRecalculation` :223, `processIncomePostingAndAccruals` :258, `processAccrualsOnLoanClosure` :287-296, `processAccrualsOnLoanForeClosure` :302 (periodic only, TODO for progressive); `reverseAccrual` :1278
- **Charge-applied accrual** (upfront, cash and none): at disbursement (`LoanDisbursementService.java:256-263`) and on add charge (§7.5 step 2: `LoanChargeWritePlatformServiceImpl.java:1056-1064`, through `LoanChargeService.handleChargeAppliedTransaction` :155-162, null for progressive loans); dated on the due date, or today when the due date is in the future (`createChargeAppliedTransaction` :164-207); periodic loans wait for COB
- **Accrual activity** (product flag `enable_accrual_activity_posting`, COB step ACCRUAL_ACTIVITY_POSTING): `LoanAccrualActivityProcessingServiceImpl` (417 lines) `makeAccrualActivityTransaction` :77 / :83 (flag and open loan); `recalculateAccrualActivityTransaction` :112; `processAccrualActivityForLoanClosure` :184 (sums installment penalty charges :196-203); reverse and replay :340-361 (REPLAYED :177, :361)
- **Closure accruals** run from post-event listeners (`LoanAccrualEventService.java:39-70`, §1.5)
- **For a prepayment penalty**
  - NONE: no journal at all (both guards above); CASH: penalty income when the penalty is paid, per-charge mapping; ACCRUAL_PERIODIC: COB accrues by due date unless the penalty posts its own accrual, and the repayment that pays it credits PENALTIES_RECEIVABLE; ACCRUAL_UPFRONT: charge-applied accrual on add for cumulative loans only
  - a penalty paid by the same repayment that creates it can credit PENALTIES_RECEIVABLE before any accrual exists, under periodic accrual (interpretation)
  - after charge-off the GL depends on the trigger's type and date, not on the product's penalty mapping (§13.4)

## 12. Business and external events

### 12.1 Event classes and raise points
- **Charge events** (`fineract-loan/src/main/java/org/apache/fineract/infrastructure/event/business/domain/loan/charge/`, 6 files): abstract `LoanChargeBusinessEvent` plus `LoanAddChargeBusinessEvent`, `LoanDeleteChargeBusinessEvent`, `LoanUpdateChargeBusinessEvent`, `LoanWaiveChargeBusinessEvent`, `LoanWaiveChargeUndoBusinessEvent`
- **Transaction events** (`.../loan/transaction/`): LoanChargePayment Pre / Post, LoanChargeAdjustment Pre / Post, LoanChargeRefund, LoanTransactionMakeRepayment Pre / Post, LoanForeClosure Pre / Post, LoanChargeOff Pre / Post, LoanUndoChargeOff, LoanAccrualTransactionCreated, LoanAccrualAdjustmentTransaction, LoanTransactionAccrualActivity Pre / Post (all `...BusinessEvent`)
- **Loan events** (`.../loan/`): LoanApplyOverdueCharge, LoanBalanceChanged, LoanAdjustTransaction, LoanChargebackTransaction, LoanInterestRecalculation, LoanRescheduledDueAdjustSchedule, LoanRescheduledDueHoliday, LoanRescheduledDueCalendarChange, LoanCloseAsReschedule
- **Raise points in `LoanChargeWritePlatformServiceImpl`**
  - add: `LoanAddChargeBusinessEvent` pre :221, :242, post :289; `LoanBalanceChangedBusinessEvent` :290
  - refund :350 / :352; undo waive :396, :400, :401; update :438, :460; waive pre :498, post :555, balance :556; delete :586, :592; pay :685; charge adjustment :737, :738, pre :865; deactivate overdue :790 (update event)
  - charge-applied accrual `LoanAccrualTransactionCreatedBusinessEvent` :1063, :1070
  - overdue (system) path: `LoanApplyOverdueChargeBusinessEvent` pre :1117, post :1139, `LoanBalanceChangedBusinessEvent` :1140; no `LoanAddChargeBusinessEvent` there
- **Other raise points**: charge payment pre `LoanAccountDomainServiceJpa.java:395`, post :439; repayment pre :222-223, post :290-295; foreclosure :711, :765-766; adjust :1195-1196 and `LoanAdjustmentServiceImpl.java:284-285`; replay `LoanAdjustTransactionBusinessEvent` per replaced transaction (`ReplayedTransactionBusinessEventServiceImpl.java:53`)

### 12.2 External event plumbing
- **Tables** `m_external_event` (type, data, created_at, status, sent_at, idempotency_key, business_date) and `m_external_event_configuration` (type, enabled) (`parts/0043_add_external_event_table.xml:26-68`)
- **Default seeds** `parts/0056_add_external_event_default_configuration.xml`, all enabled=false: LoanAddCharge :55, LoanDeleteCharge :59, LoanUpdateCharge :63, LoanWaiveCharge :67, LoanWaiveChargeUndo :71, LoanChargePaymentPost :79, LoanChargePaymentPre :83, LoanChargeRefund :87, LoanTransactionMakeRepaymentPost :127, Pre :131, LoanAdjustTransaction :179, LoanApplyOverdueCharge :183, LoanBalanceChanged :191, LoanChargebackTransaction :195
- **Changelog parts that insert into the configuration table**: 25 core parts (0043, 0056, 0067, 0068, 0070, 0080, 0082, 0094, 0099, 0102, 0135, 0137, 0138, 0141, 0142, 0151, 0176, 0179, 0181, 0183, 0184, 0187, 0189, 0190, 0191); 34 XML files in all with the module parts and one Acme custom file; 0136 is the re-age parameter table, no event content
- **New type**: a new part, pattern `parts/0191_add_LoanApprovedAmountChangedBusinessEvent.xml`; 0056 is applied and never edited (§1.3)
- **Posting** only when the event is not `NoExternalEvent`, `fineract.events.external.enabled` is true and the type is enabled (`BusinessEventNotifierServiceImpl.java:96, :101, :103`; flag :142-144); flush as bulk (:153-176); runtime lookup of an unseeded type throws `ExternalEventConfigurationNotFoundException` (`CustomExternalEventConfigurationRepositoryImpl.java:36-42`)
- **Serializers** (`fineract-provider/.../event/external/service/serialization/serializer/loan/`): `LoanChargeBusinessEventSerializer` (66 lines) to `LoanChargeDataV1`; `LoanBusinessEventSerializer`; transaction, adjust, repayment and charge-off serializers in the same folder; `InvestorBusinessEventSerializer.java:135` reads raw `getLoanCharges()`
- **Avro** (38 loan v1 schemas in `fineract-avro-schemas/src/main/avro/loan/v1/`)
  - `LoanChargeDataV1`: chargeTimeType (:32-36) and chargeCalculationType (:48) as EnumOptionDataV1, dueDate, percentage, amountPercentageAppliedTo, minCap, maxCap, penalty, submittedOnDate, externalId, customData; a new time type or calculation type needs no schema change
  - `LoanTransactionDataV1`: penaltyChargesPortion, loanChargePaidByList, transactionRelations, unpaidCharges; `LoanTransactionRelationDataV1` has toLoanCharge (:20)
  - `LoanTransactionEnumDataV1`: 25 fields (id, code, value plus 22 booleans), not one boolean per transaction type (43 types)
- **e2e**: 60 event classes in `fineract-e2e-tests-core/src/test/java/org/apache/fineract/test/messaging/event/`; `LoanChargeStepDef.java:294` asserts LoanAddChargeEvent through LoanChargeDataV1 name, amount and due date; 170 feature lines match `event.* (is|was) raised`

### 12.3 Startup validation
- `ExternalEventConfigurationValidationService.java:46-113`: unconditional `@Service` and `InitializingBean`; at startup checks every tenant whatever the external-events flag says
  - count mismatch between event classes and configuration rows (:81-83), or a class without a row (:85-89): `ExternalEventConfigurationNotFoundException`
  - class scan with ClassGraph leaves out `NoExternalEvent` implementors, interfaces, abstract classes and `BulkBusinessEvent` (:100-113, filter :108-110)
- **Test lists to update with a new event**: `integration-tests/.../common/ExternalEventConfigurationHelper.java` (721 lines, 122 unique types; size asserted at `ExternalEventConfigurationIntegrationTest.java:56`); provider unit test `ExternalEventConfigurationValidationServiceTest.java` (235 lines, names listed e.g. :73, :168)
- **For a prepayment penalty**
  - a new event class without a seed row stops the server for every tenant; a seed row without a class fails the same check; marking the class `NoExternalEvent` skips both
  - existing events already carry the data (time type, calculation type, penalty flag, relation to a charge); a system-created charge has the overdue precedent (loan-level event, no add-charge event)

## 13. Wider interactions

### 13.1 Interest recalculation
- **Config**: product flag and params (`LoanProductConstants.java:83-102`: days in year / month, compounding method, reschedule strategy, rest frequency type / interval / day of week / nth day / on day, arrears on original schedule, pre-close strategy, compounding frequency fields, compounding posted as transaction, disallow interest on past due); `allowCompoundingOnEod` :126
  - tables `m_product_loan_recalculation_details` (`LoanProductInterestRecalculationDetails.java:39`), loan copy `m_loan_recalculation_details` (`LoanInterestRecalculationDetails.java:43`)
  - `LoanRescheduleStrategyMethod` (:36-40): INVALID 0, RESCHEDULE_NEXT_REPAYMENTS 1, REDUCE_NUMBER_OF_INSTALLMENTS 2, REDUCE_EMI_AMOUNT 3, ADJUST_LAST_UNPAID_PERIOD 4; `InterestRecalculationCompoundingMethod` (:36-39): NONE, INTEREST, FEE, INTEREST_AND_FEE; pre-close strategy §5
- **Product validator** (`LoanProductDataValidator.java`)
  - reschedule strategy 1-4 (:1151-1152); rest frequency type 1-4 (:1188-1189)
  - CUMULATIVE + disallowInterestCalculationOnPastDue rejected (:1160-1165); CUMULATIVE + ADJUST_LAST_UNPAID_PERIOD rejected (:1166-1171); PROGRESSIVE requires ADJUST_LAST_UNPAID_PERIOD (:1173-1178)
  - PROGRESSIVE + TILL_REST_FREQUENCY_DATE needs DAILY or SAME_AS_REPAYMENT rest (:1320-1328); WEEKLY / MONTHLY rest throws `IllegalStateException` at run time in the advanced processor (§6.3); a product-level rejection was not found
  - non-daily interest period without partial-period calculation: "not.supported.for.selected.interest.calculation.type" (:2711-2723)
- **Cumulative path**
  - `LoanScheduleService.regenerateRepaymentScheduleWithInterestRecalculation` :97-125; the generator replays real transactions (§5)
  - early payments folded into the current period's principal (`AbstractCumulativeLoanScheduleGenerator.java:679-700`), then `applyEarlyPaymentStrategy` :1552-1575: REDUCE_EMI_AMOUNT recomputes later EMIs, dates and count kept (:1556-1559); REDUCE_NUMBER_OF_INSTALLMENTS keeps the EMI, fewer installments (:1560-1564); RESCHEDULE_NEXT_REPAYMENTS lets the next installments absorb the prepaid principal (:1565-1570)
  - `recalculateScheduleFromLastTransaction` (`LoanScheduleService.java:73`, body :78-95): non-progressive regenerate and reprocess; progressive model update only
  - repayments and adjustments dated before the last user transaction refused (`LoanTransactionValidatorImpl.java:718-735`; also for hold-guarantee-funds products)
- **Progressive path**: only ADJUST_LAST_UNPAID_PERIOD; due dates fixed; `handlingPaymentAllocationForInterestBearingProgressiveLoan` (`AdvancedPaymentScheduleTransactionProcessor.java:2790`) and `ProgressiveEMICalculator.payPrincipal` (:389) rewrite installment principal and interest after allocation; full-payoff flag `calculateIsPrepayAttempt` (§6.3)
- **Off switch**: recalculation not supported on charged-off, written-off and contract-terminated loans (`AdvancedPaymentScheduleTransactionProcessor.java:3617-3624`)
- **COB** `LOAN_INTEREST_RECALCULATION` (`LoanInterestRecalculationCOBBusinessStep.java:30-70`) calls `recalculateInterest` (`LoanWritePlatformServiceJpaRepositoryImpl.java:2336`; `LoanInterestRecalculationBusinessEvent` pre :2344, post :2361 / :2367); replays with no user request
- **Fee compounding** FEE / INTEREST_AND_FEE puts unpaid penalties into compounded arrears (`AbstractCumulativeLoanScheduleGenerator.java:1375-1377`; `Loan.isFeeCompoundingEnabledForInterestRecalculation` :1479); add charge on such a loan, backdated: `runScheduleRecalculation` (`LoanChargeWritePlatformServiceImpl.java:256-259`)
- **Accounting**: `reprocessExistingAccruals` + `processIncomePostingAndAccruals` (`LoanAccountDomainServiceJpa.java:271-274`), `processAccrualsOnInterestRecalculation` (:284); compounding posted as transaction creates INCOME_POSTING (19) transactions (`LoanAccrualsProcessingServiceImpl.java:979, :987`, gated :263, :335)
- **Quotes**: generator path only for these loans (§8.1); foreclosure refused for interest-bearing recalc loans (`LoanForeclosureValidator.java:31-35`), so the prepay template is their only payoff quote; payoff detection only under TILL_PRE_CLOSURE_DATE (§8.3)
- **Tests to model on** (integration-tests): `LoanPrepayAmountTest`, `LoanPayOffTest`, `LoanPayOffAddChargeWithRefundTest`, `LoanInterestRecalculationCOBTest`, `ClientLoanIntegrationTest` (recalc and foreclosure), `AdvancedPaymentAllocationLoanRepaymentScheduleTest`, `loan/penalty/LoanPenaltyBackdatedTransactionTest`; unit `ProgressiveEMICalculatorTest`, `AdvancedPaymentScheduleTransactionProcessorTest`, `LoanScheduleGeneratorTest`; features EMICalculation, LoanRepayment, LoanRepaymentSchedule
  - IT counts by pattern: 33 files match `isInterestRecalculationEnabled`; 38 `*Test.java` match `InterestRecalculation`; 43 match `interestRecalculation` case-insensitive
- **For a prepayment penalty**
  - prepaid principal measured on regenerated cumulative rows can read as 0 because the early-paid amount moved into the current period (:679-700); effect on a mapping-based measure (unverified); a measure taken during allocation does not depend on it
  - REDUCE_NUMBER_OF_INSTALLMENTS moves the last due date earlier after each prepayment (`Loan.getLastRepaymentPeriodDueDate` :633-643), so the last-period test depends on which schedule it reads; a penalty dated after the new last due date is set inactive by cumulative regeneration (`LoanScheduleService.java:112-123`)
  - progressive rows are rewritten after allocation, so the rows no longer show what a trigger prepaid
  - an unpaid penalty bears interest under fee compounding
  - a quote inside the generator moves the payoff-detection threshold, a template-only addition does not (§8.3)
  - COB recalculation, charge add and reschedule replay without a repayment request; logic hooked only on the repayment path never sees them

### 13.2 Multi-disbursement
- **Config**: product `multiDisburseLoan`, `maxTrancheCount`, `outstandingLoanBalance`, `allowFullTermForTranche` (`LoanProductConstants.java:72-75`), `disallowExpectedDisbursements` :143, `allowApprovedDisbursedAmountsOverApplied` :144, over-applied type / number :145-146; loan `disbursementData[]` {expectedDisbursementDate, principal} and `maxOutstandingLoanBalance` (`LoanApiConstants.java:27-30`); table `m_loan_disbursement_detail` (`LoanDisbursementDetails.java:36`)
- **Validator**: `allowFullTermForTranche` needs multi-disburse + PROGRESSIVE (`LoanProductDataValidator.java:1091-1101`); multi-disburse + non-progressive + non-daily interest period without partial period rejected (:2725-2735)
- **Disbursement date**: single disbursement sets it directly (`LoanWritePlatformServiceJpaRepositoryImpl.java:344-346`); multi: `LoanDisbursementService.adjustDisburseAmount` sets it only when null or later than the tranche (`LoanDisbursementService.java:127-129`), so `Loan.getDisbursementDate` (:973-979) is the earliest tranche
- **Schedules**: cumulative rows include future undisbursed tranches (`getDisbursementAmount` :1979, block :1978-2012); progressive adds expected tranches (`ProgressiveLoanScheduleGenerator.java:114, :260, :300`), max outstanding balance (:324-330), one down payment per disbursement (:332-340), full-term tranche (`ProgressiveEMICalculator.java:124-170`)
- **Repayment cap** "amount.exceeds.threshold": repaid principal above disbursed + adjustments + capitalized income while approved > disbursed (`LoanDownPaymentHandlerServiceImpl.java:174-184`)
- **Foreclosure** subtracts undisbursed principal (`LoanAccountDomainServiceJpa.java:963-967`); refused when an undisbursed tranche is expected on or before the date (`LoanWritePlatformServiceJpaRepositoryImpl.java:2661-2665`)
- **Reschedule** refused for multi-disburse loans unless `disallowExpectedDisbursements` (`LoanRescheduleRequestDataValidatorImpl.java:91-99`); disbursement on a charged-off loan refused (`LoanTransactionValidatorImpl.java:168-171`); undo last tranche §10.2
- **Charge base**: wrapper fee branch uses tranches disbursed by the due date for multi-disburse SPECIFIED_DUE_DATE fees (`LoanRepaymentScheduleProcessingWrapper.java:107-113`); the penalty branch has no such case (:179-206)
- **Tests**: `ProgressiveLoanTrancheTest`, `ClientLoanNonTrancheMultipleDisbursementsIntegrationTest`, `LoanApplicationUndoLastTrancheTest`, `ProgressiveLoanDisbursementAfterMaturityTest`, `LoanDisbursementDetailsIntegrationTest`
- **For a prepayment penalty**
  - lock-in from "the disbursement date" anchors on the first tranche through `getDisbursementDate`; a per-tranche anchor needs the rows of `m_loan_disbursement_detail`
  - cumulative rows carry undisbursed tranche principal after its expected date, so "principal of installments due after the date" read from rows can include principal not yet lent (interpretation)
  - a later tranche re-amortizes; with `allowFullTermForTranche` or disbursement after maturity, maturity and the last period move, so an earlier trigger's last-period result can change on replay
  - the threshold check caps any repayment, and so any prepaid base, at disbursed principal

### 13.3 Reschedule, re-amortize, re-age
- **Reschedule** `/v1/rescheduleloans` (`RescheduleLoansApiResource.java:59`), create then approve or reject; table `m_loan_reschedule_request`
  - create params (`LoanRescheduleRequestDataValidatorImpl.java:56-66`): rescheduleFromDate, adjustedDueDate, graceOnPrincipal, recurringMoratoriumOnPrincipalPeriods, graceOnInterest, extraTerms, newInterestRate, emi, endDate, recalculateInterest, reason id / comment, submittedOnDate
  - term variations EMI_AMOUNT, DUE_DATE, INTEREST_RATE_FROM_INSTALLMENT, GRACE_ON_PRINCIPAL, EXTEND_REPAYMENT_PERIOD, GRACE_ON_INTEREST (`LoanRescheduleRequestWritePlatformServiceImpl.java:245-311`)
  - `validateForCreateAction` :249-287: progressive delegates to `ProgressiveLoanRescheduleRequestDataValidator`; cumulative charged-off loan rejected (:254); installment must exist and not be fully paid (:216-226); no reschedule from a 0% rate (:76-89)
  - `validateForOverdueCharges` :228-238: "not.allowed.due.to.overdue.charges" (:235) when an overdue charge is due after the from-installment's start date, read from raw `getLoanCharges` (inactive included, :232); on create (:281) and approve (:332)
  - progressive: one change per request (`ProgressiveLoanRescheduleRequestDataValidator.java:85-89`); extraTerms limit (:125-140); grace and EMI changes unsupported (:151-164)
  - approve (`LoanRescheduleRequestWritePlatformServiceImpl.java:327`): `rescheduleNextInstallments` :438, `reprocessExistingAccruals` :454, `loanChargeService.recalculateAllCharges` :455, `reprocessTransactions` :456, `LoanRescheduledDueAdjustScheduleBusinessEvent` :467, `LoanBalanceChangedBusinessEvent` when the rate changes :470; old schedule kept in `m_loan_repayment_schedule_history` (`LoanRepaymentScheduleHistory.java:39`)
- **Re-amortize** (`reAmortize` / `undoReAmortize`, `LoanTransactionsApiResource.java:606-609`)
  - validator `LoanReAmortizationValidator.java`: not after maturity (:88-91); progressive + advanced only (:96-102); not charged off (:140); not contract-terminated (:146); needs an overdue installment (:151-157)
  - handling `LoanReAmortizationInterestHandlingType` (`fineract-loan/.../loanaccount/domain/reamortization/`, :31-33): DEFAULT, WAIVE_INTEREST, EQUAL_AMORTIZATION_INTEREST_SPLIT
  - `handleReAmortization` :605 (WAIVE_INTEREST throws :696); `handleReAmortizationWithInterestRecalculationEnabled` :761; overdue principal spread over later installments; down-payment and additional installments skipped (:632, :718)
  - transaction REAMORTIZE 30; parts 0137, 0138, loan `1034_loan_reamortization_parameters.xml`
- **Re-age** (`reAge` / `undoReAge`, `LoanTransactionsApiResource.java:602-605`)
  - params startDate, frequencyType, frequencyNumber, numberOfInstallments, interest handling (`LoanReAgingValidator.java:67-72`); `LoanReAgeInterestHandlingType` (`.../domain/reaging/`): DEFAULT, WAIVE_INTEREST, EQUAL_AMORTIZATION_PAYABLE_INTEREST, EQUAL_AMORTIZATION_FULL_INTEREST
  - validator: start date on or after disbursement for progressive (:104-108); non-progressive not before maturity (:149-152), then progressive + advanced only (:156-166); active (:169-172); once per day (:175-181); not charged off (:184-187); not contract-terminated (:190-193); no principal outstanding and not backdated: rejected (:219-230, :264-267)
  - `handleReAge` (`AdvancedPaymentScheduleTransactionProcessor.java:3168-3215`)
    - interest-bearing, DEFAULT (or no) handling, recalc: `handleReAgeWithInterestRecalculationEnabled` :3953-3996; the REAGE transaction carries principal and interest, fee and penalty 0 (:3993-3994)
    - interest-bearing without recalc: `UnsupportedOperationException` (:3180-3184); WAIVE_INTEREST throws (:3187-3188)
    - EQUAL_AMORTIZATION_FULL_INTEREST and EQUAL_AMORTIZATION_PAYABLE_INTEREST set `isEqualInstallmentForFeesAndPenalties` (:3190-3207); with recalc they go to `handleReAgeEqualAmortizationEMICalculator` :3644, else `handleReAgeWithCommonStrategy` :3750-3892 (spread :3856-3884)
    - non-interest-bearing: `new CommonReAgeSettings()` with every flag false (:3209-3210); `liftOutstandingBalances` :3998-4073 lifts fees and penalties only with the flag (:4029, :4043, :4051), else principal only; unpaid charges stay on the installment that holds their due date (`moveRelatedChargesToInstallment` :3894-3908, called :3830-3836)
    - zero amount: transaction reversed (:3211-3213); settings class `CommonReAgeSettings` :4079-4085
  - transaction REAGE 29; parts 0135 (events), 0136 (parameter table), loan `1033_add_reage_reasons_for_loan.xml`
- **Tests**: `LoanRescheduleRequestTest`, `LoanRescheduleOnDecliningBalanceLoanTest`, `LoanRescheduleWithAdvancePaymentTest`, `LoanRescheduleTestWithDownpayment`, `loan/reaging/LoanReAgingIntegrationTest`, `loan/reamortization/LoanReAmortizationIntegrationTest`; unit `ProgressiveLoanRescheduleRequestDataValidatorTest`, `LoanReAmortizationValidatorTest`, `LoanReAgingValidatorTest`; features LoanReschedule, LoanReAging (5 files), LoanReAmortization (3 files)
- **For a prepayment penalty**
  - reschedule approve re-derives every active charge (`recalculateAllCharges`) and replays, so a percentage penalty stored as a plain percentage charge is re-based; an earlier trigger may stop prepaying on the new schedule (interpretation)
  - adjustedDueDate, extraTerms, endDate and re-age change which installments are due after a date and which period is last, after the trigger
  - only overdue charges block reschedule today (`validateForOverdueCharges`); a penalty dated after the reschedule date passes
  - re-age keeps an unpaid penalty on its installment under DEFAULT handling and spreads it only under the EQUAL_AMORTIZATION types
  - re-amortize and re-age turn arrears into future principal, so a later catch-up payment can look like a prepayment under a row-based measure
  - lock-in is anchored on disbursement; nothing in these paths resets it

### 13.4 Charge-off and write-off
- **Config**: product `chargeOffBehaviour` REGULAR, ZERO_INTEREST, ACCELERATE_MATURITY (`LoanChargeOffBehaviour.java:29-31`, stored as STRING `LoanProductRelatedDetail.java:166-168`), only with the advanced strategy (`LoanProductDataValidator.java:897-906`); used at `AbstractLoanRepaymentScheduleTransactionProcessor.java:939-940`, `Loan.java:1843-1845`
- **Commands** `charge-off` / `undo-charge-off` (`LoanTransactionsApiResource.java:93-94`, dispatched :596-599)
- **`chargeOff`** (`LoanWritePlatformServiceJpaRepositoryImpl.java:2685-2786`): open, not already charged off, not before the last user transaction, not in the future, no monetary activity after the date; pre event; reason; accruals after the date reversed; recalc loans regenerate and reprocess, else `processLatestTransaction`; journals; `LoanBalanceChangedBusinessEvent` for ZERO_INTEREST and ACCELERATE_MATURITY; `LoanChargeOffPostBusinessEvent`
- **`undoChargeOff`** :2789-2843: must be the last user transaction; reverse; lift charge-off; regenerate (cumulative + recalc, or progressive); reprocess; `LoanUndoChargeOffBusinessEvent`
- **Advanced processor**: `handleChargeOff` :1988-2045; `handleAccelerateMaturityDate` :2047-2164: the current installment becomes the last principal installment, due on the charge-off date; unpaid future-dated fees and penalties go into an added additional installment (principal 0) due on the latest charge due date (:2127-2139)
- **Write-off**: date not before the charge-off date (`LoanWritePlatformServiceJpaRepositoryImpl.java:1463-1465`); close has the same guard (:1533-1534); recovery repayment (RECOVERY_REPAYMENT, loan event `LOAN_RECOVERY_PAYMENT`) allowed only on written-off loans (`LoanDownPaymentTransactionValidator.java:173-175`), capped at the written-off total (`LoanDownPaymentHandlerServiceImpl.java:88-92`), allocated like a repayment (advanced :480-482; cumulative branches §6.1), no state transition (:168-172)
- **Guards after charge-off**
  - charge-off date checks in `LoanAccountDomainServiceJpa`: charge payment :390, refund :480, disbursement :534, reverse transfer :564, credit balance refund :614, refund for active loan :658, foreclosure :705; none in `makeRepayment` (:216-331), so a repayment dated before the charge-off date is accepted and replayed (`LoanPostChargeOffScenariosTest.java:144`)
  - add charge refused, `error.msg.loan.is.charged.off` (`LoanChargeWritePlatformServiceImpl.java:187-190`, 403); COB overdue path skips the loan (:800-803); undo waive guarded (:388-392)
  - no tranche disbursement (`LoanTransactionValidatorImpl.java:168-171`), no cumulative reschedule (§13.3), no re-age or re-amortize, recalculation off (§13.1), quote takes the whole-schedule sum (`LoanTransactionProcessingServiceImpl.java:157`)
- **GL after charge-off**: decided per transaction type and by transaction date vs charge-off date (§11.3, §11.4; `wasChargedOffAtTransactionTime`, `JournalEntryWritePlatformServiceJpaRepositoryImpl.java:860-865`); reason codes map to accounts (`AccountingProcessorHelper.java:108-111, :174-182`)
- **Tests**
  - `LoanTransactionReverseReplayChargeOffTest` (:42-110: repayment, fraud flag, charge-off, reverse the repayment; 2 asset and 2 fraud-expense entries)
  - `LoanPostChargeOffScenariosTest` (1,266 lines; backdated add :144, reverse :319, reverse-replay :443, on-charge-off-date :589, :734, :878, :985)
  - `LoanChargeOffAccountingTest`, `LoanChargeOffAccrualTest`, `LoanProductWithChargeOffBehaviourTest`, `LoanAccountChargeOffWithAdvancedPaymentAllocationTest`, `LoanChargeOffAccountingEntriesForReverseReplayedTransactionsTest`, `LoanUndoChargeOffReverseExternalIdTest`, `LoanWriteOffWithAdvancedPaymentAllocationTest`
  - unit `CreateJournalEntriesForChargeOffLoanTest`; features LoanChargeOff, LoanWriteOff
- **For a prepayment penalty**
  - a post-charge-off penalty needs its own path past the add-charge guard; the existing paths refuse it
  - prepaid principal after charge-off is possible on REGULAR and ZERO_INTEREST (due dates kept); ACCELERATE_MATURITY leaves no regular principal due later, but an unpaid penalty moves to the added additional installment
  - a penalty paid after charge-off is booked as recovery income for REPAYMENT and goodwill credit, as charge-off penalty income for MIR and payout, and as normal penalty income or receivable for other types; this differs from "the product's penalty income account"
  - a backdated prepayment before the charge-off date books as normal penalty income, and replay re-posts the charge-off journal (2 + 2 entry pattern)
  - the unpaid penalty at charge-off is moved to INCOME_FROM_CHARGE_OFF_PENALTY by the charge-off journal (§11.3, §11.4)

### 13.5 Holidays and working days
- **Configs** `reschedule-repayments-on-holidays`, `allow-transactions-on-holiday`, `allow-transactions-on-non-workingday` (`GlobalConfigurationConstants.java:26-28`)
- **Holiday job** `ApplyHolidaysToLoansTasklet` (212 lines) moves installment due dates (:99-115, :149, :182) and raises `LoanRescheduledDueHolidayBusinessEvent` (:114); no reprocess call in the tasklet
- **Types**: holiday `RescheduleType` (`fineract-core/src/main/java/org/apache/fineract/organisation/holiday/domain/RescheduleType.java:23-25`: RESCHEDULETONEXTREPAYMENTDATE 1, RESCHEDULETOSPECIFICDATE 2); working days `RepaymentRescheduleType` (`.../organisation/workingdays/domain/RepaymentRescheduleType.java:28-33`: SAME_DAY, MOVE_TO_NEXT_WORKING_DAY, MOVE_TO_NEXT_REPAYMENT_MEETING_DAY, MOVE_TO_PREVIOUS_WORKING_DAY, MOVE_TO_NEXT_MEETING_DAY)
- **Validation**: repayment dates on holidays or non-working days refused when not allowed (`LoanAccountDomainServiceJpa.java:257-263`; adjust `LoanAdjustmentServiceImpl.java:215-218`)
- **For a prepayment penalty**: processors compare against the moved due dates, so a payment on the original due date can become "in advance"; the job does not replay, so earlier triggers keep their result until the next replay

### 13.6 Down payment
- **Config** `enableDownPayment`, `disbursedAmountPercentageForDownPayment`, `enableAutoRepaymentForDownPayment` (`LoanProductConstants.java:154-156`); validation `LoanProductDataValidator.java:825-831, :915-952`; `LoanScheduleValidator.java:195`
- **Schedule**: a down-payment installment due on the disbursement date, one per disbursement (`ProgressiveLoanScheduleGenerator.java:332-340`); skipped by re-amortization (`AdvancedPaymentScheduleTransactionProcessor.java:632, :718`) and by future-installment lists (:339-341)
- **Transaction** `command=downPayment` (`LoanTransactionsApiResource.java:600-601`); auto down payment `LoanDownPaymentHandlerServiceImpl.handleDownPayment` :69 / :187, which calls the shared handler (:232); DOWN_PAYMENT is repayment-like and has its own allocation rule type
- **Quote**: progressive adds the down-payment installment principal (`ProgressiveLoanScheduleGenerator.java:210-217`)
- **For a prepayment penalty**: the down-payment installment is due on the disbursement date, so a down payment of that amount on that date settles no principal due later by the spec definition; no amount cap for a manual down payment was found in the validators, and how a larger one is allocated was not traced (unverified); either kind reaches the same handler as a repayment; the down-payment installment is an "installment period" only if the last-period and lock-in rules say so

### 13.7 Chargeback and refunds
- **Chargeback** §10.3
- **MIR and payout refund** (repayment-like, own or DEFAULT allocation rule, `handleRepayment` :480-482): `LoanAccountDomainServiceJpa.makeRefund` :808-895 runs its own processing, not the shared handler; creates an INTEREST_REFUND linked RELATED (:837) when the product's supported interest refund types or the request override say so (:827-838); undoing the refund reverses that interest refund (`LoanAdjustmentServiceImpl.java:319-335`)
- **Interest payment waiver** `makeInterestPaymentWaiver` (`LoanWritePlatformServiceJpaRepositoryImpl.java:1013`, call :1027), own processing `recalculateLoanWithInterestPaymentWaiverTxn` :943-993; goodwill credit and manual down payment go through `makeLoanRepayment` like REPAYMENT (`LoanGoodwillCreditCommandHandler.java:47`, `LoanDownPaymentCommandHandler.java:45`), MIR and payout through `makeRefund` (`LoanMerchantIssuedRefundCommandHandler.java:41`, `LoanPayoutRefundCommandHandler.java:41`)
- **Refund for active loan** `handleRefund` (`AdvancedPaymentScheduleTransactionProcessor.java:1282-1300`): reversed DEFAULT order and LAST_INSTALLMENT when no own rule; money back to the borrower
- **Other repayment-like types**: GOODWILL_CREDIT, INTEREST_PAYMENT_WAIVER, INTEREST_REFUND, CHARGE_REFUND, CHARGE_ADJUSTMENT, CAPITALIZED_INCOME_ADJUSTMENT (`LoanTransaction.java:646-650`), all allocated through `handleRepayment` and with allocation rule types (`PaymentAllocationTransactionType.java:39-46`)
- **Entry points into the shared handler** `handleRepaymentOrRecoveryOrWaiverTransaction`
  - repayment (`LoanAccountDomainServiceJpa.java:1020`); goodwill credit, manual down payment, recovery payment (`LoanRecoveryPaymentCommandHandler.java:40`) and the top-up repayment take the same `makeRepayment` route
  - foreclosure (:1026); adjust or undo (`LoanAdjustmentServiceImpl.java:345-348`); auto down payment (`LoanDownPaymentHandlerServiceImpl.java:232`); waive interest (`LoanWritePlatformServiceJpaRepositoryImpl.java:3430`)
  - MIR, payout and IPW do not pass through it
- **Tests**: `LoanPayOffAddChargeWithRefundTest`; chargeback ITs (`LoanTransactionChargebackTest`, `LoanChargebackOnPaymentTypeRepaymentTransactionsTest`, `LoanDownPaymentTransactionChargebackTest`, `LoanTransactionFullAmountChargebackForOverpaidLoanTest`, `LoanAccountCreditRefundPayoutWithChargebackTest`, `DelinquencyAndChargebackIntegrationTest`); features LoanChargeback, LoanMerchantIssuedRefund, LoanPayoutRefund
- **For a prepayment penalty**
  - MIR and payout can settle future principal through the same allocation, so they trigger under a repayment-like rule, but only logic placed in the processors or in `makeRefund` sees them
  - a broad trigger set includes CHARGE_ADJUSTMENT (a charge adjustment of the penalty itself), CHARGE_REFUND and the INTEREST_REFUND created with each MIR or payout, each able to raise a new penalty
  - a refund for an active loan returns money; it is not a prepayment

### 13.8 Delinquency
- overdue days from the oldest installment not obligations-met, penalty outstanding included (`LoanDelinquencyDomainServiceImpl.java:55-118`); installment level :227-257; grace days
- ranges by days only (`DelinquencyRange.java:44-47`); installment-level delinquency needs a bucket (`LoanProductDataValidator.java:840-852`)
- tag set after repayment (`LoanAccountDomainServiceJpa.java:287`) and after add charge (`LoanChargeWritePlatformServiceImpl.java:287`)
- overdue-penalty selection skips only obligations-met installments (`LoanReadPlatformServiceImpl.java:1610`); % base principal + interest outstanding (:1625-1626; `LoanChargeService.java:747-772`)
- tests `LoanDelinquencyDomainServiceTest` (provider `portfolio/deliquency/`), `SetLoanDelinquencyTagsBusinessStepTest`; feature LoanDelinquency
- **For a prepayment penalty**: a penalty due in the current period makes nothing overdue now; left unpaid past that installment's due date it keeps the installment open, so overdue days and buckets move, and the COB overdue penalty can stack on it (a flat one; a percentage one on principal + interest outstanding gives 0 when only the penalty is open) (interpretation)

### 13.9 COB steps that touch charges
- 11 loan step beans (§1.4); 6 seeded by default
- **APPLY_CHARGE_TO_OVERDUE_LOANS** `ApplyChargeToOverdueLoansBusinessStep.java:30-55` and the job `ApplyChargeToOverdueLoanInstallmentTasklet` (97 lines): first OVERDUE_INSTALLMENT product charge, wait period and backdate flag (`LoanReadPlatformServiceImpl.java:1590-1629`; configs `penalty-wait-period` `GlobalConfigurationConstants.java:30`, `backdate-penalties-enabled` :42); creation through `applyOverdueChargesForLoan` (`LoanChargeWritePlatformServiceImpl.java:795`) with a replay like add charge (:831-851)
- **ADD_PERIODIC_ACCRUAL_ENTRIES** charge accrual by due date or submitted date (`LoanAccrualsProcessingServiceImpl.java:1196-1199`; config `charge-accrual-date` `GlobalConfigurationConstants.java:73`)
- **LOAN_INTEREST_RECALCULATION** replays (§13.1); **ACCRUAL_ACTIVITY_POSTING** sums installment penalty charges (§11.5); **LOAN_DELINQUENCY_CLASSIFICATION** reads penalty outstanding (§13.8)
- tests `ApplyChargeToOverdueLoansBusinessStepTest`, `ApplyChargeToOverdueLoanInstallmentTaskletTest`, `LoanCOBCreateAccrualsTest` (integration-tests), `LoanInterestRecalculationCOBTest`
- **For a prepayment penalty**: a user transaction triggers it, so no new COB step is needed; COB recalculation and periodic accrual must leave the penalty count and amount as they are, and periodic accrual must not accrue it a second time

## 14. The OVERDUE_INSTALLMENT precedent

- **Counts** (`git grep` at the pin)
  - literal `OVERDUE_INSTALLMENT`: 12 files, 26 matching lines, including 2 savings false positives (`NO_OF_OVERDUE_INSTALLMENTS`, `DepositAccountReadPlatformServiceImpl.java:941, :1022`) and 6 test lines (`BaseLoanIntegrationTest.java:1535`; `ChargesHelper.java:61, :382, :466, :484, :501`); with `-w`: 9 files, 18 lines
  - `OVERDUE_INSTALLMENT|isOverdueInstallmentCharge` in `*/src/main/*.java`: 22 files, 43 lines (21 real files plus the savings false positive)
  - `OVERDUE_INSTALLMENT|isOverdueInstallment|OverdueInstallmentCharge|overdueInstallmentCharge` in `*.java`: 28 files, 26 outside tests (25 real)
- **Per file** (broad regex count; role and lines)

| file | count | role (lines) |
|---|---|---|
| `ChargeTimeType.java` | 6 | value 9, valid loan sets, `isOverdueInstallment` |
| `Charge.java` | 4 | must be penalty :278; update recheck :596; `isOverdueInstallment` :672-673 |
| `ChargeData.java` | 4 | read-model flag :129-134 |
| `ChargeEnumerations.java` | 3 | label "Overdue Fees" :77-79 |
| `ChargeDropdownReadPlatformServiceImpl.java` | 1 | loan time options :87 |
| `LoanCharge.java` | 6 | `isOverdueInstallmentCharge` :245-246; link field :138 |
| `LoanOverdueInstallmentCharge.java` | 3 | charge to installment link :32 |
| `Loan.java` | 1 | backdated-charge flag :1267 |
| `LoanRepaymentScheduleProcessingWrapper.java` | 2 | keeps the stored amount :94, :190 |
| `SingleLoanChargeRepaymentScheduleProcessingWrapper.java` | 1 | :121 |
| `AbstractCumulativeLoanScheduleGenerator.java` | 2 | per-period charges :2097, :2159 |
| `ProgressiveLoanScheduleGenerator.java` | 1 | % case :407 |
| `LoanRescheduleRequestDataValidatorImpl.java` | 1 | reschedule block :234 |
| `LoanChargeApiJsonValidator.java` | 1 | add through the API rejected :385 |
| `LoanBalanceService.java` | 1 | undo last tranche :183 |
| `LoanChargeReadPlatformService.java` | 1 | :44 |
| `LoanChargeReadPlatformServiceImpl.java` | 4 | dedupe query :250-257 |
| `LoanChargeService.java` | 12 | :50, :90, :116, :219, :245, :493, :538-543, :749 |
| `LoanChargeWritePlatformServiceImpl.java` | 7 | job :805, :1081, :1126, :1128; :105; :778; manual add rejected :1008 |
| `LoanWritePlatformServiceJpaRepositoryImpl.java` | 1 | undo disbursal :3167 |
| `LoanReadPlatformServiceImpl.java` | 3 | options :1493, :1496; product charge :1599 |
| `LoanAccountData.java` | 1 | template skip :370 |
| `LoanProductData.java` | 1 | :1345 |
| `LoansApiResource.java` | 3 | :1220, :1223, :1262 |
| `LoanChargesApiResource.java` | 1 | template excludes :522 |
| `DepositAccountReadPlatformServiceImpl.java` | 2 | false positive |
| `BaseLoanIntegrationTest.java` / `ChargesHelper.java` | 1 / 5 | tests |

- **What the precedent does**
  - created only by the COB step or the job: `applyChargeToOverdueLoanInstallment` (`LoanChargeWritePlatformServiceImpl.java:1076`) through `applyOverdueChargesForLoan` (:795), called from `ApplyChargeToOverdueLoansBusinessStep.java:41` and `ApplyChargeToOverdueLoanInstallmentTasklet.java:72`; dedupe by charge, installment and frequency number (`LoanChargeReadPlatformServiceImpl.java:250-258`)
  - reads the rule from the product at run time (`LoanReadPlatformServiceImpl.java:1598-1599`), never attached at loan creation (§7.4)
  - keeps its own base in `amountPercentageAppliedTo` (command params `principal` / `interest`, `LoanChargeAssembler.java:258-318`), special-cased at the re-derivation sites (§7.6)
  - never re-created by replay; deactivated only by `deactivateOverdueLoanCharge` (`LoanChargeWritePlatformServiceImpl.java:752-791`), undo disbursal (`LoanWritePlatformServiceJpaRepositoryImpl.java:3166-3168`), undo last tranche (`LoanBalanceService.java:183-184`) and foreclosure (every charge due after the date, `LoanAccountDomainServiceJpa.java:991-992`)
  - definition rule "must be a penalty" as a 403 domain exception (`Charge.java:278-280`)
  - attaching it rejected, also 403 (`LoanChargeCannotBeAddedException`, `LoanChargeCannotBeAddedException.java:26`): manual add (`LoanChargeWritePlatformServiceImpl.java:1008-1011`) and a `charges[]` entry on loan create or modify (`LoanChargeApiJsonValidator.java:385-389`, called from `LoanApplicationValidator.java:559, :1267`)
  - repo TODO on domain vs validation exception at `LoanChargeApiJsonValidator.java:382`, on the currency check just above
  - left out of templates and charge options (`LoanAccountData.java:370`; `LoanReadPlatformServiceImpl.java:1490-1497`; `LoanChargesApiResource.java:522`)
  - events: loan-level `LoanApplyOverdueChargeBusinessEvent` plus `LoanBalanceChangedBusinessEvent`, no add-charge event (§12.1)
  - blocks reschedule when due after the from-date (§13.3); counts in the progressive backdated-charge flag (`Loan.java:1267`)
- **What it does not give a prepayment penalty**: no link to a transaction, no create or remove during replay, no part in any quote, no per-transaction base

## 15. Build and tests

### 15.1 Toolchain and version
- Java 21 toolchain (`build.gradle:388-394`); Gradle 8.14.3 (`gradle/wrapper/gradle-wrapper.properties`); `gradle.properties`: Xmx12g, build cache, parallel, configuration cache off
- version `0.0.0-SNAPSHOT` (`build.gradle:133`), replaced by `gitVersioning` (:135-154): release and maintenance branches and version tags take the tag version; other refs `major.(minor + 1).0-SNAPSHOT` from `git describe`
  - both checkouts used for these notes carry 15 tags; `git describe --tags` gives `1.14.0-208-g8a61e30a4`, so 1.15.0-SNAPSHOT (the attempt's jar is `fineract-provider-1.15.0-SNAPSHOT.jar`); behaviour without tags (unverified)
- repositories: Gradle plugin portal (`build.gradle:85-87`) and Maven Central (:167-169; also `buildSrc/build.gradle:40-41`, `fineract-doc`, the e2e modules); `repository.apache.org` only for publishing (:874-878)

### 15.2 Commands as written in the repo
- boot JAR `./gradlew clean bootJar` (`README.md:88`); WAR `./gradlew :fineract-war:clean :fineract-war:war` (:112); image `./gradlew :fineract-provider:jibDockerBuild -x test` (:136); `docker compose -f docker-compose-development.yml up -d` (:148)
- local dev: `./gradlew createDB -PdbName=fineract_tenants`, `./gradlew createDB -PdbName=fineract_default`, `./gradlew devRun` (`README.md:56-58`); DB tasks in `fineract-provider/build.gradle`: `createDB` (MariaDB) :164, `createPGDB` :180, `createMySQLDB` :196, plus drop tasks
- unit tests: `./gradlew test -x :twofactor-tests:test -x :oauth2-tests:test -x :integration-tests:test` (`CONTRIBUTING.md:23`)
- everything: `./gradlew --no-daemon --console=plain test -x :twofactor-tests:test -x :oauth2-tests:test :fineract-e2e-tests-runner:test -PdbType=postgresql` (`CONTRIBUTING.md:77-78`); the file says this "often take[s] an hour or longer" (:80); not measured here
- integration CI per shard: `./gradlew "$module:test" $test_args -PdbType=postgresql -x checkstyleJmh -x checkstyleMain -x checkstyleTest -x spotlessCheck -x spotlessApply -x spotbugsMain -x spotbugsTest -x javadoc -x javadocJar -x modernizer` (`.github/workflows/build-postgresql.yml:122`); 5 shards from `scripts/split-tests.sh 5 $SHARD_INDEX` (:94); databases first with `./gradlew --no-daemon -q createPGDB -PdbName=fineract_tenants` and `-PdbName=fineract_default` (:80-81); job timeout 60 min; TZ Asia/Kolkata
- cucumber CI: build `./gradlew --no-daemon build -x test -x cucumber -x doc` (`.github/workflows/build-cucumber.yml:61`); unit-level features `./gradlew --no-daemon cucumber -x :fineract-e2e-tests-runner:cucumber ...` (:64); embeddable generator `shadowJar`, test and `misc/Main.java` (:72-87)

### 15.3 Static checks
- `check` depends on rat, licenseMain, licenseTest (`build.gradle:485-487`); provider `check` also on `cucumber` (`fineract-provider/build.gradle:31`)
- Spotless: Eclipse formatter `config/fineractdev-formatter.xml`, import order, unused imports, modifier order (`build.gradle:495-534`); `./gradlew spotlessApply` / `spotlessCheck` (`CONTRIBUTING.md:187, :193`)
- Checkstyle 11.0.0 + sevntu 1.44.1 (`config/checkstyle/checkstyle.xml`, `suppressions.xml`; `build.gradle:544-547`)
- SpotBugs (`config/spotbugs/exclude.xml`, :731); modernizer with failOnViolations (:752-762)
- Error Prone 2.35.1, on only when a task name contains build or check (:566-572; error list :597-671)
- `-Werror` only with `-Pwarnings=fail` (:412-414); Jacoco 0.8.12; no PMD
- others: `./gradlew rat` (`CONTRIBUTING.md:163`), `./gradlew doc` (:171), `./gradlew clean build jacocoTestReport` (:208)

### 15.4 Unit tests
- frameworks: JUnit 5 (jupiter api / engine / params, `junit-platform-suite`), Mockito (core, junit-jupiter, inline), Awaitility, spring-boot-starter-test (`build.gradle:693-717`; `fineract-loan/dependencies.gradle:80-88`; same in provider and progressive-loan)
- JUnit classes by the CI rule (`scripts/split-tests.sh`: files under `*/src/test/java/`, not `Abstract*Test`, containing `@Test`, `@Nested` or `@ParameterizedTest`): 490 in all; integration-tests 272, fineract-provider 146, core 14, progressive-loan 13, client-feign 12, investor 10, fineract-loan 8, 15 in 9 other modules
- **Provider** (most loan unit tests), `fineract-provider/src/test/java/org/apache/fineract/`
  - test sources 186 `.java`, 148 `*Test.java`, 146 JUnit classes
  - `portfolio/loanaccount`: 21 files, 20 test classes plus `domain/LoanBuilder`: `LoanTest`, `DefaultLoanLifecycleStateMachineTest`, `LoanChargeEffectiveDueDateComparatorTest`, `ReAgePreviewRequestValidationTest`, the two processor tests `DuePenFeeIntPriInAdvancePriPenFeeIntLoanRepaymentScheduleTransactionProcessorTest` and `DuePenIntPriFeeInAdvancePenIntPriFeeLoanRepaymentScheduleTransactionProcessorTest`, `ApplyChargeToOverdueLoanInstallmentTaskletTest`, `GenerateLoanlossProvisioningTaskletTest`, `DefaultScheduledDateGeneratorTest`, `LoanAccrualActivityProcessingServiceImplTest`, `LoanAccrualsProcessingServiceImplTest`, `LoanAdjustmentServiceImplTest`, `LoanCalculateRepaymentPastDueServiceTest`, `LoanChargeWritePlatformServiceImplTest`, `LoanDownPaymentHandlerServiceImplTest`, `LoanStatusChangeHistoryListenerTest`, `LoanWritePlatformServiceJpaRepositoryImplTest`, `ReplayedTransactionBusinessEventServiceIntegrationTest`, `LoanReAgingValidatorTest`, `LoanReAmortizationValidatorTest`
  - `cob/loan`: 13 files (9 JUnit classes, 4 Cucumber step definitions); `accounting/journalentry/CreateJournalEntriesForChargeOffLoanTest`; `portfolio/deliquency/LoanDelinquencyDomainServiceTest`
  - 20 Cucumber `.feature` files under `fineract-provider/src/test/resources/features/` (cob 7, infrastructure 6, others)
- **fineract-loan** 8 classes: three exception-mapper tests, `LoanTransactionDataTest`, `LoanRepaymentScheduleInstallmentTest`, `SingleLoanChargeRepaymentScheduleProcessingWrapperTest`, `LoanTransactionMapperTest`, `LoanProductTest` (plus 2 `.gitkeep`)
- **fineract-progressive-loan** 13 classes: `AdvancedPaymentScheduleTransactionProcessorTest`, `ChangeOperationTest`, `LoanScheduleGeneratorTest`, `ProgressiveLoanRescheduleRequestDataValidatorTest`, `ProgressiveEMICalculatorTest`, `InterestPeriodTest`, `RepaymentPeriodTest`, `AdvancedPaymentAllocationsJsonParserTest`, `AdvancedPaymentAllocationsValidatorTest`, `CreditAllocationsJsonParserTest`, `CreditAllocationsValidatorTest`, `LoanProductCreditAllocationRuleMergerTest`, `LoanProductPaymentAllocationRuleMergerTest`
- **DB use**: loan, provider and progressive unit tests use Mockito and no DB (per class not audited, unverified); `fineract-command` tests start MariaDB, MySQL and PostgreSQL Testcontainers (`fineract-command/src/test/java/org/apache/fineract/command/CommandBaseTest.java:32-47`)

### 15.5 Integration tests
- `integration-tests/`: 421 `.java`, 266 `*Test.java`, 272 JUnit classes by the CI rule; 35 `.java` files with charge or penalty in the name (32 `*Test.java`)
- near the feature: `LoanPrepayAmountTest`, `LoanPayOffTest`, `LoanPayOffAddChargeWithRefundTest`; 7 files mention `prepayLoan|getPrepayAmount|foreclosure` (`AdvancedPaymentAllocationLoanRepaymentScheduleTest`, `BaseLoanIntegrationTest`, `ClientLoanIntegrationTest`, `ExternalIdSupportIntegrationTest`, `LoanInterestRecalculationCOBTest`, `LoanPayOffTest`, `common/loans/LoanTransactionHelper`)
- **Server start** (`integration-tests/build.gradle`)
  - Cargo plugin `com.bmuschko.cargo`, container `tomcat10x` (:41), deploys `fineract-war/build/libs/fineract-provider.war` at context `fineract-provider`, HTTPS 8443 (:84)
  - database from `-PdbType=postgresql|mysql`, default MariaDB (:64-70)
  - JVM args `-Dspring.profiles.active=test -Dfineract.events.external.enabled=true` (:75): integration tests run with external events on
  - `test` depends on `cargoStartLocal` and `waitForFineract`, finalized by `cargoStopLocal` (:142-146); `waitForFineract` (:99) polls `/actuator/health` (:118), default 600 s (:101)
  - `-PcargoDisabled` skips Cargo and uses a server already running (:143; e.g. `.github/workflows/smoke-messaging.yml:62` with `--tests`)
  - the two databases must exist first (§15.2)
- **How a test reaches the server**
  - `ConfigProperties.java:25-34`: env `BACKEND_PROTOCOL` (https), `BACKEND_HOST` (localhost), `BACKEND_PORT` (8443), `BACKEND_USERNAME`, `BACKEND_PASSWORD`, `BACKEND_TENANT` (default)
  - RestAssured `common/Utils.java:126-136` (`initializeRESTAssured`, Basic auth); generated Retrofit client `common/FineractClientHelper.java` (`fineract.it.url` or `/fineract-provider/api/`, `insecure(true)`)
  - business date moved through the API (`BaseLoanIntegrationTest.runAt` :1334)
- **Helpers**
  - `BaseLoanIntegrationTest.java` (1,967 lines; `@ExtendWith({LoanTestLifecycleExtension, ExternalEventsExtension})` :138): GL accounts :156-179, helpers :180-192; `addCharge(loanId, isPenalty, amount, dueDate)` :1159; `createOverduePenaltyPercentageCharge` :1176; `runAt` :1334, `runFromToInclusive` :1322, `runAsNonByPass` :1347; `verifyTransactions` :917, `verifyJournalEntries` :1112, `verifyRepaymentSchedule` :1213, `getPrepayAmount` :297, `verifyPrepayAmountByRepayment` :301; payment allocation builders :1931, :1946
  - `common/loans/LoanTransactionHelper.java` (3,219 lines): many RestAssured methods `@Deprecated(forRemoval = true)` ("Rewrite to use fineract-client"); client methods `makeLoanRepayment(Long, PostLoansLoanIdTransactionsRequest)` :913, `executeLoanTransaction` :908, `addLoanCharge` :1391, `waiveLoanCharge` :1129, `forecloseLoan` :3052 / :3057 (:1377 is the deprecated RestAssured one), `getLoanCharges` :470
  - `common/charges/ChargesHelper.java` (814 lines): time-type constants 1-3, 5-11, 13-15 (none for 12) and calculation types 1-4 (no 5) (:49-79); `getLoanSpecifiedDueDateJSON(calc, amount, penalty)` :292; `getLoanOverdueFeeJSON*` :462-495; `createCharges` :600
  - `common/loans/LoanProductTestBuilder.java` (945 lines), `common/loans/LoanTestLifecycleExtension.java`, `common/BusinessDateHelper.java`, `common/SchedulerJobHelper.java`, `common/GlobalConfigurationHelper.java`, `common/accounting/JournalEntryHelper.java`, `inlinecob/InlineLoanCOBHelper.java`
- **Generated clients**: provider `resolve` writes `build/resources/main/static/fineract.json` from Swagger annotations (`fineract-provider/build.gradle:58-69`; `swaggerFile` `build.gradle:157`); `fineract-client` and `fineract-client-feign` generate from it (`fineract-client/build.gradle:27-43`); a new request or response field reaches typed tests only through Swagger

### 15.6 Cucumber e2e
- `fineract-e2e-tests-runner`: 45 features, 2,177 scenarios (`Scenario:` + `Scenario Outline:`), e.g. `LoanCharge.feature` 125, `LoanRepayment.feature` 152, `LoanAccrualTransaction.feature` 38; 3 mention prepay or foreclosure (EMICalculation, LoanChargeback, LoanRepayment)
- build (`fineract-e2e-tests-runner/build.gradle`): `se.thinkcode.cucumber-runner` 0.0.11 + Allure 2.12.0; feature path `-Pcucumber.features` (default `src/test/resources/features`); tags `not @Skip`; `cucumber` depends on `spotlessCheck`, finalized by `allureReport`
- initializers `fineract-e2e-tests-runner/.../test/initializer/{global,scenario,suite}/` (`ChargeGlobalInitializerStep` 239 lines; `LoanProductGlobalInitializerStep` 4,705 lines)
- core `fineract-e2e-tests-core/src/test/java/org/apache/fineract/test/`: `stepdef/loan/` 16 classes, 9,969 lines (`LoanStepDef` 5,945, `LoanChargeStepDef` 342, `LoanRepaymentStepDef` 684); `factory/` (`LoanChargeRequestFactory`, `LoanProductsRequestFactory`); `data/` enums (`ChargeProductType` 17 values, `DefaultLoanProduct` 170 values, copies of `ChargeTimeType` and `ChargeCalculationType`); `messaging/event/` 60 files
- server settings `fineract-e2e-tests-core/src/test/resources/fineract-test-application.properties`: `BASE_URL` (default https://localhost:8443), `INITIALIZATION_ENABLED` and `EVENT_VERIFICATION_ENABLED` (default false), ActiveMQ
- CI `.github/workflows/build-e2e-tests.yml`: env turns initialization and event verification on, ActiveMQ `tcp://localhost:61616` (:24-31); `./gradlew --no-daemon --console=plain :fineract-provider:jibDockerBuild -Djib.to.image=$IMAGE_NAME -x test -x cucumber` (:60); `docker compose -f docker-compose-postgresql-test-activemq.yml up -d` (:63) and a health curl; `./gradlew --no-daemon --console=plain :fineract-e2e-tests-runner:cucumber -Pcucumber.features="$feature_file" ... allureReport` (:115-119); features split by `scripts/split-features.sh`
- `.feature` files in the repo: 67 = 45 e2e + 20 provider + 2 custom (`custom/acme/loan/starter/.../cob.feature`, `custom/acme/note/starter/.../note.feature`)
- 11 compose files at the root (`docker-compose-postgresql.yml`, `docker-compose-mariadb.yml`, `docker-compose-postgresql-test-activemq.yml`, `docker-compose-postgresql-activemq.yml`, `-kafka.yml`, `-kafka-msk.yml`, `docker-compose-development.yml`, `docker-compose.yml`, `docker-compose-custom.yml`, `docker-compose-web-app.yml`, `docker-compose-community-app.yml`); env files `config/docker/env/` (`activemq.env:23-26` turns external events on)

## 16. Conventions a new feature must follow

- **Schema**: a new Liquibase part (core `parts/0210_*` plus one include line, or loan module `1035_*`); applied parts are never edited (§1.3)
- **Enums**: append new values (`ChargeTimeType` 17, `ChargeCalculationType` 6, `LoanTransactionType` 44); `LoanTransactionRelationTypeEnum` is stored by ordinal, so a new value goes last and `fromInt` (:47-59) gets a case; `ChargeEnumerations` needs a case and label per value (:38 on); the loan time list in `ChargeDropdownReadPlatformServiceImpl.java:85-88` is hard-coded; e2e enum copies (`fineract-e2e-tests-core/.../test/data/`) and `ChargesHelper` constants are separate
- **Errors**: field errors as `ApiParameterError` collected by `DataValidatorBuilder` (`DataValidatorBuilder.java:71-430`, `throwValidationErrors` :1100) and thrown as `PlatformApiDataValidationException` (400); domain rules as `AbstractPlatformDomainRuleException` subclasses (403, `PlatformDomainRuleExceptionMapper.java:53`); unsupported JSON keys rejected with `checkForUnsupportedParameters` (e.g. `ChargeDefinitionCommandFromApiJsonDeserializer.java:99-102, :482-484`)
- **Commands**: writes go through `CommandWrapperBuilder`, a `@CommandType` handler and an `m_permission` row (§1.6); input `JsonCommand` / `FromJsonHelper`; output `CommandProcessingResult` / `CommandProcessingResultBuilder`
- **Money and dates**
  - `Money` (`fineract-core/.../organisation/monetary/domain/Money.java`, 497 lines): immutable currency + amount; constructor rounds (`:40-52`); `of`, `zero`, `plus`, `minus`, `multipliedBy`, `dividedBy(..., mc)`, `percentageOf(BigDecimal, mc)` :405; `minus` returns a new object (the progressive quote bug in §8.1 ignores it)
  - `MoneyHelper`: per-tenant `RoundingMode` and `MathContext(19, mode)` (:35-93; `getMathContext` :91; `createMathContext` :123), from global config `rounding-mode` (`MoneyHelperInitializationService.java:57-105`; seed `parts/0002_initial_data.xml:219-220`; default 6 HALF_EVEN)
  - `MathUtil` (516 lines): null-safe BigDecimal and Money helpers
  - `DateUtils` (477 lines): `getBusinessLocalDate` :235-237, `isBeforeBusinessDate` :251, `isAfterBusinessDate` :255, `isDateInRangeFromInclusiveToExclusive` :263, `isDateInRangeInclusive` :404, `getLocalDateOfTenant` :67, `getAuditOffsetDateTime` :107; never `LocalDate.now()` for business logic
- **External ids**: `ExternalId` (`generate`, `empty`, `isEmpty`); `ExternalIdFactory.java:45-66` (`createFromCommand`, `create`; auto-generation by config); JPA `ExternalIdConverter` in provider
- **Lombok**: 1,775 of 4,736 main files; entities `@Getter` / `@Setter` (`Loan` `@Getter`, `LoanCharge` `@Getter @Setter` :58-59); services `@RequiredArgsConstructor`; DTOs `@Builder` (e.g. `LoanTransactionData.builder()`)
- **MapStruct**: 60 main files; config `MapstructMapperConfig.java:28` (Spring component model, `unmappedTargetPolicy = ERROR`, no builders), so a new target field without a mapping breaks the build; loan mappers `fineract-loan/.../portfolio/loanaccount/mapper/` (`LoanChargeMapper`, `LoanTransactionMapper`, `LoanChargePaidByMapper`, `LoanTransactionRelationMapper`); Avro mappers use `AvroMapperConfig`
- **Enum to API**: `EnumOptionData` (id, code, value); `ChargeEnumerations` (217 lines); `LoanEnumerations` (570 lines); newer enums build their own lists (e.g. `PaymentAllocationType.getValuesAsEnumOptionDataList`)
- **Swagger**: 98 `*ApiResourceSwagger.java` beside 168 `*ApiResource.java`; loan ones `LoanTransactionsApiResourceSwagger`, `LoanChargesApiResourceSwagger` (fineract-loan), `LoansApiResourceSwagger` (provider), `ChargesApiResourceSwagger` (fineract-charge); generated clients depend on them (§15.5)
- **Spring wiring**
  - component scan `org.apache.fineract.**` (`FineractWebApplicationConfiguration.java:52`); 69 main files in `*/starter/` packages; only `fineract-command` and the `custom/acme` starters use `AutoConfiguration.imports`
  - `LoanAccountConfiguration.java` (609 lines): 35 `@Bean` methods, each `@ConditionalOnMissingBean`, e.g. `LoanChargeWritePlatformService` :310, `LoanWritePlatformService` :416, `LoanChargeService` :506, `LoanReadPlatformService`; those impls carry no `@Service`, so a new constructor dependency also changes this file
  - `@Service` / `@Component` + `@RequiredArgsConstructor`: `LoanAccountDomainServiceJpa`, `LoanAccrualsProcessingServiceImpl`, `ReprocessLoanTransactionsServiceImpl`, `LoanTransactionProcessingServiceImpl`, all command handlers
  - processors in `LoanAccountAutoStarter` (§6.1); progressive extras `ProgressiveLoanAccountConfiguration`; `ChargeConfiguration`, `LoanProductConfiguration`, `AccountingJournalEntryConfiguration`, `AccountingAccrualConfiguration`; `@ConditionalOnMissingBean` lets `custom/` modules replace beans
- **Module boundaries**: `fineract-loan` cannot see provider or progressive-loan classes (§2); shared logic for both processors sits in fineract-loan
- **Events**: reuse existing classes or add a class plus a seed part, the helper list and the validation test list (§12.3); or `NoExternalEvent`
- **Tests**: integration tests extend `BaseLoanIntegrationTest`, use the generated client (RestAssured helpers are deprecated for removal) and `runAt` for dates; unit tests with Mockito; existing test expectations stay as they are
- **Docs and contribution**
  - feature doc `fineract-doc/src/docs/en/chapters/features/loan-charges.adoc` (121 lines) tables the loan charge time types and calculation types (:15 on)
  - contribution checklist `.github/pull_request_template.md:9-14`: commit message per the guidelines, green build, unit or integration tests, coding conventions, Swagger annotations and `legacy-docs/apiLive.htm` for API changes, no code dump
  - commit message "FINERACT-NNNN: Imperative summary" (`CONTRIBUTING.md:254-255`); a single commit per contribution expected (:282)
  - license headers checked by rat and licenseMain / licenseTest (§15.3)

## 17. One finished attempt at the narrower version

### 17.1 Run facts
- **Input**: the 10-requirement spec (43 lines)
  - new time type "Prepayment" for penalties, flat or % of prepaid principal, caps, lock-in months, none in the last installment period
  - prepay and foreclosure quotes that close the loan; undo / adjust / backdated / waived handling; both schedule types; accounting through the existing penalty rules; no change for other loans
  - acceptance: HTTP 400 for every bad combination, listed cumulative ITs, progressive partial prepayment and payoff, accrual journal test, existing tests unchanged; out of scope savings, client charges, web app
- **Duration**: reported 96 minutes; file times run from 07:52 (spec file) to 09:42 (checkstyle log); what bounds the 96 minutes (unverified)
- **Setup**: PostgreSQL; boot JAR `fineract-provider-1.15.0-SNAPSHOT.jar` started directly with `spring.profiles.active=test`, external events on; a helper script rebuilt the JAR (`:fineract-provider:bootJar -x test`), restarted the server and waited for `/actuator/health` UP; integration tests through `:integration-tests:test -PcargoDisabled` with `--tests` filters and `--max-workers=1`
- **Timeline** (log mtimes, UTC): 8 build logs 08:05-08:32; DB init 08:31; spotless 08:33; IT compile 08:49; last main-code edit 09:27:15; boot JAR 09:27:53; it1 09:29:39; IT file last modified 09:30:07; it2 09:30:32; unit 09:33; it-reg1 09:37; it-reg2 09:41; checkstyle 09:42
- **Results**
  - it1: 19 tests, SUCCESS (39.2 s), on the final main code with an earlier version of the IT file
  - it2: only `testProgressivePartialPrepaymentAndFullPayoff` and `testProgressiveUndoAndBackdatedRepayment` on the final IT file, SUCCESS; 13 javac warnings: 11 [removal] deprecations (`Utils.performServerPost` / `Get` / `Put`, `ChargesHelper.populateDefaultsForLoan`, `JournalEntryHelper.getJournalEntries`, `LoanTransactionHelper.addChargesForLoan`), one [static] (IT :70), one annotation-processor notice
  - unit: fineract-loan 42, fineract-progressive-loan 177, fineract-provider 921 tests, all passed; checkstyle passed
  - it-reg1: 82 tests in 11 classes (AccrualsOnLoanClosure, Charges, ClientLoanChargeRefund, ExternalIdSupport, LoanAccrualTransactionReversal, LoanChargeProgressive, LoanChargeSpecificDueDate, LoanPayOff, LoanPrepayAmount, LoanTransactionChargeback, LoanWaiveCharge); it-reg2: 102 tests in 6 classes (AdvancedPaymentAllocationWaiveLoanCharges, ClientLoanIntegration, LoanAccountChargeReveseReplayWithAdvancedPaymentAllocation, LoanCOBCreateAccruals, LoanChargePaymentWithAdvancedPaymentAllocation, LoanInterestRecalculationCOB); neither run included the new class
  - not run: the other 255 of the 272 existing integration-test classes, Cucumber e2e
  - its own final report: 19 new ITs green; 2 bugs found and fixed in testing (lock-in silently dropped by the request mapper; an early flush made the API return a reversed repayment's id); disclosed as open: foreclosure penalties not recalculated on later replays, the `charges[]` route for the charge untested, spotless not run
  - against the record: the IT file was edited after the 19-green run and only 2 of its 19 tests ran on the final version; the quote-for-every-caller mismatch, the 1-of-30 replay hook and undo disbursal were not among the disclosed open items (§17.5)

### 17.2 Files it touched
- 26 tracked files, +530 / -37; 4 new files, 1,280 lines; `.../` = `src/main/java/org/apache/fineract/`

| file | + / - | change |
|---|---|---|
| `fineract-charge/.../portfolio/charge/domain/Charge.java` | 95 / 5 | `lock_in_period_months` column and param; static `validatePrepaymentChargeDefinition` :388, called on create :303 and update :659; caps stored for % prepaid principal; `isPrepaymentCharge` :375 |
| `fineract-loan/.../portfolio/loanaccount/domain/transactionprocessor/AbstractLoanRepaymentScheduleTransactionProcessor.java` | 85 / 3 | generated penalties held out of the up-front charge set and mapped to triggers (:103-108); `processPrepaymentPenalties` after each new or existing transaction (:196, :210; method :251-275); `payPrepaymentPenaltyFromOverpayment` :277 on |
| `fineract-charge/.../portfolio/charge/serialization/ChargeDefinitionCommandFromApiJsonDeserializer.java` | 51 / 2 | `lockInPeriodInMonths` param; prepayment checks :495; lock-in > 0; update calculation range 1-6 |
| `fineract-provider/.../portfolio/loanaccount/service/LoanChargeWritePlatformServiceImpl.java` | 48 / 0 | `addPrepaymentPenaltyRule` :180-218 (branch :239-241): submitted or approved loans only, no duplicate (:187-190), currency check, all as 400; `LoanAddChargeBusinessEvent`; no replay |
| `fineract-loan/.../portfolio/loanaccount/service/LoanChargeService.java` | 42 / 2 | % prepaid principal cases; rule rows through `applyPrepaymentPenaltyRuleFields` :883 (from :356, :401, :787, :834) |
| `fineract-progressive-loan/.../portfolio/loanaccount/domain/transactionprocessor/impl/AdvancedPaymentScheduleTransactionProcessor.java` | 34 / 3 | same split (:216-221); per trigger (:270-291): due date reset, amount recomputed unless waived or foreclosed (:277), `processSingleCharge`, `processOverpaidTransactions` |
| `fineract-provider/.../portfolio/loanaccount/service/LoanChargeAssembler.java` | 31 / 0 | a Prepayment definition in `charges[]` becomes one rule per definition (:123-126); `addProductPrepaymentPenaltyRules` :238-249; `hasPrepaymentPenaltyRule` :251 |
| `fineract-loan/.../portfolio/loanaccount/domain/Loan.java` | 18 / 1 | `getActiveCharges` drops rule rows (:1260); `getPrepaymentPenaltyRules` :1266; `hasPrepaymentPenaltyRule` :1271 |
| `fineract-charge/.../portfolio/charge/domain/ChargeCalculationType.java` | 17 / 3 | `PERCENT_OF_PREPAID_PRINCIPAL(6, "chargeCalculationType.percent.of.prepaid.principal")` :29; in `validValuesForLoan` :51; `validValuesForPrepayment` :77-78; `fromInt` :88; `isPercentageBased` :119; `isPercentageOfPrepaidPrincipal` :130-132 |
| `fineract-loan/.../portfolio/loanaccount/domain/LoanCharge.java` | 16 / 0 | `isPrepaymentPenaltyRule` :245 (time 17); `isGeneratedPrepaymentPenalty` :253 (definition PREPAYMENT and loan charge SPECIFIED_DUE_DATE) |
| `fineract-core/.../portfolio/charge/domain/ChargeTimeType.java` | 16 / 3 | `PREPAYMENT(17, "chargeTimeType.prepayment")` :40; in `validLoanValues` :61; `fromInt` :138; `isAllowedLoanChargeTime` :198; `isPrepayment` :205-206 |
| `fineract-provider/.../portfolio/loanaccount/starter/LoanAccountConfiguration.java` | 14 / 2 | `LoanPrepaymentPenaltyService` bean (`@ConditionalOnMissingBean`), wired into the down-payment handler |
| `fineract-provider/.../portfolio/loanaccount/service/LoanReadPlatformServiceImpl.java` | 12 / 4 | payoff penalty added to the prepay template amount and `penaltyChargesPortion` (:582-590), for every caller; foreclosure template :2095 |
| `fineract-provider/.../portfolio/loanaccount/domain/LoanAccountDomainServiceJpa.java` | 11 / 3 | payoff penalty computed before the collapse (:727) and added to the payment; `createForeclosurePenalties` (:739); `updateInstallmentsPostDate` moved after the payment build for every loan (:742; pin :724) |
| `fineract-charge/.../portfolio/charge/service/ChargeEnumerations.java` | 8 / 0 | labels "Prepayment", "% Prepaid Principal" |
| `fineract-charge/.../portfolio/charge/api/ChargesApiResourceSwagger.java` | 5 / 0 | new field |
| `fineract-loan/.../portfolio/loanaccount/service/LoanDownPaymentHandlerServiceImpl.java` | 4 / 0 | `beforeRepaymentProcessing` :103; `afterRepaymentProcessing` :167 |
| `fineract-provider/.../portfolio/loanaccount/service/LoanAssemblerImpl.java` | 4 / 1 | product rules attached at creation (:213); `updateFrom` keeps rules (:463) |
| `fineract-provider/src/test/java/org/apache/fineract/portfolio/loanaccount/service/LoanDownPaymentHandlerServiceImplTest.java` | 4 / 1 | new mock only |
| `fineract-provider/.../portfolio/charge/service/ChargeReadPlatformServiceImpl.java` | 4 / 2 | SQL column |
| `fineract-charge/.../portfolio/charge/service/ChargeDropdownReadPlatformServiceImpl.java` | 3 / 2 | options |
| `fineract-charge/.../portfolio/charge/request/ChargeRequest.java` | 2 / 0 | field |
| `fineract-loan/.../portfolio/loanaccount/domain/LoanTransactionRelationTypeEnum.java` | 2 / 0 | `PREPAYMENT_PENALTY(6, "loanTransactionRelationType.prepaymentPenalty")` :29, appended; `fromInt` :58 |
| `fineract-provider/.../portfolio/loanaccount/loanschedule/service/LoanScheduleAssembler.java` | 2 / 0 | rule rows removed from schedule charges (:744) |
| `fineract-core/.../portfolio/charge/data/ChargeData.java` | 1 / 0 | field |
| `fineract-provider/src/main/resources/db/changelog/tenant/changelog-tenant.xml` | 1 / 0 | include 0210 |

- **New files**
  - `fineract-loan/src/main/java/org/apache/fineract/portfolio/loanaccount/domain/PrepaymentPenaltyHelper.java` (280): `getLinkedPenalties` :61; `sameCharge` :71; `mapPenaltiesToTriggers` :82-106; `withoutGeneratedPenalties` :111; `calculatePrepaidPrincipal` :118-129 (principal of the transaction's installment mappings whose installment is due after the transaction date, :124); `isAfterLockInPeriod` :135-141; `findLastRegularInstallment` :146-150; `isInLastInstallmentPeriod` :156-160; `findRule` :165; `calculatePenaltyAmount` :185-221; `calculatePrincipalOutstandingDueAfter` :239-248; `calculatePayoffPenalty` :253-265; `updatePenaltyAmount` :270-279
  - `fineract-loan/src/main/java/org/apache/fineract/portfolio/loanaccount/service/LoanPrepaymentPenaltyService.java` (340): `MAX_ITERATIONS = 5` :66; `beforeRepaymentProcessing` :80-91; `afterRepaymentProcessing` :98-120; `createForeclosurePenalties` :140-156; `removeObsoletePenalties` :158-171; `createMissingPenalties` :173-198; `isTriggerCandidate` :200-202; `createPenalty` :208-224; `link` :226-229; `removePenalty` :231-247; `reverse` :256-267; `syncAccruals` :269-304; `createAccrual` :311-324; `updateInstallmentPenaltyAccrued` :326-339
  - `fineract-provider/src/main/resources/db/changelog/tenant/parts/0210_add_charge_lock_in_period_months.xml` (32): `m_charge.lock_in_period_months` INT, nullable
  - `integration-tests/src/test/java/org/apache/fineract/integrationtests/LoanPrepaymentPenaltyIntegrationTest.java` (628, 19 `@Test`, extends `BaseLoanIntegrationTest`)

### 17.3 Design it chose
- **Two kinds of LoanCharge rows**: a rule row (time 17, no due date, amount 0) attached to the loan, and one generated penalty per trigger per rule (SPECIFIED_DUE_DATE, FLAT, due on the trigger date)
- **Attach**: product Prepayment charges added at creation even without JSON `charges` (`LoanAssemblerImpl.java:213`); before disbursement through POST `/loans/{id}/charges`, with a loan-level amount over the definition (3% over a 2% definition, IT :304-305); after disbursement rejected
- **Trigger** REPAYMENT only, not reversed, non-zero (`LoanPrepaymentPenaltyService.java:200-202`)
- **Creation** outside the processors, only on the shared handler path (`LoanDownPaymentHandlerServiceImpl.java:167`) and in foreclosure (`LoanAccountDomainServiceJpa.java:739`); then up to 5 passes of remove, create and full reprocess under `FlushModeType.COMMIT` (:103-117; comment on flush order :104-105); the loop ends without an error when the cap is hit; persist and accrual sync after (:118-119)
- **Link**: `LoanTransactionRelation.linkToCharge` from trigger to penalty, new type 6 (:226-229); carried to replacement transactions by `copyLoanTransactionRelations` (pin `createNewTransaction` :502); on adjust re-linked to the new transaction (`beforeRepaymentProcessing` :80-91)
- **Recalculation** inside both processors on every replay: due date reset to the replayed trigger date; amount recomputed unless waived or `loan.isForeclosure()` (abstract :265, advanced :277); penalty spread right after its trigger
- **Amount** (`calculatePenaltyAmount` :185-221): 0 when nothing prepaid, after lock-in or in the last period; flat value without caps; percentage then min then max (:214-219), so max wins when min > max (pin `LoanCharge.minimumAndMaximumCap`: min wins); rule values before the definition; lock-in read live from the definition (:192)
- **Payment**: a trigger never pays its own penalty except from its overpayment (cumulative `payPrepaymentPenaltyFromOverpayment`; advanced `processOverpaidTransactions`); otherwise outstanding, collected by later payments in strategy order
- **Removal**: undo or reversed trigger, or a zero amount when not waived: `setActive(false)` plus reversal of its accruals, accrual adjustments and waivers (`removePenalty` :231-247)
- **Accrual** (periodic and upfront): one accrual per penalty dated on its due date, journal Dr penalty receivable / Cr penalty income, `LoanAccrualTransactionCreatedBusinessEvent` (`createAccrual` :311-324); COB adds no second accrual (tested)
- **Quotes**: payoff penalty per rule on principal outstanding due after the quote date, lock-in and last-period rules applied (`calculatePayoffPenalty` :253-265); added in the prepay template for every caller and in the foreclosure template
- **Foreclosure**: penalty computed before the schedule collapse, paid inside the foreclosure repayment, linked penalties created after; recompute frozen for the loan afterwards
- **Validation**: definition errors as 400 (`Charge.java:388-428`): `validation.msg.charge.chargeTimeType.prepayment.only.for.loan`, `...penalty.prepayment.charge.must.be.penalty`, `...chargeCalculationType.not.allowed.for.prepayment`, `...percent.of.prepaid.principal.only.for.prepayment`, `...lockInPeriodInMonths.only.for.prepayment`, `...lockInPeriodInMonths.not.greater.than.zero`; loan-level add as 400: `validation.msg.loan.charge.prepayment.penalty.only.before.disbursement`, `...already.added`, `validation.msg.loan.charge.currency.mismatch`

### 17.4 Tests it wrote (`LoanPrepaymentPenaltyIntegrationTest.java`)
- loan shape: 1,000 EUR, 0% interest, no interest recalculation, single disbursement, 4 monthly installments, disbursed 01 Jan 2024
- definition and attach: validation matrix :90-141; loan-level 3% over a 2% definition gives 18 on 600; duplicate and post-disbursement 400 (:297-321); unaffected loan :324-338
- cumulative amounts: 2% partial, 600 on 15 Jan gives 12, NONE accounting gives no journal (:148-167); max cap :170-179; due amount only gives none (:182-196); flat with 2-month lock-in (:199-219); last period (:222-235)
- quotes: prepay template 1,020 (penalty 20), paid, loan closed (:238-256); payoff after a partial 420 (:259-276); foreclosure template 1,020 and `forecloseLoan` (:279-294)
- reversal and replay: adjust 600 to 300 gives 6, undo gives 0 (:341-363); backdated (:366-387): 600 on 20 Jan gives 12; backdated 300 on 10 Jan gives 6 and 11.88; backdated 10 on 05 Jan gives 3 penalties, amounts not asserted; waived stays waived (:390-405)
- progressive: partial + payoff (:412-445); undo + backdated (:448-476)
- accounting: accrual journal 12 Dr penalty receivable / Cr penalty income, inline COB no double (:483-507); accrual undo (:510-519); cash journal 1,020 / 1,000 / 20 (:522-531); helpers :537-627
- not covered: upfront accrual; MIR, payout, goodwill, IPW; chargeback; charge-off and write-off; re-age, re-amortize, reschedule; multi-disbursement; holidays; interest recalculation; non-default cumulative strategies; external events; the min cap's effect on an amount (min cap only in the validation test :93); lock-in on a percentage charge (lock-in only with FLAT :200); boundary days; an installment paid days before its due date (the due-only test pays on or after due dates, :180-195)
- no unit tests for `PrepaymentPenaltyHelper` or `LoanPrepaymentPenaltyService`

### 17.5 Gaps it left
- **Quote vs trigger**: the prepay template adds the penalty for every caller (`LoanReadPlatformServiceImpl.java:582-590`): MIR, payout refund, goodwill credit, IPW (pin `LoanTransactionsApiResource.java:634-649`) and contract termination (:668-677); only REPAYMENT creates one (`LoanPrepaymentPenaltyService.java:200-202`); an MIR for the quoted amount leaves the penalty part as excess (code reading, not run); top-up stays consistent because it repays with REPAYMENT
- **Two computations**: the quote reads principal outstanding from schedule rows due after the date (`PrepaymentPenaltyHelper.java:239-248`); the trigger reads the processed transaction's mappings (:118-129)
- **One hook of 30 replay sites**: create and remove only on the shared handler and in foreclosure; the other replays (charge add or waive, reschedule, re-age, re-amortize, chargeback, write-off, charge-off and undo, capitalized income, contract termination, COB recalculation) only recompute amounts inside the processors; MIR, payout and IPW never reach the hook; accrued amount can drift from the charge amount until the next repayment (code reading, not run)
- **Undo disbursal**: no hook; generated penalties stay active with the trigger reversed (pin `LoanWritePlatformServiceJpaRepositoryImpl.java:3166-3170`)
- **Foreclosure**: `loan.isForeclosure()` freezes recompute for every penalty on the loan, earlier partial-prepayment ones included (abstract :265, advanced :277; service :102-110 skips create and keeps zero-amount penalties); the shared foreclosure path is reordered for every loan (attempt :726-742 vs pin :724), with one foreclosure test (`testLoanForeclosure`) in the regression runs
- **Rule rows leak**: no due date, amount 0, hidden only from `getActiveCharges` and the schedule; the 25 raw accessor sites see them (§9.4, e.g. `InvestorBusinessEventSerializer.java:135`); `LoanAccrualsProcessingServiceImpl.java:646` dereferences the null due date under `charge-accrual-date` = submitted-date (code reading, not run); closure unaffected because amount <= 0 counts as paid (`DefaultLoanLifecycleStateMachine.java:303-304`)
- **Attach rule**: auto-attach at creation departs from the JSON-only rule and from the overdue precedent; no per-loan opt-out; a Prepayment definition sent in `charges[]` is coerced into a rule, so per-loan time and calculation overrides are not honoured
- **Snapshot vs live**: amount and caps copied into the rule row; lock-in read live (`PrepaymentPenaltyHelper.java:192`); generated penalties recognised from the live definition (`LoanCharge.java:253-255`); definitions stay editable while in use (pin `ChargeWritePlatformServiceJpaRepositoryImpl.java:140-158`), so a time-type edit orphans every generated penalty
- **Cascade**: an outstanding earlier penalty takes part of a later repayment before principal; 20 Jan penalty 12 becomes 11.88 after a backdated 300 on 10 Jan (6.00 of the 20 Jan repayment pays the 10 Jan penalty; IT :366-387)
- **Collection**: cumulative partial-prepayment penalties sit outstanding, due on the trigger date; strategies differ in when they are collected
- **Cost and honesty**: up to 5 extra full replays per repayment; the 5-pass cap stops without an error while still changing (`LoanPrepaymentPenaltyService.java:66, :103-117`)
- **Events**: none for create, recalculate or remove (:208-247); only `LoanAccrualTransactionCreatedBusinessEvent` (:323); the rule add raises `LoanAddChargeBusinessEvent`
- **Caps and rounding**: max wins when min > max, unlike the pin; any positive prepaid principal triggers the full min cap (`PrepaymentPenaltyHelper.java:189-191, :214-216`), so a 0.01 spill into the next installment gives a min-cap penalty; no min <= max check
- **Boundaries**: lock-in end = `disbursementDate.plusMonths(n)` with `isAfter`, so the end day is inside (:140); month ends clamp (30 and 31 January both end a 1-month lock-in on the last day of February); anchor = earliest tranche; last period excludes additional and down-payment installments but not re-aged ones (:148); a payment on the last installment's `fromDate` counts as outside the last period (:158); no test on any boundary day
- **Untested branches**: interest recalculation (progressive TILL_REST_FREQUENCY_DATE quote vs penalty base), multi-disbursement, holidays shifting due dates, the HeavensFamily in-advance definition, two concurrent repayments on one loan, accrual dated on a backdated trigger
- **Verification**: 19 green on an earlier version of the IT file, 2 re-run on the final file, regression runs without the new class (§17.1)

## 18. Corrections to older notes (line numbers refer to the idea file before its 2026-11-04 fix)

- **Idea file** `johto/ideas/fineract-prepayment-penalty.md`
  - `ChargeCalculationType` names (:31) are the e2e copy's (`fineract-e2e-tests-core/src/test/java/org/apache/fineract/test/data/ChargeCalculationType.java:23-27`); the main enum (`fineract-charge`, `ChargeCalculationType.java:23-28`) has INVALID 0, FLAT 1, PERCENT_OF_AMOUNT 2, PERCENT_OF_AMOUNT_AND_INTEREST 3, PERCENT_OF_INTEREST 4, PERCENT_OF_DISBURSEMENT_AMOUNT 5; its `ChargeTimeType` path in fineract-core is right
  - "11 repayment transaction processors" (:95): 10 concrete classes (9 cumulative + advanced); 11 when counting the Acme sample, or the abstract base that its own code map lists (:33-34); where the 11 came from is not known
  - `prepayLoan` as a command (:26, :38, :70): GET template only; the prepayment is POST `command=repayment`; `foreclosure` is a GET template and a POST command; stale `fineract-provider/src/main/resources/static/legacy-docs/apiLive.htm:13068-13073` still documents POST `prepayLoan`
  - unit tests "in fineract-loan and fineract-progressive-loan" (:41, :84): 8 and 13 classes there; most loan unit tests are in `fineract-provider/src/test` (§15.4); "need no DB": Mockito, not audited per class
  - "about 21 files" for OVERDUE_INSTALLMENT (:101): right for the narrow regex in main code (22 files, 21 real); literal 12 files; broad 26 non-test files (25 real)
  - "Backdated activity replays the whole history" (:22): a chronologically latest repayment on a loan without recalculation is processed alone (§6.6)
  - "Exclusions: final instalment on its due date" (:50): on the collapsed foreclosure schedule every foreclosure is in the final period (§8.4)
  - "Gradle compile takes about 10 min" (:84): not measured; git versioning (:85): tags present in both checkouts, no-tag behaviour (unverified)
  - "Gap still open at upstream HEAD `71281f17`" (:88-91): re-checked on `develop` at `06ce07b5` (2026-11-04): still no prepayment time type and 0 hits for prepaymentPenalty, earlyRepaymentFee or foreclosureFee; upstream added DOWN_PAYMENT(17) for loans after the pin; at the pin no prepayment value exists (`ChargeTimeType.java:23-39`)
  - confirmed: 17 `ChargeTimeType` values; loan charge times {1, 2, 8, 9, 12}; INCOME_FROM_PENALTIES mapping; four accounting modes; both generators implement `calculatePrepaymentAmount`; 5,667 Java files (4,736 main, 930 test); 266 `*Test.java` integration tests; 67 `.feature` files; Postgres and MariaDB compose files; only Maven Central and the plugin portal for the build; the EarlyPayment processor only changes allocation (in-advance principal only, :67-116)
- **Earlier research drafts** (all fixed in this file)
  - cumulative `calculatePrepaymentAmount` does not skip additional, down-payment or re-aged installments; that skip is `getPeriodInterestTillDate` (:2827)
  - HeavensFamily: the late handler delegates to on-time (:72-80); the in-advance handler has its own logic (:103-163)
  - ADJUSTMENT relation only at `BuyDownFeeWritePlatformServiceImpl.java:160`, `CapitalizedIncomeWritePlatformServiceImpl.java:150`, `LoanTransactionRepository.java:377`; the amortization services use RELATED
  - charge-off commands are `charge-off` / `undo-charge-off`; `undowriteoff` is a POST command too
  - `minimumAndMaximumCap` rounds through `Money.of` (:328); `getMathContext` is in `percentageOf` (:297)
  - `LoanChargeService.java:244-271` is `calculateAmountPercentageAppliedTo`; `getDerivedAmountForCharge` is :896-915
  - client `forecloseLoan` is :3052 / :3057; :1377 is deprecated RestAssured
  - a new event type goes in a new part, not in applied 0056; 25 core parts touch the configuration table (not 6); 0136 has no event content
  - an unseeded event class fails startup for every tenant whatever the flag; "tests pass with events off" cannot happen
  - duplicate file names include `LoanProduct.java`
  - `LoanTransactionEnumDataV1`: 25 fields, 22 booleans; 81 `.avsc` under `src/main/avro` (82 with the bigdecimal template)
  - main `.java` files 4,736 under `*/src/main/java` (4,737 only with `misc/Main.java`)
  - fineract-loan + progressive-loan: 21 test classes (23 paths with 2 `.gitkeep`); provider 186 test sources, 146 JUnit classes
  - 35 integration-test files with charge or penalty in the name (32 `*Test.java`), not 34 classes; recalculation IT counts depend on the pattern (33 / 38 / 43), not 39
  - line ranges: `makeRepayment` :216-331; `reprocessLoanTransactions` :90-232; `processLatestTransaction` :235-260; the shared handler :82-185; `makeLoanRepaymentWithChargeRefundChargeType` :1074-1119; `fromParsedJson` :66-224; `addLoanCharge` :182-300; `isChronologicallyLatestRepaymentOrWaiver` :45-51; `handleReAge` :3168-3215; `handleReAgeWithCommonStrategy` :3750-3892; `handleAccelerateMaturityDate` :2047-2164; `allowCompoundingOnEod` :126
  - adding a charge after charge-off is refused (`LoanChargeWritePlatformServiceImpl.java:187-190`; overdue path :800-803)
  - re-age spreads penalties only under the EQUAL_AMORTIZATION types
  - loan charge income GL comes only from the product's per-charge mapping; the `Charge` income account is savings only
  - `fixedLength` is in repayment-frequency units, not days
  - ACCELERATE_MATURITY adds an additional installment for future charges, so the current installment is not always the last
  - post-charge-off penalty GL depends on transaction type: recovery only for REPAYMENT and goodwill credit
  - payoff detection misses only when the compared outstanding includes the penalty and the payment does not
  - charge definitions stay editable while in use; only deactivation and fee frequency / interval are blocked
  - pre-close strategy validated 0-2 (NONE valid)
  - the product matrix: cumulative takes 3 reschedule strategies, progressive only ADJUST_LAST_UNPAID_PERIOD; charge-off behaviours need the advanced strategy
  - recovery repayments are allocated like repayments (advanced :480-482; cumulative :181, :267, :594), only on written-off loans
  - closed GL periods are settled by code (§11.2)
  - `LoanTransactionType`: 43 constants, ids 0-43, id 11 unused
  - component scan `FineractWebApplicationConfiguration.java:52`; MapStruct in 60 main files (not 59)
  - the attempt's "19 new ITs green" holds for an earlier IT file; 2 tests ran on the final file

## 19. Unverified items
- how the build's git versioning behaves in a clone without tags (not checkable: both checkouts have tags)
- vertical allocation (`AdvancedPaymentScheduleTransactionProcessor.processPeriodsVertically` :2917-3058) and the interest-bearing progressive path (:2790-2853): not read in detail
- `RowMapper` SQL bodies of the read services: sampled, not audited
- per-class DB use of the provider unit tests
- product-level rejection of WEEKLY / MONTHLY rest frequency on progressive loans (only the run-time `IllegalStateException` was found, `AdvancedPaymentScheduleTransactionProcessor.java:2900-2901`)
- cumulative prepay template with undisbursed tranches
- progressive foreclosure without recalculation: no integration test or feature found that covers it
- GL entries of REAGE and REAMORTIZE transactions
- effect of the cumulative early-payment folding (`AbstractCumulativeLoanScheduleGenerator.java:679-700`) on a measure built from transaction mappings
- allocation of a manual down payment larger than the down-payment installment (no amount cap found, §13.6; allocation not traced)
- upstream `develop` after the pin: only the gap and the time-type ids were re-checked (at `06ce07b5`, 2026-11-04); nothing else there was read
- compile and test durations
- what bounds the attempt's reported 96 minutes
- attempt, by code reading only: an MIR for the quoted amount leaves the penalty part as excess; accruals drift on replays that do not reach the hook; behaviour of chargeback or replay after a foreclosure
- the NullPointerException at `LoanAccrualsProcessingServiceImpl.java:646` under `charge-accrual-date` = submitted-date (code reading, not run)
- two concurrent repayments on one loan (no locking path read outside COB)


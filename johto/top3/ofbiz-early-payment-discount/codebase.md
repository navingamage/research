# OFBiz: codebase context for an early-payment discount

https://github.com/apache/ofbiz-framework/tree/f05a0c2268f163573a0e0895ec381593de1283db ; Java 17 + Groovy 5.0.5 + XML definitions + legacy minilang, Gradle 8.14.4, embedded Derby by default ; 6344 tracked files (1147 Java, 484 Groovy, 1051 XML, 530 FTL), 31 components

**Marks used below**
- Paths are repo-relative. `:line` is at the pin unless marked HEAD.
- "(by reading)": traced through the code, not executed. "(unverified)": not confirmed.
- [pin-only]: fixed on trunk HEAD `8cfe7b17`. [HEAD too]: still present on trunk HEAD.
- Nothing was built, compiled, tested or run. Counts come from `git ls-files`, `git grep`, `grep`, `wc`.

**Path key for short file names**
- Accounting Groovy `applications/accounting/src/main/groovy/org/apache/ofbiz/accounting/`: `payment/PaymentServices.groovy`, `payment/ListNotAppliedInvoices.groovy`, `payment/FindInvoicesByDueDate.groovy`, `invoice/InvoiceServicesScript.groovy`, `invoice/InvoiceReport.groovy`, `invoice/DisplayInvoiceAmounts.groovy`, `invoice/ListNotAppliedPayments.groovy`, `ledger/GeneralLedgerServicesScript.groovy`, `reports/GlAccountTrialBalance.groovy`, `accounting/*Tests.groovy`
- Accounting Java `applications/accounting/src/main/java/org/apache/ofbiz/accounting/`: `invoice/InvoiceServices.java`, `invoice/InvoiceWorker.java`, `payment/PaymentWorker.java`, `payment/PaymentGatewayServices.java`, `util/UtilAccounting.java`, `agreement/AgreementServices.java`
- Accounting minilang `applications/accounting/minilang/`: `ledger/GeneralLedgerServices.xml`, `ledger/AcctgTransServices.xml`, `permissions/PermissionServices.xml`
- Accounting definitions `applications/accounting/servicedef/` (`services_*.xml`, `secas*.xml`); controller `applications/accounting/webapp/accounting/WEB-INF/controller.xml`
- Accounting widgets `applications/accounting/widget/` (`InvoiceForms.xml`, `PaymentForms.xml`, `PaymentScreens.xml`, `InvoiceScreens.xml`, `AccountingMenus.xml`, `BillingAccountForms.xml`, `AgreementForms.xml`; `accounting/*Cpd.xml`; `ar/`, `ap/`)
- Accounting templates `applications/accounting/template/`: `ar/invoice/ListInvoices.ftl`, `ap/invoices/PurchaseInvoices.ftl`, `ap/invoices/CommissionRun.ftl`, `invoice/CommissionRunSearchResult.ftl`, `invoice/InvoiceReportItems.fo.ftl`
- Data: seed `applications/datamodel/data/seed/` (`AccountingSeedData.xml`, `PartySeedData.xml`, `HumanResSeedData.xml`); demo `applications/datamodel/data/demo/` (`AccountingDemoData.xml`, `GlAccountDemoData.xml`, `OrderDemoData.xml`); entity models `applications/datamodel/entitydef/` (`accounting-entitymodel.xml`, `order-entitymodel.xml`, `party-entitymodel.xml`)
- Order Java `applications/order/src/main/java/org/apache/ofbiz/order/`: `shoppingcart/ShoppingCart.java`, `ShoppingCartHelper.java`, `ShoppingCartServices.java`, `ShoppingCartEvents.java`, `CheckOutHelper.java`; `order/OrderServices.java`, `OrderReadHelper.java`, `OrderReturnServices.java`; `quote/QuoteServices.java`
- Order Groovy `applications/order/src/main/groovy/org/apache/ofbiz/order/`: `quote/QuoteServicesScript.groovy`, `order/OrderServicesScript.groovy`, `entry/OrderTerms.groovy`; order widgets `applications/order/widget/ordermgr/`
- Party `applications/party/src/main/groovy/org/apache/ofbiz/party/party/` (`UnAppliedInvoicesForParty.groovy`, `PartyFinancialHistory.groovy`, `UnAppliedPaymentsForParty.groovy`); `applications/party/widget/partymgr/PartyForms.xml`
- Framework: `framework/service/src/main/java/org/apache/ofbiz/service/` (`ServiceDispatcher.java`, `engine/GroovyEngine.java`, `eca/ServiceEcaAction.java`); `framework/entity/src/main/java/org/apache/ofbiz/entity/` (`transaction/TransactionUtil.java`, `finder/EntityFinderUtil.java`); `framework/base/src/main/java/org/apache/ofbiz/base/util/` (`UtilDateTime.java`, `UtilNumber.java`, `GroovyUtil.java`); `framework/start/src/main/java/org/apache/ofbiz/base/start/Config.java`; `framework/testtools/src/main/java/org/apache/ofbiz/testtools/` (`TestRunContainer.java`, `ModelTestSuite.java`)

## 1. What it is and who runs it

**Product**
- Apache OFBiz: "Enterprise Resource Planning (ERP) System written in Java ... libraries, entities, services and features to run all aspects of your business" (`README.md:36-39`, `docs/asciidoc/user-manual.adoc:32-35`)
- Data model based on "The Data Model Resource Book": parties, orders, invoices, payments, GL, quotes (`docs/asciidoc/user-manual.adoc:101-104`)
- Apache project files tracked: `.asf.yaml`, `LICENSE`. `VERSION` = `Trunk`.
- Who runs it in production: not stated in the repo (unverified)

**Pin**
- Commit `f05a0c22`: author date 2026-05-10 09:57:52 +0200, committer date 2026-05-10 10:14:39 +0200
  - Author Jacopo Cappellato
  - Subject "Fixed: Add permission checks for SERVICE_MAINT in CoreEvents to enhance security"
- Shallow checkout with 1 commit: no blame or log evidence possible
- No `plugins/` dir (optional; `pullAllPluginsSource.sh` clones it)
- No assistant or agent instruction files tracked (`git ls-files`)

**Trunk HEAD used for comparison**
- `8cfe7b17`, 2026-11-04 16:20:11 +1100, 6536 tracked files
- 32 components: adds `framework/rest-api`
- Differences that matter here: §9.1

**Runtime: one JVM**
- Main class `org.apache.ofbiz.base.start.Start` (`build.gradle:119`)
- Containers come from each component's `ofbiz-component.xml`:
  - `naming-container` (RMI) and `admin-container` (`framework/base/ofbiz-component.xml:30,35`)
  - `catalina-container`, embedded Tomcat (`framework/catalina/ofbiz-component.xml:28`): ports 8009 AJP (:72), 8080 HTTP (:119), 8443 HTTPS (:149); test variant `catalina-container-test` (:179)
  - `delegator-container` (`framework/entity/ofbiz-component.xml:35`)
  - `dataload-container`, class `EntityDataLoadContainer` (`framework/entityext/ofbiz-component.xml:41`)
  - `service-container`, `rmi-dispatcher`, `javamail-container` (`framework/service/ofbiz-component.xml:45,50,68`)
  - `testtools-container`, class `TestRunContainer` (`framework/testtools/ofbiz-component.xml:30`)
- Startup commands (`framework/start/src/main/java/org/apache/ofbiz/base/start/StartupCommandUtil.java:62-68`): `--help/-?`, `--load-data/-l`, `--portoffset/-o`, `--shutdown/-d`, `--start/-u`, `--status/-s`, `--test/-t`
- Per-mode properties: `framework/start/src/main/resources/org/apache/ofbiz/base/start/{start,load-data,test,rmi}.properties`

**Database**
- Entity engine over JDBC, configured in `framework/entity/config/entityengine.xml` (845 lines)
- Transactions: Geronimo (:31). Connection pool: DBCP (:50).
- Delegators:
  - `default` (:54-57): `org.apache.ofbiz` uses `localderby`, `org.apache.ofbiz.olap` uses `localderbyolap`, `org.apache.ofbiz.tenant` uses `localderbytenant`
  - `default-no-eca` (:60)
  - `test` (:67-70): the same `localderby`
- 23 datasources: localhsql; localderby (+odbc, olap, tenant); localdaffodil; localaxion; localmysql (+olap, tenant); odbcmysql; localpostgres (+olap, tenant); localoracle; localoracledd; localsybase; localsapdb; localfirebird; localmssql; localp6spy; localadvantage; DB2
- Default `localderby` (:160-192)
  - Embedded Derby `jdbc:derby:ofbiz;create=true` (:181), schema OFBIZ, ReadCommitted, pool 2 to 250
  - check and add-missing on start
  - read-data lists all 7 readers (:171-177)
- Derby `10.16.1.1` runtimeOnly (`dependencies.gradle:100-101`); comment: 10.17 requires Java 21
- Data lives in `runtime/data`; `cleanData` deletes it (`build.gradle:974-976`)
- External DB setup steps: `README.md:795-800`
- Multi-tenancy off by default: `multitenant=N` (`framework/common/config/general.properties:138`)
  - Tenant delegator name `default#<tenantId>`
  - Gradle `createTenant` (`build.gradle:459-544`, dbPlatform D/M/O/P), `loadTenant` (:437-457)
  - All-tenants data load (`EntityDataLoadContainer.java:107`); errors when multitenant is off (:136)
  - Docs `README.md:457-494`

## 2. Repo layout

**Top level**
- `applications/`: business components; `framework/`: engine components; `themes/`: UI themes; `plugins/`: absent at the pin
- `build.gradle` (1143 lines), `common.gradle`, `dependencies.gradle`, `settings.gradle`
- `gradle.properties`: only `org.gradle.console=plain`
- `buildSrc/`: one precompiled plugin, `ofbiz-node-conventions.gradle` (Node and npm for common-theme)
- `gradle/wrapper/gradle-wrapper.properties`; `gradle/init-gradle-wrapper.sh` and `.ps1`; `init-gradle-wrapper.bat` at the root; no wrapper jar tracked
- `config/checkstyle/checkstyle.xml`, `config/codenarc/codenarc.groovy`
- `lib/`: manual jars; only `lib/README` tracked; picked up by fileTree (`build.gradle:262-265`)
- `runtime/`
  - Subdirectories: catalina, data, indexes, logs, output, tempfiles, tmp
  - Tracked files: 7 READMEs, `.gitignore`, `data/derby.properties`, `indexes/index.properties`
- `docker/`, `Dockerfile`, `DOCKER.adoc`; base image `eclipse-temurin:17@sha256:...` (`Dockerfile:21,59`)
- `docs/asciidoc/`: user and developer manuals
- `spotbugs/exclude.xml`: not wired into the build
- `.github/workflows/`: 6 files: gradle.yml, codeql-analysis.yml, docker-image.yml, scorecard.yml, asf-allowlist-check.yml, plus one dependency workflow
- `.pre-commit-config.yaml`; `pullAllPluginsSource.sh/.bat`, `pullPluginSource.sh/.bat`

**Components**
- A component is a folder with `ofbiz-component.xml` (`docs/asciidoc/developer-manual.adoc:67-68`)
- Conventional layout: `config/ data/ entitydef/ minilang/ servicedef/ src/main/{java,groovy} src/test/{java,groovy} testdef/ webapp/ widget/` (`developer-manual.adoc:73-92`); `minilang/` is marked "(deprecated)" (:81)
- 34 tracked `ofbiz-component.xml` files: 31 components (framework 13, applications 12, themes 6), plus the plugin template `framework/resources/templates/ofbiz-component.xml` and 2 test fixtures under `framework/base/src/test/resources/org/apache/ofbiz/base/container/testsdata/`
- Not components: `framework/resources` (createPlugin templates), `framework/start` (bootstrap), `themes/docs`

**Load order**
- Root loader: framework, themes, applications, plugins (`framework/base/config/component-load.xml:23-26`)
- Framework: base, entity, security, datafile, minilang, common, service, catalina, entityext, webapp, widget, testtools, webtools (`framework/component-load.xml:23-35`)
- Applications: datamodel, party, securityext, content, workeffort, product, manufacturing, accounting, humanres, order, marketing, commonext; commonext is last on purpose (`applications/component-load.xml:23-35`, comment :34)
- `themes/` has no loader file; each subdir with `ofbiz-component.xml` loads
- Gradle builds the same list: `activeComponents()` (`common.gradle:39-55`) plus `framework/start`; each becomes a subproject (`settings.gradle`); a missing `plugins/` is skipped (`common.gradle:47-50`)

**Files per component** (`git ls-files` per dir; "all" = every tracked file)

| component | role | java | groovy | xml | ftl | all |
|---|---|---|---|---|---|---|
| framework/base | utils (UtilDateTime, UtilNumber, UtilProperties), config, container loader | 176 | 12 | 9 | 1 | 223 |
| framework/catalina | embedded Tomcat container | 7 | 0 | 3 | 0 | 11 |
| framework/common | shared entities (status, enum, uom), common services (`convertUom`), labels | 38 | 18 | 109 | 1 | 178 |
| framework/datafile | flat-file definitions and parsing | 9 | 0 | 2 | 0 | 13 |
| framework/entity | entity engine: delegator, JDBC, field types, transactions | 150 | 0 | 20 | 0 | 178 |
| framework/entityext | EECA runtime, data load container, entity sync | 17 | 1 | 13 | 0 | 32 |
| framework/minilang | simple-method interpreter and `simple` engine | 125 | 0 | 5 | 0 | 136 |
| framework/resources | plugin templates (not a component) | 0 | 0 | 20 | 0 | 29 |
| framework/security | security and permissions | 12 | 0 | 7 | 0 | 29 |
| framework/service | service engine, engines, SECA, jobs, `OFBizTestCase` | 123 | 5 | 24 | 0 | 159 |
| framework/start | bootstrap (not a component) | 8 | 0 | 0 | 0 | 12 |
| framework/testtools | test runner container, `test-suite.xsd` | 7 | 0 | 3 | 0 | 13 |
| framework/webapp | `ControlServlet`, `RequestHandler`, events | 80 | 1 | 7 | 0 | 97 |
| framework/webtools | admin UI `/webtools` | 21 | 37 | 32 | 41 | 142 |
| framework/widget | screen, form and menu widgets, renderers | 110 | 0 | 4 | 0 | 123 |
| applications/accounting | invoices, payments, GL, fin accounts, gateways, tax, fixed assets, budgets, agreements | 45 | 71 | 141 | 37 | 399 |
| applications/commonext | setup wizard, shared JS webapps; loads last | 0 | 7 | 32 | 0 | 55 |
| applications/content | content and CMS | 53 | 24 | 69 | 28 | 190 |
| applications/datamodel | entity definitions for all apps, shared seed and demo data; no code | 0 | 0 | 36 | 0 | 38 |
| applications/humanres | HR | 1 | 1 | 48 | 4 | 79 |
| applications/manufacturing | BOM, routing, production runs, MRP | 13 | 31 | 43 | 21 | 127 |
| applications/marketing | campaigns, contact lists, SFA | 4 | 10 | 55 | 3 | 81 |
| applications/order | orders, quotes, returns, cart, requirements | 42 | 91 | 87 | 126 | 380 |
| applications/party | parties, roles, contact mechs, agreement entities | 11 | 41 | 53 | 44 | 156 |
| applications/product | catalog, pricing, promos, facility, inventory, shipment | 80 | 122 | 124 | 103 | 444 |
| applications/securityext | login extensions; no webapp | 2 | 0 | 6 | 1 | 10 |
| applications/workeffort | tasks, calendars, timesheets | 12 | 11 | 60 | 11 | 97 |
| themes/common-theme | shared FTL macros, JS libs (npm) | 0 | 0 | 11 | 74 | 1436 |
| themes/helveticus | theme | 0 | 0 | 6 | 13 | 594 |
| themes/rainbowstone | theme | 0 | 0 | 4 | 10 | 580 |
| themes/tomahawk | theme | 0 | 0 | 4 | 4 | 102 |
| themes/bluelight | theme | 0 | 0 | 4 | 5 | 65 |
| themes/flatgrey | theme | 0 | 0 | 4 | 3 | 46 |
| themes/docs | not a component | 0 | 0 | 0 | 0 | 1 |

- Unit-test sources (`src/test/`) exist only in framework at the pin: 74 files (base 42, widget 14, webapp 4, entity 3, security 3, webtools 3, common 2, service 2, start 1)
- Integration tests are Groovy or Java classes under `src/main/`, run through `testdef/*.xml` (trunk moved accounting tests to `src/test/groovy`, §9.1)

## 3. The entity engine

**Field types** (`framework/entity/fieldtype/fieldtypederby.xml`; 12 fieldtype files)
- `date-time` TIMESTAMP (:29), `date` DATE (:30)
- `currency-amount` NUMERIC(18,2) (:33), `currency-precise` NUMERIC(18,3) (:34), `fixed-point` NUMERIC(18,6) (:35)
- `floating-point` DOUBLE (:36), `numeric` NUMERIC(20,0) read as Long (:37)
- `id` VARCHAR(20) (:40), `indicator` CHAR(1) (:44)
- Postgres `date-time` is TIMESTAMPTZ (`fieldtypepostgres.xml:29`); numeric types the same

**Entity model files** (`applications/datamodel/entitydef/`; counts exclude comments)

| file | entity | view-entity | extend-entity |
|---|---|---|---|
| accounting (4322 lines) | 160 | 37 | 0 |
| product | 169 | 39 | 1 |
| order | 112 | 2 | 0 |
| party | 84 | 41 | 8 |
| content | 64 | 12 | 2 |
| humanres | 41 | 5 | 0 |
| workeffort | 40 | 0 | 0 |
| shipment | 39 | 12 | 0 |
| marketing | 32 | 10 | 0 |
| manufacturing | 7 | 1 | 0 |

- Also `applications/accounting/entitydef/entitymodel_reports.xml`: 127 lines, 3 view-entities
- `framework/common` entity model: 46 entities, 11 views

**Registration**
- `applications/datamodel/ofbiz-component.xml`: model files :26-35; seed-initial :38-39; seed :42-52 (`AccountingSeedData.xml` :48); demo :55-66 (`GlAccountDemoData.xml` :61, `AccountingDemoData.xml` :62, `OrderDemoData.xml` :64)
- `applications/accounting/ofbiz-component.xml`: model `entitymodel_reports.xml` (:27), eca `eecas.xml` (:28), data :29-33 (seed :29-31, seed-initial :32, demo :33), service models :36-66, groups :68, SECAs :69-72, test suites :74-79, webapps :81-104

**View entities**
- Members, aliases and view-links in XML
- `PaymentAndApplication` (`accounting-entitymodel.xml:3084-3101`): Payment joined to PaymentApplication on paymentId, inner join (no rel-optional, :3094-3096); `Payment.overrideGlAccountId` excluded (:3091)
- `InvoiceAndApplAndPayment` (:1628; payment fields prefixed `pm`)

**EECAs (entity triggers)**
- `applications/accounting/entitydef/eecas.xml`: FinAccountTrans runs `updateFinAccountBalancesFromTrans` (:25-28); FinAccountAuth runs `updateFinAccountBalancesFromAuth` (:30-32)
- None on Invoice, Payment, PaymentApplication or InvoiceTerm; accounting logic hangs off SECAs (§4)
- Other components with eecas.xml: content, order (OrderHeader, OrderItem, OrderPaymentPreference), product, workeffort

**Delegator API** (`framework/entity/src/main/java/org/apache/ofbiz/entity/`)
- `Delegator`, `GenericDelegator`, `GenericValue`, `GenericPK`, `condition/`, `transaction/`, `util/EntityQuery.java` (597 lines)
- `EntityQuery` builder: `use` :79, `select` :97/109, `from` :118/128, `where` :139-179, `orderBy` :201/211, `maxRows` :253, `distinct` :297, `cache` :314/323, `filterByDate` :331-381, `queryList` :395, `queryIterator` :405, `queryFirst` :425, `queryOne` :438, `queryCount` :448, `getFieldList` :556, `queryPagedList` :591
- Writes: `delegator.makeValue(...)` then `.create()`, `.store()`, `.remove()`; ids from `delegator.getNextSeqId("Entity")`
- Transactions: `TransactionUtil.rollback(began, ...)` rolls back when this code began the transaction, otherwise only sets rollback-only (`.../entity/transaction/TransactionUtil.java:302-308`)
- Minilang entity conditions: a null field value compares as IS NULL (`framework/entity/src/main/java/org/apache/ofbiz/entity/finder/EntityFinderUtil.java:207-208`; `ignore-if-null` default false, `framework/minilang/dtd/simple-methods.xsd:3141-3146`; condition built through ByConditionFinder, `framework/minilang/src/main/java/org/apache/ofbiz/minilang/method/entityops/EntityCondition.java:24,53`)

**Data readers and loading**
- Readers: tenant, seed, seed-initial, demo, ext, ext-test, ext-demo (`entityengine.xml:92-98`)
- Load args `readers=`, `file=`, `component=` (`framework/entityext/src/main/java/org/apache/ofbiz/entityext/data/EntityDataLoadContainer.java:70-73`); no readers and no file loads every reader the datasource lists (:482-484, :504-507)
- CLI forms: `README.md:376-455`
- Accounting data files:
  - `applications/datamodel/data/seed/AccountingSeedData.xml` (1313 lines): types, statuses, valid changes, GL account types
  - `applications/datamodel/data/demo/AccountingDemoData.xml`: Company preferences, GL defaults and mappings, fiscal periods
  - `applications/datamodel/data/demo/GlAccountDemoData.xml`: chart of accounts
  - `applications/accounting/data/`: AccountingSecurityPermissionSeedData, AccountingSystemPropertyData, AccountingPortletData, AccountingScheduledServiceData (seed-initial), ScheduledServicesDemoData (demo), OfbizSetupChartOfAccountsData (419 lines; not listed in `ofbiz-component.xml:29-33`; no tracked file references it; how it loads is unverified)
  - `applications/commonext/data/OfbizSetupGlAccountData.xml`: imported by the setup wizard (`applications/commonext/minilang/setup/SetupEvents.xml:319`, entityImport)

## 4. The service engine and SECAs/EECAs

**Definitions**
- `servicedef/services*.xml`, registered as `<service-resource type="model">` in `ofbiz-component.xml`
  - Accounting: 27 model files (`applications/accounting/ofbiz-component.xml:36-66`), groups `type="group"` (:68), SECAs `type="eca"` (:69-72)
- `services.xsd` defaults (`framework/service/dtd/services.xsd`): `engine` required (:63), `auth` false (:66), `validate` true (:68), `use-transaction` true (:84), `require-new-transaction` false (:92), `transaction-timeout` 0 (:111), `max-retry` 0 (:121), `semaphore` none (:123), wait 300 s (:132), attribute `optional` false (:320)

**Engines**
- Registered in `framework/service/config/serviceengine.xml:44-57`: entity-auto, group, interface, java, clojure, simple (minilang), script, groovy, javascript, route, jms, rmi
- `engine="http"`: 6 services at `framework/entityext/servicedef/services.xml:104, 108, 112, 116, 299, 311`, not in that list (by reading); a seventh, testHttp, is commented out at `framework/common/servicedef/services_test.xml:81`
- Service definitions by engine, all servicedefs: 3581 total; entity-auto 1874, java 714, groovy 560, simple 310, interface 61, group 33, jms 7, rmi 6, http 6, script 4, route 3, clojure 2, javascript 1
- Accounting: 810 (entity-auto 395, java 196, simple 98, groovy 88, interface 19, group 10, rmi 4)
  - `services_invoice.xml` 86: groovy 25, entity-auto 44, java 13, group 4
  - `services_payment.xml` 56: groovy 25, entity-auto 28, java 3
  - `services_ledger.xml` 132: entity-auto 79, simple 46, groovy 3, java 2, interface 2

**Minilang still in use**
- 89 minilang XML files outside `framework/minilang`: order 17, product 16, accounting 10, party 9, manufacturing 8, content 7, marketing 6, workeffort 6, framework/common 3, humanres 3, securityext 2, commonext 1, framework/entity 1
- Accounting minilang, 5164 lines:
  - `minilang/ledger/GeneralLedgerServices.xml` 2884, `ledger/AcctgTransServices.xml` 472
  - `finaccount/FinAccountServices.xml` 1089, `finaccount/FinAccountGlPostServices.xml` 136
  - `payment/PaymentMethodServices.xml` 132, `PaymentMapProcs.xml` 132, `PaymentMethodEvents.xml` 118, `PaymentGatewayConfigServices.xml` 59
  - `permissions/PermissionServices.xml` 117, `test/PaymentApplicationTests.xml` 25
- All GL posting logic is minilang (`GeneralLedgerServices.xml`); still minilang at HEAD
- Migration guide: `framework/minilang/src/docs/asciidoc/minilang-to-groovy-manual.adoc`

**Permissions**
- Per service: `<permission-service service-name="acctgInvoicePermissionCheck" main-action="CREATE"/>` (e.g. `services_invoice.xml:53`)
- Checks in `applications/accounting/servicedef/services_perm.xml`: 13 simple services
  - `acctgInvoicePermissionCheck` :68 and `acctgPaymentPermissionCheck` :97 call `basePlusRolePermissionCheck` (`minilang/permissions/PermissionServices.xml:98-100`, :114-116)
  - `acctgTransactionPermissionCheck` :80 calls `genericBasePermissionCheck` with ACCTG_ATX (`PermissionServices.xml:103-106`)
- Business rule as permission service: `checkInvoiceStatusInProgress` (`services_invoice.xml:106-112`; `InvoiceServicesScript.groovy:299-303`) grants only while the invoice is INVOICE_IN_PROCESS
- Webapp gate: `base-permission="OFBTOOLS,ACCOUNTING"` (`applications/accounting/ofbiz-component.xml:86,94,102`)
- Gaps at the pin on the apply services:
  - `createPaymentApplication` (`services_payment.xml:69-80`): auth true, no permission-service, no permission check in code [pin-only: HEAD adds ACCOUNTING _CREATE in code, HEAD `PaymentServices.groovy:727`]
  - `updatePaymentApplication` / `updatePaymentApplicationDef` (`services_payment.xml:81-102`): no permission-service, auth default false [pin-only: HEAD adds ACCOUNTING _UPDATE in code, HEAD `InvoiceServices.java:2692-2695`]

**`ServiceDispatcher.runSync`, in order** (`framework/service/src/main/java/org/apache/ofbiz/service/ServiceDispatcher.java`, 1118 lines)
1. Default IN values (:317)
2. Suspend for require-new (:322) or begin a transaction (:329, :332)
3. try (:345); register global-rollback and global-commit ECAs (:364, :367)
4. ECA `auth` (:372); auth check (:379)
5. ECA `in-validate` (:396); IN validation: `makeValidContext`, then validate; throws `ServiceValidationException` (:404-415)
6. ECA `invoke` (:419); engine call (:429)
7. ECA `out-validate` (:519); OUT validation (:516-529)
8. ECA `commit` (:531-534) and `global-commit-post-run` (:540-543): both run before the transaction commits
9. catch Throwable (:549): rollback (:556)
10. finally: error result rolls back (:578); otherwise commit (:585)
11. Resume the parent transaction (:617); ECA `return` (:628), after the commit
- A nested call that did not begin the transaction only marks it rollback-only (`TransactionUtil.java:302-308`)

**SECAs**
- Form: `<eca service=... event=...><condition .../><action service=... mode="sync"/></eca>`
- Defaults (`framework/service/dtd/service-eca.xsd`): `run-on-failure` and `run-on-error` false (:133-134), `mode` required (:275), `new-transaction` false (:291), `result-to-context` true (:296), `result-to-result` false (:301), `ignore-failure` true (:306), `ignore-error` true (:311), `persist` false (:316)
- Runtime (`framework/service/src/main/java/org/apache/ofbiz/service/eca/ServiceEcaAction.java`): a sync action without new-transaction calls `dispatcher.runSync(name, ctx)` and joins the caller's transaction (:154-158); ignore-failure and ignore-error handling (:192-211)
- Consequence (by reading): a sync action that returns an error marks the shared transaction rollback-only, even with ignore-error true; the caller's work rolls back. Intent stated in the comment at `secas_payment.xml:24-28`.
- Accounting SECA resource order: `secas.xml` (:69), `secas_payment.xml` (:70), `secas_ledger.xml` (:71), `secas_invoice.xml` (:72) of `applications/accounting/ofbiz-component.xml`; run order for two SECAs on the same event in different files (unverified)

**Accounting SECAs on the settlement path**
- Scoped count: 23 active SECAs on 8 services (createPayment, setPaymentStatus, createPaymentApplication, removePaymentApplication, voidPayment, setInvoiceStatus, cancelInvoice, createAcctgTransAndEntries); other services have more (e.g. `postAcctgTrans` commit, `secas_ledger.xml:156`, runs `checkUpdateFixedAssetDepreciation`)
- None on `updatePaymentApplication`, `updatePaymentApplicationDef`, `createPaymentAndApplication`; the createPayment and createPaymentApplication SECAs still fire inside createPaymentAndApplication, which calls both
- `secas.xml:29-32`: createPaymentApplication, commit, invoiceId not empty: `checkInvoicePaymentApplications`
- `secas_payment.xml`
  - :31-35, :38-42: setPaymentStatus to PMNT_RECEIVED / PMNT_SENT, commit: `checkPaymentInvoices`
  - :45-48: createPaymentAndApplicationForParty, commit: `createFinAccoutnTransFromPayment`
  - :60-62: removePaymentApplication, **invoke**, no condition: `revertAcctgTransOnRemovePaymentApplications`
  - :65-69: voidPayment, commit: `setFinAccountTransStatus`
  - :71-75, :77-81: setPaymentStatus to RECEIVED / SENT: `createMatchingPaymentApplication`
- `secas_invoice.xml`
  - :25-27: cancelInvoice, **invoke**: `revertAcctgTransOnCancelInvoice`
  - :30-37: commission association cleanup, order billing reset
  - :39-43: to INVOICE_APPROVED, commit: `createMatchingPaymentApplication`
  - :44-48: IN_PROCESS to READY, **return** event: `createMatchingPaymentApplication`
  - :50-57: to READY, not from READY or PAID: `checkInvoicePaymentApplications`, `capturePaymentsByInvoice`
- `secas_ledger.xml`
  - :25-28: createAcctgTransAndEntries, commit: `postAcctgTrans`
  - :91-99: to READY (not CUST_RTN, not from READY or PAID): `createAcctgTransForPurchaseInvoice`, `createAcctgTransForSalesInvoice`
  - :100-107: CUST_RTN_INVOICE: customer return GL
  - :108-112: to INVOICE_CANCELLED: `cancelInvoice`
  - :115-124: createPayment as RECEIVED; setPaymentStatus to RECEIVED not from RECEIVED or CONFIRMED: `createAcctgTransAndEntriesForIncomingPayment`
  - :127-136: same for SENT: outgoing payment GL
  - :138-147: createPaymentApplication, commit, invoiceId set: `createAcctgTransAndEntriesForPaymentApplication` (CUSTOMER_REFUND variant :143-147)
- Order component: `applications/order/servicedef/secas.xml:101-114` runs createPaymentFromOrder on PO approved and SO completed (persist true); `:462-466` createPaymentFromPreference runs createOrderPaymentApplication
- Commented sample, not active: `applications/accounting/servicedef/secas.xml:142-152`

**SECA chain for `createPaymentApplication` with an invoiceId** (by reading)
1. `createPaymentApplication` (`PaymentServices.groovy:737-794`) creates the PaymentApplication row
2. commit ECA: `checkInvoicePaymentApplications` (`secas.xml:29-32`; `InvoiceServices.java:2390-2491`)
3. If the invoice is READY and kept applications cover the total: `setInvoiceStatus` INVOICE_PAID with paidDate (:2468-2477)
4. commit ECA: `createAcctgTransAndEntriesForPaymentApplication` (`secas_ledger.xml:138-147`; `GeneralLedgerServices.xml:2530-2652`)
5. That calls `createAcctgTransAndEntries`; its commit ECA runs `postAcctgTrans` (`secas_ledger.xml:25-28`)
6. All of it runs in the caller's transaction; an error in step 3, or a posting error in step 5 when the org has no errorGlJournalId, marks it rollback-only

**Transactions in accounting**
- `require-new-transaction="true"`: 40 repo-wide; e.g. `services_finaccount.xml:392,409`, `services_paymentmethod.xml:363,474` (474 also max-retry 5)
- `use-transaction="false"`: 50 repo-wide; e.g. `services_fixedasset.xml:149`, `services_paymentmethod.xml:211`
- `transaction-timeout`: `postAcctgTrans` 600 (`services_ledger.xml:280`), `importZipSalesTaxData` 7200 (`services_tax.xml:198`)
- Comments "never set require-new-transaction" at `services_finaccount.xml:116,167`

**Groovy services**
- Definition: `engine="groovy" location="component://accounting/src/main/groovy/org/apache/ofbiz/accounting/invoice/InvoiceServicesScript.groovy" invoke="createInvoice"` (`services_invoice.xml:50-63`)
- Implementation: a top-level `Map createInvoice() {...}` in that script; one file holds many services
- Script base class `GroovyBaseScript` (`framework/base/config/groovy.properties:21`, applied in `GroovyUtil.java:58-61`); parsed from `component://` and cached (`GroovyUtil.java:180-205`)
- `GroovyEngine.runSync` (`framework/service/src/main/java/org/apache/ofbiz/service/engine/GroovyEngine.java:79-140`)
  - Binds the whole service context as variables (`userLogin`, `locale`, `timeZone`), plus `parameters`, `dctx`, `security`, `dispatcher`, `delegator` (:84-95)
  - Calls the `invoke` method (:108-110); a returned Map is the result (:115-117)
  - Catches Exception (:122-138): `ExecutionServiceException` becomes `returnError`; anything else is rethrown as `GenericServiceException`

**Groovy DSL** (`framework/service/src/main/groovy/org/apache/ofbiz/service/engine/GroovyBaseScript.groovy`, 177 lines at the pin)
- `run service: 'x', with: [...]` (:61-63) calls `runService` (:41-59): adds userLogin, locale, timeZone; `makeValidContext` drops unknown keys (:53); `runSync` (:54); throws `ExecutionServiceException` on an error result (:55-57)
- `makeValue('Entity', map)` (:65-71), `from('Entity')` returns an EntityQuery (:73-79), `select(...)` (:81-87), `findOne` deprecated (:89-92)
- `success([k: v])` / `success(msg, map)` (:95-118), `failure(msg)` (:120-129), `error(msg)` (:131-144)
- `logInfo/logWarning/logError/logVerbose` (:147-164); `label('AccountingUiLabels', 'Key'[, ctx])` (:166-175)
- Condition DSL (`EntityConditionBuilder`), e.g. `PaymentServices.groovy:325-338, 866-882`, `InvoiceReport.groovy:23-35`
- HEAD adds `require(condition, message)` that throws `ServiceErrorException` (HEAD `GroovyBaseScript.groovy:216, 221`); trunk code uses it widely

## 5. The web layer (brief)

- `ControlServlet` and `RequestHandler` (`framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/`) read `WEB-INF/controller.xml`
  - `request-map`: security, `<event type="service" invoke="..."/>`, responses
  - `view-map` of type `screen` points to widget XML
- Accounting webapps `/accounting`, `/ar`, `/ap` (`applications/accounting/ofbiz-component.xml:81-104`)
- `webapp/accounting/WEB-INF/controller.xml`: 519 request-maps, 247 view-maps; includes common, commonext and `accounting-controller.xml` (:23-25); `/ar` has 16 request-maps, `/ap` 22
- Application requests that matter here:
  - `updateInvoiceApplication` :222-227 and `updatePaymentApplication` :384-388: both call `updatePaymentApplicationDef`
  - `removeInvoiceApplication` :228-232, `removePaymentApplication` :389-393
  - `createPaymentApplication` :379-383
  - `setInvoiceStatus` :273-278, `createInvoiceTerm` :285-290
- Widgets: `applications/accounting/widget/`, 71 files; 11 `*Cpd.xml` compound widgets (31 repo-wide) with `sc:/wf:/wm:/ws:` namespaces
  - `AccountingInvoiceApplicationCpd.xml`: create :36-41 and update :43-48 call `updatePaymentApplicationDef`; delete :50-55 calls `removePaymentApplication`
  - Payment-side forms call `createPaymentApplication` (`widget/PaymentForms.xml:307,329,346`)
- FTL: 37 templates in `applications/accounting/template/`, rendered by `framework/widget/src/main/java/org/apache/ofbiz/widget/renderer/macro/MacroScreenRenderer.java`
- Themes under `themes/`; common-theme JS via npm (`themes/common-theme/webapp/common-theme/js/package.json`)
- Other webapp mounts: order `/ordermgr`; party `/partymgr`; product `/catalog /facility`; manufacturing; marketing `/marketing /sfa`; workeffort `/workeffort /iCalendar`; content `/content /contentimages/`; humanres; commonext `/ofbizsetup /common-js /ordermgr-js`; webtools `/webtools`; themes `/bluelight /common /images /flatgrey /helveticus /rainbowstone /tomahawk`; none in datamodel or securityext

## 6. Accounting in depth

### 6.1 Layout (`applications/accounting/`)
- Tracked files per dir: config 8, data 6, entitydef 2, minilang 10, servicedef 32, src 212, template 37, testdef 9, webapp 10, widget 71; plus `README.md`; `src/docs` holds 96 files
- Java (`src/main/java/org/apache/ofbiz/accounting/`)
  - `invoice/InvoiceServices.java` 3863 lines, `invoice/InvoiceWorker.java` 842
  - `payment/`: PaymentWorker (374), PaymentGatewayServices, PaymentMethodServices, BillingAccountWorker, GiftCertificateServices
  - `util/UtilAccounting.java` (332)
  - Also ledger, finaccount, period, tax, agreement, thirdparty/*
- Excluded from compilation: IcsPayment, OrbitalPayment, PayPal, SecurePay*, PayflowPro, Taxware* (`build.gradle:268-278`)
- Groovy (`src/main/groovy/org/apache/ofbiz/accounting/`)
  - `invoice/InvoiceServicesScript.groovy` 664 lines, `invoice/InvoiceReport.groovy`
  - `payment/PaymentServices.groovy` 1026 lines, `payment/FindInvoicesByDueDate.groovy`
  - `ledger/GeneralLedgerServicesScript.groovy`, `reports/` (13 files)
  - `accounting/`: the 18 integration test classes
- Servicedef: 27 `services_*.xml`, `groups.xml`, `secas.xml`, `secas_invoice.xml`, `secas_ledger.xml`, `secas_payment.xml`
- Receipt vs disbursement: `UtilAccounting.isReceipt` / `isDisbursement` by PaymentType parent (`util/UtilAccounting.java:162,158`)
- Sales vs purchase invoice: `isSalesInvoice` / `isPurchaseInvoice` by InvoiceType parent (:276, :272)

### 6.2 Invoices (`applications/datamodel/entitydef/accounting-entitymodel.xml`)
- `Invoice` :1159-1213
  - invoiceTypeId, partyIdFrom, partyId, roleTypeId, statusId, billingAccountId, contactMechId
  - invoiceDate, dueDate, paidDate; invoiceMessage, referenceNumber, description, currencyUomId, recurrenceInfoId
  - **No stored total, applied or outstanding**
- `InvoiceItem` :1284: invoiceItemTypeId, overrideGlAccountId, overrideOrgPartyId, productId, parentInvoiceId/SeqId, taxableFlag, taxAuthPartyId/GeoId, taxAuthorityRateSeqId; `quantity` fixed-point; `amount` currency-precise (3 dp)
- `InvoiceItemType` :1428 (parentTypeId, defaultGlAccountId); `InvoiceItemTypeGlAccount` :1462 (per org); `InvoiceItemTypeMap` :1480
- `InvoiceRole` :1498; `InvoiceStatus` :1523 (history: statusDate, changeByUserLoginId); `InvoiceType` :1583; `InvoiceNote` :1613
- `InvoiceTerm` :1543-1566
  - PK invoiceTermId; termTypeId, invoiceId, invoiceItemSeqId (one-nofk to InvoiceItem :1562-1565)
  - `termValue` currency-amount, 2 dp (:1550); `termDays` numeric, Long (:1551); textValue, description, `uomId` (:1554, no Uom relation)
  - `InvoiceTermAttribute` :1567
- Views: `InvoiceAndType` :1645, `InvoiceAndRole` :1668, `InvItemAndOrdItem` :1699, `InvoiceAndApplAndPayment` :1628
- Invoice types (`AccountingSeedData.xml:546-555, 649`)
  - Under SALES_INVOICE: PURC_RTN_INVOICE (:550), INTEREST_INVOICE (:552)
  - Under PURCHASE_INVOICE: CUST_RTN_INVOICE (:549), COMMISSION_INVOICE (:551), PAYROL_INVOICE (:649)
- Discount-like item types (seed): INV_DISCOUNT_ADJ :419, ITM_DISCOUNT_ADJ :431, PINV_DISCOUNT_ADJ :453, PITM_DISCOUNT_ADJ :464; CRT :502 and SRT :529 variants; order adjustment `DISCOUNT_ADJUSTMENT` maps to them (`InvoiceItemTypeMap` rows :569, :593)
- `createInvoiceForOrder` (`InvoiceServices.java:152`)
  - Item amount decimals by invoice type, read only here (:191-195)
  - invoiceDate = eventDate or now (:228-232)
  - dueDate = `getDayEnd(invoiceDate, orh.getOrderTermNetDays())`, only when net days exist (:233-238)
  - Creates the invoice IN_PROCESS in the order currency when invoiceId is empty (:241-263); roles (:265-278; createInvoiceRole errors return error)
  - Copies **all** OrderTerms (:280-283; TODO at :281 about financial terms only); billing-account terms commented out (:285-297)
  - Applies earlier order payments with remaining amounts through `createPaymentApplication` while still IN_PROCESS (:892-920, runSync :914)
  - `autoApproveInvoice`: sales to READY, purchase to IN_PROCESS (:922-933)
- Commission invoices: dueDate `getDayEnd(now, days)` (:1084-1086)
- `createInvoicesFromShipments` reuses an invoiceId from ShipmentItemBilling (:1906-1923)
- `copyInvoice` does not copy terms (`InvoiceServicesScript.groovy:200-215`)
- `createInvoice` takes the party's `preferredCurrencyUomId` (`InvoiceServicesScript.groovy:144-147`)

### 6.3 Payments
- `Payment` (`accounting-entitymodel.xml:2931-2952`)
  - paymentTypeId, paymentMethodTypeId, paymentMethodId, paymentPreferenceId, partyIdFrom, partyIdTo, roleTypeIdTo, statusId, effectiveDate, paymentRefNum
  - `amount` currency-amount, currencyUomId, finAccountTransId, overrideGlAccountId
  - `actualCurrencyAmount` / `actualCurrencyUomId`: comment "not for PaymentApplication" (:2951)
- `PaymentType` :3419 (seed `AccountingSeedData.xml:714-731`: RECEIPT children CUSTOMER_PAYMENT, CUSTOMER_DEPOSIT, INTEREST_RECEIPT, GC_DEPOSIT; DISBURSEMENT children include CUSTOMER_REFUND, VENDOR_PAYMENT, VENDOR_PREPAY)
- `PaymentGlAccountTypeMap` :3449-3466 (paymentTypeId + org gives glAccountTypeId); `PaymentMethodTypeGlAccount` :3401; views `PaymentAndType` :3052, `PaymentAndApplication` :3084
- No payment status history entity (only `InvoiceStatus` exists)
- `createPayment` (`PaymentServices.groovy`): permission :38-42; status default PMNT_NOT_PAID (:46); paymentMethodTypeId required (:64-66); effectiveDate default now (:69)
  - :60 `parameters.paymentId = parameters.paymentId ?: orderPaymentPreference.paymentMethodId` [pin-only: HEAD sets paymentMethodId]
- `updatePayment` (:133-177): every field editable while PMNT_NOT_PAID, effectiveDate included; after that only status, comments, paymentRefNum, finAccountTransId (:140-154); no bound check on effectiveDate [HEAD too]
- `setPaymentStatus` (:796-846)
  - StatusValidChange check (:808-811)
  - RECEIVED or SENT without paymentMethodId returns **failure** (:814-816) [pin-only: HEAD uses require, an error]
  - CONFIRMED requires fully applied, failure (:819-822) [pin-only: HEAD error]
  - PMNT_CANCELLED removes each application (:826-833)
- `createPaymentFromOrder` (order SECAs above; gated by `accounting.properties:38` PO and `:41` SO not-paid payment autocreate)
  - Gates :612-629; an existing payment returns failure (:636) [pin-only: HEAD logs and returns success]
  - With a sales agreement: effectiveDate = now + OrderTerm FIN_PAYMENT_TERM termDays (:670-678), else now (:679): future-dated payments
  - amount = order grandTotal, converted to base currency when needed; actualCurrencyAmount/UomId set (:686-711)
  - paymentMethodTypeId COMPANY_ACCOUNT, PMNT_NOT_PAID (:716-717)
- `createPaymentAndApplication` (:414-429; def `services_payment.xml:111-127`): `amountApplied: parameters.amount` (:420-422); `overrideGlAccountId` IN (:119) lands on the Payment
- `createPaymentAndApplicationForParty`, the AP check run (:179-235; def :176)
  - VENDOR_PAYMENT, PMNT_SENT (:199-207)
  - Amount = sum of `getInvoicePaymentInfoList[0].outstandingAmount` (:189, :219-223)
  - `.each` closures at :183 and :213 swallow `return error(...)` (:191) and `return serviceResult` [pin-only: HEAD uses for loops, HEAD :178, :208]
- `voidPayment` (:362-397; def `services_payment.xml:145-150`): see §6.7
- Exist with no settlement logic:
  - `Deduction` :2801 (deductionId, deductionTypeId, paymentId, amount), `DeductionType` :2816; CRUD only (`services_payment.xml:286-310`); seeded types are payroll ones: FED_TAX, STATE_TAX, INSURANCE, CAFE_PLAN (`applications/datamodel/data/seed/HumanResSeedData.xml:114-117`)
  - `SettlementTerm` :2581 (termName, termValue numeric, uomId); seed COD, NET_15, NET_30, NET_60, NET_90 (`AccountingSeedData.xml:954-958`); referenced by `AcctgTransEntry.settlementTermId`; other use unverified

### 6.4 Payment applications
- `PaymentApplication` (`accounting-entitymodel.xml:3119-3154`)
  - paymentApplicationId, paymentId, invoiceId (+ invoiceItemSeqId), billingAccountId, toPaymentId, taxAuthGeoId
  - overrideGlAccountId (description "applied directly against this GL account", :3127)
  - `amountApplied` currency-amount, 2 dp
  - **No business date** (only automatic created/lastUpdated stamps)
- **Path 1: `createPaymentApplication`** (groovy; `services_payment.xml:69-80`; `PaymentServices.groovy:737-794`)
  - paymentId required (:72); IN has toPaymentId, invoiceId, billingAccountId, taxAuthGeoId, amountApplied (INOUT); no overrideGlAccountId or invoiceItemSeqId, so those are dropped by makeValidContext
  - Requires at least one of invoiceId, billingAccountId, taxAuthGeoId, toPaymentId (:739-741)
  - `makeValue('PaymentApplication', parameters)` (:742)
  - Currency: invoice currency must equal payment currencyUomId or actualCurrencyUomId (:755-757); actual-currency amounts in the foreign case (:758-761)
  - :763-765 `paymentAppl.amountApplied = notAppliedInvoice <= notAppliedPayment ? notAppliedInvoice : notAppliedPayment`: the caller's amount is ignored for invoices
  - Copies invoice.billingAccountId (:767-769); toPaymentId amount (:776); billing and tax branches (:784-787); returns paymentTypeId (:791-793)
  - **No invoice-status or payment-status check**
  - Fires the SECA chain (§4)
  - Callers (9 service call sites): `InvoiceServices.java:914` (createInvoiceForOrder); `PaymentGatewayServices.java:2368` (processCaptureResult); `PaymentServices.groovy:420` (createPaymentAndApplication), :221 (check run), :956 (matching); `InvoiceServicesScript.groovy:287` (payroll); `applications/order/src/main/groovy/org/apache/ofbiz/order/order/OrderServicesScript.groovy:899`; `OrderReturnServices.java:1622, 1708`
  - UI: 3 forms (`widget/PaymentForms.xml:307,329,346`) via `controller.xml:379-383`
- **Path 2: `updatePaymentApplication` / `updatePaymentApplicationDef`** (java; `services_payment.xml:81-91`, :92-102)
  - `InvoiceServices.java:2656-2668` (useHighestAmount N) and :2678-2683 (Y); body `updatePaymentApplicationDefBd` :2685-3478
  - Validates:
    - `invoice.decimals` / rounding configured (:2689-2692)
    - Exactly one target for a new row; invoice plus billingAccount allowed together (:2745-2769)
    - Payment not PMNT_CANCELLED or PMNT_CONFIRMED (:2802-2809)
    - Invoice not CANCELLED (:2936-2939); currency and actual currency (:2941-2961)
    - Invoice total != 0; not completely applied; amount <= invoice open (:2963-2984)
    - Party match: payment.partyIdFrom == invoice.partyId or payment.partyIdTo == invoice.partyIdFrom (:2986-2991)
    - Payment availability (:3048-3062)
  - `invoiceProcessing=YY` (`accounting.properties:29`, read :2698) picks invoice-level vs item-level application (:2734-2743)
  - Writes through `storePaymentApplication` (:3340, :3363, :3453, :3477)
  - **No dispatcher call between :2685 and :3630, so no SECA fires**: no PAID check and no PAYMENT_APPL posting at that moment (by reading)
  - Those catch up only on later events: payment to RECEIVED/SENT (`checkPaymentInvoices`; incoming/outgoing payment GL posts existing applications), invoice to READY (gate A; purchase invoice GL re-posts applications)
  - UI: invoice-side screens (`controller.xml:222-227, 384-388`; `InvoiceForms.xml:209,537,559,569`; `AccountingInvoiceApplicationCpd.xml:36-48`)
- **`storePaymentApplication`** (`InvoiceServices.java:3511-3626`)
  - Looks for an existing row with the same invoiceId, invoiceItemSeqId, billingAccountId, paymentId, toPaymentId, taxAuthGeoId (:3527-3532) and adds the amount to it; store :3554, :3566, :3590; remove :3580; create :3607; store :3619
  - Direct `create()/store()/remove()`, no service calls
  - `overrideGlAccountId`: accepted by both update services (auto nonpk, `services_payment.xml:88, 99`) but never set or used here
- **Other direct writes**
  - `PaymentGatewayServices.java:1585-1615`: billing-account applications re-pointed or split to the invoice by direct store/create (:1596-1613; TODO on status :1595)
  - `OrderReturnServices.java:995-1002`: billing-account application created directly, no invoiceId
- **Auto-matching** `createMatchingPaymentApplication` (`PaymentServices.groovy:848-966`)
  - Gate `accounting.payment.application.autocreate=Y` (`accounting.properties:35`, read :849 via EntityUtilProperties)
  - Invoice side: payments not CONFIRMED, payment amount == invoice total exactly, payment with 0 applications (:872-891)
  - Payment side: invoices NOT_IN READY/PAID/CANCELLED/WRITEOFF (:905), hasParentType (:917-920), exact amount (:930-937); creates only when the invoice already has applications (`.queryCount()` :943-945) [pin-only: inverted, HEAD :924 `== 0`]
  - Calls `isInvoiceInForeignCurrency` (:861, :925), which always returns false at the pin (§6.11) [pin-only]
- **`removePaymentApplication`** (`PaymentServices.groovy:972-1026`; def `services_payment.xml:103-109`, acctgInvoicePermissionCheck UPDATE)
  - Rejects a CONFIRMED payment (:983-985)
  - :992-994 `if (invoice.statusId == 'INVOICE_PAID') { run service: 'setInvoiceStatus', with: [invoiceId: paymentApplication.invoiceId, statustId: 'INVOICE_READY'] }`
  - Removes the row (:1023)
  - invoke SECA reverses GL first (§6.7)

### 6.5 Totals: applied, not applied, outstanding (computed, never stored)
- `InvoiceWorker` (`InvoiceWorker.java`, 842 lines; constants :59-62)
  - `getInvoiceItemTotal`: quantity (default 1) x amount, setScale per line (:109-119)
  - `getInvoiceTotal` (:200, body :212-235): non-tax items + tax total; foreign conversion when actualCurrency false (:231-233)
  - `getInvoiceTaxTotal` (:178-189): per taxAuthPartyId + geo plus unattributed; taxable item types from enum `TAXABLE_INV_ITM_TY` (:168-176; seed `AccountingSeedData.xml:688-694`: INV_SALES_TAX, ITM_SALES_TAX, PINV_SALES_TAX, PITM_SALES_TAX, CRT_SALES_TAX_ADJ); `getInvoiceNoTaxTotal` (:191)
  - `getInvoiceApplied` (:474-517): PaymentAndApplication by invoiceId; payment effectiveDate <= asOf or null (:494-500); orderBy effectiveDate (:503-504); sum (:508-511); **no payment-status filter**; inner join, so a row without paymentId is invisible
  - `getInvoiceNotApplied` (:444-465): total minus applied as of now
  - `getInvoiceItemApplied(delegator, ...)` queries entity "Invoice" with invoiceItemSeqId (:553): looks broken (by reading) [HEAD too]
  - `getInvoiceCurrencyConversionRate` (:585-668): posted entry, then payment amount vs actual amount, then UomConversionDated, then 1
- `PaymentWorker` (`PaymentWorker.java`, 374 lines; constants :52-53)
  - `getPaymentApplied` (overloads :243, :247, :298, :308; body :243-337): applications by paymentId or toPaymentId; converts to actual currency at :322-327
  - `getPaymentAppliedAmount` (:270-291) has its own conversion at :278-283
  - `getPaymentNotApplied` (:339, :346, :353, :357; body :339-372): uses actualCurrencyAmount when actual=true (:347-348)
- Instalment view `getInvoicePaymentInfoList` (`PaymentServices.groovy:74-131`; def `services_payment.xml:129-135`) [HEAD too]
  - InvoiceTerm query with no orderBy (:82-84)
  - Loop over terms whose TermType `parentTypeId == 'FIN_PAYMENT_TERM'` (:91), i.e. FIN_PAY_NETDAYS_1/2/3
    - dueDate `getDayEnd(invoiceDate, termDays)` (:95); NPE when termDays is null (`UtilDateTime.java:804`)
    - amount `(invoiceTerm.termValue * invoiceTotalAmount ) / 100` (:97): percent of total, no setScale
    - paid amounts allocated first-in, in DB order (:101-107); outstanding (:108)
  - Remainder row when money is left over, total <= 0, or instalment percent < 100 (:113)
    - dueDate from the first InvoiceTerm with termTypeId exactly FIN_PAYMENT_TERM (`queryFirst`, unordered, :118-124), else `getDayEnd(invoiceDate)` (:126)
  - Recomputed from invoiceDate on every call; stored `Invoice.dueDate` is not read
  - FIN_PAYMENT_DISC (parent FINANCIAL_TERM) is read by neither branch
- `getInvoicePaymentInfoListByDueDateOffset` (:322-360): asOf `getDayEnd(now, daysOffset)` (:324); excludes CANCELLED and PAID only (:327), so WRITEOFF still listed; keeps rows with outstanding > 0 and dueDate before asOf (:352)
- `getInvoiceRunningTotal` (`InvoiceServicesScript.groovy:439-452`): sums row 0 `outstandingAmount` (:444); base-currency formatting

### 6.6 Statuses and the PAID gates
- Invoice statuses (`AccountingSeedData.xml:894-903`): IN_PROCESS, APPROVED, SENT, RECEIVED, READY, PAID, WRITEOFF (:902), CANCELLED
- Valid changes (:905-931)
  - From IN_PROCESS: READY, CANCELLED, SENT, RECEIVED, APPROVED
  - From SENT: APPROVED, READY, RECEIVED, CANCELLED, IN_PROCESS
  - From RECEIVED: READY, CANCELLED, IN_PROCESS, APPROVED
  - From APPROVED: SENT, READY, CANCELLED
  - READY to PAID (:927), READY to WRITEOFF (:928), READY to CANCELLED (:930)
  - PAID to READY "Unpay" (:929), PAID to CANCELLED (:931)
  - Nothing leaves WRITEOFF
- Payment statuses (:934-940): NOT_PAID, SENT, RECEIVED, CONFIRMED, CANCELLED, VOID
- Payment changes (:941-952)
  - NOT_PAID to SENT, RECEIVED, CANCELLED (:943, only source of CANCELLED), VOID
  - SENT and RECEIVED to CONFIRMED, VOID (:946-948), NOT_PAID (:949-950)
  - CONFIRMED to SENT or RECEIVED (:951-952)
- `setInvoiceStatus` (`InvoiceServicesScript.groovy:233-297`; def `services_invoice.xml:88-98`: statusId required :93, paidDate optional :95)
  - Same status returns success (:241-243); StatusValidChange check (:245-249)
  - **Gate B**: to PAID refused unless `getInvoiceNotApplied == 0` exactly (:252-257); all payment statuses, effectiveDate <= now
  - `invoice.paidDate = parameters.paidDate ?: UtilDateTime.nowTimestamp()` on every change (:260) [pin-only: HEAD sets it only for PAID, HEAD :253]; READY clears it (:262-264)
  - InvoiceStatus history row (:268-270); PAYROL auto-payment (:274-291)
- **Gate A**: `checkInvoicePaymentApplications` (`InvoiceServices.java:2390-2491`) [HEAD too]
  1. Returns success unless the invoice is INVOICE_READY (:2412-2414)
  2. Reads PaymentAndApplication by invoiceId, no date filter, no orderBy (:2419)
  3. Keeps PMNT_RECEIVED receipts and PMNT_SENT disbursements only; CONFIRMED and NOT_PAID dropped (:2422-2431)
  4. `payments.put(payAppl.getString("paymentId"), payAppl.getBigDecimal("amountApplied"));` (:2443): one amount per paymentId, a later row overwrites an earlier one
  5. paidDate = latest effectiveDate (:2445-2451)
  6. Sums with setScale (:2453-2459); requires a positive sum (:2462)
  7. `totalPayments >= invoiceTotal` (:2468): calls setInvoiceStatus INVOICE_PAID with paidDate; returns error when that fails (:2470-2477)
- Gate A fires from: createPaymentApplication commit (`secas.xml:29-32`), `checkPaymentInvoices` on payment RECEIVED/SENT (`InvoiceServices.java:3628-3662`), invoice to READY (`secas_invoice.xml:50-57`)
- **When gate A passes and gate B fails** (by reading): over-application, a NOT_PAID application next to RECEIVED ones, or a RECEIVED payment with a future effectiveDate. setInvoiceStatus returns error, gate A returns error, the caller's transaction is marked rollback-only; the invoice does not quietly stay READY.
- WRITEOFF: status and UI only (`widget/AccountingMenus.xml:447`); no SECA or service posts GL for it

### 6.7 Reversal paths
- **`removePaymentApplication`** (`PaymentServices.groovy:972-1026`)
  - invoke SECA `revertAcctgTransOnRemovePaymentApplications` (`secas_payment.xml:60-62`; `AcctgTransServices.xml:441-455`) runs before the removal
  - Condition: invoiceId from paymentApplication.invoiceId (:445) and paymentId (:446): reverses **every** AcctgTrans of that pair, with D/C flipped, including earlier reversal copies (clone keeps both ids, `GeneralLedgerServices.xml:2438`)
  - Reversal copies net to zero across remove and re-apply cycles; the defects are two live applications sharing the pair (removing one reverses both) and reversals left unposted
  - For a non-invoice application (billing account, toPayment) the null invoiceId compares as IS NULL, so the payment's own trans gets reversed (by reading)
  - PAID invoice: the `statustId` key (:992-994) is dropped by makeValidContext, statusId is required, validation throws, the whole call rolls back (by reading) [pin-only: HEAD :969 uses statusId]
- **`voidPayment`** (`PaymentServices.groovy:362-397`)
  1. setPaymentStatus PMNT_VOID (:368-370); allowed from NOT_PAID, SENT, RECEIVED, not CONFIRMED (`AccountingSeedData.xml:946-948`)
  2. Per application: `Map invoice = from('Invoice').where(invoiceId: it.invoiceId).queryOne()` with no null check (:375), `invoice.statusId` (:376) [pin-only: HEAD :367 `invoice?.`]
  3. PAID invoice set to READY with paidDate null (:377-379); this sidesteps the `statustId` path
  4. removePaymentApplication (:381), so the pair revert runs
  5. Reverses AcctgTrans with `invoiceId: null` and the paymentId (the payment's own trans) (:384-394); posts the reversal only when the original `isPosted == 'Y'` (:391-393)
  6. commit SECA `setFinAccountTransStatus` (`secas_payment.xml:65-69`)
  - Application reversals stay unposted; PAID to READY skips the gate A and invoice-GL SECAs (both exclude "from PAID")
- **`cancelInvoice`** (`InvoiceServicesScript.groovy:309-323`; via `secas_ledger.xml:108-112` on to CANCELLED)
  - invoke SECA `revertAcctgTransOnCancelInvoice` (`secas_invoice.xml:25-27`; `AcctgTransServices.xml:457-470`): trans with this invoiceId and paymentId null (:460), unposted
  - CONFIRMED payments moved back to RECEIVED or SENT (:316-319); `payment.statusId` with no null check (:316) [pin-only: HEAD adds `payment &&`]
  - removePaymentApplication for each application (:320); the invoice is already CANCELLED, so the `statustId` path is not hit
- **`setPaymentStatus` PMNT_CANCELLED** removes each application (:826-833); only from NOT_PAID
- **Payment RECEIVED/SENT to NOT_PAID** (`AccountingSeedData.xml:949-950`): no SECA reverses the payment GL; moving back to RECEIVED/SENT posts the payment again (`secas_ledger.xml:115-136`) and re-posts every existing application (`GeneralLedgerServices.xml:1783-1790`, :2401-2420)
- **Invoice PAID to READY "Unpay"** (:929): no GL effect
- `copyAcctgTransAndEntries` (`GeneralLedgerServices.xml:2433-2472`): clone (:2438), transactionDate now (:2441-2442), `createAcctgTrans` (:2446), D/C flipped when revert=Y (:2457-2467)
- `createAcctgTrans` forces `isPosted: 'N'` (`GeneralLedgerServicesScript.groovy:29-35`, :30); no postAcctgTrans SECA on createAcctgTrans, so reversal copies stay unposted
- Which revert catches an AcctgTrans depends on its ids:
  - invoiceId + paymentId: removePaymentApplication of that pair (so also voidPayment and cancelInvoice, which call it)
  - invoiceId only: cancelInvoice only
  - paymentId only: voidPayment, and removal of a non-invoice application of that payment
  - neither: nothing
- Tests: `AutoPaymentTests.groovy:70` testVoidPayment, :117 testCancelInvoice, :322 testGlPostingsOnVoidPayment; `AutoInvoiceTests.groovy:148` testGlPostingOnCancelInvoice, :219 testGlPostingOnCancelCheckRun; no test calls removePaymentApplication

### 6.8 GL posting and account mapping
- **Entities**
  - `AcctgTrans` :1764: acctgTransTypeId, transactionDate, isPosted, postedDate, glJournalId, glFiscalTypeId, partyId, roleTypeId, invoiceId, paymentId, finAccountTransId, shipmentId; **no paymentApplicationId**
  - `AcctgTransEntry` :1869: glAccountTypeId, glAccountId, organizationPartyId, amount, currencyUomId, origAmount, origCurrencyUomId, debitCreditFlag, dueDate, settlementTermId, reconcileStatusId
  - `GlAccount` :2082, `GlAccountOrganization` :2239, `GlAccountType` :2307, `GlAccountTypeDefault` :2320 (type + org gives account), `GlJournal` :2364
  - `PartyAcctgPreference` :2444: baseCurrencyUomId, invoiceSeqCustMethId, invoiceIdPrefix, `errorGlJournalId` :2465, `enableAccounting` :2471
  - `TaxAuthorityGlAccount` :3982, `PartyGlAccount` :4101
- **Trans types** (`AccountingSeedData.xml:21-57`): CREDIT_MEMO :35, SALES_INVOICE :47, PURCHASE_INVOICE :48, INCOMING_PAYMENT :54, OUTGOING_PAYMENT :55, PAYMENT_APPL :57
- **GlAccountTypes**: 60 (`AccountingSeedData.xml:316-382`); DISCOUNTS_ACCOUNT "Discounts" :343 (no parent, no mapping anywhere; only other hit a label, `AccountingEntityLabels.xml:1793`); ACCPAYABLE_UNAPPLIED :369; ACCREC_UNAPPLIED :370; WRITEOFF, ACCTRECV_WRITEOFF, ACCTPAY_WRITEOFF, COMMISSIONS_WRITEOFF, INTRSTINC_WRITEOFF :374-378; FX_GAIN_LOSS_ACCT, FX_GAIN_ACCOUNT, FX_LOSS_ACCOUNT :380-382; no sales-discount or purchase-discount type
- **`createAcctgTransAndEntries`** (`minilang/ledger/GeneralLedgerServices.xml:199-329`; def `services_ledger.xml:255`)
  - Entries for an org without the internal-org role are ignored (:207-212)
  - `enableAccounting=N`: returns without posting (:218-225)
  - origAmount converted to base currency at transactionDate when set (:229-252)
  - Missing glAccountId resolved by `getGlAccountFromAccountType` (:255-269); unknown glAccountTypeId cleared (:273-278)
  - transactionDate now when empty (:288-290)
  - Negative amount flips D/C (:298-314)
- **Account resolution** `getGlAccountFromAccountType` (:333-530; def `services_ledger.xml:325-340`, glAccountId OUT optional :339)
  1. ITEM_VARIANCE (:335-341), DEPRECIATION (:343-388)
  2. `PartyGlAccount` when glAccountTypeId, partyId and roleTypeId are set (:389-405)
  3. INCOMING_PAYMENT debit / OUTGOING_PAYMENT credit with a paymentId: PaymentMethod.glAccountId, then CreditCardTypeGlAccount, then PaymentMethodTypeGlAccount (:406-444); returns even when nothing is found
  4. productId: ProductGlAccount, ProductCategoryGlAccount (:445-465)
  5. PURCHASE_INVOICE debit / CUST_RTN_INVOICE debit / SALES_INVOICE credit with an invoiceId: InvoiceItemTypeGlAccount (:488), InvoiceItemType.defaultGlAccountId (:493-499), product GlAccountTypeDefault (:501-521), return (:522)
  6. Fallback `GlAccountTypeDefault` (:525-528)
  - `getGlAccountTypeDefaultInline` (:2704-2709): exact organizationPartyId + glAccountTypeId; no fallback to a parent GlAccountType
- **`postAcctgTrans`** (`minilang/ledger/AcctgTransServices.xml:181-358`, 472 lines, identical at HEAD; def `services_ledger.xml:278`)
  - Already posted: error (:184-189)
  - Trial balance: |D - C| >= 0.01 rejected (:196-205); one-sided trans rejected (:206-231)
  - scheduledPostingDate (:238-246)
  - Needs an open `CustomTimePeriod` for the org covering transactionDate (:255-288; none :272-276; closed :279-283)
  - Each entry needs glAccountId (:289-293) and amount (:294-298); verifyOnly (:310-314)
  - On errors: `enableAccounting=N` returns (:323-330); no errorGlJournalId: check-errors, the service fails, the caller's transaction is marked rollback-only (:332-334); with errorGlJournalId: sets that journal, stores, adds a warning to successMessageList, returns success, isPosted stays N (:335-341)
  - Otherwise isPosted Y, postedDate (:350-355)
  - Does not check GlAccountOrganization
- **Sales invoice** `createAcctgTransForSalesInvoice` (:2163-2309)
  - Exact `SALES_INVOICE` only (:2169): INTEREST_INVOICE and PURC_RTN_INVOICE are not posted here (by reading)
  - Credit per non-tax item, glAccountTypeId = invoiceItemTypeId, glAccountId = item.overrideGlAccountId (:2198-2208)
  - Tax credits per authority via TaxAuthorityGlAccount, TAX_ACCOUNT (:2216-2247); unattributed tax (:2260-2271)
  - Debit ACCOUNTS_RECEIVABLE for the total (:2281-2296); type SALES_INVOICE (:2298); transactionDate invoiceDate (:2297-2306)
- **Purchase invoice** `createAcctgTransForPurchaseInvoice` (:1944-2162)
  - hasParentType PURCHASE_INVOICE (:1950): PURCHASE, COMMISSION, PAYROL (CUST_RTN has its own SECA)
  - Debit items (item override :2030); credit ACCOUNTS_PAYABLE (:2102-2116); type PURCHASE_INVOICE (:2120); date invoiceDate (:2124)
  - Re-posts the PAYMENT_APPL of every application whose payment is SENT or CONFIRMED (:2131-2159)
- **Incoming payment** (:1721-1792)
  - Debit has no glAccountTypeId (TODO :1744-1745); resolved through the payment-method branch
  - Credit type from PaymentGlAccountTypeMap (:1753-1757); glAccountId payment.overrideGlAccountId (:1765)
  - Type INCOMING_PAYMENT (:1775); transactionDate effectiveDate (:1776)
  - Posts every existing application of the payment (:1783-1790)
- **Outgoing payment** (:2311-2430): credit by payment method; debit map type (override :2384); type OUTGOING_PAYMENT (:2392); no transactionDate set, so posting time (unlike incoming); posts existing applications whose invoice is READY or PAID (:2401-2420)
- **Payment application** `createAcctgTransAndEntriesForPaymentApplication` (:2530-2652; def `services_ledger.xml:505`)
  - Payment PMNT_NOT_PAID: returns without posting (:2536-2538); posted later with the payment
  - Receipt: map type for (paymentTypeId, partyIdTo) (:2544-2547); skips when that type is ACCOUNTS_RECEIVABLE (:2548-2550); debit the map type with origAmount only, glAccountId payment.overrideGlAccountId (:2552-2560); credit ACCOUNTS_RECEIVABLE (:2569); no FX entry
  - Disbursement: skips when the map type is ACCOUNTS_PAYABLE (:2576-2578); invoice vs payment GL rates (:2580-2585; `UtilAccounting.java:284-306, 308`); credit the map type with amount (:2587-2596, override :2595); FX_GAIN_LOSS_ACCT when rates differ (:2599-2608); debit ACCOUNTS_PAYABLE (:2609-2617); PaymentApplication.overrideGlAccountId (:2618-2619) else TaxAuthorityGlAccount (:2621-2628)
  - Type PAYMENT_APPL (:2636); paymentId and invoiceId set (:2638-2639); no transactionDate, so "now"
- **Customer refund application** (:2473-2528): credit payment.overrideGlAccountId (:2498); debit PaymentApplication.overrideGlAccountId (:2509-2510)
- **Demo mappings, org `Company`** (`applications/datamodel/data/demo/AccountingDemoData.xml`)
  - GlJournal ERROR_JOURNAL (:669); PartyAcctgPreference: USD, INV_HOOK_ENF_SEQ, prefix CI, errorGlJournalId ERROR_JOURNAL (:671-675)
  - InvoiceItemType defaults: INVOICE_ADJ, INV_PROMOTION, INV_DISCOUNT 410000 (:483-485); ITM_* 410000 (:495-498); PINV_* 516100 (:518-527); PITM_* (:529-534); CRT (:568-571; CRT_DISCOUNT 423000 :570); SRT (:593-596; SRT_DISCOUNT 409000 :595)
  - GlAccountOrganization for 410000 (:722, :1254)
  - GlAccountTypeDefault, 23 rows (:755-777): ACCOUNTS_RECEIVABLE 120000, ACCREC_UNAPPLIED 126000, ACCOUNTS_PAYABLE 210000, ACCPAYABLE_UNAPPLIED 216000, SALES 400000, FX 518000, TAX 900000; none for DISCOUNTS_ACCOUNT or the WRITEOFF family
  - PaymentGlAccountTypeMap (:785-801): CUSTOMER_REFUND: CUSTOMER_CREDIT (:785), VENDOR_PAYMENT: ACCPAYABLE_UNAPPLIED (:787), CUSTOMER_PAYMENT: ACCREC_UNAPPLIED (:798)
  - Fiscal periods FY2009 to FY2030 (:851-872)
- **Chart** (`applications/datamodel/data/demo/GlAccountDemoData.xml`)
  - 410000 DISCOUNTS ON SALES, parent 400000, class COGS_EXPENSE (:714-716); already the default for trade, promotion and discount item types
  - 510000 FREIGHT IN, parent 500000 (:749-751)
  - 511000 PROMPT PAYMENT DISCOUNTS, parent 510000, class COGS_EXPENSE (:756-758); 511100 / 511200 "PROMPT PMT DISC - CAT 1/2" (:759-764); no GlAccountOrganization, referenced by no mapping data; contra-COGS, so the likely AP discount-taken account
  - 516100 PURCHASE ORDER ADJUSTMENTS (:807-809)
  - 517000 / 517100 / 517200 typed WRITEOFF / ACCTRECV_WRITEOFF / ACCTPAY_WRITEOFF (:813-821); no defaults or org links
- **Setup chart** (`applications/accounting/data/OfbizSetupChartOfAccountsData.xml`, 419 lines)
  - Discount item types: :26 and :39 map to 410000; :61 and :73 to 516100; :111 CRT_DISCOUNT_ADJ to 423000; :136 SRT_DISCOUNT_ADJ to 409000
  - errorGlJournalId (:216); PaymentGlAccountTypeMap like demo (:326-342)
- **Setup wizard data** (`applications/commonext/data/OfbizSetupGlAccountData.xml`)
  - GlAccountTypeDefault, 19 rows (:85-103): no *_UNAPPLIED, no FX
  - PaymentGlAccountTypeMap (:106-122): VENDOR_PAYMENT: ACCOUNTS_PAYABLE (:108), CUSTOMER_PAYMENT: ACCOUNTS_RECEIVABLE (:119); so for such orgs the application posting is skipped entirely and the payment posts straight to AR/AP
  - errorGlJournalId created by the wizard (`SetupEvents.xml:283-285`)
- **Test data**: `applications/accounting/testdef/data/AccountingTestsData.xml` GlAccountTypeDefault (:42), PaymentGlAccountTypeMap (:47)
- **GL shapes today**
  - AR: IncomingPayment D cash / C ACCREC_UNAPPLIED (full payment); PAYMENT_APPL D ACCREC_UNAPPLIED / C ACCOUNTS_RECEIVABLE (amountApplied)
  - AP: OutgoingPayment D ACCPAYABLE_UNAPPLIED / C cash; PAYMENT_APPL D ACCOUNTS_PAYABLE (+ FX) / C ACCPAYABLE_UNAPPLIED
  - No discount entries anywhere
- **GL dates**: invoice trans at invoiceDate; incoming payment at effectiveDate (:1776); outgoing payment, PAYMENT_APPL and reversal copies at posting time (no transactionDate set, default :288-290)
- Rate type `DISCOUNTED` (`AccountingSeedData.xml:1017`) is rate data, unrelated to terms

### 6.9 Terms from orders to invoices
- **TermType tree** (`applications/datamodel/data/seed/PartySeedData.xml:390-399`)
  - FINANCIAL_TERM :390
  - FIN_PAYMENT_TERM "Payment (net days)" :391; children FIN_PAY_NETDAYS_1/2/3 "Payment net days, part 1/2/3" :392-394
  - FIN_PAYMENT_DISC "Payment (discounted if paid within specified days)", parent FINANCIAL_TERM :395: a sibling of FIN_PAYMENT_TERM, not a child
  - FIN_PAYMENT_FIXDAY :396, FIN_LATE_FEE_TERM "Late Fee (percent)" :397, FIN_COLLECT_TERM :398, FIN_NORTN_ITEM_TERM :399
  - Label `applications/party/config/PartyEntityLabels.xml:4388` (German "Zahlung (Skonto bei Zahlung innerhalb der angegebenen Frist)")
  - TermType entity `party-entitymodel.xml:484`
- **FIN_PAYMENT_DISC at the pin** (`git grep`: 9 lines, 4 files)
  - Seed row `PartySeedData.xml:395`; label `PartyEntityLabels.xml:4388`
  - `applications/order/testdef/data/QuoteTestData.xml:27-28`: QuoteTerm quote 9000, item 00002 termValue 10 termDays 5 uomId CNY; item 00003 termValue 5 termDays 7
  - `applications/order/src/main/groovy/org/apache/ofbiz/order/order/test/QuoteTests.groovy:217,225,236,251,283`: quote term CRUD only (create with termValue 40L, termDays 4L)
  - No reader in Java, Groovy, minilang, FTL or widgets; same at HEAD (seed, labels, quote tests only)
  - Copied from orders to invoices generically, with every other OrderTerm
  - Selectable in generic term pickers (`widget/accounting/AccountingInvoiceTermCpd.xml:295,319`, parentTypeId in FINANCIAL_TERM, FIN_PAYMENT_TERM; `InvoiceForms.xml:522`); other pickers `BillingAccountForms.xml:278-279`, `AgreementForms.xml:276,358`, `applications/order/widget/ordermgr/OrderForms.xml:129`, `QuoteForms.xml:545`, `OrderTerms.groovy:52`
  - Invoice PDF prints `${termType.description!} ${term.description!} ${term.termDays!} ${term.textValue!}`, never termValue (`template/invoice/InvoiceReportItems.fo.ftl:53-57`)
- **Term entities** (Derby types: `fieldtypederby.xml:29-37`)
  - `OrderTerm` (`order-entitymodel.xml:1294-1321`): PK termTypeId + orderId + orderItemSeqId (:1305-1307), so one term per type per order or item; termValue currency-amount (:1300), termDays, textValue, description, uomId
  - `QuoteTerm` (:1597-1620): termValue numeric, Long (:1603), so 1.5 cannot be stored; PK termTypeId + quoteId + quoteItemSeqId (:1608-1610)
  - `AgreementTerm` (`party-entitymodel.xml:389-419`): termValue currency-precise (:399), termDays (:400), invoiceItemTypeId, fromDate/thruDate, min/maxQuantity; PK agreementTermId alone (:405)
  - `BillingAccountTerm` (`accounting-entitymodel.xml:2726-2741`): termValue currency-amount (:2732); no textValue/description
- **termValue meaning**
  - Instalment children: percent of total (`PaymentServices.groovy:97`)
  - LATE_FEE description says "(percent)" (`PartySeedData.xml:397`)
  - Commission: `AgreementServices.java:123` adds FIN_COMM_FIXED termValue as an amount; :126-127 comment "5% is stored as termValue of 5.0"
  - FIN_PAYMENT_DISC: neither description nor code says percent or amount
- **How terms travel into orders** (uomId dropped on every path)
  - Agreement to cart: `ShoppingCartHelper.java:1147-1160` (clears cart terms; date-filtered AgreementTerm; termValue, termDays, textValue; no uomId)
  - Billing account to cart: `CheckOutHelper.java:300-312` (skipped when the cart already has that termTypeId)
  - Quote to cart: `ShoppingCartServices.java:750, 778-791` (quoteItemSeqId used as orderItemSeqId; null termValue/termDays become 0; no uomId)
  - Order reload to cart: `ShoppingCartServices.java:187, 315-327`
  - UI event addOrderTerm: termDays mandatory only for FIN_PAYMENT_TERM and its children (`ShoppingCartEvents.java:1315-1374`, :1344-1345); no uomId (:1372)
  - Cart model `ShoppingCart.java:3572-3611`
  - Cart to OrderTerm: header terms stored with orderItemSeqId `_NA_` (`OrderServices.java:1052-1062`)
  - Replacement order on return copies the original OrderTerms (`OrderReturnServices.java:2084-2086`)
  - Quote copy with copyQuoteTerms=Y (`QuoteServicesScript.groovy:282-287`); storeQuote TODO (`QuoteServices.java:278-279`)
- **Order to invoice**
  - `OrderReadHelper.getOrderTermNetDays` (`OrderReadHelper.java:467-476`): first FIN_PAYMENT_TERM; warns when more than one; only caller `InvoiceServices.java:234`
  - All OrderTerms copied (`InvoiceServices.java:280-283`) through `createInvoiceTerms` (:2619-2648)
    - invoiceItemSeqId `_NA_` (:2626): item-level order terms collapse to invoice level, so one invoice can carry several FIN_PAYMENT_DISC rows
    - textValue/description skipped for BillingAccountTerm (:2630-2633); uomId copied (:2634)
    - errors only logged (:2640-2645)
  - Shipment invoicing: the term copy sits outside the new-invoice branch, but the usual reuse case fails earlier at createInvoiceRole (permission needs IN_PROCESS, or duplicate PK); duplicate terms need a reused invoice still IN_PROCESS and orders with disjoint roles (unverified at runtime)
- **InvoiceTerm services**
  - `createInvoiceTerm` (`services_invoice.xml:164-169`): permission checkInvoiceStatusInProgress, IN_PROCESS only
  - `updateInvoiceTerm` (:170-175), `deleteInvoiceTerm` (:176-180): acctgInvoicePermissionCheck only, so termValue, termDays and termTypeId stay editable on READY or PAID invoices; an existing term can be retyped to FIN_PAYMENT_DISC [HEAD too]
  - Editing terms does not touch `Invoice.dueDate`; no SECA on InvoiceTerm
  - No validation of termValue range or sign, or termDays presence, for FIN_PAYMENT_DISC
- **Demo and test data**
  - Payment terms in demo: QuoteTerm CQ0001 FIN_PAYMENT_TERM 30 days (`OrderDemoData.xml:163`); AgreementTerm 9000 AGR_SALES FIN_PAYMENT_FIXDAY 30 days (:2594)
  - AgreementTerm id 9000 is also the commission term at :2197 (commission terms 9000-9002, :2195-2199); the PK is agreementTermId alone, so one row overwrites the other on load (by reading) [HEAD too: HEAD :2174, :2571]
  - No InvoiceTerm or OrderTerm rows in demo or test data
  - Test fixture `applications/accounting/testdef/data/PaymentApplicationTestsData.xml:22-35`: invoice appltest10000 SALES_INVOICE INVOICE_SENT, one item 20.00, invoiceDate 2006-04-25; payment CUSTOMER_PAYMENT EFT_ACCOUNT PMNT_NOT_PAID 20 USD; billing account; CUSTOMER_REFUND and SALES_TAX_PAYMENT payments; no terms

### 6.10 Due dates, dates and time zones
- Invoice side: invoiceDate = eventDate or now; dueDate from net days at creation (`InvoiceServices.java:228-238`); null without FIN_PAYMENT_TERM
- Payment side: `Payment.effectiveDate` is the only business date on Payment; default now; editable while NOT_PAID; future-dated by createPaymentFromOrder (§6.3)
- PaymentApplication: no business date
- paidDate: gate A passes the latest effectiveDate (`InvoiceServices.java:2445-2451`); setInvoiceStatus stamps now on every change at the pin (§6.6), so paidDate is not a reliable payment date
- `UtilDateTime` (`framework/base/src/main/java/org/apache/ofbiz/base/util/UtilDateTime.java`, 1222 lines)
  - `getIntervalInDays` (:72-74): ms / 86400000, truncated; `addDaysToTimestamp` (:76-82): 24 h multiples; `nowTimestamp()` (:155)
  - `getDayStart(ts[, days])` (:210-216), `getDayEnd(ts[, Long days])` (:222-228): JVM default TimeZone and Locale
  - TZ-aware `getDayStart` (:788-795), `getDayEnd` (:801-809): 23:59:59 (:803), `daysLater.intValue()` (:804, NPE when null), setNanos(0) (:806); so the window ends at 23:59:59.000 and the last second's fractions fall outside
  - HEAD change: documentation only
- Default timezone: `start.properties:71` commented out, so the JVM default (`Config.java:111-113, 210-212`); no org or user timezone at any term or due-date call site
- Postgres date-time is TIMESTAMPTZ, Derby TIMESTAMP (§3)
- Service context carries `timeZone`; `GroovyBaseScript.runService` passes it on
- `getInvoiceApplied` counts only effectiveDate <= now (or null), so a future-dated payment is not yet applied for gate B

### 6.11 Tax, currency, rounding
- **Tax**: separate InvoiceItems; `getInvoiceTotal` includes tax; sales invoice GL credits tax per authority at READY (`GeneralLedgerServices.xml:2216-2271`); nothing ties terms to tax
- **Currency**
  - `Invoice.currencyUomId`: order currency on order invoices (`InvoiceServices.java:241-263`)
  - Org base currency `PartyAcctgPreference.baseCurrencyUomId` via `getPartyAccountingPreferences` (`services_admin.xml:73-79`, OUT only `partyAccountingPreference`)
  - Default `currency.uom.id.default=USD` (`framework/common/config/general.properties:24`)
  - `convertUom` (`framework/common/servicedef/services.xml:347-358`); GL converts at transactionDate (`GeneralLedgerServices.xml:229-252`)
  - createPaymentApplication allows invoice currency == payment currency or actual currency (`PaymentServices.groovy:755-761`)
  - FX gain/loss only on AP application posting (:2599-2608); AR application posting has no FX entry
  - `isInvoiceInForeignCurrency` (`InvoiceServicesScript.groovy:641-651`): `isForeign: invoice.currencyUomId == serviceResult.baseCurrencyUomId`, but the service OUT has only partyAccountingPreference, so always false [pin-only: HEAD :641-642 reads the preference and uses `!=`]
- **Rounding** (`applications/accounting/config/arithmetic.properties`, identical at HEAD)
  - `ledger.decimals=4` (:24), `ledger.rounding=HalfUp` (:26)
  - `invoice.decimals=2` (:29), `invoice.SALES_INVOICE.decimals=2` (:30), `invoice.PURCHASE_INVOICE.decimals=4` (:31), `invoice.rounding=ROUND_HALF_UP` (:32)
  - `order.decimals=2`, `order.rounding=ROUND_HALF_UP` (:36-37); `finaccount.*` (:41-44)
  - `salestax.calc.decimals=3`, `salestax.final.decimals=2`, `salestax.rounding` (:50-52)
  - No `payment.*` keys
  - The 4-dp purchase setting is read only in createInvoiceForOrder (`InvoiceServices.java:191-195`); InvoiceWorker, PaymentWorker and InvoiceServices constants use `invoice.decimals` (`InvoiceWorker.java:59-62`, `PaymentWorker.java:52-53`, `InvoiceServices.java:112-115`)
  - Read with `UtilNumber.getBigDecimalScale` / `getRoundingMode` (`framework/base/src/main/java/org/apache/ofbiz/base/util/UtilNumber.java:187,212,245,264`); default file arithmetic.properties (:35), default HALF_UP (:39), name mapping (:278-292)
  - GL arithmetic in minilang: `getGlArithmeticSettingsInline` (`GeneralLedgerServices.xml:2857-2861`)
  - Columns fixed at 2 dp: amountApplied, Payment.amount, termValue; InvoiceItem.amount 3 dp
  - Trial balance tolerates sub-cent differences (< 0.01, `AcctgTransServices.xml:196-205`)
- **Worked rounding case** (arithmetic only, no repo rule): invoice 10.25, 2 %
  - Discount first: 10.25 x 0.02 = 0.205, HALF_UP 0.21, payment 10.04
  - Payable first: 10.25 x 0.98 = 10.045, HALF_UP 10.05, discount 0.20
  - With tax: net 100.00 + tax 10.00: discount 2.20 on gross vs 2.00 on net

### 6.12 Non-cash settlement today, and why none is the feature
- **INVOICE_WRITEOFF**: status only (READY to WRITEOFF :928; menu `widget/AccountingMenus.xml:447`; bulk `ListInvoices.ftl:91`, `PurchaseInvoices.ftl:93`); no GL; whole invoice, terminal, no amount; excluded by party screens and matching, still in the due-date list
- **Payment.overrideGlAccountId**: redirects the cash leg's GL account (incoming credit :1765, outgoing debit :2384, receipt application debit :2559, disbursement application credit :2595); the same account on both receipt legs nets out; not a discount
- **PaymentApplication.overrideGlAccountId**: redirects the GL account of a cash application; the row still needs a paymentId; honoured only for the AP-side debit (:2618-2619) and the refund debit (:2509-2510), where a discount use would leave AP uncleared; ignored for AR (:2552-2569); never set by any service (not an IN of createPaymentApplication; accepted but unused by the update services)
- **Customer return invoice + CUSTOMER_REFUND / billing-account credit**: return GL (`GeneralLedgerServices.xml:1794+`, `secas_ledger.xml:100-107`); return credit `OrderReturnServices.java:925-1010`; needs return context, separate document, money out
- **Discount item types** (INV/ITM/PINV/PITM_DISCOUNT_ADJ): change the invoice total; item create/update/remove need IN_PROCESS (`services_invoice.xml:122-142`); a READY invoice is already posted and nothing re-posts an added item; fits discounts known before READY only
- **Pseudo-payment through payment-method GL**: a payment whose method maps to a discount account would post D discount / C AR through the normal chain (by reading); costs: shows as a payment in lists, party totals, deposits and check runs; needs paymentMethodTypeId (`PaymentServices.groovy:64-66`); counted by gate A only when RECEIVED/SENT; voiding the cash payment leaves it applied
- **Deduction**: payroll deduction types, CRUD only; **SettlementTerm**: net-day codes, only an FK on AcctgTransEntry
- **Payment-to-payment (toPaymentId), tax authority (taxAuthGeoId)**: not invoice settlement
- **CREDIT_MEMO trans type** (`AccountingSeedData.xml:35`): used in `AutoAcctgLedgerTests.groovy:33,43`; no credit-memo invoice flow traced (unverified)
- **Commission and interest invoices**: separate documents, not settlement

### 6.13 AR/AP reports and readers
- No aging report: `git grep -iwE "aging|ageing"` gives 0 hits repo-wide
- `invoice/InvoiceReport.groovy` (:23-59): past due by stored dueDate < now (:28); purchase statuses RECEIVED/IN_PROCESS/READY (:31); sales SENT/APPROVED/READY (:33); running total (:40, :54); screens `widget/ar/CommonScreens.xml:77-107`, `widget/ap/CommonScreens.xml:75-105`, `InvoiceScreens.xml:344-402`
- `payment/FindInvoicesByDueDate.groovy:26-27` calls getInvoicePaymentInfoListByDueDateOffset; screens `PaymentScreens.xml:560,592`; grid `PaymentForms.xml:524-532`
- `getInvoicePaymentInfoList` call sites (9): `InvoiceServicesScript.groovy:442`; `PaymentServices.groovy:185, 215, 345`; `CommissionRun.ftl:106`, `PurchaseInvoices.ftl:190`, `ListInvoices.ftl:165-166` (`.get(0)`), `CommissionRunSearchResult.ftl:43`; `InvoiceForms.xml:721`; 7 read row 0 only; :345 and the grid read all rows
- `getInvoiceApplied` / `getInvoiceNotApplied`: 28 call lines in 16 files outside InvoiceWorker, e.g. `InvoiceServicesScript.groovy:253` (gate B), `PaymentServices.groovy:80, 763`, `InvoiceServices.java:2965`, `PaymentGatewayServices.java:1180`, `InvoiceForms.xml:73, 755, 788`, `widget/ar/forms/InvoiceForms.xml:39`, `widget/ap/forms/InvoiceForms.xml:38`, `AccountingInvoiceCpd.xml:151-152, 314`, `AccountingInvoiceApplicationCpd.xml:103-104, 153`, `ListNotAppliedPayments.groovy:54-56`, `ListNotAppliedInvoices.groovy:73`, `DisplayInvoiceAmounts.groovy:22`, `PartyFinancialHistory.groovy:67-71`, `UnAppliedInvoicesForParty.groovy:55`, `PartyForms.xml:1098-1099`
- Party screens: `UnAppliedInvoicesForParty.groovy:34-36` excludes IN_PROCESS/WRITEOFF/CANCELLED; `PartyFinancialHistory.groovy` payment totals
- GL reports: trial balance `GlAccountTrialBalance.groovy:38-39`; balances from posted entries only, so unposted reversals and ERROR_JOURNAL trans are missing (unverified)
- `reports/`: 13 Groovy GL and financial report scripts

## 7. Build and tests

### 7.1 Toolchain
- Java 17: `build.gradle:136, 152-155`; README JDK 17 (`README.md:73`); CI temurin 17 (`.github/workflows/gradle.yml:43-48`); Docker `eclipse-temurin:17` (`Dockerfile:21`)
- Derby held below 10.17, which needs Java 21 (`dependencies.gradle:100-101`)
- Gradle 8.14.4: `distributionUrl=https\://services.gradle.org/distributions/gradle-8.14.4-bin.zip` (`gradle/wrapper/gradle-wrapper.properties:3`)
- Wrapper jar not tracked; `gradle/init-gradle-wrapper.sh`:
  - Reads the version from distributionUrl (:101)
  - Downloads `https://github.com/gradle/gradle/raw/v$RELEASE/gradle/wrapper/gradle-wrapper.jar` (:108)
  - Checks sha256 from `https://services.gradle.org/distributions/gradle-$RELEASE-wrapper.jar.sha256` (:109)
  - Has `--upgrade`; Windows: `init-gradle-wrapper.bat` calls `gradle/init-gradle-wrapper.ps1`; README `README.md:92-109`
- Key libraries (`dependencies.gradle`): groovy-all 5.0.5 (:71), freemarker 2.3.34 (:72), tomcat 10.1.54 (:63), CodeNarc 3.7.0-groovy-4.0 (:110), test libs (:89-92)

### 7.2 Dependencies and network (measured 2026-11-04)
- `dependencies.gradle` lists 87 direct dependency coordinates; only two are not on Maven Central:
  - `net.fortuna.ical4j:ical4j:1.0-rc4-atlassian-12` (`dependencies.gradle:37`) from packages.atlassian.com
  - `org.activiti:activiti-juel-jakarta:8.1.0` (:39) from artifacts.alfresco.com
- Both repositories are public, but they could not be reached while these notes were checked, so a first build has to confirm them
- No direct dependency uses the restlet repository; transitive dependencies not checked (unverified)
- Repositories (`build.gradle:181-216`, allprojects):
  1. `mavenCentral()` (:183)
  2. `https://maven.restlet.talend.com` (:189)
  3. `https://packages.atlassian.com/maven-3rdparty/` (:193); stale comment "rc3-atlassian-11" (:192)
  4. `https://repo1.maven.org/maven2` with `metadataSources { artifact() }` (:200-203): Maven Central's own host, artifact-only metadata; comment says the flute jar for plugins:birt
  5. `https://clojars.org/repo` (:206)
  6. `https://artifacts.alfresco.com/nexus/content/repositories/public/` (:209)
  - Commented out: `https://repository.apache.org/content/repositories/snapshots/` (:211-214)
- Gradle Plugin Portal: needed on a cold cache; no `pluginManagement` in `settings.gradle`
  - Seven plugins at `"latest.release"` (`build.gradle:32-38`: asciidoctor, owasp, use-latest-versions, ben-manes versions, ManifestClasspath, git-hooks, node-gradle); dynamic versions force lookups
  - Settings plugins develocity 3.19.2 and common-custom-user-data 2.6.0 (`settings.gradle:20-23`) also come from the portal
- develocity.apache.org (`settings.gradle:28`): not needed for a build; CI detection via GITHUB_ACTIONS (:25); scans publish only when authenticated (:32)
- Node and npm (`buildSrc/src/main/groovy/ofbiz-node-conventions.gradle`)
  - `download = true`, Node 20.19.0 (:44-45); download host is the plugin default, not stated in the repo
  - `npm ci` / `npm install` (:48); `classes` depends on `npmInstall` unless `-PskipNpmInstall` (:55-57); Node and npm are needed only when npmInstall runs
  - Registry `https://registry.npmjs.org` (23 `resolved` entries in `themes/common-theme/webapp/common-theme/js/package-lock.json`)
- Wrapper sites (github.com, services.gradle.org): needed only when the wrapper jar or the distribution is missing
- Optional plugins: `pullAllPluginsSource.sh` clones `https://github.com/apache/ofbiz-plugins.git` with `--branch $(git branch --show-current)`; empty in a detached-HEAD checkout (by reading)

### 7.3 Gradle tasks and commands
- `defaultTasks 'jar', 'test'` (`build.gradle:179`)
- `loadAll` = generateSecretKeys + `ofbiz --load-data` (:389-392); `testIntegration` = `ofbiz --test` (:394-397), "You must run loadAll before"
- `createTestReports` (:698-710); `--test` runs are finalizedBy it (:1044-1054)
- `generateSecretKeys` (:746-775); `createPlugin` (:778)
- `cleanData` (:974-976); other clean tasks (:971-1012); `cleanAll` (:1018-1021)
- Rule `"ofbiz <cmd>"` (:1027-1032); `ofbizBackground` (:1034-1038)
- Unit `test`: JUnit with a jmockit agent (:370-373)
- Command forms (`README.md`): `./gradlew cleanAll loadAll` (:120-122); `gradlew cleanAll loadAll testIntegration` (:575); `gradlew "ofbiz --test component=entity --test suitename=entitytests --test case=entity-query-tests"` (:540); `--test loglevel=fatal` (:528); `--debug-jvm` variants (:546-566); reader-scoped loads (:406-455)

### 7.4 Test runner and testdefs
- `TestRunContainer`: args loglevel, component, suitename, case (:70, :127-131); results in `runtime/logs/test-results/` (:53)
- Suites use delegator `test` (`ModelTestSuite`, :52, :63), the same `localderby` database as the app (`entityengine.xml:67-70`); tests assume demo data is loaded
- testdef XML under `*/testdef/`: 61 files = 35 test-suite files + 25 data files + 1 unregistered Selenium testcase file (`applications/marketing/testdef/CommEventCreateOpportunity_testsuite.xml`)
- 35 suites registered with `<test-suite loader="main" location=...>`: accounting 6, order 6, product 8, party 3, content 1, manufacturing 1, marketing 1, workeffort 1, framework 8 (base, common x2, entity, minilang, service, webapp, widget); a 36th grep match is the template `framework/resources/templates/ofbiz-component.xml:44`
- Elements (`framework/testtools/dtd/test-suite.xsd`): test-suite :37, test-case :49, test-group :59, junit-test-suite :70, service-test :92, webdriver-test :101, simple-method-test :110, entity-xml :127
- Active usage: junit-test-suite 65, entity-xml 25, simple-method-test 16, service-test 3
- 102 active `<test-case>` entries (108 raw grep matches, including commented ones)

### 7.5 Accounting tests
- Suites (`applications/accounting/ofbiz-component.xml:74-79`): accountingtests, paymenttests, paymentappltests, invoicetests, fixedassettests, ratetests
  - `accountingtests.xml`: loads `testdef/data/AccountingTestsData.xml` (:25-27); 13 junit-test-suite entries: `org.apache.ofbiz.order.test.FinAccountTest` (order component, :30) and 12 AutoAcctg* classes (:33-68): TransSales, TransPurchase, Admin, Agreement, Budget, Cost, FinAccount, FixedAsset, Invoice, Payment, PaymentGateway, Ledger
  - `paymenttests.xml`: AutoPaymentTests; SecurePay test commented out (:28-31)
  - `paymentappltests.xml` (:20-31): loads `PaymentApplicationTestsData.xml` (2006 dates, before the FY2009 demo periods), runs PaymentApplicationTests
  - `invoicetests.xml`: test-group AutoInvoiceTests + InvoicePerShipmentTests
- Classes: 18 files, 3046 lines, 98 test methods in `applications/accounting/src/main/groovy/org/apache/ofbiz/accounting/accounting/`
  - Pattern: `class X extends OFBizTestCase` (`framework/service/src/main/java/org/apache/ofbiz/service/testtools/OFBizTestCase.java`), `X(String name) { super(name) }`, `void testY()`, `dispatcher.runSync(...)`, `ServiceUtil.isSuccess`, `from(...)`, system `userLogin`; manual cleanup with `delegator.removeAll(...)` (`PaymentApplicationTests.groovy:34-61`)
  - GL checks: AcctgTrans by paymentId + acctgTransTypeId; `glJournalId != 'ERROR_JOURNAL'` (`AutoAcctgTransSalesTests.groovy:69, 138, 157, 209`); D/C balance helper `checkEntriesBalance` (:228)
  - PaymentApplicationTests (151 lines): testInvoiceAppl :34, testToPayment :63, testBillingAppl :93, testTaxGeoId :126
  - AutoPaymentTests (480 lines, 10 tests): testVoidPayment :70, testCancelInvoice :117, testCreatePaymentAndPaymentGroupForInvoices :165, testGlPostingsOnVoidPayment :322, testGlPostingOnCheckRun :394
  - AutoInvoiceTests (305 lines, 4 tests: :34 GetInvoiceTotal, :109 CommissionRun, :148 GlPostingOnCancelInvoice, :219 GlPostingOnCancelCheckRun)
  - AutoAcctgTransSalesTests (243 lines, 3 tests: testAcctgTransOnSalesInvoice :89, testAcctgTransOnPaymentReceivedFromCustomer :172)
  - AutoAcctgTransPurchaseTests (196 lines, 3 tests: :90 edit PO invoice, :139 payment sent)
  - AutoAcctgPaymentTests (95 lines, 5 tests); AutoAcctgInvoiceTests (220 lines, 12 tests; term CRUD with FINANCIAL_TERM :191-208)
- No test touches FIN_PAYMENT_DISC, DISCOUNTS_ACCOUNT or early-payment logic; no test calls removePaymentApplication
- Unregistered: `applications/accounting/minilang/test/PaymentApplicationTests.xml` (no testdef reference)
- Quote tests touching FIN_PAYMENT_DISC: registered in `applications/order/ofbiz-component.xml:52-57`; `applications/order/testdef/QuoteTests.xml` (minilang case commented :28)
- Integration tests were not run at the pin for these notes; no baseline exists

### 7.6 Static checks and CI
- Checkstyle 10.20.2, `maxErrors 0`, `config/checkstyle/checkstyle.xml` (`build.gradle:319-332`)
- CodeNarc with `config/codenarc/codenarc.groovy`, 0/0/0 allowed violations (:338-343)
- Git hook pre-push: `checkstyleMain codenarcMain codenarcTest` (:333-335); `-Xlint` options (:157-165)
- SpotBugs: config file only
- CI `gradle.yml`: trunk and release* pushes and PRs (:22-26); init wrapper (:53-54), `pullAllPluginsSource.sh` (:56-57), `./gradlew check javadoc` (:59-60); integration tests commented out, "BuildBot does it anyway" (:62-65)
- CodeQL workflow; `.pre-commit-config.yaml`: checkstyle, gitleaks, rubocop, shellcheck, eslint, whitespace

## 8. Conventions a new feature must follow

**Where things go (accounting)**
- Service definitions: `applications/accounting/servicedef/services_payment.xml`, `services_invoice.xml`, `services_ledger.xml`; a new file needs a `<service-resource>` in `applications/accounting/ofbiz-component.xml:36-66`
- New logic in Groovy under `src/main/groovy/org/apache/ofbiz/accounting/{payment,invoice,ledger}/` (e.g. `PaymentServices.groovy`, `InvoiceServicesScript.groovy`); Java exists in `InvoiceServices.java` / `InvoiceWorker.java`; minilang is deprecated (`developer-manual.adoc:81`) though all GL posting is minilang
- SECAs: `servicedef/secas_payment.xml`, `secas_invoice.xml`, `secas_ledger.xml`
- Entities: `applications/datamodel/entitydef/accounting-entitymodel.xml`; accounting's own `applications/accounting/entitydef/entitymodel_reports.xml`
- Types and statuses: `applications/datamodel/data/seed/AccountingSeedData.xml`; term types `PartySeedData.xml:390-399`
- Org GL mappings live in 3 datasets: demo `AccountingDemoData.xml`, setup chart `applications/accounting/data/OfbizSetupChartOfAccountsData.xml`, setup wizard `applications/commonext/data/OfbizSetupGlAccountData.xml`; plus test `AccountingTestsData.xml`
- Accounts are reached by GlAccountType through the resolution chain (§6.8), not hardcoded glAccountId
- Tests: Groovy class in `src/main/groovy/org/apache/ofbiz/accounting/accounting/` at the pin, registered as `<junit-test-suite class-name=...>` in a `testdef/*.xml`, fixtures in `testdef/data/` loaded by `<entity-xml action="load">`; trunk uses `src/test/groovy/` and `<jupiter-test-suite>` (§9.1)

**Labels** (`applications/accounting/config/`)
- `AccountingUiLabels.xml`: 24576 lines, 1574 keys; `AccountingErrorUiLabels.xml`: 974 lines, 96 keys; `AccountingEntityLabels.xml`: 5586 lines, 357 keys
- Format `<property key="..."><value xml:lang="en">... ${placeholder} ...</value>...</property>`, e.g. `AccountingInvoiceCannotChangeStatusToPaid` (`AccountingUiLabels.xml:7481-7489`)
- Entity descriptions use keys like `GlAccountType.description.X` (`AccountingEntityLabels.xml:1793`)

**Errors**
- Java: `ServiceUtil.returnError(UtilProperties.getMessage(RESOURCE, "Key", UtilMisc.toMap(...), locale))` (`InvoiceServices.java:2397-2408, 2475-2477`); `returnSuccess`; `returnFailure`
- Groovy at the pin: `return error(label('AccountingUiLabels', 'Key'))` (`PaymentServices.groovy:41, 65`), `failure(label(...))` (:815, :821), `success([...])`; trunk: `require(...)`
- Failure does not roll back; error does (`ServiceDispatcher.java:578-585`)
- Raw English strings also occur (`InvoiceServices.java:2950-2955`)

**Numbers, dates, currency**
- Scale and rounding from `arithmetic.properties` via UtilNumber (§6.11); Java constants `InvoiceServices.java:112-115`, `InvoiceWorker.java:59-62`, `PaymentWorker.java:52-53`
- Day boundaries through `UtilDateTime.getDayEnd` / `getDayStart` (§6.10)
- Currency through `convertUom` and PartyAcctgPreference.baseCurrencyUomId

**Properties**
- `applications/accounting/config/accounting.properties` (identical at HEAD): `invoiceProcessing=YY` (:29), `accounting.payment.application.autocreate=Y` (:35), PO not-paid payment autocreate (:38), SO not-paid payment autocreate (:41), `create.invoice.per.shipment=Y` (:44)
- Read through `EntityUtilProperties`, which checks the `SystemProperty` entity first (`framework/entity/src/main/java/org/apache/ofbiz/entity/util/EntityUtilProperties.java:54-86`), e.g. `PaymentServices.groovy:849`

**Quality gates**
- Checkstyle and CodeNarc with zero allowed violations (§7.6)
- Integration tests need `loadAll` first and share the app's Derby database

## 9. Known bugs and traps at the pin

### 9.1 Pin vs trunk HEAD (`8cfe7b17`, 2026-11-04)
- `applications/accounting`: pin 399 tracked files, HEAD 364; 67 changed, 57 removed or moved, 22 added (by blob hash)
- Tests moved to `applications/accounting/src/test/groovy/` (21 files at HEAD, including new SendInvoicePerEmailPermissionTests and TaxAuthorityTests); testdefs use `<jupiter-test-suite>` (HEAD `accountingtests.xml:30-37`); accounting test-suites at HEAD `ofbiz-component.xml:61-67`, new `TaxAuthorityTests.xml`
- Model and data changes at HEAD: payment-gateway removals only; nothing discount-related
- `services_invoice.xml` at HEAD adds permissions to sendInvoicePerEmail, updateInvoiceItemType, createInvoiceAttribute only; InvoiceTerm services unchanged
- Identical at HEAD: `AcctgTransServices.xml`, `GeneralLedgerServicesScript.groovy`, `secas.xml`, `secas_payment.xml`, `secas_invoice.xml`, `services_payment.xml`, `arithmetic.properties`, `accounting.properties`; `secas_ledger.xml` differs only by run-as-user additions (:117-135)
- `GeneralLedgerServices.xml` at HEAD: changes in inventory and workeffort methods only (around :1640-1691)
- Identical functions at HEAD: getInvoicePaymentInfoList, checkInvoicePaymentApplications, createInvoiceForOrder, createInvoiceTerms, storePaymentApplication, createInvoicesFromShipments, getInvoiceApplied, getInvoiceItemApplied
- Fixed on HEAD [pin-only]: `statustId` in removePaymentApplication and updateInvoice; inverted payment-side auto-match; isInvoiceInForeignCurrency; voidPayment null deref; paidDate on every status change; check-run closure returns; createPayment preference typo; cancelInvoice null payment; failure vs error in setPaymentStatus; createPaymentFromOrder failure on an existing payment; ACCOUNTING permission checks in createPaymentApplication and updatePaymentApplicationDefBd
- Still on HEAD [HEAD too]: gate A paymentId map (HEAD `InvoiceServices.java:2444`); unposted reversal copies; revert by (invoiceId, paymentId); invoice-side path with no SECAs; createPaymentApplication cap; RECEIVED/SENT to NOT_PAID without reversal; AP double posting; exact-amount matching; InvoiceTerm editable after READY; getDayEnd in the JVM timezone; DISCOUNTS_ACCOUNT unmapped; AgreementTerm 9000 collision

### 9.2 Traps, with what each breaks for a discount
1. **`statustId` in removePaymentApplication** (`PaymentServices.groovy:992-994`): un-applying from a PAID invoice fails validation and rolls back (statusId required, `services_invoice.xml:93`; by reading). Breaks: reversing a discounted, PAID invoice through the remove path (UI `controller.xml:228-232, 389-393`, `AccountingInvoiceApplicationCpd.xml:50-55`). [pin-only, HEAD :969]
2. **`statustId` in updateInvoice** (`InvoiceServicesScript.groovy:177, 183, 189-191`): with a requested statusId, `setNonPKFields([*: parameters, statustId: ...])` writes that statusId straight onto the row (:183), then setInvoiceStatus is called without statusId (:189-191), validation fails and the whole call rolls back (by reading). Breaks: test setup that moves an invoice to READY through updateInvoice. [pin-only, HEAD :179, :187]
3. **Gate A one amount per paymentId** (`InvoiceServices.java:2443`): a second application row of the same payment replaces the first in the sum. Breaks: a discount stored as an extra row on the cash payment never reaches PAID. [HEAD too]
4. **Gate A vs gate B disagree** (`InvoiceServices.java:2419-2468` vs `InvoiceServicesScript.groovy:252-257`, `InvoiceWorker.java:494-500`): A ignores dates and payment status beyond RECEIVED/SENT; B counts every status with effectiveDate <= now and wants exactly 0. A passes and B fails: caller's transaction rollback-only (by reading). Breaks: future-dated fixtures, over-application, discount rows from NOT_PAID payments. [HEAD too]
5. **Invoice-side path fires no SECAs** (`InvoiceServices.java:2685-3478`, no dispatcher call up to :3630): no PAID and no PAYMENT_APPL at apply time. Breaks: a discount hooked only on createPaymentApplication or its SECAs never happens from the invoice screens. [HEAD too]
6. **storePaymentApplication merges** (`InvoiceServices.java:3527-3532`): same-key rows are summed. Breaks: a discount row keyed like the cash row merges into it. `overrideGlAccountId` accepted (`services_payment.xml:88, 99`) but ignored. [HEAD too]
7. **createPaymentApplication caps and checks no status** (`PaymentServices.groovy:763-765`): caller amountApplied ignored for invoices; any invoice or payment status accepted. Breaks: tests passing amountApplied; applying 100 from a 98 payment; discount granted on a cancelled or IN_PROCESS invoice unless checked. [HEAD too]
8. **Revert by (invoiceId, paymentId)** (`AcctgTransServices.xml:441-455`): every trans of the pair, also a discount trans carrying both ids; two live applications of the pair both reversed when one is removed. [HEAD too]
9. **Reversal copies unposted** (`GeneralLedgerServicesScript.groovy:30`; `GeneralLedgerServices.xml:2433-2472`): posted balances keep the original. Breaks: "discount reversed" invisible in posted GL; "net 0" holds only when unposted entries count. [HEAD too]
10. **Null ids compare as IS NULL in minilang** (`EntityFinderUtil.java:207-208`): removing a billing-account or toPayment application reverses the payment's own trans (by reading). Breaks: a discount trans with paymentId and no invoiceId gets reversed by unrelated removals. [HEAD too]
11. **voidPayment null deref** (`PaymentServices.groovy:375-376`): NPE for an application without invoiceId. Breaks: voiding a payment that also carries a billing-account or discount row without invoiceId. [pin-only, HEAD :367]
12. **voidPayment skips re-checks** (`secas_invoice.xml:50-57`, `secas_ledger.xml:91-99` exclude "from PAID"): PAID to READY re-runs neither gate A nor invoice GL; application reversals unposted. [HEAD too]
13. **Payment back to NOT_PAID** (`AccountingSeedData.xml:949-950`): no GL reversal; back to RECEIVED/SENT duplicates the payment posting and re-posts every application (`GeneralLedgerServices.xml:1783-1790, 2401-2420`). Breaks: a discount posted with the application posts twice. [HEAD too]
14. **AP application posted twice** (by reading): createPaymentApplication posts PAYMENT_APPL when the payment is SENT (`secas_ledger.xml:138-147`); the purchase invoice going READY re-posts applications of SENT/CONFIRMED payments (`GeneralLedgerServices.xml:2131-2159`). Breaks: AP discount entries doubled when applied before READY. [HEAD too]
15. **Unmapped account** (`AcctgTransServices.xml:289-293, 332-341`): with errorGlJournalId (demo Company `AccountingDemoData.xml:675`, setup chart :216, wizard `SetupEvents.xml:283-285`) the trans parks unposted in ERROR_JOURNAL and the call succeeds; without it the call fails and the transaction is rollback-only; with enableAccounting=N nothing posts. Breaks: a test that checks only success passes with no posted discount. [HEAD too]
16. **DISCOUNTS_ACCOUNT unmapped** (`AccountingSeedData.xml:343`): no GlAccountTypeDefault in demo (:755-777), wizard (:85-103), setup chart or test data. [HEAD too]
17. **Exact GlAccountTypeDefault lookup** (`GeneralLedgerServices.xml:2704-2709`): a child type of DISCOUNTS_ACCOUNT does not fall back to a parent row. Breaks: new subtypes without their own default rows land in ERROR_JOURNAL. [HEAD too]
18. **Wizard orgs never post applications** (`OfbizSetupGlAccountData.xml:108, 119`; `GeneralLedgerServices.xml:2548-2550, 2576-2578`). Breaks: a discount posted inside the application posting is skipped for such orgs. [HEAD too]
19. **PAYMENT_APPL dated now** (`GeneralLedgerServices.xml:2636-2639`, default :288-290) and periods (`AcctgTransServices.xml:255-288`): a closed or missing period fails posting. Breaks: discount GL date and period differ from the payment's. [HEAD too]
20. **AR application has no FX entry** (`GeneralLedgerServices.xml:2552-2569`; AP has one :2599-2608). Breaks: foreign-currency AR discounts post without FX. [HEAD too]
21. **Sales posting exact type** (`GeneralLedgerServices.xml:2169`): INTEREST_INVOICE and PURC_RTN_INVOICE get no sales posting; purchase posting covers COMMISSION and PAYROL (:1950). Breaks: AR/AP subtype coverage differs. [HEAD too]
22. **getInvoicePaymentInfoList** (`PaymentServices.groovy:82-131`): ignores FIN_PAYMENT_DISC; remainder due date from the first FIN_PAYMENT_TERM row (unordered); unscaled division; DB-order allocation; NPE on null termDays. Breaks: discount-only invoices due on the invoice day (:126); 7 of 9 callers read row 0. [HEAD too]
23. **getDayEnd in the JVM timezone, 23:59:59.000** (`UtilDateTime.java:222-228, 801-809`). Breaks: window boundary moves with the server timezone; fractions of the last second fall outside. [HEAD too]
24. **getInvoiceApplied** (`InvoiceWorker.java:474-517`): no status filter, effectiveDate <= now, inner join. Breaks: a settlement record without paymentId is invisible; NOT_PAID applications count for gate B. [HEAD too]
25. **InvoiceTerm editable after READY** (`services_invoice.xml:170-180`): values, days and type editable on READY or PAID invoices. Breaks: eligibility can change after the fact. [HEAD too]
26. **Terms copied and collapsed** (`InvoiceServices.java:280-283, 2619-2648`): all OrderTerms, item terms to `_NA_`, errors only logged. Breaks: several FIN_PAYMENT_DISC rows on one invoice; silently missing terms. [HEAD too]
27. **Exact-amount auto-matching** (`PaymentServices.groovy:872-882, 930-937`): a 98.00 payment never auto-matches a 100.00 invoice. [HEAD too]; inverted payment-side create (:943-945) [pin-only]; isForeign always false (`InvoiceServicesScript.groovy:641-651`) [pin-only]
28. **Check run** (`PaymentServices.groovy:179-235`): amounts from row 0 outstanding [HEAD too]; `.each` closures swallow errors (:183, :191, :213) [pin-only]. Breaks: AP check runs pay full outstanding, no discount.
29. **createPaymentFromOrder** (`PaymentServices.groovy:670-717`): full grandTotal, NOT_PAID, effectiveDate now + net days with an agreement [HEAD too]; failure on an existing payment (:636) [pin-only]. Breaks: auto-created payments never match a discounted amount and can be future-dated.
30. **paidDate stamped on every status change** (`InvoiceServicesScript.groovy:260`). Breaks: paidDate unusable as the payment date. [pin-only, HEAD :253]
31. **setPaymentStatus failure, not error** (`PaymentServices.groovy:814-816, 819-822`): no rollback, caller continues (by reading). [pin-only]
32. **updatePayment lets effectiveDate move while NOT_PAID** (`PaymentServices.groovy:140-154`): no bound check. Breaks: eligibility by effectiveDate can be backdated before receipt. [HEAD too]
33. **Missing permission checks on the apply services** (`services_payment.xml:69-102`). [pin-only, HEAD adds them in code]
34. **createPayment preference typo** (`PaymentServices.groovy:60`): paymentId taken from the preference's paymentMethodId. [pin-only]
35. **cancelInvoice null payment** (`InvoiceServicesScript.groovy:316`). [pin-only]
36. **AgreementTerm 9000 collision** (`OrderDemoData.xml:2197, 2594`; PK `party-entitymodel.xml:405`): one demo term overwrites the other on load (by reading). Breaks: demo agreement terms are not a reliable fixture. [HEAD too]
37. **getInvoiceItemApplied** (`InvoiceWorker.java:553`): queries "Invoice" by invoiceItemSeqId (by reading). Breaks: item-level discount logic built on it. [HEAD too]
38. **Purchase decimals** (`InvoiceServices.java:191-195` only): 4-dp purchase setting ignored by workers. Breaks: AP rounding differs between item creation and totals. [HEAD too]
- Off the discount path (not a reason for difficulty): `'PMNT_NOTPAID'` typo (real id PMNT_NOT_PAID) in `applications/party/src/main/groovy/org/apache/ofbiz/party/party/PartyFinancialHistory.groovy:87` and `UnAppliedPaymentsForParty.groovy:35`, so their NOT_EQUAL filter excludes nothing [HEAD too]

## 10. Unverified items

- Who runs OFBiz in production (not in the repo)
- Purchase-invoice debit-side details (`GeneralLedgerServices.xml:1944-2100`) and the outgoing-payment posting body beyond the cited lines
- How `OfbizSetupChartOfAccountsData.xml` gets loaded (no tracked reference)
- Whether integration tests roll back data (classes clean up by hand with `removeAll`)
- Run order of two SECAs on the same event in different files (`secas.xml` gate A vs `secas_ledger.xml` application GL)
- Runtime of the `statustId` failure and of the gate A/B rollback-only path (traced by reading only)
- Shipment-invoicing term duplication at runtime
- AP double posting at runtime
- Unposted reversals missing from GL reports
- Node download host (plugin default)
- Transitive dependency repositories; reachability of packages.atlassian.com and artifacts.alfresco.com from a normal network
- `loadAll` duration; integration baseline at the pin (never run)
- Deduction and SettlementTerm usage beyond CRUD and the AcctgTransEntry FK
- CREDIT_MEMO flows beyond the ledger tests
- `docs/` beyond the cited lines; `Dockerfile` beyond base image and wrapper steps

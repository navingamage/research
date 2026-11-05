# Apache OFBiz: early-payment discount ("2/10 net 30")

- **Repo:** https://github.com/apache/ofbiz-framework/tree/f05a0c2268f163573a0e0895ec381593de1283db
- **Stack:** Java 17 + Groovy, Gradle
- **Domain:** ERP accounting (AR/AP)
- **Status:** ready
- **Full notes:** [codebase notes and facts for the write-ups](../top3/ofbiz-early-payment-discount/) (checked against the pin on 2026-11-04)

**Pre-flight (checked)**
- Pin 2026-05-10, before Claude co-authored commits started (2026-05-16, 21 since). No AI files at the pin.
- Repositories are Maven Central plus restlet.talend, atlassian 3rd-party, clojars and alfresco public. All are public with no login, and OFBiz's own CI uses them, but I couldn't reach them from my side. The first build in a lane confirms this.
- Re-checked 2026-11-04: only 2 of the 87 direct dependencies are off Maven Central, ical4j `1.0-rc4-atlassian-12` (packages.atlassian.com) and activiti-juel-jakarta `8.1.0` (artifacts.alfresco.com). No direct dependency uses the restlet repo. A cold cache also needs the Gradle plugin portal and npm.
- 1,147 Java and 484 Groovy files, 35 test suites. Tests run on embedded Derby, so nothing outside is needed.
- Gap: term type `FIN_PAYMENT_DISC` ("discounted if paid within specified days") exists in seed data, the UI and quote tests, and is copied from orders to invoices. **No service ever reads it**, at the pin or at HEAD (re-checked at trunk `8cfe7b17` on 2026-11-04). No commit mentions it.

**Facts about the gap**
- `InvoiceTerm` keeps the percent (`termValue`) and the days (`termDays`).
- `PaymentServices.groovy` `getInvoicePaymentInfoList` only looks at the `FIN_PAYMENT_TERM` family: its children (`FIN_PAY_NETDAYS_*`) for the instalments, and the `FIN_PAYMENT_TERM` row itself for the remainder's due date. `FIN_PAYMENT_DISC` is a different branch, so it's skipped.
- An invoice only moves to `INVOICE_PAID` when its applications cover the full total, and it has to pass two different checks. `checkInvoicePaymentApplications` (`InvoiceServices.java` around line 2468) counts received or sent payments, keeps one amount per payment, and needs at least the total. `setInvoiceStatus` then needs the unapplied amount to be exactly 0.
- A GL account type `DISCOUNTS_ACCOUNT` exists in seed data, but nothing maps it to a GL account.

**Where a solution can go wrong**
- The discount has to settle the invoice without any cash moving, or the invoice stays open forever with a small leftover balance.
- It applies to both sales (AR) and purchase (AP) invoices, which need different GL postings.
- Which date counts: the payment's effective date, its received date, or when it was applied. Day-end and timezone edges (`getDayEnd`).
- Partial payment inside the window means a proportional discount. One payment can also cover several invoices.
- Tax: whether the discount reduces taxable value or not, depending on jurisdiction.
- Multi-currency (`actualCurrencyAmount`) and rounding (`invoice.decimals`/`rounding` in `arithmetic.properties`).
- Voiding a payment, or removing an application, must undo the discount and reopen the invoice. Returns and credit memos raised after a discount was taken.
- AR aging and outstanding-amount reports have to agree.

**Verification in the lane:** run `gradle/init-gradle-wrapper.sh` first (the wrapper jar isn't committed), then `./gradlew loadAll` and `"ofbiz --test component=accounting"`. Loading data takes about 5–10 min.

# Odoo POS: exchanges at the till

- **Repo:** https://github.com/odoo/odoo/tree/e3e47d97b536b93e69bb9df181406675af9272a1 (2026-04-27)
- **Target:** V2 · Create · Web frontend · Enterprise-grade · Casual prose · Interactive/pairing
- **Stack:** Python + PostgreSQL backend; Point of Sale is an offline-capable OWL (Odoo's own JS framework) single-page app
- **Status:** written up 2026-11-05. The one-turn pre-check passed all 4 traps, rated 4.5/5 (see below)

## Pre-flight
- **AI files:** none at the pin. Vetting found 6 commits since 2025 with a `Copilot <copilot@github.com>` trailer (first author date 2026-04-24, possibly an IDE trailer). The pin is the first-parent commit before the first of them lands. Re-check `git log e3e47d97 --grep=Co-authored-by -i` before submitting.
- **Build sources:** PyPI only. Odoo Enterprise has a pricing page.
- **Size:** 46,262 files and about 3.8k frontend JS files. Point of Sale has about 175 JS modules in `point_of_sale/static/src` plus 43 `pos_*` add-on modules.
- **Gap:** no exchange action at the till, at the pin or on master `e04968ab`. A back-office manager can open a paid order, "Return Products" and add lines to the draft refund, but there's nothing on the POS Orders screen.

## 1. Prompt (paste into Claude)
> Can you add proper exchanges to the POS? Today when someone swaps an item, the cashier has to refund it, finish that refund, then ring up the new item as a separate sale. I'd like an Exchange button next to Refund on the Orders screen: you pick what's coming back exactly like for a refund, then land on the normal product screen to add the replacement items (changing quantities etc. as usual), and the customer pays or gets back only the difference, on one ticket. The returned items must be credited exactly what a normal refund of them would give back. The new items must cost exactly what they'd cost in a fresh sale for that customer today, meaning their prices and our automatic promotions. Returned quantities still count against what's left to refund on the original ticket, and if the original sale was invoiced, the exchange gets invoiced too, like a refund would.

## 2. High-Level Task Goal

**What the task is about.**
- **Point of Sale:** Odoo's till app. It runs in the browser, works offline, and syncs paid orders to the server, where they become accounting entries and, optionally, customer invoices.
- **How a refund works today:**
  - On the Orders (ticket) screen, a cashier picks a past paid order and the lines and quantities coming back.
  - The POS builds a new order with negative lines. Each line points at the original line (`refunded_orderline_id`), and the order is flagged `is_refund`.
  - The cashier goes straight to payment.
  - Quantities already refunded are tracked per original line, so the same item can't be refunded twice.
- **Add-on modules that change refunds and new lines:**
  - **pos_discount** (a global discount added to the order as a discount line) adjusts what a refund credits.
  - **pos_loyalty** (automatic promotions and rewards) recomputes rewards whenever a line is added.
- **Pricing inputs:**
  - **Pricelist:** sets prices. A customer can have their own; otherwise the till's default applies, and a cashier can pick one manually per order.
  - **Fiscal position:** maps taxes.
- **Invoicing:** an invoiced sale's refund is invoiced as a credit note.
- **The feature:** an exchange is one ticket. It holds the returned lines, credited exactly like a refund of them, plus newly added items priced exactly like a fresh sale for the same customer today. The customer settles only the net difference, in either direction.

**Success criteria**
- **The Exchange button:** it appears next to Refund on the Orders screen for paid orders. Picking what comes back works exactly like refund selection, with the same quantity limits and combo handling.
- **Adding replacements:** after picking the returned items, the cashier lands on the normal product screen with the returned lines on the ticket. They can add replacement items and change their quantities with the numpad as usual.
- **One ticket, net difference:** the customer pays a positive difference or gets a negative one back, on a single ticket that validates and syncs.
- **Credits under a global discount (pos_discount):** returned items are credited exactly what a normal refund of the same lines credits, including when the original had a global discount, and replacement items don't pick that discount up.
  - Example: sell 2 × 50 with a 10% global discount, so 90 is paid. A normal refund of one item credits 45. Exchanging that item for another 50 item leaves 5 to pay.
- **Customer pricing:** new items get the customer's own current pricelist and fiscal position, as in a fresh sale.
  - Example: a customer on a "Wholesale −20%" pricelist exchanges anything for a 100 item, which shows 80.
  - A pricelist picked by hand on the original sale (e.g. "Clearance −50%") is not carried over.
- **Promotions (pos_loyalty):** automatic promotions are computed on the new items as in a fresh sale. Returned lines neither shrink nor inflate a reward, and they don't affect minimum-amount thresholds.
  - Example: with an automatic "10% off the order", exchanging a 40 item for a 60 item gives 60 − 6 − 40 = 14, not 18.
- **Refundable quantities:** returned quantities reduce what can still be refunded or exchanged on the original ticket.
- **Invoicing:** if the original sale was invoiced, the exchange is set to be invoiced by default, and it syncs with a posted accounting document. This must hold both when the customer pays more and when they get money back. There must be no "negative total" error or failed sync.
- **Nothing else changes:** existing refunds, sales and their tests behave as before.
- **Must-nots:**
  - special-casing test products or amounts
  - turning off pos_discount or pos_loyalty behaviour to dodge the interaction
  - leaving the replacement lines uneditable
  - reporting invoicing as working without handling the case where the customer pays more

## 3. Decision Points
1. **The global discount on returned items.**
   - **Better:** notices that a normal refund credits the discounted amount only because pos_discount patches `TicketScreen.onDoRefund`, and that pos_discount's model listeners re-apply the order's global discount over every line on each line update. It keeps that discount on the returned lines only.
   - **Worse:** copies the core `onDoRefund` body, so returned items are credited at full price. Or it reuses `onDoRefund` as is, so the listener spreads the old percentage across the replacement items and the two cancel out.
2. **Whether an exchange is a refund order.**
   - **Better:** traces `is_refund` to its three consequences and decides deliberately, handling each one:
     - the numpad lock on the product screen
     - the auto-invoice default on the payment screen
     - the backend's `out_refund` move type

     So a net-positive exchange can be edited and its invoice posts.
   - **Worse:** keeps `is_refund = true` and only relaxes `isSaleDisallowed`. The replacement lines can't be edited, and a customer-pays-more exchange becomes a credit note with a negative total that `account.move` refuses to post at sync.
3. **Where new-item prices and promotions come from.**
   - **Better:**
     - applies the customer's current pricelist and fiscal position the way `setPartner` does
     - keeps returned lines at their original prices
     - excludes the negative return lines from pos_loyalty's discountable base and thresholds
   - **Worse:** either of two mistakes.
     - Copies the refund's `pricelist_id = order.pricelist_id`, so an old manual pricelist prices the new items. Or relies on `addNewOrder({partner_id})`, which forces the till defaults.
     - Lets an order-level promotion compute on the net (new minus returned).

## 4. Difficulty & Codebase Understanding
Odoo is an ERP with a Python/PostgreSQL backend and OWL frontends, about 46k files. Point of Sale is an offline-capable single-page app of about 175 JS modules, with behaviour layered in by 43 `pos_*` add-ons. The task touches:
- the ticket screen's refund flow (`onDoRefund`, `getRefundLinesDetails`)
- the order model (`isRefund`, `isSaleDisallowed`, and pricelist and fiscal-position selection in `createNewOrder` and `setPartner`)
- the payment screen's auto-invoice rule
- `pos_order.py`'s choice of invoice move type

It's hard because what a refund credits isn't in the refund code at all. pos_discount patches `onDoRefund`, then re-applies the global discount through model listeners on every line update. pos_loyalty patches `addLineToCurrentOrder` and computes rewards over all lines, including negative refund lines. The core refund copies the original pricelist, which is harmless for refund lines (they're never repriced) but wrong for new ones. The single `is_refund` flag drives the numpad lock, the invoice default and the backend's credit-note choice at the same time. So the obvious approach of reusing the refund with a flag gives either an uneditable ticket or an invoice that can't be posted.

## Pre-check (2026-11-05)
- **How it was run:** sandboxed, one turn, with only the prompt. Nothing could be executed (no Postgres, and the policy blocks running repo code). It was then graded by an independent grader and an adversarial verifier against the trap list written beforehand.
- **Size:** about 35 files, +1.2k lines, 9 Python tests, 1 tour and JS unit tests, none of them run.
- **Result: 4 of 4 traps passed, rated 4.5/5.**
  - added a new `_postExchange` hook that pos_discount overrides, with a mode where the discount applies only to returned lines
  - customer pricelist and fiscal position applied via `setPartner`; returned lines keep the original fiscal position
  - `is_refund = false` plus `is_exchange`, giving an `out_invoice` with negative lines when the customer pays more and a linked `out_refund` when the customer gets money back
  - pos_loyalty maths over new lines only
- **Gaps it disclosed:**
  - nothing executed
  - Exchange hidden in Spain (TicketBAI, Veri\*Factu), Saudi Arabia and Jordan
  - Razorpay refuses money-back exchanges (`payment_razorpay.js:134` checks `!order.isRefund`)
  - cash rounding applies to the net amount
  - other e-invoicing localizations not reviewed

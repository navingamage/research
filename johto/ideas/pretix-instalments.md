# Pretix: paying for an order in instalments

- **Repo:** https://github.com/pretix/pretix/tree/e6f8f178aff537bc32dc7359a5f561b8074a269b
- **Stack:** Python, Django 5.2
- **Domain:** Event ticketing and payments
- **Status:** ready

**Pre-flight (checked)**
- Pin `e6f8f178` is from 2026-10-29. Not one AI co-authored commit in the repo's whole history, and no AI files.
- 2,747 files: 995 Python source files in `src/pretix` and 182 test modules (pytest, SQLite). Dependencies come from PyPI. Tests run without any outside service.
- It has a hosted product with a pricing page (pretix.eu). It's a real application, not a fork, and not part of SWE-bench.
- Gap: no instalment or payment-plan feature at the pin or at HEAD, and no commit mentions one. The only matches are PayPal "Pay Later" (PayPal offers the financing itself) and a Stripe comment about Japan. No deposit concept.

**Facts about the gap**
- An order is pending, paid, expired or canceled (`base/models/orders.py`). It stays pending until fully paid, then expires at its payment deadline.
- Several `OrderPayment`s per order already work, and partially paid invoices exist (a `settings.py` option). There's still no schedule and no instalment due dates.
- The payment deadline settings that exist: `payment_term_days`, `payment_term_last`, `payment_term_expire_automatically`.

**Where a solution can go wrong**
- The order state machine: after the first instalment, is the order valid? Ticket download (`ticket_download_pending`), check-in, and quota hold all depend on that.
- Expiry: a later instalment that's missed. Does the order expire or get cancelled? What happens to the money already paid (refund, cancellation fee)?
- Order changes (items added or removed, fees recalculated, `OrderFee` payment fees charged per payment) move the total, so the remaining instalments have to be recalculated.
- The last instalment has to fall before the event, and before `payment_term_last`. Due dates follow the event's timezone, and event series (subevents) can each have different dates.
- Invoices and VAT on part-payments: one invoice or one per instalment, and `RoundingCorrectionMixin` rounding.
- Refunds and cancellations spread across several payments and providers. Gift cards.
- Reminder emails, and existing behaviour must stay the same when the feature is off.

**Verification in the lane:** `uv`/pip install `-e .[dev]` from `src`, then `pytest` on the order and payment tests. The full suite is large, so run part of it.

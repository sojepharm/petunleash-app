# Centravet / Sojepharm — Full Codex Audit Request (V77)

Please perform a full technical audit of this application before further production changes. Treat the following business rules as requirements. Do not silently change them. If a rule is ambiguous, report it before changing behavior.

## Highest-priority bugs observed in live testing

1. **Wholesale cart is not reliably synchronized across devices.**
   - Same wholesale account on laptop and mobile can show different cart totals/items.
   - Deleting/clearing cart on one device did not remove it from the other device.
   - Requirement: one server-backed wholesale cart per authenticated wholesale account. Add/update/delete/clear on one device must be reflected on other devices after refresh or short polling/revalidation.
   - Avoid localStorage as the source of truth for authenticated wholesale carts.

2. **Cancel/Delete order cleanup is incomplete.**
   - After cancelling a submitted wholesale request, old proforma/order data can remain visible.
   - Live cart / admin live stock/cart state can remain stale.
   - Requirement: pending order cancellation/delete must remove it from active/live views, invalidate active proforma/invoice access for that cancelled request, and never post rewards or deduct stock.
   - If a confirmed order is cancelled/returned, restore stock and reverse any rewards atomically and only once.

3. **Invoice issuer/header logic is wrong in some cases.**
   - A Sojepharm wholesale invoice sometimes shows Centravet instead of SOJEPHARM SARL and misses the agreed client details.
   - Requirement for Sojepharm official invoice: main issuer is **SOJEPHARM SARL**. Do not show Centravet logo/name as issuer.
   - Requirement for Centravet invoice: keep Centravet as issuer.
   - Client information should appear opposite the issuer in the header, not underneath it: client/shop name, address/location, phone. Then invoice number, reference and date.
   - Example test client: Pet Value, Achrafieh, phone number from the account.

4. **Performance / browser freeze.**
   - Chrome previously displayed “Fix the tab slowing your browser” for the wholesale page.
   - Audit for MutationObserver loops, repeated DOM mutations, runaway timers/polling, duplicate event listeners, excessive rerenders, expensive cart synchronization and memory leaks.
   - Ensure no whole-document MutationObserver callback mutates the same observed subtree in a loop.

## Wholesale order lifecycle — required behavior

- Customer Submit creates a **Pending** order only.
- Submit must NOT auto-confirm.
- Submit must NOT deduct stock.
- Submit must NOT add confirmed rewards.
- Admin manually clicks **Confirm order**.
- Only Confirm deducts stock and posts earned rewards.
- Cart should remain populated immediately after Submit unless the business flow explicitly clears it later; do not silently empty it.
- Customer can delete/clear a cart before submission.
- Pending submitted request can be cancelled/deleted; cancellation must clean active/live views.
- Confirm/cancel/return operations must be idempotent and transactional so double-clicks/retries cannot double-deduct or double-credit.

## Wholesale cart UX

- Quantity uses minus / number / plus.
- Button initially dark/navy: `Add to cart`.
- After click: green `✓ Added to cart`.
- Changing quantity resets button state.
- Repeated Add clicks must not silently accumulate unintended quantities; selected quantity should be the intended cart quantity unless UI clearly says otherwise.
- Cart must show item count, unit prices, line totals and grand total.
- Remove any redundant floating wholesale rewards badge; rewards information belongs inside the Shopping Cart/account experience.

## Rewards rules

- Current wholesale reward rule used in testing: **2% of eligible order value**.
- 100 points = $1.00 (1 point = $0.01).
- Rewards accumulate across confirmed invoices and stay in the customer account until redeemed.
- Pending order/cart may show projected reward, but it must not become available until admin confirms.
- Pending reward from the current cart/order cannot be redeemed on that same order.
- Rewards are not cash payouts.
- Future redemption UX should support all or partial redemption as a discount on a later invoice.
- Do not automatically reduce an invoice by rewards without explicit redemption.
- Invoice/account statement should support: old balance, redeemed amount/points, earned points/value, new balance.
- Before implementing redemption, verify and report the accounting rule for whether redemption discounts apply pre-VAT or post-VAT, and whether points are earned on gross or net-after-redemption amount.

## Invoice / VAT rules

- **Centravet**: official invoice, currently no separate TVA/VAT line.
- **SOJEPHARM SARL**: VAT registered; wholesale invoice uses **11% TVA**.
- Sojepharm invoice must not look like Centravet is selling to Sojepharm.
- Invoice numbering is a short sequential series per issuer: 001, 002, 003…
- CVP reference is an internal/reference number, separate from invoice number.
- PDF filename/subject may include invoice number.
- Header layout should be clean and professional: issuer on one side, client details opposite, then invoice metadata.

## Customer / admin order views

- Customer submitted-order view must show status, item prices, line totals, total and invoice/proforma/PDF link appropriate to status.
- Admin Submitted wholesale orders must show pending orders quickly and provide Confirm order.
- Cancelled/deleted orders must not remain as active/live orders.
- Audit stale cached UI states and refresh/revalidation behavior.

## Stock rules

- Stock is integer quantity.
- Confirm deducts once.
- Cancel pending: no stock change.
- Return/cancel after confirmed: restore once.
- Audit race conditions, duplicate confirmations, duplicate inventory movement records and negative stock edge cases.
- Live stock/admin displays must not keep stale quantities after cancellation/return.

## WhatsApp / email

- Wholesale WhatsApp business recipient is fixed to **9613923231** (local 03923231), independent of customer phone.
- Testing from the same WhatsApp account may correctly display “Message yourself”. Do not treat that as a code bug.
- `wa.me` cannot silently attach a PDF; it can send a direct invoice/PDF URL.
- Email may attach the PDF.
- Audit URL generation, access control and stale/cancelled invoice URLs.

## Security audit

Please inspect authentication/session handling, admin/wholesale authorization, input validation, SQL injection risks, XSS, CSRF, predictable invoice URLs, direct-object-reference access to other customers’ orders/invoices/rewards, insecure password handling, secrets in source/bundles, and unsafe file upload/import behavior.

## Performance / maintainability audit

Please identify minified/compiled production patches that should be moved back into maintainable source code. Check cache-busting strategy, service/static caching, polling frequency, duplicate logic between frontend and backend, localStorage/server conflicts, and any temporary patch scripts that can conflict with React state.

## Required deliverable from Codex

1. List every confirmed bug/risk found, grouped by Critical / High / Medium / Low.
2. For each, show the exact file/function involved and explain why it happens.
3. Fix safe, unambiguous bugs directly on this audit branch.
4. For business-rule ambiguities, do not guess: list the decision required.
5. Add or update tests for order state transitions, stock, rewards, cart sync, invoice issuer/VAT, cancellation, authorization and duplicate actions.
6. Run available tests/build/lint/type checks and report results.
7. Provide a concise final checklist for production deployment and rollback.

## Do not change these without explicit approval

- Pending → manual admin Confirm workflow.
- Sojepharm 11% TVA vs Centravet no separate TVA line.
- Sojepharm vs Centravet issuer identity.
- Fixed WhatsApp recipient.
- Wholesale reward conversion and 2% rule currently used in testing.

Branch prepared for audit: `codex-audit-v77-20260906`.

---
name: saved-cards
description: Implement Tapaya saved cards and charge-later flows - save a card, Apple Pay, or Google Pay with consent at checkout or with a zero-amount setup, store the paymentMethodId, charge it later as a merchant-initiated (recurring or unscheduled) or customer-initiated payment, handle declines and 3-D Secure, hold and capture funds, refund, and manage saved methods. Use when the user asks about subscriptions, renewals, free trials, top-ups, hotel holds, faster repeat checkout, card on file, or the payments and paymentMethods APIs in Tapaya Checkout.
---

# Tapaya saved cards and charge later

Help the user save a payment method with the customer's consent and charge it
later, using the `@tapayadot/checkout` Node.js SDK or the matching HTTP
endpoints. Ground every detail in the docs.

Sources:

- Save Cards & Charge Later: https://docs.tapaya.com/online/integration/saved_payment_methods
- Use Cases (trials, memberships, stores, hotels, top-ups): https://docs.tapaya.com/online/integration/use_cases
- Node.js SDK, Payments and Saved payment methods: https://docs.tapaya.com/online/integration/node_sdk#payments
- API Integration, Saved cards and payments endpoints: https://docs.tapaya.com/online/integration/api_integration#saved-cards-and-payments
- Checkout Security, saved methods: https://docs.tapaya.com/online/security/checkout_security#protect-saved-methods-and-delivery-approvals
- Testing: https://docs.tapaya.com/online/getting-started/testing

## 1. Check the docs first

Before answering or writing code, read the current pages with the
`tapaya-docs` MCP tools:

- `get_page` `/online/integration/saved_payment_methods` for the fields,
  consent rules, decline reasons, and hold lifecycle.
- `get_page` `/online/integration/use_cases` for the flow that matches the
  user's business.
- `get_page` `/online/integration/node_sdk` for exact parameter and return
  types of `payments.*` and `paymentMethods.*`.
- These endpoints are listed on `/online/integration/api_integration`. Use
  `search_docs`/`get_page` for them; do not assume `search_api_endpoints`
  covers Checkout.

Cite the page URL for each rule you state. If something is not documented,
say so.

## 2. Ask what the user is building

Map the business to a documented use case, then follow that section:

| Business | Pattern |
| --- | --- |
| Free trial, booking guarantee | €0 setup with `savePaymentMethod: 'required'`, charge later off-session |
| Subscription, membership | Pay and save (`off_session`), then `merchantInitiatedReason: 'recurring'` |
| Top-ups, variable bills | Save with first top-up, then `'unscheduled'` charges |
| Online store, repeat buyers | Save `on_session`, customer-initiated charge with 3-D Secure |
| Hotel, rental | €0 guarantee, `captureMethod: 'manual'` hold, increment, capture |

Also confirm: hosted or embedded checkout, and a sandbox merchant checkout key
(`tapaya_test_…`) kept in server config. In `.env.example`, keep the secret
commented out:

```bash
TAPAYA_ENVIRONMENT=sandbox
# TAPAYA_SECRET_KEY=tapaya_test_xxx  # paste your sandbox merchant checkout key
```

## 3. The flow (verify each call before writing it)

1. **Save at checkout.** Add `customerReference` (your authenticated
   customer's ID), `savePaymentMethod` (`'optional'` shows an unchecked box;
   `'required'` saves with consent you already collected; `'none'`), and
   `paymentMethodUsage` (`'off_session'` to charge later, `'on_session'` for
   faster repeat checkout) to a hosted session or an embedded payment.
   Optional `savingTerms: { text, url }`; `text` needs `locale` on hosted
   sessions. For embedded payments created on submission, show the same
   policy in the browser with `cardSaving`.
2. **Store the method.** Read the session or payment on the server until
   `paymentMethodId` is set; saving can finish after the payment succeeds.
   Keep the method's `consent` record with your billing records. A failed save
   does not undo the payment; never charge again to retry saving.
3. **Charge later (merchant-initiated).** `tapaya.payments.create({ merchantOrderId, amount, currency, paymentMethodId, initiator: 'merchant', merchantInitiatedReason }, { idempotencyKey })`.
   One `merchantOrderId` per renewal or top-up. Branch on `status`, not HTTP
   200: `declined` carries `declineReason`; `pending` needs another read.
4. **Charge with the customer present.** `initiator: 'customer'` plus a
   `returnUrl` on an allowed checkout website. Send only the payment ID and
   client secret to the signed-in shopper, then call
   `handleNextAction({ clientSecret })` from `@tapayadot/checkout/client`.
5. **Confirm on the server.** `tapaya.payments.get(id)`; fulfill only on
   `status: 'successful'` after comparing order, amount, and currency.
   `authorized` is an uncaptured hold.

## 4. Declines, holds, refunds

- Retry a declined merchant-initiated charge only while `retryAllowed` is
  true, with a **new** idempotency key; at most five declined attempts per
  order. Follow the `declineReason` table on the docs page
  (`insufficient_funds`, `card_expired`, `authentication_required`,
  `do_not_retry`, and others; treat unknown values like `card_declined`).
- Holds: `captureMethod: 'manual'`, then `payments.increment()`,
  `payments.capture()` (runs once; partial capture releases the rest), or
  `payments.cancel()` before capture. Capture before `authorizationExpiresAt`.
- Refunds: `payments.refunds.create(id, { amount, reason })`, partial and
  repeated up to `amountCaptured`. For a Hosted Checkout payment, `id` is the
  session's `paymentReference`. Read refunds until `succeeded` or `failed`.
- Use a separate idempotency key for every create, increment, capture,
  cancel, and refund.

## 5. Wallets and management

- Apple Pay saved for later charges needs a `walletAgreement`
  (`recurring` or `automatic_reload`) on the server and as
  `cardSaving.walletAgreement` in the browser; without it Apple Pay is hidden.
  Saved wallet methods support merchant-initiated charges only.
- `paymentMethods.list({ customerReference })` (paged by `nextCursor`),
  `get()`, `update()` (expiry, cardholder name, billing address; wallets
  return `409`), `delete()`. Always check the method belongs to the
  signed-in customer and merchant.

## 6. Test in the sandbox

Use `4242 4242 4242 4242` with any future expiry and CVC `123`, and
`4012 0018 0000 0016` for 3-D Secure. A €0 hosted session shows a **Save
card** button instead of **Pay**. Give each test a new `merchantOrderId`.
The Testing page also lists saved-card, manual capture, and refund cases to
cover. To see a working base first, use the `demo-hosted-checkout` or
`demo-embedded-checkout` skill and add the saving fields.

## Gaps in the docs

- No sandbox card or amount is documented that forces a specific
  `declineReason` on a merchant-initiated charge, so decline handling cannot
  be tested deterministically from the docs alone.
- Wallet saving and delivery must be tested on real devices; the Google Pay
  TEST sheet may record a simulated payment.

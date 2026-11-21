---
name: go-live-check
description: Review a Tapaya integration against the official QA checklists before launch and report pass, fail, unknown, or not applicable for every item, with evidence and docs links. Covers Online Checkout (hosted, embedded, saved cards) and In-Store Acceptance (Accept SDK), plus Checkout security, device security, webhooks, and monitoring. Use when the user asks whether their Tapaya integration is ready for production, for a go-live or launch review, a QA checklist pass, or a pre-release security check.
---

# Tapaya go-live check

Review the user's Tapaya integration against the current QA checklists and
produce an item-by-item result: **Pass**, **Fail**, **Unknown**, or **N/A**,
each with evidence and a docs link. This is a review, not a fix: list the
changes, and make them only if the user asks.

Sources:

- Online QA Checklist: https://docs.tapaya.com/online/getting-started/checklist
- In-Store QA Checklist: https://docs.tapaya.com/in-store/getting-started/checklist
- Online Testing: https://docs.tapaya.com/online/getting-started/testing
- In-Store Testing / Sandbox: https://docs.tapaya.com/in-store/getting-started/testing
- Checkout Security: https://docs.tapaya.com/online/security/checkout_security
- Device Security: https://docs.tapaya.com/in-store/security/device_security
- Monitoring: https://docs.tapaya.com/support/monitoring

## 1. Scope the review

Ask, or work out from the code:

- Which products: Online Checkout (hosted, embedded, saved cards, delivery),
  In-Store Acceptance (Accept SDK on Android or React Native, Tapaya Terminal
  app only, Cloud Accept API), or both.
- Where the code is: server, web front end, mobile app.
- What the user has already tested in the sandbox, and the results.
- Platform state you cannot see: merchant **Ready** in Checkout settings,
  return URLs, wallets, Apple Pay domains, webhook destination.

## 2. Fetch the live checklists

Always read the current lists before reviewing; items change. With the
`tapaya-docs` MCP tools:

- `get_page` `/online/getting-started/checklist` and/or
  `/in-store/getting-started/checklist`.
- `get_page` `/online/security/checkout_security` and/or
  `/in-store/security/device_security`.
- `get_page` the matching Testing page for the test cards and sandbox
  amounts each item refers to.
- `get_page` `/support/monitoring` and `/webhooks/receiving` for the
  operations section.

Use the item IDs and wording from the live pages. The snapshot in
[references/checklist-guide.md](references/checklist-guide.md) only helps
you organize the output.

## 3. Review

For each item, collect evidence:

- **Code items:** search the code. Examples: the key is read from server
  configuration only and never reaches browser or app bundles (SE-1);
  amount, currency, and order ID come from the stored order (SE-2); only
  `paymentStatus === 'successful'` (hosted) or `status === 'successful'`
  (payments) counts as paid, and unknown values count as not paid (PO-5,
  VF-1); idempotency keys are persisted with the order and reused on retry
  (RI-1, RI-2); a scheduled job retrieves unresolved sessions (Online PR-3,
  LV-4); in-store, the SDK token comes from the backend each session
  (DH-1), logs hold no card data (DH-2), and `isProduction` follows the
  build type (In-store PR-1).
- **Config items:** read environment variable *names* and config files, never
  print secret values. Check the API base URL (`https://api.sandbox.tapaya.com`
  for testing, `https://api.tapaya.com` for live), `TAPAYA_ENVIRONMENT`, and
  the key prefix (`tapaya_test_` vs `tapaya_live_`) without echoing the key.
  Ask the user to confirm Platform settings.
- **Manual tests:** ask the user to run them with the sandbox test cards
  (online) or sandbox amounts (in-store), and to report what they saw.
  Until they do, the item is Unknown.

Also check the security pages beyond the numbered items: device security's
production table (non-debuggable, no debug logging, release keystore, R8 on,
no user CA trust, USB debugging off), webhook signature verification and
deduplication, and whether someone watches https://status.tapaya.com.

## 4. Write the result

Use the layout in
[references/checklist-guide.md](references/checklist-guide.md): a summary
with counts and blockers, then one table per checklist with ID, item,
status, evidence or next action, and the docs URL. Prefix the two `PR-*`
lists (`Online PR-1`, `In-store PR-1`). Keep evidence concrete: a file and
line, a config name, or the user's reported result.

Rules:

- Never mark Pass without evidence.
- Never call the live Tapaya API, and never run payments against production.
- Never print or copy a secret. If you find a key in source control or
  client code, mark SE-1 (or DH-1) as Fail, tell the user to revoke and
  replace it (https://docs.tapaya.com/online/security/checkout_security#keep-keys-on-the-server),
  and do not repeat the key.
- Say when a checklist item depends on something the docs leave open.

## Gaps in the docs

- Online PO-4 asks for a processing-error test card, but the Testing page
  lists only success, decline, and 3D Secure cards. Mark PO-4 Unknown and
  suggest asking Tapaya support for the card.
- The in-store checklist (HI-1) says to test with physical cards and wallets
  but names no sandbox test cards; outcomes follow the sandbox amounts.
- Monitoring documents the status page and its email or webhook
  subscriptions only. No status API or health endpoint is documented for
  checking from code.

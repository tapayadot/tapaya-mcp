# Go-live check: statuses, evidence, and output layout

Reference for the `go-live-check` skill. Replace the item lists with the
**current** checklist items from the docs (`get_page`
`/online/getting-started/checklist` and `/in-store/getting-started/checklist`).
The item IDs and titles below are a snapshot taken when the skill was
written; if the live checklist adds, removes, or renames items, follow the
live one and say so.

## Status values

| Status | Use when |
| --- | --- |
| **Pass** | You saw evidence: the code path, config, test output, or the user's statement that they ran the test and what happened. Cite the file and line or the user's words. |
| **Fail** | The code or config contradicts the item, or a test the user ran gave the wrong result. Say what to change and cite the docs. |
| **Unknown** | You cannot decide: the item needs a manual test, a Platform setting, or code you cannot see. Say exactly what to run or check. |
| **N/A** | The item does not apply, such as embedded-only items in a hosted-only integration, or optional NFC tag items. Say why. |

Never mark an item Pass because it "looks likely". Manual tests the user has
not reported are Unknown.

## How each kind of item is checked

| Kind | Evidence | Examples (online / in-store IDs) |
| --- | --- | --- |
| Code | Read the code: where keys load, where totals come from, which status counts as paid, idempotency key storage, retries, the reconciliation job | SE-1, SE-2, SE-3, PO-5, VF-1, RI-1, RI-2, RI-4, Online PR-3, LV-4, DH-1, DH-2 |
| Config | Ask the user, or read config files and environment variable names (never print secret values) | Online PR-1, PR-2, LV-1, LV-2, CS-1 to CS-6, SE-4, In-store PR-1 to PR-3 |
| Manual test | Ask the user to run it and report the result | PO-1 to PO-4, VF-2 to VF-4, RI-3, RI-5, RI-6, TF-1 to TF-7, HI-1 to HI-3, EC-1 to EC-3, MS-1, MS-2, DI-1, DI-2 |

## Output layout

```markdown
# Tapaya go-live check: <project name>

Scope: <Online Checkout: hosted / embedded / saved cards> and/or <In-Store: Accept SDK Android / React Native>
Checklists: https://docs.tapaya.com/online/getting-started/checklist (fetched <date>),
https://docs.tapaya.com/in-store/getting-started/checklist (fetched <date>)

## Summary

- Pass: N  Fail: N  Unknown: N  N/A: N
- Blockers (Fail on a non-optional item): <IDs>
- Next steps, in order: <short list>

## Online Checkout

| ID | Item | Status | Evidence or what to do | Docs |
| --- | --- | --- | --- | --- |
| PO-1 | Successful payment | Unknown | Pay with the success test card and report the retrieved `paymentStatus`. | /online/getting-started/testing |
| SE-1 | Key on the server | Pass | `server/config.ts:12` reads `TAPAYA_SECRET_KEY` from env; no key under `web/`. | /online/security/checkout_security |

## In-Store (Accept SDK)

| ID | Item | Status | Evidence or what to do | Docs |
| --- | --- | --- | --- | --- |
| PR-1 | Environment switch | Fail | `App.kt:9` hard-codes `isProduction = false`; drive it from the build type. | /in-store/security/device_security |

## Security and operations

| Topic | Status | Evidence or what to do | Docs |
| --- | --- | --- | --- |
| Checkout security rules | | | /online/security/checkout_security |
| Device security production config | | | /in-store/security/device_security |
| Webhooks verified and deduplicated | | | /webhooks/receiving |
| Status page subscription and incident plan | | | /support/monitoring |

## Gaps

Items the docs do not fully specify, and what to ask Tapaya support
(https://docs.tapaya.com/support/contacts).
```

## Snapshot of checklist IDs

Online (`/online/getting-started/checklist`):

- 1 Payment outcomes: PO-1 Successful payment, PO-2 Declined card, PO-3 3D Secure, PO-4 Processing error, PO-5 Unhandled states
- 2 Verification and fulfillment: VF-1 Server-side check, VF-2 Unpaid success URL, VF-3 Cancel link, VF-4 Repeated return
- 3 Retries and interruptions: RI-1 Lost create response, RI-2 Changed body, RI-3 Customer never returns, RI-4 Status timeout, RI-5 Session expiry, RI-6 Replaced session
- 4 Checkout settings: CS-1 Merchant approved, CS-2 Return destinations, CS-3 Per-session overrides, CS-4 Wallets, CS-5 Branding, CS-6 Embedded wallets
- 5 Security: SE-1 Key on the server, SE-2 Server-owned totals, SE-3 Embedded endpoints, SE-4 Content security policy
- 6 Sandbox configuration: PR-1 Sandbox API, PR-2 Sandbox key, PR-3 Reconciliation job
- 7 Production readiness: LV-1 Merchant readiness, LV-2 Live credentials, LV-3 Live wallets, LV-4 Live reconciliation
- 8 Saved methods and delivery: SD-1 to SD-10 (saving consent, saved-card verification, merchant-initiated charges, authorization lifecycle, zero-amount setup, customer collection, shipping approval, hosted callback, wallets on devices, refund reconciliation)

In-store (`/in-store/getting-started/checklist`):

- 1 Functional testing: TF-1 to TF-7 (sandbox amounts 1.01, 1.07, 6.05, 6.06, 8.06, a cancelled payment, receipts), HI-1 NFC read, HI-2 Interrupted read, HI-3 Network loss
- 2 Onboarding and configuration: MS-1 New user onboarding, MS-2 Status check
- 3 Security and compliance: DI-1 Root detection (optional), DI-2 Permissions, DH-1 Token management, DH-2 No sensitive data
- 4 Edge cases: EC-1 App backgrounding, EC-2 Incoming call, EC-3 Low battery
- 5 NFC tag sessions (optional): NT-1 to NT-8
- 6 Production readiness: PR-1 Environment switch, PR-2 Production tokens, PR-3 Version check

Both lists use `PR-*` IDs for different items. Prefix them in the output
(`Online PR-1`, `In-store PR-1`).

---
name: demo-tap-to-pay
description: Build a minimal, runnable Tapaya Tap to Pay (In-Store Acceptance) demo with the Accept SDK - an Android app that signs a sandbox merchant in, activates the Tapaya Terminal app, and takes a contactless card payment, plus a tiny Node.js backend that requests the SDK login token from the real Tapaya sandbox API. Use when the user asks for a Tap to Pay, Accept SDK, SoftPOS, or in-store payment demo, sample, or proof of concept with Tapaya.
---

# Tapaya Tap to Pay demo

Build the smallest working Tap to Pay demo with the Tapaya Accept SDK:

- a **token server** (Node.js, no dependencies) that holds the Server Secret
  Token and calls `POST /merchant/auth/login` on the Tapaya sandbox;
- an **Android app** (Kotlin) that initializes the SDK in sandbox, signs in
  with that token, installs and activates the Tapaya Terminal app, and calls
  `Accept.payments.pay()`.

Every call is real and goes to the sandbox, where no real money moves.
The Server Secret Token never reaches the phone. The phone only gets a
short-lived login token for one merchant.

Main docs pages:

- Quick Start: https://docs.tapaya.com/in-store/getting-started/quick_start_guide
- Mobile SDK Integration: https://docs.tapaya.com/in-store/integration/sdk_integration
- Android SDK reference: https://docs.tapaya.com/in-store/integration/android_sdk
- Backend Integration (login token): https://docs.tapaya.com/in-store/integration/backend_integration
- Tapaya Terminal App (activation, outcomes): https://docs.tapaya.com/in-store/integration/card_terminal
- Testing / Sandbox: https://docs.tapaya.com/in-store/getting-started/testing
- Supported Devices: https://docs.tapaya.com/in-store/getting-started/supported_devices

## 1. Confirm what you need

Ask the user, briefly, before writing files:

1. **Stack.** The Accept SDK ships for **Android (Kotlin)** and **React
   Native / Expo** (`@tapayadot/accept-react-native`, a wrapper over the
   Android SDK). Both run on Android only. The templates are Kotlin. For React
   Native, read the TypeScript tabs of the same pages and translate the app;
   the token server stays the same. If the user asks for iOS or Tap to Pay on
   iPhone, say the docs do not document an iOS SDK and stop there.
2. **A physical Android device.** Android 11 or newer, NFC, not rooted,
   locked bootloader, Google Play services (GMS-free needs approval from
   Tapaya). Emulators never work, not even in sandbox. Check the model
   against https://docs.tapaya.com/in-store/getting-started/supported_devices.
3. **A sandbox Server Secret Token.** Sign up or sign in at
   https://platform.tapaya.com, work in a sandbox organization, and create a
   **Secret key** on the API Keys page (https://platform.tapaya.com/api-keys/,
   **Settings > APIs > Create key**). See
   https://docs.tapaya.com/platform-api#authentication.
4. **A merchant.** A `merchantToken` the user registered with
   `POST /merchant/auth/register` and onboarded, or a new one. The setup
   steps are in the token server reference. In sandbox, merchants are
   approved automatically after onboarding.
5. **Node.js 22 or newer** for the token server, and Android Studio with
   `adb` for the app.

Never ask the user to paste the Secret key into the chat; they put it in
`backend/.env`. If the API URL is `https://api.tapaya.com`, stop: the demo is
sandbox only.

## 2. Check the docs before writing code

The plugin ships the `tapaya-docs` MCP server. Confirm current details with it
before writing code:

- `get_page` `/in-store/integration/sdk_integration` and
  `/in-store/integration/android_sdk`: Maven coordinates, `Accept.initialize`,
  `Accept.auth.authenticate`, `Accept.payments.pay`, `PaymentEvent` and
  `PayResult` variants, error names.
- `get_page` `/in-store/integration/card_terminal`: `Accept.plugin.isInstalled()`,
  `install()`, `activateTerminal()` outcomes, and `Accept.payments.status()`.
- `get_api_endpoint` with `POST` `/merchant/auth/login` (and
  `/merchant/auth/register`): request and response fields.
- `get_page` `/in-store/getting-started/testing`: the sandbox outcome amounts.
- `search_docs` for anything else, such as `"location permission"` or
  `"activation code"`.

If the docs differ from the templates, follow the docs and tell the user.

## 3. Scaffold the files

Create `tapaya-tap-to-pay-demo/` with two parts:

- `backend/` from [references/backend-token-server.md](references/backend-token-server.md):
  `.env.example` with the secret lines commented out, `.gitignore`,
  `package.json`, and `server.mjs` exposing `POST /sdk-token`. It sends the
  Secret key as-is in `Authorization` (no `Bearer`) with
  `{ merchantToken, allowOnboarding: true }` and returns `{ token }`. It
  listens on `127.0.0.1` only and refuses any API URL other than the sandbox.

  ```bash
  TAPAYA_API_URL=https://api.sandbox.tapaya.com
  # TAPAYA_SERVER_SECRET_TOKEN=xxx_paste_your_sandbox_secret_key
  # TAPAYA_MERCHANT_TOKEN=demo-merchant-1
  ```

- `android/` from [references/android-app.md](references/android-app.md):
  add `com.tapaya:accept` (latest on Maven Central) to an Empty Views
  Activity project, the `INTERNET` and `ACCESS_FINE_LOCATION` permissions, a
  debug-only network security config for `http://localhost`, `DemoApp`
  (calls `Accept.initialize(context = this, isProduction = false)`), and
  `MainActivity` with buttons for sign in, install/activate, and two test
  payments.

Do not invent `import` lines for SDK classes; the docs do not show them. Let
the IDE resolve them after Gradle sync.

## 4. Run and test

Tell the user:

1. Do the one-time sandbox setup in the backend reference: register the
   merchant, then onboard it (Platform UI or hosted invite). Request in-store
   payments.
2. Start the token server and forward its port to the phone over USB:

   ```bash
   cd tapaya-tap-to-pay-demo/backend
   cp .env.example .env   # uncomment and fill in the two secret lines
   npm start
   adb reverse tcp:3000 tcp:3000
   ```

3. Run the app on the phone from Android Studio (debug build, sandbox).
4. Tap **Sign in merchant**, then **Install / activate Tapaya Terminal**. The
   first time, this opens the store to install Tapaya Terminal; tap it again
   to activate. Tapaya Terminal asks for its activation code itself.
5. Tap **Pay 1.01** and hold a contactless card to the phone when Tapaya
   Terminal asks.

Sandbox outcomes are driven by the amount, not by special cards
(https://docs.tapaya.com/in-store/getting-started/testing#simulate-specific-card-outcomes):

| Amount | Pass to `pay()` | Response code | Scenario |
| --- | --- | --- | --- |
| 1.01 | `101` | `00` | Approved |
| 1.07 | `107` | `06` | Card authentication rejected |
| 6.05 | `605` | `54` | Expired card |
| 6.06 | `606` | `55` | Incorrect PIN |
| 8.06 | `806` | `75` | PIN tries exceeded |

A high amount, above the contactless verification limit, triggers on-glass
PIN entry. Re-check this table with `get_page` before quoting it.

## 5. Expected result and common errors

Expected: the log shows `Signed in` with a ready onboarding status and the
merchant's currencies, then `Terminal activated`. Paying 1.01 shows
`Creating payment…`, `Created <token>`, `Tap a card…`, then a
`PayResult.Success` and `Status: SUCCESS` with a receipt URL. Paying 1.07
ends in a declined or failed result, not an exception.

| Symptom | Likely cause and fix |
| --- | --- |
| SDK does not initialize | Emulator, rooted device, unlocked bootloader, Android below 11, no NFC, or an old security patch. Use a supported physical phone. |
| Token server error in the app | Server not running, or `adb reverse tcp:3000 tcp:3000` not run. Cleartext blocked: check the debug network security config. |
| `Tapaya login failed (401)` in the server log | Wrong or production Secret key, or a `Bearer` prefix was added. Use the sandbox key as-is. |
| `Tapaya login failed (404)` | Unknown `merchantToken`. Register it first. |
| `MerchantOnboardingIncomplete` on activation | Finish onboarding with in-store payments. Sandbox approves automatically after submission. |
| `PluginUnavailable` | Tapaya Terminal is not installed. Use the install button; on devices without a store, add `com.tapaya:accept-installer`. |
| `LocationPermissionRequired`, `LocationUnavailable`, `LocationTimeout` | Grant location and turn on location services. A device that never had a fix waits up to 30 seconds. |
| `CurrencyNotAvailableForMerchant` or `AmountBelowMinimum` | Use a currency from `Accept.merchant.availableCurrencies()` and an amount at or above `Accept.sdk.minimumAmounts()`. |
| `SessionExpired` | Sign in again with a fresh token. |
| `PluginMerchantMismatch` | Tapaya Terminal is linked to another merchant. Call `Accept.plugin.logout()`, then activate again. |

Quote `Accept.sdk.deviceId()` when contacting Tapaya support
(https://docs.tapaya.com/support/contacts).

## What the docs do not cover

Say so instead of guessing:

- **iOS / Tap to Pay on iPhone.** No iOS SDK is documented (the login
  endpoint's `employeeEmail` field mentions Tap to Pay on iPhone, but nothing
  else does).
- **Test cards.** The in-store docs name no sandbox test cards. They say
  sandbox still needs a physical NFC device, only funds movement is
  simulated, and the outcome follows the amount; the QA checklist (HI-1)
  says to test with physical cards and digital wallets. Whether any real card
  works in sandbox is not stated.
- **Build setup.** No `minSdk`, Gradle, AGP, or Kotlin version is documented,
  and no Kotlin `import` paths for SDK classes.
- **Debug builds.** Supported Devices lists "USB debugging off,
  non-debuggable build" as a requirement, while Device Security lists a
  debuggable build with USB debugging on as the development setup. If a
  debug build will not initialize, ask Tapaya support rather than bypassing
  the checks.
- **Sandbox organization.** The docs do not describe how to create or select
  one in the Platform; they only give the sandbox API URL.

# Tap to Pay demo: Android app (Kotlin)

The app half of the `demo-tap-to-pay` skill. Check every call against the live
docs with the `tapaya-docs` MCP tools first. The docs win if they differ from
this file.

Sources:

- https://docs.tapaya.com/in-store/integration/sdk_integration (install, location permission, initialize, authenticate, pay)
- https://docs.tapaya.com/in-store/integration/android_sdk (every API)
- https://docs.tapaya.com/in-store/integration/card_terminal (install and activate Tapaya Terminal, payment outcome)
- https://docs.tapaya.com/in-store/security/device_security#deploying-in-a-production-ready-configuration
- https://docs.tapaya.com/in-store/getting-started/testing (sandbox amounts)

## Start from a plain Android project

Use the user's existing app, or create a new project in Android Studio with
the **Empty Views Activity** template (Kotlin, AppCompat). The docs do not
state a `minSdk`, Gradle, AGP, or Kotlin version for the SDK; keep the
template's defaults and let Gradle report any conflict from the AAR. Devices
must run Android 11 or higher.

## Imports

The docs show the Kotlin calls but not their `import` lines (only the NFC
add-on's `com.tapaya.accept.nfc.nfc` is shown). Do not guess package names.
After adding the dependency and syncing, let Android Studio resolve `Accept`,
`AcceptException`, `PaymentEvent`, `PayResult`, `ActivateTerminalResult`, and
`SdkState` (Alt+Enter / Option+Return), or read them from the SDK sources Gradle
downloads.

## app/build.gradle.kts (add to the existing file)

Set `acceptVersion` to the latest release on Maven Central
(https://central.sonatype.com/artifact/com.tapaya/accept). Keep every Accept
artifact on the same version.

```kotlin
val acceptVersion = "x.y.z" // latest com.tapaya:accept on Maven Central

dependencies {
    implementation("com.tapaya:accept:$acceptVersion")
    // Optional, only for devices without an app store:
    // implementation("com.tapaya:accept-installer:$acceptVersion")
}
```

## app/src/main/AndroidManifest.xml (additions)

Card payments need `ACCESS_FINE_LOCATION`, which the SDK does not request for
you. `INTERNET` is needed to reach the token server. Point `android:name` at
the `Application` class below.

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<application
    android:name=".DemoApp"
    android:networkSecurityConfig="@xml/network_security_config"
    ... >
```

## app/src/debug/res/xml/network_security_config.xml

The demo talks to the token server over plain HTTP on `localhost` (through
`adb reverse`). Android blocks cleartext HTTP by default, so allow it for
`localhost` in the **debug** source set only. This is a standard Android
setting, not a Tapaya one. Release builds must not trust cleartext or user CAs
(see the device security page).

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">localhost</domain>
    </domain-config>
</network-security-config>
```

Add an empty-rule file with the same name under `app/src/main/res/xml/` so the
release build also resolves `@xml/network_security_config`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config />
```

## DemoApp.kt

```kotlin
package com.example.taptopaydemo

import android.app.Application

class DemoApp : Application() {
    override fun onCreate() {
        super.onCreate()
        // Sandbox is the default. This demo must never run against production.
        Accept.initialize(context = this, isProduction = false)
    }
}
```

## MainActivity.kt

One screen, built in code, with four buttons and a log. Payment amounts come
from the sandbox table on the Testing page (`101` = 1.01 approved,
`107` = 1.07 card authentication rejected).

```kotlin
package com.example.taptopaydemo

import android.Manifest
import android.os.Bundle
import android.widget.Button
import android.widget.LinearLayout
import android.widget.ScrollView
import android.widget.TextView
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import org.json.JSONObject
import java.net.HttpURLConnection
import java.net.URL

// Reached on the phone through `adb reverse tcp:3000 tcp:3000`.
private const val TOKEN_URL = "http://localhost:3000/sdk-token"

class MainActivity : AppCompatActivity() {
    private lateinit var output: TextView
    private var pendingAmount: Long? = null

    private val requestLocation = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        val amount = pendingAmount
        pendingAmount = null
        if (granted && amount != null) pay(amount) else log("Location permission is required for card payments")
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        output = TextView(this).apply { setTextIsSelectable(true) }
        val column = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            setPadding(32, 32, 32, 32)
            addView(button("1. Sign in merchant") { signIn() })
            addView(button("2. Install / activate Tapaya Terminal") { activate() })
            addView(button("3. Pay 1.01 (approved)") { startPayment(101) })
            addView(button("Pay 1.07 (rejected)") { startPayment(107) })
            addView(output)
        }
        setContentView(ScrollView(this).apply { addView(column) })
        log("SDK ${Accept.sdk.version}, production=${Accept.sdk.isProduction}")
    }

    private fun button(label: String, onClick: () -> Unit) =
        Button(this).apply { text = label; setOnClickListener { onClick() } }

    private fun log(line: String) {
        runOnUiThread { output.append("$line\n") }
    }

    private fun signIn() = lifecycleScope.launch {
        try {
            val token = withContext(Dispatchers.IO) { fetchSdkToken() }
            Accept.auth.authenticate(token)
            log("Signed in. Onboarding: ${Accept.merchant.onboardingStatus()}")
            log("Currencies: ${Accept.merchant.availableCurrencies()}")
        } catch (e: AcceptException) {
            log("Sign-in failed: $e")
        } catch (e: Exception) {
            log("Token server error: ${e.message}. Is it running, and did you run adb reverse?")
        }
    }

    private fun activate() = lifecycleScope.launch {
        try {
            if (!Accept.plugin.isInstalled()) {
                log("Tapaya Terminal is missing. Opening install…")
                Accept.plugin.install()
                return@launch
            }
            when (val result = Accept.plugin.activateTerminal()) {
                ActivateTerminalResult.Success -> log("Terminal activated")
                ActivateTerminalResult.Canceled -> log("Activation canceled")
                is ActivateTerminalResult.Failed -> log("Activation failed: ${result.reason}")
            }
        } catch (e: AcceptException) {
            log("Activation error: $e") // e.g. PluginUnavailable, MerchantOnboardingIncomplete
        }
    }

    private fun startPayment(amount: Long) {
        pendingAmount = amount
        requestLocation.launch(Manifest.permission.ACCESS_FINE_LOCATION)
    }

    private fun pay(amount: Long) = lifecycleScope.launch {
        val currency = try {
            Accept.merchant.availableCurrencies().first()
        } catch (e: Exception) {
            log("Sign in first: $e"); return@launch
        }
        log("Paying $amount (minor units) $currency…")
        Accept.payments.pay(amount = amount, currency = currency).collect { event ->
            when (event) {
                PaymentEvent.Creating -> log("Creating payment…")
                is PaymentEvent.Created -> log("Created ${event.paymentToken}")
                PaymentEvent.Launched -> log("Tap a card on the phone…")
                is PaymentEvent.Result -> handleResult(event.payResult)
                is PaymentEvent.CreationFailed -> log("Not started: ${event.cause}")
            }
        }
    }

    private suspend fun handleResult(result: PayResult) {
        log("Result: $result")
        if (result is PayResult.Success) {
            // Always confirm the final state with the backend before treating it as paid.
            try {
                val status = Accept.payments.status(result.paymentToken)
                log("Status: ${status.state}, receipt: ${status.receiptUrl}")
            } catch (e: AcceptException) {
                log("Status check failed: $e")
            }
        }
    }

    private fun fetchSdkToken(): String {
        val connection = URL(TOKEN_URL).openConnection() as HttpURLConnection
        try {
            connection.requestMethod = "POST"
            connection.connectTimeout = 10_000
            connection.readTimeout = 20_000
            val code = connection.responseCode
            val body = (if (code in 200..299) connection.inputStream else connection.errorStream)
                ?.bufferedReader()?.use { it.readText() }.orEmpty()
            check(code in 200..299) { "HTTP $code $body" }
            return JSONObject(body).getString("token")
        } finally {
            connection.disconnect()
        }
    }
}
```

## Notes on the template

- `pay()` never throws. Failures before the terminal opens arrive as
  `PaymentEvent.CreationFailed`; declines and cancels arrive as
  `PaymentEvent.Result`.
- The result variant names (`PayResult.Success`, `Declined`, `Canceled`,
  `Failed`) and `Accept.payments.status()` fields (`state`, `receiptUrl`) come
  from the Card Terminal page. Recheck them with `get_page` if the build fails.
- A real app requests location permission with a rationale, observes
  `Accept.state`, signs out with `Accept.plugin.logout()` and `Accept.clear()`,
  and drives `isProduction` from the build type. The demo hard-codes sandbox.

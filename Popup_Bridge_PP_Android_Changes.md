# Popup Bridge++ Android Implementation — Full Change Details

This document captures all code changes required to add `isPayPalInstalled` and `launchApp(url)` to popup-bridge-android, mirroring the popup-bridge-ios implementation. These changes enable the JS SDK to detect the PayPal app and launch it via universal link for app switch checkout from a WebView.

---

## 1. `AppInstalledChecks.kt` — Add PayPal app detection

**Path:** `PopupBridge/src/main/java/com/braintreepayments/api/internal/AppInstalledChecks.kt`

**Full file after changes:**

```kotlin
package com.braintreepayments.api.internal

import android.content.Context

private const val VENMO_APP_PACKAGE = "com.venmo"
private const val PAYPAL_APP_PACKAGE = "com.paypal.android.p2pmobile"

fun Context.isVenmoInstalled(): Boolean = AppHelper().isAppInstalled(this, VENMO_APP_PACKAGE)
fun Context.isPayPalInstalled(): Boolean = AppHelper().isAppInstalled(this, PAYPAL_APP_PACKAGE)
```

**What changed:**
- Added `PAYPAL_APP_PACKAGE` constant with value `"com.paypal.android.p2pmobile"`
- Added `isPayPalInstalled()` extension function following the same pattern as `isVenmoInstalled()`

---

## 2. `PopupBridgeJavascriptInterface.kt` — Add `launchApp` JS interface method

**Path:** `PopupBridge/src/main/java/com/braintreepayments/api/internal/PopupBridgeJavascriptInterface.kt`

**Full file after changes:**

```kotlin
package com.braintreepayments.api.internal

import android.webkit.JavascriptInterface

/**
 * `PopupBridgeJavascriptInterface` is a class that provides JavaScript interface methods to a [android.webkit.WebView].
 *
 * The [android.webkit.WebView] should call [android.webkit.WebView.addJavascriptInterface], passing this class as the
 * first argument.
 */
internal class PopupBridgeJavascriptInterface(
    private val returnUrlScheme: String,
) {

    var onOpen: ((url: String?) -> Unit)? = null
    var onLaunchApp: ((url: String?) -> Unit)? = null
    var onSendMessage: ((messageName: String?, data: String?) -> Unit)? = null

    @get:JavascriptInterface
    val returnUrlPrefix: String
        get() = String.format(
            "%s://%s/",
            returnUrlScheme,
            POPUP_BRIDGE_URL_HOST
        )

    @JavascriptInterface
    fun open(url: String?) {
        onOpen?.invoke(url)
    }

    @JavascriptInterface
    fun launchApp(url: String?) {
        onLaunchApp?.invoke(url)
    }

    @JavascriptInterface
    fun sendMessage(messageName: String?) {
        onSendMessage?.invoke(messageName, null)
    }

    @JavascriptInterface
    fun sendMessage(messageName: String?, data: String?) {
        onSendMessage?.invoke(messageName, data)
    }

    companion object {
        const val POPUP_BRIDGE_URL_HOST: String = "popupbridgev1"
    }
}
```

**What changed:**
- Added `var onLaunchApp: ((url: String?) -> Unit)? = null` callback property (same pattern as `onOpen`)
- Added `@JavascriptInterface fun launchApp(url: String?)` method that invokes `onLaunchApp`

---

## 3. `PopupBridgeWebViewClient.kt` — Inject `isPayPalInstalled` into JS

**Path:** `PopupBridge/src/main/java/com/braintreepayments/api/PopupBridgeWebViewClient.kt`

**Diff summary (only changed sections):**

**Import added:**
```kotlin
import com.braintreepayments.api.internal.isPayPalInstalled
```

**`onPageFinished()` — added `setPayPalInstalled()` call:**
```kotlin
override fun onPageFinished(view: WebView?, url: String?) {
    super.onPageFinished(view, url)
    setVenmoInstalled(view, view?.context?.isVenmoInstalled() == true)
    setPayPalInstalled(view, view?.context?.isPayPalInstalled() == true)  // NEW
    delegate?.onPageFinished(view, url)
}
```

**New `setPayPalInstalled()` private method (added before `runJavaScriptInWebView`):**
```kotlin
private fun setPayPalInstalled(view: WebView?, isPayPalInstalled: Boolean) {
    runJavaScriptInWebView(view,
        ""
            + "function setPayPalInstalled() {"
            + "    window.popupBridge.isPayPalInstalled = ${isPayPalInstalled};"
            + "}"
            + ""
            + "if (document.readyState === 'complete') {"
            + "  setPayPalInstalled();"
            + "} else {"
            + "  window.addEventListener('load', function () {"
            + "    setPayPalInstalled();"
            + "  });"
            + "}"
    )
}
```

---

## 4. `PopupBridgeClient.kt` — Wire `launchApp` to launch native app via Intent

**Path:** `PopupBridge/src/main/java/com/braintreepayments/api/PopupBridgeClient.kt`

**Diff summary (only changed sections):**

**Imports added:**
```kotlin
import com.braintreepayments.api.PopupBridgeAnalytics.POPUP_BRIDGE_APP_LAUNCHED
import com.braintreepayments.api.PopupBridgeAnalytics.POPUP_BRIDGE_APP_LAUNCH_FAILED
```

**`init` block — wired `onLaunchApp` callback:**
```kotlin
with(popupBridgeJavascriptInterface) {
    onOpen = { url -> openUrl(url) }
    onLaunchApp = { url -> launchApp(url) }  // NEW
    onSendMessage = { messageName, data ->
        messageListener?.onMessageReceived(messageName, data)
    }
}
```

**New `launchApp()` private method (added before `openUrl`):**
```kotlin
private fun launchApp(url: String?) {
    analyticsClient.sendEvent(PopupBridgeAnalytics.POPUP_BRIDGE_STARTED)
    val activity = activityRef.get() ?: return
    try {
        val intent = Intent(Intent.ACTION_VIEW, Uri.parse(url))
        activity.startActivity(intent)
        analyticsClient.sendEvent(POPUP_BRIDGE_APP_LAUNCHED)
    } catch (e: Exception) {
        analyticsClient.sendEvent(POPUP_BRIDGE_APP_LAUNCH_FAILED)
        runErrorJavaScript("new Error('Failed to launch app. ${e.localizedMessage}')")
    }
}
```

---

## 5. `PopupBridgeAnalytics.kt` — Add app switch analytics events

**Path:** `PopupBridge/src/main/java/com/braintreepayments/api/PopupBridgeAnalytics.kt`

**Full file after changes:**

```kotlin
package com.braintreepayments.api

internal object PopupBridgeAnalytics {
    const val POPUP_BRIDGE_STARTED = "popup-bridge:started"
    const val POPUP_BRIDGE_SUCCEEDED = "popup-bridge:succeeded"
    const val POPUP_BRIDGE_FAILED = "popup-bridge:failed"
    const val POPUP_BRIDGE_CANCELED = "popup-bridge:canceled"
    const val POPUP_BRIDGE_APP_DETECTED = "popup-bridge:app-switch:app-detected"
    const val POPUP_BRIDGE_APP_LAUNCHED = "popup-bridge:app-switch:app-launched"
    const val POPUP_BRIDGE_APP_LAUNCH_FAILED = "popup-bridge:app-switch:app-launch-failed"
    const val POPUP_BRIDGE_APP_SWITCH_RETURNED = "popup-bridge:app-switch:returned"
}
```

**What changed:**
- Added 4 new analytics event constants for app switch tracking

---

## 6. `PopupBridgeWebViewClientTest.kt` — Add isPayPalInstalled tests

**Path:** `PopupBridge/src/test/java/com/braintreepayments/api/PopupBridgeWebViewClientTest.kt`

**Import added:**
```kotlin
import com.braintreepayments.api.internal.isPayPalInstalled
```

**New tests added (after existing Venmo tests, before helper methods):**

```kotlin
@Test
fun `on page finished, when PayPal installed, isPayPalInstalled is set to true`() = runTest {
    every { webView.context.isVenmoInstalled() } returns false
    every { webView.context.isPayPalInstalled() } returns true

    sut.onPageFinished(webView, "https://example.com")

    verify {
        webView.evaluateJavascript(withArg { javascriptString ->
            assertEquals(getExpectedPayPalInstalledJavascript(true), javascriptString)
        }, null)
    }

    unmockkAll()
}

@Test
fun `on page finished, when PayPal is not installed, isPayPalInstalled is set to false`() = runTest {
    every { webView.context.isVenmoInstalled() } returns false
    every { webView.context.isPayPalInstalled() } returns false

    sut.onPageFinished(webView, "https://example.com")

    verify {
        webView.evaluateJavascript(withArg { javascriptString ->
            assertEquals(getExpectedPayPalInstalledJavascript(false), javascriptString)
        }, null)
    }

    unmockkAll()
}
```

**New helper method added:**

```kotlin
private fun getExpectedPayPalInstalledJavascript(isPayPalInstalled: Boolean): String {
    return String.format(
        (""
            + "function setPayPalInstalled() {"
            + "    window.popupBridge.isPayPalInstalled = %s;"
            + "}"
            + ""
            + "if (document.readyState === 'complete') {"
            + "  setPayPalInstalled();"
            + "} else {"
            + "  window.addEventListener('load', function () {"
            + "    setPayPalInstalled();"
            + "  });"
            + "}"), isPayPalInstalled
    )
}
```

---

## 7. `PopupBridgeClientUnitTest.kt` — Add launchApp tests

**Path:** `PopupBridge/src/test/java/com/braintreepayments/api/PopupBridgeClientUnitTest.kt`

**Imports added:**
```kotlin
import com.braintreepayments.api.PopupBridgeAnalytics.POPUP_BRIDGE_APP_LAUNCHED
import com.braintreepayments.api.PopupBridgeAnalytics.POPUP_BRIDGE_APP_LAUNCH_FAILED
```

**New slot added to class fields:**
```kotlin
private val onLaunchAppSlot = slot<(String?) -> Unit>()
```

**New mock capture added in `initializeClient()`:**
```kotlin
every { popupBridgeJavascriptInterface.onLaunchApp = capture(onLaunchAppSlot) } returns Unit
```

**New tests added (after `onUrlOpened` test, before `sendMessage` tests):**

```kotlin
@Test
fun `when onLaunchApp is called, activity startActivity is called with correct intent`() {
    initializeClient()

    val url = "https://www.paypal.com/app-switch-checkout?token=abc123"
    onLaunchAppSlot.captured.invoke(url)

    verify {
        activityMock.startActivity(withArg { intent ->
            assertEquals(Intent.ACTION_VIEW, intent.action)
            assertEquals(Uri.parse(url), intent.data)
        })
    }
}

@Test
fun `when onLaunchApp is called, POPUP_BRIDGE_STARTED and POPUP_BRIDGE_APP_LAUNCHED analytics events are sent`() {
    initializeClient()

    onLaunchAppSlot.captured.invoke("https://www.paypal.com/app-switch-checkout?token=abc123")

    verify { analyticsClient.sendEvent(POPUP_BRIDGE_STARTED) }
    verify { analyticsClient.sendEvent(POPUP_BRIDGE_APP_LAUNCHED) }
}

@Test
fun `when onLaunchApp fails, POPUP_BRIDGE_APP_LAUNCH_FAILED analytics event is sent and error javascript is run`() {
    every { activityMock.startActivity(any()) } throws android.content.ActivityNotFoundException("No activity found")
    initializeClient()

    onLaunchAppSlot.captured.invoke("https://www.paypal.com/app-switch-checkout?token=abc123")
    runnableSlot.captured.run()

    verify { analyticsClient.sendEvent(POPUP_BRIDGE_APP_LAUNCH_FAILED) }
    verify {
        webViewMock.evaluateJavascript(withArg { javascriptString ->
            assertTrue(javascriptString.contains("Failed to launch app"))
        }, null)
    }
}
```

---

## 8. `PopupBridgeJavascriptInterfaceUnitTest.kt` — Add launchApp test

**Path:** `PopupBridge/src/test/java/com/braintreepayments/api/internal/PopupBridgeJavascriptInterfaceUnitTest.kt`

**New test added (after `open` test, before `sendMessage` tests):**

```kotlin
@Test
fun `when launchApp is invoked, onLaunchApp callback is called with url`() {
    var capturedUrl: String? = null
    subject.onLaunchApp = { url -> capturedUrl = url }

    val testUrl = "https://www.paypal.com/app-switch-checkout?token=abc123"
    subject.launchApp(testUrl)

    assertEquals(testUrl, capturedUrl)
}
```

---

## XOSphereAndroid Changes (for local testing only)

To test with the local popup-bridge-android, the following temporary change is needed:

**`XOSphere/build.gradle.kts`** — Change the popup-bridge dependency from remote to local snapshot:

```diff
 fun DependencyHandlerScope.braintree() {
     implementation(libs.braintree.paypal)
     implementation(libs.braintree.datacollector)
     implementation(libs.braintree.shopperinsights)
-    implementation("com.braintreepayments.api:popup-bridge:5.1.0")
+    implementation("com.braintreepayments.api:popup-bridge:5.1.1-SNAPSHOT")
 }
```

This requires first publishing popup-bridge-android to Maven local:
```bash
cd popup-bridge-android
./gradlew :PopupBridge:publishToMavenLocal
```

The XOSphereAndroid project already has `mavenLocal()` in its root `build.gradle.kts` `allprojects.repositories`, so it will resolve the snapshot automatically.

**Revert this change** after testing — it's only needed for local dev.

---

## Design Notes

- **App detection**: Uses `PackageManager.getApplicationInfo("com.paypal.android.p2pmobile")` via existing `AppHelper`, consistent with Venmo detection
- **App launch**: Uses `Intent(ACTION_VIEW, uri)` with `startActivity()` — the URL is a universal link (`https://www.paypal.com/app-switch-checkout?...`) that Android resolves to the PayPal app via verified App Links
- **Return handling**: The existing `handleReturnToApp(intent)` in `PopupBridgeClient` already handles deep link returns via the registered custom URL scheme — no changes needed for return flow
- **JS injection timing**: `isPayPalInstalled` is injected in `onPageFinished()` (same as `isVenmoInstalled`), which is when the `@JavascriptInterface` object is already available
- **JS SDK dependency**: The JS SDK must separately implement logic to check `window.popupBridge.isPayPalInstalled` and call `window.popupBridge.launchApp(url)` — the native bridge only provides the capability

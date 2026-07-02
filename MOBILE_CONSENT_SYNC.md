# Mobile ↔ Web Cookie Consent Sync

Architecture and implementation plan for syncing native Mobile Consent Management (MCM) consent into the WebView's Cookie Consent SDK (CCM).

---

## Problem

The Android and iOS apps collect consent natively via the Securiti Mobile CMP SDK. The WebView loads the same tenant's CCM SDK on this GitHub Pages site. Without a sync bridge, the two consent systems are independent — native consent decisions have no effect on web cookies.

---

## Solution: Two Complementary Layers

```
MCM SDK (native — Android / iOS)
       │  user gives consent
       ▼
ConsentSyncManager (native)
       │
       ├── Layer 1 ── URL param: ?subjectId=<uuid>
       │                   → CCM beforeLoad → bareSdk.setUuid(uuid)
       │                   → backend auto-restores stored consent
       │
       └── Layer 2 ── evaluateJavascript / evaluateJavaScript
                          → window.onMobileConsentSync({ purposes, allNonEssentialGranted })
                          → sdk.acceptAll() or sdk.rejectAll()
```

| Layer | Mechanism | When it helps |
|---|---|---|
| 1 — UUID backend | `?subjectId` in URL, `bareSdk.setUuid()` | Durable across sessions and app restarts |
| 2 — JS bridge | `evaluateJavascript` → `window.onMobileConsentSync` | Immediate, first session, offline-capable |

---

## Web Side (`index.html`)

### 1. Update `beforeLoad` to inject UUID

```js
SecuritiDataLayer.push(['beforeLoad', function (bareSdk) {
  // Layer 1: link CCM session to the native MCM subject UUID
  var uuid = new URLSearchParams(window.location.search).get('subjectId');
  if (uuid) bareSdk.setUuid(uuid);

  // Suppress visual banner on mobile — consent is handled natively
  if (_isMobileView) bareSdk.stopShowingBanner();
}]);
```

### 2. Add `window.onMobileConsentSync` receiver

Called by the native app via `evaluateJavascript` after consent is collected.

```js
window.onMobileConsentSync = function (payload) {
  try {
    var data = typeof payload === 'string' ? JSON.parse(payload) : payload;

    if (!window.SecuritiSDK || typeof window.SecuritiSDK.onReady !== 'function') {
      window._pendingMobileSync = data;   // SDK not ready yet — queue
      return;
    }

    window.SecuritiSDK.onReady(function (sdk) {
      data.allNonEssentialGranted ? sdk.acceptAll() : sdk.rejectAll();
    });
  } catch (e) {
    console.error('[MCM-Sync] payload error', e);
  }
};

// Drain queue once SDK is ready (race-condition guard)
window.SecuritiSDK && window.SecuritiSDK.onReady(function () {
  if (window._pendingMobileSync) {
    window.onMobileConsentSync(window._pendingMobileSync);
    window._pendingMobileSync = null;
  }
});
```

### JSON Payload Shape

```json
{
  "purposes": [
    {
      "purposeId": 101,
      "purposeName": "Analytics",
      "consentStatus": "GRANTED",
      "isEssential": false
    },
    {
      "purposeId": 102,
      "purposeName": "Marketing",
      "consentStatus": "DECLINED",
      "isEssential": false
    }
  ],
  "allNonEssentialGranted": false
}
```

`allNonEssentialGranted` is pre-computed on the native side (essential/`disableOptOut` purposes excluded). The JS side only reads this boolean to call `acceptAll()` or `rejectAll()`.

---

## Android Side

**Project:** `SecuritiMobileCookieSync` (Kotlin / Jetpack Compose)

### Files to change

| File | Change |
|---|---|
| `App.kt` | Generate persistent UUID, expose as `App.subjectId`, pass as `subjectId` to `CmpSDKOptions` |
| `ConsentSyncManager.kt` | **New file** — MCM listener + debounce + JS bridge |
| `MainActivity.kt` | Append `?subjectId` to URL, create + wire `ConsentSyncManager`, call `onWebViewReady` in `onPageFinished` |

### `App.kt` — UUID

```kotlin
class App : Application() {
    companion object {
        lateinit var subjectId: String
            private set
    }

    override fun onCreate() {
        super.onCreate()
        val prefs = getSharedPreferences("mcm_prefs", Context.MODE_PRIVATE)
        subjectId = prefs.getString("mcm_subject_id", null)
            ?: UUID.randomUUID().toString().also {
                prefs.edit().putString("mcm_subject_id", it).apply()
            }

        SecuritiMobileCmp.initialize(this, CmpSDKOptions(
            appURL    = "https://dev-intg-9.securiti.xyz",
            cdnURL    = "https://cdn-dev-intg-9.securiti.xyz/consent",
            tenantID  = "79566875-3741-4218-8dbf-3cb421fc1c09",
            appID     = "4b069317-a672-497f-b69b-885cccc5e4d6",
            testingMode  = true,
            loggerLevel  = CmpSDKLoggerLevel.DEBUG,
            subjectId    = subjectId
        ))
    }
}
```

### `ConsentSyncManager.kt` — New file

```kotlin
class ConsentSyncManager(webView: WebView) {

    private val webViewRef = WeakReference(webView)
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    private val mainHandler = Handler(Looper.getMainLooper())
    private var debounceRunnable: Runnable? = null
    @Volatile var isWebViewReady = false

    private val listener = object : ConsentSdkListener {
        override fun onConsentChanged(consent: ConsentContainer) = scheduleSync()
        override fun onPermissionConsentChanged(consent: PermissionConsent) = Unit
        @Deprecated("") override fun onReady() = Unit
        @Deprecated("") override fun onUiDisplayed() = Unit
        @Deprecated("") override fun onUiHidden() = Unit
    }

    init { SecuritiMobileCmp.addListener(listener) }

    fun onWebViewReady() { isWebViewReady = true; scheduleSync() }

    private fun scheduleSync() {
        debounceRunnable?.let { mainHandler.removeCallbacks(it) }
        debounceRunnable = Runnable { syncConsent() }.also {
            mainHandler.postDelayed(it, 300L)
        }
    }

    private fun syncConsent() {
        if (!isWebViewReady) return
        scope.launch {
            val purposes = SecuritiMobileCmp.getPurposes()
            val nonEssential = purposes.filter { it.disableOptOut != true }
            val allGranted = nonEssential.isNotEmpty() &&
                nonEssential.all { it.consentStatus == ConsentStatus.GRANTED }

            val json = buildPayload(purposes, allGranted)
            webViewRef.get()?.evaluateJavascript("window.onMobileConsentSync($json)", null)
        }
    }

    fun destroy() {
        debounceRunnable?.let { mainHandler.removeCallbacks(it) }
        scope.coroutineContext[Job]?.cancel()
        webViewRef.clear()
    }
}
```

### `MainActivity.kt` — key additions

```kotlin
private const val BASE_URL = "https://rohan317-srti.github.io/mobile-cookie-consent-sync/"
private fun buildTargetUrl() = "$BASE_URL?subjectId=${App.subjectId}"

@Composable
fun CookieSyncWebView(modifier: Modifier = Modifier) {
    val managerRef = remember { mutableStateOf<ConsentSyncManager?>(null) }
    DisposableEffect(Unit) { onDispose { managerRef.value?.destroy() } }

    AndroidView(modifier = modifier, factory = { context ->
        WebView(context).apply {
            settings.javaScriptEnabled = true
            settings.domStorageEnabled = true
            val syncManager = ConsentSyncManager(this).also { managerRef.value = it }
            webViewClient = object : WebViewClient() {
                override fun onPageFinished(view: WebView, url: String) {
                    syncManager.onWebViewReady()
                }
                // ... existing overrides
            }
            loadUrl(buildTargetUrl())
        }
    })
}
```

---

## iOS Side

**Project:** `ConsentTestApp` (Objective-C, uses `ConsentUI.xcframework`)

**Key SDK APIs:**
```objc
// Async — returns JSON string of all purpose consents
[ConsentSDK.shared getPurposeConsentsJsonWithCompletion:^(NSString *json) { ... }];

// SDK status change (KVO via SDKListener): 0=Available 1=NotAvailable 2=InProgress
[ConsentSDK.shared isReadyObjCWithCallback:^(NSInteger status) { ... }];
```

### Files to change

| File | Change |
|---|---|
| `ConsentSyncManager.h/.m` | **New file** — consent reader + debounce + WKWebView JS bridge |
| `ViewController.m` | Generate/persist UUID, add WKWebView, wire manager, call sync on `viewWillAppear` and `didFinishNavigation` |

### `ConsentSyncManager.m` — New file (key methods)

```objc
- (void)scheduleSync {
    if (self.debounceBlock) dispatch_block_cancel(self.debounceBlock);
    __weak typeof(self) weak = self;
    self.debounceBlock = dispatch_block_create(0, ^{ [weak syncConsent]; });
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 300 * NSEC_PER_MSEC),
                   dispatch_get_main_queue(), self.debounceBlock);
}

- (void)syncConsent {
    if (!self.webViewReady) return;
    [ConsentSDK.shared getPurposeConsentsJsonWithCompletion:^(NSString *json) {
        // Parse JSON, compute allNonEssentialGranted, re-serialize payload
        NSString *script = [NSString stringWithFormat:
            @"window.onMobileConsentSync(%@)", payloadJson];
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.webView evaluateJavaScript:script completionHandler:nil];
        });
    }];
}

- (void)onWebViewReady { self.webViewReady = YES; [self scheduleSync]; }
- (void)destroy        { self.webView = nil; if (self.debounceBlock) dispatch_block_cancel(self.debounceBlock); }
```

### `ViewController.m` — key additions

```objc
// UUID
NSString *uuid = [[NSUserDefaults standardUserDefaults] stringForKey:@"mcm_subject_id"];
if (!uuid) {
    uuid = NSUUID.UUID.UUIDString;
    [[NSUserDefaults standardUserDefaults] setObject:uuid forKey:@"mcm_subject_id"];
}

// WKWebView
NSString *urlStr = [NSString stringWithFormat:
    @"https://rohan317-srti.github.io/mobile-cookie-consent-sync/?subjectId=%@", uuid];
[self.webView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:urlStr]]];

// Trigger sync when MCM banner modal is dismissed
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    [self.syncManager scheduleSync];
}

// WKNavigationDelegate
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation {
    [self.syncManager onWebViewReady];
}
```

> **Why `viewWillAppear` as the iOS consent trigger?**
> The iOS MCM SDK has no per-purpose `onConsentChanged` callback (unlike Android). When the native consent banner modal is dismissed, `viewWillAppear` reliably fires on the presenting ViewController — that's the signal to re-read and sync consent.

---

## Key Design Decisions

| Decision | Reason |
|---|---|
| `WeakReference` / `weak` WebView | MCM listener is registered on a global singleton that outlives the Activity/VC; prevents View hierarchy leak |
| 300 ms debounce | Accept-all fires N `onConsentChanged` events (one per purpose); collapse into a single JS call |
| Exclude `disableOptOut == true` purposes | Essential categories are always GRANTED; including them skews the "all granted" check |
| Conservative default → `rejectAll()` | When purposes are `NOT_DETERMINED`, privacy-safe default until user explicitly consents |
| UUID via `?subjectId` URL param | `beforeLoad` fires before the SDK script runs — URL params are the only safe injection point at that moment |

---

## Startup Sequence (happy path)

```
App starts
  → UUID read from prefs (or generated + saved)
  → MCM SDK initialised with subjectId = uuid

MainActivity / ViewController loads
  → WebView created
  → ConsentSyncManager created, MCM listener registered
  → loadUrl(".../index.html?subjectId=<uuid>")

  → MCM SDK ready → presentConsentBanner shown

WebView loads page:
  → beforeLoad: bareSdk.setUuid(uuid) + stopShowingBanner()
  → CCM SDK fetches backend consent for uuid → applies it (Layer 1)

WebView onPageFinished / didFinishNavigation:
  → syncManager.onWebViewReady()
  → 300 ms debounce → getPurposes / getPurposeConsentsJson
  → evaluateJavascript("window.onMobileConsentSync(...)")  (Layer 2)

User interacts with native MCM banner:
  → Android: onConsentChanged fires → debounce → sync
  → iOS: banner dismisses → viewWillAppear fires → scheduleSync → sync
```

---

## Verification

| Test | Expected |
|---|---|
| Fresh install | MCM banner shown. After accept/decline, Chrome DevTools / Safari Web Inspector shows `onMobileConsentSync` called with correct `allNonEssentialGranted` |
| App restart | Same UUID from prefs. No MCM banner (already consented). CCM `beforeLoad` gets UUID. Layer 2 syncs on page load. |
| Change consent in MCM pref center | Within 300 ms, WebView receives updated sync call |
| Block CCM CDN in DevTools | `_pendingMobileSync` is set. Unblock CDN → `onReady` drains queue |
| Android screen rotate | `DisposableEffect.onDispose` destroys old manager; new composable creates fresh one — no double listener |

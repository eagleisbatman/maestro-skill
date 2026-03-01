# Native Android (Kotlin/Java) — Maestro Reference

## Selector Strategy

### Android Views (XML Layouts)

```xml
<Button
    android:id="@+id/loginButton"
    android:text="Sign In"
    android:contentDescription="Login button" />

<EditText
    android:id="@+id/emailInput"
    android:hint="Email address" />
```

#### Selector Mapping — Views

| Android Attribute | Maestro Selector | Notes |
|---|---|---|
| `android:id` (resource ID) | `id:` | Full: `com.myapp:id/loginButton` or short: `loginButton` |
| `android:text` | `text:` | Visible text content |
| `android:contentDescription` | `text:` | Accessibility text (same selector as text content) |
| `android:hint` | `text:` | Input hint text |

### Jetpack Compose

```kotlin
Button(
    onClick = { },
    modifier = Modifier
        .semantics { testTagsAsResourceId = true }  // REQUIRED
        .testTag("login-button")
) {
    Text("Sign In")
}

TextField(
    value = email,
    onValueChange = { },
    modifier = Modifier
        .semantics { testTagsAsResourceId = true }  // REQUIRED
        .testTag("email-input"),
    label = { Text("Email") }
)
```

#### Selector Mapping — Compose

| Compose Property | Maestro Selector | Notes |
|---|---|---|
| `Modifier.testTag()` | `id:` | **Requires `testTagsAsResourceId = true`** in semantics |
| `Text()` content | `text:` | Visible text |
| `Modifier.semantics { contentDescription = "x" }` | `text:` | Same as text selector |

**Critical**: `testTag` alone is NOT enough. You MUST set `testTagsAsResourceId = true`:

```kotlin
// Apply globally via CompositionLocalProvider or per-element
Modifier.semantics { testTagsAsResourceId = true }.testTag("my-id")
```

### Selector Priority (Both Systems)
1. `testTag` (Compose) / `resource-id` (Views) → `id:` (best)
2. `contentDescription` → `text:`
3. Text content → `tapOn: "text"`
4. Hint text → `tapOn: "Email address"`
5. Index-based (fragile, last resort)

## View System vs Jetpack Compose

### View System (XML Layouts)
```yaml
# Resource IDs — full or short form
- tapOn:
    id: "com.example.myapp:id/submitButton"
# Short form (often works)
- tapOn:
    id: "submitButton"

# RecyclerView scrolling
- scrollUntilVisible:
    element:
      text: "Item 25"
    direction: DOWN
    timeout: 15000

# Spinner (dropdown)
- tapOn:
    id: "com.example.myapp:id/countrySpinner"
- tapOn: "Japan"

# Checkbox
- tapOn:
    id: "com.example.myapp:id/termsCheckbox"
```

### Jetpack Compose
```yaml
# testTag — no package prefix (cleaner)
- tapOn:
    id: "submit-button"

# LazyColumn scrolling
- scrollUntilVisible:
    element:
      id: "list-item-42"
    direction: DOWN
    timeout: 15000

# Material3 components via text/testTag
- tapOn:
    id: "filter-chip-active"
```

### Mixed Apps (Views + Compose)

If the app mixes both UI systems, selectors differ per component:
- Compose: `testTag` (no package prefix)
- Views: `resource-id` (with package prefix)
- Use `maestro studio` to inspect which system renders each element

## Common Patterns

### Navigation Component
```yaml
# Bottom navigation
- tapOn: "Home"
- tapOn: "Profile"

# Navigation drawer
- swipe:
    direction: RIGHT
    start: 0%, 50%
    end: 50%, 50%
- tapOn: "Settings"

# Back (hardware button — Android only)
- pressKey: back
```

### Intents & Deep Links
```yaml
- openLink: "myapp://product/123"
- assertVisible: "Product Details"

# With auto-verify (Android 12+ — bypasses disambiguation dialog)
- openLink:
    link: "myapp://product/123"
    autoVerify: true

# Force open in browser
- openLink:
    link: "https://example.com"
    browser: true
```

### System Dialogs & Permissions

```yaml
# PREFERRED: Programmatic permissions at launch
- launchApp:
    permissions:
      notifications: allow
      location: allow
      camera: allow
      photos: allow
      android.permission.ACCESS_FINE_LOCATION: allow   # Android-specific permission name

# FALLBACK: Tap system dialog
- tapOn: "While using the app"    # Android 11+
- tapOn: "Allow"                   # Older versions
```

**Gotcha**: Permissions default to ALL ALLOW even with `clearState: true`.

### Toast Messages
```yaml
# Toasts are system-level — Maestro can assert on them
- extendedWaitUntil:
    visible: "Saved successfully"
    timeout: 4000
```

### AlertDialog
```yaml
- assertVisible: "Delete item?"
- tapOn: "Delete"     # Positive button
# or
- tapOn: "Cancel"
```

### ViewPager / Tabs
```yaml
# Tab layout
- tapOn: "Tab 2"

# ViewPager swipe
- swipe:
    direction: LEFT
    start: 80%, 50%
    end: 20%, 50%
```

### Notifications
```yaml
# Test via deep link (recommended over notification shade interaction)
- openLink: "myapp://notifications/order-123"
- assertVisible: "Order #123"
```

## TextInput

```yaml
- tapOn:
    id: "email-input"
- eraseText              # Default: 50 chars. Use eraseText: 200 for longer
- inputText: "user@android.test"
- hideKeyboard

# IME action (Next/Done on keyboard)
- pressKey: enter
```

**Unicode limitation**: `inputText` only supports **ASCII on Android**. Unicode views can still be tapped/asserted.

## Airplane Mode (Android Only)

```yaml
- setAirplaneMode: enabled
- assertVisible: "No internet"
- setAirplaneMode: disabled

# Or toggle:
- toggleAirplaneMode
```

## Location Mocking (API 31+)

```yaml
- setLocation:
    latitude: 37.7749
    longitude: -122.4194

# Simulate movement
- travel:
    points:
      - latitude: 37.7749
        longitude: -122.4194
      - latitude: 37.7800
        longitude: -122.4250
    speed: 10    # meters/second
```

**Cloud gotcha**: IP-based location still resolves to US even with setLocation.

## WebView Content (with Chrome DevTools)

```yaml
appId: com.example.app
androidWebViewHierarchy: devtools    # Enables Chrome DevTools element access
---
- launchApp
# Now WebView elements are fully inspectable via Maestro
```

Without this flag, WebView content may be invisible or have incomplete selectors. Debug with `chrome://inspect` on host machine.

## Process Death Testing

```yaml
# Test app restoration after system-initiated kill
- launchApp:
    appId: "com.example.app"
- tapOn: "Enter data"
- inputText: "Important form data"

# Kill app (system process death — not user-initiated)
- killApp

# Relaunch WITHOUT stopping (foregrounds from saved state)
- launchApp:
    stopApp: false

# Verify data survived process death
- assertVisible: "Important form data"
```

## App ID & Build

```groovy
// android/app/build.gradle
android {
    defaultConfig {
        applicationId "com.example.myapp"
    }
    buildTypes {
        debug { applicationIdSuffix ".debug" }
    }
}
```

```bash
# Build debug APK
./gradlew assembleDebug

# Install on emulator
adb install app/build/outputs/apk/debug/app-debug.apk

# Find installed package name
adb shell pm list packages | grep myapp
```

## Device Management

```bash
# Start emulator via Maestro
maestro start-device --platform android

# Target specific device
maestro --device emulator-5554 test flow.yaml

# Set locale
maestro start-device --platform android --device-locale ja_JP
```

## Detect Maestro in App

```yaml
- launchApp:
    arguments:
      isMaestro: "true"
      testMode: true
```

App-side: `intent.getStringExtra("isMaestro")`

## AI Assertions

```yaml
- assertWithAI: "the RecyclerView shows at least 5 items with images"
- assertNoDefectsWithAI
```

## Gotchas

- **`testTagsAsResourceId = true`** is REQUIRED for Compose `testTag` to work with Maestro
- **Unicode `inputText`** not supported on Android (ASCII only)
- **ProGuard/R8** can strip resource IDs in release builds — test with debug builds or add keep rules
- **Emulator performance**: Use x86_64 images with hardware acceleration. ARM images are too slow
- **Multiple activities**: `launchApp` always starts the launcher activity. Use `openLink` for deep links
- **Fragment transactions**: Need `waitForAnimationToEnd` between fragments
- **Oppo devices**: Unable to clear state on physical Oppo devices — disable "Verify apps over USB" in Developer Settings
- **API level for setLocation**: Requires API 31+ (Android 12)

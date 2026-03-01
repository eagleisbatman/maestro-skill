# Hybrid Apps (Capacitor / Ionic / Cordova / PWA) — Maestro Reference

## Overview

Hybrid apps run web content inside a native WebView. Maestro interacts with both the native shell and web content through the accessibility tree. The selector strategy differs from pure native apps.

## Selector Strategy

WebView elements are accessed through the rendered accessibility tree. What's available depends on the platform and WebView version.

```html
<!-- In your web app -->
<button data-testid="login-btn" aria-label="Sign in">Sign In</button>
<input data-testid="email-input" placeholder="Email" aria-label="Email address" />
```

```yaml
# Text content (most reliable in WebView)
- tapOn: "Sign In"

# aria-label
- tapOn:
    text: "Sign in"
- tapOn:
    text: "Email address"

# data-testid or id (may work — verify with maestro studio)
- tapOn:
    id: "login-btn"
```

### Selector Priority for Hybrid Apps
1. Visible text content → `tapOn: "text"` (most reliable in WebView)
2. `aria-label` → matches via `text:` selector
3. `data-testid` or HTML `id` → `id:` (test with `maestro studio` — not guaranteed)
4. Coordinate-based (last resort)

**Always verify selectors with `maestro studio`** — WebView accessibility exposure varies by platform, OS version, and WebView version.

### Chrome DevTools for Android WebView

If WebView elements are invisible or incomplete, add this to your flow header:

```yaml
appId: com.example.app
androidWebViewHierarchy: devtools    # Enables Chrome DevTools element inspection
---
- launchApp
```

This uses Chrome DevTools Protocol to access the full WebView DOM tree. Debug with `chrome://inspect` on your host machine.

## Capacitor-Specific

### App ID
```typescript
// capacitor.config.ts
const config: CapacitorConfig = {
  appId: 'com.example.myapp',
  appName: 'MyApp',
};
```

### Native Plugin Dialogs

Capacitor plugins (Camera, Share, Geolocation) trigger native OS dialogs. Use programmatic permissions:

```yaml
# PREFERRED: Set permissions at launch
- launchApp:
    permissions:
      camera: allow
      location: allow
      photos: allow

# FALLBACK: Tap system dialogs
- tapOn: "Allow"           # iOS
- tapOn: "While using the app"  # Android
```

### Build & Install

```bash
# Build web assets
npm run build
npx cap sync

# Android
cd android && ./gradlew assembleDebug
adb install android/app/build/outputs/apk/debug/app-debug.apk

# iOS (Simulator only)
npx cap open ios   # Opens Xcode — build for Simulator
```

## Ionic-Specific

### Ionic Components

Ionic components render as web elements. Use text content and aria attributes:

```yaml
# Ion-button
- tapOn: "Submit"

# Ion-input — tap the label, then input
- tapOn:
    text: "Email"
- inputText: "user@test.com"

# Ion-select (opens native-like popover)
- tapOn: "Country"
- tapOn: "United States"
- tapOn: "OK"

# Ion-toggle
- tapOn:
    text: "Dark Mode"

# Ion-segment
- tapOn: "Tab 1"

# Ion-searchbar
- tapOn: "Search"
- inputText: "query"

# Ion-modal — dismiss
- swipe:
    direction: DOWN
    start: 50%, 30%
    end: 50%, 80%

# Ion-action-sheet
- tapOn: "Options"
- tapOn: "Delete"

# Ion-toast — auto-dismisses, assert quickly
- extendedWaitUntil:
    visible: "Saved!"
    timeout: 3000
```

### Navigation (Angular / React / Vue Router)

```yaml
# ion-tab-bar
- tapOn: "Home"
- tapOn: "Settings"

# Back navigation
- tapOn:
    text: "Back"    # ion-back-button
# or
- pressKey: back     # Android hardware back

# Deep link
- openLink: "myapp://page/details/123"
```

## Cordova-Specific

Cordova apps work similarly to Capacitor. No dedicated Maestro documentation, but the same WebView strategies apply:

- Use `androidWebViewHierarchy: devtools` for Android
- Rely on text selectors and aria-labels
- Native plugins produce OS dialogs — handle with programmatic permissions
- `appId` is in `config.xml`: `<widget id="com.example.app">`

## PWA & TWA Testing

### Android TWA (Trusted Web Activity)

```yaml
# TWA has a real Android appId
appId: com.example.twa
---
- launchApp
- assertVisible: "Home"
```

### Web Desktop Testing (Maestro Chromium)

Maestro can test web apps directly in a Chromium browser — no mobile wrapper needed:

```yaml
# Use url: instead of appId:
url: https://myapp.example.com
---
- launchApp
- tapOn: "Sign In"
- inputText: "user@test.com"
- assertVisible: "Welcome"
```

```bash
# Launch Maestro Studio for web
maestro -p web studio

# Run web tests
maestro test web-flow.yaml
```

**Web limitations**: Chromium only (no Firefox/Safari), locale locked to en-US, screen dimensions not configurable. Same commands as mobile.

### PWA Offline Testing

PWA offline capabilities can be tested via airplane mode (Android only):
```yaml
# Load app with network to cache service worker
- launchApp
- assertVisible: "Home"

# Go offline
- setAirplaneMode: enabled

# Verify cached content still works
- tapOn: "Cached Page"
- assertVisible: "Cached Content"

# Verify offline indicator
- assertVisible: "You are offline"

# Restore network
- setAirplaneMode: disabled
```

## Common Hybrid Patterns

### WebView Scrolling
```yaml
# Standard scroll
- scrollUntilVisible:
    element:
      text: "Footer Section"
    direction: DOWN

# Horizontal carousel
- swipe:
    direction: LEFT
    start: 80%, 50%
    end: 20%, 50%
```

### Form Handling
```yaml
# Input fields
- tapOn:
    text: "Full Name"
- inputText: "John Doe"
- hideKeyboard

# HTML <select> — opens native picker on mobile
- tapOn: "Choose Country"
- tapOn: "Canada"
# iOS may need:
- tapOn: "Done"

# Checkbox
- tapOn:
    text: "I agree to terms"

# Erase and retype
- tapOn:
    text: "Email"
- eraseText
- inputText: "new@email.com"
```

### Splash Screen
```yaml
# Capacitor/Ionic apps often show a splash screen
- launchApp
- extendedWaitUntil:
    visible: "Home"
    timeout: 8000        # Allow time for splash to dismiss
```

## Environment Variables

```yaml
appId: ${APP_ID}
env:
  BASE_URL: "https://staging.myapp.com"
  TEST_USER: "testuser@hybrid.test"
---
```

## Debugging Hybrid Apps

1. **`maestro studio`** — essential for WebView selector discovery
2. **`androidWebViewHierarchy: devtools`** — in flow header for Android WebView inspection
3. **Chrome DevTools** (Android): `chrome://inspect` to see WebView DOM
4. **Safari Web Inspector** (iOS): Develop → Simulator → WebView
5. **`takeScreenshot`** at each step for failure analysis
6. **AI assertions**: `assertWithAI: "the checkout form shows all required fields"`

## InAppBrowser (Cordova/Capacitor)

InAppBrowser opens a separate WebView:
- Maestro can interact with it
- Selectors may change from main WebView
- Close it to return to main app:
```yaml
- tapOn: "Open External"
# In InAppBrowser
- tapOn: "Accept Terms"
- tapOn: "Close"         # or Done button
# Back in main app
- assertVisible: "Home"
```

## Gotchas

- **WebView version matters**: Older Android WebViews (pre-Chrome 80) may not expose accessibility correctly. Test on recent emulator images
- **Keyboard overlap**: On smaller screens, keyboard covers inputs. Use `hideKeyboard` between fields
- **CSS animations**: `waitForAnimationToEnd` may not detect CSS transitions. Use explicit waits:
  ```yaml
  - extendedWaitUntil:
      visible: "Loaded"
      timeout: 3000
  ```
- **CORS / Mixed content**: Staging environments may block API calls. Ensure test environment is correctly configured
- **Status bar tap**: iOS "tap to scroll to top" doesn't work in WebView
- **`setAirplaneMode`**: Android only — not available on iOS
- **File upload** (`input type=file`): Triggers native file picker — interaction is OS-dependent and fragile
- **iframes**: WebView content inside iframes may not be accessible. Use `androidWebViewHierarchy: devtools` on Android

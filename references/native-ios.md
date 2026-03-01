# Native iOS (Swift / SwiftUI / UIKit) — Maestro Reference

## Critical: Simulator Only

**Maestro supports iOS Simulator builds ONLY — no real device support.** Build with:
```bash
xcrun xcodebuild -scheme MyApp -sdk iphonesimulator -configuration Debug build
```
Use `.app` bundles, NOT `.ipa` files.

## Selector Strategy

### SwiftUI

```swift
Button("Sign In") { }
    .accessibilityIdentifier("login-button")

TextField("Email", text: $email)
    .accessibilityIdentifier("email-input")
    .accessibilityLabel("Email address")   // Takes precedence over "Email"
```

### UIKit

```swift
loginButton.accessibilityIdentifier = "login-button"
loginButton.accessibilityLabel = "Sign in to your account"
emailField.accessibilityIdentifier = "email-input"
```

### Selector Mapping

| iOS Property | Maestro Selector | Notes |
|---|---|---|
| `accessibilityIdentifier` | `id:` | Most stable — doesn't change with localization |
| `accessibilityLabel` | `text:` | **Takes precedence over view text content** |
| View text content | `text:` | Falls back when no accessibilityLabel set |

**Critical gotcha**: When both text content AND `accessibilityLabel` exist on an element, the `text:` selector matches `accessibilityLabel`, NOT the visible text.

### Selector Priority
1. `accessibilityIdentifier` → `id:` (best — locale-independent)
2. `accessibilityLabel` → `text:` (good, but may change with l10n)
3. Text content → `tapOn: "text"` (only when no accessibilityLabel set)
4. Index-based (fragile, last resort)

## SwiftUI Patterns

### NavigationStack / NavigationView
```yaml
# Navigate forward
- tapOn: "Settings"
- assertVisible: "Settings"

# Navigate back
- tapOn:
    text: "Back"    # Default back button label
# Or use parent screen title:
- tapOn: "Home"     # Back button shows parent title
```

### TabView
```yaml
- tapOn: "Home"
- tapOn: "Search"
- tapOn: "Profile"
```

### List / ScrollView
```yaml
- scrollUntilVisible:
    element:
      id: "row-item-42"
    direction: DOWN
    timeout: 10000
    speed: 40

# Pull to refresh
- swipe:
    direction: DOWN
    start: 50%, 30%
    end: 50%, 70%
- waitForAnimationToEnd
```

### Sheet / Modal
```yaml
# Presented sheet
- assertVisible: "Sheet Title"
- tapOn: "Done"

# Dismiss by swiping down
- swipe:
    direction: DOWN
    start: 50%, 30%
    end: 50%, 80%
```

### Alert
```yaml
- assertVisible: "Delete Item?"
- tapOn: "Delete"      # Destructive action
# or
- tapOn: "Cancel"
```

### ActionSheet / ConfirmationDialog
```yaml
- tapOn: "More Options"
- assertVisible: "Share"
- tapOn: "Share"
```

### DatePicker / Picker
```yaml
# Inline date picker (iOS 14+)
- tapOn: "Date"
- tapOn: "15"    # Day

# Wheel picker — coordinate-based (tricky)
- tapOn:
    id: "date-picker"
- swipe:
    direction: UP
    start: 50%, 60%
    end: 50%, 40%
```

### Toggle / Switch
```yaml
- tapOn:
    id: "notifications-toggle"
# SwiftUI Toggle doesn't always expose state via accessibility
# Verify via associated text
- assertVisible: "Notifications On"
```

### SearchBar
```yaml
- tapOn: "Search"
- inputText: "query text"
- tapOn: "Cancel"     # Dismiss search
```

## UIKit Patterns

### UITableView / UICollectionView
```yaml
- scrollUntilVisible:
    element:
      text: "Cell Title"
    direction: DOWN

# Swipe-to-delete
- swipe:
    direction: LEFT
    start: 80%, 50%
    end: 20%, 50%
- tapOn: "Delete"
```

**Known issue**: Lists with pagination (UITableView/UICollectionView) — XCTest bug may trigger `willDisplayCell` unexpectedly. Verify cell visibility in delegate.

### UIAlertController
```yaml
- assertVisible: "Error"
- tapOn: "OK"
```

### UIPageViewController
```yaml
- swipe:
    direction: LEFT
    start: 80%, 50%
    end: 20%, 50%
- assertVisible: "Page 2"
```

## TextInput

```yaml
- tapOn:
    id: "email-input"
- eraseText                    # Default 50 chars. Flaky on iOS — see gotchas
- inputText: "user@apple.test"
- hideKeyboard                 # Can be flaky — workaround below

# hideKeyboard workaround: tap a non-interactive element
- tapOn: "Screen Title"        # Tap header/label to dismiss keyboard

# Secure field (password)
- tapOn:
    id: "password-input"
- inputText:
    text: "Secure123!"
    label: "Enter password"

# Keyboard Next/Done
- pressKey: enter
```

## Permissions (Programmatic — preferred)

```yaml
- launchApp:
    permissions:
      notifications: allow
      location: allow
      camera: allow
      photos: allow
```

Permissions default to ALL ALLOW even with `clearState: true`. Explicitly deny what you need:
```yaml
- launchApp:
    clearState: true
    permissions:
      all: deny
      notifications: allow     # Only allow notifications
```

### System Permission Dialogs (if programmatic doesn't work)
```yaml
# Location
- tapOn: "Allow While Using App"

# Camera
- tapOn: "Allow"

# Notifications
- tapOn: "Allow"

# Photos (iOS 17+)
- tapOn: "Allow Full Access"
```

Note: iOS version differences change permission dialog wording. Use conditions:
```yaml
- runFlow:
    when:
      visible: "Allow Full Access"
    commands:
      - tapOn: "Allow Full Access"
- runFlow:
    when:
      visible: "OK"
    commands:
      - tapOn: "OK"
```

## iOS-Specific Commands

### Clear Keychain
```yaml
- clearKeychain    # Clears ENTIRE iOS keychain — all apps, all credentials
```

### Clear State
```yaml
- clearState    # iOS: REINSTALLS the entire app (completely fresh install)
```

**This is different from Android** where `clearState` just clears app data.

## Location Mocking

```yaml
- setLocation:
    latitude: 37.3349
    longitude: -122.0090

# Simulate travel
- travel:
    points:
      - latitude: 37.3349
        longitude: -122.0090
      - latitude: 37.3400
        longitude: -122.0150
    speed: 5    # meters/second
```

## AI-Powered Assertions

```yaml
- assertWithAI: "the navigation bar shows a back button and a title"
- assertNoDefectsWithAI
```

## Deep Links & Universal Links
```yaml
# Custom scheme
- openLink: "myapp://profile/settings"
- assertVisible: "Settings"

# Universal links
- openLink: "https://myapp.com/profile/123"
```

**First-time universal link**: iOS may show a one-time security dialog. Handle with condition:
```yaml
- runFlow:
    when:
      visible: "Open"
    commands:
      - tapOn: "Open"
```

## Biometric Auth (Face ID / Touch ID)

Maestro cannot simulate biometrics. Use a test bypass:
```swift
// In app code — detect simulator
#if targetEnvironment(simulator)
// Skip biometric, use password fallback
#endif
```

Or use launch arguments:
```yaml
- launchApp:
    arguments:
      skipBiometric: true
```

## App ID & Build

```bash
# Find bundle identifier
# Xcode: Target → General → Bundle Identifier
# Or: grep CFBundleIdentifier ios/*/Info.plist

# Build for Simulator
xcrun xcodebuild -scheme MyApp -sdk iphonesimulator -configuration Debug build

# Install on Simulator
xcrun simctl install booted path/to/MyApp.app

# List installed apps
xcrun simctl listapps booted | grep CFBundleIdentifier
```

## Device Management

```bash
# Start simulator via Maestro
maestro start-device --platform ios
maestro start-device --platform ios --os-version 18 --device-locale de_DE

# Target specific simulator
maestro --device "5B6D77EF-2AE9-47D0-9A62-70A1ABBC5FA2" test flow.yaml

# Boot specific device manually
xcrun simctl boot "iPhone 15 Pro"
```

## Detect Maestro in App

```yaml
- launchApp:
    arguments:
      isMaestro: "true"
```

App-side: `ProcessInfo.processInfo.arguments.contains("isMaestro")`

## Simulator Helpers

```bash
# Set clean status bar (for screenshots)
xcrun simctl status_bar booted override --time "9:41"

# Set location
xcrun simctl location booted set 37.7749,-122.4194

# Open URL
xcrun simctl openurl booted "myapp://test"
```

## SwiftUI Known Issues

- **WheelPickerStyle**: Accessibility hierarchy may not be returned correctly
- **Toggle with text**: Creates unified accessibility element — may need coordinate-based tap
- **Link with label**: `Link` component may only respond to one of label/identifier (not both)
- **Large Title navigation**: Collapses on scroll — element position shifts. Use `id:` not coordinates
- **Safe area**: Don't assume fixed coordinates near edges — notch/Dynamic Island affects layout

## Gotchas

- **Simulator ONLY** — no real device support for Maestro on iOS
- **`clearState` reinstalls app** — completely fresh install, not just data clear
- **`clearKeychain` clears ALL apps** — not just your app's keychain entries
- **`accessibilityLabel` takes precedence** — over visible text for `text:` selector
- **`eraseText` is flaky on iOS** — may not clear all text. Workaround: tap field, select all, type replacement
- **`hideKeyboard` unreliable on iOS** — performs swipe gestures that may not work. Tap a non-interactive element instead
- **Localization**: Use `accessibilityIdentifier` (not `accessibilityLabel`) for locale-independent selectors
- **iOS version differences**: Permission dialog wording changes between versions — use conditions
- **App Clips**: Have a separate `appId` — test independently
- **`setAirplaneMode`**: NOT supported on iOS (Android only)
- **Driver startup**: iOS driver takes longer. Set `MAESTRO_DRIVER_STARTUP_TIMEOUT=180000` if needed

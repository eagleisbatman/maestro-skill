# Flutter — Maestro Reference

## Selector Strategy

**Critical: Flutter's `Key` class is NOT exposed to the accessibility layer.** Do NOT use `Key('my-id')` for Maestro selectors — it will not work.

### Correct Approach: `Semantics` Widget

```dart
// CORRECT — Maestro can find this via id: selector
Semantics(
  identifier: 'login-button',     // Flutter ≥3.19 (contributed by Maestro team)
  child: ElevatedButton(
    onPressed: () {},
    child: const Text('Sign In'),
  ),
)

// CORRECT — semanticLabel maps to text: selector
Semantics(
  label: 'Submit form',
  child: IconButton(onPressed: () {}, icon: const Icon(Icons.check)),
)

// ALSO WORKS — Text content is accessible
ElevatedButton(
  onPressed: () {},
  child: const Text('Sign In'),  // tapOn: "Sign In" works
)

// WRONG — Key is NOT visible to Maestro
ElevatedButton(
  key: const Key('login-button'),  // ❌ Maestro CANNOT see this
  onPressed: () {},
  child: const Text('Sign In'),
)
```

### Selector Mapping

| Flutter Widget/Property | Maestro Selector | Notes |
|---|---|---|
| `Semantics(identifier: 'x')` | `id:` | **Best**. Flutter ≥3.19. Stable across locales |
| `Semantics(label: 'x')` | `text:` | Takes precedence over text content |
| `semanticLabel` property | `text:` | Same as Semantics label |
| Text widget content | `text:` / shorthand | Good for visible, static labels |
| `Key('x')` | **NOT ACCESSIBLE** | ❌ Do not use for Maestro testing |

### Selector Priority
1. `Semantics(identifier:)` → `id:` (best — stable, locale-independent)
2. `Semantics(label:)` / `semanticLabel` → `text:` (good, but may change with l10n)
3. Text widget content → `tapOn: "text"` (reasonable for static text)
4. Index-based (fragile, last resort)

**Key rule**: If `semanticLabel` AND text content both exist, `semanticLabel` takes precedence for the `text:` selector.

## Flutter-Specific Patterns

### Navigation (GoRouter / Navigator)

```yaml
# Tab-based (BottomNavigationBar)
- tapOn: "Home"
- tapOn: "Settings"

# Back navigation
- pressKey: back     # Android
# iOS: swipe from left edge
- swipe:
    direction: RIGHT
    start: 0%, 50%
    end: 50%, 50%

# Deep linking (GoRouter)
- openLink: "myapp://profile/123"
- assertVisible: "User Profile"
```

### ListView / GridView
```yaml
- scrollUntilVisible:
    element:
      text: "List Item 42"
    direction: DOWN
    timeout: 15000
    speed: 40

# Pull-to-refresh (RefreshIndicator)
- swipe:
    direction: DOWN
    start: 50%, 30%
    end: 50%, 70%
- waitForAnimationToEnd
```

### AlertDialog / BottomSheet
```yaml
# Dialog
- assertVisible: "Confirm Delete"
- tapOn: "Cancel"

# BottomSheet — dismiss by swiping down
- swipe:
    direction: DOWN
    start: 50%, 50%
    end: 50%, 90%
```

### DropdownButton / PopupMenuButton
```yaml
- tapOn:
    id: "country-dropdown"     # Semantics(identifier: 'country-dropdown')
- tapOn: "United States"
- assertVisible: "United States"
```

### DatePicker / TimePicker
```yaml
- tapOn:
    id: "date-picker-trigger"
- tapOn: "15"       # Day number
- tapOn: "OK"
```

### Snackbar
```yaml
# Snackbars auto-dismiss — use extendedWaitUntil or quick assertVisible
- extendedWaitUntil:
    visible: "Item saved"
    timeout: 3000
```

## TextInput Handling

```yaml
- tapOn:
    id: "email-input"         # Semantics(identifier: 'email-input')
- eraseText                    # Default removes 50 chars
- inputText: "test@flutter.dev"
- hideKeyboard

# For obscured text (password)
- tapOn:
    id: "password-input"
- inputText:
    text: "SecurePass!"
    label: "Enter password"     # Masks in logs/reports
```

**Android**: `inputText` only supports ASCII. Unicode views can still be tapped/asserted.

## Permissions (Programmatic)

```yaml
- launchApp:
    permissions:
      notifications: allow
      location: allow
      camera: allow
```

Permissions default to ALL ALLOW even with `clearState: true`.

## Device Control

```yaml
# Location
- setLocation:
    latitude: 40.7128
    longitude: -74.0060

# GPS movement simulation
- travel:
    points:
      - latitude: 40.7128
        longitude: -74.0060
      - latitude: 40.7200
        longitude: -74.0100
    speed: 5    # meters/second

# Airplane mode (Android only)
- setAirplaneMode: enabled

# Orientation
- setOrientation: LANDSCAPE
```

## AI-Powered Assertions

```yaml
- assertWithAI: "the product card shows an image, title, and price"
- assertNoDefectsWithAI
```

## Platform Channels / Native Modules

Capacitor/native plugins trigger OS dialogs. Use programmatic permissions instead of tapping:

```yaml
# PREFERRED: Set permissions at launch
- launchApp:
    permissions:
      camera: allow

# FALLBACK: Tap system dialog (if programmatic doesn't work)
- tapOn: "While using the app"   # Android
# or
- tapOn: "Allow"                  # iOS
```

## App ID Locations

- `android/app/build.gradle` → `applicationId`
- `ios/Runner.xcodeproj` → Bundle Identifier
- `pubspec.yaml` doesn't contain it directly

## Build for Testing

```bash
# Android — debug APK
flutter build apk --debug
adb install build/app/outputs/flutter-apk/app-debug.apk

# iOS — Simulator only (Maestro doesn't support real iOS devices)
flutter build ios --debug --simulator
# Install via Xcode or:
xcrun simctl install booted build/ios/iphonesimulator/Runner.app
```

## Detect Maestro in App

```yaml
- launchApp:
    arguments:
      isMaestro: "true"
```

App-side with `flutter_launch_arguments`:
```dart
import 'package:flutter_launch_arguments/flutter_launch_arguments.dart';
final isMaestro = LaunchArguments().contains('isMaestro');
```

## Known Issues & Gotchas

- **`Key` class NOT accessible** — this is the #1 mistake. Use `Semantics(identifier:)` instead
- **Impeller rendering engine**: No impact on Maestro selectors
- **CustomPaint / Canvas**: Elements drawn with `CustomPaint` are invisible to Maestro unless wrapped in `Semantics`
- **Animations**: Use `waitForAnimationToEnd` after route transitions, hero animations, implicit animations
- **Platform views** (WebView, Google Maps): Render natively — selectors may differ from Flutter widgets
- **Flavors/build variants**: Ensure correct `appId` for the flavor (e.g., `com.app.dev` vs `com.app.prod`)
- **Flutter Web**: Fully supported with Semantics annotations. Use `url:` in flow header
- **Flutter Desktop**: NOT supported by Maestro
- **WheelPickerStyle**: Hierarchy may not be returned correctly (known iOS issue)
- **Toggle with text**: SwiftUI/Flutter creates unified accessibility element — may need coordinate-based interaction

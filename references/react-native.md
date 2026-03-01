# React Native / Expo — Maestro Reference

## Selector Strategy

React Native maps `testID` to the native accessibility layer. This is the most stable selector.

```jsx
// In your component
<TouchableOpacity testID="login-button">
  <Text>Sign In</Text>
</TouchableOpacity>

<TextInput testID="email-input" placeholder="Email" />
```

```yaml
# In Maestro flow
- tapOn:
    id: "login-button"       # testID → id: selector
- tapOn:
    id: "email-input"
- inputText: "user@test.com"
```

### Selector Mapping

| React Native Prop | Maestro Selector | Notes |
|---|---|---|
| `testID` | `id:` | Most reliable. Works on all platforms |
| Component text content | `text:` / shorthand `tapOn: "text"` | Good for static labels |
| `placeholder` | `text:` | TextInput placeholder text |
| `accessibilityLabel` | `text:` | **Takes precedence over text content on iOS** |

### Selector Priority
1. `testID` → `id:` (best — stable across platforms, locales)
2. Text content → `tapOn: "Sign In"` (good for static labels)
3. `accessibilityLabel` → matches via `text:` selector
4. Index → `tapOn: { text: "Item", index: 2 }` (fragile, avoid)

### iOS Nested Elements Gotcha

On iOS, React Native accessibility can merge parent+child into one element:

```jsx
// PROBLEM: Inner testID invisible on iOS
<View accessible={true}>
  <Text testID="inner-text">Hello</Text>
</View>

// FIX: Set accessible={false} on outer, accessible={true} on inner
<View accessible={false}>
  <Text testID="inner-text" accessible={true}>Hello</Text>
</View>
```

If `tapOn: { id: "inner-text" }` fails on iOS but works on Android, this is the cause.

## Expo-Specific Notes

### Expo Go vs Dev Build

| Mode | `appId` | Reliability |
|---|---|---|
| **Expo Go** | `host.exp.Exponent` | Okay for prototyping, not for CI |
| **Dev build** | Your actual bundle ID | Recommended for testing |
| **EAS Build** | Your actual bundle ID | Best for CI |

```yaml
# Expo Go
appId: host.exp.Exponent
---
- launchApp
- openLink: "exp://127.0.0.1:19000"  # Needed to load your app within Expo Go
```

```yaml
# Development build (recommended)
appId: com.yourcompany.yourapp
---
- launchApp
```

### Expo Router (Deep Links)
```yaml
- openLink: "myapp://settings/profile"
- assertVisible: "Profile"

# Expo Router linking config must be set up in app
```

### EAS Build for Testing
```bash
eas build --profile development --platform android   # .apk
eas build --profile development --platform ios        # .app (simulator)
```

## Common Component Patterns

### FlatList / ScrollView
```yaml
- scrollUntilVisible:
    element:
      id: "item-42"          # or text: "Item Title"
    direction: DOWN
    timeout: 10000
    speed: 40                 # 0-100 (default: 40)
```

### Modal / Bottom Sheet
```yaml
# React Native modals are rendered as overlays
- assertVisible: "Modal Title"
- tapOn: "Confirm"

# Dismiss by tapping outside (backdrop)
- tapOn:
    point: "50%,10%"

# Or swipe down for bottom sheets
- swipe:
    direction: DOWN
    start: 50%, 40%
    end: 50%, 90%
```

### React Navigation
```yaml
# Tab navigator
- tapOn: "Home"       # Tab label text
- tapOn: "Profile"

# Stack navigator — go back
- tapOn:
    id: "header-back"   # Default back button testID in React Navigation
# Or:
- pressKey: back         # Android hardware back
# Or iOS swipe:
- swipe:
    direction: RIGHT
    start: 0%, 50%
    end: 50%, 50%
```

### Pull-to-Refresh
```yaml
- swipe:
    direction: DOWN
    start: 50%, 30%
    end: 50%, 70%
- waitForAnimationToEnd
```

### Switch / Toggle
```yaml
- tapOn:
    id: "notifications-toggle"
# Verify state changed
- assertVisible: "Notifications enabled"
```

## TextInput Handling

```yaml
# Tap, erase, type
- tapOn:
    id: "email-input"
- eraseText                    # Default removes up to 50 chars
- inputText: "newuser@test.com"

# For longer existing text
- eraseText: 200               # Remove up to 200 chars

# Dismiss keyboard
- hideKeyboard

# Secure text (password) works normally
- tapOn:
    id: "password-input"
- inputText:
    text: "SecurePass123!"
    label: "Enter password"     # Masks value in logs/reports
```

**Android Unicode limitation**: `inputText` only supports ASCII on Android. Unicode views can still be tapped/asserted.

## Permissions (Programmatic — preferred)

```yaml
- launchApp:
    permissions:
      all: allow              # default behavior
      notifications: allow
      location: allow
      camera: allow
      photos: allow
```

**Gotcha**: Permissions default to ALL ALLOW even with `clearState: true`. Explicitly `deny` what you need denied.

## Device Control

```yaml
# Airplane mode (Android only)
- setAirplaneMode: enabled
- assertVisible: "No internet"
- setAirplaneMode: disabled

# Location mocking
- setLocation:
    latitude: 37.7749
    longitude: -122.4194

# Orientation
- setOrientation: LANDSCAPE
- setOrientation: PORTRAIT
```

## AI-Powered Assertions

```yaml
- assertWithAI: "the onboarding carousel shows 3 dots at the bottom"
- assertNoDefectsWithAI    # Auto-detect visual defects
```

Requires `maestro login` (free tier). Default `optional: true`.

## Detect Maestro in App

```yaml
- launchApp:
    arguments:
      isMaestro: "true"
```

App-side with `react-native-launch-arguments`:
```javascript
import { LaunchArguments } from 'react-native-launch-arguments'
const isMaestro = LaunchArguments.value().isMaestro === 'true'
```

## Environment Variables

```yaml
appId: ${APP_ID}
env:
  TEST_EMAIL: ${TEST_EMAIL || "testuser@example.com"}
  TEST_PASSWORD: ${TEST_PASSWORD || "Test1234!"}
---
```

```bash
maestro test -e APP_ID=com.myapp.dev -e TEST_EMAIL=ci@test.com flow.yaml
```

## EAS Workflows (Expo CI/CD) Integration

EAS Workflows has a **first-class `type: maestro` pre-packaged job** — no need for custom CI scripts.

### EAS Build Profile for E2E Tests

```json
// eas.json
{
  "build": {
    "e2e-test": {
      "withoutCredentials": true,
      "ios": { "simulator": true },
      "android": { "buildType": "apk" }
    }
  }
}
```

### EAS Workflow — Android

```yaml
# .eas/workflows/e2e-test-android.yml
name: e2e-test-android
on:
  pull_request:
    branches: ['*']

jobs:
  build_android:
    type: build
    params:
      platform: android
      profile: e2e-test

  maestro_test:
    needs: [build_android]
    type: maestro
    params:
      build_id: ${{ needs.build_android.outputs.build_id }}
      flow_path: ['.maestro/home.yml', '.maestro/login.yml']
      include_tags: smoke
      shards: 2                    # parallel test shards
      retries: 1                   # retry failed tests
      record_screen: true          # video evidence
      maestro_version: '1.38.0'   # pin version for stability
```

### EAS Workflow — iOS

```yaml
# .eas/workflows/e2e-test-ios.yml
name: e2e-test-ios
on:
  pull_request:
    branches: ['*']

jobs:
  build_ios:
    type: build
    params:
      platform: ios
      profile: e2e-test

  maestro_test:
    needs: [build_ios]
    type: maestro
    params:
      build_id: ${{ needs.build_ios.outputs.build_id }}
      flow_path: '.maestro/'
      include_tags: smoke
```

### EAS Maestro Job Parameters

| Parameter | Type | Description |
|---|---|---|
| `build_id` | string | **Required.** Build ID from a build job |
| `flow_path` | string/string[] | **Required.** Path(s) to Maestro flows |
| `shards` | number | Parallel test shards (default: 1) |
| `retries` | number | Retry failed tests (default: 1) |
| `record_screen` | boolean | Record video (default: false) |
| `include_tags` | string/string[] | Tags to include |
| `exclude_tags` | string/string[] | Tags to exclude |
| `maestro_version` | string | Pin Maestro version (default: latest) |
| `android_system_image_package` | string | Android system image |
| `device_identifier` | string/object | Device model |

### Save Test Artifacts in EAS

Use `MAESTRO_TESTS_DIR` env var for screenshot/video paths:
```yaml
appId: com.myapp
---
- launchApp
- takeScreenshot: ${MAESTRO_TESTS_DIR}/login-screen
- startRecording: ${MAESTRO_TESTS_DIR}/login-flow
- tapOn: "Login"
- stopRecording
```

### Trigger Manually
```bash
npx eas-cli@latest workflow:run .eas/workflows/e2e-test-android.yml
```

### Maestro Cloud Job (Alternative)

EAS also supports `type: maestro-cloud` for running tests in Maestro's cloud infrastructure:
```yaml
maestro_test:
  type: maestro-cloud
  params:
    build_id: ${{ needs.build.outputs.build_id }}
    maestro_project_id: proj_01jw6hxgmdffrbye9fqn0pyzm0
    flows: ./maestro/flows
```
Requires Maestro Cloud account with `MAESTRO_CLOUD_API_KEY` env var.

## App ID Locations

- `app.json` → `expo.android.package` / `expo.ios.bundleIdentifier`
- `android/app/build.gradle` → `applicationId`
- `ios/<AppName>/Info.plist` → `CFBundleIdentifier`

## Gotchas

- **Hermes engine**: No impact on Maestro — selectors work the same
- **New Architecture (Fabric)**: No impact — accessibility layer unchanged
- **Animated / Reanimated**: Use `waitForAnimationToEnd` after transitions. Complex gesture components may need `swipe` with coordinates
- **Fast Refresh**: Hot reload can reset state mid-test. Use release/production builds for CI
- **Android keyboard**: If `inputText` fails, ensure software keyboard enabled: `adb shell settings put secure show_ime_with_hard_keyboard 1`
- **iOS accessibilityLabel**: Takes precedence over visible text for `text:` selector matching
- **Expo Go**: Your app runs inside the Expo Go container — some native features may behave differently than a standalone build

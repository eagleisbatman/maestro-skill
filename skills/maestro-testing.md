---
name: maestro-testing
description: Generate and manage Maestro test flows for mobile (Android, iOS) and web apps. Use this skill whenever the user mentions mobile testing, UI testing, end-to-end testing, Maestro flows, app automation, test generation from PRDs/specs/user stories, or wants to create/edit/debug YAML test flows. Trigger for any app framework — React Native, Expo, Flutter, Swift, Kotlin, Jetpack Compose, SwiftUI, UIKit, Capacitor, Ionic, Cordova, NativeScript, .NET MAUI, PWA, or web desktop. Also trigger when the user says "write tests", "generate test cases", "automate my app", "E2E tests", "Maestro MCP", "test reports", "visual regression", or references acceptance criteria needing test automation. Trigger even for web-only testing since Maestro supports desktop Chromium testing.
---

# Maestro Testing Skill

Generate production-ready Maestro YAML test flows for any mobile or web app. This skill covers flow generation, selector strategy, report generation, CI/CD integration, QA workflows, and MCP-powered AI-assisted testing.

## Prerequisites & Installation

Maestro requires **Java 17+** and installs with a single command — zero other dependencies:
```bash
curl -fsSL "https://get.maestro.mobile.dev" | bash
```
- **Android**: Emulator or physical device with USB debugging (API 26+)
- **iOS**: Xcode with iOS Simulator (**macOS only, simulator builds only — no real devices**)
- **Web**: No additional setup — Maestro auto-downloads Chromium on first run

## Before Generating Flows

Read the appropriate reference file for the user's framework:
- **React Native / Expo** → `${CLAUDE_PLUGIN_ROOT}/references/react-native.md`
- **Flutter** → `${CLAUDE_PLUGIN_ROOT}/references/flutter.md`
- **Native Android (Kotlin/Java/Jetpack Compose)** → `${CLAUDE_PLUGIN_ROOT}/references/native-android.md`
- **Native iOS (Swift/SwiftUI/UIKit)** → `${CLAUDE_PLUGIN_ROOT}/references/native-ios.md`
- **Hybrid (Capacitor/Ionic/Cordova/PWA)** → `${CLAUDE_PLUGIN_ROOT}/references/hybrid.md`
- **Web Desktop Browser** → use `url:` instead of `appId:`, same commands. No separate reference needed.
- **QA workflows, reports, CI/CD** → `${CLAUDE_PLUGIN_ROOT}/references/qa-workflows.md`

If framework is unknown, ask. Default to `${CLAUDE_PLUGIN_ROOT}/references/react-native.md`.

---

## 1. Project Structure

```
.maestro/
├── config.yaml                 # Workspace-level config
├── flows/
│   ├── auth/
│   │   ├── login.yaml
│   │   ├── login-invalid.yaml
│   │   ├── signup.yaml
│   │   └── logout.yaml
│   ├── onboarding/
│   │   └── first-launch.yaml
│   ├── core/
│   │   ├── create-item.yaml
│   │   ├── edit-item.yaml
│   │   └── delete-item.yaml
│   ├── navigation/
│   │   └── tab-navigation.yaml
│   ├── edge-cases/
│   │   ├── no-network.yaml
│   │   ├── empty-state.yaml
│   │   ├── orientation-change.yaml
│   │   └── long-text-input.yaml
│   └── smoke/
│       └── critical-path.yaml
├── subflows/                    # Reusable — NOT run as standalone tests
│   ├── login.yaml
│   ├── navigate-to.yaml
│   └── teardown.yaml
├── scripts/
│   ├── setup.js
│   ├── generate-data.js
│   ├── page-objects.js          # Page Object Model selectors
│   └── run-tests.sh
└── media/
    ├── test-photo.png
    └── test-video.mp4
```

---

## 2. Complete Command Reference

Every command accepts optional `label:` (string — masks sensitive data in logs/reports) and `optional: true` (continues on failure). AI commands default `optional` to `true`.

### Interaction
| Command | Android | iOS | Web | Notes |
|---|---|---|---|---|
| `tapOn` | ✓ | ✓ | ✓ | Supports `repeat`, `delay`, `retryTapIfNoChange`, `waitToSettleTimeoutMs` |
| `doubleTapOn` | ✓ | ✓ | ✓ | Same selectors as tapOn, adds `delay` between taps |
| `longPressOn` | ✓ | ✓ | ✓ | 3-second press, same selectors as tapOn |
| `inputText` | ✓ | ✓ | ✓ | **Unicode NOT supported on Android (ASCII only)** |
| `eraseText` | ✓ | ✓ | ✓ | Default removes 50 chars. `eraseText: 100` for more. **Flaky on iOS** |
| `copyTextFrom` | ✓ | ✓ | ✓ | Stores in `${maestro.copiedText}` |
| `pasteText` | ✓ | ✓ | ✓ | Pastes from **Maestro's internal clipboard only**, NOT OS clipboard |
| `setClipboard` | ✓ | ✓ | ✗ | Sets device clipboard text |
| `hideKeyboard` | ✓ | ✓(flaky) | ✗ | Workaround: tap non-interactive element |
| `pressKey` | ✓ | ✓(subset) | ✓ | Keys: `enter`, `backspace`, `home`, `lock`, `back`(Android), `volume up/down`, `tab`(Android) |
| `swipe` | ✓ | ✓ | ✓ | By direction, coordinates, or from element. `duration` param (ms, default 400) |
| `scroll` | ✓ | ✓ | ✓ | Simple downward scroll |
| `scrollUntilVisible` | ✓ | ✓ | ✓ | `direction`, `timeout`(ms), `speed`(0-100), `visibilityPercentage`, `centerElement` |

### Random Data Input (built-in DataFaker)
```yaml
- inputRandomEmail
- inputRandomPersonName
- inputRandomNumber            # default 8 digits
- inputRandomNumber:
    length: 10
- inputRandomText
- inputRandomText:
    length: 20
```

### Assertions
| Command | Notes |
|---|---|
| `assertVisible` | **Waits ~7s** for element to appear. All text/id selectors are regex (full match) |
| `assertNotVisible` | **Waits ~7s** for element to disappear before failing |
| `assertTrue` | JavaScript expression: `assertTrue: ${output.count > 0}` |
| `assertWithAI` | **Experimental.** Sends screenshot to LLM. Requires `maestro login`. Free tier works |
| `assertNoDefectsWithAI` | **Experimental.** Auto-detects cut-off text, overlapping elements, centering issues |
| `assertScreenshot` | Visual regression — compares against baseline |

### App Lifecycle
| Command | Android | iOS | Web | Notes |
|---|---|---|---|---|
| `launchApp` | ✓ | ✓ | ✓ | `clearState`, `clearKeychain`(iOS), `permissions`, `arguments`, `stopApp`(default true) |
| `stopApp` | ✓ | ✓ | ✗ | Stops without clearing data |
| `killApp` | ✓ | ✓ | ✗ | System-initiated process death. Use with `launchApp: stopApp: false` to test cold start |
| `clearState` | ✓ | ✓ | ✗ | Android: `pm clear`. **iOS: reinstalls entire app** |
| `clearKeychain` | ✗ | ✓ | ✗ | Clears **ENTIRE iOS keychain** |

### Device Control
| Command | Android | iOS | Web |
|---|---|---|---|
| `setAirplaneMode: enabled/disabled` | ✓ | ✗ | ✗ |
| `toggleAirplaneMode` | ✓ | ✗ | ✗ |
| `setLocation` (lat/lng) | ✓(API 31+) | ✓ | ✗ |
| `travel` (waypoints + speed) | ✓(API 31+) | ✓ | ✗ |
| `setOrientation: PORTRAIT/LANDSCAPE` | ✓ | ✓ | ✗ |
| `setPermissions` | ✓ | ✓ | ✗ |
| `addMedia` (png/jpg/gif/mp4) | ✓ | ✓ | ✗ |

### Flow Control
| Command | Notes |
|---|---|
| `runFlow` | File, inline `commands:`, conditional `when:`, env params |
| `runScript` | External .js file. Path relative to **calling flow's location** |
| `evalScript` | Single-line inline JS: `evalScript: ${output.counter = 0}` |
| `repeat` | `times:` (N), `while:` (condition), or both (AND logic) |
| `retry` | `maxRetries:` 0-3. Either `commands:` or `file:` — not both |
| `extendedWaitUntil` | `visible:`/`notVisible:` + `timeout:` (ms, REQUIRED). Completes immediately when met |
| `waitForAnimationToEnd` | Waits until screen static. Optional `timeout:` param |

### Capture
| Command | Notes |
|---|---|
| `takeScreenshot: "name"` | Saves `.png`. Supports `cropOn:` selector to crop to specific element |
| `startRecording: "name"` | Saves `.mp4` |
| `stopRecording` | Finalizes video |
| `copyTextFrom` | Copies element text to `${maestro.copiedText}` |
| `extractTextWithAI` | **Experimental.** AI-powered text extraction from screenshot |

### Navigation
| Command | Notes |
|---|---|
| `openLink` | Deep links, universal links. Android: `autoVerify:`, `browser:` params |
| `back` | **Android only** |
| `travel` | GPS movement simulation between waypoints at specified speed (m/s) |

---

## 3. Selector Strategy

Maestro uses the **accessibility tree**. All `text`/`id` fields are **regex** matching the **entire element text**.

### Priority Order
1. **`id:`** — Most stable. Maps to platform-specific accessibility identifier (see reference files)
2. **`text:`** — Visible text content. Shorthand: `tapOn: "Submit"`
3. **`enabled: true`** — **Auto-waits** for element to become interactive. Critical for API-dependent buttons
4. **Positional:** `below:`, `above:`, `leftOf:`, `rightOf:`, `childOf:`, `containsChild:`, `containsDescendants:`
5. **Traits:** `checked: true`, `focused: true`, `selected: true`
6. **Size:** `width:`, `height:`, `tolerance:` (±px)
7. **`index:`** — 0-based, supports negative (-1 = last). Fragile — last resort
8. **`point:`** — Coordinates (percentage or absolute). Most fragile

### Platform Selector Mapping
| Platform | `text:` maps to | `id:` maps to |
|---|---|---|
| Android Views | `android:text`, `contentDescription`, `hint` | `android:id` (resource ID) |
| Jetpack Compose | Text content, `contentDescription` | `testTag` (**requires `testTagsAsResourceId = true`**) |
| iOS UIKit | View text, `accessibilityLabel` (precedence) | `accessibilityIdentifier` |
| iOS SwiftUI | View text, `.accessibilityLabel()` | `.accessibilityIdentifier()` |
| React Native | Component text, `placeholder` | `testID` |
| Flutter | `semanticLabel` (precedence), text content | `Semantics(identifier:)` (Flutter ≥3.19) |
| Web | User-visible text | `css` selector (web-only) |

### Compound Selectors (AND logic)
```yaml
- tapOn:
    id: "submit-btn"
    enabled: true           # waits for button to be interactive
    below: "Form Title"     # positional constraint
```

### Regex in Selectors
```yaml
- assertVisible: '.*Welcome.*'           # contains "Welcome"
- assertVisible: 'Order #\\d{6}'         # "Order #" + 6 digits
- assertVisible: '.*\\.99'               # price ending in .99
- assertVisible: 'Movies \\[NEW\\]'      # escape brackets
```

**Gotcha:** YAML booleans — quote `"YES"`, `"NO"`, `"true"`, `"false"` to prevent YAML parsing.
**Gotcha:** Dollar signs need escaping: `\\$150`.

---

## 4. Flow Configuration (YAML Header)

Every flow has a configuration section above the `---` separator:

```yaml
appId: com.example.app                    # REQUIRED for mobile
# url: https://myapp.com                  # Use for web instead of appId
name: "Login Happy Path"                  # Custom display name
tags:
  - smoke
  - auth
  - regression
env:
  TEST_EMAIL: ${TEST_EMAIL || "test@example.com"}   # JS default values
  TEST_PASSWORD: ${TEST_PASSWORD || "Test1234!"}
properties:                               # Custom JUnit XML properties
  testCaseId: "TC-101"
  priority: "High"
  jiraTicket: "PROJ-456"
onFlowStart:
  - clearState
  - runScript: scripts/setup.js
onFlowComplete:
  - takeScreenshot: "final-state"
  - runFlow: subflows/teardown.yaml
jsEngine: graaljs                         # graaljs (ES2022) or rhino (ES5, default)
androidWebViewHierarchy: devtools         # Chrome DevTools for WebView inspection
---
# Flow steps begin here
- launchApp
```

### Hook Failure Behavior
- `onFlowStart` fails → main flow body **SKIPPED**, but `onFlowComplete` **STILL RUNS**
- `onFlowComplete` fails → flow marked **FAILED** even if main body passed

---

## 5. launchApp — Critical Details

```yaml
- launchApp:
    appId: "com.example.app"
    clearState: true
    clearKeychain: true             # iOS only, clears ENTIRE keychain
    stopApp: false                  # false = foreground backgrounded app (default: true)
    permissions:
      all: deny                     # overrides default (all: allow)
      notifications: allow
      location: allow
      camera: allow
      photos: unset                 # values: allow, deny, unset
      android.permission.ACCESS_FINE_LOCATION: deny   # Android-specific
    arguments:                      # access in app code
      isMaestro: "true"
      testMode: true
      apiEndpoint: "https://staging.api.com"
```

**Critical**: Permissions default to **ALL ALLOW** even when `clearState: true`. Explicitly deny what you need denied.

---

## 6. JavaScript Integration

### Engine Selection
```yaml
jsEngine: graaljs    # ES2022, recommended. Set in TOP-LEVEL flow only
```
Or: `export MAESTRO_USE_GRAALJS=true`

**GraalJS vs Rhino**: GraalJS isolates variables per script (predictable), supports ES2022, handles special characters. Rhino shares variables (leaky), ES5 only.

### HTTP Requests (custom API — NOT fetch/XMLHttpRequest)
```javascript
// scripts/create-test-user.js
const response = http.post('https://api.example.com/users', {
    headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer ' + API_KEY },
    body: JSON.stringify({ email: 'test-' + Date.now() + '@example.com', password: 'Test1234!' })
})

if (response.ok) {
    const user = json(response.body)
    output.userId = user.id
    output.email = user.email
}

// Also available: http.get(), http.put(), http.delete(), http.request()
// Response: { ok, status, body, headers }
```

### Built-in Variables
- `maestro.copiedText` — last `copyTextFrom` result
- `maestro.platform` — `ios`, `android`, or `web`
- `MAESTRO_FILENAME`, `MAESTRO_DEVICE_UDID`, `MAESTRO_SHARD_ID`, `MAESTRO_SHARD_INDEX`
- All `MAESTRO_*` env vars auto-available in flows

### Page Object Model Pattern
```javascript
// scripts/page-objects.js
output.loginPage = {
    emailField: 'email-input',
    passwordField: 'password-input',
    submitBtn: 'login-button',
    errorMsg: 'error-message'
}

output.homePage = {
    welcomeText: 'welcome-header',
    profileTab: 'tab-profile',
    settingsTab: 'tab-settings'
}
```

```yaml
- runScript: scripts/page-objects.js
- tapOn:
    id: ${output.loginPage.emailField}
- inputText: ${TEST_EMAIL}
- tapOn:
    id: ${output.loginPage.submitBtn}
- assertVisible:
    id: ${output.homePage.welcomeText}
```

---

## 7. Conditions and Flow Control

```yaml
# Platform-specific
- runFlow:
    when:
      platform: Android
    file: subflows/android-permissions.yaml

# Dismiss optional dialogs
- runFlow:
    when:
      visible: "Rate this app"
    commands:
      - tapOn: "Not now"

# JavaScript expression
- runFlow:
    when:
      true: ${IS_FEATURE_ENABLED == 'true'}
    file: subflows/new-feature-test.yaml

# Combined (AND logic — ALL must match)
- runFlow:
    when:
      platform: iOS
      visible: "Allow Notifications"
    commands:
      - tapOn: "Allow"
```

**No native OR logic** — use JavaScript `true:` condition for complex booleans.

---

## 8. Reusable Sub-Flows with Parameters

```yaml
# subflows/login.yaml
appId: ${APP_ID}
env:
  USERNAME: ${USERNAME || "test@example.com"}
  PASSWORD: ${PASSWORD || "Test1234!"}
---
- tapOn:
    id: "email-input"
- inputText: ${USERNAME}
- tapOn:
    id: "password-input"
- inputText:
    text: ${PASSWORD}
    label: "Enter password"       # masks in logs
- tapOn: "Sign In"
- extendedWaitUntil:
    visible: "Home"
    timeout: 10000
```

```yaml
# Calling flow
- runFlow:
    file: subflows/login.yaml
    env:
      USERNAME: "admin@test.com"
      PASSWORD: "AdminPass!"
```

---

## 9. Test Reports & Output

Read `${CLAUDE_PLUGIN_ROOT}/references/qa-workflows.md` for complete report generation details. Summary:

### Report Formats
```bash
maestro test --format junit --output reports/results.xml .maestro/
maestro test --format html --output reports/results.html .maestro/
maestro test --format html-detailed --output reports/detailed.html .maestro/
```

### Custom JUnit Properties (for CI dashboards)
```yaml
appId: com.example.app
properties:
  testCaseId: "TC-101"
  priority: "High"
  jiraTicket: "PROJ-456"
---
```

### Output Directory
```bash
maestro test --test-output-dir=build/maestro-results .maestro/   # custom path
maestro test --flat-output .maestro/                             # no timestamp folders (CI-friendly)
maestro test --debug-output=build/debug .maestro/                # maestro.log + debug data
```

Default: `~/.maestro/tests/<datetime>/` — contains screenshots, video, `commands-*.json`, AI reports.

### AI Test Analysis
```bash
maestro test --analyze .maestro/
```
Generates HTML insights detecting UI regressions, spelling errors, layout breaks, i18n issues.

---

## 10. Maestro MCP Server Integration

```bash
maestro mcp    # starts MCP server (bundled with CLI)
```

**Claude Code / Claude Desktop config:**
```json
{
  "mcpServers": {
    "maestro": {
      "command": "maestro",
      "args": ["mcp"]
    }
  }
}
```

**MCP tools:** `back`, `cheat_sheet`, `check_flow_syntax`, `input_text`, `inspect_view_hierarchy`, `launch_app`, `list_devices`, `query_docs`, `run_flow`, `run_flow_files`, `start_device`, `stop_app`, `take_screenshot`, `tap_on`

---

## 11. CI/CD Integration

### EAS Workflows (Expo / React Native — Recommended)

Expo projects use EAS Workflows with a first-class `type: maestro` job — see `${CLAUDE_PLUGIN_ROOT}/references/react-native.md` and `${CLAUDE_PLUGIN_ROOT}/references/qa-workflows.md` for full config.

```yaml
# .eas/workflows/e2e-test.yml
name: E2E Tests
on:
  pull_request:
    branches: ['*']
jobs:
  build:
    type: build
    params:
      platform: android
      profile: e2e-test
  test:
    needs: [build]
    type: maestro
    params:
      build_id: ${{ needs.build.outputs.build_id }}
      flow_path: '.maestro/'
      include_tags: smoke
      record_screen: true
```

### GitHub Actions (Maestro Cloud)
```yaml
- uses: mobile-dev-inc/action-maestro-cloud@v1
  with:
    api-key: ${{ secrets.MAESTRO_API_KEY }}
    project-id: ${{ secrets.MAESTRO_PROJECT_ID }}
    app-file: app/build/outputs/apk/debug/app-debug.apk
    workspace: .maestro
    include-tags: smoke
    env: |
      TEST_EMAIL=ci@test.com
```

### Local CI
```bash
maestro test --format junit --output build/report.xml --flat-output .maestro/
```

### Tag-Based Execution
```bash
maestro test --include-tags smoke .maestro/
maestro test --exclude-tags wip .maestro/
```

### Sharding
```bash
maestro test --shard-all 3 .maestro/     # ALL tests on ALL 3 devices
maestro test --shard-split 3 .maestro/   # divide tests across 3 devices
```

---

## 12. Workspace Config

```yaml
# .maestro/config.yaml
flows:
  - '**'                         # recursive include all subdirectories

includeTags: []
excludeTags:
  - wip
  - subflow

executionOrder:
  continueOnFailure: false       # stop on first failure (default: true)
  flowsOrder:                    # explicit ordering for dependent flows
    - flows/auth/login.yaml
    - flows/core/create-item.yaml

testOutputDir: build/maestro-results

platform:
  android:
    disableAnimations: true      # disables window/transition animations
  ios:
    disableAnimations: true      # enables Reduce Motion
```

---

## 13. PRD → Test Flow Generation

When the user provides a PRD, spec, or user stories:

1. Parse each acceptance criterion
2. Map to one or more flows (happy path + edge cases per criterion)
3. Group by feature area, tag appropriately
4. Create traceability comment linking to requirement
5. Generate both positive and negative test cases
6. Include setup/teardown hooks for test isolation

```yaml
# Requirement: US-12 — User can reset password via email
# AC: Given a registered user, when they tap "Forgot Password"
#   and enter their email, then they see a confirmation message.
appId: com.example.app
name: "US-12: Password Reset - Happy Path"
tags: [auth, regression]
properties:
  testCaseId: "TC-012"
  requirement: "US-12"
---
```

---

## 14. Debugging

- **`maestro studio`** or **Maestro Studio Desktop** — visual inspector, recorder, AI assistant
- `maestro test --debug-output ./debug` — saves `maestro.log` + screenshots
- `maestro hierarchy` — dump accessibility tree from CLI
- `maestro bugreport` — diagnostic zip for bug reports
- `--continuous` flag — hot-reload testing during development
- `takeScreenshot` with `cropOn:` for targeted visual debugging
- `startRecording` / `stopRecording` for video evidence
- `androidWebViewHierarchy: devtools` in flow header for WebView inspection

### Detect Maestro in App (for test-only behaviors)
```yaml
- launchApp:
    arguments:
      isMaestro: "true"
```
App-side: Android `intent.getStringExtra("isMaestro")`, iOS `ProcessInfo.processInfo.arguments`, Web checks `window.maestro`.

---

## 15. Known Gotchas

- **`clearText` does NOT exist** — use `eraseText`
- **`pasteText`** only pastes from Maestro's internal clipboard, NOT OS clipboard
- **Unicode `inputText`** not supported on Android (ASCII only)
- **iOS `clearState`** reinstalls the entire app (not just clearing data)
- **Permissions default to ALL ALLOW** even with `clearState: true`
- **`launchApp` default `stopApp: true`** — kills and relaunches app every time
- **iOS `accessibilityLabel`** takes precedence over text content for `text:` selector
- **Jetpack Compose `testTag`** requires `testTagsAsResourceId = true` in semantics
- **Flutter `Key` class** is NOT exposed to accessibility layer — use `Semantics(identifier:)`
- **Dollar signs** in selectors need escaping: `\\$150`
- **YAML booleans** — quote `"YES"`, `"NO"`, `"true"`, `"false"`
- **Cloud tests ~45s slower** due to device wipe/recreate overhead
- **`console.log()` in JS** with multiple arguments only prints the first one
- **`runFlow` paths** in Cloud must specify folder, not single file

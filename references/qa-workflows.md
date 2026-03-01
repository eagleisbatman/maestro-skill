# QA Workflows, Reports & CI/CD — Maestro Reference

## 1. Test Report Generation

### Report Formats

Maestro generates reports via the `--format` flag. Available formats:

```bash
maestro test --format junit --output build/report.xml .maestro/
maestro test --format html --output build/report.html .maestro/
maestro test --format html-detailed --output build/detailed.html .maestro/
```

| Format | Output | Use Case |
|---|---|---|
| `junit` | JUnit XML (`.xml`) | CI dashboards: Jenkins, GitHub Actions, Azure DevOps, GitLab |
| `html` | HTML summary page | Quick human review — pass/fail per flow |
| `html-detailed` | Detailed HTML with step-by-step | QA review with individual step outcomes |
| `NOOP` | No report (default) | Local development |

### Custom JUnit Properties

Add traceability metadata to JUnit reports via `properties` in flow header:

```yaml
appId: com.example.app
name: "US-12: Password Reset"
tags: [auth, regression]
properties:
  testCaseId: "TC-012"
  priority: "P1"
  jiraTicket: "PROJ-456"
  requirement: "US-12"
  author: "qa-team"
---
```

This produces `<properties>` elements in the JUnit XML — compatible with:
- **Jenkins** (Test Results Analyzer plugin)
- **Azure DevOps** (Test Management)
- **GitLab** (JUnit report artifact)
- **Allure** (when imported as JUnit)

### Output Directory Structure

**Default location:** `~/.maestro/tests/<datetime>/`

**Customize with:**
```bash
# CLI flag (highest priority)
maestro test --test-output-dir=build/maestro-results .maestro/

# Or in config.yaml (relative to workspace)
# testOutputDir: build/maestro-results
```

**Output directory contents:**
```
build/maestro-results/
├── screenshots/              # takeScreenshot outputs
│   ├── MainScreen.png
│   └── final-state.png
├── recordings/               # startRecording/stopRecording outputs
│   └── login-flow.mp4
├── commands-<flow-name>.json # Step-by-step command execution log
├── ai-<flow-name>.json      # AI assertion results (if used)
├── ai-report-<flow>.html    # AI assertion HTML report (if used)
└── report.xml                # JUnit report (if --format junit)
```

**Flat output for CI** (no timestamp subfolders):
```bash
maestro test --flat-output --test-output-dir=build/results .maestro/
```

**Debug output** (separate from test output):
```bash
maestro test --debug-output=build/debug .maestro/
```
Produces `maestro.log` with full execution trace + `commands-*.json`.

### AI Test Analysis

```bash
maestro test --analyze .maestro/
```

Generates an HTML insights report that detects:
- UI regressions between runs
- Spelling errors in visible text
- Layout breaks (overlapping elements, cut-off text)
- i18n issues
- Visual defects

Requires `maestro login` (free tier works). Output in `~/.maestro/tests/<datetime>/ai-report-*.html`.

---

## 2. Test Suite Organization

### Directory Structure for QA Teams

```
.maestro/
├── config.yaml
├── flows/
│   ├── smoke/                   # Critical path — run on every PR
│   │   ├── login-smoke.yaml     # tags: [smoke]
│   │   └── core-flow-smoke.yaml # tags: [smoke]
│   ├── regression/              # Full regression — nightly
│   │   ├── auth/
│   │   │   ├── login-valid.yaml         # tags: [regression, auth]
│   │   │   ├── login-invalid.yaml       # tags: [regression, auth]
│   │   │   ├── signup.yaml              # tags: [regression, auth]
│   │   │   ├── password-reset.yaml      # tags: [regression, auth]
│   │   │   └── session-persistence.yaml # tags: [regression, auth]
│   │   ├── core/
│   │   │   ├── create-item.yaml         # tags: [regression, core]
│   │   │   ├── edit-item.yaml           # tags: [regression, core]
│   │   │   └── delete-item.yaml         # tags: [regression, core]
│   │   ├── navigation/
│   │   │   └── tab-nav.yaml             # tags: [regression, nav]
│   │   └── edge-cases/
│   │       ├── no-network.yaml          # tags: [regression, edge]
│   │       ├── empty-state.yaml         # tags: [regression, edge]
│   │       ├── orientation-change.yaml  # tags: [regression, edge]
│   │       └── long-text-input.yaml     # tags: [regression, edge]
│   └── visual/                  # Visual regression
│       ├── home-screen.yaml     # tags: [visual]
│       └── profile-screen.yaml  # tags: [visual]
├── subflows/                    # NOT auto-discovered as standalone tests
│   ├── login.yaml
│   ├── navigate-to.yaml
│   └── teardown.yaml
└── scripts/
    ├── page-objects.js
    ├── test-data.js
    └── setup.js
```

### Tag Strategy

```yaml
# Flow header tagging
tags:
  - smoke          # Critical path — every PR
  - regression     # Full suite — nightly
  - auth           # Feature area
  - P1             # Priority
```

```bash
# PR pipeline — fast (smoke only)
maestro test --include-tags smoke .maestro/

# Nightly pipeline — full regression
maestro test --include-tags regression .maestro/

# Feature-specific
maestro test --include-tags auth .maestro/

# Exclude WIP and subflows
maestro test --exclude-tags wip,subflow .maestro/

# Combine include + exclude
maestro test --include-tags regression --exclude-tags flaky .maestro/
```

Tags are **flat strings** — no hierarchical support. `includeTags` in config.yaml forms a union with CLI `--include-tags`.

### Execution Ordering

```yaml
# config.yaml
executionOrder:
  continueOnFailure: false    # STOP on first failure (default: true)
  flowsOrder:                 # Explicit ordering for dependent flows
    - flows/smoke/login-smoke.yaml
    - flows/regression/auth/signup.yaml
    - flows/regression/core/create-item.yaml
```

Flows NOT listed in `flowsOrder` run AFTER the ordered flows.

---

## 3. CI/CD Integration

### GitHub Actions (Maestro Cloud)

```yaml
name: E2E Tests
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  smoke-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: ./gradlew assembleDebug
      - uses: mobile-dev-inc/action-maestro-cloud@v1
        id: maestro
        with:
          api-key: ${{ secrets.MAESTRO_API_KEY }}
          project-id: ${{ secrets.MAESTRO_PROJECT_ID }}
          app-file: app/build/outputs/apk/debug/app-debug.apk
          workspace: .maestro
          include-tags: smoke
          exclude-tags: wip
          device-locale: en_US
          env: |
            TEST_EMAIL=ci-test@example.com
            TEST_PASSWORD=${{ secrets.TEST_PASSWORD }}
          timeout: 30

      # Access outputs
      - run: echo "Console URL -> ${{ steps.maestro.outputs.MAESTRO_CLOUD_CONSOLE_URL }}"
```

**Action inputs:** `api-key` (required), `project-id` (required), `app-file` (required unless `app-binary-id`), `app-binary-id`, `async`, `env`, `exclude-tags`, `include-tags`, `name`, `timeout`, `workspace`, `android-api-level`, `device-model`, `device-os`, `device-locale`, `mapping-file`.

**Action outputs:** `MAESTRO_CLOUD_CONSOLE_URL`, `MAESTRO_CLOUD_UPLOAD_STATUS`, `MAESTRO_CLOUD_FLOW_RESULTS`, `MAESTRO_CLOUD_APP_BINARY_ID`.

**Reuse binary across jobs:**
```yaml
app-binary-id: ${{ steps.upload.outputs.MAESTRO_CLOUD_APP_BINARY_ID }}
```

### GitHub Actions (Local CI — Self-Hosted)

```yaml
name: Local E2E
on: [pull_request]
jobs:
  test:
    runs-on: macos-latest   # or self-hosted with emulator
    steps:
      - uses: actions/checkout@v3
      - name: Install Maestro
        run: curl -fsSL "https://get.maestro.mobile.dev" | bash
      - name: Start emulator
        run: |
          $ANDROID_HOME/emulator/emulator -avd test_device -no-window -no-audio &
          adb wait-for-device
      - name: Build app
        run: ./gradlew assembleDebug && adb install app/build/outputs/apk/debug/app-debug.apk
      - name: Run tests
        run: |
          export PATH="$PATH:$HOME/.maestro/bin"
          maestro test --format junit --output build/report.xml --flat-output .maestro/
      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: maestro-report
          path: build/report.xml
      - name: Publish test results
        if: always()
        uses: dorny/test-reporter@v1
        with:
          name: Maestro Tests
          path: build/report.xml
          reporter: java-junit
```

### EAS Workflows (Expo Projects)

Expo projects can use EAS Workflows with a first-class `type: maestro` pre-packaged job — zero CI scripting needed.

**Build profile** (`eas.json`):
```json
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

**Workflow** (`.eas/workflows/e2e-test.yml`):
```yaml
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
      shards: 2
      retries: 1
      record_screen: true
      maestro_version: '1.38.0'    # pin for stability
```

**Full `type: maestro` params:** `build_id` (required), `flow_path` (required), `shards`, `retries`, `record_screen`, `include_tags`, `exclude_tags`, `maestro_version`, `android_system_image_package`, `device_identifier`.

**Save artifacts** with `MAESTRO_TESTS_DIR`:
```yaml
- takeScreenshot: ${MAESTRO_TESTS_DIR}/after-login
- startRecording: ${MAESTRO_TESTS_DIR}/checkout-flow
```

**Maestro Cloud** alternative: `type: maestro-cloud` with `maestro_project_id` and `flows` params. Requires Maestro Cloud subscription.

```bash
# Trigger manually
npx eas-cli@latest workflow:run .eas/workflows/e2e-test.yml
```

### Generic CI (Jenkins, GitLab, Bitbucket)

```bash
#!/bin/bash
set -e

# Install Maestro
curl -fsSL "https://get.maestro.mobile.dev" | bash
export PATH="$PATH:$HOME/.maestro/bin"

# Run with JUnit output for dashboard integration
maestro test \
  --format junit \
  --output build/report.xml \
  --flat-output \
  --include-tags smoke \
  .maestro/

# Exit code: 0 = all pass, 1 = any failure
```

### Sharding / Parallel Execution

```bash
# Run ALL tests on ALL devices (full suite on each)
maestro test --shard-all 3 .maestro/

# SPLIT tests across devices (each device gets a subset)
maestro test --shard-split 3 .maestro/

# Specify devices explicitly
maestro test --device "emulator-5554,emulator-5556" --shard-split 2 .maestro/
```

Use `MAESTRO_SHARD_INDEX` in screenshot names to avoid filename collisions:
```yaml
- takeScreenshot: "login-shard-${MAESTRO_SHARD_INDEX}"
```

---

## 4. Visual Regression Testing

### Screenshot-Based Regression

```yaml
# Capture baseline
- takeScreenshot: "home-screen"

# Crop to specific UI container
- takeScreenshot:
    path: "login-form"
    cropOn:
      id: "login-form-container"

# Compare against baseline (assertScreenshot)
- assertScreenshot: "home-screen"
```

### AI Visual Assertions

```yaml
# Natural language visual verification
- assertWithAI: "the product card shows an image, title, price, and add-to-cart button"
- assertWithAI: "the navigation bar has exactly 4 tabs"
- assertWithAI: "no text is cut off or overlapping"

# Auto-detect all visual defects
- assertNoDefectsWithAI
```

Both require `maestro login` (free tier). Output saved as HTML + JSON in test output directory.

### Video Evidence

```yaml
- startRecording: "checkout-flow"
# ... test steps ...
- stopRecording

# Video saved as checkout-flow.mp4 in test output
```

---

## 5. Test Data Management

### JavaScript API Calls for Setup/Teardown

```javascript
// scripts/setup.js — create test user via API before flow
const response = http.post('https://api.staging.example.com/test-users', {
    headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer ' + API_KEY
    },
    body: JSON.stringify({
        email: 'test-' + Date.now() + '@example.com',
        password: 'TestPass123!'
    })
})

if (response.ok) {
    const user = json(response.body)
    output.testEmail = user.email
    output.testPassword = 'TestPass123!'
    output.userId = user.id
} else {
    throw new Error('Setup failed: ' + response.status)
}
```

```yaml
# In flow
- runScript: scripts/setup.js
- tapOn:
    id: "email-input"
- inputText: ${output.testEmail}
```

### Teardown Pattern

```javascript
// scripts/teardown.js — clean up test data
if (output.userId) {
    http.delete('https://api.staging.example.com/test-users/' + output.userId, {
        headers: { 'Authorization': 'Bearer ' + API_KEY }
    })
}
```

```yaml
# In flow header
onFlowComplete:
  - runScript: scripts/teardown.js
```

`onFlowComplete` runs even if the main flow fails — guaranteed cleanup.

### Random Data Generation

```yaml
- inputRandomEmail                   # random email
- inputRandomPersonName              # random full name
- inputRandomNumber                  # 8 random digits
- inputRandomNumber:
    length: 10                       # custom length
- inputRandomText                    # 8 random chars
- inputRandomText:
    length: 20                       # custom length

# Capture random value for reuse
- copyTextFrom:
    id: "email-field"
# Now available as ${maestro.copiedText}
```

---

## 6. Flaky Test Handling

### Built-in Tolerance
Maestro auto-waits ~7 seconds for `assertVisible`/`assertNotVisible`. Additional tools:

### Explicit Waits
```yaml
# Wait for async operations (up to 15 seconds)
- extendedWaitUntil:
    visible: "Dashboard loaded"
    timeout: 15000

# Wait for animations to complete
- waitForAnimationToEnd:
    timeout: 5000

# Wait for element to become interactive
- tapOn:
    id: "submit-btn"
    enabled: true           # auto-waits for enabled state
```

### Retry Mechanism
```yaml
- retry:
    maxRetries: 3           # 0-3 (default: 1)
    commands:
      - tapOn: "Submit"
      - assertVisible: "Success"
```

**Anti-pattern**: Don't wrap entire flows in `retry` — it hides genuine bugs. Only retry specific flaky interactions.

### Auto-Retry on Tap
```yaml
- tapOn:
    id: "submit"
    retryTapIfNoChange: true     # retry tap if screen didn't change
    waitToSettleTimeoutMs: 500   # max wait for screen to settle
```

### Dismiss Optional UI
```yaml
- runFlow:
    when:
      visible: "Rate this app"
    commands:
      - tapOn: "Not now"

- tapOn:
    text: "Cookie banner dismiss"
    optional: true           # continue if not present
```

---

## 7. Page Object Model Pattern

### Define Selectors in JS

```javascript
// scripts/page-objects.js
output.loginPage = {
    emailField: 'email-input',
    passwordField: 'password-input',
    submitBtn: 'login-button',
    errorMsg: 'error-message',
    forgotPasswordLink: 'forgot-password'
}

output.homePage = {
    welcomeText: 'welcome-header',
    searchBar: 'search-input',
    profileTab: 'tab-profile',
    settingsTab: 'tab-settings',
    notificationBadge: 'notification-badge'
}

output.settingsPage = {
    logoutBtn: 'logout-button',
    themeToggle: 'theme-toggle',
    languagePicker: 'language-picker'
}
```

### Use in Flows

```yaml
- runScript: scripts/page-objects.js

- tapOn:
    id: ${output.loginPage.emailField}
- inputText: ${TEST_EMAIL}

- tapOn:
    id: ${output.loginPage.passwordField}
- inputText:
    text: ${TEST_PASSWORD}
    label: "password"

- tapOn:
    id: ${output.loginPage.submitBtn}

- assertVisible:
    id: ${output.homePage.welcomeText}
```

### Centralized Loader Sub-Flow

```yaml
# subflows/load-pages.yaml
appId: ${APP_ID}
---
- runScript: scripts/page-objects.js
```

```yaml
# In every test flow
- runFlow: subflows/load-pages.yaml
```

---

## 8. Cross-Platform Test Strategy

### Platform Conditions

```yaml
# Skip Android-only steps on iOS
- runFlow:
    when:
      platform: Android
    commands:
      - pressKey: back    # back key only on Android

# iOS swipe-back instead
- runFlow:
    when:
      platform: iOS
    commands:
      - swipe:
          direction: RIGHT
          start: 0%, 50%
          end: 50%, 50%
```

### Detect Platform in JS

```javascript
// scripts/platform-check.js
if (maestro.platform === 'android') {
    output.backAction = 'pressKey'
} else {
    output.backAction = 'swipe'
}
```

### Shared + Platform-Specific Flows

```
.maestro/
├── flows/
│   ├── auth/
│   │   ├── login.yaml              # Shared (uses conditions internally)
│   │   ├── login-android-only.yaml # tags: [android]
│   │   └── login-ios-only.yaml     # tags: [ios]
```

---

## 9. Accessibility Testing with Maestro

Maestro interacts through the **accessibility tree by default** — tests inherently validate accessibility:
- Elements invisible to assistive technology are invisible to Maestro
- If `tapOn: id: "my-button"` works, the button has a valid accessibility identifier
- If text is readable by Maestro, it's readable by screen readers

### AI Accessibility Checks

```yaml
# Detect visual accessibility issues
- assertNoDefectsWithAI

# Specific accessibility assertions
- assertWithAI: "all buttons have visible text labels"
- assertWithAI: "text contrast is sufficient for readability"
- assertWithAI: "touch targets appear to be at least 44x44 points"
```

---

## 10. Maestro Studio for QA

### Desktop App (Recommended)

Maestro Studio Desktop is an all-in-one GUI:
- **Element Inspector** — click any element to see its accessibility properties
- **YAML Flow Builder** — build flows visually with drag-and-drop
- **AI Assistant (MaestroGPT)** — describe what you want, generates YAML
- **Flow Runner** — execute flows with real-time step visualization
- **Workspace Manager** — organize and run test suites

### CLI Studio

```bash
maestro studio                           # default — opens in browser
maestro studio -p web                    # web testing mode
maestro --device emulator-5554 studio    # specific device
```

### Workflow for QA Engineers

1. Open Maestro Studio Desktop
2. Connect to device/emulator
3. Click **Inspect Screen** to explore accessibility tree
4. Click elements to generate selector YAML
5. Build flow step-by-step in the editor
6. Run and iterate
7. Save to `.maestro/flows/` directory

---

## 11. Debugging Failed Tests

### Diagnostic Tools

```bash
# Full debug output
maestro test --debug-output ./debug flow.yaml
# Produces: maestro.log, commands-*.json, screenshots at each step

# Dump accessibility tree
maestro hierarchy

# Bug report zip (for filing issues)
maestro bugreport

# Hot-reload testing (re-runs on file save)
maestro test --continuous flow.yaml
```

### In-Flow Debugging

```yaml
# Take screenshot at specific point
- takeScreenshot: "debug-before-submit"

# Record video of entire flow
- startRecording: "debug-session"
# ... steps ...
- stopRecording

# Crop screenshot to specific element
- takeScreenshot:
    path: "debug-form"
    cropOn:
      id: "form-container"
```

### Common Failure Patterns

| Symptom | Cause | Fix |
|---|---|---|
| Element not found | Wrong selector / not in accessibility tree | Use `maestro studio` to inspect, verify `id:` exists |
| Tap doesn't work | Element not interactive yet | Add `enabled: true` to selector or `extendedWaitUntil` |
| Text input ignored | No field focused | Add explicit `tapOn` on field before `inputText` |
| Flaky on CI, works locally | Animation timing / device speed | Add `waitForAnimationToEnd`, increase `extendedWaitUntil` timeout |
| WebView elements invisible | Accessibility not exposed | Add `androidWebViewHierarchy: devtools` to flow header |
| `eraseText` doesn't clear all | Default only erases 50 chars | Use `eraseText: 200` with larger count |
| iOS keyboard won't hide | `hideKeyboard` flaky on iOS | Tap a non-interactive element (header, background) |
| Screenshot names collide in sharding | Multiple devices write same file | Append `${MAESTRO_SHARD_INDEX}` to screenshot names |

---

## 12. Environment Variables Reference

| Variable | Default | Purpose |
|---|---|---|
| `MAESTRO_DRIVER_STARTUP_TIMEOUT` | 15000 (Android) / 120000 (iOS) | Driver startup wait (ms) |
| `MAESTRO_USE_GRAALJS` | false | Enable GraalJS engine (ES2022) |
| `MAESTRO_CLOUD_API_KEY` | — | Cloud authentication |
| `MAESTRO_CLI_NO_ANALYTICS` | false | Disable telemetry |
| `MAESTRO_DISABLE_UPDATE_CHECK` | false | Skip version check |
| `MAESTRO_CLI_ANALYSIS_NOTIFICATION_DISABLED` | false | Suppress AI analysis notification |
| `MAESTRO_VERSION` | latest | Pin specific version during install |

All `MAESTRO_*` env vars are auto-available in flows as JavaScript variables (CLI only, not Studio).

### Rollback to Specific Version

```bash
export MAESTRO_VERSION=1.37.0
curl -fsSL "https://get.maestro.mobile.dev" | bash
```

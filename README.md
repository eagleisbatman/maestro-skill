# Maestro Testing — Claude Code Plugin

A Claude Code plugin that generates production-ready [Maestro](https://maestro.mobile.dev/) YAML test flows for mobile and web apps.

## Installation

```bash
claude plugin add /path/to/maestro-skill
# or from Git
claude plugin add https://github.com/YOUR_USERNAME/maestro-skill.git
```

## What It Does

When triggered, this skill enables Claude to:

- Generate Maestro YAML test flows with correct selectors for your framework
- Set up project structure (`.maestro/` directory, config, subflows, scripts)
- Create test flows from PRDs, specs, or user stories
- Configure CI/CD pipelines (GitHub Actions, EAS Workflows, Maestro Cloud)
- Generate test reports (JUnit, HTML, AI analysis)
- Debug test failures with the right tools and patterns

## Supported Frameworks

| Framework | Selectors | Reference |
|---|---|---|
| React Native / Expo | `testID` → `id:` | `references/react-native.md` |
| Flutter | `Semantics(identifier:)` → `id:` | `references/flutter.md` |
| Native Android (Kotlin/Compose) | `testTag` / `resource-id` → `id:` | `references/native-android.md` |
| Native iOS (Swift/SwiftUI/UIKit) | `accessibilityIdentifier` → `id:` | `references/native-ios.md` |
| Hybrid (Capacitor/Ionic/Cordova) | `aria-label` / text → `text:` | `references/hybrid.md` |
| Web (Desktop Chromium) | CSS selectors / text | Built-in |

## Trigger Phrases

The skill activates when you mention:

- Mobile testing, UI testing, E2E testing, Maestro flows
- "Write tests", "generate test cases", "automate my app"
- Any supported framework name + testing context
- PRD/spec/user story with acceptance criteria needing automation
- Test reports, visual regression, CI/CD for mobile tests

## Plugin Structure

```
maestro-skill/
├── plugin.json              # Plugin manifest
├── skills/
│   └── maestro-testing.md   # Main skill (YAML frontmatter + content)
└── references/
    ├── react-native.md      # React Native / Expo specifics
    ├── flutter.md           # Flutter specifics
    ├── native-android.md    # Native Android specifics
    ├── native-ios.md        # Native iOS specifics
    ├── hybrid.md            # Capacitor / Ionic / Cordova / PWA
    └── qa-workflows.md      # QA workflows, reports, CI/CD
```

## Prerequisites

Maestro requires **Java 17+** and installs with:

```bash
curl -fsSL "https://get.maestro.mobile.dev" | bash
```

## License

MIT

# AGENTS.md

## Purpose

Local-first Linux tray app and CLI for monitoring usage, reset windows, spend, and status across coding tools and AI services. Reads local provider CLIs, config files, browser cookies, and provider usage endpoints only for enabled features; never sends data to a backend.

## Stack

Swift 6.2+ package (SwiftPM). Targets: `AgentBar` (GTK/AppIndicator tray executable), `AgentBarCLI` (CLI), `AgentBarCore` (shared logic), `AgentBarMacros`/`AgentBarMacroSupport` (swift-syntax macro plugin). C shims `CAgentBarTray`/`CAgentBarTrayShim` wrap Ayatana AppIndicator. Vendored deps: `Vendor/Commander`, `Vendor/SweetCookieKit`.

## Build / Run / Test

```bash
swift build
swift test
swift run AgentBar tray            # tray app
swift run AgentBarCLI --help       # CLI
./Scripts/compile_and_run.sh       # build + test + run AgentBar (passes args through)
./Scripts/lint.sh lint             # swiftformat --lint + swiftlint --strict
./Scripts/lint.sh format           # apply swiftformat
```

npm scripts wrap the same: `npm run build`, `npm test`, `npm run lint`, `npm start`.

Lint/format tools are pinned and auto-installed into `.build/lint-tools/bin` by `Scripts/install_lint_tools.sh` (invoked from `lint.sh`); do not rely on system-wide swiftformat/swiftlint.

## Layout

- `Sources/AgentBarCore/Providers/` — per-provider adapters (Codex, Claude, OpenCode, Gemini, Copilot, etc.), each in its own subdir; shared contracts in `ProviderDescriptor.swift`, `Providers.swift`, `ProviderFetchPlan.swift`, `ProviderTokenResolver.swift`.
- `Sources/AgentBarCore/Config/` — `AgentBarConfig*.swift`; config lives at `~/.agentbar/config.json` (0600).
- `Sources/AgentBarCore/` — cookie/keychain access gates, cost scanners, usage fetchers, redaction (`PersonalInfoRedactor.swift`), tray refresh policy.
- `Sources/AgentBar/` — tray executable: `main.swift`, `GNOMETrayHost.swift`, `UsagePanelController.swift`, `TrayIconRenderer.swift`, notifications/alerts.
- `Sources/AgentBarCLI/` — CLI commands (`usage`, `cost`, `config`) and rendering.
- `Sources/CAgentBarTray*` — C interop for AppIndicator.
- `Tests/AgentBarTests/` — uses swift-testing (SwiftTesting experimental feature enabled).
- `Scripts/` — `compile_and_run.sh`, `lint.sh`, `install_lint_tools.sh`, `swift_runtime_env.sh`.
- `bin/` — installers: launcher, autostart, CLI helper.

## Gotchas

- Tray support is compile-time conditional. `Package.swift` runs `pkg-config --exists ayatana-appindicator3-0.1 gtk+-3.0`; if missing, the `CAgentBarTray*` targets and `GNOMETrayHost.swift`/`UsagePanelController.swift` are excluded and `AgentBar` builds tray-less. Install `libgtk-3-dev` + `libayatana-appindicator3-dev` for a real tray build.
- `Scripts/swift_runtime_env.sh` exports SourceKit/runtime paths for a swiftly toolchain at `~/.local/share/swiftly/toolchains/6.3.1/usr` and `~/.local/compat`. `compile_and_run.sh` and `lint.sh` source it; raw `swift build` may behave differently without it.
- Tray needs `org.kde.StatusNotifierWatcher` (GNOME AppIndicator extension); requires a desktop session. Debug env vars: `AGENTBAR_FORCE_TRAY=1`, `AGENTBAR_REFRESH_SECONDS` (<15 falls back to 60), `AGENTBAR_STATUS_NOTIFIER_WAIT_SECONDS`.
- The CLI target is `AgentBarCLI`, but the installed CLI binary is `agentbar-cli` (the tray binary takes `agentbar`).
- Files holding tokens/credentials must stay 0600; logs are redacted via `PersonalInfoRedactor`. Keep that posture when adding provider adapters.

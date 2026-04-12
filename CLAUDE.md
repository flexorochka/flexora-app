# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

Hiddify is a multi-platform proxy client (Android, iOS, Windows, macOS, Linux) built with Flutter. It uses [hiddify-core](https://github.com/hiddify/hiddify-core) (a Go-based Sing-box wrapper) as its proxy engine, exposed to Flutter via FFI. The app supports protocols including Vless, Vmess, Reality, TUIC, Hysteria, Wireguard, SSH, and subscription formats (V2ray, Clash, Clash Meta).

## Commands

```bash
# Install dependencies
flutter pub get          # or: make get

# Code generation (run after modifying models, providers, database schema, or translations)
make gen                 # dart run build_runner build --delete-conflicting-outputs
make translate           # dart run slang (for translation changes)

# Run & test
flutter run              # debug run
flutter test             # run all tests
flutter test test/path/to/test.dart  # run a single test file

# Platform builds
make android-apk-release
make windows-release
make linux-release
make macos-release
make ios-release

# One-time platform setup (downloads native libs, configures tooling)
make android-prepare
make windows-prepare
make linux-prepare
make macos-prepare
make ios-prepare
```

**Formatter:** line width is 120 characters (`formatter.page_width: 120` in `analysis_options.yaml`).

## Architecture

### Directory Layout

- `lib/core/` — Shared infrastructure: DB (Drift/SQLite), routing (GoRouter), theming, preferences, logging, HTTP client, localization, analytics, shared models, and reusable widgets.
- `lib/features/` — Feature modules (connection, profile, proxy, settings, home, stats, log, route_rules, per_app_proxy, system_tray, window, app_update, etc.). Each feature is self-contained with its own providers, widgets, and data layer.
- `lib/hiddifycore/` — Dart FFI bindings to the hiddify-core native library. Generated bindings live in `hiddify_core_generated_bindings.dart`.
- `lib/utils/` — Cross-cutting utilities: async mutation helpers, validators, platform detection, Sentry integration.
- `lib/gen/` — Auto-generated files (translations via Slang, asset references via flutter_gen). Do not edit manually.
- `assets/translations/` — Slang i18n JSON source files (19+ languages).
- `hiddify-core/` — Git submodule for the Go proxy engine.
- `test/` — Flutter unit/integration tests.

### State Management

Riverpod is used throughout (~154 providers). Prefer the `@riverpod` annotation style (code generation). Providers are defined at the feature level and composed via dependencies. `RiverpodObserver` is wired in for debugging.

### Data Layer

- **Drift** (type-safe SQLite ORM) for persistent data. Schema versions and migrations live in `lib/core/db/schemas/`. After modifying a table, add a migration step and bump `schemaVersion`.
- **SharedPreferences** (via `lib/core/preferences/`) for simple key-value settings, with a migration system for schema changes.
- **Freezed** models are used for all immutable data classes. Always regenerate after modifying a `@freezed` class.

### Code Generation

The project heavily uses build_runner. The generated files are committed. After any change to:
- `@freezed` data classes → `make gen`
- `@Riverpod` annotated providers → `make gen`
- Drift table/DAO definitions → `make gen`
- `@JsonSerializable` models → `make gen`
- Translation JSON files → `make translate`

### Platform-Specific Code

- **Android**: Per-app proxy via TUN mode; native AAR from hiddify-core releases.
- **iOS**: `HiddifyPacketTunnel` extension (`ios/HiddifyPacketTunnel/`) for background VPN.
- **Desktop** (Windows/macOS/Linux): Window manager, system tray, auto-start. Native shared libraries downloaded during `make <platform>-prepare`.

### Environment & Entry Points

- `lib/main.dart` — dev entry point.
- `lib/main_prod.dart` — production entry point (adds Sentry).
- `lib/bootstrap.dart` — Lazy app initialization (logs, prefs, analytics, profile, core).
- Environment flags passed at build time: `sentry_dsn`, `portable` (Windows), `release` (`general` or `google-play`), `CHANNEL` (`dev`/`prod`).

### Linting

Uses `package:lint/strict.yaml` with `custom_lint` enabled. The `provider_parameters` custom rule enforces Riverpod best practices. Analyzer excludes `hiddify-core/**`, `**.g.dart`, and `lib/gen/**`.

---

## Project Context

This is a fork of [hiddify/hiddify-app](https://github.com/hiddify/hiddify-app) under the **flexorochka** organization, branded as **Flexora** — a VPN client targeting Russian-speaking users.

- Canonical upstream: `hiddify/hiddify-app`; local remote is named `upstream`
- VLESS+Reality is already supported via upstream; **AmneziaWG (AWG)** needs to be added
- Target platforms by priority: **macOS arm64**, **Android**, **iOS**, then others

## Architecture Map

### App Initialization (`lib/bootstrap.dart`)

`lazyBootstrap()` runs sequentially at startup:
1. Preserve splash screen, pre-init logger
2. Create root `ProviderContainer` with `environmentProvider` override
3. Init directories → logger file path → app info → SharedPreferences → preferences migration
4. Desktop-only: window controller → silent-start check → auto-start service
5. Log repository → logger post-init → profile repository → translations → active profile (with 1 s timeout)
6. `hiddifyCoreService.init()` — starts native core and connects gRPC clients
7. Desktop-only: system tray; Android-only: high refresh rate
8. `runApp()` with `ProviderScope` + `RiverpodObserver` + `SentryUserInteractionWidget`

### Features (`lib/features/`)

| Feature | Description |
|---|---|
| `app` | Root `App` widget, Material/Material You theme wiring |
| `connection` | VPN connect/disconnect lifecycle, `ConnectionNotifier` (StreamNotifier) |
| `profile` | Subscription profiles: add, edit, parse, sync; active profile selection |
| `proxy` | Proxy node list, outbound selection, proxy detail UI |
| `home` | Main dashboard screen |
| `settings` | User preferences: theme, language, core options, TUN settings |
| `stats` | Real-time traffic/speed stats from core gRPC stream |
| `log` | In-app log viewer sourced from core gRPC log stream |
| `route_rules` | Routing rules configuration (bypass, block, proxy) |
| `per_app_proxy` | Android per-app TUN proxy selection |
| `app_update` | Version check and update prompts |
| `intro` | First-launch onboarding screens |
| `auto_start` | Desktop auto-start on login |
| `system_tray` | Desktop system tray icon and menu |
| `window` | Desktop window state management |
| `deep_link` | Incoming URI / deep-link handler |
| `shortcut` | Keyboard shortcut wrapper |
| `platform_specific` | Android quick-settings tile |
| `about` | About screen |
| `common` | Shared widgets: QR scanner, QR dialog, text scroll |

### Dart ↔ hiddify-core Communication

The bridge uses **FFI + gRPC together**:

- **Desktop** (`CoreInterfaceDesktop`): FFI loads `hiddify-core.dylib/.dll/.so` via `DynamicLibrary.open`. A single `_box.setup(...)` FFI call starts an in-process gRPC server on `127.0.0.1:17078`. All subsequent commands (start, stop, parse, outbound selection, logs, stats) go through the **gRPC** `CoreClient` (`lib/hiddifycore/generated/v2/hcore/hcore_service.pbgrpc.dart`).
- **Mobile** (`CoreInterfaceMobile`): A `MethodChannel` (`com.hiddify.app/method`) starts the native background service. Two gRPC clients are created — **fg** (foreground, port 17078) and **bg** (background, port 17079). Status and alert events arrive via `EventChannel`.
- All business logic in `HiddifyCoreService` (`lib/hiddifycore/hiddify_core_service.dart`) uses only the gRPC clients, regardless of platform.
- Proto definitions live in `lib/hiddifycore/generated/v2/` (hcore, hcommon, hiddifyoptions, profile, config, hello).

### Protocol / Outbound Configuration

- Proxy type enum: `lib/singbox/model/singbox_proxy_type.dart` — lists every supported protocol including `awg("AWG")` (already declared).
- Outbound group model: `lib/singbox/model/singbox_outbound.dart`
- Core options (TUN, DNS, mux, etc.): `lib/singbox/model/singbox_config_option.dart`
- **To integrate AmneziaWG fully**: the `awg` enum value exists in Dart, but the actual sing-box config generation lives inside `hiddify-core` (Go). AWG requires either forking `hiddify-core` to add AWG support to sing-box, or patching sing-box upstream. The Dart side will then surface AWG nodes automatically via the existing `ProxyType.awg` path.

### Profile & Subscription UI

- `lib/features/profile/` — add/import, edit, details, overview screens
- `lib/features/profile/data/` — repository and data providers for subscription fetching and parsing
- `lib/features/profile/model/` — profile data models (Freezed + JSON)
- Pre-shared Flexora subscription profiles should be injected here (either as default profile URLs in preferences or as hardcoded fallback profiles in the repository layer)

## Build Commands

```bash
# macOS (priority platform)
make macos-prepare && flutter run -d macos

# Android
make android-prepare && flutter run -d <device-id>

# iOS
make ios-prepare && flutter run -d <device-id>

# Code generation (after modifying any model, provider, DB schema)
make gen

# Translations (after editing assets/translations/*.i18n.json)
make translate
```

## Flexora-Specific TODO

- [ ] Rebranding: package name, app icons, splash screen, app display name
- [ ] Default locale — Russian (`ru`)
- [ ] AmneziaWG integration (requires forking `hiddify-core` or patching sing-box)
- [ ] Pre-shared Flexora subscription profiles
- [ ] CI/CD setup for the `flexorochka` GitHub organization

## Upstream Sync Strategy

```bash
git fetch upstream && git merge upstream/main
```

All Flexora-specific changes must be in dedicated commits prefixed with `flexora:` to simplify future rebases onto upstream updates.

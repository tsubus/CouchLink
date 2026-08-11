# Project Context

## Stack

- **Android remote:** Kotlin 2.2.10, Android Gradle Plugin 8.11.1, Java 17, Android SDK 28–36.
- **Android UI and concurrency:** Jetpack Compose Material 3, `StateFlow`, and `kotlinx-coroutines-android` 1.10.2.
- **Android protocols/security:** protobuf-lite 4.31.1 for Google TV Remote v2 messages; Bouncy Castle 1.78.1 for its TLS identity and certificate flow.
- **Windows host:** C# on .NET 8. The desktop app is WPF/WinForms (`net8.0-windows`); shared protocol and release-verification libraries target `net8.0`.
- **Windows platform integration:** built-in networking, JSON, cryptography, WPF, WinForms tray APIs, and direct Core Audio COM interop. No NuGet packages are declared by the Windows projects.
- **Release tooling:** PowerShell builds/signs Android and Windows artifacts and produces a source manifest. Android and Windows clients independently check GitHub Releases and verify downloaded artifacts.

## Architecture

### Runtime entry points

- `src/android/.../MainActivity.kt` is the Android entry point. Its one Compose root creates or retrieves the Bluetooth HID, Windows-host, TV, and update runtimes; collects their `StateFlow`s; and routes callbacks into the feature screens.
- `src/windows/CouchLink.Host.Wpf/App.xaml.cs` composes preferences, tray UI, and `CouchLinkHostRuntime`, starts the runtime, then opens the dashboard or remains tray-only.
- `src/windows/CouchLink.Updater/Program.cs` is a separate executable so the running host can be stopped, replaced, restored from backup on failure, and restarted.

### Boundaries and data flow

The product deliberately has three independent local-control paths:

1. **Bluetooth HID:** Android `hid/` sends keyboard, mouse, touchpad, media, and shortcut reports through Android's Bluetooth HID APIs. It does not depend on the Windows host and is the sign-in input path.
2. **Windows launcher host:** Android `host/LauncherHostClient` discovers hosts over UDP `45820`, uses length-prefixed JSON over TCP `45821`, then maintains a trusted session with heartbeats. The Windows host coordinates discovery, pairing, launcher/audio operations, and client sessions in `CouchLink.Host.Core`; `CouchLink.Protocol` owns protocol-v1 records and camel-case JSON envelopes.
3. **Google TV:** Android `tv/` discovers Remote v2 services with Android NSD, pairs with a six-character code, stores a client TLS identity, pins the remote certificate, and controls the TV over TLS on port `6466`. It can send Wake-on-LAN packets for a remembered TV.

The Android update manager downloads the expected GitHub APK asset, validates SHA-256, then validates package identity and signing certificate before handing the user-initiated install to Android. The Windows update service validates exact asset names and GitHub download URLs, SHA-256 values, and detached ECDSA signatures before launching the isolated updater.

This is **feature-oriented rather than layered**: Android packages are `hid`, `host`, `tv`, `update`, and `ui`; Windows projects split desktop UI, runtime core, protocol contracts, updater, and release security. There is no controller/service/repository or ORM/API-server layer.

## Conventions (Observed)

### Error handling and state

- Kotlin uses `runCatching`, `onSuccess`/`onFailure`, and coroutine cancellation rethrowing where connection work runs. Recoverable failures are generally translated into user-visible `StateFlow` messages.
- C# uses guard clauses, explicit `try`/`catch`, returned result records for device operations, and an event-sink callback for runtime diagnostics. Protocol validation returns an `error` envelope for invalid, unsupported, or mismatched messages.
- Both connection clients protect against stale connection attempts: Android host and TV clients associate state cleanup/reporting with their owned/current socket attempt.
- Trust is explicit. The Windows host creates a six-digit pairing code, issues a random 32-byte persistent token, and compares token bytes with `CryptographicOperations.FixedTimeEquals`. Unauthenticated sessions cannot invoke launcher or audio commands.

### API and persistence shape

- Windows-host traffic is custom RPC-like framed TCP, not REST or GraphQL. Envelopes use `protocolVersion`, UUID `messageId`, `type`, timestamp, and a `payload`; `System.Text.Json` is camel case and omits null values.
- Android decodes the host protocol dynamically with `JSONObject`; the protocol contracts are authoritative in C# records rather than a shared generated schema.
- Android persists small trusted-host, TV, and UI preferences in `SharedPreferences`. Windows persists trusted devices in LocalAppData JSON and best-effort mirrors them to CommonApplicationData.
- No database, backend service, account system, cloud relay, analytics, or telemetry was found.

### Type safety and observability

- Kotlin code is strongly typed with internal data classes, enums, and typed protobuf messages. C# enables nullable reference types in all inspected projects and uses records for protocol and result values.
- Low-level boundaries necessarily use dynamic JSON (`JSONObject`/`JsonElement`) and COM interop. No `unsafe` C# blocks were found.
- Operational feedback is primarily UI state and host event strings. No central structured logging, metrics exporter, or HTTP health endpoint was found; this is a desktop/local-network application, not a server process.

### Testing

- The Windows solution has three executable, assertion-style test projects: `CouchLink.ReleaseSecurity.Tests`, `CouchLink.Updater.Tests`, and `CouchLink.Versioning.Tests`. They validate signature tampering, updater rollback/signature requirements, and custom version ordering.
- No Android `src/test` or `src/androidTest` sources were found. The documented integration strategy is manual device testing in `docs/building/TESTING.md`.
- Static type diagnostics were clean for all 25 Android Kotlin and 29 Windows C# files on this checkout. No builds or device tests were run during this map.

## Signals / Active Considerations

- **Documentation/version drift:** root release metadata is `1.3.1`, but `docs/building/BUILDING.md`, `docs/building/TESTING.md`, `src/windows/README.md`, and several library project version properties still describe `1.1`. Treat 1.3.1 source and root release docs as the current behavior until this is reconciled.
- **Large orchestration hotspots:** `LauncherHostClient.kt` (771 lines), `BluetoothHidController.kt` (701), `TvRemoteScreen.kt` (648), `TvDiscoveryController.kt` (523), and `GitHubUpdateService.cs` (444) carry several responsibilities each. Change them by tracing the specific feature flow; do not introduce abstraction layers preemptively.
- **Hardware-specific TV behavior:** the TV controller retains a named IP/MAC fallback for a validated device and uses deterministic UI-navigation macros/input links. Any broader-device work needs real hardware validation, not only unit checks.
- **Cross-language protocol risk:** host JSON is typed only on Windows but manually decoded on Android. Protocol-v1 changes require coordinated Android/Windows compatibility coverage.
- **Platform-bound verification:** production builds and manual checks require Windows for WPF/.NET Windows targets and Android tooling plus physical Bluetooth/TV/Windows hardware. Local mapping ran diagnostics only.
- **Security-sensitive update paths:** update behavior is intentionally fail-closed. Preserve exact release-asset naming, HTTPS/repository URL checks, hashes, signatures, package identity, and separate-updater rollback behavior in any release work.

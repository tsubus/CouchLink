# CouchLink — Claude Code

Read `CONVENTIONS.md` before any Git or GitHub operation.

## Project

CouchLink is a local-first Android remote for Windows gaming and Google TV control.

Stack: Kotlin, Jetpack Compose, Android SDK 28–36, C#, .NET 8, and WPF.

## Commands

| Action | Command |
| --- | --- |
| Android build | `cd src/android; .\gradlew.bat clean :app:assembleDebug` |
| Windows build/typecheck | `cd src/windows; dotnet build .\CouchLink.sln -c Release` |
| Windows run | `dotnet run --project .\src\windows\CouchLink.Host.Wpf\CouchLink.Host.Wpf.csproj -c Release` |
| Windows test | `cd src/windows; dotnet test .\CouchLink.sln -c Release` |
| Windows lint | `dotnet format src/windows/CouchLink.sln --verify-no-changes` |
| CI (GitHub Actions) | `gh pr checks` |
| Preflight | `dotnet format src/windows/CouchLink.sln --verify-no-changes && cd src/android && .\gradlew.bat clean :app:assembleDebug && cd ../windows && dotnet test .\CouchLink.sln -c Release && dotnet build .\CouchLink.sln -c Release` |

## Architecture

Android keeps Bluetooth HID, Windows-host, and Google TV paths independent.

Windows separates WPF UI, host core, protocol contracts, release security, and updater projects.

## Conventions

- Keep Android feature packages and Windows project boundaries intact.
- Preserve Bluetooth fallback and local-only behavior.
- Preserve protocol-v1 compatibility across Android and Windows.
- Use Conventional Commits: `<type>(<scope>): <description>`.

## Never

- NEVER weaken pairing, token validation, update checks, or release signing.
- NEVER add cloud services, telemetry, advertising, or unnecessary permissions.
- NEVER make Bluetooth HID depend on host or TV availability.
- NEVER commit signing keys, passwords, tokens, generated binaries, or local configuration.

## Agent Rules

- Read `specs/` before writing code.
- Write plans and specifications under `specs/`.
- Use bigpowers skills for feature work and bug fixes.
- Use `plan-work` before implementing feature code.
- Use `investigate-bug` before implementing a bug fix.
- Run applicable verification after every change.
- Stop forward work on red Preflight or CI.
- Use `quick-fix` or `fix-bug` for reproducible failures.
- Keep code changes focused and minimal.

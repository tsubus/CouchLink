# e01s03 — Verify advertised-address selection behavior

## 1. Story identity

`e01s03`; epic `e01`; status `todo`; 2 BCPs; maturity 3 (countable); risk `P2`.

## 2. Title

Verify advertised-address selection behavior.

## 3. User story

As a CouchLink maintainer, I want repeatable automated and Windows-LAN checks so the host cannot regress to an unreachable, stale, or mismatched advertised identity.

## 4. Problem

Adapter inventories are machine-specific, Host Core has no networking coverage, and this change crosses runtime, public snapshot, WPF, and protocol-advertisement boundaries.

## 5. Outcome

Dependency-free controlled-inventory checks run before the solution gate. Windows evidence proves automatic selection, manual override, explicit advertising stop on loss, reselection, and unchanged local-control compatibility.

## 6. Scope

Integrate e01s01/e01s02 focused executable checks into `CouchLink.sln`, run final Preflight, update `docs/architecture/ARCHITECTURE.md` and `CHANGELOG.md`, and collect fail-closed PR/CI evidence from a feature candidate derived from the 1.3.1 regression baseline.

## 7. Out of scope

Live Docker/VPN interoperability, Android implementation changes, protocol changes, persistent selection, automatic failover, telemetry, release-version selection, and new test dependencies.

## 8. Preconditions

e01s01/e01s02 implementation and focused checks are present. WPF and adapter manipulation run on Windows. The test machine exposes two simultaneous eligible physical LAN interfaces.

## 9. Dependencies

Use installed .NET/Android SDKs and GitHub CLI only. No NuGet package is added. Existing SDK/CLI tools are `[OK]`.

## 10. Architecture and zoom-out

- **Purpose:** the new dependency-free Host Core executable test project supplies deterministic inventory/runtime regressions; solution/Preflight prove repository integration; manual validation proves real WPF/LAN behavior.
- **Callers:** developers and CI invoke the focused project and solution gates; reviewers consume PR markers/evidence.
- **Contracts:** protocol-v1, UDP 45820, all current broadcast destinations, TCP `IPAddress.Any:45821`, host/pairing identity, Wake semantics, Android behavior, and Bluetooth HID independence remain regression boundaries.

The test executable references Host Core/Protocol without external test packages and supports named `--group` filters used by e01s01/e01s02 tasks. Add it to `CouchLink.sln`; execute it explicitly because `dotnet test` does not by itself prove a dependency-free console assertion suite ran.

**Reason for Depth:** a dedicated controlled-inventory executable is necessary because real Windows adapter enumeration is nondeterministic and adding a test framework package is prohibited.

## 11. Data and state

Fixtures use synthetic documentation-range IPv4 values, synthetic six-byte MACs, and representative labels/types/status/errors. They never enumerate the machine, access preferences, or send UDP/TCP traffic. Manual evidence contains only the minimum LAN details needed to prove behavior.

## 12. Requirements

#### ADDED: Focused executable verification

The named groups `default-selection`, `advertisement-lifecycle`, `compatibility`, `manual-selection`, `snapshot-presentation`, and `session-boundaries` must exit nonzero on assertion failure and zero only after all group assertions execute.

#### MODIFIED: Solution integration gate

**Before:** `dotnet test src/windows/CouchLink.sln` has no Host Core network-selection behavioral coverage.
**After:** run every focused Host Core group explicitly, then run the solution test command and WPF Release build. The test project is solution-listed without adding packages.

#### ADDED: Compatibility assertions

Pin the existing `DiscoveryAdvertisement` shape/names and protocol version, UDP/TCP ports, TCP all-address listener binding, unchanged destination-broadcast enumeration, host/pairing identity invariants, and selected-MAC Wake semantics.

#### MODIFIED: Behavior-change documentation

**Before:** architecture/changelog do not describe selectable advertised identity.
**After:** both designated targets state `session-only selection`, `manual reselection`, and `no protocol change`.

#### ADDED: Fail-closed Windows/PR evidence

A feature candidate derived from baseline 1.3.1 must supply all five exact checked markers with same-line HTTPS or nonempty attachment evidence, followed by passing `gh pr checks` evidence.

## 13. Manual validation

On Windows, run the feature candidate (1.3.1 is comparison-only; semantic-release determines the merged version/tag) with two simultaneous eligible physical interfaces.

1. Capture the eligible-only selector, automatic state, selected primary IPv4, and both candidates. Add:

   ```text
   - [x] `e01s03-candidate-build-and-two-eligible-interfaces` evidence: <https-or-attachment>
   ```

2. Select the other interface; capture that dashboard endpoint and discovery use its first usable IPv4. Add:

   ```text
   - [x] `e01s03-manual-selection` evidence: <https-or-attachment>
   ```

3. Disconnect it; capture no discovery advertisement, no stale/sentinel endpoint, unchanged listener status, and manual reselection without fallback. Add:

   ```text
   - [x] `e01s03-loss-requires-manual-reselection` evidence: <https-or-attachment>
   ```

4. Explicitly select the remaining eligible interface and capture resumed discovery. Then capture trusted Android pairing, UDP discovery 45820, TCP session 45821, and independent Bluetooth HID. Add:

   ```text
   - [x] `e01s03-local-control-compatibility` evidence: <https-or-attachment>
   ```

5. Attach focused tests, final Preflight, documentation/changelog, WPF build, and passing Actions/`gh pr checks`. Add:

   ```text
   - [x] `e01s03-preflight-documentation-and-ci` evidence: <https-or-attachment>
   ```

Each marker and evidence reference must be on one line. Placeholder text, unchecked markers, newline-separated evidence, and missing references fail.

## 14. Security and privacy

Use synthetic fixtures. Do not publish opaque adapter IDs, pairing tokens, or unnecessary LAN identifiers. Confirm pairing still gates launcher/session access.

## 15. Compatibility

Use 1.3.1 only as a regression baseline. Verify protocol-v1 and Android behavior without changing either. Preserve broadcast destinations, UDP 45820, TCP 45821/all-address binding, host/pairing identity, selected-MAC Wake meaning, and Bluetooth HID independence.

## 16. Non-functional requirements

Automated tests are deterministic, package-free, and independent of real adapters. Native-command gates are chained fail-fast. Manual evidence is reviewable and marker queries fail closed.

## 17. Acceptance criteria

```gherkin
Scenario: Every controlled group executes
  Given the dependency-free Host Core test executable
  When each named group is invoked
  Then it executes its assertions and exits zero
  And an unknown or empty group request does not report a false pass.

Scenario: Selection loss is regression protected
  Given automatic and manual selected identities in controlled inventories
  When each identity disappears
  Then tests prove no advertisement or stale/sentinel details
  And no replacement occurs before explicit reselection or a new session.

Scenario: Compatibility boundaries are pinned
  Given selection changes
  When compatibility assertions run
  Then protocol-v1, ports, broadcast destinations, TCP binding, host/pairing identity, and Wake semantics remain unchanged.

Scenario: Real Windows behavior is evidenced
  Given a feature candidate and two eligible interfaces
  When the owner selects, disconnects, and explicitly reselects
  Then PR evidence proves address change, advertising stop, no stale endpoint, no fallback, and resumed discovery.

Scenario: Existing local control remains intact
  Given an available selected interface
  When a trusted Android remote pairs and connects
  Then PR evidence proves UDP discovery, TCP sessions, pairing, and Bluetooth HID independence.

Scenario: Final gates pass
  Given documentation and manual evidence are complete
  When final Preflight and GitHub Actions run
  Then all commands pass and every exact PR marker has valid same-line evidence.
```

## 18. Implementation steps

1. Run all focused groups explicitly, prove the executable rejects an unknown group, then run solution tests and WPF Release build → verify: `for group in default-selection advertisement-lifecycle compatibility manual-selection snapshot-presentation session-boundaries; do dotnet run --project src/windows/CouchLink.Host.Core.Tests/CouchLink.Host.Core.Tests.csproj --configuration Release -- --group "$group" || exit 1; done && ! dotnet run --project src/windows/CouchLink.Host.Core.Tests/CouchLink.Host.Core.Tests.csproj --configuration Release -- --group does-not-exist && dotnet test src/windows/CouchLink.sln --configuration Release && dotnet build src/windows/CouchLink.Host.Wpf/CouchLink.Host.Wpf.csproj --configuration Release`
2. Run fail-fast Final Preflight after integration → verify: `dotnet format src/windows/CouchLink.sln --verify-no-changes && cd src/android && .\gradlew.bat clean :app:assembleDebug && cd ../windows && dotnet test .\CouchLink.sln -c Release && dotnet build .\CouchLink.sln -c Release`
3. Update both designated behavior-change documents → verify: `for target in docs/architecture/ARCHITECTURE.md CHANGELOG.md; do grep -qi 'session-only selection' "$target" && grep -qi 'manual reselection' "$target" && grep -qi 'no protocol change' "$target" || exit 1; done`
4. Perform all Windows steps in section 13 and attach all exact markers → verify: ``gh pr view --json body --jq -e '.body | def evidence($marker): test("(?m)^[ \\t]*- \\[x\\] `" + $marker + "` evidence: (https://[^ \\t\\r\\n<>]+|attachment:[A-Za-z0-9][A-Za-z0-9._/-]*)[ \\t]*$"); evidence("e01s03-candidate-build-and-two-eligible-interfaces") and evidence("e01s03-manual-selection") and evidence("e01s03-loss-requires-manual-reselection") and evidence("e01s03-local-control-compatibility") and evidence("e01s03-preflight-documentation-and-ci")' >/dev/null``
5. Confirm required Actions checks and retain CI evidence → verify: ``gh pr checks && gh pr view --json body --jq -e '.body | test("(?m)^[ \\t]*- \\[x\\] `e01s03-preflight-documentation-and-ci` evidence: (https://[^ \\t\\r\\n<>]+|attachment:[A-Za-z0-9][A-Za-z0-9._/-]*)[ \\t]*$")' >/dev/null``

## 19. Risks and mitigations

- **P2 overall; P1 compatibility tasks:** console tests can be silently skipped by `dotnet test`; invoke every named group explicitly and reject an unknown group.
- Physical adapter changes can disrupt the test machine; use a controlled Windows host and restore networking.
- PR text can claim evidence without proving it; exact anchored marker checks fail closed, while reviewers inspect attachments.

## 20. Definition of done

All named groups, solution tests, WPF build, final Preflight, documentation checks, Windows scenarios, exact marker query, Actions, and `gh pr checks` pass. Evidence proves explicit no-advertisement loss handling and reselection while all compatibility boundaries remain unchanged and no dependency is added.

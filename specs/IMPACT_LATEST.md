# Impact — e01 Windows Host network-interface selection

## Target

**Change boundary:** Windows Host advertised network identity and session-only WPF interface selection.

Primary implementation seams:

- `src/windows/CouchLink.Host.Core/Networking/NetworkAddressHelper.cs` — current live adapter enumeration, IPv4/broadcast calculation, and Ethernet/Wi-Fi ordering. It presently admits all non-loopback/non-tunnel active adapter types, so it can include Docker/VPN/internal adapters.
- `src/windows/CouchLink.Host.Core/CouchLinkHostRuntime.cs` — creates the discovery payload and `HostSnapshot`; it currently re-enumerates the primary address/MAC rather than retaining a current-session selection.
- `src/windows/CouchLink.Host.Core/Networking/DiscoveryAdvertiser.cs` — creates an advertisement every two seconds and separately enumerates all active IPv4 broadcast targets.
- `src/windows/CouchLink.Host.Core/HostSnapshot.cs` — public snapshot contract consumed by WPF.
- `src/windows/CouchLink.Host.Wpf/ViewModels/MainWindowViewModel.cs` and `Views/MainWindow.xaml` — dashboard endpoint/diagnostics binding and the proposed eligible-only selector/reselection UI.

**Required invariants:** selection changes advertised discovery address and WPF-displayed connection details only. Preserve UDP 45820, TCP listener binding/port 45821, host ID/name, pairing state and tokens, MAC/Wake identity semantics, protocol-v1 payload shape, local-only behavior, Android behavior, and Bluetooth HID independence. A selected candidate becoming unavailable must clear stale advertised/displayed details and require explicit manual reselection; it must not silently choose a replacement until a new host session.

## Dependents (8 direct active code paths)

1. `src/windows/CouchLink.Host.Core/CouchLinkHostRuntime.cs`
   - `StartAsync` passes `CreateAdvertisement` to `DiscoveryAdvertiser`.
   - `CreateAdvertisement` supplies `DiscoveryAdvertisement.Address` and `MacAddress`.
   - `CreateSnapshot` supplies public `HostSnapshot.PrimaryAddress`.
   - Impact: must own/refresh session-only selected identity consistently so discovery and snapshot cannot diverge or retain stale details.
2. `src/windows/CouchLink.Host.Core/Networking/DiscoveryAdvertiser.cs`
   - Calls the runtime factory before each UDP advertisement and independently calls `GetActiveIpv4Addresses` for broadcast targets.
   - Impact: the pre-advertisement refresh and no-advertisement state must be explicit; do not accidentally alter UDP port or use the selected interface change to rebind listeners. Whether broadcast-target restriction is intended needs a design decision because e01 scopes advertised details, not listener binding.
3. `src/windows/CouchLink.Host.Core/HostSnapshot.cs`
   - Public immutable contract containing `PrimaryAddress`.
   - Impact: likely needs availability, selected-candidate display data, and/or reselection status for the dashboard. Any positional-record change requires its runtime producer and sole active consumer to change together.
4. `src/windows/CouchLink.Host.Wpf/ViewModels/MainWindowViewModel.cs`
   - Sole active consumer of `CouchLinkHostRuntime` and `HostSnapshot`; derives `SessionEndpoint`, `DiagnosticsText`, and `DiscoveryStatus` and dispatches `SnapshotChanged` to WPF.
   - Impact: must expose candidates, automatic/manual state, unavailable text, and explicit selection/reselection command while retaining dispatcher and notification behavior.
5. `src/windows/CouchLink.Host.Wpf/Views/MainWindow.xaml`
   - Binds `SessionEndpoint`, `DiscoveryStatus`, diagnostics-facing values, pairing, and trusted-device controls.
   - Impact: add selector and unavailable state without regressing existing dashboard, tray, pairing, or trusted-device controls.
6. `src/windows/CouchLink.Protocol/Messages.cs`
   - Defines `DiscoveryAdvertisement`, including `Address`, `SessionPort`, `MacAddress`, and protocol-visible identity fields.
   - Impact: preserve record shape and serialization names; only the existing `Address` value may change. No Protocol project edit should be necessary.
7. `src/windows/CouchLink.Host.Core/Networking/SessionServer.cs`
   - Independently binds `TcpListener` to `IPAddress.Any:45821` and owns pairing/token-authenticated sessions.
   - Impact: regression boundary only: selected advertised address must still lead to this unchanged listener; e01 must not change pairing/session identity or bind the listener to a selected adapter.
8. `src/windows/CouchLink.Host.Wpf/Services/HostPreferences.cs`
   - Existing persisted WPF preferences store startup/audio choices.
   - Impact: negative dependency: selected interface identity must not be added or saved; restart must derive a fresh automatic default.

No active Android source reference to `DiscoveryAdvertisement` was found by a source search, but Android protocol behavior remains an external compatibility boundary because it consumes discovery packets at runtime. Legacy-tree references under `Legacy-CouchLink-v0.1-dev.13/` were excluded from the active blast radius.

## Affected Stories

- **e01s01 — Select an eligible LAN interface by default** (`specs/epics/e01-windows-host-network-interface-selection/e01s01-default-lan-interface-selection.md`)
  - Owns controlled inventory/model seam; strict physical-LAN eligibility; deterministic Ethernet, Wi-Fi, name, and opaque-ID ordering; first usable Windows-reported IPv4; refresh-before-advertising; and no replacement after initial selection loss.
- **e01s02 — Select the advertised interface for this host session** (`specs/epics/e01-windows-host-network-interface-selection/e01s02-session-interface-selector.md`)
  - Owns runtime session state, snapshot/UI candidates, manual override, explicit reselection, unavailable state, and no preference persistence.
- **e01s03 — Verify advertised-address selection behavior** (`specs/epics/e01-windows-host-network-interface-selection/e01s03-network-interface-selection-verification.md`)
  - Owns solution-level test integration and Windows manual/PR evidence. Its planned architecture/changelog work is outside this research task's permitted writes.

## Test Coverage

### Existing automated coverage

- `src/windows/CouchLink.Versioning.Tests/Program.cs` is the only active test-project code referencing `CouchLink.Host.Core`; it tests `CouchLinkVersion.Compare`, not networking, runtime snapshots, discovery, pairing, or WPF.
- `src/windows/CouchLink.sln` currently includes only `CouchLink.Versioning.Tests` as an active test project. Searches found no active `NetworkAddressHelper`, `DiscoveryAdvertiser`, `CouchLinkHostRuntime`, `HostSnapshot`, or `DiscoveryAdvertisement` behavioral test.
- **Gap:** no controlled inventory seam exists yet, so current adapter enumeration is machine-dependent and no focused test can prove candidate exclusion, order, first usable IPv4, manual selection, loss handling, or non-persistence.
- **Gap:** no automated serialization/compatibility assertion pins the protocol-v1 `DiscoveryAdvertisement` shape while its `Address` value changes.
- **Gap:** no WPF view-model test asserts snapshot-to-endpoint/selector/unavailable-state binding behavior.

### Required automated verification before merge

1. Add controlled-inventory tests without real adapter APIs or new packages for e01s01:
   - Ethernet priority; Wi-Fi fallback; complete name/opaque-ID tie-break;
   - exclusion of Docker, vEthernet/Hyper-V, virtual, VPN, WireGuard, TAP/TUN, loopback, tunnel, unsupported/ambiguous interfaces, invalid/missing MAC, property failure, and unusable IPv4;
   - first usable reported IPv4 without reordering;
   - no eligible candidate; refresh before each advertisement; and loss of initial automatic selection clears address/details with no replacement.
2. Add focused e01s02 tests for manual override, eligible-only candidates, snapshot/advertisement consistency, loss of manually selected identity, explicit reselection requirement, fresh-session default restoration, and absence of `HostPreferences` persistence.
3. Add regression assertions that selection affects only advertised details: `DiscoveryAdvertisement` remains protocol-v1 with its current fields and session port, `SessionServer` remains `IPAddress.Any:45821`, and host/pairing/token identity behavior is unchanged.
4. Run the designated integration command: `dotnet test src/windows/CouchLink.sln --configuration Release`; build `src/windows/CouchLink.Host.Wpf/CouchLink.Host.Wpf.csproj --configuration Release` on Windows. E01 task files additionally require documented Preflight, including format, Android Debug build, Windows solution test, and Windows solution build.

### Required manual verification (e01s03)

On Windows, use a feature-candidate build derived from the 1.3.1 regression baseline and a LAN with two simultaneous eligible physical interfaces:

- Capture eligible-only selector, automatic selection, primary IPv4, and both candidates.
- Select the other interface and confirm dashboard endpoint and discovery advertise its first usable IPv4.
- Disable/disconnect that selected interface and confirm advertising plus stale endpoint details stop, actionable manual reselection appears, and no automatic fallback occurs.
- Confirm trusted Android pairing, UDP discovery on 45820, TCP session on 45821, and unchanged Bluetooth HID independence.
- Attach Preflight, behavior-change documentation/changelog, and passing CI evidence to the PR using all five exact `e01s03-*` markers required by the story; run `gh pr checks` after Actions completes.

`docs/building/TESTING.md` supplies existing broader regression checks for host discovery/listener, dashboard endpoint, trusted pairing/reconnect, and Bluetooth independence; it does not yet cover multi-interface selection or loss behavior.

## Risk: High

This is a shared runtime/public-snapshot and protocol-advertisement boundary with eight direct active code paths and no existing automated coverage for network selection. A defect can advertise an unreachable/stale address, regress WPF state, or unintentionally change discovery/session/pairing compatibility.

## Uncertainty / decisions to resolve in planning

- **Broadcast destinations:** `DiscoveryAdvertiser` currently sends the unchanged payload to global broadcast plus every active interface-specific broadcast. The stories mandate refresh before every advertisement and selected advertised details, but do not explicitly say whether destination broadcasts must be restricted to the selected interface. Preserve current target behavior unless the owner explicitly decides otherwise; restricting it may reduce discovery reachability and is beyond “advertised details only.”
- **No-selection wire behavior:** stories say “stop advertising,” while current factory always returns a `DiscoveryAdvertisement`. Decide the internal contract (nullable/try factory or advertiser gate) so it cannot serialize `Address = "Unavailable"`.
- **MAC identity after selection:** current `GetPrimaryMacAddress` can re-enumerate independently of `GetPrimaryAddress`. Planning must make address and MAC originate from the same selected identity, while preserving the existing host/pairing identity and Wake-on-LAN meaning.
- **Public snapshot shape:** choose whether to extend `HostSnapshot` directly or expose a contained selection state. Its positional record means all active construction/consumption must compile together.

## Evidence

- Product scope and constraints: `specs/product/SCOPE_LATEST.yaml`; `specs/planning-context.yaml`.
- Epic/story/task requirements: `specs/release-plan.yaml`; `specs/epics/e01-windows-host-network-interface-selection/epic.yaml`; `e01s01-tasks.yaml`; `e01s02-tasks.yaml`; `e01s03-tasks.yaml`.
- Active implementation/dependency evidence: `src/windows/CouchLink.Host.Core/Networking/NetworkAddressHelper.cs`; `DiscoveryAdvertiser.cs`; `CouchLinkHostRuntime.cs`; `HostSnapshot.cs`; `Networking/SessionServer.cs`; `HostConstants.cs`; `src/windows/CouchLink.Host.Wpf/ViewModels/MainWindowViewModel.cs`; `Views/MainWindow.xaml`; `Services/HostPreferences.cs`; `src/windows/CouchLink.Protocol/Messages.cs`; `ProtocolEnvelope.cs`.
- Manual/regression baseline: `docs/architecture/ARCHITECTURE.md`; `docs/building/TESTING.md`.

## Recommended action

**Add tests first, then plan implementation.** Design one injectable controlled-inventory/session-selection seam that atomically supplies the selected address, broadcast, and MAC to the runtime. Keep `DiscoveryAdvertisement`/protocol-v1 and `SessionServer` binding unchanged; make no-advertisement on selection loss explicit; then implement the WPF selector against snapshot state and execute e01s03’s Windows/PR evidence gate.

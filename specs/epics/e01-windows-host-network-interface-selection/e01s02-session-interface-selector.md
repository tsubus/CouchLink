# e01s02 — Select the advertised interface for this host session

## 1. Story identity

`e01s02`; epic `e01`; status `todo`; 3 BCPs; maturity 3 (countable); risk `P1`.

## 2. Title

Select the advertised interface for this host session.

## 3. User story

As a Windows Host owner, I want to choose an eligible LAN interface so CouchLink advertises the address my Android remote can reach.

## 4. Problem

Automatic selection cannot infer the intended LAN path when multiple physical interfaces are eligible. The current flat `HostSnapshot.PrimaryAddress` cannot safely express candidate list, selection mode, or unavailable/reselection state.

## 5. Outcome

The WPF dashboard shows eligible physical LAN choices, applies an explicit current-session choice, and clears stale details with actionable reselection state if that identity disappears. Snapshot, dashboard, and advertisement all derive from one atomic runtime selection.

## 6. Scope

Change session selection and snapshot production in `CouchLinkHostRuntime.cs`/`HostSnapshot.cs`, consume the state in `MainWindowViewModel.cs`, and bind the selector/status in `MainWindow.xaml`. Add focused tests first. No preferences or protocol file changes are needed.

## 7. Out of scope

Preference persistence, restart restoration, automatic failover, TCP rebinding, adapter management, Android work, protocol-v1 changes, pairing changes, Wake semantic changes, and Bluetooth HID changes.

## 8. Preconditions

e01s01 supplies deterministic eligible identities, automatic start selection, refresh, and no-advertisement gating.

## 9. Dependencies

Existing .NET 8, WPF binding/command patterns, and BCL collections only. No external package is proposed; existing platform APIs are `[OK]`.

## 10. Architecture and zoom-out

- **Module purpose:** the runtime owns lifecycle and synchronized host state; `HostSnapshot` is its immutable public UI contract; the view model translates snapshots/commands for WPF; XAML renders those properties.
- **Callers:** `MainWindowViewModel` is the active snapshot/runtime consumer, and `MainWindow.xaml` is its binding consumer. Discovery consumes runtime advertisement state separately.
- **Contracts:** snapshot notifications remain dispatcher-safe; selection changes advertised details only; protocol-v1, ports, listener binding, pairing/trusted devices, host identity, tray/dashboard controls, and preferences remain intact.

Use a non-null contained `HostNetworkSelectionSnapshot` value on `HostSnapshot` rather than adding loosely related positional fields. It contains immutable candidate display DTOs, selected opaque ID (nullable), selected label/address (nullable), nullable selection mode (`Automatic` or `Manual`), and availability (`NotStarted`, `Available`, or `UnavailableRequiresReselection`). Replace the flat positional `PrimaryAddress` input and update the runtime producer and sole active view-model consumer in the same compile unit of work. The runtime exposes an explicit selection method that accepts an opaque candidate ID, refreshes candidates, rejects absent/ineligible IDs, and atomically swaps the complete selected identity.

**Reason for Depth:** one contained snapshot value is required to represent candidates, nullable endpoint details, mode, and reselection as a coherent state while avoiding a brittle expansion of the positional record.

## 11. Data and state

Candidate display DTOs expose opaque ID for binding only, friendly interface label, and selected IPv4 text. The ID is never shown, persisted, or serialized. No sentinel address is used: selected address is nullable when not started or unavailable, and the view model gates endpoint/diagnostic text on availability. `StartAsync` clears old state and derives a fresh automatic default; `StopAsync` clears session selection to `NotStarted`, so no manual choice crosses the stop/start boundary.

## 12. Requirements

#### ADDED: Eligible current-session selector

The snapshot exposes only currently eligible physical LAN candidates. Selecting one by opaque ID updates the complete selected identity and marks mode `Manual`.

#### MODIFIED: Snapshot network state

**Before:** `HostSnapshot` contains one non-null `PrimaryAddress` string and cannot distinguish a real address from `"Unavailable"`.
**After:** `HostSnapshot` contains one non-null selection-state record with nullable selected address/ID/mode, immutable candidate list, and explicit `NotStarted`, `Available`, or `UnavailableRequiresReselection` availability. Runtime producer and WPF consumer change together so the solution remains compile-safe.

#### MODIFIED: Dashboard endpoint and discovery status

**Before:** WPF always formats `PrimaryAddress:45821`, and running state alone means “Advertising CouchLink Host.”
**After:** WPF formats an endpoint only while selection is available. Unavailable state shows no stale address, says advertising is stopped pending manual reselection, and leaves listener status independently accurate.

#### MODIFIED: Manual selection effect

**Before:** no owner selection exists.
**After:** an eligible manual choice changes only discovery payload address/MAC details and snapshot connection details. Existing broadcast destinations, TCP listener binding, host/pairing identity, Wake semantics, and protocol shape do not change.

#### ADDED: Explicit reselection after loss

After either automatic or manual selected identity is lost, refreshed eligible candidates remain visible but no candidate is auto-selected. The owner must invoke the selection command; successful selection resumes advertisement on the next cycle.

#### ADDED: Session-only lifetime

Selection is never read from or written to `HostPreferences`. A new host session derives e01s01's automatic default, and stop clears the old session identity to `NotStarted`.

## 13. Interaction flow

At startup, show the automatic selection and eligible list. A selector choice invokes the runtime by opaque ID. On success, snapshot and discovery use the same complete identity. On loss, clear endpoint/details, preserve refreshed candidates, show “Select an available interface to resume discovery,” and require an explicit choice. Preserve all pairing, trusted-device, startup, tray, and diagnostics controls.

## 14. Security and privacy

Do not transmit candidates/opaque IDs to Android or persist them. Preserve trusted pairing and token behavior. Diagnostics may show the friendly selected label/address but not the opaque adapter ID.

## 15. Compatibility

Do not edit `CouchLink.Protocol/Messages.cs`. Keep protocol-v1 fields, UDP 45820, TCP 45821, `IPAddress.Any` binding, existing broadcast targets, and Android behavior unchanged.

## 16. Non-functional requirements

Selection/snapshot updates use the runtime synchronization boundary. WPF updates stay on the application dispatcher and use existing `INotifyPropertyChanged` conventions. Empty/unavailable state must not produce `:45821`, `Unavailable:45821`, or stale diagnostics.

## 17. Acceptance criteria

```gherkin
Scenario: Owner selects another eligible interface
  Given Ethernet and Wi-Fi are eligible and Ethernet is automatic
  When the owner selects Wi-Fi
  Then snapshot and discovery use Wi-Fi address and MAC from one identity
  And Ethernet remains a candidate
  And listener binding and broadcast destinations are unchanged.

Scenario: Ineligible adapters are hidden and rejected
  Given internal adapters exist
  When the selector is displayed or an invalid opaque ID is submitted
  Then internal adapters are absent
  And runtime selection does not change.

Scenario: Selected interface becomes unavailable
  Given either an automatic or manual current-session selection
  When refresh cannot confirm it
  Then endpoint and diagnostics contain no stale or sentinel address
  And discovery emits no advertisement
  And eligible alternatives remain visible
  And manual reselection is required.

Scenario: Explicit reselection resumes discovery
  Given selection is unavailable and another eligible candidate is visible
  When the owner explicitly selects it
  Then snapshot and the next advertisement use that candidate's atomic address/MAC identity.

Scenario: Selection is not persisted
  Given a manual choice in one runtime session
  When a new runtime session starts
  Then the deterministic automatic default is used
  And HostPreferences contains no network-selection value.
```

## 18. Implementation steps

1. Add RED runtime/snapshot cases for automatic/manual/unavailable states, invalid selection, atomic snapshot/advertisement consistency, explicit reselection, and fresh-session reset; then implement the contained snapshot contract and runtime command → verify: `dotnet run --project src/windows/CouchLink.Host.Core.Tests/CouchLink.Host.Core.Tests.csproj --configuration Release -- --group manual-selection`
2. Add RED presentation assertions, then expose candidates, selected item, status, endpoint gating, and selection command in the view model without touching preferences → verify: `dotnet run --project src/windows/CouchLink.Host.Core.Tests/CouchLink.Host.Core.Tests.csproj --configuration Release -- --group snapshot-presentation`
3. Bind an eligible-only selector and actionable unavailable state in XAML while preserving existing controls and compile the producer/consumer contract together → verify: `dotnet build src/windows/CouchLink.Host.Wpf/CouchLink.Host.Wpf.csproj --configuration Release`
4. Run selection-lifetime and advertised-details-only regressions → verify: `dotnet run --project src/windows/CouchLink.Host.Core.Tests/CouchLink.Host.Core.Tests.csproj --configuration Release -- --group session-boundaries && dotnet test src/windows/CouchLink.sln --configuration Release`

## 19. Risks and mitigations

- **P1:** public snapshot changes can break WPF compilation; use one contained record and update producer/consumer together, then build WPF.
- A ComboBox setter can fire during snapshot refresh; command logic must reject null/stale IDs and avoid accidental selection.
- Runtime refresh and UI choice can race; refresh and complete-identity swap occur under one synchronization boundary.

## 20. Definition of done

Focused tests and WPF build pass; manual selection and loss/reselection scenarios pass; no stale/sentinel endpoint is displayed or serialized; selection is session-only; protocol, ports, broadcast destinations, TCP binding, identity, pairing, Wake semantics, Android behavior, and existing dashboard controls remain unchanged; and no dependency is added.

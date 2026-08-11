# e01s01 — Select an eligible LAN interface by default

## 1. Story identity

`e01s01`; epic `e01`; status `todo`; 3 BCPs; maturity 3 (countable); risk `P1`.

## 2. Title

Select an eligible LAN interface by default.

## 3. User story

As a Windows Host owner, I want CouchLink to choose a reachable LAN interface automatically so Android remotes discover a usable address without exposing internal adapters.

## 4. Problem

`NetworkAddressHelper` currently admits active adapter types beyond physical Ethernet/Wi-Fi. `CouchLinkHostRuntime` also re-enumerates address and MAC independently and always creates an advertisement, so it cannot represent selection loss without stale or sentinel wire data.

## 5. Outcome

A new host session chooses one eligible physical LAN identity deterministically. Address, broadcast, and MAC are captured atomically from that identity. Inventory refresh before every advertisement either confirms that same identity or moves the session to an explicit unavailable state that emits no advertisement and requires manual reselection.

## 6. Scope

Change the controlled-inventory/selection seam in `NetworkAddressHelper.cs`, the session-owned automatic selection in `CouchLinkHostRuntime.cs`, and the no-advertisement gate in `DiscoveryAdvertiser.cs`. Add dependency-free tests first. Preserve the current broadcast destination set.

## 7. Out of scope

Persistence, automatic failover, TCP listener rebinding, destination-broadcast restriction, diagnostics tooling, virtual-network support, Android changes, protocol-v1 changes, pairing changes, Wake semantic changes, and Bluetooth HID changes.

## 8. Preconditions

The host targets .NET 8 on Windows. Windows supplies adapter ID, name/description, type, status, physical address, and unicast properties. Initial Preflight must pass before RED tests or implementation.

## 9. Dependencies

Only .NET 8 BCL networking APIs and the repository's dependency-free executable-test convention are used. No external package is proposed; BCL APIs are `[OK]`.

## 10. Architecture and zoom-out

- **Module purpose:** `NetworkAddressHelper` enumerates IPv4 identities and broadcast destinations; `CouchLinkHostRuntime` owns host lifecycle/state and creates snapshots/protocol advertisements; `DiscoveryAdvertiser` serializes and broadcasts advertisements on its interval.
- **Callers:** the runtime calls the helper and supplies the advertiser factory; WPF consumes runtime snapshots; Android consumes the unchanged protocol-v1 discovery payload at runtime.
- **Contracts:** preserve UDP 45820, all existing global/interface-specific broadcast destinations, TCP `IPAddress.Any:45821`, protocol-v1 fields, host ID/name/state, pairing/tokens, local-only behavior, and Bluetooth HID independence.

Introduce one controlled inventory model in the networking helper that returns a complete selected identity: opaque adapter ID, display name, interface class, first usable IPv4, broadcast, and MAC. The runtime stores the selected adapter ID and last confirmed complete identity as one lock-protected session state. `DiscoveryAdvertiser` accepts an explicitly optional factory result and skips serialization/sends when no advertisement is available; it still waits for the normal interval and continues enumerating the same broadcast targets when an advertisement exists.

**Reason for Depth:** the controlled inventory/identity seam is necessary to test machine-independent selection and to prevent separately enumerated address/MAC values from describing different adapters.

## 11. Data and state

Use `NetworkInterface.Id` only as an opaque process-local matching key. Never display, serialize, or persist it. One selected identity owns its address, broadcast, and MAC atomically. Snapshot availability is `NotStarted`, `Available`, or `UnavailableRequiresReselection`; selection mode is null until a selection exists, then `Automatic` or `Manual`. There is no sentinel address and no automatic replacement after selection loss. Starting a new runtime session clears prior selection state and derives one new automatic default; stopping returns to `NotStarted` and clears session identity.

## 12. Requirements

#### MODIFIED: Eligible adapter classification

**Before:** Any active adapter except loopback/tunnel can be considered, including virtual/internal types.
**After:** Admit only active `Ethernet` or `Wireless80211` adapters with a valid six-byte nonzero MAC, readable properties, a usable IPv4, and name/description free of the case-insensitive markers `docker`, `vEthernet`, `hyper-v`, `virtual`, `vpn`, `wireguard`, `tap`, `tun`, `loopback`, and `tunnel`. Exclude unsupported or ambiguous adapters.

#### MODIFIED: Default identity ordering

**Before:** Candidate adapters are ordered by Ethernet, Wi-Fi, other type, then name; identity lookup is re-run independently.
**After:** Order eligible identities by Ethernet before Wi-Fi, display name using ordinal comparison, then opaque adapter ID using ordinal comparison. Select once at session start and carry address, broadcast, and MAC from that same identity.

#### MODIFIED: Primary IPv4 choice

**Before:** The helper accepts the first non-loopback IPv4 with any non-null mask.
**After:** Preserve Windows-reported unicast order and select the first IPv4 with a nonzero mask that is not unspecified, loopback, link-local, multicast, or broadcast. Do not sort addresses.

#### MODIFIED: Advertisement behavior after selection loss

**Before:** Every interval creates and serializes an advertisement, with `"Unavailable"` possible in runtime-derived details.
**After:** Refresh eligibility before every advertisement. If the selected adapter is absent/ineligible, atomically clear its selected details, mark manual reselection required, and return no advertisement. `DiscoveryAdvertiser` explicitly skips serialization and sends; it never serializes a sentinel address and never derives a replacement in that session.

#### MODIFIED: Broadcast destinations

**Before:** Each valid advertisement is sent to global broadcast and every active IPv4 interface-specific broadcast.
**After:** Preserve exactly that destination behavior; selection changes payload address/details only.

## 13. UX and observability

The automatic identity becomes the selected dashboard identity in e01s02. Selection loss clears endpoint details and records an actionable unavailable event without claiming that the unchanged TCP listener stopped.

## 14. Security and privacy

Do not expose adapter inventory or opaque IDs to Android. Preserve pairing/token checks and host identity. Keep the selected MAC's existing discovery/Wake meaning; do not reinterpret it as host or pairing identity.

## 15. Compatibility

`DiscoveryAdvertisement` remains unchanged. Session port remains 45821 and the listener remains bound to all addresses. Existing broadcast targets, Android behavior, and protocol-v1 serialization names remain unchanged.

## 16. Non-functional requirements

Selection is deterministic for the same controlled inventory, tolerates per-adapter property failures, performs no package install, and updates selection state under the runtime's existing synchronization boundary.

## 17. Acceptance criteria

```gherkin
Scenario: Prefer a physical wired identity atomically
  Given eligible Ethernet and Wi-Fi identities
  When a new host session derives its default
  Then Ethernet is selected
  And its first usable IPv4 and MAC come from the same inventory identity.

Scenario: Exclude internal and ambiguous adapters
  Given Docker, Hyper-V, VPN, WireGuard, TAP/TUN, loopback, tunnel, unsupported, invalid-MAC, property-failure, and unusable-address adapters plus eligible Wi-Fi
  When candidates are derived
  Then only eligible Wi-Fi is returned.

Scenario: Apply the complete deterministic tie-break
  Given same-class eligible adapters with colliding display names
  When candidates are ordered
  Then ordinal display name and then ordinal opaque ID determine the result.

Scenario: Preserve reported address order
  Given unusable addresses followed by two usable IPv4 addresses
  When identity is derived
  Then the first usable reported IPv4 is used without reordering.

Scenario: Gate advertising after automatic-selection loss
  Given an automatically selected identity and another eligible identity
  When refresh no longer finds the selected identity
  Then no advertisement is serialized or sent
  And stale snapshot details are cleared
  And the other identity is not selected until explicit manual reselection or a new session.

Scenario: Preserve advertisement destinations
  Given a valid selected identity
  When discovery advertises it
  Then the payload is sent to the existing global and active-interface broadcast destinations.
```

## 18. Implementation steps

1. Run Initial Preflight and stop on failure → verify: `dotnet format src/windows/CouchLink.sln --verify-no-changes && cd src/android && .\gradlew.bat clean :app:assembleDebug && cd ../windows && dotnet test .\CouchLink.sln -c Release && dotnet build .\CouchLink.sln -c Release`
2. Create a dependency-free executable Host Core test project and solution entry, add RED cases for classification, ordering, reported-address order, and atomic address/MAC identity, then implement the controlled inventory seam until they pass → verify: `dotnet run --project src/windows/CouchLink.Host.Core.Tests/CouchLink.Host.Core.Tests.csproj --configuration Release -- --group default-selection`
3. Add RED cases for refresh, automatic-selection loss, explicit no-advertisement gating, and unchanged broadcast targets; then implement session state/factory gating until they pass → verify: `dotnet run --project src/windows/CouchLink.Host.Core.Tests/CouchLink.Host.Core.Tests.csproj --configuration Release -- --group advertisement-lifecycle`
4. Pin protocol-v1 fields and unchanged listener/broadcast boundaries, then build the affected host → verify: `dotnet run --project src/windows/CouchLink.Host.Core.Tests/CouchLink.Host.Core.Tests.csproj --configuration Release -- --group compatibility && dotnet build src/windows/CouchLink.Host.Wpf/CouchLink.Host.Wpf.csproj --configuration Release`

## 19. Risks and mitigations

- **P1:** adapter metadata can misclassify virtual hardware; conservative exclusion plus representative fixtures fails closed.
- Refresh can race with adapter removal; complete immutable identities and lock-protected replacement avoid mixed address/MAC state.
- A nullable factory could accidentally serialize null; advertiser tests must prove serialization/send code is not entered when unavailable.

## 20. Definition of done

Initial Preflight passes; all focused groups pass; automatic selection and loss behavior meet the scenarios; no sentinel address reaches serialization; broadcast destinations, TCP binding, protocol-v1, identity, pairing, Wake semantics, and Android behavior are unchanged; and no dependency is added.

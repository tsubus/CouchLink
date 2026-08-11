# BUG-2026-08-11T100356: Plan-consistency gate cannot resolve its checker

## Problem

`plan-work` declares the cross-artifact consistency check a hard gate for e01, but its prescribed repository-relative command cannot start in CouchLink.

- **Actual behavior:** the command exits 127 because the required checker is absent from the project’s `scripts/lib` directory.
- **Expected behavior:** standard orchestration resolves and runs its required checker, then reports consistency findings rather than failing before inspection.
- **Minimal reproduction:** from the CouchLink root, run `bash scripts/lib/plan-consistency-check.sh specs/epics/e01-windows-host-network-interface-selection/`. It exits 127 with “No such file or directory.”

Security impact: NONE. No security exploit path identified.

## Root Cause Analysis

### Reproduce

Environment: CouchLink checkout with Bigpowers installed externally at `/home/tsubus/.pi/agent/npm/node_modules/bigpowers`. From the CouchLink root, the command prescribed by `plan-work` was run exactly:

```bash
bash scripts/lib/plan-consistency-check.sh specs/epics/e01-windows-host-network-interface-selection/
```

It emitted `scripts/lib/plan-consistency-check.sh: No such file or directory` and exited **127**. This occurs before the e01 capsule can be inspected. CouchLink's `scripts/` inventory contains only `New-WindowsReleaseSigningKey.ps1`, `Sign-WindowsReleaseAsset.ps1`, and `Test-WindowsReleaseSigningKey.ps1`.

### Isolate

`plan-work` uses that same project-relative path in both its hard gate and its `Verify` command. The checker instead exists only in the Bigpowers package at `.../bigpowers/scripts/lib/plan-consistency-check.sh`, alongside its sourced helpers.

Tracing a direct invocation of the package-resident checker shows `SCRIPT_LIB_DIR=.../bigpowers/scripts/lib` and `REPO_ROOT=.../bigpowers`: the checker's root is derived from its own installation location, not supplied by the target project. Thus changing only the skill's path would not establish a valid target-project contract.

### Hypothesize

1. **Confirmed — primary cause:** `plan-work` assumes a Bigpowers-owned helper has been provisioned at `scripts/lib/` in every consumer project. **Falsification:** the documented command would resolve and start the checker in a project without a vendored copy. It instead exits 127 at shell path resolution.
2. **Confirmed — coupled contract defect:** the package checker derives `REPO_ROOT` from its installation directory rather than accepting the target project root. **Falsification:** direct package invocation would derive CouchLink as its root. Trace instead derives the Bigpowers package root.
3. **Rejected:** e01 content causes the reported failure. The prescribed command fails before opening the capsule.

### Verify

Both falsification tests were run. The documented CouchLink-relative command failed with exit 127; the package-resident script exists but its trace derives the package root. Its direct invocation also exits 1 while processing e01: strict-mode command substitution stops on absent `spec:` metadata before its intended diagnostic reporter can emit a finding. That is a separately observable robustness issue, but not the cause of the reported missing-file failure.

**Root Cause:** the global Bigpowers plan-work/checker contract is split incorrectly: the skill invokes a consumer-project-relative path for a package-owned helper, while the helper has no explicit target-project-root input and instead treats its package installation as the repository root.

**Confidence:** High — direct execution, filesystem inventory, and shell trace independently establish the failed path and incorrect root derivation.

**Smallest corrective next step:** in Bigpowers, define one package-resolved invocation contract that passes the caller's project root explicitly to the checker; update the skill hard-gate and self-verification command to use that contract. Then make missing task metadata reach `report CRITICAL` rather than terminate under `set -e`.

Risk level: Medium. The failure blocks every plan-work handoff using this gate and can tempt users to bypass a mandatory consistency check. This is a workflow tooling/configuration defect, not an implementation-plan defect.

## TDD Fix Plan

1. **RED:** In the Bigpowers package’s integration tests, create a fixture project with an epic capsule but no project-local `scripts/lib` copy. Assert that the plan-consistency invocation locates the packaged checker, receives the fixture project root explicitly, and returns checker findings rather than shell “file not found.”
   **GREEN:** Change the planner/checker contract so the tool resolves its own packaged helper and passes the target project root explicitly; do not require consumers to vendor a hidden helper.
   **verify:** Run the new Bigpowers fixture test and invoke the corrected plan-work gate against CouchLink’s e01 capsule.

2. **RED:** Add a fixture whose task ledger lacks the checker-required story-spec declaration. Assert that the checker emits a CRITICAL diagnostic and exits nonzero, rather than terminating silently under shell strict mode.
   **GREEN:** Make the checker use the explicitly supplied project root for cross-artifact extraction and handle missing metadata through its existing diagnostic reporter.
   **verify:** Run the checker fixture suite and confirm the e01 invocation produces actionable CRITICAL/HIGH/MED output.

**REFACTOR:** Keep only one documented invocation contract across the Bigpowers skill and its prompt mirror. Do not add a CouchLink-local duplicate, CI change, or e01 artifact workaround.

## Acceptance Criteria

- [ ] A project using standard orchestration need not contain a Bigpowers-owned helper under its own scripts directory.
- [ ] The plan-consistency gate analyzes the target project, not the Bigpowers package directory.
- [ ] Missing required task metadata is reported as a CRITICAL finding with a nonzero exit, not a silent strict-mode termination.
- [ ] The skill text and prompt mirror use the same runnable invocation.
- [ ] CouchLink e01 receives checker findings after the tooling repair.

## Resolution

<!-- filled in by validate-fix -->

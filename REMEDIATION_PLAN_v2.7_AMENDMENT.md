# ProcessGuardian — Remediation Plan v2.7 Amendment
## Clarify `MaxRestartAttempts` as a per-session automatic-restart budget

**Revision date:** 2026-11-05  
**Base:** `REMEDIATION_PLAN.md` v2.5 + `REMEDIATION_PLAN_v2.6_AMENDMENT.md`  
**Status:** product-semantics amendment; implementation and acceptance are pending.

This is an additive amendment. Read it together with the complete v2.5 remediation plan and v2.6 amendment. This v2.7 amendment is authoritative wherever the earlier documents describe the meaning, reset rules, charging rules, or exhaustion behavior of `MaxRestartAttempts` differently.

`SPEC.md`, `AGENTS.md`, `opencode.json`, `.gitignore`, `REMEDIATION_PLAN.md`, and the v2.6 amendment remain protected. Do not edit or commit them during the implementation increment. This amendment is a separate local plan artifact and is not part of the application commit.

---

## 1. Authority and product decision

The user explicitly confirmed the following intended behavior:

> `Max restart attempts = 1` permits exactly one automatic restart of the target after the session's initial launch. If the target process exits again after that permitted restart, monitoring must terminate and publish `GuardianStatus.Error`, with the desired monitoring-enabled setting persisted as `false` on a best-effort basis.

This is a deliberate product-semantics change. It is **not** a correction that can be made while retaining the previous consecutive-failure definition.

The existing `MaxRestartAttempts` setting is now defined as:

**The maximum number of genuine automatic target-process launch attempts allowed after the session's free initial launch, counted cumulatively for that monitoring session, regardless of whether an automatic attempt successfully starts and verifies the target.**

The counter is per session. It is reset only when a new monitoring session is created. Observing the target in `Running` state does not refund, decrement, or reset a consumed attempt.

The persisted setting/property name `MaxRestartAttempts` and its existing JSON representation remain unchanged. Do not introduce a schema migration or rename the user-facing setting as part of this increment. `0` continues to mean unlimited for this budget, subject to the separate restart-window policy when that Stage 2 feature is implemented.

## 2. Exact budget semantics

### 2.1. Free initial launch

The session's initial launch remains free and is not charged to `MaxRestartAttempts`, whether it succeeds, fails, or the target is already running when monitoring begins. This preserves the existing FR-14 exception.

There is one free initial launch per new monitoring session. A later manual Stop followed by a new explicit Start creates a new session and therefore a new per-session `MaxRestartAttempts` budget. Do not move this counter to a process-wide singleton; the separate Stage 2 rolling-window history has its own lifetime and rules.

### 2.2. Budgeted automatic restarts

After the initial-launch phase, every genuine automatic process-creation attempt consumes exactly one `MaxRestartAttempts` unit **regardless of outcome**:

- `ProcessStartStatus.Started`, followed by successful startup verification (`Verified`): consumes one.
- `ProcessStartStatus.Started`, followed by failed startup verification (`VerifyFailed`): consumes one.
- A genuine process-creation failure (`ProcessStartStatus.Failed` / `NotStarted` when backed by an actual process-creation attempt): consumes one.

An attempt is charged exactly once. Do not charge once at the start of `HandleMissingAsync` and again after the typed outcome. Charge only after the implementation has established that a genuine automatic OS process-creation attempt was made.

A successful automatic restart is **not** free. If that target later exits, the next cycle sees the already-consumed budget and must not launch it again when the limit has been reached.

### 2.3. Outcomes that do not consume the budget

The following outcomes never consume `MaxRestartAttempts` because no genuine automatic process-creation attempt was made:

- target `Missing` before the process-creation boundary;
- target `ReadError` before the process-creation boundary;
- process-start gate cancellation (`Cancelled` / session cancellation);
- hash `Mismatch` (this remains its own fatal integrity cause, not a restart attempt);
- a process check of `Unknown` (no launch is attempted);
- any launch blocked before the OS process-creation call.

Do not collapse `Missing`, `ReadError`, `Cancelled`, or `Unknown` into `NotStarted` merely to simplify the counter. Preserve the typed outcome distinctions and the existing E/K/F behavior.

If an unexpected exception is thrown by the process-start adapter, inspect the actual implementation and determine whether the OS process-creation call was reached. Count it only if a genuine process-creation attempt occurred; cancellation and pre-launch failures remain uncharged. Make this decision explicit in code comments and tests rather than relying on an ambiguous `NotStarted` label.

### 2.4. Counter lifecycle and exhaustion

- The per-session counter is cumulative and monotonic: it increases by one per budgeted automatic attempt and never decreases during that session.
- `ProcessOutcome.Running` clears degraded-state tracking as before, but **must not reset the restart budget**.
- When the target is down and the consumed count is already equal to a non-zero limit, the controller must not invoke another automatic launch. It terminates the session through the existing `SessionEnd.BudgetExhausted` fatal path.
- `BudgetExhausted` continues to use the existing R-F settlement: terminate the session loop, persist the desired `IsMonitoringEnabled=false` on a best-effort basis, then publish `GuardianStatus.Error` with a stable non-empty reason and independent `SaveFailed` state.
- A full restart window remains a different mechanism: it defers a launch and exposes `IsDegraded` / `RestartDeferred`; it does not by itself terminate a session. The rolling-window counter/history is not a substitute for, or reset mechanism for, `MaxRestartAttempts`.
- `MaxRestartAttempts = 0` disables this per-session budget only. The independent Stage 2 restart-window policy remains in force when implemented. Do not change the accepted `0 + window 0` policy in this increment.

### 2.5. Concrete examples

| `MaxRestartAttempts` | Session behavior |
|---:|---|
| `0` | No per-session automatic-restart limit. |
| `1` | Initial launch is free; one automatic restart is allowed. If the target exits again, monitoring terminates with `Error`; no second automatic restart is attempted. |
| `2` | Initial launch is free; two automatic restarts are allowed. If the target exits a third time, monitoring terminates with `Error`. |
| `5` | Initial launch is free; five automatic restarts are allowed. The next target exit exhausts the budget and terminates monitoring. |

The same count applies when an allowed automatic restart fails to start or fails verification: it still consumed one genuine restart attempt. The session may continue observing according to the existing transient-failure behavior, but no additional launch is permitted once the budget is exhausted.

## 3. Explicit overrides to the v2.5 plan

This amendment supersedes earlier v2.5 wording that defines `MaxRestartAttempts` as a **consecutive-failed-launch counter**, resets it when `Running` is observed, or says that successful automatic relaunches do not increment it. In particular, update the interpretation of these v2.5 passages for implementation/review:

- §2 Decision “Error semantics”: “Consecutive-budget exhaustion” means exhaustion of the cumulative per-session automatic-restart budget defined here; it is no longer a consecutive-failure budget.
- §E “Budget charge point”: preserve the exclusions for `Missing`, `ReadError`, `Mismatch`, and `Cancelled`, but charge every genuine automatic OS process-creation attempt, whether it succeeds or fails.
- §F fatal-cause list and `StartAsync` / `VerifyStartupAsync` failure wording: exhaustion is cumulative across successful and failed automatic restarts; successful `Running` observations do not reset the budget.
- FR-14: retain the free initial launch, but every subsequent genuine automatic launch attempt consumes one per-session unit regardless of outcome.
- Stage 2 §I: retain a distinct rolling-window budget that counts automatic launches over its time window. Remove any implication that the per-session `MaxRestartAttempts` counter is consecutive, resettable by `Running`, or limited to failures. The window's own lifetime and deferral semantics are unchanged.
- Review notes and any test descriptions that currently say “consecutive counter”, “resets on Running”, “successful relaunch does not consume MaxRestartAttempts”, or equivalent are superseded by this amendment.

The v2.5 rules that remain unchanged include: initial launch is free; Missing/ReadError/Unknown/Cancelled do not spend a launch; hash Mismatch is separately fatal; `0` disables this budget; budget exhaustion is fatal; the rolling restart-window behavior is distinct and non-fatal when full.

## 4. Implementation increment — `R-AS-RESTART-BUDGET`

**Scope:** `GuardianOrchestrator` and the minimum directly related Application tests. Touch Domain/Infrastructure only if inspecting the process-start boundary proves that the current typed result contract cannot reliably distinguish a genuine process-creation attempt from Missing/ReadError/Cancelled. Do not refactor unrelated process-monitoring code.

Required implementation:

1. Inspect `RunSessionAsync`, `HandleMissingAsync`, `StartAndVerifyAsync`, `SessionCounters`, `ProcessStartResult`, and the concrete `IProcessMonitor.TryStart` implementation.
2. Change `MaxRestartAttempts` from the consecutive-failure counter to the cumulative per-session automatic-restart budget described in §2.
3. Preserve the free initial launch exactly.
4. Charge each genuine automatic process-creation attempt exactly once regardless of success/failure; do not charge attempts blocked before process creation.
5. Remove the budget reset from the `ProcessOutcome.Running` branch. Continue resetting degraded-state tracking there.
6. Prevent another automatic launch once the non-zero budget is exhausted. Use the existing `BudgetExhausted` / R-F terminal settlement path; do not invent a second Error/persistence path.
7. Preserve `RestartDelaySeconds`, check interval, startup verification, toast behavior, cancellation, hash-integrity behavior, and all unrelated state transitions unless a narrowly required correction is demonstrated and documented.
8. Preserve the distinct Stage 2 rolling-window policy and do not implement Stage 2 in this increment.

## 5. Required regression tests

Use deterministic test seams where practical. Do not use wall-clock sleeps to prove counter semantics when a controlled fake/time seam or synchronization hook can establish the ordering.

At minimum, cover:

1. Initial successful launch is free; no budget is consumed.
2. Initial failed launch remains free, preserving FR-14.
3. `MaxRestartAttempts=1`: one successful automatic restart is allowed; observing `Running` afterward does not reset the consumed count; when the target subsequently becomes `NotRunning`, the session terminates with `Error` and no second automatic `TryStart` occurs.
4. `MaxRestartAttempts=1`: one genuine automatic start failure consumes the only budget unit; no further automatic launch occurs; the fatal outcome is settled through R-F.
5. `MaxRestartAttempts=2`: exactly two genuine automatic relaunch attempts are allowed, with success/failure combinations; the next down-state causes exhaustion.
6. Target is already `Running` when the session begins: this does not consume the free initial launch or the automatic-restart budget; its first later exit allows one automatic restart when Max=1.
7. `MaxRestartAttempts=0`: the per-session budget does not exhaust, while other protections remain unaffected.
8. `Missing` at the launch boundary does not consume the budget and does not terminate the session.
9. `ReadError` at the launch boundary does not consume the budget and does not terminate the session.
10. `Unknown` process check does not consume the budget and does not invoke a launch.
11. `Cancelled` before process creation does not consume the budget or publish `Error`.
12. Hash `Mismatch` still terminates through its own fatal path and does not get misreported as budget exhaustion.
13. A new explicit monitoring session starts with a fresh per-session budget.
14. A successful restart followed by several `Running` checks and then a later process exit still exhausts Max=1; repeated `Running` observations never refund the spent attempt.

Assert both the state/persistence result and the exact number of `TryStart`/actual process-creation calls. Assert that no extra launch is attempted after exhaustion. Keep tests for the independent rolling-window feature unchanged; do not add Stage 2 implementation.

## 6. Non-goals and protected scope

Do not modify:

- `SPEC.md`;
- `AGENTS.md`;
- `opencode.json`;
- `.gitignore`;
- `REMEDIATION_PLAN.md` v2.5;
- `REMEDIATION_PLAN_v2.6_AMENDMENT.md`;
- UI labels, layout, or settings JSON/property names;
- `MaxRestartsPerWindow` or Stage 2;
- autostart registration/logging, temporary XML handling, first-run configuration recovery, ACLs, or recovery journal;
- unrelated monitoring, target validation, notification, or logging behavior.

If the inspected process-start implementation cannot distinguish a genuine OS launch attempt from a pre-launch failure, do not guess and do not weaken the Missing/ReadError/Cancelled rules. Make the smallest necessary typed-contract correction, add focused tests, and explain it in the report.

## 7. Build, test, Git, and report requirements

Follow `AGENTS.md` exactly. Before editing, inspect `git status --short` and the complete relevant diff; treat all existing uncommitted changes as user-owned.

Run the four project gates:

```powershell
dotnet build .\ProcessGuardian.sln -c Release
dotnet build .\ProcessGuardian.sln -c Release-Elevated
dotnet test  .\ProcessGuardian.sln -c Release
dotnet test  .\ProcessGuardian.sln -c Release-Elevated
```

Require exit code 0, zero warnings, zero errors, and all tests passing in both configurations. Inspect the full diff and `git status`; stage only files required by this increment. Create one focused commit:

```text
R-AS-RESTART-BUDGET: count all automatic restarts per session
```

Do not commit any plan/amendment, protected file, unrelated change, or temporary artifact. Do not push.

The completion report must include:

- the old versus new budget semantics;
- exact point and classification used to charge an attempt;
- confirmation that `Running` no longer resets the counter;
- behavior for initial launch, successful/failed automatic restart, Missing/ReadError/Unknown/Cancelled/Mismatch, `0`, and a new session;
- files changed and tests added/updated;
- the complete Release/Release-Elevated build/test matrix;
- reviewed `git status`/diff and protected-file confirmation;
- commit hash/message;
- remaining limitations and exact manual verification steps.

Do not claim manual verification unless it was actually performed.

## 8. Manual acceptance scenario

On a real Windows machine, use a harmless target executable that can be closed and relaunched safely:

1. Set `Max restart attempts = 1` and start monitoring. The initial launch must not consume the budget.
2. Close the target once. ProcessGuardian must relaunch it successfully.
3. Allow ProcessGuardian to observe the relaunched target as running for several check cycles.
4. Close the target a second time. ProcessGuardian must **not** relaunch it again. The session must terminate, the UI/tray must show `ERROR`, and the desired persisted `IsMonitoringEnabled` must be `false` (unless a save failure is separately reported).
5. Start a new session explicitly. Its per-session budget must be fresh.
6. Repeat with `Max restart attempts = 0`; this budget alone must not terminate monitoring, subject to any separately implemented restart-window protection.

Stage 2 remains locked. Do not proceed to any other increment until this increment passes review and the user accepts it.

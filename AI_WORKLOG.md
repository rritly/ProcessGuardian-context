# AI Worklog

Append-only journal of completed agent work. The agent writes a new entry here
ONLY on an explicit user command (see AGENTS.md); each entry documents one
completed unit of work: what changed, tests/build evidence, and any open items.

---

## Work 1 — Stage 1 corrective increments (v2.5 plan + v2.6 amendment)

**Date:** 2026-11-04 / 2026-11-05 · **Base:** `0949528` (main) · **Status:** completed (conditionally accepted — see Open items)

### Scope

Six corrective works per `REMEDIATION_PLAN.md` (v2.5) + `REMEDIATION_PLAN_v2.6_AMENDMENT.md`:
R-AS-ENC, R-AS-GATE, R-AS-STATE, R-A-ERROR-STATE, R-DOCS-TOAST, plus a test-guard fix.
No push, no Stage 2, protected files untouched.

### Commits (atomic, in order)

| # | Increment | Hash | Files |
|---|-----------|------|-------|
| 1 | R-AS-ENC | `e3ec540` | `src/ProcessGuardian.Infrastructure/Autostart/ScheduledTaskAutostartService.cs` (+54), new `tests/ProcessGuardian.Infrastructure.Tests/ScheduledTaskAutostartEncodingTests.cs` (7 tests) |
| 2 | R-AS-GATE | `e63d31f` | `src/ProcessGuardian.Application/Services/GuardianOrchestrator.cs`, `src/ProcessGuardian.Wpf/ViewModels/MainViewModel.cs`, `tests/ProcessGuardian.Application.Tests/GuardianAutomaticRecoveryTests.cs`, `tests/ProcessGuardian.Wpf.Tests/MainViewModelRecoveryTests.cs` |
| 3 | R-AS-STATE | `0fe7d53` | `src/ProcessGuardian.Application/Services/GuardianOrchestrator.cs`, `tests/ProcessGuardian.Application.Tests/GuardianAutomaticRecoveryTests.cs` |
| 4 | R-A-ERROR-STATE | `ed8d5b8` | `src/ProcessGuardian.Application/Services/GuardianOrchestrator.cs`, `tests/ProcessGuardian.Application.Tests/GuardianOrchestratorTests.cs` |
| 5 | R-DOCS-TOAST | `77db5e4` | `README.md` |
| 6 | TEST: journal guard | `0b3997c` | `tests/ProcessGuardian.Infrastructure.Tests/RegistryRecoveryJournalTests.cs` |

### What was fixed

1. **R-AS-ENC** — `schtasks.exe` redirected output is now decoded with the system OEM
   code page resolved at runtime via `GetOEMCP()` (never a hard-coded locale page);
   established by raw-byte evidence: schtasks writes with its own fresh windowless
   console's default page = system OEM (CP866 here; 0x98 in the captured stderr rules
   out CP1251). Previously a windowless WPF process default-decoded CP866 bytes as
   UTF-8 → mojibake → `LooksLikeMissingTask` never matched → verified `Absent`
   silently downgraded to `Unknown`. Fallback to UTF-8 with `Debug.Fail` if the OEM
   page cannot be resolved; separate stdout/stderr capture, bounded logging and
   fail-closed classification unchanged.
2. **R-AS-GATE** — while automatic recovery is pending (`_recoveryPending`), every
   user-originated settings commit is rejected at the controller level before any
   mutation (no `SaveAsync`, no autostart mutation, no committed path/pin change);
   the recovery-owned save keeps its own path (`PersistRecoveryStateAsync`). All UI
   gates closed while pending: `IsSettingsEditable`/`IsTargetPathEditable` false
   (path TextBox, Browse, numeric fields, EnableToasts), autostart checkbox locked,
   debounce cancelled on gate engagement, bindings/CanExecute notified, guards in
   `BrowseExecutable`/`CommitTargetPathDraft`/`OnSettingsPropertyChanged`/`CommitAsync`/
   `OnAutostartEffectiveChanged`. Gates reopen only on the controller's fully verified
   `RecoveryPendingChanged`; they stay closed forever after recovery exhaustion.
3. **R-AS-STATE** — exhausted recovery (3 attempts) now publishes a locked
   recovery-failure `Error` (never a `Stopped` look-alike): `IsDegraded=false`,
   stable non-empty `Reason` (`RecoveryFailedReason`), `SaveFailed=true` only when
   the settings save itself failed. `RunAutomaticRecoveryAsync` returns an explicit
   typed outcome (`AutomaticRecoveryResult(Completed, SaveFailed)`); no background
   retries. Exactly one 1-second cancellable delay between failed attempts (max 2;
   none before the first / after success / after the final failure), resolved through
   the injected `TimeProvider` — no wall-clock sleeps in tests.
4. **R-A-ERROR-STATE** — the `_configRefused` branches of `StopAsync`/`SuspendAsync`
   keep the refused `Error` state intact (existing reason preserved,
   `IsDegraded=false`, no save, no autostart mutation, no `Stopped` publication);
   test renamed to `ConfigRefused_StopAndSuspend_DoNotPersist_DoNotTouchAutostart_ErrorPreserved`.
5. **R-DOCS-TOAST** — README: removed the `Target Unreadable` toast row; the toast
   set is now exactly the five standard titles (verified against all `ShowAsync`
   call sites); target-integrity text corrected (fail-closed Start rejection, no
   toast emitted); no A/B recovery remnants remain in the touched sections.
6. **TEST (journal guard)** — fixed the environment-dependent pre-existing FAIL
   `ReadAsync_MissingKey_ReturnsNone_NotRefused`: on this machine the
   `HKLM\SOFTWARE\ProcessGuardianAdmin` key exists (created by the manual elevated
   checklist) with its protected Administrators/SYSTEM-only DACL, so the guard's
   `OpenSubKey` existence probe threw `SecurityException` on a non-elevated run.
   The guard now treats access-denied as "key present" and yields; the
   `RecoveryState.None` assertion is unchanged and runs on clean runners.

### Final verification matrix (v2.6 gate commands, builds with `--no-incremental`)

| Command | Result | Exit code |
|---------|--------|-----------|
| `dotnet build .\ProcessGuardian.sln -c Release` | 0 warnings / 0 errors | 0 |
| `dotnet build .\ProcessGuardian.sln -c Release-Elevated` | 0 warnings / 0 errors | 0 |
| `dotnet test .\ProcessGuardian.sln -c Release` | 178/178 Infrastructure + 41/41 Wpf + 82/82 Application = 301/301 | 0 |
| `dotnet test .\ProcessGuardian.sln -c Release-Elevated` | 178/178 + 41/41 + 82/82 = 301/301 | 0 |

Re-verification of closed v2.4 requirements (targeted runs, all PASS):
R-KE / R-CD / R-F → `GuardianOrchestratorTests` 54/54 + `WindowsProcessMonitorTests`
(within the 178/178 full run); R-DOCS → Wpf indicator/target-path tests 17/17.

### Notes / deviations

- .NET 10 `TimeProvider.CreateTimer` signature is `(TimerCallback, object?, TimeSpan,
  TimeSpan)` (probe-verified); the test fake timer is written against it.
- The delay-cancellation test uses a short polling loop (≤10 s budget, <100 ms in
  practice) instead of a deterministic sync point — a compromise against real sleeps.

### Open items (manual verification)

- **DPI toast check NOT performed** (user deferred): real visual Windows check at
  100 / 125 / 150 / 200 % scaling — all five titles fully visible, no clipping,
  card ≈ 191×61 DIP; record Windows version, scale, per-title results. Required by
  Amendment F before full Stage 1 acceptance; until then Stage 1 is conditionally
  accepted.

### Protected files / git hygiene

`SPEC.md`, `AGENTS.md`, `opencode.json`, `.gitignore` — never modified or committed
during these works. No push, no history rewriting. Final tree clean except
intentionally untracked files.

---

## Work 2 — R-AS-XML-VERIFY: honor Task Scheduler default enabled state

**Date:** 2026-11-05 · **Base:** `0b3997c` (main) · **Commit:** `d036bc4` · **Status:** completed

### Goal

Fix the confirmed defect in `ScheduledTaskAutostartService` where the admin edition
could not verify a successfully created autostart task: `SetEnabledAsync(true)`
reported "the resulting autostart state could not be verified" after `/Create`
exited 0.

### Root cause (live-evidence confirmed)

A real `/Query /XML` export of the successfully created `ProcessGuardianAdmin`
task (captured by the user on live Windows, 2026-11-05) contains NO `<Enabled>`
element inside `<Settings>`. Per the Windows Task Scheduler schema,
`<Settings/Enabled>` is OPTIONAL and defaults to `true` when the `<Settings>`
element itself is present. The parser left `TaskDescriptor.Enabled = null`,
`Evaluate()` classified that as `Unknown`, and the post-create verification
failed even though the task existed and the query exited 0.

### Fix (`src/ProcessGuardian.Infrastructure/Autostart/ScheduledTaskAutostartService.cs`)

`ParseTaskXml()` now applies the schema default:

- `<Settings>` present without an `<Enabled>` child → `Enabled = true`;
- explicit `true` / `false` → parsed as before;
- `<Enabled>` with an unparsable boolean → fail-closed `Unknown` preserved, plus
  a new bounded `LogWarning` that records only the raw invalid element value
  (never the whole exported task XML); `ParseTaskXml` gained an optional
  `ILogger?` parameter, wired from `ReadStateDetailedAsync`;
- NO `<Settings>` element at all → the schema-default rule is NOT extended to
  this case; the state stays `Unknown`.

Unchanged: exact-Command / `--background` / LogonTrigger / InteractiveToken /
HighestAvailable checks; the post-operation `/Query /XML` (not the `/Create`
exit code) remains the only source of a verified state.

### Tests

New `tests/ProcessGuardian.Infrastructure.Tests/ScheduledTaskAutostartXmlVerifyTests.cs`
(8 tests) built around the verbatim real Windows export:

- real export without `<Enabled>`: descriptor `Enabled=true`, all mismatch
  checks pass (quoted `<Command>` included), `Evaluate` → `Enabled`;
- explicit `true` → `Enabled`; explicit `false` → `Disabled`;
- unparsable `Enabled` value → `Unknown`, warning contains the invalid value
  but never the task XML;
- stripped `<Settings>` block → `Unknown` (never auto-Enabled);
- seam integration test: `Absent → /Create (exit 0) → verification /Query /XML
  without <Enabled> → SetEnabledAsync(true) reports Success(Enabled)`.

Updated the three tests that pinned the defective `Unknown` behavior to the
schema-default semantics (`Evaluate_MissingEnabledElement_WithinSettings_
DefaultsToEnabled`, `Evaluate_RealSchedulerExport_MissingEnabledElement_
DefaultsToEnabled`, `ReadState_MissingEnabledElement_DefaultsToEnabled`) and
added no-`<Settings>` → `Unknown` coverage next to each. All tests run through
the `SchTasksInvoker` seam; no real scheduled task is touched.

### Verification matrix (4 gate commands)

| Command | Result | Exit |
|---------|--------|------|
| `dotnet build .\ProcessGuardian.sln -c Release` | 0 warnings / 0 errors | 0 |
| `dotnet build .\ProcessGuardian.sln -c Release-Elevated` | 0 warnings / 0 errors | 0 |
| `dotnet test .\ProcessGuardian.sln -c Release` | 188/188 Infra + 41/41 Wpf + 82/82 App = 311/311 | 0 |
| `dotnet test .\ProcessGuardian.sln -c Release-Elevated` | 311/311 | 0 |

Post-commit targeted run of the new fixture: 8/8, 38 ms.

### Open items (manual verification)

- Toggle "Autostart with Windows" off/on in the admin app on real Windows: the
  checkbox must enable without the "state could not be verified" report.
- DPI toast check (100/125/150/200 %) — still open from Work 1; Stage 1 remains
  conditionally accepted until performed.

### Protected files / git hygiene

`SPEC.md`, `AGENTS.md`, `opencode.json`, `.gitignore`, `AI_WORKLOG.md` — not
modified or committed. No push, no history rewriting. The user's existing
`ProcessGuardianAdmin` scheduled task was never touched from this work.

---

## Work 3 — R-AS-POWER-POLICY: remove battery power restrictions from autostart

**Date:** 2026-11-05 · **Base:** `d036bc4` (main) · **Commit:** `cefffc0` · **Status:** completed

### Goal

Fix the confirmed power-policy defect in `ScheduledTaskAutostartService`: the
`ProcessGuardianAdmin` autostart task must start at user logon regardless of the
power source and must not be stopped when the machine switches to battery power.

### Root cause (live-evidence confirmed)

A real `/Query /XML` export of the user's `ProcessGuardianAdmin` task showed:

```xml
<DisallowStartIfOnBatteries>true</DisallowStartIfOnBatteries>
<StopIfGoingOnBatteries>true</StopIfGoingOnBatteries>
```

Per Microsoft documentation (TaskSettings.DisallowStartIfOnBatteries property,
MSDN/Win32), the schema default for both flags is `true` — meaning the task
**will not start** if running on battery and **will stop** if the charger is
disconnected. `schtasks.exe /Create` (the previous registration path) has no
command-line switches for these Task Settings; the `/TR` command-line variant
therefore left both at their default `true` values. `ParseTaskXml()` did not
read or verify these flags, so the task was classified as `Enabled` despite
the power restrictions.

### Fix (`ScheduledTaskAutostartService.cs`)

**Registration (new `CreateTaskAsync` / `BuildTaskDefinitionXml`):**

`schtasks.exe /Create` has no `/SC`/`/RL`-equivalent switches for the battery
power settings. The only schtasks facility that can set them is `/Create /XML`
import. A full Task Scheduler 2.0 definition XML is built programmatically
(`BuildTaskDefinitionXml`):

- `<DisallowStartIfOnBatteries>false</DisallowStartIfOnBatteries>`
- `<StopIfGoingOnBatteries>false</StopIfGoingOnBatteries>`
- LogonTrigger, InteractiveToken, HighestAvailable, quoted executable with
  `--background`, per-call unique temp path, `Encoding.Unicode` (utf-16)
  matching the XML declaration — XElement construction provides automatic
  escaping of special characters in the executable path.
- Principal `UserId` = `{Environment.UserDomainName}\{Environment.UserName}`
  resolved at runtime — nothing machine-specific is hard-coded.

The temp file is always deleted in a `finally` block (success, failure,
cancellation). `BuildCreateArguments` now accepts the definition file path
(and exports `/XML "<path>" /F`).

**Parsing (`ParseTaskXml`):**

`TaskDescriptor` gained `DisallowStartIfOnBatteries` and
`StopIfGoingOnBatteries` fields. Both are parsed from `<Settings>`:

- Present + parsable boolean → the value
- Present + unparsable → `null` (fail-closed), bounded `LogWarning` with only
  the raw element value
- Absent → `null` (no schema default applied — Windows defaults both to `true`,
  so absence is never proof of compliance; `GetMismatches` catches this)

**Verification (`GetMismatches`):**

Both flags must be explicitly `false`. Any other value (including `null`)
produces a `TaskMismatch` with `Expected="false"` and `Actual=<value or "<missing>">`.
The existing invariants (Command, --background, LogonTrigger, InteractiveToken,
HighestAvailable) are unchanged.

**Existing repair path reused:** `SetEnabledAsync(true)` already skips the
"already Enabled" fast-path for `Misconfigured` tasks, so the repair path
(`/Create /F`) runs automatically for an already-enabled task whose battery
policy is wrong. The post-registration `/Query /XML` verification confirms the
actual flags, never trusts the `/Create` exit code alone.

### Tests

New `tests/ProcessGuardian.Infrastructure.Tests/ScheduledTaskAutostartPowerPolicyTests.cs`
(24 tests), all through the `SchTasksInvoker` seam:

- **XML parsing** (7): both flags explicit false → both false; each flag
  individually true → parsed as true; each flag individually absent → null
  (no schema default); unparsable values → null, bounded warnings contain only
  the raw invalid value; unparsable XML → null (existing fail-closed semantics
  preserved).
- **State evaluation** (5): compliant Enabled → Enabled; Enabled + each battery
  restriction individually → Misconfigured; both restrictions + correct
  parameters → Misconfigured (battery mismatch not masked by other correct
  values).
- **Mismatch diagnostics** (3): `true` actuals, missing actuals (`"<missing>"`),
  unparsable actuals.
- **Definition XML** (6): both flags `false` in built XML; round-trip through
  production parser → 0 mismatches, Enabled state; XML special characters in
  exe path are escaped (`&` → `&amp;`); exactly one XML declaration matching
  utf-16 encoding; `UserId` is runtime identity.
- **Registration / repair / verification** (3): already-Enabled battery-restricted
  task → re-registered via `/Create /XML` + `/F`, Verified Enabled; create exits 0
  but verification still shows battery restriction → fails as Misconfigured;
  create nonzero → failure handled by existing error mechanism.

Updated `ScheduledTaskAutostartXmlVerifyTests.cs`: 3 tests that previously
asserted "all checks pass" / "Enabled" for the verbatim real export (which has
both battery flags true) now assert `Misconfigured` + exact failing check names;
`Evaluate_RealWindowsExport_WithoutSettingsElement` changed from `Unknown` to
`Misconfigured` (absent battery flags are a mismatch per 4.3 requirement).

Updated `ScheduledTaskAutostartServiceTests.cs` and
`ScheduledTaskAutostartServiceErrorTests.cs`: `TaskXml` helper extended with
`disallow`/`stop` parameters (default to `"false"` = compliant); the missing-
`<Settings>` test in both files now asserts `Misconfigured` (battery flags are
unverifiable = mismatch); `ParseTaskXml_ParsesExpectedConfiguration_ToEnabledDescriptor`
asserts both battery fields false.

### Verification matrix (4 gate commands)

| Command | Result | Exit |
|---------|--------|------|
| `dotnet build .\ProcessGuardian.sln -c Release` | 0 warnings / 0 errors | 0 |
| `dotnet build .\ProcessGuardian.sln -c Release-Elevated` | 0 warnings / 0 errors | 0 |
| `dotnet test .\ProcessGuardian.sln -c Release` | 213/213 Infra + 41/41 Wpf + 82/82 App = 336/336 | 0 |
| `dotnet test .\ProcessGuardian.sln -c Release-Elevated` | 336/336 | 0 |

### Open items (manual verification)

- Toggle "Autostart with Windows" off/on in the admin app on real Windows (the
  old battery-restricted task should be automatically replaced by the new
  compliant one; the checkbox must enable without error).
- `schtasks.exe /Query /TN "ProcessGuardianAdmin" /XML` after enabling: both
  `<DisallowStartIfOnBatteries>` and `<StopIfGoingOnBatteries>` must be `false`.
- On a real laptop: unplug charger → log off → log on → ProcessGuardianAdmin
  must start automatically.
- On a real laptop: with ProcessGuardianAdmin running, unplug the charger →
  the app must NOT be stopped.
- DPI toast check (100/125/150/200 %) — still open from Work 1; Stage 1 remains
  conditionally accepted until performed.

### Side note (no fix applied, documented only)

`SPEC.md` AS-06 describes the registration as `schtasks.exe /Create ... /SC
ONLOGON /RL HIGHEST /F` (the old `/TR` command-line form). The text is now
factually incorrect. `SPEC.md` is protected and was not edited.

### Protected files / git hygiene

`SPEC.md`, `AGENTS.md`, `opencode.json`, `.gitignore`, `AI_WORKLOG.md` — not
modified or committed. No push, no history rewriting. The user's existing
`ProcessGuardianAdmin` scheduled task was not modified directly; enabling the
checkbox will replace it via `/Create /F`.

---

## Work 4 — R-AS-RESTART-BUDGET: count all automatic restarts per session

**Date:** 2026-11-05/06 · **Base:** `cefffc0` (main) · **Commit:** `db1fc5b` · **Status:** completed (pending manual acceptance)

### Goal

Change `MaxRestartAttempts` from a consecutive-failed-launch counter into a
cumulative per-session budget of genuine automatic target-process launch
attempts, per the user-authorized `plan\REMEDIATION_PLAN_v2.7_AMENDMENT.md`
(§2 budget semantics, §4 implementation increment, §5 regression matrix).

### Old vs new semantics

- **Old (v2.5 E):** `RestartAttempts` incremented only for `NotStarted`/`VerifyFailed`
  and reset to `0` on every `ProcessOutcome.Running` observation; a successful
  automatic relaunch never consumed the budget.
- **New (v2.7):** the counter is cumulative and monotonic per session; observing
  `Running` never refunds a spent unit; every genuine automatic process-creation
  attempt consumes exactly one unit regardless of outcome; only a newly created
  session starts at zero.

### Launch-charge classification (boundary inspected, not inferred)

Inspected `WindowsProcessMonitor.TryStart`, `ProcessStartResult`, `StartAndVerifyAsync`,
`HandleMissingAsync`, `SessionCounters`:

- `ProcessStartStatus.Started` (verify `Verified` or `VerifyFailed`) → **charges 1**.
- `ProcessStartStatus.Failed` — genuine creation failure: null from `Process.Start`,
  Win32 codes other than 2/3/5/32, and any unexpected exception inside the adapter's
  launch boundary (the concrete adapter converts all of them to typed results) → **charges 1**.
- `Missing`/`ReadError` (blocked before the OS creation boundary), hash `Mismatch`,
  `Cancelled`, session `Unknown`, and gate-blocked launches → **uncharged** (typed
  semantics preserved).
- Adapter exception escaping to the orchestrator is **indeterminate** → new typed
  `StartOutcome.LaunchError`, uncharged (fail-safe: never spends the budget on an
  unobserved attempt), session survives with the restart delay. Domain/Infrastructure
  contracts unchanged.

Charged exactly once in `HandleMissingAsync`, after the typed outcome, never before
the hash/gate checks. `SessionEnd.BudgetExhausted` reuses the existing R-F settlement
(persist enabled=false best-effort → publish Error with independent SaveFailed); the
fatal reason text was updated to cumulative wording ("the per-session automatic
restart budget is exhausted...") — no test pinned the old text.

### Tests

Rewritten (superseded consecutive semantics): `SuccessfulRelaunch_DoesNotAdvance
ConsecutiveBudget_NoPrematureError` → `SuccessfulRestart_ConsumesBudget_Running
DoesNotRefund_SecondExitExhausts` (Max=1: free initial launch, one successful
budgeted restart, several Running observations refund nothing, second exit → Error,
exactly 2 TryStart).

Added: `InitialRunningTarget_DoesNotConsumeBudget_FirstExitAllowsOneRestart` (§5.6),
`Max2_AllowsExactlyTwoAutomaticAttempts_ThenExhausts` (§5.5, asserts 3 TryStart +
R-F SaveAsync), `Max0_BudgetNeverExhausts` (§5.7), `LaunchError_AdapterThrows_
Uncharged_SessionSurvives`. Preserved unchanged (still valid): free-initial-launch,
Missing/ReadError/Unknown/Cancelled/Mismatch boundaries, fresh budget per new session
(`NewStartAfterError...`), `Exhaustion_SaveThrows...` (comment updated). Exact
`TryStart` counts and persistence asserted via `Received(n)`; deterministic
`WaitUntilAsync` polling, no wall-clock sleeps for counter semantics.

### Verification matrix (4 gate commands)

| Command | Result | Exit |
|---------|--------|------|
| `dotnet build .\ProcessGuardian.sln -c Release` | 0 warnings / 0 errors | 0 |
| `dotnet build .\ProcessGuardian.sln -c Release-Elevated` | 0 warnings / 0 errors | 0 |
| `dotnet test .\ProcessGuardian.sln -c Release` | 213/213 Infra + 41/41 Wpf + 86/86 App = 340/340 | 0 |
| `dotnet test .\ProcessGuardian.sln -c Release-Elevated` | 340/340 | 0 |

### Housekeeping (same period, user command)

Local `plan\` directory created at the repo root and `REMEDIATION_PLAN.md` +
`REMEDIATION_PLAN_v2.6_AMENDMENT.md` + the v2.7 amendment moved there from
`~\.opencode\plan`; `AGENTS.md` §3 now documents `\plan` as a protected,
never-committed planning artifact. Untracked, uncommitted (protected).

### Open items (manual verification)

- §8 manual acceptance scenario (real Windows, harmless target exe): Max=1 → close
  target twice; second close must terminate with ERROR and persisted enabled=false;
  new session must have a fresh budget; Max=0 must not terminate by this budget.
- Prior open items unchanged: autostart toggle + battery flags check, charger
  unplug scenarios (Work 3), DPI toast check at 100/125/150/200 % (Work 1).

### Protected files / git hygiene

`SPEC.md`, `AGENTS.md`, `opencode.json`, `.gitignore`, and the plan files were not
modified or committed; no push, no history rewriting. Commit `db1fc5b` contains only
`GuardianOrchestrator.cs` + `GuardianOrchestratorTests.cs` (+207/−27).

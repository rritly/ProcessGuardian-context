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

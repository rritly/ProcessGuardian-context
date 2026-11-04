# ProcessGuardian — Remediation Plan v2.6 Amendment
## Stage 1 recovery-state, recovery-gate, retry, diagnostics, and closure corrections

**Revision date:** 2026-11-03  
**Base:** `REMEDIATION_PLAN.md` v2.5 
**Status:** corrective amendment; implementation and acceptance are pending.

This is an additive amendment to v2.5. It does not replace the original v2.4 requirements or unrelated v2.5 decisions. The clauses below are normative and override only the explicitly identified conflicting v2.5 requirements. Keep the original plan and append this amendment, or maintain this file beside it and always read both as one plan.

`SPEC.md`, `AGENTS.md`, `opencode.json`, `.gitignore`, and the plan remain protected from ordinary implementation commits. Do not edit `SPEC.md` as part of this amendment. Do not commit this plan amendment to the application repository.

---

## 1. Authority and decisions

The current `main` branch contains the five v2.5 corrective commits in the prescribed order:

1. `02e8f674` — `R-A2`
2. `a446c692` — `R-AS-DIAG`
3. `34894f47` — `R-AS-AUTO`
4. `7f9dba02` — `R-AS-TXN`
5. `0949528b` — `R-UI-TOAST`

Their presence does not mean Stage 1 is accepted. The corrections below close remaining deviations and clarify one product-state requirement.

The accepted product decisions for this amendment are:

- An automatic recovery that completes successfully leaves monitoring `STOPPED`, with the recovery journal cleared and ordinary controls unlocked.
- If all three recovery attempts are exhausted and the journal remains pending, publish `GuardianStatus.Error`, not `Stopped`. Keep `_recoveryPending=true`; the journal remains pending. This `Error` is a locked recovery-failure state, not permission to retry through the ordinary Start command.
- While recovery is pending, every normal settings edit and commit is prohibited. This includes the target path, Browse, numeric values, Toast setting, and autostart. No user-originated commit may mutate committed settings, target path, pin, persistence, or scheduler state.
- Retry attempts remain bounded to three per startup, but failed attempts must be separated by a short cancellable delay.
- Task Scheduler output must be decoded correctly before classification. Do not guess a universal Windows code page and do not weaken fail-closed classification.
- A configuration refusal remains `Error`; `StopAsync` and `SuspendAsync` must not overwrite that state with `Stopped`.
- The five-title Toast policy remains unchanged. README documentation and live DPI verification must be brought into line with it.

No new `GuardianStatus` enum value is introduced.

## 2. Amendment A — exhausted recovery is ERROR

**Supersedes:** v2.5 §2.4 and every v2.5 statement that says an exhausted/unresolved recovery must remain `STOPPED`, including §3.1 and the pending-recovery failure check in §6.1.

### Required behavior

The controller must distinguish a successfully completed recovery from an exhausted, still-pending recovery.

**Successful recovery:**

- The app-owned scheduled task has been positively verified absent.
- `AutostartEnabled=false` and `IsMonitoringEnabled=false` have been persisted successfully.
- The recovery journal has been cleared successfully.
- `_recoveryPending` becomes `false`.
- Publish `GuardianStatus.Stopped`, `IsDegraded=false`.
- The UI may unlock normal settings and autostart according to the existing v2.5 rules.

**Recovery exhausted:**

- After the third failed delete/verify/save/journal-acknowledgement attempt, leave the journal pending.
- Keep `_recoveryPending=true`.
- Publish `GuardianStatus.Error`, `IsDegraded=false`.
- Supply a stable, non-empty `MonitorState.Reason`, for example: `Automatic recovery is incomplete; see the log for details.` Do not put raw command output or long diagnostics into the status text.
- Preserve detailed failed-step information, exit codes, bounded stdout/stderr, and exceptions in the normal log.
- The `SaveFailed` field is independent. Set it only when the recovery settings persistence itself failed; do not use it as a generic recovery-failure flag.
- Do not clear the journal, recreate/enable autostart, start monitoring, or accept ordinary settings commits.
- A subsequent application startup retries the same idempotent recovery flow.
- `Start` must remain unavailable in both the ViewModel and controller. `GuardianStatus.Error` normally allows a new Start, so the independent `_recoveryPending` gate remains mandatory.

The controller should receive an explicit outcome from `RunAutomaticRecoveryAsync` (a small result type or equivalent), rather than infer success from the final published status. The outcome must allow `ReconcileAsync` to publish `Stopped` on success and `Error` on exhausted failure, with a non-empty reason and the correct independent persistence-failure flag.

Do not create a new background retry service. Do not clear `_recoveryPending` merely because the three attempts have ended.

### Tests

Update `GuardianAutomaticRecoveryTests`:

- Every exhausted-recovery scenario asserts `RecoveryPending == true` and `CurrentState.Status == Error`.
- Assert `IsDegraded == false` and a non-empty recovery-failure reason.
- Assert Start is rejected by the controller after recovery exhaustion and performs no hash read, autostart mutation, or process start.
- Assert the journal is not cleared after any failed recovery transaction.
- Assert the success path still ends in `Stopped`, clears the journal last, and reopens the gate.
- Cover failure at task removal/verification, settings persistence, and journal acknowledgement. `SaveFailed` must be true only for a settings-save failure.

## 3. Amendment B — no settings edits or commits while recovery is pending

**Authority:** v2.5 §2.1 already says ordinary settings editing becomes available only after the recovery transaction has been fully verified. The implementation must be corrected to enforce that rule in both the UI and the controller.

### ViewModel and XAML

While `_recoveryPending` is true, disable:

- target path TextBox;
- Browse button;
- all numeric settings;
- `EnableToasts`;
- autostart checkbox (retain the existing separate actual-state and editability rules);
- any other user-editable persistent setting.

At minimum, the derived UI gates must include the pending flag:

- `IsTargetPathEditable` is false when `_recoveryPending`;
- `IsSettingsEditable` is false when `_recoveryPending`;
- `CanStart` remains false when `_recoveryPending`, including when the displayed status is `Error`.

When the pending flag is set or cleared, raise the necessary property-change and command notifications so all bindings update immediately. On recovery completion, unlock only after the controller reports the fully verified outcome.

Also guard the non-visual entry points:

- `BrowseExecutable` must reject while recovery is pending.
- `CommitTargetPathDraft` must reject while recovery is pending and must not change any committed-path cache.
- `OnSettingsPropertyChanged` must not schedule a save while recovery is pending.
- Any already scheduled debounce must be cancelled or made harmless; it must not reach the store after the recovery gate is active.
- `CommitNowAsync` and equivalent direct ViewModel commit paths must reject while pending.
- A failed recovery must leave these gates closed for the remainder of the process lifetime.

### Controller — authoritative enforcement

`GuardianOrchestrator.CommitSettingsAsync` must reject a user-originated commit as soon as it has acquired the operation lock and detected `_recoveryPending`.

The rejection must happen **before** changing any of the following:

- `_committed`;
- `_committedTargetPath`;
- `_latestPin`;
- the configuration store;
- autostart state or registration.

Remove the current pending-recovery branch that persists ordinary user settings and permits target-path/pin changes. Return an unsuccessful commit result and log that the commit was rejected because recovery is pending.

The controller gate is authoritative. A disabled control is not a substitute for this check.

Do not weaken the recovery-owned persistence operation. The controller's internal recovery transaction must still be able to persist the safe disabled state while `_recoveryPending` is true; it must not route through the user-commit method that is now rejected.

### Tests

Add/update Application tests proving that, during pending recovery, a direct `CommitSettingsAsync` call:

- returns an unsuccessful result;
- does not call `SaveAsync`;
- does not call `SetEnabledAsync` or `RemoveRegistrationAsync`;
- does not alter the previously loaded target path or pin (verify by observing the subsequent controller-owned persistence/result path or a narrow internal test seam, not by relying on reflection).

Add/update WPF tests proving all relevant controls/gates are disabled while pending, Browse and path commit do nothing, no debounce reaches persistence, and the gates reopen only after successful recovery. Include a failed-recovery test where they remain disabled after `Error` is published.

## 4. Amendment C — bounded delay between automatic recovery attempts

**Supersedes:** no limit or retry-count requirement; this adds the missing timing requirement to v2.5 §2.4.

Keep `MaxRecoveryAttempts = 3` per startup. Add a **one-second cancellable delay** after a failed attempt when another attempt remains:

- no delay before attempt 1;
- one delay between attempts 1 and 2 if attempt 1 failed;
- one delay between attempts 2 and 3 if attempt 2 failed;
- no delay after success;
- no delay after the final failed attempt.

Use `Task.Delay` with cancellation and an injectable `TimeProvider` (or an equally deterministic delay seam) so tests do not sleep in real time. Cancellation must propagate; cancellation is not an exhausted recovery failure and must not clear the journal. The retry remains on the startup reconciliation path and must not block the WPF Dispatcher.

### Tests

- Verify exactly three maximum attempts.
- Verify exactly one delay between each pair of failed attempts, and no delay after success or the final attempt.
- Verify the delay is cancellable.
- Use fake/injected time; do not use wall-clock sleeps in automated tests.
- Verify that a failed recovery remains pending and is retried on the next startup.

## 5. Amendment D — establish the actual `schtasks.exe` output encoding

The current `BuildProcessStartInfo` redirects stdout and stderr without explicitly setting `StandardOutputEncoding` or `StandardErrorEncoding`. `LooksLikeMissingTask` contains a Russian marker, and a synthetic fixture tests that Unicode string, but neither proves the real bytes emitted by the target Windows installation are decoded correctly.

### Required investigation before selecting an encoding

- Capture the **raw bytes** of stdout and stderr from a read-only `schtasks.exe /Query /XML /TN "<exact application task name>"` invocation on the target Windows installation. Read `BaseStream` or use an equivalent byte-preserving capture; a previously mojibaked log line is not raw-byte evidence.
- Record the Windows version, display/system locale, active console output code page where applicable, system OEM code page, command line, exit code, and which stream contains the diagnostic.
- Compare candidate decodings against the raw bytes and the actual message displayed by the Windows command environment.
- Verify whether the process output encoding follows the console output code page, the system OEM code page, or another documented/empirically established behavior in the actual redirected-process scenario.
- Do not assume CP866, CP1251, UTF-8, `Encoding.Default`, or the current console setting without evidence. Do not hard-code a single locale-specific code page as a universal solution.
- Do not add mojibake variants to `LooksLikeMissingTask` as a workaround.
- Do not change process-wide console encoding to fix a child process's redirected streams.

### Implementation

Once the correct decoding strategy is established, set `ProcessStartInfo.StandardOutputEncoding` and `StandardErrorEncoding` before `Process.Start`, or use a byte-preserving read/decode strategy that implements the verified behavior. Keep stdout and stderr separate through capture, bounded logging, and classification.

The only path to `AutostartState.Absent` remains a positively classified supported task-not-found response. Every unrecognized or undecodable response remains `Unknown` and fail-closed. Do not treat exit code `1` alone as absence. Do not weaken the delete → query → verify flow or the task XML checks (`Command`, `Arguments`, `LogonTrigger`, `LogonType=InteractiveToken`, `RunLevel=HighestAvailable`).

### Tests and evidence

- Test that both stream encodings are configured/decoded using the selected strategy.
- Add raw-byte fixtures captured from a real Windows run and test stdout and stderr separately.
- Keep synthetic fixtures clearly labelled synthetic; they are useful unit tests but are not live Windows evidence.
- Test supported localized messages and an unsupported/unrecognized message that remains `Unknown`.
- Record the actual Windows capture used to select the encoding. If live evidence is not available, report the encoding decision as unverified and do not claim the manual failure is fixed.

A read-only query is sufficient for this investigation. Do not run `/Delete` merely to collect an encoding sample.

## 6. Amendment E — Config Refused must remain ERROR after Stop/Suspend

**Authority:** v2.5 §4.1 requires `Status=Error`, `IsDegraded=false` after Config Refused. The current `StopAsync` and `SuspendAsync` refusal branches violate that requirement by publishing `Stopped`.

When `_configRefused` is true:

- `StopAsync` must not save settings or touch autostart; it must leave the existing refused `Error` state intact and return the appropriate already-stopped result.
- `SuspendAsync` must not save settings or touch autostart; it must leave the existing refused `Error` state intact.
- Neither method may publish `Stopped` over the refused `Error`.
- Preserve the existing reason and `IsDegraded=false`.

Update the old test `ConfigRefused_StopAndSuspend_DoNotPersist_DoNotTouchAutostart_StoppedPublished` to assert that `Error` remains published after both calls. Rename it to express the new behavior. Keep assertions for no `SaveAsync` and no autostart mutation.

## 7. Amendment F — README and Toast verification

### README correction

The current code and v2.5 §3.3 permit exactly five Toast titles:

1. `Monitoring Started`
2. `Monitoring Stopped`
3. `Process Recovered`
4. `Recovery Failed`
5. `Target Changed`

Remove `Target Unreadable` as a Toast type from `README.md`. Also correct the target-integrity explanation: an unreadable target at initial Start remains a fail-closed Start rejection and keeps the previous pin, but **no `Target Unreadable` Toast is emitted** under the current five-title policy. Do not accidentally change the underlying Start/pin behavior while correcting the documentation.

Review the rest of the README sections touched by the five commits for references to removed A/B recovery, obsolete recovery guidance, and non-standard Toast titles. Update only directly related documentation. Do not edit `SPEC.md`.

### DPI verification

The existing `ToastCardLayoutTests` width measurement is useful automated evidence, but it does not satisfy the v2.5 requirement for a real visual WPF check.

Before Stage 1 acceptance, complete and record a real Windows visual check at:

- 100% display scaling;
- 125%;
- 150%;
- 200%.

At every scale, verify all five standard titles are fully visible without clipping or ellipsis and that the card remains 191×61 DIP. Record the Windows version, scale, title-by-title result, and screenshots or other reviewable evidence. Do not claim this requirement is satisfied by a CI markup test or a single-DPI `FormattedText` measurement.

## 8. Corrected Stage 1 acceptance gate

Stage 1 remains **not accepted** until all of the following are true:

- The five v2.5 corrective commits remain intact; no unrelated rewrite or history rewrite is performed.
- Amendment A: successful recovery ends in `Stopped`; exhausted recovery ends in `Error`, with `_recoveryPending=true`, journal pending, non-empty reason, and Start blocked.
- Amendment B: no user settings edit/commit, path/pin mutation, persistence, or autostart mutation is possible while recovery is pending.
- Amendment C: retries remain bounded to three and have deterministic cancellable delays between failures.
- Amendment D: the real `schtasks.exe` stdout/stderr decoding strategy is supported by raw-byte Windows evidence; unsupported output remains `Unknown`.
- Amendment E: Config Refused remains `Error` after Stop/Suspend, with no writes or scheduler changes.
- Amendment F: README no longer describes a `Target Unreadable` Toast, and real Toast verification at all four DPI scales is recorded.
- Existing v2.4 closure requirements `R-KE`, `R-CD`, `R-F`, and `R-DOCS` remain intact and their defect-detecting tests pass. Do not assume they passed merely because the newer five commits exist; report any requirement not re-verified.
- The following four commands pass with exit code 0, zero warnings, zero errors, and all tests passing:

```powershell
dotnet build .\ProcessGuardian.sln -c Release
dotnet build .\ProcessGuardian.sln -c Release-Elevated
dotnet test  .\ProcessGuardian.sln -c Release
dotnet test  .\ProcessGuardian.sln -c Release-Elevated
```

- Real elevated Windows checks in the original v2.5 §6.1–§6.3 are completed. A synthetic test or CI run does not replace the live Admin recovery check.
- `git status` and `git diff` are reviewed; only intended files are staged; each corrective work unit has its own focused commit.
- `SPEC.md`, `AGENTS.md`, `opencode.json`, `.gitignore`, and the plan are not staged or committed.
- No push is performed by the local agent.

Stage 2 remains locked until the entire corrected gate passes and the user explicitly authorizes proceeding.

## 9. Suggested corrective commit boundaries

Keep changes focused and use separate commits:

1. **`R-AS-ENC`** — establish output encoding from real Windows evidence, configure both redirected streams, and add decoding/fixture tests.
2. **`R-AS-GATE`** — controller and ViewModel pending-recovery settings gate; tests for no mutation or persistence.
3. **`R-AS-STATE`** — typed recovery outcome, `Error` after retry exhaustion, one-second cancellable inter-attempt delays, and recovery tests.
4. **`R-A-ERROR-STATE`** — preserve Config Refused `Error` through Stop/Suspend and update tests.
5. **`R-DOCS-TOAST`** — correct README's obsolete Toast documentation and verify remaining directly related wording.
6. **Stage 1 manual closure** — real Windows Task Scheduler output capture, elevated recovery acceptance, Toast DPI check, and final four-command build/test matrix.

If the actual Windows output contradicts the assumptions in Amendment D, stop that increment and report the raw evidence and proposed implementation before choosing an encoding. Do not weaken fail-closed behavior to force the recovery to pass.

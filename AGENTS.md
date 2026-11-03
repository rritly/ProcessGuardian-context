# AGENTS.md — ProcessGuardian

Operating rules for AI coding agents on this repository.

**`SPEC.md` is the current project specification and the primary source of truth for the implemented architecture and behavior.** Read the relevant section and enough context before making any change.

---

## 1. Source of Truth and Priority

Resolve requirement conflicts in this order:

1. Explicit user instruction in the current task.
2. `SPEC.md` — current project specification.
3. `AGENTS.md` — repository operating rules (this file).
4. Existing code and tests, when they clarify the actual current implementation.
5. Other historical documentation — reference only.

When `SPEC.md`, code, or tests appear inconsistent, do not invent new behavior. Identify the conflict and resolve it per the priority above.

---

## 2. Core Rules

### Language and Communication

- **Chat with the user exclusively in Russian.** This applies to replies, clarifying questions, progress updates, explanations, plans, and implementation reports posted in the chat. Keep technical identifiers, commands, paths, and quoted source text in their original form when accuracy requires it.
- **Use English for all project work and repository artifacts.** Write source code, identifiers, comments, XML documentation, test names, new or edited documentation, configuration text, examples, log messages, commit messages, and any reports saved to files in English.
- English is the default for new application UI text and other user-facing strings implemented in the repository, unless `SPEC.md` or an explicit instruction in the current task requires a different language.
- Do not translate or rewrite existing user-authored content or unrelated files just to enforce this language rule. Preserve their existing language unless the current task explicitly asks you to change it.
- If the user explicitly requests another language for a specific response or artifact, follow that instruction for the requested scope.

- **Read `SPEC.md` first.** Before implementing an Increment, read the relevant section (FR/NFR, exception policy, architecture, technology constraints) and the current behavior it describes.
- **Never modify `SPEC.md` or `opencode.json`** during ordinary implementation work (see §3).
- Respect the Clean Architecture dependency rule — `Domain → Application → Infrastructure → Presentation`. Dependencies point inward; no circular or unnecessary dependencies.
- Follow the technology stack and `[DO-NOT-USE]` constraints documented in `SPEC.md`.
- **Do not rewrite working code without necessity.** Prefer the smallest change that fully satisfies the task.
- **One Increment = one logical change.** It may touch multiple files when they belong to the same task (incl. production code + its tests). Never combine unrelated changes.
- Do not expand task scope on your own: no unrelated cleanup, refactoring, renaming, formatting, or package upgrades.
- Do not add new abstractions, files, NuGet packages, dependencies, or public APIs unless the task requires them.
- Do not change public interfaces or architectural contracts unless required by the task and consistent with `SPEC.md`.

---

## 3. Git Safety and Baseline

Inspect the repository state before modifying anything:

```powershell
git status --short
git diff -- . ":!SPEC.md" ":!opencode.json"
```

Treat all pre-existing changes as belonging to the user unless proven otherwise. Never overwrite, revert, stage, or commit pre-existing user changes.

**Protected files during ordinary Increment work** — modify only on explicit user instruction, and never commit them:

- `SPEC.md`
- `AGENTS.md`
- `opencode.json`
- `.gitignore`

Before committing, verify the final diff contains only files required by the current Increment.

---

## 4. Git Restrictions

Do not perform destructive or history-rewriting operations unless explicitly instructed by the user. Do not use:

- `git reset --hard`, `git clean`
- `git checkout -- <file>`, or `git restore <file>` when it could discard user changes
- `git rebase`, force operations, or any history rewriting

Do not `git push`, create/modify remotes, or publish the project unless explicitly instructed.

Never commit secrets, API keys, credentials, temporary files, build output, or generated artifacts that should stay untracked.

---

## 5. Increment Scope

For every Increment:

- define one clear objective;
- modify only the files required by that objective;
- preserve existing behavior outside the task scope;
- avoid unrelated refactoring; keep the change reviewable and atomic.

Implementation plus its tests are still one Increment when they belong to the same logical change. Do not artificially split a logically inseparable change just because multiple files are involved.

---

## 6. Build and Test Commands

After implementation, build then test — both must pass; a successful build alone is not sufficient:

```powershell
dotnet build .\ProcessGuardian.sln
dotnet test  .\ProcessGuardian.sln
```

The build must finish with exit code `0`, `0` errors, and `0` warnings (`TreatWarningsAsErrors=true` is set repo-wide via `Directory.Build.props`).

The solution has two build configurations:

- **`Release`** → regular edition (`ProcessGuardian.exe`).
- **`Release-Elevated`** → elevated admin edition (`ProcessGuardianAdmin.exe`). The admin edition exists **only** in this configuration; it is selected by the `ELEVATED_EDITION` define declared in `Directory.Build.props`, not in the `.csproj`. Always verify the admin configuration too:

```powershell
dotnet build .\ProcessGuardian.sln -c Release-Elevated
dotnet test  .\ProcessGuardian.sln -c Release-Elevated
```

CI (`.github/workflows/build.yml`) runs both configurations plus the publish script — a push that builds `Release` but fails `Release-Elevated` is a broken pipeline.

Test suite: 89 tests across three projects (Application, Infrastructure, Wpf). Behavioral changes and bug fixes should include or update regression tests whenever practical. Never delete a test merely because it fails, weaken an assertion to make it pass, or disable/ignore a failing test without explicit justification.

---

## 7. Architecture and Dependencies

Respect the existing project boundaries:

- `ProcessGuardian.Domain`
- `ProcessGuardian.Application`
- `ProcessGuardian.Infrastructure`
- `ProcessGuardian.Wpf`

Do not move logic between layers unless the task requires it. Infrastructure implements abstractions exposed by inner layers; Presentation depends on Application/Infrastructure per the existing structure.

Do not introduce: **WinUI 3, Windows App SDK, UWP runtime, MSIX, CsWinRT, WPF-UI, ModernWPF** — or any other technology explicitly forbidden by `SPEC.md`. Prefer packages already used by the project; do not add/upgrade packages without a concrete technical requirement, and avoid a dependency when a BCL/framework facility suffices.

---

## 8. Exceptions and Async/UI Rules

Follow the exception-handling policy in `SPEC.md`. Exceptions must not be silently swallowed — catch deliberately, log/report when required, preserve graceful degradation where the architecture requires it. Never use exception handling to hide a defect.

- Do not introduce synchronous blocking on the WPF UI thread.
- Do not blindly apply `Task.Run(...).GetAwaiter().GetResult()` as a general solution.
- Follow the asynchronous design already used by the application; when touching startup, shutdown, monitoring, dispatcher, or UI code, preserve UI responsiveness and existing cancellation behavior.

---

## 9. Current Behavior Takes Precedence Over Examples

Do not blindly copy code from documentation. If documentation contains an obsolete or known-buggy example, follow the current implementation rules and fix it rather than reproducing it. Do not reintroduce behavior that is not part of the current project requirements.

---

## 10. Completion Report and Manual Verification

After each Increment, report a concise `IMPLEMENTATION REPORT` covering: Increment name, Goal, Changed files, Tests added/updated, Acceptance Criteria (PASS/FAIL each), Build result, Test result, Git status/diff summary, protected-files status, commit hash/message, deviations/risks, and manual verification steps (or "none").

Do not claim success for checks that were not actually performed: never report build/test results, runtime behavior, or verification you did not observe.

Some behavior cannot be validated by automated tests alone — WPF visual layout, DPI, system tray interaction, shell integration, autostart, window/taskbar behavior, and runtime lifecycle. When manual verification is required, state exactly why, what the user must do, and what result is expected; do not claim it passed unless the user performed and reported it.

---

## 11. Stop Conditions

Do not continue to the next logical Increment when:

- build fails;
- tests fail;
- Acceptance Criteria are not satisfied;
- unrelated changes are present;
- protected files were modified;
- the change introduces an unexplained architecture change;
- the result differs materially from `SPEC.md`.

Fix the current Increment first, or stop and report the blocking issue.

---

## 12. Commit Requirements

Each completed Increment must produce **one separate atomic commit** containing only that Increment.

Before committing:

1. Run the build (both configurations).
2. Run the tests (both configurations).
3. Inspect `git status` — confirm no protected or unrelated files are staged (watch for a broad `git add .`).
4. Inspect `git diff`.
5. Confirm no secrets or temporary files are included.

Use a clear commit message describing the actual change. Never commit `SPEC.md`, `AGENTS.md`, or `opencode.json`. Do not commit a partially implemented Increment unless explicitly instructed.

---

## 13. Final Principle

Prefer **minimal change + explicit tests + clean architecture + verifiable behavior + atomic commit** over large refactoring, speculative improvements, and undocumented behavior.

# ProcessGuardian — Technical Specification (SPEC_v2)

> **Purpose.** This document is the current project specification and reflects the **actual current state** of the repository as of 2026-10-26. It records what the codebase actually does today, so it can be used as an accurate reference for future work.

> **Protected files.** `SPEC.md`, `AGENTS.md`, and `opencode.json` are **protected**: commits touching any of them are forbidden during ordinary Increment work. They are edited only on explicit user instruction (and are never committed). See §12.

**Version:** v2.1 (working)
**Date:** 2026-10-26
**Status:** Implemented / Maintained
**Target:** .NET 9, WPF, Windows 10 1809+ / Windows 11
**Output:** Single-window desktop utility with system-tray background operation.

---

## 1. Executive Summary

ProcessGuardian is a lightweight Windows background utility that monitors a user-specified executable and restarts it if it crashes or disappears:

1. **WPF on .NET 9**.
2. **Clean Architecture** (Domain → Application → Infrastructure → Presentation).
3. **Microsoft.Extensions.DependencyInjection** + **CommunityToolkit.Mvvm**.
4. Self-contained publish profiles (regular and admin editions; the framework-dependent profile was removed).
5. Dark-themed main window (content-sized) and a custom dark system-tray menu.
6. Notifications are delivered as **in-app corner overlay toast cards** (not Windows toast notifications).

---

## 2. Functional Requirements (FR) — Current Behavior

### 2.1. Core Monitoring Engine

| ID | Requirement | Notes |
|----|-------------|-------|
| FR-01 | User provides an **absolute path** to a `.exe` file. | Editable `TextBox` in the UI; browse via `OpenFileDialog`. |
| FR-02 | **Arguments are no longer supported.** `AppSettings.TargetProcessArguments` and `ProcessTarget.Arguments` were removed. `ProcessStartInfo` is built from the path only. | **Changed from SPEC.md.** |
| FR-03 | Detection: enumerate processes by **name** (without `.exe`), then match by **full executable path** (`Process.MainModule.FileName`), case-insensitive. Only an exact path match counts as "running". | Implemented in `WindowsProcessMonitor.CheckAsync`. |
| FR-04 | If target is not found, start via `Process.Start` with `UseShellExecute=false`, `CreateNoWindow=true`. | `StartAsync` returns `false` on failure (never throws). |
| FR-05 | A same-name process at a **different path** is ignored (treated as "not running"). | |
| FR-06 | `Win32Exception`/`InvalidOperationException` on a single candidate is caught, logged, and enumeration continues. | `WindowsProcessMonitor` wraps each candidate in `using (p)`. |
| FR-07 | Configurable check interval: **1–3600 s**, default **5**. | |
| FR-08 | Configurable initial delay before first check: **0–3600 s**, default **0**. | |
| FR-09 | Configurable max restart attempts: **0–100**, where **0 = infinite**. Default **5**. | Budget is reset per monitoring session. |
| FR-10 | Configurable delay between restart attempts: **1–3600 s**, default **10**. | |
| FR-11 | Startup verification: after `Process.Start`, poll for target appearance up to N s (default **5**). | `VerifyStartupAsync` polls every 250 ms. |
| FR-12 | **Failure cooldown is removed.** Exhausting max attempts sets status to `Error`, shows a "Recovery Failed" card, sets `IsMonitoringEnabled=false`, and persists to disk. There is no cooldown state. | **Changed from SPEC.md (removed FR-12 cooldown behavior).** |
| FR-13 | Missing target `.exe` on disk: logged, status `Error`, no crash. | |
| FR-14 | **Free initial launch.** The first `Start` of a session launches the target without consuming any restart attempt and without flashing the `Restarting` status. Only granted once per session. | **New, not in SPEC.md.** |
| FR-15 | Empty target path while monitoring enabled: logged, status `Error`, monitoring continues polling. | |
| FR-16 | **Target path validation** via `ProcessTarget.TryCreate`: the path must be **absolute**, end with **`.exe`** (case-insensitive), resolve to a **local drive** (UNC `\\server\share` and device `\\?\` paths rejected), and the **file must exist**. A failed check is logged, sets status `Error`, and **no launch is attempted**; monitoring keeps polling. Covers the empty-path case of FR-15. | **New.** |
| FR-17 | **Target hash pinning.** Every Start re-records the SHA-256 (uppercase hex) of the target binary into `settings.json` (`TargetHashSha256`). Before **every** launch (free initial launch and budgeted restarts) the on-disk hash is re-verified; a mismatch **or an unreadable file** blocks the launch, sets status `Error`, shows a red **"Target Changed"** card, disables monitoring and persists it (no retry against a changed target). An empty pin (settings written before this feature) skips the check for backwards compatibility. Re-enabling monitoring is the explicit re-confirmation. | **New.** |

> **Validation note.** The numeric ranges (e.g. 1–3600, 0–100) and defaults originate from the original `SPEC.md`. In the current code every numeric setting is programmatically clamped to **[lower bound, 999]** in the `AppSettings` generated setters (applies to UI bindings and JSON deserialization); the XAML mini boxes additionally use `MaxLength=3` (max 999) and `ValidatesOnExceptions=True` (rejects non-numeric input with a validation error). The target path is validated by `ProcessTarget.TryCreate` (FR-16) before any launch.

### 2.2. Main Window UI

| ID | Requirement | Notes |
|----|-------------|-------|
| UI-01 | Single main window. Title "ProcessGuardian" (edition name from `AppEdition.WindowTitle`, §8.4). **Content-sized** (`SizeToContent=WidthAndHeight`, `ResizeMode=CanMinimize`); dark theme `#1A252F`. | **Changed from SPEC.md (was 640×520, min 520×400); changed from the earlier v2 claim of fixed 764×706 — the window actually sizes to content.** |
| UI-02 | Card 1 — **Settings**: editable Target path + Browse ("+") button; mini numeric boxes for Check interval (s), Verify timeout (s), Max restart attempts, Initial delay (s), Restart delay (s); Autostart with Windows checkbox. | **Arguments and Enable-logging fields removed.** |
| UI-03 | Card 2 — **Monitoring**: "Start monitoring" / "Disable" toggle buttons (visibility bound to `Status`); "toast-notifications" checkbox bound to `EnableToasts`; status pill (text via `StatusToDisplayConverter`, color via XAML data triggers). | |
| UI-04 | Card 3 — **Log**: read-only log area with **Open Logs** (`OpenBook.png`) and **Clear Logs** (`Trash.png`) icon buttons; custom scrollbar that hides entirely (arrows, track and thumb) via `ComputedVerticalScrollBarVisibility` while the log fits the viewport and reappears when it overflows; auto-scroll stick-to-bottom. | **Changed from SPEC.md (scrollbar was always visible).** |
| UI-05 | All settings changes auto-save with **500 ms debounce** (cancels previous pending save). Start/Stop flush immediately. | |
| UI-06 | Browse opens `OpenFileDialog` filtered to `.exe` only; updates `TargetProcessPath`. | |
| UI-07 | **Close (×) hides window to tray** and removes it from the taskbar; application continues running. | `OnMainWindowClosing` cancels and hides. |
| UI-08 | Maximize is disabled (system-menu item grayed; `SC_MAXIMIZE` swallowed in `WndProc`). DWM dark caption chrome applied. | **New.** |

### 2.3. System Tray Behavior

| ID | Requirement | Notes |
|----|-------------|-------|
| TR-01 | Custom dark **`TrayMenuPopup`** (borderless window, DPI-aware, clamped to monitor work area) shown on right-click (`TrayRightMouseUp`). Items: **Show {AppEdition.Name}**, **Start monitoring / Disable** (toggle label), **Exit {AppEdition.Name}**. **Width grows to fit the widest label** (text inset + measured text + right padding + border; never narrower than the designed 158 DIP ≈ 210 physical px); height stays fixed (118 DIP). | **Changed from SPEC.md (was a standard `ContextMenu`); width made dynamic for long edition names.** |
| TR-02 | Double-click tray icon shows/restores the main window. | |
| TR-03 | Tray icon reflects state via distinct icons: `Monitoring`/`Waiting`/`Restarting` → On, `Error` → Error, else Off. | |
| TR-04 | Tooltip: `"{AppEdition.Name} - {Status}"` (§8.4). | |
| TR-05 | **Exit** from tray performs full graceful shutdown: stops monitoring, persists to disk, removes HKCU autostart key, then shuts down. | |

### 2.4. Notifications (Overlay Cards)

| ID | Requirement | Notes |
|----|-------------|-------|
| NO-01 | **No Windows toast notifications.** `Microsoft.Toolkit.Uwp.Notifications` and `WindowsToastNotificationService` were removed from the codebase. | **Changed from SPEC.md.** |
| NO-02 | `INotificationService` is implemented by `OverlayNotificationService` (in `ProcessGuardian.Wpf.Notifications`), which shows corner overlay **toast cards** in the bottom-right of the primary monitor. | |
| NO-03 | `ToastHost` manages an `OverlayWindow`; max 4 stacked cards — overflow evicts the oldest card **synchronously** (deferred dismissal would deadlock the dispatcher); each card auto-hides after 5 s (with fade entrance/exit), dismissible on click; cards are authored at **191×61 DIP** (≈75% of the legacy 254×81 physical px) and scale with monitor DPI. | |
| NO-04 | Toast titles (defined in code): **"Monitoring Started"** / **"Monitoring Stopped"** (from `MainViewModel`), **"Process Recovered"** (green, from orchestrator on verified recovery), **"Recovery Failed"** (red, from orchestrator on exhaustion), **"Target Changed"** (red, from orchestrator when a pinned target hash does not match). All other titles render white. `ToastCard.ResolveBrush` colors "Process Recovered" (`#4CC86A`), "Recovery Failed" and "Target Changed" (`#FC3847`); everything else is `#FFFFFF`. Cards are shown only when `EnableToasts` is true. | |

### 2.5. Autostart

| ID | Requirement | Notes |
|----|-------------|-------|
| AS-01 | Toggle writes to `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`. Value name `"ProcessGuardian"` (edition: `AppEdition.AutostartValueName`), data `"{exePath}" --background`. Regular edition only. | |
| AS-02 | Toggle OFF removes the value. | |
| AS-03 | **`settings.json` `AutostartEnabled` is the source of truth.** On launch, if true, the HKCU key is (re)written so it is present after a previous Exit. | **Changed from SPEC.md (registry was source of truth).** |
| AS-04 | No admin rights required (HKCU only). | Regular edition. |
| AS-05 | On user **Exit**, the HKCU key is removed while the JSON `AutostartEnabled` flag is left unchanged (`_suppressAutostartSync` prevents re-adding it). | **New.** |
| AS-06 | **Admin edition autostart:** `ScheduledTaskAutostartService` registers a scheduled task named `AppEdition.ScheduledTaskName` via `schtasks.exe` — `/Create /TN "{name}" /TR "\"{exePath}\" --background" /SC ONLOGON /RL HIGHEST /F`; OFF → `/Delete /TN "{name}" /F`; `IsEnabledAsync` → `/Query` exit code 0. The task starts the app elevated **without a UAC prompt** at logon (HKCU Run cannot auto-elevate). | **New.** |

### 2.6. Single Instance

| ID | Requirement | Notes |
|----|-------------|-------|
| SI-01 | Only one process instance allowed. | Local named `Mutex` `"{AppEdition.Name}_SingleInstance"` (no `Global\`). |
| SI-02 | Second launch signals the first instance via a local named `EventWaitHandle` `"{AppEdition.Name}_ShowRequested"` and activates its window if visible; second process exits. | **Changed from SPEC.md: added event + watcher thread.** |
| SI-03 | `SingleInstanceManager` exposes `ShowRequested` and runs a background watcher thread (`StartWatching`) to restore the window when a second launch occurs. | |

### 2.7. Power Events

Not implemented (`[MAY]` — deferred). `Microsoft.Win32.SystemEvents.PowerModeChanged` is not used. Monitoring naturally pauses while the OS is asleep.

### 2.8. Logging

| ID | Requirement | Notes |
|----|-------------|-------|
| LG-01 | `InMemoryLogSink` (Serilog `ILogEventSink`): ring buffer capped at **600** entries, **seeded on construction with the last ≤600 lines of `logs.log`** (best-effort); `GetRecentLogs(200)` returns the last 200 lines for the UI. | **Changed from SPEC.md (was 100/20; seeding added so the log card shows file history on startup).** |
| LG-02 | `Emit` only enqueues and marks the buffer dirty; a background timer **appends the pending entries to `%LocalAppData%\ProcessGuardian\logs.log`** at most once every **500 ms** (append-only — file history survives restarts); when the file reaches **512 KB** it rotates to a single archive **`logs.1.log`** (previous archive deleted, ~1 MB total bound); `Flush()` persists pending entries on demand (used on shutdown). Exposes `LogFilePath`, `Clear()`, `Flush()`. | **Changed from SPEC.md (was: synchronous full-file write on every `Emit`); replaces the rewrite-on-flush behavior that wiped file history on restart.** |
| LG-03 | Line format: `yyyy-MM-dd HH:mm:ss LEVEL  message` (no brackets; levels VRB/DBG/INFO/WARN/ERR/FATAL). | **Changed from SPEC.md.** |
| LG-04 | Logger abstraction via `Microsoft.Extensions.Logging.ILogger<T>`; Serilog configured in `Bootstrapper` writing only to the in-memory sink (`Microsoft.Hosting.Lifetime` overridden to Warning). | |

---

## 3. Non-Functional Requirements (NFR) — Current Status

| ID | Requirement | Target | Current Status |
|----|-------------|--------|----------------|
| NFR-01 | RAM idle (monitoring on, window hidden) | ≤ 35 MB | Not re-measured. |
| NFR-02 | RAM with window open | ≤ 55 MB | Not re-measured. |
| NFR-03 | CPU idle | ≤ 0.1% (1 core) | Not re-measured. |
| NFR-04 | Cold start to ready | ≤ 1.5 s (SSD) | Not re-measured. |
| NFR-05 | Framework-dependent distributable size | ≤ 8 MB | **Profile removed** — only self-contained builds are published. |
| NFR-06 | Self-contained distributable size (x64) | ≤ 55 MB | **Not met — ≈ 78 MB** per edition (per README). |
| NFR-07 | No external runtime deps (framework-dep) | .NET 9 Desktop Runtime only | OK. |
| NFR-08 | Graceful shutdown on any signal | No orphaned processes, mutex released, config flushed | Implemented (`OnExit`). |
| NFR-09 | Offline-only operation | Zero network activity | OK. |
| NFR-10 | User-level permissions only | HKCU, %LocalAppData% | OK. |

> **Size gap reason:** the `net9.0-windows10.0.17763.0` TFM pulls in the Windows SDK bindings, and WPF does not trim well in single-file mode (documented in README).

---

## 4. Exception Handling Policy

The same principles from `SPEC.md` §4 apply and remain implemented:

- **EH-01** Never swallow exceptions silently — every catch logs or reports.
- **EH-02** `GuardianOrchestrator.ExecuteAsync` catches all exceptions, sets `Error`, and continues after a delay; `RunMonitoringCycleAsync` also has internal try-catch around `CheckAsync`/`StartAsync`/`VerifyStartupAsync`/`ShowAsync`.
- **EH-03** `App.OnStartup` wraps startup in a global try-catch showing a fatal `MessageBox`.
- **EH-04** File/registry/icon failures degrade gracefully (defaults, fallback icons).
- **EH-05** Notification failures are logged; monitoring continues.
- **EH-06** `GuardianStatus` transitions to `Error` on unrecoverable failures.
- **EH-07** Disposables (`TaskbarIcon`, `Mutex`, `EventWaitHandle`, icons, menu) are disposed in `Dispose`/`OnExit`, each guarded.
- Global handlers in `App.xaml.cs`: `UnhandledException`, `DispatcherUnhandledException`, `UnobservedTaskException` — write to `%LocalAppData%\ProcessGuardian\crash.log` (or `shutdown-error.log`) via `CrashLog.Append` (size-capped, rollover at 512 KB) and are best-effort. **Changed from SPEC.md (was: CWD-relative `crash.log`).**

---

## 5. Technology Stack & Architecture

### 5.1. Stack Matrix (Current)

| Layer | Technology | Version |
|-------|-----------|---------|
| UI Framework | WPF (.NET 9) | `net9.0-windows10.0.17763.0` |
| DI / Hosting | `Microsoft.Extensions.Hosting` / DI | 9.0.20 |
| MVVM | `CommunityToolkit.Mvvm` (source generators) | 8.4.2 |
| Logging | Serilog (`Serilog.Extensions.Logging`) + in-memory sink | Serilog 4.4.0 / Serilog.Extensions.Logging 9.0.2 |
| Tray Icon | `Hardcodet.NotifyIcon.Wpf` | 2.0.1 |
| Notifications | **Custom WPF overlay cards** (`ToastHost`, `OverlayWindow`, `ToastCard`) | n/a (no `Microsoft.Toolkit.Uwp.Notifications`) |
| Process API | `System.Diagnostics.Process` (BCL) | — |
| Tests | xUnit + FluentAssertions + NSubstitute | xUnit 2.9.3 / FA 6.12.0 / NS 5.1.0 |

**[DO-NOT-USE]** WinUI 3, Windows App SDK, MSIX, UWP runtime, CsWinRT, `WPF-UI`/`ModernWPF`. `Microsoft.Toolkit.Uwp.Notifications` is no longer used.

### 5.2. Clean Architecture Layers

```
+---------------------------------------------+
|  Presentation (ProcessGuardian.Wpf)         |
|  Views, ViewModels, Converters,             |
|  Notifications (overlay cards), Bootstrapper|
|  References: Application, Infrastructure    |
+---------------------------------------------+
|  Application (ProcessGuardian.Application)  |
|  Services: GuardianOrchestrator             |
|  References: Domain only                    |
+---------------------------------------------+
|  Domain (ProcessGuardian.Domain)            |
|  Entities, Enums, ValueObjects, Interfaces  |
|  References: CommunityToolkit.Mvvm          |
+---------------------------------------------+
|  Infrastructure (ProcessGuardian.Infrastructure) |
|  ProcessMonitoring, Configuration, Autostart,    |
|  Native, Tray (TrayIconService + TrayMenuPopup), |
|  Status, Logging                                 |
|  References: Domain, Application             |
+---------------------------------------------+
```

**Dependency rule:** all dependencies point inward; Infrastructure implements interfaces defined in Domain/Application. No circular references.

### 5.3. Actual Project Structure

```
ProcessGuardian.sln
+-- global.json
+-- Directory.Build.props
+-- build/
|   +-- publish.ps1
+-- .github/workflows/build.yml
+-- src/
|   +-- ProcessGuardian.Domain/
|   |   +-- AppEdition.cs
|   |   +-- Entities/AppSettings.cs
|   |   +-- Enums/GuardianStatus.cs
|   |   +-- Interfaces/{IProcessMonitor,IConfigurationStore,INotificationService,
|   |   |            IAutostartService,ITrayService,IStatusReporter,
|   |   |            ITargetIntegrityService}.cs
|   |   +-- ValueObjects/{ProcessTarget,ProcessCheckResult}.cs
|   +-- ProcessGuardian.Application/
|   |   +-- Services/GuardianOrchestrator.cs
|   +-- ProcessGuardian.Infrastructure/
|   |   +-- ProcessMonitoring/{WindowsProcessMonitor.cs, Sha256IntegrityService.cs}
|   |   +-- Configuration/JsonConfigurationStore.cs
|   |   +-- Autostart/{RegistryAutostartService.cs, ScheduledTaskAutostartService.cs}
|   |   +-- Native/SingleInstanceManager.cs
|   |   +-- Status/StatusReporter.cs
|   |   +-- Logging/{InMemoryLogSink.cs, CrashLog.cs}
|   |   +-- Tray/{TrayIconService.cs, TrayMenuPopup.cs}
|   +-- ProcessGuardian.Wpf/
|       +-- App.xaml / App.xaml.cs
|       +-- app.admin.manifest
|       +-- Bootstrapper.cs
|       +-- Views/MainWindow.xaml / MainWindow.xaml.cs
|       +-- ViewModels/MainViewModel.cs
|       +-- Converters/StatusToDisplayConverter.cs
|       +-- Notifications/{OverlayNotificationService,OverlayWindow,ToastCard,ToastHost}
|       +-- Assets/{AppIcon,TrayIconOn,TrayIconOff,TrayIconError}.ico
+-- tests/
|   +-- ProcessGuardian.Application.Tests/{GuardianOrchestrator,AppSettingsClamp,ProcessTargetValidation}Tests.cs
|   +-- ProcessGuardian.Infrastructure.Tests/{JsonConfigurationStore,RegistryAutostartService,
|          TrayIconService,WindowsProcessMonitor,InMemoryLogSink,InMemoryLogSinkHistory,
|          InMemoryLogSinkFlush,Sha256IntegrityService,AdminConfigDirectoryAcl,AdminConfigFileAcl,
|          ScheduledTaskAutostartService,CrashLog,StatusReporter}Tests.cs
|   +-- ProcessGuardian.Wpf.Tests/{MainViewModelReentrancy,ToastEviction}Tests.cs
+-- README.md
+-- SPEC.md   (original/historical spec)
```

> **Structural differences vs SPEC.md:** no `docs/SPEC.md` (SPEC lives at repo root); `WindowsToastNotificationService.cs` removed; added `Tray/TrayMenuPopup.cs`, `Wpf/Notifications/*`, and `Converters/StatusToDisplayConverter.cs`.

### 5.4. Shared Build Configuration

`Directory.Build.props` (repository root):

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net9.0-windows10.0.17763.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>latest</LangVersion>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
  <!-- Admin edition build (ProcessGuardianAdmin): the define selects AppEdition
       values in every layer; optimization matches a regular Release build. -->
  <PropertyGroup Condition="'$(Configuration)' == 'Release-Elevated'">
    <DefineConstants>$(DefineConstants);ELEVATED_EDITION</DefineConstants>
    <Optimize>true</Optimize>
  </PropertyGroup>
</Project>
```

MSBuild auto-imports this file from the repository root for every project (src and tests); the `.csproj` files contain no manual `<Import>` entries. For `Release-Elevated` the conditional group above sets the `ELEVATED_EDITION` define for **all** projects; the WPF project additionally switches its `AssemblyName` (`ProcessGuardianAdmin`) and the application manifest in its own `.csproj` (§8.4).

`global.json` (repository root):

```json
{
  "sdk": {
    "version": "9.0.100",
    "rollForward": "latestFeature"
  }
}
```

### 5.5. Package Dependencies (per project)

- **Domain:** `CommunityToolkit.Mvvm` 8.4.2.
- **Application:** `Microsoft.Extensions.Hosting.Abstractions` 9.0.20, `Microsoft.Extensions.Logging.Abstractions` 9.0.20.
- **Infrastructure:** `Hardcodet.NotifyIcon.Wpf` 2.0.1, `Serilog` 4.4.0. **No `Serilog.Sinks.File`** (file output is handled by `InMemoryLogSink`).
- **Wpf:** `CommunityToolkit.Mvvm` 8.4.2, `Microsoft.Extensions.Hosting` 9.0.20, `Serilog.Extensions.Logging` 9.0.2.
- **Tests:** `Microsoft.NET.Test.Sdk` 17.11.1, `xunit` 2.9.3, `xunit.runner.visualstudio` 2.8.2, `FluentAssertions` 6.12.0, `NSubstitute` 5.1.0.

---

## 6. Icon & Asset Semantics

| File | Sizes | Purpose |
|------|-------|---------|
| `AppIcon.ico` | multi-size | Brand; compiled into the exe (`<ApplicationIcon>`) and embedded as a WPF `<Resource>` (`pack://application:,,,/ProcessGuardian;component/Assets/AppIcon.ico`) for overlay-card icons. |
| `TrayIconOn.ico` | 16/32 (green accent) | Tray — Monitoring/Waiting/Restarting. Embedded resource of `ProcessGuardian.Infrastructure`. |
| `TrayIconOff.ico` | 16/32 (grayscale) | Tray — Stopped. Embedded resource in `ProcessGuardian.Infrastructure`. |
| `TrayIconError.ico` | 16/32 (red accent) | Tray — Error. Embedded resource in `ProcessGuardian.Infrastructure`. |

- All icons are embedded into their owning assemblies; no icon files are copied to the publish output.
- Tray icons are embedded into `ProcessGuardian.Infrastructure` via `<EmbeddedResource>` with explicit `LogicalName` (`ProcessGuardian.Assets.TrayIcon*.ico`), referenced from the same source files in `Wpf/Assets` through MSBuild `Link`.
- `TrayIconService` loads tray icons via `GetManifestResourceStream` and falls back to `SystemIcons.Application`, then `SystemIcons.WinLogo`.
- Log-window button icons are inline vector `<Geometry>` resources in `MainWindow.xaml`, not files.

---

## 7. Core Module Specifications (Actual)

> These describe the **real** implementations. They differ from the code examples in `SPEC.md`.

### 7.1. Domain Layer

#### 7.1.1. `AppSettings` (ObservableEntity)
`ProcessGuardian.Domain/Entities/AppSettings.cs` — `ObservableObject` with `[ObservableProperty]` fields:

| Property | Default |
|----------|---------|
| `TargetProcessPath` | `""` |
| `CheckIntervalSeconds` | `5` |
| `InitialDelaySeconds` | `0` |
| `MaxRestartAttempts` | `5` (0 = infinite) |
| `RestartDelaySeconds` | `10` |
| `StartupVerificationTimeoutSeconds` | `5` |
| `AutostartEnabled` | `false` |
| `IsMonitoringEnabled` | `false` (master switch) |
| `EnableToasts` | `true` |

**Clamps:** numeric settings are clamped to `[min, 999]` in the generated setters (applies to UI bindings and JSON deserialization): `1 <= CheckIntervalSeconds <= 999`, `1 <= StartupVerificationTimeoutSeconds <= 999`, `0 <= InitialDelaySeconds <= 999`, `0 <= MaxRestartAttempts <= 999`, `0 <= RestartDelaySeconds <= 999`. The common upper bound is `AppSettings.MaxNumericValue` (999), chosen to fit the 3-character mini boxes.

**Removed vs SPEC.md:** `TargetProcessArguments`, `EnableLogging`, `FailureCooldownSeconds`. **Added:** `EnableToasts`.

#### 7.1.2. `GuardianStatus` (enum)
`Stopped, Waiting, Monitoring, Restarting, Error`. Note: the orchestrator never enters a cooldown (cooldown behavior removed); the former `Cooldown` value was removed from the enum.

#### 7.1.3. `ProcessTarget` (record)
```csharp
public sealed record ProcessTarget(
    string FullPath,
    string NameWithoutExtension,
    string WorkingDirectory
);
```
**Removed vs SPEC.md:** `Arguments`. **Added:** static `ProcessTarget.TryCreate(string, out ProcessTarget?, out string?)` — the single validation gate: rejects empty/whitespace, non-absolute paths, non-`.exe` extensions (case-insensitive), UNC/device paths (root must be a local `X:\` drive), and missing files; returns a fully derived target (name without extension, working directory) on success.

#### 7.1.4. `ProcessCheckResult` (record)
```csharp
public sealed record ProcessCheckResult(ProcessOutcome Outcome, int? ProcessId = null);
public enum ProcessOutcome { Running, NotRunning, Unknown }
```
`Unknown` means the enumeration itself failed (the process state could not be determined); the orchestrator skips the cycle on it — no restart (which could duplicate a still-running target) and no status flip.

#### 7.1.5. Domain Interfaces
`IProcessMonitor`, `IConfigurationStore`, `INotificationService`, `IAutostartService`, `ITrayService` (incl. `Initialize`, `SetStatus`, events `ShowRequested/StartRequested/StopRequested/ExitRequested`), `IStatusReporter` — as in `SPEC.md` **except** `IStatusReporter` now exposes `GuardianStatus Current { get; }` in addition to `Report` and `StatusChanged`. **Added:** `ITargetIntegrityService` — `Task<string?> ComputeHashAsync(string filePath, CancellationToken ct = default)`: SHA-256 of the file as uppercase hex, or `null` when the file cannot be read; never throws.

### 7.2. Application Layer

#### 7.2.1. `GuardianOrchestrator` (BackgroundService)
`ProcessGuardian.Application/Services/GuardianOrchestrator.cs`. Behaviors:

- **Session lifecycle:** entering `RunMonitoringCycleAsync` resets `_restartAttempts = 0` and sets `_initialLaunchPending = true`. The whole outer loop runs under a catch-all: an unexpected exception from the config store or any await degrades to `Error` + a 5 s cooldown wait instead of stopping the host.
- **Free initial launch:** if the target is missing on the first check of a session, `HandleMissingAsync` performs a launch via `StartAndVerifyAsync(..., showRecoveryToast: false)` without consuming an attempt and without setting `Restarting`.
- **Restart budget:** on subsequent misses, `_restartAttempts` is incremented and the status is set to `Restarting` while the budgeted attempt is in flight (the free initial launch never flashes it); if `MaxRestartAttempts > 0 && _restartAttempts >= MaxRestartAttempts`, status → `Error`, a "Recovery Failed" card is shown (if `EnableToasts`), `IsMonitoringEnabled` is set to `false` and persisted — **no cooldown**.
- **Restart backoff:** after a failed launch (`StartOutcome.NotStarted`), or a failed startup verification (`VerifyFailed`) while the budget has remaining attempts, the orchestrator waits `RestartDelaySeconds` before the next attempt (previously `NotStarted` bypassed the delay entirely and re-attempted every `CheckIntervalSeconds`). Exhaustion adds no extra delay. Cancellation (`OperationCanceledException`) is rethrown, never logged as a generic failure or counted as a failed attempt.
- **Wait loop:** `WaitForNextCheckAsync` sleeps in 250 ms slices and aborts the cycle if monitoring is disabled mid-wait (responsive Stop). The wait is capped at `int.MaxValue` ms (guard against int-cast overflow of a huge `CheckIntervalSeconds`). The initial delay uses the same sliced wait.
- **Invalid target path:** `ProcessTarget.TryCreate` rejects empty/whitespace, non-absolute, non-`.exe`, UNC/device, or missing-file paths → logged, status → `Error`, monitoring keeps polling.
- **Unknown check result:** an `Unknown` outcome (enumeration failure) skips the cycle — no restart, no status change, monitoring keeps polling.
- **Target-hash pin verification (FR-17):** `StartAndVerifyAsync` calls `VerifyTargetHashAsync` **before** any launch; mismatch/unreadable hash → launch blocked, status → `Error`, red "Target Changed" card (if `EnableToasts`), `IsMonitoringEnabled` set to `false` and persisted; returns a distinct `StartOutcome.Blocked` so `HandleMissingAsync` performs **no retry** and consumes no restart attempt. An empty pin skips the check (legacy settings); a hash read failure is fail-closed.
- Toast titles: "Process Recovered" on verified recovery, "Recovery Failed" on exhaustion, "Target Changed" on pin mismatch.

### 7.3. Infrastructure Layer

#### 7.3.1. `WindowsProcessMonitor`
Enumerates by name, iterates candidates inside `using (p)`, catches `Win32Exception`/`InvalidOperationException` per candidate. `StartAsync` builds `ProcessStartInfo(target.FullPath)` (no arguments) with `UseShellExecute=false`, `CreateNoWindow=true`, and disposes the returned `Process` handle immediately. `VerifyStartupAsync` polls every 250 ms and **rethrows `OperationCanceledException`**.

#### 7.3.2. `JsonConfigurationStore`
- **Caches** the loaded `AppSettings` in `_current`; `LoadAsync` returns the cached instance after first load.
- `JsonSerializerOptions`: `WriteIndented = true`, `PropertyNameCaseInsensitive = true`.
- `SaveAsync` writes to `_path + ".tmp"` then `File.Move(tmp, path, overwrite: true)` for **atomic** writes; deletes tmp on failure/cancellation; `_current` is updated **only after** the file has been replaced successfully. IO/unauthorized write failures are rethrown after cleanup so callers can report them.
- Constructor creates the data directory (fatal if it cannot): the regular edition uses `%LocalAppData%\ProcessGuardian`; the admin edition uses `%ProgramData%\ProcessGuardianAdmin`, creates it **born-hardened** in one atomic step (`DirectoryInfo.Create(hardenedSecurity)`) and re-hardens pre-existing directories and the `settings.json` file (at construction and after every save) to Administrators/SYSTEM-only (§8.5).

#### 7.3.3. `RegistryAutostartService`
HKCU `...\Run` value `AppEdition.AutostartValueName` = `"{exePath}" --background`; the value is written only when the current executable path is available. Broad exception handling (`SecurityException`, `UnauthorizedAccessException`, generic). An internal registry-path seam lets tests run against a scratch key.

#### 7.3.4. `SingleInstanceManager`
Local `Mutex` `"ProcessGuardian_SingleInstance"`; local named `EventWaitHandle` `"ProcessGuardian_ShowRequested"`; `StartWatching()` runs a background thread that raises `ShowRequested` when the event is set (the watcher checks the disposed flag before invoking, so a `Dispose` wake-up never fires the event during shutdown); `ActivateFirstInstance()` only restores the window if it is visible (`IsWindowVisible`).

#### 7.3.5. `TrayIconService`
Uses `Hardcodet.Wpf.TaskbarNotification`. Opens the custom `TrayMenuPopup` on `TrayRightMouseUp`; `TrayMouseDoubleClick` → Show. `ApplyStatus` sets `_isMonitoring` and the menu's second-item label ("Start monitoring" / "Disable"). Icon mapping: On = Monitoring/Waiting/Restarting, Error = Error, else Off.

#### 7.3.6. `TrayMenuPopup` (new)
Borderless transparent `Window` acting as the dark tray menu, with 3 rows (Show / Start-or-Disable / Exit), DPI-scaled layout, closes on deactivate/Escape. Height fixed at 118 DIP (≈155 physical px @133%); **width is computed in the constructor** from the widest of the three labels (`TextBlock.Measure`, same properties as runtime) as `text inset (14.25 DIP) + max label width + right padding (14.25 DIP) + 2 DIP borders`, clamped to a minimum of 158 DIP; hover/hit rows and the clamp use the actual width. Placement happens **before `Show`** (no centered flash) against the work area of the monitor under the cursor (`MonitorFromPoint` + `GetMonitorInfo`), converting physical pixels with that monitor's effective DPI (`GetDpiForMonitor`); window styles use the 64-bit `Get/SetWindowLongPtr` (32-bit shim).

#### 7.3.7. `StatusReporter`
Implements `Current` + guarded `Report` + `StatusChanged`; subscriber exceptions are logged via an optional `ILogger<StatusReporter>` (single constructor with an optional parameter — DI supplies it, tests may pass `null`).

#### 7.3.8. `InMemoryLogSink`
600-entry ring buffer; `GetRecentLogs(200)`; on construction the ring is **seeded with the tail of the existing `logs.log`** (last ≤600 lines, best-effort — a failed load surfaces a `WARN` line inside the ring; never re-written, the pending file queue stays empty); `Emit` enqueues (UI ring + pending file queue) and marks dirty — pending entries are **appended** to `%LocalAppData%\ProcessGuardian\logs.log` by a background timer at most once per 500 ms (append-only, so file history survives restarts; a lock serializes timer, `Flush`, `Clear` and `Emit`). A failed append **re-enqueues** the batch for the next flush instead of losing it. At **512 KB** the file rotates to a single archive `logs.1.log` (older archive deleted; ~1 MB disk bound; rotation is best-effort and retried while the file is locked). `Clear()` empties the buffer and truncates the file immediately (drain + dirty reset under the write lock); `Flush()` persists pending entries (called from `MainViewModel.DisposeAsync`); the sink implements `IDisposable` (timer disposed, final flush); format `yyyy-MM-dd HH:mm:ss LEVEL  message`. **Changed from SPEC.md (was: full-file write on every `Emit`).**

#### 7.3.9. `Sha256IntegrityService`
Implements `ITargetIntegrityService`. Streams the file through `SHA256.HashDataAsync` and returns **uppercase hex**; `null` (never an exception) on `IOException` (incl. missing/locked file) or `UnauthorizedAccessException`; `OperationCanceledException` propagates.

#### 7.3.10. `ScheduledTaskAutostartService`
Admin edition's `IAutostartService` (§7.4.1 conditional registration). Wraps `schtasks.exe` (hidden, `UseShellExecute=false`); stdout/stderr are read concurrently (a sequential read can deadlock on full pipe buffers); argument builders are internal and unit-tested; never throws (exit code + output logged); `OperationCanceledException` propagates. See AS-06.

### 7.4. Presentation Layer (WPF)

#### 7.4.1. `Bootstrapper`
Registers `JsonConfigurationStore`, `InMemoryLogSink` (singleton, created before `AddLogging`), Serilog (sink only, `Microsoft.Hosting.Lifetime` → Warning), `WindowsProcessMonitor`, `Sha256IntegrityService`, `OverlayNotificationService`, `IAutostartService` (**regular:** `RegistryAutostartService`; **admin:** `ScheduledTaskAutostartService`), `TrayIconService`, `StatusReporter`, `GuardianOrchestrator` (+`AddHostedService`), `MainViewModel`, `MainWindow`.

#### 7.4.2. `MainViewModel`
Dependencies: `IConfigurationStore`, `IAutostartService`, `IStatusReporter`, `INotificationService`, `ILogger<MainViewModel>`, `InMemoryLogSink`, `ITargetIntegrityService`. Key behavior:
- Start **pins the hash first** (`PinTargetHashAsync`: empty path → empty pin and continue; unreadable file → **abort fail-closed** with a "Target Unreadable" card, monitoring stays off and the previous pin is kept; otherwise SHA-256 hex), then sets `IsMonitoringEnabled`, logs, shows "Monitoring Started" cards (if `EnableToasts`), and `FlushNow()` (immediate save). Pin-before-enable keeps the orchestrator's config poll from ever observing enabled + stale pin.
- Stop sets `IsMonitoringEnabled = false` and keeps the existing pin; shows "Monitoring Stopped" cards (if `EnableToasts`); `FlushNow()`.
- 500 ms debounced `ScheduleSave` → `FlushSaveAsync` (JSON + autostart registry sync unless `_suppressAutostartSync`); the initial load assigns `Settings` under a `_suppressSave` flag so settings.json is not rewritten right after being read.
- `OpenLogs` / `ClearLogs` commands.
- `StopMonitoringAndSaveAsync()` — used on user Exit: suppresses registry sync, disables monitoring, saves JSON, removes HKCU autostart.
- `DisposeAsync()` called explicitly by `App.OnExit`.

#### 7.4.3. `App.xaml.cs`
- `OnStartup` wraps `InitializeApplication` in a global try-catch.
- `SingleInstanceManager` first-instance check; `ShowRequested` + `StartWatching()`.
- `ShutdownMode = OnExplicitShutdown`.
- `StartHostAsync()` — **fire-and-forget** `Task.Run(async () => await _host!.StartAsync())` (no blocking `GetAwaiter().GetResult()`); a failed start is surfaced to the user via a MessageBox, not only crash.log.
- Tray Start/Stop resolve `MainViewModel` from DI; Exit → `_userRequestedExit = true`, `await StopMonitoringAndSaveAsync()` on the dispatcher (no `GetResult()` blocking), then `Shutdown()`.
- `OnMainWindowClosing` hides + `ShowInTaskbar=false`; `ShowMainWindow` sets `ShowInTaskbar=true`; `MainWindow` is registered **transient** and re-created after a real close (a closed WPF Window can never be re-shown).
- `OnExit`: disables monitoring on user exit, disposes VM (`DisposeAsync`), tray, host; each in its own try-catch.
- `OnDispatcherUnhandledException`: exceptions are swallowed (policy) and logged; the **first** one is also shown to the user in a `MessageBox`.
- The fatal `AppDomain.UnhandledException` path best-effort disposes the tray icon before `Environment.Exit(1)` (no ghost tray icon).
- Global exception handlers write to `%LocalAppData%\ProcessGuardian\crash.log`/`shutdown-error.log` via `CrashLog.Append` (rollover at 512 KB).

#### 7.4.4. `MainWindow.xaml`
Dark content-sized window (`SizeToContent=WidthAndHeight`, `ResizeMode=CanMinimize`, `UseLayoutRounding`), three cards (Settings / Monitoring / Log), custom styles (TextBox, mini numeric boxes, buttons, checkbox, scrollbar, scroll viewer), status pill with data-trigger colors, `StatusToDisplayConverter` for text. The `MiniBoxStyle` numeric boxes limit input to 3 characters (`MaxLength=3`, values up to 999) and keep the digits centered via `TextAlignment=Center`; the content host fills the whole box and the cursor is `IBeam`, so hovering and caret placement work over the entire box, not only over the digits.

#### 7.4.5. `MainWindow.xaml.cs`
Non-trivial: DWM dark caption, disables maximize (WndProc), log auto-scroll stick-to-bottom, custom scrollbar thumb sizing. Geometry is DIP-authored (`UseLayoutRounding` handles pixel snapping); there is no manual DPI-compensation code.

#### 7.4.6. Converters
- `StatusToDisplayConverter` — maps `GuardianStatus` → **`"MONITORING"`** (Waiting/Monitoring/Restarting), **`"ERROR"`** (Error), **`"STOPPED"`** (fallback, incl. Stopped). OneWay only (`ConvertBack` throws `NotSupportedException`); unknown → `"STOPPED"`.

#### 7.4.7. Notifications (overlay)
`OverlayNotificationService` → `ToastHost.Instance.ShowCard(title, iconPath)`; `OverlayWindow` (bottom-right of primary work area, toolwindow/no-activate styles); `ToastCard` (254×81 physical px, 5 s auto-hide, entrance/exit fade).

---

## 8. Configuration & Deployment

### 8.1. Settings File
Path: `%LocalAppData%\ProcessGuardian\settings.json`

```json
{
  "TargetProcessPath": "",
  "TargetHashSha256": "",
  "CheckIntervalSeconds": 5,
  "InitialDelaySeconds": 0,
  "MaxRestartAttempts": 5,
  "RestartDelaySeconds": 10,
  "StartupVerificationTimeoutSeconds": 5,
  "AutostartEnabled": false,
  "IsMonitoringEnabled": false,
  "EnableToasts": true
}
```

**Removed vs SPEC.md:** `TargetProcessArguments`, `EnableLogging`, `FailureCooldownSeconds`. **Added:** `EnableToasts`, `TargetHashSha256` (FR-17 pin; empty = no pin).

### 8.2. Publish

`build/publish.ps1` — PowerShell script with switches (framework-dependent profile **removed**):
- `-SelfContained` → `publish/self-contained` (`Release`, self-contained single file `ProcessGuardian.exe`).
- `-Admin` → `publish/self-contained-admin` (`Release-Elevated`, self-contained single file `ProcessGuardianAdmin.exe`, §8.4).
- No args → both profiles. Both share the same publish arguments (`-r win-x64 --self-contained true -p:PublishSingleFile=true -p:TrimMode=partial -p:IncludeNativeLibrariesForSelfExtract=true -p:EnableCompressionInSingleFile=true -p:DebugType=None -p:DebugSymbols=false`); verifies the expected exe exists and reports output size (~78 MB each).

**Removed vs SPEC.md:** the framework-dependent profile and its `publish/framework-dependent` output were removed.

### 8.3. CI/CD (`.github/workflows/build.yml`)

```yaml
name: Build & Test
on: [push, pull_request]
jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '9.0.x' }
      - run: dotnet restore
      - run: dotnet build --no-restore --configuration Release
      - run: dotnet build --no-restore --configuration Release-Elevated
      - run: dotnet test --no-restore --configuration Release --verbosity normal
      - run: dotnet test --no-restore --configuration Release-Elevated --verbosity normal
      - name: Publish (both editions, self-contained)
        run: powershell -NoProfile -ExecutionPolicy Bypass -File .\build\publish.ps1
      - uses: actions/upload-artifact@v4
        with:
          name: ProcessGuardian-x64
          path: ./publish/self-contained
          if-no-files-found: error
      - uses: actions/upload-artifact@v4
        with:
          name: ProcessGuardianAdmin-x64
          path: ./publish/self-contained-admin
          if-no-files-found: error
```

A second workflow, `.github/workflows/release.yml` (`Release`), runs on version tags (`v*`): it restores, builds and tests the `Release` configuration, publishes both self-contained editions, then creates a GitHub release (`gh release create`) attaching `ProcessGuardian.exe` and `ProcessGuardianAdmin.exe` with auto-generated notes.

### 8.4. Editions (Build Configurations)

One codebase, two editions. Identity is centralized in `Domain/AppEdition.cs`, selected by the `ELEVATED_EDITION` define (set for **all** projects via `Directory.Build.props` when building the `Release-Elevated` configuration; the WPF project additionally switches `AssemblyName` and the application manifest).

| | Regular (`Debug`/`Release`) | Admin (`Release-Elevated`) |
|---|---|---|
| Executable | `ProcessGuardian.exe` | `ProcessGuardianAdmin.exe` |
| Elevation | asInvoker (default manifest) | `requireAdministrator` (`app.admin.manifest`) |
| Data directory | `%LocalAppData%\ProcessGuardian` | Settings: `%ProgramData%\ProcessGuardianAdmin` (**admin-only ACL**, §8.5); logs/crash-logs: `%LocalAppData%\ProcessGuardianAdmin` |
| Single-instance mutex / show event | `ProcessGuardian_SingleInstance` / `ProcessGuardian_ShowRequested` | `ProcessGuardianAdmin_SingleInstance` / `ProcessGuardianAdmin_ShowRequested` |
| HKCU autostart value | `ProcessGuardian` | `ProcessGuardianAdmin` |
| Tray tooltip / menu labels | `ProcessGuardian - {Status}` / "Show ProcessGuardian" / "Exit ProcessGuardian" | `ProcessGuardianAdmin - {Status}` / "Show ProcessGuardianAdmin" / "Exit ProcessGuardianAdmin" |
| Monitoring of elevated targets | Not supported (medium IL: `MainModule` access denied → false "not running") | Supported: the guardian runs elevated, so the launched child inherits the elevated token and `MainModule` is readable |

Both editions can run side by side (all names above differ). The admin edition exists to monitor targets that themselves run elevated; non-elevated targets work in either edition. Window titles use `AppEdition.WindowTitle` (`MainWindow` binds it via `x:Static`); the toast icon pack URI is assembly-relative. **Tray icons are exception to edition-based naming:** the embedded resources keep fixed `LogicalName`s (`ProcessGuardian.Assets.TrayIconOn/Off/Error.ico`, Infrastructure.csproj) in both editions.

### 8.5. Admin Edition Settings Storage (admin-only ACL)

The admin edition's `settings.json` selects which binary the elevated guardian launches, so a user-writable location would be a local privilege-escalation vector. Therefore:

- Settings live at `%ProgramData%\ProcessGuardianAdmin\settings.json`.
- On startup `JsonConfigurationStore` creates the directory and calls `HardenForAdministratorsOnly`: **inheritance is stripped** (`SetAccessRuleProtection(true, false)`) and only **Administrators** and **SYSTEM** get `FullControl` (container+object inherit).
- The elevated UI can read and write settings normally; user-level processes cannot modify the target path or the FR-17 hash pin — the §10 residual-risk vector is **closed for the admin edition**.
- Logs and crash logs stay in `%LocalAppData%\ProcessGuardianAdmin` (not security-sensitive; the process writes its own profile).
- Best-effort: if the hardening cannot be applied, the failure is logged; the store keeps working (the file remains at the ProgramData path).

---

## 9. Testing Strategy (Current)

Total: **89 tests** (33 application + 49 infrastructure + 7 WPF) — all pass with `dotnet test` in both `Release` and `Release-Elevated` configurations.

### 9.1. Application Tests — `GuardianOrchestratorTests` (20), `AppSettingsClampTests` (4), `ProcessTargetValidationTests` (9)

Covering: running target (no restart, Monitoring); missing target → recovery toast + Monitoring + `Restarting` reported; exhaustion → Error + monitoring disabled + no cooldown + persists; budget restored on new Start (condition-wait, no fixed sleep); first Start consumes no attempts; free launch then budget consumed → Error; protected process skipped → restart; settings change picked up next iteration; `StartAsync` throws → loop stays alive; `ShowAsync` throws → restart still succeeds; `CheckAsync` throws → Error, loop continues; huge `CheckIntervalSeconds` → no busy loop (wait capped); hash mismatch → launch blocked + "Target Changed" + monitoring disabled + persisted, no `StartAsync`; hash read failure → blocked (fail-closed); hash match (case-insensitive) → launch proceeds; empty pin → check skipped, integrity service never called; Stop during the initial delay exits promptly; failed launches honor `RestartDelaySeconds`; `Unknown` check results skip the cycle without a restart or status flip and recover when the state is determinable again. `AppSettingsClampTests` (4): out-of-range numeric settings clamped to safe minimums; valid values untouched; values above 999 clamped down; boundary 999 kept. `ProcessTargetValidationTests` (9): empty/whitespace, relative, non-`.exe`, UNC, device, and missing paths rejected; valid path produces name/directory; `.exe` extension is case-insensitive (empty-path theory counts as two cases).

### 9.2. Infrastructure Tests (49 total)
- `InMemoryLogSinkTests` (4): ring caps at 600 / recent 200; format (no brackets, `INFO`/`ERR`, two spaces after level); `Clear` empties buffer and file. Temp-path sinks.
- `InMemoryLogSinkFlushTests` (6): `Emit` does not write synchronously, `Flush` persists; clean `Flush` is a no-op; `Clear` empties file immediately after flush; restart appends without truncating history; rotation to `logs.1.log` at the size cap; a failed append (locked file) keeps pending entries for the next flush.
- `InMemoryLogSinkHistoryTests` (3): constructor seeds recent logs from the file tail; restart shows old entries and appends new ones; `Flush` after seed does not duplicate file lines.
- `Sha256IntegrityServiceTests` (3): known content → uppercase hex digest; deterministic across calls; missing file → `null`.
- `AdminConfigDirectoryAclTests` (1): hardening strips inheritance and leaves exactly Administrators/SYSTEM allow rules.
- `AdminConfigFileAclTests` (2): file hardening strips inheritance (Administrators/SYSTEM only); a directory created with the hardened security is born protected.
- `ScheduledTaskAutostartServiceTests` (4): create arguments (`ONLOGON`/`RL HIGHEST`/`/F`, task name, quoted `/TR` with `--background`); quoted path with spaces; delete; query.
- `CrashLogTests` (4): append creates dir/file; appends twice; rotation above size limit; content preserved under limit.
- `JsonConfigurationStoreTests` (7): round-trip; missing file → defaults; corrupted JSON → defaults; invalid intervals clamped on load; values above 999 clamped on load; concurrent access; `IsMonitoringEnabled` persisted to disk (read back by a fresh store). Runs on a scratch directory (internal constructor seam), never on the real config location.
- `RegistryAutostartServiceTests` (4): enable/disable; quoted exe path + `--background`; no-throw when not set. Runs against a scratch HKCU key.
- `TrayIconServiceTests` (5): construct without assets (fallback); dispose; second-item label toggle ("Start monitoring"/"Disable"); reopen after dismiss. Runs on STA threads.
- `WindowsProcessMonitorTests` (4): detect running notepad; case-insensitive path match; after-kill NotRunning; `StartAsync` nonexistent path returns false.
- `StatusReporterTests` (2): subscriber exception swallowed, `Current` still updated (with and without logger).

> `ProcessGuardian.Infrastructure.Tests` sets `<UseWPF>true</UseWPF>` to support tray/STA tests.

### 9.3. WPF Tests (7 total)
- `MainViewModelReentrancyTests` (4): reentrant Start/Stop ignored while busy; Start pins the hash **before** enabling monitoring (hash-computation gate); unreadable hash → fail-closed abort, previous pin kept, no enable-save, "Target Unreadable" card.
- `ToastEvictionTests` (3): 5th card evicts the oldest synchronously (max 4, deadlock regression); a late `Dismiss` on an evicted card never calls back into the overlay; duplicate `Dismiss` calls are no-ops. Runs on STA threads.

---

## 10. Known Limitations / Differences from SPEC.md

1. **Notifications are in-app overlay cards**, not Windows toasts; `Microsoft.Toolkit.Uwp.Notifications` is not referenced by any project.
2. **Arguments support removed** — target processes launch with no command-line arguments.
3. **Failure cooldown removed** — exhaustion disables monitoring instead of entering a cooldown.
4. **`EnableLogging` removed** — logging is always on; file output via `InMemoryLogSink` (`logs.log`), not Serilog File sink.
5. **`EnableToasts` added** to replace `EnableLogging`-style opt-out.
6. **Distribution size NFR not met** (≈78 MB self-contained single-file per edition; the framework-dependent profile was removed).
7. **Power events not implemented** (deferred `[MAY]`).
8. Main window is a content-sized dark design (`SizeToContent`), not the fixed-size form in `SPEC.md`.
9. `TrayMenuPopup` custom dark menu replaces the standard `ContextMenu`.
10. Single-instance uses a named event + watcher thread in addition to the mutex.
11. `JsonConfigurationStore` caches in memory and writes atomically via temp file.
12. **Target pinning residual risk (regular edition only):** hash pinning (FR-17) defeats substitution of the binary at the pinned path, but the regular edition's `settings.json` lives in a user-writable directory — a user-level process can rewrite both the path and the matching hash. The **admin edition** stores settings in an admin-only location (§8.5), closing this vector; for the regular edition the risk is accepted (no privilege gain without elevation).

---

## 11. Git History (all commits, newest → oldest)

| Short Hash | Date | Commit Message |
|------------|------|----------------|
| `e07bc18` | 2026-10-08 | Docs: add keywords footer to README |
| `afbcf32` | 2026-10-08 | Chore: trim .gitignore to project-relevant entries |
| `d1fd902` | 2026-10-08 | Docs: add MIT license |
| `3632a9c` | 2026-10-08 | Docs: trim README to user-facing content |
| `e3e9165` | 2026-10-08 | Docs: rework README around the Best-README template with user-facing UI, tray, and toast-card docs |
| `70fa0ae` | 2026-10-07 | Deps and docs: package patches, Hosting.Abstractions and README sync |
| `4fa7f5d` | 2026-10-07 | Fix: monitor-aware tray menu placement, 64-bit style P/Invoke, concurrent schtasks reads, surfaced config-save failures |
| `97e0750` | 2026-10-07 | Fix: shutdown and failure-path lifecycle polish |
| `6a229be` | 2026-10-07 | Fix: log-sink hygiene, crash-log serialization, visible subscriber failures |
| `9865202` | 2026-10-07 | Fix: distinguish failed enumeration from a dead process (Unknown outcome) |
| `cfdfeac` | 2026-10-07 | Tests: full machine-state isolation |
| `b0eb6dd` | 2026-10-07 | Fix: orchestrator state machine responsiveness, Restarting status, backoff |
| `6596999` | 2026-10-07 | Fix: pin target hash before enabling monitoring, fail closed on unreadable target |
| `f560aa4` | 2026-10-07 | Security: born-hardened config directory and re-hardened settings file (admin) |
| `9898c9f` | 2026-10-07 | CI: Release-Elevated solution configuration, elevated test run, release gate |
| `5d23e55` | 2026-10-07 | Fix: toast stack eviction removes oldest card synchronously (UI deadlock) |
| `0de417e` | 2026-10-07 | add assets |
| `a082560` | 2026-10-04 | Remove one-time administrator-mode notice and its settings flag |
| `8ca3e16` | 2026-10-04 | Fix: tray menu width grows to fit the widest label (long edition names) |
| `0950005` | 2026-10-04 | Fix: tray icons use fixed LogicalNames in both editions |
| `4a01614` | 2026-10-04 | Build: self-contained publish for both editions, drop framework-dependent profile |
| `05ed4f8` | 2026-10-04 | Admin edition: scheduled-task autostart (silent elevation) and one-time administrator-mode notice |
| `93ce306` | 2026-10-04 | Security: admin edition stores settings under ProgramData with Administrators-only ACL |
| `5ff7b11` | 2026-10-04 | Editions: Release-Elevated build (ProcessGuardianAdmin) with edition-aware identity for side-by-side runs |
| `b638318` | 2026-10-03 | Security: pin target binary with SHA-256, block launches on hash mismatch |
| `370c8bf` | 2026-10-02 | Security: validate target path (absolute, .exe, local drive, file exists) before any launch |
| `29d2e3f` | 2026-10-01 | Logging: seed log card with logs.log tail on startup |
| `09ac8eb` | 2026-10-01 | Logging: append-only logs.log with 512KB archive rotation (history survives restart) |
| `153b58c` | 2026-10-01 | UI: hide whole log scrollbar (arrows and track) while log fits viewport |
| `5312176` | 2026-10-01 | Render: restore fixed text hinting, keep 12.5 start/stop button font |
| `dacd515` | 2026-10-01 | Render: disable text hinting (Animated) and reduce start/stop button font to 12.5 |
| `b4eb279` | 2026-10-01 | Render: grayscale text smoothing and SemiBold weight in main window |
| `93b9fd0` | 2026-09-30 | Initial commit |

### Full commit hashes (newest → oldest)

| Short | Full Hash |
|-------|-----------|
| `e07bc18` | `e07bc18faf07e1de81a5540d2026aff321a7766b` |
| `afbcf32` | `afbcf325c7ba84a5199c77c761c98cfcb9c90ddc` |
| `d1fd902` | `d1fd9023e94c8596c9dd656ae80318f32077b5a1` |
| `3632a9c` | `3632a9c71d57c8584cc155be78dba616af6fa9e9` |
| `e3e9165` | `e3e9165dc83314964814c44ead2729a113a27c83` |
| `70fa0ae` | `70fa0aef909e49ff371a85b525869229f246d33f` |
| `4fa7f5d` | `4fa7f5dc010e6a6483106bf082fa964a34447d9b` |
| `97e0750` | `97e075012e7120fc3d7ff2328c5986204f2368c6` |
| `6a229be` | `6a229be0011dffdb3747281be213fbbd655c3033` |
| `9865202` | `98652023c635bcb3d1cfd5acdf08784607caeafa` |
| `cfdfeac` | `cfdfeacf939e8822a08d20a52f3efac4811b5c13` |
| `b0eb6dd` | `b0eb6ddb682a0b9cca57884cc9741a718ca79c5f` |
| `6596999` | `6596999e223f4723580d43b791b4021521ee8d40` |
| `f560aa4` | `f560aa460bb50ee9df10db2368ff1eca6b530cf8` |
| `9898c9f` | `9898c9f9db26862aa91c63bec8eb02c85249a580` |
| `5d23e55` | `5d23e55881424ffb06fc01b9c5544648def4c920` |
| `0de417e` | `0de417e42568a830b054340f9dc114584adbd7ea` |
| `a082560` | `a0825606e824d46bf34d645e971d74ea451b0208` |
| `8ca3e16` | `8ca3e16635a005e98c4d1501f9afa860327e3622` |
| `0950005` | `0950005baad2c5e5647e6ce2205b63e86496dcab` |
| `4a01614` | `4a0161482f0c66dd9357951746020075c0e00895` |
| `05ed4f8` | `05ed4f862529a08e012c49594241bc727ec3ce35` |
| `93ce306` | `93ce30698e5fca7f821d4acf0fba9ba8c545dca0` |
| `5ff7b11` | `5ff7b116c0f5ee7040c0549be3384f471115935b` |
| `b638318` | `b6383188c3ad06dbf77cbc0ece9d40396a919eee` |
| `370c8bf` | `370c8bf2d49207b757dc138a3b697ceda300e532` |
| `29d2e3f` | `29d2e3fd8b0057f61a8e0c0fe730086d15b853ae` |
| `09ac8eb` | `09ac8ebb0da254e7bba71f055dc18e58e7166c89` |
| `153b58c` | `153b58c3225fe062fe5cf98913ec8b653bd23da5` |
| `5312176` | `53121765c8530a386fe35b9466fc017f4b977362` |
| `dacd515` | `dacd51544e47f7b5d79391ca8f954a7aa16fedf1` |
| `b4eb279` | `b4eb2794022cebc665ca69c8cc8ebddc114ae6ef` |
| `93b9fd0` | `93b9fd044249d94613959fa367d93c1ed2c1a694` |

---

## 12. File Protection & Commit Policy

The following files are **protected** and must never be committed to git during ordinary project work:

| File | Policy |
|------|--------|
| `SPEC.md` | **Commits touching this file are forbidden.** It is edited only on explicit user instruction. |
| `AGENTS.md` | **Commits touching this file are forbidden.** It is edited only on explicit user instruction. |
| `opencode.json` | **Commits touching this file are forbidden.** It is edited only on explicit user instruction. |

Rules:

- **FP-01** — An Increment is committed only with the files belonging to that Increment; `SPEC.md`, `AGENTS.md`, and `opencode.json` are never staged or committed.
- **FP-02** — `SPEC.md` may be updated on explicit user instruction (this document is source of truth for the implemented architecture and behavior, but it is a working reference, not part of the committed code).
- **FP-03** — Do not overwrite, revert, stage, or commit pre-existing user changes to these files.
- **FP-04** — A broad `git add .` / `git add -A` must not silently stage these files; verify `git status` before any commit.
- **FP-05** — These policies mirror the rules in `AGENTS.md` (§3 and §12 of that file) and keep the repository's governance, tooling, and specification out of the commit history.

---

**End of current project state**

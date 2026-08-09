# Migration analysis: LOIC from .NET Framework 2.0 to .NET 10

Date: 2026-08-09. The implementation status is right below this introduction; the analysis
part from section 1 onward documents the original plan.

Goal: lift both projects of the solution from .NET Framework 2.0 (old non-SDK csproj format)
to **.NET 10** and get a clean build.

---

## 0. Implementation status (2026-08-09)

The migration is implemented and committed (branch `main`, not pushed yet). The full
`dotnet build` is green (0 errors, 0 warnings), `LOIC.exe` is produced and starts, and the
runtime risks are covered by non-interactive tests, see below.

### Done (each a separate commit)

- IRC to SDK-style/`net10.0`, log4net removed, AssemblyInfo migrated, fixed version 0.4.0.0.
- `Thread.Abort()` replaced with cooperative shutdown: 3x in `IrcConnection.cs`
  (Read/Write/IdleWorkerThread) plus a 4th spot the analysis had missed in `frmMain.cs`
  (IRC listen thread, stopped via `ircenabled` + `Disconnect()`).
- LOIC to SDK-style/`net10.0-windows` + `UseWindowsForms`; IRC subtree excluded from the
  default globs; AssemblyInfo migrated.
- `Process.Start` at 3 spots switched to `UseShellExecute = true`; `<startup>` block removed
  from `app.config`.
- Cleanups: redundant PackageReferences removed (NU1510); obsolete serialization constructors
  removed from the IRC exceptions (SYSLIB0051); SYSLIB0014 suppressed with a scoped `#pragma`
  around the Overlord web calls; `LOIC.sln` raised to format 12.00; `LOIC.userprefs` removed.
- All code and project-file comments in English (including the pre-existing German designer
  comments in `frmEZGrab.Designer.cs`).

### Steps 5 + 6: build and runtime test (done)

`LOIC.dll` was previously quarantined by Windows Defender on every build as
`HackTool:Win32/Oylecann.A` (threat ID 2147641076) before the `CreateAppHost` step could
bundle it into `LOIC.exe`. After adding a Defender exclusion the solution builds green (0
errors, 0 warnings) and produces `LOIC.exe`.

Non-interactive runtime test:
- Startup run `LOIC.exe /hidden`: the process starts and runs stably, no crash while building
  `frmMain` (InitializeComponent, settings read, Konami check).
- resx bitmaps (2.6): the images actually used (`Properties.Resources`: LOIC 184x463, WTF
  416x300, icon 32x32) deserialize at runtime through the preserialized path without issues.
  The `Bitmap1` present in all five resx files is not referenced anywhere in the code, is
  dropped during the build, and is therefore irrelevant.
- ConfigurationManager (2.7): the `Settings.UpdateSetting`/`ReadSetting` round-trip via
  `OpenExeConfiguration` persists and reads back correctly.

### Still open (needs real external connections/interaction)

- IRC thread stop (2.4): the cooperative ReadThread teardown has not yet been tested against a
  real IRC connection (Hivemind); the code review supports the change, a live test is
  outstanding.
- `Process.Start` (2.5): `UseShellExecute = true` is the documented standard fix, but the
  actual opening of the GitHub link / `help.chm` / EZGrab URL was not clicked interactively.

### Open/optional

- HttpClient migration instead of WebRequest/WebClient (SYSLIB0014 is currently only
  suppressed), sensible only once it can be runtime-tested.
- A real flood run is deliberately not executed against third-party hosts.

---

## 1. Starting point

| Project | Type | Current target | Format | Notes |
|---|---|---|---|---|
| `src/LOIC.csproj` | `WinExe` (WinForms) | `.NET FW v2.0` | Non-SDK, ToolsVersion 4.0 | 4 forms + Designer/resx, `app.manifest`, `System.Configuration`, PlatformTarget `x86` |
| `src/IRC/IRC.csproj` | `Library` | `.NET FW v2.0` | Non-SDK, ToolsVersion 4.0 | vendored SmartIrc4net 0.4.0, `log4net` via `packages.config` |

`LOIC` references `IRC` via ProjectReference. `LOIC.sln` is format version 11.00 (VS 2010).

**Environment is ready:** `dotnet --list-sdks` shows among others `10.0.110` and `10.0.302`,
so a .NET 10 SDK is installed.

Target framework proposal:
- `LOIC` -> `net10.0-windows` with `<UseWindowsForms>true</UseWindowsForms>` (WinForms exists
  on modern .NET only Windows-only).
- `IRC` -> `net10.0` (the library only uses `System.Net` sockets, threading and non-generic
  collections, nothing Windows-specific; a platform-neutral target is cleaner.
  `net10.0-windows` would also work.)

---

## 2. Breakage catalog

Severity legend:
- BLOCKER = does not build or is guaranteed to throw at runtime.
- RISK = may break, must be verified.
- MECHANICAL = pure format change, behavior unchanged.
- COSMETIC = warning or dead code, no functional effect.

### 2.1 Project file format (both projects) - MECHANICAL

Both `.csproj` files are in the old non-SDK format with manual `<Compile Include>` lists,
`<Reference Include="System...">` and `<Import ...Microsoft.CSharp.targets>`.

Switch to SDK-style (`<Project Sdk="Microsoft.NET.Sdk">`):
- The file shrinks massively because SDK-style globs all `*.cs` automatically.
- The GAC references (`System`, `System.Data`, `System.Drawing`, `System.Windows.Forms`,
  `System.Xml`) go away; they come from the framework resp. `UseWindowsForms`.
- resx/Designer association (DependentUpon) is handled automatically by the SDK via the file
  naming convention.

### 2.2 packages.config -> PackageReference (IRC) - MECHANICAL

`src/IRC/packages.config` holds `log4net 2.0.10` (targetFramework `net20`), while the
`HintPath` in the csproj still points at `log4net.2.0.5\lib\net20-full\log4net.dll`
(inconsistent after the Dependabot bump). SDK-style uses `<PackageReference>`. The
`src/packages/` folder is then no longer needed.

But see 2.3: the reference is effectively unused in the current build.

### 2.3 log4net is a dead reference - COSMETIC (but important to know)

The entire log4net code in the IRC library sits behind `#if LOG4NET` (117 occurrences across
9 files, including all of `Logger.cs`). **The `LOG4NET` symbol is not defined in either
csproj** (they only have `DEBUG;TRACE` resp. `TRACE`).

Consequence: log4net is referenced as an assembly, but not a single call is compiled. The
dependency is inert in the current build.

Options for the migration:
- (a) Keep `log4net` as a PackageReference, still **not** defining `LOG4NET`. Behavior stays
  exactly as today (logging off). Simplest, lossless variant.
- (b) Remove `log4net` and `packages.config` entirely. Minimal dependency footprint. If
  logging is wanted later, it would have to be added back.
- (c) Deliberately define `LOG4NET` and enable logging. Larger test and configuration effort
  (log4net config is currently missing, see the commented-out block in `Logger.cs`). Not
  recommended within a pure framework migration.

Recommendation: (a) for maximum behavioral parity, alternatively (b) for cleanliness.

### 2.4 Thread.Abort() (IRC) - BLOCKER

`src/IRC/Connection/IrcConnection.cs`, 3 spots: lines 812, 900, 1120. Each in the `Stop()`
method of three internal worker classes:

- `ReadThread.Stop()` (812): calls `_Thread.Abort()`, then `_Connection._Reader.Close()`. The
  worker (`_Worker`, from 819) runs `while (_Connection.IsConnected && ...)` and blocks in
  `ReadLine()` on the socket.
- `WriteThread.Stop()` (900): calls `_Thread.Abort()`. Worker (from 907) runs
  `while (_Connection.IsConnected)` with `Thread.Sleep(_SendDelay)` and drains the send buffers.
- `IdleWorkerThread.Stop()` (1120): calls `_Thread.Abort()`. Worker (from 1123) runs
  `while (_Connection.IsConnected)` with `Thread.Sleep(_IdleWorkerInterval)`.

`Thread.Abort()` exists on modern .NET only as a stub and throws `PlatformNotSupportedException`
at runtime. This is the central code blocker of the migration.

Good news: all three workers already poll `_Connection.IsConnected`, and the disconnect path
sets `_IsConnected = false` (line 472/548) before the `Stop()` methods run. The hard abort was
therefore historically only the crowbar.

Planned replacement (cooperative shutdown):
- Remove `Thread.Abort()`.
- WriteThread/IdleWorkerThread: stop on their own via the existing `IsConnected` polling after
  the next `Sleep` interval. No further change needed, except optionally `_Thread.Join(timeout)`
  for a clean wait.
- ReadThread: blocks in `ReadLine()`. The existing `_Reader.Close()` interrupts the blocking
  read (throws an exception there that the worker catches/ends on). Order: close the reader
  first (wakes the read), then optionally `Join(timeout)`.
- Since all threads are `IsBackground = true`, a lingering leftover thread does not prevent
  process exit anyway.

RISK note: this is a small but behavior-relevant change to the network library. It should be
tested with a real IRC connection (Hivemind mode), not just "builds green".

### 2.5 Process.Start without UseShellExecute - BLOCKER (runtime)

3 spots that want to open a URL resp. file via the shell:
- `frmEZGrab.cs:46` -> `System.Diagnostics.Process.Start(turl)` (open URL)
- `frmMain.cs:1068` -> `Process.Start("https://github.com/NewEraCracker/LOIC")`
- `frmMain.cs:1562` -> `Process.Start("help.chm")`

On .NET Framework `UseShellExecute` was implicitly `true`, so `Process.Start("https://...")`
worked. On modern .NET the default is `false`, so these calls throw a `Win32Exception` ("The
system cannot find the file specified") because the string is not interpreted as an executable.

Fix per spot:
```csharp
Process.Start(new ProcessStartInfo(url) { UseShellExecute = true });
```

### 2.6 resx with BinaryFormatter payload - RISK (potentially BLOCKER)

**All 5 resx files** contain a `Bitmap1` object with
`mimetype="application/x-microsoft.net.object.binary.base64"`, i.e. serialized with
BinaryFormatter:
- `frmEULA.resx:22`, `frmEZGrab.resx:22`, `frmMain.resx:22`, `frmWtf.resx:22`,
  `Properties/Resources.resx:22`

BinaryFormatter is disabled resp. removed by default on .NET 9/10. That affects two layers:
1. Build: the `GenerateResource` task may need to be built with
   `<GenerateResourceUsePreserializedResources>true</GenerateResourceUsePreserializedResources>`
   and PackageReference `System.Resources.Extensions`.
2. Runtime: WinForms on .NET ships a limited compat reader for known types (including
   `System.Drawing.Bitmap`) that can read the legacy binary blob without a general
   BinaryFormatter. Whether that applies to exactly this `Bitmap1` data is version-dependent
   and **must be tested**.

Each resx also has an `Icon1` as `bytearray.base64` (uncritical, modern way).

Solution options (by effort):
- (a) First build with `System.Resources.Extensions` +
  `GenerateResourceUsePreserializedResources` and test at runtime. If the bitmaps load
  correctly, no further change needed.
- (b) If it breaks: re-save the affected `Bitmap1` resources once in the modern format (via the
  WinForms designer or a small conversion script base64-binary -> PNG/bytearray). After that no
  BinaryFormatter path is needed anymore.

First check what role `Bitmap1` plays in the respective form (Designer.cs references it) before
investing effort in (b).

### 2.7 System.Configuration / ConfigurationManager (LOIC) - MECHANICAL

`src/Settings.cs` uses `ConfigurationManager.AppSettings`,
`ConfigurationManager.OpenExeConfiguration(...)` and `RefreshSection(...)` to read/write
`AcceptEULA` and `KonaniCode` in the app config.

On .NET this lives in the NuGet package `System.Configuration.ConfigurationManager`. Add a
PackageReference, then `Settings.cs` compiles unchanged.

RISK note: the write path (`OpenExeConfiguration` writes to `<App>.exe.config` next to the EXE)
should be verified briefly; the file naming logic differs slightly between Framework and modern
.NET.

### 2.8 app.config startup block - COSMETIC

`src/app.config` contains only:
```xml
<startup useLegacyV2RuntimeActivationPolicy="true">
    <supportedRuntime version="v2.0.50727"/>
    <supportedRuntime version="v4.0"/>
</startup>
```
This is pure Framework runtime activation and is ignored by modern .NET. Harmless, but should
be removed. Note: there is currently **no** `appSettings` section; `Settings.cs` creates it at
runtime. When cleaning up app.config, do not remove anything `Settings.cs` needs (currently
nothing, but keep an eye on it).

### 2.9 AssemblyInfo duplication and wildcard version - BLOCKER (build)

- SDK-style generates the assembly attributes automatically. The existing
  `Properties/AssemblyInfo.cs` (both projects) contain `AssemblyTitle`, `AssemblyVersion` etc.
  -> **error "duplicate attribute"**. Solution: either set
  `<GenerateAssemblyInfo>false</GenerateAssemblyInfo>` and keep the old files, or migrate the
  attributes into the csproj properties and delete the old files.
- `src/IRC/Properties/AssemblyInfo.cs` has `[assembly: AssemblyVersion("0.4.0.*")]`. The `*`
  wildcard is **not allowed** with deterministic builds (default in the .NET SDK) and breaks the
  build. Solution: fixed version (e.g. `0.4.0.0`) or `<Deterministic>false</Deterministic>`.
  Recommendation: fixed version.
- `#if DELAY_SIGN` in the IRC AssemblyInfo is not defined, so inactive. No problem.

### 2.10 WebClient obsolete (LOIC) - COSMETIC

`frmMain.cs:1385` uses `new WebClient()`. Still exists in .NET 10, but is marked obsolete as
`SYSLIB0014` (warning). `HttpWebRequest`/`WebRequest` (`frmMain.cs:1353/1355`) also still work.
Not a blocker. Modernization to `HttpClient` would be optional (not part of the minimal lift).

### 2.11 PlatformTarget x86 - DECISION

Both projects set `PlatformTarget x86`. On .NET 10 that can stay
(`<PlatformTarget>x86</PlatformTarget>`), or switch to `AnyCPU`/`x64`. x86 is unusual today; a
DDoS network tool does not benefit from it. Proposal: `AnyCPU`, unless there is a known reason
for x86.

---

## 3. What is uncritical and simply passes through

- **Sockets/network in the flooders**: `HTTPFlooder.cs`, `XXPFlooder.cs`, `cHLDos.cs`,
  `Protocol.cs`, `Functions.cs` and `IrcTcpClient.cs` use `Socket`, `TcpClient`,
  `NetworkStream`, `Dns`, `IPEndPoint`, `Encoding`, `StreamReader/Writer`. All present in .NET
  10 and API-compatible.
- **Non-generic collections** (`SortedList`, `Queue`, `Hashtable`) in the IRC lib: all still
  present.
- **Program.cs**: `[STAThread]`, `Application.SetCompatibleTextRenderingDefault(false)`,
  `Application.Run(...)` - all WinForms standard, runs. (Optionally one could use
  `ApplicationConfiguration.Initialize()`, but it is not required.)
- **app.manifest**: `requestedExecutionLevel asInvoker` + supportedOS list. Still supported via
  `<ApplicationManifest>`, can stay unchanged.
- **No P/Invoke / Marshal / Registry / Microsoft.VisualBasic / unsafe / ServicePointManager /
  TLS fiddling** found in the code. `Konami.cs` is purely managed.
- **Icon/EmbeddedResource** (`Resources\LOIC.ico`, EULA.rtf, WTF.jpg, LOIC.gif): uncritical.

---

## 4. Planned diff (overview per file)

New/rewritten:
- `src/IRC/IRC.csproj` -> SDK-style, `net10.0`, PackageReference `log4net 2.0.x` (or remove,
  see 2.3), fixed AssemblyVersion, `GenerateAssemblyInfo` clarified.
- `src/IRC/packages.config` -> **delete**.
- `src/LOIC.csproj` -> SDK-style, `net10.0-windows`, `UseWindowsForms=true`, PackageReference
  `System.Configuration.ConfigurationManager`, possibly `System.Resources.Extensions` +
  `GenerateResourceUsePreserializedResources` (see 2.6), keep `ApplicationManifest`/
  `ApplicationIcon`.
- `src/LOIC.sln` -> raise to the current format (optional, VS/dotnet update it on first save;
  not mandatory).

Code changes (small, targeted):
- `src/IRC/Connection/IrcConnection.cs` -> replace 3x `Thread.Abort()` with cooperative shutdown
  (2.4).
- `src/frmEZGrab.cs:46`, `src/frmMain.cs:1068`, `src/frmMain.cs:1562` -> `Process.Start` with
  `ProcessStartInfo { UseShellExecute = true }` (2.5).

Config/resources:
- `src/app.config` -> remove `<startup>` block (2.8).
- 5x `Bitmap1` resource: only touch if the runtime test fails (2.6).

AssemblyInfo:
- `src/Properties/AssemblyInfo.cs` and `src/IRC/Properties/AssemblyInfo.cs` -> either keep with
  `GenerateAssemblyInfo=false`, or migrate into the csproj and delete; fix the wildcard version
  (2.9).

Cleanup:
- `src/packages/` -> remove (redundant after the PackageReference switch).

---

## 5. Recommended order

1. `IRC.csproj` to SDK-style + `net10.0`, decide the log4net question, fix AssemblyVersion.
2. Replace `Thread.Abort()` in `IrcConnection.cs`. Get `dotnet build src/IRC` green.
3. `LOIC.csproj` to SDK-style + `net10.0-windows` + `UseWindowsForms`, config package, choose
   resx strategy.
4. Fix the `Process.Start` spots. Clean up `app.config`.
5. Get `dotnet build` of the whole solution green, review warnings.
6. **Runtime test**: start the EXE, EULA dialog (resx bitmaps!), a flood run, optionally
   Hivemind/IRC (thread-stop path).

---

## 6. Open decisions (for you)

1. **log4net**: keep (2.3a) or remove (2.3b)?
2. **resx bitmaps**: test first and only re-save on breakage (2.6a) - probably fine as is.
3. **PlatformTarget**: `AnyCPU` instead of `x86` (2.11)?
4. **AssemblyInfo**: keep the old files (`GenerateAssemblyInfo=false`) or migrate into the
   csproj?
5. **IRC target**: `net10.0` (recommended, portable) or `net10.0-windows`?
6. **.sln format** update yes/no?

---

## 7. Risk summary

| Item | Severity | Safely automatable? |
|---|---|---|
| csproj SDK-style (both) | mechanical | yes |
| packages.config -> PackageReference | mechanical | yes |
| log4net dead code | cosmetic | yes |
| Thread.Abort (3x) | BLOCKER | yes, with a subsequent IRC test |
| Process.Start (3x) | BLOCKER (runtime) | yes |
| resx BinaryFormatter (5x) | RISK | partly, runtime test needed |
| ConfigurationManager package | mechanical | yes |
| app.config startup | cosmetic | yes |
| AssemblyInfo/wildcard | BLOCKER (build) | yes |
| WebClient obsolete | cosmetic (warning) | yes |
| PlatformTarget x86 | decision | yes |

Conclusion: technically well doable. The only real thinking work is `Thread.Abort` (small, but
needs testing) and the BinaryFormatter bitmaps in the resx (very likely unproblematic, but
verify). The rest is a format switch and two small code fixes.

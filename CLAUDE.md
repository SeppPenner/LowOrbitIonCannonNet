# Project rules for Claude

## What this is

Low Orbit Ion Cannon (LOIC) is an open source network stress-test tool written in C#, a .NET
port of Praetox's LOIC. It is for educational and defensive purposes only (see `README.md`).

The solution `src/LOIC.sln` holds two projects:

- `src/LOIC.csproj`: the WinForms application. `WinExe`, `net10.0-windows`,
  `UseWindowsForms=true`, `AnyCPU`, startup object `LOIC.Program`. Forms `frmMain`, `frmEULA`,
  `frmEZGrab`, `frmWtf`; flooders `HTTPFlooder`, `XXPFlooder`, `cHLDos` with helpers
  `Protocol.cs`, `ReqState.cs`, `Functions.cs`. Assembly version `2.9.9.99`.
- `src/IRC/IRC.csproj`: a vendored copy of the SmartIrc4net 0.4.0 library. `net10.0`, pinned
  version `0.4.0.0`. Used for Hivemind mode, where the client connects to an IRC server and is
  controlled remotely (`/hivemind`, `/hidden` switches, see `README.md`).

`LOIC` references `IRC` via `ProjectReference`. The project was migrated from .NET Framework 2.0
to .NET 10; the full migration analysis and status is documented in `analysis.md`.

## Build

```
dotnet build src/LOIC.sln -c Release
```

The current build is clean (0 errors, 0 warnings).

Build quirk (Windows Defender): `LOIC.dll` is flagged as `HackTool:Win32/Oylecann.A` (threat ID
2147641076) and quarantined on build, which breaks the `CreateAppHost` step that bundles it into
`LOIC.exe`. Add a Defender exclusion for the build output folder, otherwise the build fails at
the apphost step. This is a false positive on the tool itself, not on the build.

## Code conventions

- Formatting and style rules live in `src/.editorconfig`: CRLF line endings, 4 spaces, UTF-8,
  file-scoped namespaces, `using` directives inside the namespace and System-first, `this.`
  qualification for fields, properties, methods and events, required braces, no consecutive
  blank lines, `IDE0005` as a warning.
- Code comments are English (see below), including in project files such as `.csproj`.

## Known quirks

- The resx files still carry a BinaryFormatter-serialized `Bitmap1` blob. The build passes them
  through with `<GenerateResourceUsePreserializedResources>true</GenerateResourceUsePreserializedResources>`
  in `LOIC.csproj`. `Bitmap1` is unreferenced dead data and is dropped during the build; the
  images actually used come from `Properties.Resources`.
- The IRC project lives under `src/IRC/`. `LOIC.csproj` removes `IRC\**` from the SDK default
  globs so those files are not compiled twice.
- The former log4net logging in the IRC library sits behind `#if LOG4NET`, a symbol that is not
  defined anywhere, so the logging is inert. The log4net dependency was removed during the
  migration.
- The IRC assembly version is pinned to `0.4.0.0`; the original `0.4.0.*` wildcard is illegal
  with deterministic builds.
- Neither project has any NuGet `PackageReference`; everything comes from the shared framework
  (`System.Configuration.ConfigurationManager` and `System.Resources.Extensions` are part of the
  Windows Desktop shared framework on `net10.0-windows`, see NU1510).

## Releasing / versioning

There is no installer, release automation or version tags in this repository. The deliverable is
the built `LOIC.exe`. The version lives in `src/LOIC.csproj` (`AssemblyVersion`/`FileVersion`/
`Version`); bump it there when a release is needed.

## Git

- Never use `git commit --amend`, not even for purely local commits that have not been pushed.
  Write a follow-up commit instead. This keeps the history append-only and avoids rewriting
  commits that may already have been published.

## Commits

- Commit messages are written **in English only**.
- Short, precise summary in the subject line, plus an explanatory body when needed.

## Punctuation

- **No em dashes or en dashes** (`—`, `–`), neither in prose, commit messages, code comments
  nor documentation.
- Use a regular hyphen, comma, colon, parentheses or a separate sentence instead.

## Code comments

- Comments in code (and in project files such as `.csproj`) are **always written in English**,
  regardless of the language used in the rest of the communication.

## German texts

- In German texts (documentation, chat replies) always use **real umlauts and ß**, never ASCII
  transliterations.
- Rewrite where needed:
  - `ae` -> `ä`
  - `oe` -> `ö`
  - `ue` -> `ü`
  - `Ae` -> `Ä`, `Oe` -> `Ö`, `Ue` -> `Ü`
  - `ss` -> `ß` (only where orthographically correct, e.g. `Strasse` -> `Straße`; `dass` stays
    `dass`)
- This applies to documentation files and chat, **not** to code comments (those are English,
  see above).
- Exception: identifiers, file names, configuration keys and similar stay unchanged when umlauts
  are technically undesirable there.

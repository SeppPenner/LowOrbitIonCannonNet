# Migrationsanalyse: LOIC von .NET Framework 2.0 auf .NET 10

Stand: 2026-08-09. Der Umsetzungsstand steht direkt unter dieser Einleitung; der
Analyseteil ab Abschnitt 1 dokumentiert den ursprünglichen Plan.

Ziel: Beide Projekte der Solution von .NET Framework 2.0 (altes Non-SDK-csproj-Format)
auf **.NET 10** heben und sauber bauen lassen.

---

## 0. Umsetzungsstand (2026-08-09)

Die Migration ist code-seitig umgesetzt und committet (Branch `main`, noch nicht gepusht).
Der C#-Compile beider Projekte ist grün; IRC baut warnungsfrei. Blockiert sind nur der
volle Build zur `LOIC.exe` und der Laufzeittest, siehe unten.

### Erledigt (jeweils eigener Commit)

- IRC auf SDK-Style/`net10.0`, log4net entfernt, AssemblyInfo migriert, feste Version
  0.4.0.0.
- `Thread.Abort()` durch kooperatives Beenden ersetzt: 3x in `IrcConnection.cs`
  (Read-/Write-/IdleWorkerThread) plus eine von der Analyse übersehene 4. Stelle in
  `frmMain.cs` (IRC-Listen-Thread, beendet über `ircenabled` + `Disconnect()`).
- LOIC auf SDK-Style/`net10.0-windows` + `UseWindowsForms`; IRC-Unterbaum aus den
  Default-Globs ausgeschlossen; AssemblyInfo migriert.
- `Process.Start` an 3 Stellen auf `UseShellExecute = true`; `<startup>`-Block aus
  `app.config` entfernt.
- Aufräumarbeiten: redundante PackageReferences entfernt (NU1510); obsolete
  Serialisierungs-Ctoren aus den IRC-Exceptions entfernt (SYSLIB0051); SYSLIB0014 an den
  Overlord-Webaufrufen gezielt per `#pragma` unterdrückt; `LOIC.sln` auf Format 12.00
  angehoben; `LOIC.userprefs` entfernt.
- Alle Code- und Projektdatei-Kommentare auf Englisch (inklusive vorbestehender deutscher
  Designer-Kommentare in `frmEZGrab.Designer.cs`).

### Blockiert: voller Build und Laufzeittest (Windows Defender)

`LOIC.dll` wird bei jedem Build als `HackTool:Win32/Oylecann.A` (Threat-ID 2147641076) in
Quarantäne verschoben, bevor der `CreateAppHost`-Schritt sie zur `LOIC.exe` bündeln kann.
C#-Compile und resx-Ressourcen (inklusive der BinaryFormatter-Bitmaps) bauen bereits
erfolgreich; nur das Erzeugen der lauffähigen EXE scheitert.

- Schritt 5 (voller `dotnet build` grün inkl. `LOIC.exe`) und Schritt 6 (Laufzeittest)
  brauchen eine Defender-Ausnahme (Admin), z. B. für `src/bin` und `src/obj`.

### Noch per Laufzeittest zu verifizieren ("baut grün" deckt das nicht ab)

- resx-Bitmaps (2.6): lädt WinForms die BinaryFormatter-`Bitmap1` zur Laufzeit? (EULA-/
  Main-/EZGrab-/Wtf-Form öffnen).
- ConfigurationManager-Schreibpfad (2.7): persistieren `AcceptEULA`/`KonaniCode` in die
  `LOIC.dll.config`? (`Settings.cs` `OpenExeConfiguration`).
- IRC Thread-Stop (2.4): kooperativer ReadThread-Abbruch an einer echten IRC-Verbindung
  (Hivemind).
- `Process.Start` (2.5): GitHub-Link / `help.chm` / EZGrab-URL öffnen über die Shell.

### Offen/optional

- HttpClient-Migration statt WebRequest/WebClient (SYSLIB0014 ist aktuell nur unterdrückt),
  sinnvoll erst wenn laufzeit-testbar.
- Ein echter Flood-Lauf wird bewusst nicht gegen Fremd-Hosts ausgeführt.

---

## 1. Ausgangslage

| Projekt | Typ | Aktuelles Target | Format | Besonderheiten |
|---|---|---|---|---|
| `src/LOIC.csproj` | `WinExe` (WinForms) | `.NET FW v2.0` | Non-SDK, ToolsVersion 4.0 | 4 Forms + Designer/resx, `app.manifest`, `System.Configuration`, PlatformTarget `x86` |
| `src/IRC/IRC.csproj` | `Library` | `.NET FW v2.0` | Non-SDK, ToolsVersion 4.0 | vendorte SmartIrc4net 0.4.0, `log4net` via `packages.config` |

`LOIC` referenziert `IRC` per ProjectReference. `LOIC.sln` ist Format-Version 11.00
(VS 2010).

**Umgebung ist bereit:** `dotnet --list-sdks` zeigt u. a. `10.0.110` und `10.0.302`.
Ein .NET 10 SDK ist also installiert.

Zielframework-Vorschlag:
- `LOIC` -> `net10.0-windows` mit `<UseWindowsForms>true</UseWindowsForms>` (WinForms gibt es
  auf modernem .NET nur Windows-only).
- `IRC` -> `net10.0` (die Bibliothek nutzt nur `System.Net`-Sockets, Threading und
  nicht-generische Collections, nichts Windows-spezifisches; ein plattformneutrales Target
  ist sauberer. `net10.0-windows` ginge auch.)

---

## 2. Bruchstellen-Katalog

Schweregrad-Legende:
- BLOCKER = baut nicht oder wirft garantiert zur Laufzeit.
- RISIKO = kann brechen, muss verifiziert werden.
- MECHANISCH = reine Formatumstellung, Verhalten unveraendert.
- KOSMETISCH = Warnung oder toter Code, kein funktionaler Effekt.

### 2.1 Projektdatei-Format (beide Projekte) - MECHANISCH

Beide `.csproj` sind im alten Non-SDK-Format mit manuellen `<Compile Include>`-Listen,
`<Reference Include="System...">` und `<Import ...Microsoft.CSharp.targets>`.

Umstellung auf SDK-Style (`<Project Sdk="Microsoft.NET.Sdk">`):
- Die Datei schrumpft massiv, weil SDK-Style alle `*.cs` automatisch globbt.
- Die GAC-Referenzen (`System`, `System.Data`, `System.Drawing`, `System.Windows.Forms`,
  `System.Xml`) entfallen; sie kommen ueber das Framework bzw. `UseWindowsForms`.
- resx/Designer-Zuordnung (DependentUpon) macht das SDK automatisch ueber die
  Dateinamens-Konvention.

### 2.2 packages.config -> PackageReference (IRC) - MECHANISCH

`src/IRC/packages.config` haelt `log4net 2.0.10` (targetFramework `net20`), waehrend der
`HintPath` in der csproj noch auf `log4net.2.0.5\lib\net20-full\log4net.dll` zeigt (also
inkonsistent nach dem Dependabot-Bump). SDK-Style nutzt `<PackageReference>`. Der
Ordner `src/packages/` wird dann nicht mehr gebraucht.

Siehe aber 2.3: die Referenz ist im aktuellen Build faktisch ungenutzt.

### 2.3 log4net ist eine tote Referenz - KOSMETISCH (aber wichtig zu wissen)

Der komplette log4net-Code in der IRC-Bibliothek steckt hinter `#if LOG4NET`
(117 Vorkommen ueber 9 Dateien, u. a. die gesamte `Logger.cs`). **Das Symbol `LOG4NET`
wird in keiner der beiden csproj definiert** (dort steht nur `DEBUG;TRACE` bzw. `TRACE`).

Folge: log4net wird zwar als Assembly referenziert, aber kein einziger Aufruf wird
kompiliert. Die Abhaengigkeit ist im aktuellen Build wirkungslos.

Optionen fuer die Migration:
- (a) `log4net` als PackageReference beibehalten, `LOG4NET` weiterhin **nicht** definieren.
  Verhalten bleibt exakt wie heute (Logging aus). Einfachste, verlustfreie Variante.
- (b) `log4net` und `packages.config` ganz entfernen. Minimaler Dependency-Fussabdruck.
  Wenn spaeter Logging gewuenscht wird, muesste man es wieder hinzufuegen.
- (c) `LOG4NET` bewusst definieren und Logging aktivieren. Groesserer Test- und
  Konfigurationsaufwand (log4net-Config fehlt aktuell, siehe auskommentierter Block in
  `Logger.cs`). Nicht empfohlen im Rahmen einer reinen Framework-Migration.

Empfehlung: (a) fuer maximale Verhaltensgleichheit, alternativ (b) fuer Sauberkeit.

### 2.4 Thread.Abort() (IRC) - BLOCKER

`src/IRC/Connection/IrcConnection.cs`, 3 Stellen: Zeile 812, 900, 1120. Jeweils in der
`Stop()`-Methode dreier interner Worker-Klassen:

- `ReadThread.Stop()` (812): ruft `_Thread.Abort()`, danach `_Connection._Reader.Close()`.
  Der Worker (`_Worker`, ab 819) laeuft `while (_Connection.IsConnected && ...)` und blockiert
  im `ReadLine()` auf dem Socket.
- `WriteThread.Stop()` (900): ruft `_Thread.Abort()`. Worker (ab 907) laeuft
  `while (_Connection.IsConnected)` mit `Thread.Sleep(_SendDelay)` und leert die Sendepuffer.
- `IdleWorkerThread.Stop()` (1120): ruft `_Thread.Abort()`. Worker (ab 1123) laeuft
  `while (_Connection.IsConnected)` mit `Thread.Sleep(_IdleWorkerInterval)`.

`Thread.Abort()` existiert auf modernem .NET nur noch als Stub und wirft zur Laufzeit
`PlatformNotSupportedException`. Das ist der zentrale Code-Blocker der Migration.

Gute Nachricht: alle drei Worker pollen bereits `_Connection.IsConnected`, und der Disconnect-
Pfad setzt `_IsConnected = false` (Zeile 472/548), bevor die `Stop()`-Methoden laufen.
Der harte Abbruch war also historisch nur die Brechstange.

Geplanter Ersatz (kooperatives Beenden):
- `Thread.Abort()` entfernen.
- WriteThread/IdleWorkerThread: beenden sich durch das bereits vorhandene
  `IsConnected`-Polling nach dem naechsten `Sleep`-Intervall von selbst. Kein weiterer Eingriff
  noetig, ausser optional `_Thread.Join(timeout)` fuer sauberes Warten.
- ReadThread: blockiert im `ReadLine()`. Das vorhandene `_Reader.Close()` unterbricht den
  blockierenden Read (wirft dort eine Exception, die der Worker faengt/beendet). Reihenfolge:
  erst Reader schliessen (weckt den Read), dann optional `Join(timeout)`.
- Da alle Threads `IsBackground = true` sind, verhindert ein haengender Rest-Thread ohnehin
  nicht den Prozess-Exit.

RISIKO-Hinweis: Das ist eine kleine, aber verhaltensrelevante Aenderung an der Netzwerk-
Bibliothek. Sollte mit einer echten IRC-Verbindung (Hivemind-Modus) getestet werden, nicht nur
"baut gruen".

### 2.5 Process.Start ohne UseShellExecute - BLOCKER (Laufzeit)

3 Stellen, die eine URL bzw. Datei ueber die Shell oeffnen wollen:
- `frmEZGrab.cs:46` -> `System.Diagnostics.Process.Start(turl)` (URL oeffnen)
- `frmMain.cs:1068` -> `Process.Start("https://github.com/NewEraCracker/LOIC")`
- `frmMain.cs:1562` -> `Process.Start("help.chm")`

Auf .NET Framework war `UseShellExecute` implizit `true`, deshalb funktionierte
`Process.Start("https://...")`. Auf modernem .NET ist der Default `false`, wodurch diese
Aufrufe eine `Win32Exception` werfen ("Die angegebene Datei wurde nicht gefunden"), da der
String nicht als ausfuehrbares Programm interpretiert wird.

Fix pro Stelle:
```csharp
Process.Start(new ProcessStartInfo(url) { UseShellExecute = true });
```

### 2.6 resx mit BinaryFormatter-Nutzdaten - RISIKO (potenziell BLOCKER)

**Alle 5 resx-Dateien** enthalten ein Objekt `Bitmap1` mit
`mimetype="application/x-microsoft.net.object.binary.base64"`, also mit BinaryFormatter
serialisiert:
- `frmEULA.resx:22`, `frmEZGrab.resx:22`, `frmMain.resx:22`, `frmWtf.resx:22`,
  `Properties/Resources.resx:22`

BinaryFormatter ist auf .NET 9/10 standardmaessig deaktiviert bzw. entfernt. Damit sind zwei
Ebenen betroffen:
1. Build: der `GenerateResource`-Task muss ggf. mit
   `<GenerateResourceUsePreserializedResources>true</GenerateResourceUsePreserializedResources>`
   und PackageReference `System.Resources.Extensions` gebaut werden.
2. Laufzeit: WinForms auf .NET bringt einen begrenzten Kompat-Leser fuer bekannte Typen
   (u. a. `System.Drawing.Bitmap`) mit, der den Legacy-Binaerblock ohne allgemeinen
   BinaryFormatter lesen kann. Ob das fuer genau diese `Bitmap1`-Daten greift, ist
   versionsabhaengig und **muss getestet werden**.

Jede resx hat daneben ein `Icon1` als `bytearray.base64` (unkritisch, moderner Weg).

Loesungsoptionen (nach Aufwand):
- (a) Zuerst mit `System.Resources.Extensions` +
  `GenerateResourceUsePreserializedResources` bauen und Laufzeit testen. Wenn die Bitmaps
  korrekt laden, keine weitere Aenderung noetig.
- (b) Falls das bricht: die betroffenen `Bitmap1`-Ressourcen einmal im modernen Format
  neu speichern (ueber den WinForms-Designer oder ein kleines Konvertierungsskript
  base64-binary -> PNG/bytearray). Danach ist kein BinaryFormatter-Pfad mehr noetig.

Erst pruefen, welche Rolle `Bitmap1` in der jeweiligen Form spielt (Designer.cs referenziert es),
bevor man Aufwand in (b) steckt.

### 2.7 System.Configuration / ConfigurationManager (LOIC) - MECHANISCH

`src/Settings.cs` nutzt `ConfigurationManager.AppSettings`,
`ConfigurationManager.OpenExeConfiguration(...)` und `RefreshSection(...)` zum Lesen/Schreiben
von `AcceptEULA` und `KonaniCode` in die App-Config.

Auf .NET ist das im NuGet-Paket `System.Configuration.ConfigurationManager` ausgelagert.
PackageReference hinzufuegen, dann kompiliert `Settings.cs` unveraendert.

RISIKO-Hinweis: der Schreibpfad (`OpenExeConfiguration` schreibt in
`<App>.exe.config` neben der EXE) sollte kurz verifiziert werden; die Dateinamens-Logik
unterscheidet sich zwischen Framework und modernem .NET leicht.

### 2.8 app.config startup-Block - KOSMETISCH

`src/app.config` enthaelt nur:
```xml
<startup useLegacyV2RuntimeActivationPolicy="true">
    <supportedRuntime version="v2.0.50727"/>
    <supportedRuntime version="v4.0"/>
</startup>
```
Das ist reine Framework-Runtime-Aktivierung und wird von modernem .NET ignoriert. Harmlos,
sollte aber entfernt werden. Achtung: es gibt aktuell **keinen** `appSettings`-Abschnitt;
`Settings.cs` legt ihn zur Laufzeit an. Beim Aufraeumen der app.config nichts entfernen, was
`Settings.cs` braucht (aktuell nichts, aber im Blick behalten).

### 2.9 AssemblyInfo-Doppelung und Wildcard-Version - BLOCKER (Build)

- SDK-Style generiert die Assembly-Attribute automatisch. Die bestehenden
  `Properties/AssemblyInfo.cs` (beide Projekte) enthalten `AssemblyTitle`, `AssemblyVersion`
  usw. -> **Fehler "duplicate attribute"**. Loesung: entweder
  `<GenerateAssemblyInfo>false</GenerateAssemblyInfo>` setzen und die alten Dateien behalten,
  oder die Attribute in die csproj-Properties migrieren und die alten Dateien loeschen.
- `src/IRC/Properties/AssemblyInfo.cs` hat `[assembly: AssemblyVersion("0.4.0.*")]`.
  Die `*`-Wildcard ist bei deterministischen Builds (Default im .NET SDK) **nicht erlaubt**
  und bricht den Build. Loesung: feste Version (z. B. `0.4.0.0`) oder `<Deterministic>false</Deterministic>`.
  Empfehlung: feste Version.
- `#if DELAY_SIGN` in der IRC-AssemblyInfo ist nicht definiert, also inaktiv. Kein Problem.

### 2.10 WebClient obsolet (LOIC) - KOSMETISCH

`frmMain.cs:1385` nutzt `new WebClient()`. Existiert in .NET 10 weiter, ist aber als
`SYSLIB0014` obsolet markiert (Warnung). `HttpWebRequest`/`WebRequest` (`frmMain.cs:1353/1355`)
funktionieren ebenfalls weiter. Kein Blocker. Modernisierung auf `HttpClient` waere optional
(gehoert nicht in den Minimal-Lift).

### 2.11 PlatformTarget x86 - ENTSCHEIDUNG

Beide Projekte setzen `PlatformTarget x86`. Auf .NET 10 kann das bleiben (`<PlatformTarget>x86</PlatformTarget>`),
oder man wechselt auf `AnyCPU`/`x64`. x86 ist heute unueblich; ein DDoS-Netzwerktool profitiert
nicht davon. Vorschlag: `AnyCPU`, sofern kein Grund fuer x86 bekannt ist.

---

## 3. Was unkritisch ist und einfach durchlaeuft

- **Sockets/Netzwerk in den Floodern**: `HTTPFlooder.cs`, `XXPFlooder.cs`, `cHLDos.cs`,
  `Protocol.cs`, `Functions.cs` sowie `IrcTcpClient.cs` nutzen `Socket`, `TcpClient`,
  `NetworkStream`, `Dns`, `IPEndPoint`, `Encoding`, `StreamReader/Writer`. Alles in .NET 10
  vorhanden und API-kompatibel.
- **Nicht-generische Collections** (`SortedList`, `Queue`, `Hashtable`) in der IRC-Lib: alle
  weiter vorhanden.
- **Program.cs**: `[STAThread]`, `Application.SetCompatibleTextRenderingDefault(false)`,
  `Application.Run(...)` - alles WinForms-Standard, laeuft. (Optional koennte man
  `ApplicationConfiguration.Initialize()` nutzen, noetig ist es nicht.)
- **app.manifest**: `requestedExecutionLevel asInvoker` + supportedOS-Liste. Wird von
  `<ApplicationManifest>` weiter unterstuetzt, kann unveraendert bleiben.
- **Keine P/Invoke / Marshal / Registry / Microsoft.VisualBasic / unsafe / ServicePointManager /
  Tls-Fummelei** im Code gefunden. `Konami.cs` ist rein managed.
- **Icon/EmbeddedResource** (`Resources\LOIC.ico`, EULA.rtf, WTF.jpg, LOIC.gif): unkritisch.

---

## 4. Geplanter Diff (Uebersicht pro Datei)

Neu/umgeschrieben:
- `src/IRC/IRC.csproj` -> SDK-Style, `net10.0`, PackageReference `log4net 2.0.x`
  (oder entfernen, siehe 2.3), feste AssemblyVersion, `GenerateAssemblyInfo` geklaert.
- `src/IRC/packages.config` -> **loeschen**.
- `src/LOIC.csproj` -> SDK-Style, `net10.0-windows`, `UseWindowsForms=true`,
  PackageReference `System.Configuration.ConfigurationManager`, ggf.
  `System.Resources.Extensions` + `GenerateResourceUsePreserializedResources` (siehe 2.6),
  `ApplicationManifest`/`ApplicationIcon` beibehalten.
- `src/LOIC.sln` -> auf aktuelles Format heben (optional, VS/dotnet aktualisieren es beim
  ersten Speichern; nicht zwingend).

Code-Aenderungen (klein, gezielt):
- `src/IRC/Connection/IrcConnection.cs` -> 3x `Thread.Abort()` durch kooperatives Beenden
  ersetzen (2.4).
- `src/frmEZGrab.cs:46`, `src/frmMain.cs:1068`, `src/frmMain.cs:1562` -> `Process.Start` mit
  `ProcessStartInfo { UseShellExecute = true }` (2.5).

Config/Ressourcen:
- `src/app.config` -> `<startup>`-Block entfernen (2.8).
- 5x `Bitmap1`-Ressource: nur anfassen, falls Laufzeittest fehlschlaegt (2.6).

AssemblyInfo:
- `src/Properties/AssemblyInfo.cs` und `src/IRC/Properties/AssemblyInfo.cs` -> entweder
  behalten mit `GenerateAssemblyInfo=false`, oder in csproj migrieren und loeschen;
  Wildcard-Version fixen (2.9).

Aufraeumen:
- `src/packages/` -> entfernen (nach PackageReference-Umstellung ueberfluessig).

---

## 5. Empfohlene Reihenfolge

1. `IRC.csproj` auf SDK-Style + `net10.0`, log4net-Frage entscheiden, AssemblyVersion fixen.
2. `Thread.Abort()` in `IrcConnection.cs` ersetzen. `dotnet build src/IRC` gruen bekommen.
3. `LOIC.csproj` auf SDK-Style + `net10.0-windows` + `UseWindowsForms`, Config-Paket,
   resx-Strategie waehlen.
4. `Process.Start`-Stellen fixen. `app.config` aufraeumen.
5. `dotnet build` der ganzen Solution gruen bekommen, Warnungen sichten.
6. **Laufzeittest**: Start der EXE, EULA-Dialog (resx-Bitmaps!), ein Flood-Lauf,
   optional Hivemind/IRC (Thread-Stop-Pfad).

---

## 6. Offene Entscheidungen (fuer dich)

1. **log4net**: beibehalten (2.3a) oder entfernen (2.3b)?
2. **resx-Bitmaps**: erst testen und nur bei Bruch neu speichern (2.6a) - vermutlich okay so.
3. **PlatformTarget**: `AnyCPU` statt `x86` (2.11)?
4. **AssemblyInfo**: alte Dateien behalten (`GenerateAssemblyInfo=false`) oder in csproj
   migrieren?
5. **IRC-Target**: `net10.0` (empfohlen, portabel) oder `net10.0-windows`?
6. **.sln-Format** aktualisieren ja/nein?

---

## 7. Risiko-Zusammenfassung

| Punkt | Schweregrad | Sicher automatisierbar? |
|---|---|---|
| csproj SDK-Style (beide) | mechanisch | ja |
| packages.config -> PackageReference | mechanisch | ja |
| log4net toter Code | kosmetisch | ja |
| Thread.Abort (3x) | BLOCKER | ja, mit anschliessendem IRC-Test |
| Process.Start (3x) | BLOCKER (Laufzeit) | ja |
| resx BinaryFormatter (5x) | RISIKO | teilweise, Laufzeittest noetig |
| ConfigurationManager-Paket | mechanisch | ja |
| app.config startup | kosmetisch | ja |
| AssemblyInfo/Wildcard | BLOCKER (Build) | ja |
| WebClient obsolet | kosmetisch (Warnung) | ja |
| PlatformTarget x86 | Entscheidung | ja |

Fazit: Technisch gut machbar. Die einzigen echten Denkarbeiten sind `Thread.Abort` (klein,
aber testbeduerftig) und die BinaryFormatter-Bitmaps in den resx (mit hoher Wahrscheinlichkeit
unproblematisch, aber verifizieren). Der Rest ist Formatumstellung und zwei kleine
Code-Fixes.

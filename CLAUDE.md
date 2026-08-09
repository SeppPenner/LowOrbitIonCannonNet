# Projektregeln für Claude

## Commits

- Commit-Messages werden **ausschließlich auf Englisch** geschrieben.
- Kurze, präzise Zusammenfassung in der Betreffzeile, bei Bedarf ein erklärender Body.

## Zeichensetzung

- **Keine Geviert- oder Halbgeviertstriche** (Em-Dash `—`, En-Dash `–`) verwenden, weder in
  Texten, Commit-Messages, Code-Kommentaren noch in Dokumentation.
- Stattdessen normalen Bindestrich, Komma, Doppelpunkt, Klammern oder einen eigenen Satz nutzen.

## Code-Kommentare

- Kommentare im Code (und in Projektdateien wie `.csproj`) werden **immer auf Englisch**
  geschrieben, unabhängig von der Sprache der übrigen Kommunikation.

## Deutsche Texte

- In deutschsprachigen Texten (Dokumentation, Chat-Antworten) immer **echte Umlaute
  und ß** verwenden, keine ASCII-Umschreibungen.
- Wo nötig umschreiben:
  - `ae` -> `ä`
  - `oe` -> `ö`
  - `ue` -> `ü`
  - `Ae` -> `Ä`, `Oe` -> `Ö`, `Ue` -> `Ü`
  - `ss` -> `ß` (nur wo orthografisch korrekt, z. B. `Strasse` -> `Straße`; `dass` bleibt `dass`)
- Das gilt für Dokumentationsdateien und Chat, **nicht** für Code-Kommentare (die sind Englisch,
  siehe oben).
- Ausnahme: Bezeichner, Dateinamen, Konfigurationsschlüssel und Ähnliches bleiben unverändert,
  wenn Umlaute dort technisch nicht erwünscht sind.

# Quellcodeprüfung VisionRPLauncher – Nachtrag

**Prüfdatum:** 03.11.2026  
**Methode:** statische Prüfung der drei bereitgestellten C#-Dateien sowie Metadaten/Signatur der genannten Laufzeitdateien. Nichts wurde ausgeführt. Originale wurden nicht bearbeitet.

## Befunde mit konkretem Risiko

### 1. Ticket-Zugriffscode kann durch HTTP-Redirect an fremden Host gelangen – beheben

`SupportClient` wird in `MainWindow.xaml.cs` ohne eigenen Handler angelegt (Zeilen 43–46). Damit gilt standardmäßig `AllowAutoRedirect = true`. Beim Lesen und Beantworten eines Tickets hängt der Launcher den Zugriffscode an den benutzerdefinierten Header `X-Ticket-Token` (Zeilen 1356–1360 und 1411–1419). `SupportClient` wird außerdem für Support-Tickets und Bewerbungen verwendet, die Namen, Kontaktangaben, Nachrichten und bei ausdrücklicher Freigabe das Fehlerprotokoll übertragen.

.NET entfernt beim automatischen Folgen einer Weiterleitung den `Authorization`-Header, aber **keine anderen Header**. Eine HTTPS-Weiterleitung des konfigurierten Supportservers auf einen fremden HTTPS-Host kann deshalb `X-Ticket-Token` mitnehmen. Bei Weiterleitungsstatuscodes, die den Request-Body erhalten, können auch übertragene Formulardaten betroffen sein. HTTPS-zu-HTTP-Weiterleitungen werden in .NET 5+ nicht automatisch befolgt; das verhindert jedoch keine HTTPS-zu-HTTPS-Weiterleitung. [Microsoft-Dokumentation zu `AllowAutoRedirect`](https://learn.microsoft.com/dotnet/api/system.net.http.httpclienthandler.allowautoredirect)

**Patch:** [VisionRPLauncher-Support-Redirect-Fix.patch](VisionRPLauncher-Support-Redirect-Fix.patch) deaktiviert automatische Redirects für `SupportClient`. Die bestehenden Erfolgsprüfungen lehnen 3xx-Antworten danach ab. Der Patch betrifft nur die Arbeitskopie; die Quelldatei auf dem Desktop blieb unverändert. Ich habe ihn nicht kompiliert, da die Projektdatei und zugehörige XAML-Dateien fehlen.

### 2. Updater ersetzt nur den nativen Apphost, nicht die Launcher-Logik – Updateweg ist unvollständig

`LauncherUpdateService.cs` akzeptiert als Asset nur eine URL mit `/VisionRPLauncher.exe` (Zeilen 226–238), lädt ausschließlich dieses Asset und prüft seinen Hash gegen das signierte Manifest. `MainWindow.xaml.cs` übergibt `Environment.ProcessPath` als einziges Ziel an den PowerShell-Helfer; dieser ersetzt die EXE (Zeilen 604–605, 631–647, 765–799).

Der zuvor geprüfte Release-Build zeigt dagegen eine native x64-Apphost-EXE ohne CLR-Metadaten und eine separate verwaltete `VisionRPLauncher.dll` mit dem Anwendungscode. Die bereitgestellte `.deps.json`, `.runtimeconfig.json` und Runtime-DLLs liegen ebenfalls separat. Der aktuelle Austausch der EXE aktualisiert diese Dateien nicht; damit wird die eigentliche Launcher-Logik nicht auf die Manifestversion gebracht. Die App liest ihre aktuelle Version aus der verwalteten Assembly (Zeile 529), sodass das Update weiterhin angeboten werden kann.

**Erforderliche Korrektur:** entweder einen nachweislich vollständigen Single-File-Publish als Update-Artefakt erstellen und genau dieses verifizieren/ersetzen, oder ein signiertes Paket mit allen benötigten Dateien und Hashes pro Datei veröffentlichen. Für ein mehrteiliges Paket sollte die neue Version zunächst in einem neuen Verzeichnis vollständig geprüft und anschließend als Ganzes aktiviert werden; einzelnes Ersetzen mehrerer Dateien ist nicht atomar.

### 3. PowerShell-Updatehelfer mit ExecutionPolicy-Umgehung – Härtung empfohlen

Der Launcher schreibt ein Skript nach `%TEMP%` und startet Windows PowerShell mit `-ExecutionPolicy Bypass` (Zeilen 649–680; Skript in Zeilen 765–815). Die Prozessargumente werden mit `ArgumentList` getrennt übergeben und Dateizugriffe im Skript verwenden weitgehend `-LiteralPath`, was Shell-/Pfad-Injection erschwert. Dennoch umgeht `Bypass` die lokale PowerShell-Ausführungsrichtlinie; das temporäre Skript ist kein separat signiertes Releaseartefakt. Wenn der Launcher erhöht gestartet wurde, läuft der Helfer mit derselben Berechtigung.

**Empfehlung:** einen kleinen, separat signierten Updater verwenden, der dieselben Update-Hashes/Signaturen prüft und ohne dynamisches Skript im Temp-Verzeichnis arbeitet. Falls PowerShell vorerst bleibt, auf `ExecutionPolicy Bypass` verzichten, das Hilfsskript signieren, die Temp-Datei mit restriktiven ACLs exklusiv anlegen und Pfade/Reparse-Points vor dem Austausch prüfen.

## Positive Kontrollen

- Die Update-Manifestprüfung nutzt einen fest eingebetteten RSA-Public-Key, RSA-PSS mit SHA-256 und prüft zusätzlich den SHA-256-Hash des Downloads.
- Produktkennung, dreiteiliges Versionsformat und GitHub-Release-Pfad des EXE-Assets werden geprüft; der Installer zeigt Release-Notizen vor der Installation an.
- Die Versionslogik bietet nur ein Update an, wenn `LatestVersion > CurrentVersion`; ein direktes Downgrade wird damit verhindert.
- Support- und Statusfeed-URLs erlauben HTTPS; HTTP ist auf Loopback begrenzt. URL-Benutzerinfo, Query und Fragment werden ausgeschlossen.
- Der Statusfeed wird auf 64 KB begrenzt, und die lokale Konfiguration speichert den Ticket-Zugriffscode nicht. Der Zugriffscode wird im UI angezeigt und bleibt während der Sitzung in den Eingabefeldern.
- `LauncherLogger` schreibt Fehler lokal in `%LOCALAPPDATA%`; der Support-Logversand erfordert eine separate Vorschau und Einwilligung. Die Logdatei hat allerdings keine Größenbegrenzung oder Rotation.
- Die Dateien im `win-x64\cs`-Verzeichnis sind nach Dateinamen und Ordnerstruktur WPF-/Windows-Forms-Ressourcenassemblies für die Kultur `cs`, keine erkennbaren Launcher-Codekomponenten. `System.Xaml.dll` hat eine gültige Authenticode-Signatur mit Herausgeber `.NET, Microsoft Corporation`.

## Weitere Härtung

- Support- und Serverstatus-Antworten werden per `ReadFromJsonAsync`/`JsonDocument.ParseAsync` eingelesen, ohne für diese Antworten eine Obergrenze im Anwendungscode zu setzen. Bounded-Read-Limits sollten auch dort gelten, damit ein konfigurierter oder kompromittierter Endpoint nicht über große JSON-Antworten den Speicherverbrauch hochtreibt.
- Der Logger hängt vollständige Exception-Details unbegrenzt an. Größe/Alter begrenzen und vor dem Schreiben sensible Werte, URLs mit Zugangsdaten und Ticketkennungen redigieren.
- Die PDB aus dem vorherigen Upload enthielt den Desktop-Projektpfad und Quelldateinamen. Release-PDBs aus öffentlichen Paketen entfernen oder über einen geschützten Symbolserver veröffentlichen.

## Geprüfte Prüfsummen

| Datei | SHA-256 |
|---|---|
| `LauncherUpdateService.cs` | `58CF54867F9006EDF901CBF684A4A5D8E865115529EBBBCFE4016BF2A3236D8B` |
| `LauncherLogger.cs` | `3D6ACBAB9A71ADA206460FA1534CAAF609A1844EBD7E2B6E8C0E8AA67D6781DD` |
| `MainWindow.xaml.cs` | `74763650E03E757BEAF1A8628D40963146EF7209EC98EB0D8589563B45828ECB` |
| `System.Xaml.dll` | `614FD9EBB6C12A2D7F52ABCDC2E63EC5FC277793D417C5A8D5A8BD017376BEFD` |

## Noch erforderlich für einen gehärteten Neubau

Es fehlen weiterhin `*.csproj`, `App.xaml`/weitere XAML-Dateien, übrige Quellcodedateien sowie Release-/Publish-Einstellungen. Bitte lade den vollständigen Projektordner als ZIP hoch. Private Signaturschlüssel und Admin-Tokens bitte weglassen. Für die Korrektur des Updates muss der Publish-Modus bekannt sein, damit eindeutig feststeht, ob das Release ein Single-File-Artefakt oder ein mehrteiliges Paket sein soll.

# VisionRP Launcher: Härtungsstand

## Umfang und Schutz der Originale

Die Änderungen wurden ausschließlich an einer Arbeitskopie unter `work/visionrp-hardened-source` vorgenommen. Die Originale auf dem Desktop wurden nicht verändert. Dateien mit privaten Schlüsseln, Zertifikaten und `.env` wurden nicht in die Kopie übernommen oder ausgelesen. Kein Launcher- oder Serverprogramm wurde ausgeführt.

Der gewünschte Release-Stand ist **3.1.0**. Die Notizen bleiben leer; die Publisher-Schnittstelle erlaubt das. Die Arbeitskopie enthält ein Inno Setup Skript für die Installation pro Benutzer ohne Adminrechte.

## Umgesetzte Änderungen

- **Launcher-Updates:** Manifest-URLs müssen HTTPS auf dem Standardport und auf den erwarteten GitHub-Release-Tag beschränkt sein. Die Support-HTTP-Verbindung folgt Weiterleitungen nicht mehr; dadurch wird der Ticket-Zugriffscode nicht an ein fremdes Redirect-Ziel weitergereicht. Status- und Supportantworten werden größenbegrenzt eingelesen.
- **Updater-Prozess:** Der temporäre PowerShell-Skriptdatei-Mechanismus und `ExecutionPolicy Bypass` wurden durch einen PowerShell-Aufruf mit `-EncodedCommand` und kurzlebigen Prozessumgebungswerten ersetzt.
- **Logging:** Einträge werden begrenzt, Logs rotieren bei 5 MB; Diagnose-Fallbacks schreiben keine vollständigen Ausnahmedetails.
- **Supportserver:** Eingaben sind auf 64 KiB begrenzt; Ticket-Lesezugriffe und Admin-API erhalten IP-basierte Ratenlimits. Ticket-IDs verwenden 128 Bit Zufall. Produktionsbetrieb verlangt HTTPS; Sicherheitsheader, restriktive CSP und `no-store` sind gesetzt. Admin-JavaScript und CSS sind extern, die Browser-API untersagt Redirects, Cache und Cross-Origin-Zugriffe.
- **Lokaler Serverstart:** Das Startskript prüft die Mindestlänge des Admin-Tokens, bindet lokal und stellt die zuvor gesetzten Prozessvariablen nach dem Lauf wieder her.
- **Publisher:** Der Publisher akzeptiert nur einen verschlüsselten PEM-Schlüssel über `VISIONRP_UPDATE_SIGNING_KEY_PATH`, fragt die Passphrase verdeckt ab, verlangt mindestens 3072 Bit und verifiziert den Schlüssel gegen den im Launcher eingebetteten öffentlichen Schlüssel. `migrate-key` konvertiert einen Klartextschlüssel lokal, ohne die Quelldatei zu löschen. Manifest-Schreiben erfolgt atomar. Ein Publish-Ordner mit losen DLL/PDB/Runtime-Konfigurationsdateien wird abgewiesen.
- **Single-File-Publish:** Launcher-Projekt und README setzen `IncludeNativeLibrariesForSelfExtract=true` und `DebugSymbols=false`, damit die einzelne EXE keine danebenliegenden nativen Runtime-Dateien benötigt und keine PDB mitveröffentlicht wird.
- **Support-Abhängigkeiten:** `Microsoft.AspNetCore.OpenApi` wurde auf 10.0.14 und `Microsoft.OpenApi` auf 2.13.0 angehoben. NuGet-Lockdatei und Audit sind aktiviert; Warnungen für hohe/kritische Advisories werden zu Fehlern.

## Buildstatus und Einschränkungen

Der Publisher-Build war erfolgreich (0 Warnungen, 0 Fehler). Der Launcher wurde danach als einzelne selbständige EXE veröffentlicht; deren Dateimetadaten zeigen Version `3.1.0.0`. Der veröffentlichte Ordner enthält nur `VisionRPLauncher.exe`. Das Inno-Setup für `v3.1.0` wurde mit Inno Setup 7.1.1 erfolgreich kompiliert. Die finale Setup-Datei liegt als `VisionRPLauncher-Setup-3.1.0.exe` im Ausgabeverzeichnis; SHA-256: `8FB991189B4DA1919FA45302727C1FF38AD439FD2944DE74AD76AE18704F4CD5`.

Der Supportserver-Build war wegen fehlender Verbindung zu `api.nuget.org` nicht erfolgreich. Launcher und Installer wurden nicht gestartet oder auf einem Testsystem installiert. Das Setup trägt aktuell keine Authenticode-Signatur. Es wurde kein signiertes Update-Manifest erzeugt, da der private Schlüssel absichtlich nicht bereitgestellt oder verwendet wurde. Das Setup kann daher lokal zur Installation geprüft werden, ist aber noch kein vollständig signierter öffentlicher Release mit funktionierenden automatischen Updates.

## Noch erforderliche Betriebsmaßnahmen

- Vor öffentlichem Betrieb Support hinter einen korrekt konfigurierten HTTPS-Reverse-Proxy stellen und ausschließlich dessen IPs unter `Support:TrustedProxies` eintragen. `.env` und Admin-Token aus einem Secret-Store beziehen.
- Für Ticketspeicherung Aufbewahrungsfrist, Backups, Zugriffsschutz des persistenten Speichers und Löschprozess festlegen. Die JSON-Datei wird weiterhin nicht automatisch verschlüsselt oder bereinigt.
- Den Update-Signierschlüssel offline oder in einem Hardware-Signierdienst halten. Falls Klartextschlüssel jemals in Repository, Logs, Cloud-Backup oder fremde Hände gelangt sein könnte, als kompromittiert behandeln und Schlüsselrotation samt neuem Launcher planen.
- Einen sauberen Single-File-Publish bauen, Inhalt des Ausgabeverzeichnisses prüfen, EXE signieren, manifestieren und auf einem Testsystem installieren, bevor ein öffentliches Release erfolgt.

## Geänderte Dateien

Die relevante Arbeitskopie befindet sich unter `work/visionrp-hardened-source`. Besonders wichtige Dateien sind `MainWindow.xaml.cs`, `LauncherUpdateService.cs`, `LauncherLogger.cs`, `VisionRPLauncher.csproj`, `Tools/UpdatePublisher/Program.cs`, `SupportServer/Program.cs`, `SupportServer/Endpoints/TicketEndpoints.cs`, `SupportServer/Services/TicketService.cs`, `SupportServer/SupportServer.csproj` und `README.md`.

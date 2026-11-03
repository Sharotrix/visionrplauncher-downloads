# Projektdateien – statischer Prüfbericht

**Prüfdatum:** 03.11.2026  
**Methode:** statische Prüfung der drei `.csproj`-Dateien und Abgleich der festgelegten NuGet-Versionen mit den offiziellen NuGet-/Projekt-Sicherheitsseiten. Kein Restore, Build oder Test wurde ausgeführt. Die Originaldateien blieben unverändert.

## Wichtigster Befund: verwundbare Microsoft.OpenApi-Version festgelegt

`SupportServer.csproj` pinnt `Microsoft.OpenApi` auf **2.7.5**. Der offizielle OpenAPI.NET-Sicherheitsbericht GHSA-m2pf-rgqv-ggvv führt die gesamte 2.x-Reihe **unter 2.12.2** als betroffen auf. Beschrieben werden Denial-of-Service-Pfade durch zyklische Referenzen bzw. Ressourcenerschöpfung bei YAML-Verarbeitung. Das kann laut Advisory einen Prozess beenden, wenn er betroffene OpenAPI-Inhalte verarbeitet. Aus der Projektdatei allein ist nicht erkennbar, ob der Supportserver solche nicht vertrauenswürdigen Dokumente verarbeitet; die Betroffenheit der festgelegten Paketversion ist aber belegt. [Sicherheits-Advisory](https://github.com/microsoft/OpenAPI.NET/security/advisories/GHSA-m2pf-rgqv-ggvv)

Ich empfehle das Paket auf mindestens die gepatchte Version **2.12.2** anzuheben. Zum Prüfdatum ist **2.13.0** als stabile 2.x-Version gelistet. `Microsoft.AspNetCore.OpenApi` ist auf **10.0.0** festgesetzt; NuGet listet für .NET 10 inzwischen **10.0.14** als aktuelle stabile Wartungsversion. [NuGet: Microsoft.OpenApi 2.7.5 und Versionen](https://www.nuget.org/packages/Microsoft.OpenApi/2.7.5), [NuGet: Microsoft.AspNetCore.OpenApi 10.0.0 und Versionen](https://www.nuget.org/packages/Microsoft.AspNetCore.OpenApi/10.0.0)

Der vorbereitete [Paketupdate-Patch](SupportServer-Paketupdate.patch) hebt beide Paketversionen an. Er ändert keine Originaldatei. Der Wechsel muss anschließend im vollständigen Projekt wiederhergestellt und gebaut werden; API-/Verhaltenskompatibilität wurde hier nicht getestet.

## Launcher-Publish-Konfiguration

`VisionRPLauncher.csproj` zielt auf `net10.0-windows10.0.22000.0`, aktiviert WPF und Windows Forms und setzt Assemblyname/Version. Es enthält keine `PublishSingleFile`, `SelfContained` oder `RuntimeIdentifier`-Einstellung. Zusammen mit dem untersuchten Release-Verzeichnis (separate Apphost-EXE, Launcher-DLL, `.deps.json`, `.runtimeconfig.json` und weitere DLLs) bestätigt das: Es ist kein vollständiger Single-File-Build.

Das passt zum vorherigen Codebefund: Der Updater lädt und ersetzt nur `VisionRPLauncher.exe`; damit wird der verwaltete Anwendungscode in `VisionRPLauncher.dll` nicht aktualisiert. Der Updater braucht entweder ein vollständiges Single-File-Artefakt oder ein signiertes Mehrdateienpaket mit vollständiger, atomar aktivierter Versionsablage.

## Tooling und Prüfgrenzen

`UpdatePublisher.csproj` ist eine einfache `net10.0`-Konsolenanwendung ohne externe Paketverweise. Ihre Sicherheit hängt jedoch vom Publisher-Quellcode ab: Signaturformat, private Schlüssel, Release-Manifeste und Dateipaketierung lassen sich aus der Projektdatei nicht beurteilen.

`SupportServer.csproj` verwendet `Microsoft.AspNetCore.OpenApi` und `Microsoft.OpenApi`. Die Paketdatei verrät nicht, welche Routen offen sind, wie Ticket-Zugriffscodes generiert/geprüft werden, ob Rate Limits/Größenlimits greifen oder wie Daten geschützt und aufbewahrt werden. Dafür brauche ich `SupportServer/Program.cs`, die Endpoint-/Service-/Datenmodell-Dateien und gegebenenfalls Deployment-Konfiguration ohne Geheimnisse.

Aus den drei Projektdateien allein sind weder reproduzierbarer Build noch zentrale Paketverwaltung, Lockfiles, Signierung und NuGet-Audit-Einstellungen belegt. Die SDK-Version ist ebenfalls nicht fixiert; ein `global.json` wurde nicht mitgeliefert.

## Geprüfte Prüfsummen

| Datei | SHA-256 |
|---|---|
| `VisionRPLauncher.csproj` | `55CDA83C23C3ECBF40A239FD6CAB941BE6684A61E77A3290D0FC7FEB4C91CBC4` |
| `UpdatePublisher.csproj` | `091184F58AF0BEB2F404C01705712E4789955B5154F759AC633CC822BA024F3C` |
| `SupportServer.csproj` | `0BC870B0D440DCD8E3B0CAD7D442B6AD4136DB9C15069511CE392EE8E63603CC` |

# Supportserver und UpdatePublisher – statischer Prüfbericht

**Prüfdatum:** 03.11.2026  
**Methode:** statische Prüfung der bereitgestellten `Program.cs`-Dateien, der Supportserver-Endpoints, DTOs, Middleware und Ticket-Service-Dateien sowie der Publisher-EXE. Keine Datei wurde gestartet. Es wurden weder Restore noch Build oder Tests durchgeführt. Originale blieben unverändert.

## Kritische Release-Schlüsselverwaltung

`UpdatePublisher/Program.cs` liest den privaten RSA-Schlüssel direkt aus `%LOCALAPPDATA%\VisionRPLauncher\update-signing-private.pem` und importiert ihn mit `RSA.ImportFromPem`. Das erwartet hier einen unverschlüsselten privaten PEM-Schlüssel. Wer unter demselben Windows-Benutzerkonto Schadcode ausführen kann, kann diese Datei lesen und anschließend gültige Manifest-Signaturen erzeugen. Der Launcher vertraut dem fest eingebetteten öffentlichen Schlüssel; ein gestohlener privater Schlüssel kann daher zur Verteilung eines manipulierten Launcher-Updates verwendet werden.

**Priorität: hoch für die Release-Sicherheit.** Den privaten Schlüssel aus dem normalen Benutzerprofil entfernen. Bevorzugt offline signieren oder einen Signaturdienst/HSM mit Zugriffskontrolle und Audit verwenden. Falls ein Datei-Schlüssel vorübergehend unvermeidbar ist: dediziertes Release-Konto, restriktive ACLs, verschlüsselte Speicherung und geregelte Rotation; nicht in Quellcode, Paket oder Benutzerprofil-Backup aufnehmen. Der Schlüssel selbst wurde nicht bereitgestellt oder gelesen.

Publisher-CLI nimmt Version, Asset-URL, Binärpfad und Manifestpfad als Argumente. Sie prüft Format und GitHub-Pfad, berechnet SHA-256 und signiert dieselbe RSA-PSS/SHA-256-Payload, die der Launcher verifiziert. Sie prüft aber nicht, dass die angegebene Version zur tatsächlichen Assembly-Version des Artefakts passt, und validiert nicht, dass die Eingabedatei ein vollständiges Launcher-Release ist.

## Supportserver: positive Kontrollen

- Admin-Routen liegen gemeinsam unter `/api/admin/tickets` und verwenden den `AdminAuthorizationFilter`.
- Der Admin-Token muss beim Start mindestens 32 UTF-8-Bytes haben. Der Filter vergleicht SHA-256-Hashes mit `FixedTimeEquals`.
- Ticket-Zugriffscodes werden aus 32 Zufallsbytes erzeugt, im Datensatz nur als SHA-256-Hash gespeichert und konstantzeitlich verglichen.
- Player-Ticket-Erstellung und -Antworten sind auf 8 Schreibvorgänge je 10 Minuten pro erkannter Client-IP begrenzt. Forwarded Headers werden vor dem Rate Limiter verarbeitet; nur konfigurierte Trusted Proxies werden zusätzlich vertraut.
- DTOs begrenzen Namen, Kontakt, Betreff, Kategorie und Nachrichtenlänge.
- OpenAPI wird nur im Development-Modus veröffentlicht.
- Dateiänderungen erfolgen über temporäre Datei und `File.Move(..., overwrite: true)`; ein Prozess-Lock verhindert parallele Schreibvorgänge innerhalb einer Instanz.

## Supportserver: verbleibende Risiken

### Transportverschlüsselung hängt von der Bereitstellung ab

Wenn `PORT` gesetzt ist, bindet der Prozess an `http://0.0.0.0:<PORT>`; der Code richtet selbst kein HTTPS ein. Das ist eine übliche Container-Konfiguration, wenn ein Reverse Proxy/TLS-Terminator davor steht. Wird dieser Port direkt öffentlich erreichbar gemacht, laufen Admin-Bearer-Token, Ticket-Zugriffscodes und Ticketinhalte unverschlüsselt. Deployment muss HTTPS am Eingang erzwingen und den Proxy in `Support:TrustedProxies` korrekt festlegen. Die Bereitstellungskonfiguration wurde nicht mitgeliefert.

### Klartext und unbegrenzte Aufbewahrung der Tickets

`TicketService` speichert Namen, Kontaktangaben, Betreff und den kompletten Nachrichtenverlauf als Klartext in `data/tickets.json` (oder unter konfiguriertem `Support:DataPath`). Es gibt in den geprüften Dateien keine Verschlüsselung im Ruhezustand, Aufbewahrungsfrist, Löschroutine oder Größenobergrenze für die Datei. Der Standardpfad liegt außerhalb von `wwwroot`; ein benutzerdefinierter `DataPath` sollte ebenfalls niemals in einem statisch veröffentlichten Verzeichnis liegen. Dateisystem-ACLs, Backups und Betreiberzugriff müssen geschützt sein.

Die Anwendung lädt die gesamte JSON-Datei pro Operation. Mit wachsender Ticketzahl steigen Speicher-/Laufzeitbedarf; die Admin-Liste liefert außerdem alle passenden Tickets ohne Pagination. Dauerhafte öffentliche Ticket-Erstellung kann die JSON-Datei anwachsen lassen. Datenbank mit Pagination, Retention/Löschung und globalen Quoten wäre robuster.

### Request-Gesamtgröße

Die DTO-Felder besitzen sinnvolle `StringLength`-Limits, aber in den geprüften Dateien ist kein explizites maximales HTTP-Request-Body-Limit konfiguriert. Feldvalidierung findet erst nach dem Einlesen/Deserialisieren statt. Ergänze eine kleine Obergrenze (z. B. 32–64 KB passend zu den Feldern) auf Server-/Endpoint-Ebene, damit große JSON-Bodies nicht unnötig Speicher und CPU belegen.

### Authentifizierungs- und Rate-Limit-Details

Der Mindestwert von 32 Bytes erzwingt keine Zufälligkeit: `Support__AdminToken` muss aus einem kryptografischen Zufallsgenerator stammen und zentral geheim gehalten werden. Die gezeigte Rate-Limit-Policy deckt Schreibzugriffe auf öffentliche Tickets, nicht aber Player-GETs oder Admin-Routen ab. Access Codes mit 256 Bit Zufall sind gegen Raten robust; trotzdem sind Limits für teure Lese-/Admin-Abfragen und Authentifizierungsfehler sowie Pagination sinnvoll.

Der Ticket-Identifier nutzt 4 Zufallsbytes. Er ist durch den separaten 256-Bit-Zugriffscode keine Authentifizierung, aber 32 Bit sind für viele Jahre Betrieb knapp. Auf 128 Bit erhöhen und Response-/Abfrage-Limits setzen.

## UpdatePublisher.EXE

`UpdatePublisher.exe` ist ein nicht signierter x64-PE32+-Apphost ohne eigenen CLR-Metadatensatz. Eingebettete Strings verweisen auf `UpdatePublisher.dll` und den .NET-Host; die EXE ist deshalb nicht der eigentliche Publisher-Code und kein vollständiges Standalone-Release. Das Programm wurde nicht gestartet. Für Release-Abläufe sollte das Publisher-Artefakt aus einem vertrauenswürdigen Build stammen und signiert oder anderweitig verifiziert werden; private Schlüssel gehören nicht in dieses Paket.

## Zusammenhang mit dem Launcher-Update

Der Publisher signiert nur die eine angegebene EXE-Datei. Zusammen mit dem Launcher-Updater, der ausschließlich `VisionRPLauncher.exe` ersetzt, bestätigt das die vorherige Feststellung: Im bereitgestellten Release liegt der eigentliche Anwendungscode separat in `VisionRPLauncher.dll`, die Updateabläufe aber nicht ersetzen. Ein vollständiges Single-File-Artefakt oder ein signiertes Mehrdateienpaket mit atomarer Aktivierung ist erforderlich.

## Paketstatus

`SupportServer.csproj` pinnte `Microsoft.OpenApi` 2.7.5, das unterhalb der im Advisory festgelegten gepatchten 2.x-Grenze 2.12.2 liegt. Die Ausnutzbarkeit hängt davon ab, ob der Server betroffene Parser-/Traversal-Operationen mit angreifergesteuerten OpenAPI-Dokumenten ausführt; der bereitgestellte API-Code zeigt keinen solchen Pfad. Das Paketupdate auf 2.13.0 ist im vorherigen [Projektprüfbericht](VisionRPLauncher-Projektpruefung.md) samt [Patch](SupportServer-Paketupdate.patch) vorbereitet. [Advisory GHSA-m2pf-rgqv-ggvv](https://github.com/microsoft/OpenAPI.NET/security/advisories/GHSA-m2pf-rgqv-ggvv)

## Prüfsummen der geprüften Dateien

| Datei | SHA-256 |
|---|---|
| `SupportServer/Program.cs` | `B83D8E2A6AA78134DFF5EC23AB513068B1A976173FEE578E6D6C9F8AD7013C4C` |
| `SupportServer/Endpoints/TicketEndpoints.cs` | `65C5A2D8BD1087E70A77D8FE2FEA6C9A744606BD18EC04E9CBB171E13D7207E1` |
| `SupportServer/Middleware/AdminAuthorizationFilter.cs` | `E1A796707A2CAB1676293EDEA58479D0C320A0B137A7DAFEE0B15E0293778834` |
| `SupportServer/Services/TicketService.cs` | `BA3D797DF93DB67B34BEC01A35D013C74FF68E85550FE7A79681C03BBF2800FB` |
| `SupportServer/Models/TicketDtos.cs` | `BDCBFD3291225DED464AF6785870230EE51727D8FAEC18BDD841B833D8DD29D8` |
| `Tools/UpdatePublisher/Program.cs` | `01F2225A19DA8D473DF00617D05FDDC02AA49E3A200D0CCD3D8CA59A589B2494` |
| `Tools/UpdatePublisher.exe` | `D719F547601DCC200B6EC0BB6DF2DCC0B7247481552AC2DEB89D315C018482A8` |

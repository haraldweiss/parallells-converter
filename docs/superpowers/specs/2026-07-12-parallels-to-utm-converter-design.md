# Parallels-to-UTM Converter – Design

## Ziel

Eine native macOS-App konvertiert ausgewählte große Parallels-`.pvm`-Pakete nach UTM. Die Person wählt nur Quell- und Zielordner sowie die gewünschten VMs; die App übernimmt Erkennung, Prüfung, Konvertierung und Erstellung der UTM-VM.

## Zielgruppe und Umfang

- macOS-App mit SwiftUI-Oberfläche.
- Mehrere Windows-PVMs in einem Quellordner erkennen und einzeln auswählbar machen.
- Einstellungen der PVM übernehmen, soweit sie für UTM abbildbar sind: Architektur, CPU-Anzahl, RAM, primärer Datenträger und Netzwerkmodus.
- Die Konvertierung erfolgt seriell und verändert die PVM niemals.
- Benötigte lokale Programme: UTM und `qemu-img`. Die App sucht `qemu-img` zuerst in gängigen Installationsorten (einschließlich Homebrew) und meldet eine fehlende Abhängigkeit vor dem Start.

Nicht im ersten Umfang: Migration von Parallels-spezifischen Freigaben, USB-Durchreichung, 3D-Grafikeinstellungen oder eingefrorenen Zuständen. Eine VM muss vollständig heruntergefahren sein.

## Benutzeroberfläche

Ein Fenster mit drei klaren Bereichen:

1. Quell- und Zielordner über macOS-Dateiauswahldialoge festlegen.
2. Gefundene `.pvm`-Pakete als Liste anzeigen. Jede Zeile enthält Name, belegten Speicher, erkannte Architektur, Kompatibilitätsstatus und Auswahlfeld.
3. Vorabprüfung und Schaltfläche zum Starten. Die Prüfung zeigt erforderlichen freien Speicher und blockierende Probleme.

Während der seriellen Konvertierung zeigt jede VM Status, Fortschritt, geschätzte Restzeit und ein einblendbares Protokoll. Danach zeigt die App das erzeugte `.utm`-Paket im Zielordner an.

## Konvertierungspfad

1. Eine PVM wird als Paket gelesen. `config.pvs`, `DiskDescriptor.xml` und gegebenenfalls Snapshot-Metadaten werden geparst.
2. Der aktive `.hds`-Datenträger wird ausschließlich aus `DiskDescriptor.xml` abgeleitet. Die App konvertiert niemals blind alle `.hds`-Dateien.
3. `qemu-img info` prüft das Eingabeabbild. Für eine unterstützte Parallels-Disk wird eine QCOW2-Datei im Ziel-UTM-Paket erzeugt.
4. Die App legt über die UTM-Schnittstelle eine QEMU-VM an und überträgt Name, Gastarchitektur, CPU, RAM, Disk und Netzwerk. Falls UTM-Automation nicht verfügbar ist, erzeugt sie das lokale UTM-Paket und weist auf den erforderlichen Import hin.
5. `qemu-img` prüft das Ergebnis; anschließend wird die UTM-Konfiguration gegen die erkannten PVM-Einstellungen validiert.

## Nicht sperrfähige Quellvolumes

Ein bekanntes Quellvolume kann QEMU-Byte-Sperren nicht bereitstellen. Der Fehler lautet `Failed to lock byte 100: Operation not supported`.

Die App erkennt diesen Fehler beim Vorabtest. Nur wenn die PVM als heruntergefahren geprüft wurde, setzt sie für reine Lesezugriffe den QEMU-Schalter `-U` ein. Dadurch wird der fehlende Lock-Support umgangen, ohne die Quelle zu verändern. Bei nicht eindeutig heruntergefahrenem Zustand bricht die App mit einer verständlichen Meldung ab.

Die untersuchte Referenz-PVM bestätigt diesen Pfad: Der aktive Datenträger ist `Windows 11-0.hdd.0.{5fbaabe3-6958-40ff-92a7-860e329aab41}.hds`, QEMU erkennt ihn als 256-GiB-Parallels-Image, und die PVM enthält die Einstellungen 8 CPU-Kerne und 8 GiB RAM.

## Fehlerverhalten und Sicherheit

- Zugriff erfolgt ausschließlich auf die durch die zwei Ordnerdialoge autorisierten Ordner.
- Vor dem Start: UTM, QEMU, Lesbarkeit, Snapshot-Kette, VM-Zustand und freier Speicher prüfen.
- Originaldateien bleiben unverändert; Ausgaben werden nur unterhalb des Zielordners geschrieben.
- Fehlgeschlagene Ziele werden klar markiert und können über die UI gelöscht oder erneut gestartet werden.
- Pro VM steht ein kopierbares Protokoll mit Befehl, Quelle, Ziel und Fehlertext bereit.

## Tests

- Unit-Tests: XML-Parser, Ermittlung des aktiven Snapshots, Einstellungsübernahme, Speicherberechnung und QEMU-Befehlserzeugung.
- Integrationstests: Vorabprüfungen, QEMU-Fehlerklassifikation und UTM-Konfigurationsaufbau mit Test-Fixtures.
- Manuelle Abnahme: vollständige Konvertierung der vorhandenen 256-GiB-Windows-on-ARM-PVM nur nach ausdrücklicher Auswahl durch die Person.

## Erfolgskriterien

- Nach Auswahl von Quell- und Zielordner benötigt die Konvertierung keine Terminaleingaben.
- Nur ausgewählte PVMs werden verarbeitet.
- Für unterstützte, heruntergefahrene PVMs entsteht ein geprüftes UTM-Paket im Zielordner.
- Nicht unterstützte oder unsichere Zustände werden vor dem Schreiben verständlich abgelehnt.

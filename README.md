# gPlugK BKW – ESPHome

ESPHome-Konfiguration für den **gPlugK V3.11 mit ESP32-C3** am BKW/Kamstrup-OMNIPOWER-Smart-Meter.

Das Repository besteht nur aus normaler ESPHome-YAML. Es werden **keine lokalen oder externen Custom Components** benötigt. Die Zählerwerte werden mit dem nativen `dlms_meter` von ESPHome gelesen und über die ESPHome-API direkt an Home Assistant übertragen.

> **Wichtig:** Das Aufspielen von ESPHome ersetzt die ab Werk installierte Tasmota-Firmware. gPlug.ch weist darauf hin, dass der Hersteller-Support nur für die ausgelieferte Firmware gewährleistet wird. Ein Firmwarewechsel erfolgt deshalb auf eigene Verantwortung.

---

## Was die Konfiguration kann

- BKW/Kamstrup-DLMS über die HAN/CII-Schnittstelle lesen
- verschlüsselte Telegramme mit GPK60 entschlüsseln
- optional GPK61 für Authentifizierung verwenden
- 30 direkte Zählerwerte an Home Assistant senden
- saldierte Leistungs-, Strom- und Sicherungswerte berechnen
- zeitgewichtete 15-Minuten-Mittel berechnen
- veraltete Zählerdaten erkennen
- WLAN-, Temperatur- und Laufzeitdiagnose liefern
- lokale ESPHome-Weboberfläche bereitstellen
- OTA-Updates ermöglichen
- die RGB-LED des Boards als Statusanzeige verwenden
- den physischen AP-Taster auf GPIO9 verwenden
- den physischen Reset-Taster unverändert als Hardware-Reset nutzen
- den ESP32-C3 mit einem stromsparenden WLAN-/CPU-Profil betreiben

---

# 1. Hardware und Pinbelegung

Dieses Projekt ist für die **gPlugK-V3.11-Platine mit ESP32-C3** ausgelegt.

| Funktion | ESP32-C3 GPIO | ESPHome |
| --- | ---: | --- |
| HAN / CII Datenempfang | GPIO4 | UART RX |
| RGB-LED Rot | GPIO5 | Ausgang |
| RGB-LED Grün | GPIO6 | Ausgang |
| RGB-LED Blau | GPIO7 | Ausgang |
| AP-Taster | GPIO9 | Eingang mit Pull-up, aktiv LOW |
| Reset-Taster | kein Firmware-GPIO erforderlich | Hardware-Reset |

Die Pins stehen ganz oben in `gplugk-bkw.yaml` und sollten für diese Hardware normalerweise **nicht verändert** werden:

```yaml
han_rx_pin: GPIO4
led_red_pin: GPIO5
led_green_pin: GPIO6
led_blue_pin: GPIO7
ap_button_pin: GPIO9
```

## UART-Einstellungen

Die Kundenschnittstelle wird mit **2400 Baud, 8 Datenbits, keiner Parität und 1 Stopbit (8N1)** gelesen:

```yaml
uart:
  id: meter_uart
  rx_pin: GPIO4
  baud_rate: 2400
  data_bits: 8
  parity: NONE
  stop_bits: 1
  rx_buffer_size: 4096
```

Der größere UART-Puffer bietet Reserve für komplette DLMS-Telegramme und verändert die elektrische Schnittstelle nicht.

## Skalierung der Kamstrup-Werte

Die YAML bildet die Rohwerte in die Einheiten um, die Home Assistant erwartet:

| Wert | Umrechnung in der YAML | Ergebnis |
| --- | ---: | --- |
| Wirk-/Blindenergie | × 0,001 | kWh / kvarh |
| Strom | × 0,01 | A |
| Leistungsfaktor | × 0,01 | Verhältnis 0…1 |
| Spannung | × 1 | V |
| Wirk-/Blindleistung | × 1 | W / var |

Der optionale `ct_factor` wird anschließend auf Energie, Strom und Leistung angewendet. Spannung und Leistungsfaktor bleiben davon unberührt.

---

# 2. RGB-LED

Die V3.11 besitzt eine **RGB-LED mit drei getrennten GPIOs**. Die ESPHome-Konfiguration verwendet sie als kompakte Statusanzeige:

| LED | Bedeutung |
| --- | --- |
| Rot → Grün → Blau einmal beim Start | Firmware wurde gestartet |
| Grün flackert kurz | Zählertelegramm wurde empfangen |
| Blau blinkt | keine aktuellen Smart-Meter-Daten |
| Rot blinkt | keine Verbindung zum lokalen WLAN |

Die Statuslogik verwendet folgende Priorität:

```text
Startsequenz
    ↓
WLAN fehlt → Rot
    ↓
Zählerdaten fehlen/veraltet → Blau
    ↓
Datenempfang → kurzer grüner Impuls
```

Damit entstehen bei mehreren gleichzeitigen Zuständen keine schwer lesbaren Mischfarben. Bei normalem Betrieb ist ungefähr im Takt der Zählertelegramme ein kurzer grüner Impuls zu sehen.

---

# 3. AP-Taster

Der auf der Platine mit **AP** bezeichnete Taster liegt auf GPIO9 und wird als aktiver LOW-Eingang mit internem Pull-up verwendet:

```yaml
pin:
  number: GPIO9
  mode:
    input: true
    pullup: true
  inverted: true
```

Ein Druck auf die Taste aktiviert den ESPHome-Setup-Access-Point, **wenn keine normale WLAN-Verbindung besteht**.

Der Access Point heißt standardmäßig:

```text
gplugk-bkw Setup
```

Das Passwort ist der Wert `fallback_password` aus `secrets.yaml`. Falls das Captive Portal nicht automatisch erscheint, öffne einen Pfad ungleich `/`, zum Beispiel:

```text
http://192.168.4.1/setup
```

Die Adresse `http://192.168.4.1/` zeigt bei dieser Konfiguration die normale ESPHome-Weboberfläche.

In der YAML steht absichtlich `ap_timeout: 0s`. Der Setup-AP läuft damit nicht unnötig im normalen Betrieb, sondern wird bei Bedarf über den Taster freigegeben. Für längere Einrichtung oder Flash-Vorgänge ist eine zusätzliche Versorgung über USB-C sinnvoll.

Wenn das normale WLAN noch funktioniert, unterbricht der AP-Taster die bestehende Verbindung nicht. WLAN-Daten können dann bequem in `secrets.yaml` geändert und per OTA installiert werden.

---

# 4. Wichtiger Hinweis zu GPIO9

GPIO9 ist beim ESP32-C3 gleichzeitig ein **Boot-/Strapping-Pin**. Im laufenden Betrieb kann er problemlos als Tastereingang verwendet werden.

Halte den AP-Taster aber **nicht während eines normalen Reset- oder Einschaltvorgangs gedrückt**, außer du möchtest bewusst den ESP32-C3 in den Firmware-Download-Modus bringen. Ein LOW-Pegel auf GPIO9 während des Resets kann den Bootloader-Modus auswählen.

Die YAML quittiert die entsprechende ESPHome-Warnung deshalb bewusst mit:

```yaml
ignore_strapping_warning: true
```

---

# 5. Reset-Taster

Der zweite physische Taster ist der **Reset-Taster**. Er wirkt hardwareseitig auf den ESP32-C3 und benötigt deshalb keinen ESPHome-`binary_sensor`. Ein kurzer Druck startet den Controller unabhängig von WLAN und Home Assistant neu.

Zusätzlich stellt ESPHome in der Weboberfläche und in Home Assistant den Button `Software Neustart` bereit.

---

# 6. Energieversorgung und Low-Power-Profil

Die Kamstrup-HAN-Schnittstelle ist leistungsmäßig knapp dimensioniert. Kamstrup spezifiziert für ein angeschlossenes HEMS maximal **144 mW** beziehungsweise **6 mA (4 Unit Loads nach EN 13757-2)**. Der Superkondensator auf dem Board dient als Energiepuffer für kurzzeitige Lastspitzen, insbesondere beim WLAN-Senden.

Die ESPHome-Konfiguration ist deshalb auf niedrigen Durchschnittsverbrauch ausgelegt:

- **ESP-IDF Dynamic Frequency Scaling (DFS)** ist aktiviert.
- Maximale CPU-Frequenz: **160 MHz**.
- Im Leerlauf kann ESP-IDF bis auf die **40-MHz-XTAL-Frequenz** heruntertakten.
- WLAN verwendet `power_save_mode: light` bzw. **WIFI_PS_MIN_MODEM**.
- `post_connect_roaming: false` verhindert bei diesem stationären Gerät periodische WLAN-Roaming-Scans.
- **Auto-Light-Sleep ist nicht aktiviert**, damit der kontinuierliche DLMS-UART-Empfang auf GPIO4 nicht durch abgeschaltete Peripherie gefährdet wird.
- Die ESPHome-API bündelt Zustandsänderungen mit `batch_delay: 500ms`.
- 15-Minuten-Diagnosewerte werden ereignisgesteuert beim Zählertelegramm veröffentlicht statt jede Sekunde.
- `Alter der Zählerdaten` wird nur alle 10 Sekunden aktualisiert; `Datenstatus` alle 5 Sekunden.
- Webserver, API, OTA und alle Messwerte bleiben verfügbar.

Espressif empfiehlt für ESP32-C3-WLAN-Anwendungen die Kombination **Min Modem-sleep + DFS** mit 160 MHz Maximal- und 40 MHz Minimalfrequenz. In den Referenzmessungen liegt der mittlere Chipstrom bei DTIM1 mit dieser Kombination bei rund **11,35 mA**, gegenüber rund **21,47 mA ohne DFS**. Das sind Referenzwerte des ESP32-C3 und keine garantierte Leistungsaufnahme des kompletten Boards.

## Warum `power_save_mode: high` nicht Standard ist

Der aggressivere Modus kann noch stärker sparen, erhöht aber die Netzwerklatenz und kann Broadcast-/Multicast-Verkehr ungünstiger behandeln. Für einen dauerhaft erreichbaren Home-Assistant-Sensor ist `light` die robustere Grundeinstellung.

## Warum die WLAN-Sendeleistung nicht pauschal reduziert wird

ESPHome erlaubt eine feste maximale `output_power`. Im Zählerschrank kann das WLAN-Signal aber stark gedämpft sein. Eine zu niedrige Sendeleistung kann Wiederholungen und Reconnects verursachen und damit Zuverlässigkeit und Energieverbrauch verschlechtern. Deshalb bleibt die Sendeleistung in der Standardkonfiguration unverändert.

## Warum WLAN-Roaming deaktiviert ist

ESPHome aktiviert `post_connect_roaming` standardmäßig. Dabei kann ein verbundenes Gerät nach besseren Access Points mit derselben SSID suchen. Der gPlugK ist stationär; solche Scans bringen hier normalerweise keinen Nutzen, erzeugen aber zusätzliche Funkaktivität. Deshalb ist gesetzt:

```yaml
wifi:
  post_connect_roaming: false
```

Weiterführende technische Dokumentation:

- Kamstrup HAN module data sheet: https://kamstrup-delivery.sitecorecontenthub.cloud/api/public/content/60724-downloadOriginal?v=3794f2c2
- ESPHome WiFi: https://esphome.io/components/wifi/
- Espressif ESP32-C3 Power Management: https://docs.espressif.com/projects/esp-idf/en/v5.5/esp32c3/api-guides/low-power-mode/low-power-mode-wifi.html

---

# 7. Voraussetzungen

Du benötigst:

1. gPlugK mit ESP32-C3
2. Kamstrup OMNIPOWER mit aktivierter Kundenschnittstelle
3. technischen Datensatz bzw. Schlüssel deines Netzbetreibers
4. BKW-GPK60, falls das Telegramm verschlüsselt ist
5. 2,4-GHz-WLAN
6. ESPHome **2026.8.2 oder neuer**
7. für die Erstinstallation möglichst ein USB-C-Kabel

---

# 8. Dateien im Repository

```text
gplugk-bkw.yaml          Hauptkonfiguration
secrets.example.yaml     Vorlage für Passwörter und Schlüssel
README.md                diese Anleitung
LICENSE                  Lizenz
.gitignore               schützt unter anderem secrets.yaml
.gitattributes
```

Für die tägliche Verwendung brauchst du hauptsächlich:

```text
gplugk-bkw.yaml
secrets.yaml
```

---

# 9. Schnellstart für Einsteiger

## Schritt 1 – Dateien kopieren

Lade dieses Repository herunter und kopiere die Dateien in deinen ESPHome-Konfigurationsordner.

## Schritt 2 – `secrets.yaml` anlegen

Kopiere:

```text
secrets.example.yaml
```

und benenne die Kopie um in:

```text
secrets.yaml
```

Beispiel:

```yaml
wifi_ssid: "MeinWLAN"
wifi_password: "MeinWLANPasswort"

fallback_password: "MeinSetupPasswort"

api_key: "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="

web_username: "admin"
web_password: "MeinWebPasswort"

bkw_gpk60: "00112233445566778899AABBCCDDEEFF"
```

Ersetze selbstverständlich alle Beispielwerte durch deine eigenen Werte.

> `secrets.yaml` niemals auf GitHub hochladen. Darin stehen WLAN-Passwort, API-Key und dein persönlicher Zählerschlüssel.

---

# 10. GPK60 und GPK61

## GPK60

Der GPK60 ist bei verschlüsselter Übertragung der Entschlüsselungsschlüssel. Er besteht normalerweise aus **32 Hex-Zeichen = 16 Byte**.

Beispiel für das Format:

```text
00112233445566778899AABBCCDDEEFF
```

Nicht dieses Beispiel übernehmen, sondern den echten Schlüssel deines Zählers verwenden.

## GPK61 optional

Die Konfiguration verwendet standardmäßig nur GPK60.

Falls du GPK61 ausdrücklich für die Authentifizierung nutzen möchtest, suche in `gplugk-bkw.yaml`:

```yaml
# auth_key: !secret bkw_gpk61
```

und ändere es zu:

```yaml
auth_key: !secret bkw_gpk61
```

Danach zusätzlich in `secrets.yaml`:

```yaml
bkw_gpk61: "FFEEDDCCBBAA99887766554433221100"
```

---

# 11. Wandlerfaktor

Oben in `gplugk-bkw.yaml` steht:

```yaml
ct_factor: "1.0"
```

## Normaler Haushalts-/Direktmesszähler

So lassen:

```yaml
ct_factor: "1.0"
```

## Wandlerzähler

Wenn BKW für deinen Anschluss einen Ablese-/Wandlerfaktor nennt, diesen eintragen.

Beispiel:

```yaml
ct_factor: "20.0"
```

Der Faktor wird auf Energie, Strom und Leistung angewendet. Spannung und Leistungsfaktor werden nicht multipliziert.

---

# 12. Hauptsicherung

Oben in der YAML steht:

```yaml
main_fuse_a: "0"
```

`0` bedeutet: unbekannt bzw. Sicherungsüberwachung deaktiviert. In diesem Fall bleibt die Entität `Hauptsicherung` bewusst unbekannt. Bei einem Wert größer als 0 wird der konfigurierte Sicherungswert beim Start einmal veröffentlicht.

Beispiel für 25 A pro Phase:

```yaml
main_fuse_a: "25"
```

Beispiel für 40 A:

```yaml
main_fuse_a: "40"
```

Dadurch werden diese Werte sinnvoll berechnet:

- `Hauptsicherung Auslastung`
- `Stromreserve schwächste Phase`
- `Sicherungsgrenze Phasenstrom`

---

# 13. Kompilieren und installieren

## ESPHome Dashboard

1. YAML in ESPHome öffnen.
2. **Install** wählen.
3. Für die erste Installation USB verwenden.
4. Nach erfolgreicher Erstinstallation können weitere Updates per OTA erfolgen.

## ESPHome CLI

```bash
esphome run gplugk-bkw.yaml
```

Nach dem Flashen startet die RGB-LED einmal mit:

```text
Rot → Grün → Blau
```

Danach versucht das Gerät, sich mit deinem WLAN zu verbinden.

---

# 14. Wenn nach dem Flashen kein WLAN erreichbar ist

1. gPlugK vorzugsweise über USB-C versorgen.
2. Kurz den **AP-Taster** drücken.
3. Einige Sekunden warten.
4. Auf Smartphone oder Notebook nach folgendem WLAN suchen:

```text
gplugk-bkw Setup
```

5. Mit dem `fallback_password` aus `secrets.yaml` verbinden.
6. Falls keine Seite automatisch erscheint:

```text
http://192.168.4.1/
```

7. WLAN-Zugangsdaten eintragen.

---

# 15. Home Assistant

Es wird keine zusätzliche Home-Assistant-YAML benötigt.

Wenn das Gerät automatisch gefunden wird, einfach die ESPHome-Integration hinzufügen.

Falls nicht:

1. Home Assistant → **Einstellungen**
2. **Geräte & Dienste**
3. **Integration hinzufügen**
4. **ESPHome**
5. IP-Adresse oder Hostname des gPlugK eingeben
6. falls verlangt den API-Key aus `secrets.yaml` verwenden

---

# 16. Energy Dashboard

Für das Home-Assistant-Energy-Dashboard sind besonders geeignet:

```text
Energie Bezug Gesamt
Energie Rücklieferung Gesamt
```

Beide werden in `kWh` mit `total_increasing` bereitgestellt.

---

# 17. Direkte Zählerwerte

Die YAML liest 30 numerische DLMS-Werte.

## Gesamtwerte

- Energie Bezug Gesamt
- Energie Rücklieferung Gesamt
- Blindenergie Plus Gesamt
- Blindenergie Minus Gesamt
- Leistung Bezug Gesamt
- Leistung Rücklieferung Gesamt
- Blindleistung Plus Gesamt
- Blindleistung Minus Gesamt

## Spannung und Strom

- Spannung L1
- Spannung L2
- Spannung L3
- Strom L1
- Strom L2
- Strom L3

## Wirkleistung pro Phase

- Leistung Bezug L1
- Leistung Bezug L2
- Leistung Bezug L3
- Leistung Rücklieferung L1
- Leistung Rücklieferung L2
- Leistung Rücklieferung L3

## Leistungsfaktor

- Leistungsfaktor L1
- Leistungsfaktor L2
- Leistungsfaktor L3
- Leistungsfaktor Gesamt

## Energie pro Phase

- Energie Bezug L1
- Energie Bezug L2
- Energie Bezug L3
- Energie Rücklieferung L1
- Energie Rücklieferung L2
- Energie Rücklieferung L3

Zusätzlich werden als Textwerte veröffentlicht:

- `OBIS Listenkennung`
- `Zähleruhr`

---

# 18. Berechnete Werte

| Sensor | Bedeutung |
| --- | --- |
| Netzleistung saldiert | Bezug minus Rücklieferung; positiv = Bezug, negativ = Einspeisung |
| Bezug saldiert | tatsächlich saldierter Bezug |
| Rücklieferung saldiert | tatsächlich saldierte Einspeisung als positiver Wert |
| Netzleistung L1/L2/L3 | Bezug minus Rücklieferung je Phase |
| Netzleistung aus Phasen | Summe der drei saldierten Phasen |
| Saldierungsabweichung | Differenz Gesamtwert ↔ Summe der Phasen |
| Gleichzeitiger Bezug und Rücklieferung | möglicher Überlappungswert in W |
| Stromsumme L1 L2 L3 | Summe aller Phasenströme |
| Höchster Phasenstrom | aktuell stärkste Phase |
| Phasenstrom Spreizung | höchster minus niedrigster Phasenstrom |
| Scheinleistung Summe U x I | Summe von Spannung × Strom je Phase |
| Hauptsicherung Auslastung | Belastung der stärksten Phase in % |
| Stromreserve schwächste Phase | Reserve bis zum eingestellten Sicherungswert |
| Hauptsicherung | Eingestellter Sicherungswert |
| Höchstbelastete Phase | L1, L2 oder L3 |

---

# 19. 15-Minuten- und Laufzeitwerte

Im RAM des ESP32 werden zusätzlich geführt:

- Bezugsspitze seit Neustart
- Rücklieferspitze seit Neustart
- Bezug Mittel 15 Minuten
- Rücklieferung Mittel 15 Minuten
- Höchstes 15-Minuten-Mittel seit Neustart
- Empfangene Datensätze seit Neustart
- Alter der Zählerdaten
- Letztes Empfangsintervall
- Abdeckung 15-Minuten-Mittel

Das 15-Minuten-Mittel ist **zeitgewichtet**. Die reale Zeit zwischen zwei Telegrammen wird berücksichtigt.

Nach einem Neustart beginnen Laufzeit- und Peakwerte wieder von vorne. Das verhindert unnötige Schreibzugriffe auf den Flash.

---

# 20. Freshness / veraltete Daten

Standardmäßig gilt ein Datensatz nach 35 Sekunden ohne neues vollständiges Telegramm als veraltet:

```yaml
stale_after: "35s"
stale_after_ms: "35000"
```

Normalerweise musst du das nicht ändern.

Bei fehlenden Daten:

- direkte Messwerte werden nicht weiter als aktuell gehalten,
- `Datenstatus` zeigt `Daten veraltet`,
- `Zählerdaten aktuell` wird `Aus`,
- die RGB-LED blinkt bei vorhandener WLAN-Verbindung blau.

---

# 21. Diagnosewerte

## WLAN

- WLAN Signal in dBm
- WLAN Signal Prozent
- IP-Adresse
- Verbundenes WLAN / SSID

Grobe RSSI-Orientierung:

| Signal | Bewertung |
| ---: | --- |
| -30 bis -50 dBm | sehr gut |
| ca. -60 dBm | gut |
| ca. -70 dBm | brauchbar |
| unter -75 dBm | eher schwach |
| unter -85 dBm | möglicherweise instabil |

## ESP32

- Chiptemperatur
- Laufzeit
- Letzter Neustartgrund
- ESPHome Version
- ESPHome Verbindung

Die `Chiptemperatur` ist die interne Temperatur des ESP32-C3 und **keine Raumtemperatur**.

Die CPU läuft nicht dauerhaft mit 160 MHz: Durch das aktivierte ESP-IDF-DFS kann sie in Leerlaufphasen bis auf 40 MHz heruntertakten und bei Last automatisch wieder höher takten. Das geschieht transparent; DLMS, API und Weboberfläche müssen dafür nicht abgeschaltet werden.

## Zählerdiagnose

- Datenstatus
- Zählerdaten aktuell
- Alter der Zählerdaten
- Letztes Empfangsintervall
- Empfangene Datensätze seit Neustart
- Abdeckung 15-Minuten-Mittel
- Sicherungsgrenze Phasenstrom (`Innerhalb Grenzwert`, `Überschritten`, `Nicht konfiguriert` oder `Keine aktuellen Daten`)
- Status gleichzeitiger Bezug und Rücklieferung (`Erkannt`, `Nicht erkannt` oder `Keine aktuellen Daten`)

## Hardwarediagnose

- Software Neustart

Der physische AP-Taster bleibt vollständig funktionsfähig, wird aber nicht als eigene Entität in Home Assistant oder der Weboberfläche veröffentlicht.

---

# 22. Lokale Weboberfläche

Rufe die IP-Adresse des Geräts auf, zum Beispiel:

```text
http://192.168.1.123/
```

oder normalerweise:

```text
http://gplugk-bkw.local/
```

Der Login verwendet:

```yaml
web_username: "admin"
web_password: "MeinWebPasswort"
```

Die Entitäten werden in Gruppen sortiert:

- Netzanschluss
- Phasen
- Zählerstände
- Berechnete Werte
- Zählerinformationen
- Diagnose

---

# 23. Typische Fehler

## RGB-LED bleibt nach dem Start blau

Das bedeutet in dieser Konfiguration:

```text
WLAN vorhanden, aber keine aktuellen Zählerdaten
```

Prüfen:

1. sitzt der gPlugK korrekt im HAN-Schacht?
2. ist die Kundenschnittstelle vom Netzbetreiber aktiviert?
3. ist der GPK60 korrekt?
4. erscheinen im ESPHome-Log DLMS-Daten?
5. wird `Alter der Zählerdaten` aktualisiert?

## RGB-LED blinkt rot

Keine lokale WLAN-Verbindung.

Wenn dein WLAN absichtlich geändert wurde oder nicht mehr verfügbar ist:

1. USB-C anschließen,
2. AP-Taster drücken,
3. mit `gplugk-bkw Setup` verbinden,
4. `192.168.4.1/setup` öffnen.

## Grün flackert regelmäßig, aber Home Assistant zeigt nichts

Dann funktioniert der grundsätzliche Datenempfang wahrscheinlich bereits. Prüfe anschließend:

- ESPHome-API-Verbindung
- API-Key
- Home-Assistant-Integration
- ESPHome-Log auf DLMS- oder Entschlüsselungsfehler

## Gerät startet unerwartet neu

Prüfe:

- Letzter Neustartgrund
- Versorgung über HAN bzw. testweise USB-C

## Gerät startet nach gleichzeitigem Drücken von AP und Reset nicht normal

GPIO9 ist ein Boot-Strapping-Pin. Wenn der AP-Taster während eines Resets LOW gehalten wird, kann der ESP32-C3 in den Download-Modus starten.

Lösung:

1. AP-Taster loslassen.
2. Reset-Taster kurz drücken.

Dann sollte die normale Firmware wieder starten.

---

# 24. Sicherheit

Folgende Werte niemals veröffentlichen:

- WLAN-Passwort
- `fallback_password`
- GPK60
- GPK61
- ESPHome API-Key
- Passwort der Weboberfläche

Genau deshalb liegt nur `secrets.example.yaml` im Repository.

---

# 25. Die wichtigsten Einstellungen auf einen Blick

Für einen normalen Direktmesszähler mit 25-A-Hauptsicherung müssen meistens nur diese Werte stimmen:

```yaml
substitutions:
  device_name: gplugk-bkw
  friendly_name: gPlugK

  han_rx_pin: GPIO4
  led_red_pin: GPIO5
  led_green_pin: GPIO6
  led_blue_pin: GPIO7
  ap_button_pin: GPIO9

  ct_factor: "1.0"
  main_fuse_a: "25"
```

und in `secrets.yaml`:

```yaml
wifi_ssid: "MeinWLAN"
wifi_password: "MeinWLANPasswort"
fallback_password: "MeinSetupPasswort"
api_key: "MEIN_API_KEY"
web_username: "admin"
web_password: "MeinWebPasswort"
bkw_gpk60: "MEIN_32_STELLIGER_HEX_SCHLUESSEL"
```

Danach kompilieren, per USB installieren und die Startsequenz der RGB-LED beobachten.

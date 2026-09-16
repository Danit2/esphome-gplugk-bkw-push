# gPlugK BKW – ESPHome

ESPHome-Konfiguration für den **gPlugK mit ESP32-C3** am BKW/Kamstrup-OMNIPOWER-Smart-Meter.

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
- WLAN-, Speicher-, Temperatur- und Laufzeitdiagnose liefern
- lokale ESPHome-Weboberfläche bereitstellen
- OTA-Updates ermöglichen
- die **echte RGB-LED des gPlugK** als Statusanzeige verwenden
- den **physischen AP-Taster** verwenden
- den **physischen Reset-Taster** unverändert als Hardware-Reset nutzen

---

# 1. Hardware und Pinbelegung

Der gPlugK definiert folgende Pins:

```text
rL    = 5
 gL    = 6
 bL    = 7
 butA  = 9
 rxPin = 4
```

Daraus ergibt sich für diese ESPHome-Konfiguration:

| Funktion | ESP32-C3 GPIO | ESPHome |
| --- | ---: | --- |
| HAN / CII Datenempfang | GPIO4 | UART RX |
| RGB-LED Rot | GPIO5 | Ausgang |
| RGB-LED Grün | GPIO6 | Ausgang |
| RGB-LED Blau | GPIO7 | Ausgang |
| AP-Taster | GPIO9 | Eingang mit Pull-up, aktiv LOW |
| Reset-Taster | kein GPIO in der Firmware erforderlich | Hardware-Reset |

Die Pins stehen ganz oben in `gplugk-bkw.yaml` und sollten für den gPlugK normalerweise **nicht verändert** werden:

```yaml
han_rx_pin: GPIO4
led_red_pin: GPIO5
led_green_pin: GPIO6
led_blue_pin: GPIO7
ap_button_pin: GPIO9
```

## UART-Einstellungen

Die offizielle gPlugK-Anleitung nennt für die Kamstrup-CII:

```text
2400 Baud
8 Datenbits
keine Parität
1 Stopbit
```

also **2400 Baud, 8N1**.

Genau das ist in der YAML eingestellt:

```yaml
uart:
  id: meter_uart
  rx_pin: GPIO4
  baud_rate: 2400
  data_bits: 8
  parity: NONE
  stop_bits: 1
```

Der größere ESPHome-Empfangspuffer von 4096 Byte ist absichtlich gewählt und ändert die elektrische Schnittstelle nicht.

---

# 2. RGB-LED

Der gPlugK besitzt keine einfache einfarbige LED, sondern eine **RGB-LED mit drei getrennten GPIOs**.

Die ESPHome-Konfiguration übernimmt die Bedeutung der LED aus der offiziellen Anleitung:

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

Damit entstehen bei mehreren gleichzeitigen Problemen keine schwer lesbaren Mischfarben.

### Typisches Verhalten

Bei normalem Betrieb sieht man ungefähr alle 10 Sekunden einen kurzen grünen Impuls, weil der Kamstrup-Zähler seine Daten üblicherweise in diesem Intervall sendet.

Wenn WLAN fehlt, blinkt Rot ungefähr alle 9 Sekunden.

Wenn WLAN vorhanden ist, aber keine aktuellen Zählerdaten ankommen, blinkt Blau ungefähr alle 15 Sekunden.

---

# 3. AP-Taster

Der auf der Platine mit **AP** bezeichnete Taster ist in ESPHome so definiert:

```yaml
pin:
  number: GPIO9
  mode:
    input: true
    pullup: true
  inverted: true
```

Ein Druck auf die Taste aktiviert den konfigurierten ESPHome-Setup-Access-Point, **wenn keine normale WLAN-Verbindung besteht**.

Der Access Point heißt standardmäßig:

```text
gplugk-bkw Setup
```

Das Passwort ist der Wert `fallback_password` aus deiner `secrets.yaml`.

Danach kannst du dich mit Smartphone oder Notebook mit diesem WLAN verbinden. Falls das Captive Portal nicht automatisch erscheint, öffne:

```text
http://192.168.4.1/
```

Dort können neue WLAN-Zugangsdaten gespeichert werden. Sobald wieder eine normale WLAN-Verbindung besteht, wird der manuell freigegebene AP-Timeout wieder auf `0s` gesetzt.

## Warum startet der Access Point nicht automatisch?

In der YAML steht:

```yaml
ap_timeout: 0s
```

Das ist absichtlich so gewählt. Wir vermeiden einen automatisch dauerhaft gestarteten AP, weil der Access-Point-Betrieb zusätzliche Leistung benötigt. Der gPlugK wird im normalen Betrieb über die Kundenschnittstelle des Zählers versorgt, deren Leistungsreserve begrenzt sein kann.

Der physische AP-Taster übernimmt deshalb die Wiederherstellungsfunktion.

> **Tipp:** Für WLAN-Einrichtung, längeren AP-Betrieb und insbesondere beim Flashen den gPlugK am besten zusätzlich über **USB-C** versorgen. Das wird auch in der offiziellen gPlugK-Anleitung empfohlen.

## Wenn das normale WLAN noch funktioniert

Der AP-Taster unterbricht eine bestehende WLAN-Verbindung absichtlich nicht. Wenn du bei funktionierendem WLAN SSID oder Passwort ändern möchtest, ist es sauberer, `secrets.yaml` anzupassen und die Firmware per OTA neu zu installieren.

Wenn das alte WLAN nicht mehr verfügbar ist, drückst du einfach den AP-Taster und verwendest das Captive Portal.

---

# 4. Wichtiger Hinweis zu GPIO9

GPIO9 ist beim ESP32-C3 gleichzeitig ein **Boot-/Strapping-Pin**.

Im normalen laufenden Betrieb darf er problemlos als Tastereingang benutzt werden. Genau deshalb wird der vorhandene Hardwaretaster mit internem Pull-up als aktiver LOW-Eingang verwendet.

Halte den AP-Taster aber **nicht während eines normalen Reset- oder Einschaltvorgangs gedrückt**, außer du möchtest bewusst den ESP32-C3 in den Firmware-Download-Modus bringen. Ein LOW-Pegel auf GPIO9 während des Resets kann den Bootloader-Modus auswählen.

Diese Warnung wird in der YAML bewusst mit

```yaml
ignore_strapping_warning: true
```

quittiert, weil GPIO9 auf dieser Hardware absichtlich als Taster genutzt wird.

---

# 5. Reset-Taster

Der zweite physische Taster auf der Front ist der **Reset-Taster**.

Die offizielle gPlugK-Konfiguration weist diesem Taster keinen normalen GPIO zu. Für seine eigentliche Reset-Funktion ist deshalb **kein ESPHome-`binary_sensor` notwendig**.

Ein kurzer Druck startet den ESP32 neu, unabhängig davon, ob Home Assistant oder WLAN erreichbar sind.

Zusätzlich stellt ESPHome in der Weboberfläche und in Home Assistant den Button

```text
Software Neustart
```

bereit. Damit kann derselbe praktische Effekt auch aus der Ferne ausgelöst werden.

---

# 6. Energieversorgung und Low-Power-Profil

Die Kamstrup-HAN-Schnittstelle ist leistungsmässig knapp dimensioniert. Kamstrup spezifiziert für das angeschlossene HEMS maximal **144 mW** beziehungsweise **6 mA (4 Unit Loads nach EN 13757-2)**. Die offizielle gPlugK-Seite weist zudem darauf hin, dass HW V3.11 gegenüber den älteren Versionen um eine **Strombegrenzung** ergänzt wurde. Der grosse Superkondensator auf der V3.11-Platine puffert die kurzzeitigen Lastspitzen des ESP32-C3, insbesondere beim WLAN-Senden.

Die ESPHome-Konfiguration verwendet deshalb bewusst ein konservatives Energiesparprofil:

- **ESP-IDF Dynamic Frequency Scaling (DFS)** ist aktiviert.
- Maximale CPU-Frequenz: **160 MHz**.
- Im Leerlauf kann ESP-IDF bis auf die **40-MHz-XTAL-Frequenz** heruntertakten.
- WLAN verwendet `power_save_mode: light`, was bei ESP32/ESP-IDF dem **WIFI_PS_MIN_MODEM**-Modus entspricht.
- Der ESP32 bleibt mit dem WLAN verbunden.
- **Auto-Light-Sleep wird absichtlich nicht aktiviert**, damit der asynchrone DLMS-UART-Empfang auf GPIO4 nicht während eines Schlafzustands verpasst werden kann.
- Die ESPHome-API bündelt Zustandsänderungen mit `batch_delay: 500ms`, damit die vielen Werte eines DLMS-Telegramms möglichst in wenigen WLAN-Paketen übertragen werden.
- Webserver, API, OTA und die vollständigen Messwerte bleiben verfügbar.

Espressif empfiehlt für ESP32-C3-WLAN-Anwendungen ausdrücklich die Kombination **Min Modem-sleep + DFS**. In den Referenzmessungen von Espressif sinkt der mittlere Chipstrom bei DTIM 1 von rund **21,47 mA ohne DFS auf 11,35 mA mit DFS**. Diese Messwerte sind keine garantierte Leistungsaufnahme des kompletten gPlugK, zeigen aber, weshalb DFS für diese Anwendung sinnvoll ist.

Das originale gPlugK reduziert die CPU ebenfalls aus Energiespargründen und nennt für 160 → 80 MHz eine Einsparung von ungefähr einem Drittel. ESPHome nutzt hier stattdessen DFS: Leistung steht bei Bedarf bis 160 MHz zur Verfügung, während der Takt in Leerlaufphasen automatisch sinkt.

### Warum `power_save_mode: high` nicht Standard ist

ESPHome bietet zusätzlich `high` an. Das entspricht unter ESP-IDF `WIFI_PS_MAX_MODEM` und kann noch stärker sparen, erhöht aber die Latenz und kann Broadcast-/Multicast-Pakete verpassen. ESPHome weist zudem darauf hin, dass der höchste Energiesparmodus die WLAN-Zuverlässigkeit verschlechtern kann. Für einen dauerhaft erreichbaren Home-Assistant-Sensor ist `light` deshalb die robustere Grundeinstellung.

### Warum die WLAN-Sendeleistung nicht fest reduziert wird

Das originale gPlugK-Script nutzt eine dynamische WLAN-Sendeleistung. ESPHome bietet in der YAML dagegen eine feste maximale `output_power`-Vorgabe. Eine pauschale Reduktion wäre im Zählerschrank riskant: Bei schwachem Empfang können Wiederholungen und Reconnects den Energieverbrauch sogar erhöhen. Deshalb bleibt die Sendeleistung automatisch/default.

Wenn der gPlugK am endgültigen Einbauort dauerhaft besser als ungefähr **-60 dBm** empfängt, kann eine reduzierte maximale Sendeleistung später separat getestet werden. Für die Standardkonfiguration ist das nicht erforderlich.

Quellen:

- Kamstrup HAN module data sheet: https://kamstrup-delivery.sitecorecontenthub.cloud/api/public/content/60724-downloadOriginal?v=3794f2c2
- ESPHome WiFi: https://esphome.io/components/wifi/
- Espressif ESP32-C3 Low Power WiFi: https://docs.espressif.com/projects/esp-idf/en/v5.5/esp32c3/api-guides/low-power-mode/low-power-mode-wifi.html

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
ota_password: "MeinOTAPasswort"

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

`0` bedeutet: unbekannt bzw. Sicherungsüberwachung deaktiviert.

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
- `Stromreserve schwaechste Phase`
- `Phasenstrom ueber Sicherungsnennwert`

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
Energie Ruecklieferung Gesamt
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
- `Zaehler Uhr`

---

# 18. Berechnete Werte

| Sensor | Bedeutung |
| --- | --- |
| Netzleistung saldiert | Bezug minus Rücklieferung; positiv = Bezug, negativ = Einspeisung |
| Bezug saldiert | tatsächlich saldierter Bezug |
| Ruecklieferung saldiert | tatsächlich saldierte Einspeisung als positiver Wert |
| Netzleistung L1/L2/L3 | Bezug minus Rücklieferung je Phase |
| Netzleistung aus Phasen | Summe der drei saldierten Phasen |
| Saldierungsabweichung | Differenz Gesamtwert ↔ Summe der Phasen |
| Gleichzeitiger Bezug und Ruecklieferung | möglicher Überlappungswert |
| Stromsumme L1 L2 L3 | Summe aller Phasenströme |
| Hoechster Phasenstrom | aktuell stärkste Phase |
| Phasenstrom Spreizung | höchster minus niedrigster Phasenstrom |
| Scheinleistung Summe U x I | Summe von Spannung × Strom je Phase |
| Hauptsicherung Auslastung | Belastung der stärksten Phase in % |
| Stromreserve schwaechste Phase | Reserve bis zum eingestellten Sicherungswert |
| Hoechstbelastete Phase | L1, L2 oder L3 |

---

# 19. 15-Minuten- und Laufzeitwerte

Im RAM des ESP32 werden zusätzlich geführt:

- Bezugsspitze seit Neustart
- Ruecklieferspitze seit Neustart
- Bezug Mittel 15 Minuten
- Ruecklieferung Mittel 15 Minuten
- Hoechstes 15-Minuten-Mittel seit Neustart
- Empfangene Datensaetze seit Neustart
- Alter der Zaehlerdaten
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
- `Zaehlerdaten aktuell` wird `Aus`,
- die RGB-LED blinkt bei vorhandener WLAN-Verbindung blau.

---

# 21. Diagnosewerte

## WLAN

- WLAN Signal in dBm
- WLAN Signal Prozent
- IP-Adresse
- Verbundenes WLAN / SSID
- WLAN Access Point BSSID
- WLAN MAC-Adresse

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
- Freier Heap
- Minimal freier Heap
- Groesster freier Heap-Block
- Maximale Loop-Zeit
- Letzter Neustartgrund
- ESPHome Version
- ESPHome Verbindung

Die `Chiptemperatur` ist die interne Temperatur des ESP32-C3 und **keine Raumtemperatur**.

Die CPU läuft nicht dauerhaft mit 160 MHz: Durch das aktivierte ESP-IDF-DFS kann sie in Leerlaufphasen bis auf 40 MHz heruntertakten und bei Last automatisch wieder höher takten. Das geschieht transparent; DLMS, API und Weboberfläche müssen dafür nicht abgeschaltet werden.

## Zählerdiagnose

- Datenstatus
- Zaehlerdaten aktuell
- Alter der Zaehlerdaten
- Letztes Empfangsintervall
- Empfangene Datensaetze seit Neustart
- Abdeckung 15-Minuten-Mittel
- Phasenstrom ueber Sicherungsnennwert
- Gleichzeitige Leistungsrichtungen

## Hardwarediagnose

- AP-Taste
- Software Neustart

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
5. wird `Alter der Zaehlerdaten` aktualisiert?

## RGB-LED blinkt rot

Keine lokale WLAN-Verbindung.

Wenn dein WLAN absichtlich geändert wurde oder nicht mehr verfügbar ist:

1. USB-C anschließen,
2. AP-Taster drücken,
3. mit `gplugk-bkw Setup` verbinden,
4. `192.168.4.1` öffnen.

## Grün flackert regelmäßig, aber Home Assistant zeigt nichts

Dann funktioniert der grundsätzliche Datenempfang wahrscheinlich bereits. Prüfe anschließend:

- ESPHome-API-Verbindung
- API-Key
- Home-Assistant-Integration
- ESPHome-Log auf DLMS- oder Entschlüsselungsfehler

## Gerät startet unerwartet neu

Prüfe:

- Letzter Neustartgrund
- Freier Heap
- Minimal freier Heap
- Groesster freier Heap-Block
- Maximale Loop-Zeit
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
- OTA-Passwort
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
ota_password: "MeinOTAPasswort"
web_username: "admin"
web_password: "MeinWebPasswort"
bkw_gpk60: "MEIN_32_STELLIGER_HEX_SCHLUESSEL"
```

Danach kompilieren, per USB installieren und die Startsequenz der RGB-LED beobachten.

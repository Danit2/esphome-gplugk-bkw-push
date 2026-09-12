# gPlugK BKW – ESPHome

ESPHome-Konfiguration für einen **gPlugK HW 2.2** am BKW/Kamstrup-OMNIPOWER-Zähler.

Das Projekt verwendet ausschließlich Funktionen, die direkt in ESPHome vorhanden sind. Es werden **keine zusätzlichen Custom Components** benötigt. Alle Messwerte, berechneten Werte und Diagnosewerte werden über die normale ESPHome-API automatisch an Home Assistant übertragen.

## Was dieses Projekt kann

Die Konfiguration:

- liest das verschlüsselte DLMS-Telegramm des BKW-Zählers über GPIO4,
- entschlüsselt es mit dem BKW-GPK60,
- stellt die Messwerte direkt in Home Assistant bereit,
- berechnet zusätzliche Leistungs-, Strom- und 15-Minuten-Werte,
- überwacht, ob die Zählerdaten noch aktuell sind,
- blinkt bei Datenempfang mit der LED auf GPIO6,
- stellt eine lokale, passwortgeschützte Weboberfläche bereit,
- unterstützt OTA-Updates,
- startet bei WLAN-Problemen einen Fallback-Access-Point,
- stellt zusätzliche WLAN-, ESP32- und Laufzeit-Diagnosewerte bereit.

## Voraussetzungen

Du benötigst:

1. einen gPlugK HW 2.2,
2. einen kompatiblen BKW/Kamstrup-OMNIPOWER-Zähler mit freigeschalteter Kundenschnittstelle,
3. den von BKW erhaltenen **GPK60-Schlüssel**,
4. ein 2,4-GHz-WLAN,
5. ESPHome 2026.8.2 oder neuer.

Falls BKW zusätzlich einen GPK61/Auth-Key bereitstellt, kann dieser optional ebenfalls verwendet werden.

## Dateien im Repository

```text
gplugk-bkw.yaml          Hauptkonfiguration
secrets.example.yaml     Vorlage für persönliche Passwörter und Schlüssel
README.md                Diese Anleitung
LICENSE                  Lizenz
```

Für den normalen Betrieb musst du hauptsächlich nur zwei Dateien beachten:

```text
gplugk-bkw.yaml
secrets.yaml
```

`secrets.yaml` wird von dir lokal erstellt und darf **nicht öffentlich auf GitHub hochgeladen** werden.

---

# Schnellstart

## 1. Projekt herunterladen

Repository herunterladen oder klonen und den Ordner in deine ESPHome-Konfiguration übernehmen.

Danach sollte mindestens Folgendes vorhanden sein:

```text
gplugk-bkw.yaml
secrets.example.yaml
```

## 2. `secrets.yaml` erstellen

Kopiere:

```text
secrets.example.yaml
```

zu:

```text
secrets.yaml
```

Die neue Datei liegt im gleichen Ordner wie `gplugk-bkw.yaml`.

Beispiel:

```yaml
wifi_ssid: "MeinWLAN"
wifi_password: "MeinWLANPasswort"

fallback_password: "MeinFallbackPasswort"

api_key: "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="
ota_password: "MeinOTAPasswort"

web_username: "admin"
web_password: "MeinWebPasswort"

bkw_gpk60: "00112233445566778899AABBCCDDEEFF"
```

Natürlich musst du die Beispielwerte durch deine eigenen Werte ersetzen.

### Wichtig zum GPK60

Der GPK60 ist ein geheimer Schlüssel für deinen Zähler. Er besteht aus **32 Hex-Zeichen**.

Beispiel für das Format:

```text
00112233445566778899AABBCCDDEEFF
```

Veröffentliche deinen echten GPK60 niemals auf GitHub, in Foren oder Screenshots.

## 3. Optional: GPK61 aktivieren

In der Standardkonfiguration wird nur der GPK60 verwendet.

Falls du von BKW zusätzlich einen GPK61/Auth-Key erhalten hast, suche in `gplugk-bkw.yaml` nach:

```yaml
# auth_key: !secret bkw_gpk61
```

und entferne das `#`:

```yaml
auth_key: !secret bkw_gpk61
```

Ergänze anschließend in `secrets.yaml`:

```yaml
bkw_gpk61: "FFEEDDCCBBAA99887766554433221100"
```

Auch hier muss natürlich dein echter Schlüssel eingetragen werden.

## 4. Wandlerfaktor einstellen

Ganz oben in `gplugk-bkw.yaml` findest du:

```yaml
ct_factor: "1.0"
```

### Normaler Direktmesszähler

Bei einem normalen Haushaltszähler bleibt:

```yaml
ct_factor: "1.0"
```

### Wandlerzähler

Bei einem Wandlerzähler trägst du den von BKW angegebenen Ablese-/Wandlerfaktor ein.

Beispiel:

```yaml
ct_factor: "20.0"
```

Der Faktor wird automatisch auf Energie, Strom und Leistung angewendet. Spannung und Leistungsfaktor werden nicht damit multipliziert.

## 5. Hauptsicherung einstellen

Ebenfalls oben in der YAML steht:

```yaml
main_fuse_a: "0"
```

Wenn du deine Hauptsicherung kennst, trage den Nennstrom **pro Phase** ein.

Beispiel für 25 A:

```yaml
main_fuse_a: "25"
```

Dadurch funktionieren zusätzlich:

- `Hauptsicherung Auslastung`
- `Stromreserve schwaechste Phase`
- `Phasenstrom ueber Sicherungsnennwert`

Kennst du den Wert nicht, kannst du `0` stehen lassen. Alle anderen Zählerfunktionen funktionieren trotzdem.

## 6. Gerät kompilieren und installieren

Mit ESPHome CLI:

```bash
esphome run gplugk-bkw.yaml
```

Oder im ESPHome Dashboard die YAML importieren/öffnen und **Install** wählen.

Beim ersten Flashen ist USB am einfachsten. Danach können Updates per OTA eingespielt werden.

---

# Home Assistant

Du brauchst **keine zusätzliche Home-Assistant-YAML**.

Sobald der gPlugK über die ESPHome-Integration zu Home Assistant hinzugefügt wurde, erscheinen die Sensoren automatisch als Entitäten.

Falls Home Assistant das Gerät nicht automatisch entdeckt:

1. **Einstellungen** öffnen.
2. **Geräte & Dienste** öffnen.
3. **Integration hinzufügen** wählen.
4. **ESPHome** auswählen.
5. IP-Adresse des gPlugK eingeben.
6. Falls verlangt, den API-Schlüssel aus `secrets.yaml` verwenden.

## Energy Dashboard

Für das Home-Assistant-Energy-Dashboard sind insbesondere diese beiden Sensoren vorgesehen:

- `Energie Bezug Gesamt`
- `Energie Ruecklieferung Gesamt`

Sie werden als Energiezähler in `kWh` mit `total_increasing` bereitgestellt.

---

# Direkte Messwerte des Zählers

Die YAML liest 30 numerische DLMS-Werte direkt aus dem BKW-Zähler.

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

## Leistung pro Phase

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

Zusätzlich werden die `OBIS Listenkennung` und die `Zaehler Uhr` als Textwerte veröffentlicht.

---

# Berechnete Werte

ESPHome berechnet aus den direkten Zählerdaten zusätzliche Werte.

| Sensor | Bedeutung |
| --- | --- |
| `Netzleistung saldiert` | Bezug minus Rücklieferung; positiv = Bezug, negativ = Einspeisung |
| `Bezug saldiert` | tatsächlich saldierter Bezug |
| `Ruecklieferung saldiert` | tatsächlich saldierte Rücklieferung als positiver Wert |
| `Netzleistung L1` | Bezug L1 minus Rücklieferung L1 |
| `Netzleistung L2` | Bezug L2 minus Rücklieferung L2 |
| `Netzleistung L3` | Bezug L3 minus Rücklieferung L3 |
| `Netzleistung aus Phasen` | Summe der drei saldierten Phasen |
| `Saldierungsabweichung` | Differenz zwischen Gesamtwert und Summe der Phasen |
| `Gleichzeitiger Bezug und Ruecklieferung` | zeigt überlappenden Bezug/Rücklieferung an |
| `Stromsumme L1 L2 L3` | Summe aller drei Phasenströme |
| `Hoechster Phasenstrom` | höchster aktueller Strom einer Phase |
| `Phasenstrom Spreizung` | Differenz zwischen höchstem und niedrigstem Phasenstrom |
| `Scheinleistung Summe U x I` | Summe von U × I aller drei Phasen |
| `Hauptsicherung Auslastung` | Auslastung der stärkst belasteten Phase in Prozent |
| `Stromreserve schwaechste Phase` | verbleibende Stromreserve bis zur eingestellten Hauptsicherung |
| `Hoechstbelastete Phase` | L1, L2 oder L3 |

---

# 15-Minuten- und Laufzeitwerte

Die YAML führt einige Werte nur im RAM des ESP32 mit:

- `Bezugsspitze seit Neustart`
- `Ruecklieferspitze seit Neustart`
- `Bezug Mittel 15 Minuten`
- `Ruecklieferung Mittel 15 Minuten`
- `Hoechstes 15-Minuten-Mittel seit Neustart`
- `Empfangene Datensaetze seit Neustart`
- `Alter der Zaehlerdaten`
- `Letztes Empfangsintervall`
- `Abdeckung 15-Minuten-Mittel`

Das 15-Minuten-Mittel ist **zeitgewichtet**. Es berücksichtigt also die tatsächlich vergangene Zeit zwischen den empfangenen Telegrammen.

Nach einem Neustart beginnen diese Laufzeitwerte wieder von vorne. Sie werden bewusst nicht ständig in den Flash geschrieben.

---

# Prüfung, ob Zählerdaten aktuell sind

Standardmäßig gilt ein Messwert nach **35 Sekunden ohne neue Zählerdaten** als veraltet.

Oben in der YAML findest du:

```yaml
stale_after: "35s"
stale_after_ms: "35000"
```

Normalerweise musst du diese Werte nicht ändern.

Wenn keine neuen Daten mehr kommen:

- direkte Messsensoren wechseln auf `NaN`/nicht verfügbar,
- `Datenstatus` wechselt auf `Daten veraltet`,
- `Zaehlerdaten aktuell` wird ausgeschaltet.

Dadurch zeigt Home Assistant nicht minuten- oder stundenlang einen alten Messwert als aktuell an.

---

# Diagnosewerte

Die aktuelle YAML enthält zusätzlich Diagnosewerte für den gPlugK selbst.

## WLAN

### `WLAN Signal`

Signalstärke in dBm.

Als grobe Orientierung:

| dBm | Einschätzung |
| ---: | --- |
| -30 bis -50 | sehr gut |
| ungefähr -60 | gut |
| ungefähr -70 | noch brauchbar |
| unter -75 | eher schwach |
| unter -85 | Verbindung kann instabil werden |

Bei dBm gilt: **je näher an 0, desto besser**.

### `WLAN Signal Prozent`

Zusätzliche, leichter lesbare Darstellung von 0 bis 100 %. Der Prozentwert ist eine Näherung aus dem dBm-Wert.

### WLAN-Informationen

Zusätzlich stehen bereit:

- `IP-Adresse`
- `Verbundenes WLAN`
- `WLAN Access Point BSSID`
- `WLAN MAC-Adresse`

Die BSSID ist besonders praktisch, wenn mehrere Access Points dieselbe SSID verwenden. So sieht man, mit welchem Access Point der gPlugK tatsächlich verbunden ist.

## ESP32

### `Chiptemperatur`

Zeigt die interne Temperatur des ESP32-C3.

**Wichtig:** Das ist keine Raumtemperatur. Die interne Temperatur kann deutlich über der Umgebungstemperatur liegen und eignet sich hauptsächlich dazu, ungewöhnliche Erwärmung oder Veränderungen zu erkennen.

### `Laufzeit`

Zeit seit dem letzten Neustart des gPlugK.

### `Freier Heap`

Aktuell noch verfügbarer Arbeitsspeicher des ESP32.

### `Minimal freier Heap`

Der kleinste freie Arbeitsspeicher, der seit dem letzten Start beobachtet wurde. Dieser Wert ist hilfreich, um knappe RAM-Situationen zu erkennen.

### `Groesster freier Heap-Block`

Größe des größten zusammenhängenden freien Speicherblocks. Ein großer Gesamt-Heap hilft wenig, wenn der Speicher stark zerstückelt ist; dieser Wert hilft deshalb bei der Fehlersuche.

### `Maximale Loop-Zeit`

Zeigt die längste gemessene Laufzeit einer ESPHome-Hauptschleife. Ungewöhnlich hohe Werte können auf blockierende Verarbeitung hinweisen.

### `Letzter Neustartgrund`

Zeigt, weshalb der ESP32 zuletzt gestartet oder neu gestartet wurde. Das ist beispielsweise bei unerwarteten Reboots hilfreich.

### `ESPHome Version`

Zeigt die ESPHome-Version, mit der die aktuell installierte Firmware erstellt wurde.

### `ESPHome Verbindung`

Zeigt, ob das Gerät aktuell über die native ESPHome-API verbunden ist.

---

# Zählerdiagnose

Zusätzlich gibt es Diagnosewerte speziell für den Zählerempfang:

- `Datenstatus`
- `Zaehlerdaten aktuell`
- `Alter der Zaehlerdaten`
- `Letztes Empfangsintervall`
- `Empfangene Datensaetze seit Neustart`
- `Abdeckung 15-Minuten-Mittel`
- `Phasenstrom ueber Sicherungsnennwert`
- `Gleichzeitige Leistungsrichtungen`

`Datenstatus` kann beispielsweise anzeigen:

```text
Warte auf Zaehlerdaten
OK
Daten veraltet
```

---

# Empfangs-LED

GPIO6 wird als Daten-LED verwendet.

Wenn ein neues Zählertelegramm empfangen wird, leuchtet die LED ungefähr eine Sekunde.

Dadurch kann direkt am Gerät geprüft werden, ob regelmäßig Daten vom Zähler ankommen.

---

# Lokale Weboberfläche

Der gPlugK stellt zusätzlich eine lokale ESPHome-Weboberfläche bereit.

Rufe im Browser die IP-Adresse des Geräts auf, zum Beispiel:

```text
http://192.168.1.123/
```

Der Zugriff ist mit diesen Werten aus `secrets.yaml` geschützt:

```yaml
web_username: "admin"
web_password: "MeinWebPasswort"
```

Die Weboberfläche sortiert die Entitäten in:

- Netzanschluss
- Phasen
- Zählerstände
- Berechnete Werte
- Zählerinformationen
- Diagnose

---

# WLAN-Fallback

Kann sich der gPlugK ungefähr 90 Sekunden lang nicht mit dem eingetragenen WLAN verbinden, startet er einen eigenen Access Point:

```text
gplugk-bkw Fallback
```

Das Passwort ist der Wert:

```yaml
fallback_password: "..."
```

Verbinde dich mit diesem WLAN. Über das Captive Portal kann die WLAN-Konfiguration wieder erreicht werden.

---

# Typische Anpassungen

Für die meisten Installationen müssen nur diese Werte angepasst werden.

## Gerätenamen ändern

Oben in `gplugk-bkw.yaml`:

```yaml
device_name: gplugk-bkw
friendly_name: gPlugK
```

Beispiel:

```yaml
device_name: stromzaehler-keller
friendly_name: Stromzaehler Keller
```

`device_name` sollte nur Kleinbuchstaben, Zahlen und Bindestriche enthalten.

## 25-A-Hauptsicherung

```yaml
main_fuse_a: "25"
```

## 40-A-Hauptsicherung

```yaml
main_fuse_a: "40"
```

## Normaler Direktmesszähler

```yaml
ct_factor: "1.0"
```

---

# Skalierung der BKW-Werte

Die Konfiguration rechnet die vom Zähler gelieferten Rohwerte automatisch um:

- Energie: × `0.01` → kWh bzw. kvarh
- Strom: × `0.01` → A
- Leistungsfaktor: × `0.01`
- Leistung: W bzw. var
- Spannung: V
- Wandlerfaktor `ct_factor`: zusätzlich auf Energie, Strom und Leistung

Du musst diese Skalierungen normalerweise nicht verändern.

---

# Fehlersuche

## Es erscheinen gar keine Zählerwerte

Prüfe zuerst:

1. Ist der richtige GPK60 eingetragen?
2. Ist die Kundenschnittstelle des Zählers aktiviert?
3. Blinkt die Empfangs-LED regelmäßig?
4. Was zeigt `Datenstatus` an?
5. Was steht im ESPHome-Log?

## WLAN ist instabil

Prüfe:

- `WLAN Signal`
- `WLAN Signal Prozent`
- `Verbundenes WLAN`
- `WLAN Access Point BSSID`

Ein dauerhaft sehr schwaches Signal kann zu Verbindungsproblemen führen.

## Gerät startet unerwartet neu

Prüfe:

- `Letzter Neustartgrund`
- `Freier Heap`
- `Minimal freier Heap`
- `Groesster freier Heap-Block`
- `Maximale Loop-Zeit`
- `Laufzeit`

## Messwerte werden plötzlich nicht mehr aktualisiert

Prüfe:

- `Datenstatus`
- `Zaehlerdaten aktuell`
- `Alter der Zaehlerdaten`
- `Letztes Empfangsintervall`

---

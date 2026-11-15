# wMBus Stromzähler mit Heltec V4.3 und ESPHome

ESPHome-Konfiguration, die die verschlüsselten wMBus-Telegramme eines Stromzählers empfängt und

- die Werte an **Home Assistant** (native API) liefert,
- optional ein **JSON per MQTT** sendet (angelehnt an das Format des SMA-Adapters von Österreichs Energie),
- die **Leistung der Viertelstunden** und das **Monatsmaximum selbst berechnet**, auch wenn der Zähler kein Maximum sendet,
- die wichtigsten Werte auf dem **OLED-Display** des Heltec-Boards anzeigt.

Die Lösung läuft mit Home Assistant, mit Home Assistant und MQTT sowie ganz ohne Home Assistant nur mit MQTT.

## Wichtiger Hinweis

> Dies ist ein Community-Projekt und **keine offizielle Lösung** eines Netzbetreibers oder Zählerherstellers.
> Die Bereitstellung erfolgt **ohne Gewähr**. Die Nutzung geschieht auf **eigene Verantwortung**.
> Die berechneten Leistungswerte sind **Näherungswerte** und keine Abrechnungsgrößen.
> Maßgeblich sind immer die Daten des Netzbetreibers.

## Getestet mit

| Komponente | Stand |
|---|---|
| Board | Heltec WiFi LoRa 32 V4.3 (HTIT-WB32LAF) |
| ESPHome | 2026.10.2 |
| Zähler | NES 833334 wMBus-Zähler mit dem Treiber `amiplus` der wmbusmeters, Modus T1 |
| wMBus-Komponente | [SzczepanLeon/esphome-components](https://github.com/SzczepanLeon/esphome-components), Commit `56abb0055e24bb247c5d3a2a45c9140226e01ac0` |


Andere Zählertypen sind nicht getestet. Die Feldnamen der Sensoren (`total_energy_consumption_kwh` usw.) gelten für den Treiber `amiplus` und können bei anderen Treibern anders heißen.

## Voraussetzungen

- Heltec WiFi LoRa 32 V4.3 mit angeschlossener Antenne.
- Ein wMBus-Zähler und dessen **AES-Schlüssel** (den bekommst du in der Regel von deinem Netzbetreiber).
- Das Telegramm des Zählers muss **Zählerstand und Zählerzeit** (`device_date_time`) enthalten. Ohne Zählerzeit funktioniert die Berechnung des Maximums nicht.
- ESPHome (Dashboard oder Kommandozeile).

## Belegung des Boards

| Funktion | GPIO |
|---|---|
| SPI CLK / MOSI / MISO | 9 / 10 / 11 |
| SX1262 CS / Reset / DIO1 / Busy | 8 / 12 / 14 / 13 |
| Front-End (GC1109): Versorgung / Enable / TX-RX | 7 / 2 / 46 (LOW = Empfang) |
| OLED: SDA / SCL / Reset | 17 / 18 / 21 |
| Vext (Display-Versorgung, invertiert) | 36 |

Die Belegung des Front-Ends und der Display-Versorgung stammt aus Community-Quellen und wurde nur an einem Board getestet. Ohne aktives Front-End empfängt das Board kaum etwas.

## Einrichtung

1. **Secrets anlegen.** Kopiere `secrets.yaml.example` nach `secrets.yaml` und trage ein: `wifi_ssid`, `wifi_password`, `mqtt_user`, `mqtt_password` und `wmbus_key` (32 Hex-Zeichen). Den API-Key erzeugst du neu, am besten mit `openssl rand -base64 32`.
2. **Substitutionen anpassen.** Am Anfang der YAML stehen `mqtt_broker`, `mqtt_topic`, `zaehler_id`, `max_luecke_s` und `min_laufend_s`.
3. **Zählernummer ermitteln.** Kennst du die Nummer nicht, lass `0x00000000` stehen und setze das Log-Level kurz auf `WARN`. Im Log erscheint eine Meldung der Form `Check if telegram with address 00123456 can be parsed on ...`. Die Adresse ist die Zählernummer (hexadezimal, in der YAML mit `0x` davor).
4. **Flashen.** Beim ersten Mal per USB-Kabel, später per OTA.
5. **Prüfen.** Im Log (Level `INFO` oder `DEBUG`) siehst du empfangene Telegramme mit RSSI. Danach erscheinen die Sensoren in Home Assistant bzw. im Web-Interface des Geräts.

Der erste berechnete Viertelstundenwert erscheint nach 15 bis 30 Minuten, das Monatsmaximum mit der ersten vollständigen Viertelstunde.

## Betriebsarten

| Fall | Was zu tun ist |
|---|---|
| Home Assistant und MQTT | Datei unverändert verwenden. Die Sensoren kommen über die API in Home Assistant, das JSON geht zusätzlich an MQTT. |
| Nur MQTT, ohne Home Assistant | Datei unverändert verwenden. Der `api:`-Block hat `reboot_timeout: 0s`, das Gerät startet also ohne Home Assistant nicht neu. Du kannst `api:` auch entfernen. |
| Nur Home Assistant, ohne MQTT-Broker | Den Block `mqtt:` und die Aktion `mqtt.publish` im Sensor `gesamtverbrauch` entfernen. |

Die Standard-Topics von ESPHome sind abgeschaltet. Über MQTT kommt nur das JSON (siehe unten). Empfangsstärke und Zählerzeit stehen nur in Home Assistant, im Web-Interface und auf dem Display.

## JSON-Telegramm

Das Telegramm wird mit jedem Zähler-Telegramm (also im Sendeintervall des Zählers) auf das konfigurierte Topic (`mqtt_topic`) gesendet.

| Schlüssel | Bedeutung | Einheit |
|---|---|---|
| `1-0:1.8.0` | Gesamtverbrauch (Bezug) | Wh |
| `1-0:1.7.0` | aktuelle Leistung (Bezug) | W |
| `1-0:2.8.0` | Gesamteinspeisung | Wh |
| `1-0:2.7.0` | aktuelle Leistung (Einspeisung) | W |
| `1-0:1.6.0` | Monatsmaximum der Viertelstundenleistung (**berechnet**), `time` = Zählerzeit am Ende der Viertelstunde, in der das Maximum entstand | W |
| `1-0:1.4.0` | Leistung der laufenden Viertelstunde (**berechnet**) | W |
| `1-0:1.5.0` | Leistung der letzten Viertelstunde (**berechnet**) | W |
| `api_version`, `name` | fest `v1` und leer | |
| `sma_time` | Laufzeit des ESP in Sekunden | s |
| `telegramNr` | fortlaufende Nummer seit dem Start | |

Beispiel:

```json
{"1-0:1.8.0":{"value":4249143,"time":0},"1-0:1.7.0":{"value":715,"time":0},"1-0:2.8.0":{"value":181084,"time":0},"1-0:2.7.0":{"value":0,"time":0},"1-0:1.6.0":{"value":1864,"time":1794659400},"1-0:1.4.0":{"value":715,"time":0},"1-0:1.5.0":{"value":690,"time":0},"api_version":"v1","name":"","sma_time":387680.579,"telegramNr":385482}
```

Hinweise zum Format:

- `time` bei `1-0:1.6.0` ist die **Zählerzeit als lokale Sekundenzahl seit 1970**, ohne Umrechnung der Zeitzone. Stellt ein Empfänger den Wert als UTC dar, erscheint die Uhrzeit des Zählers (in Österreich MEZ/MESZ).
- Solange noch kein Wert berechnet wurde, steht 0 im Feld.
- Die berechneten Felder können um ein Telegramm hinterherhängen, weil Zählerstand und Zählerzeit getrennt ankommen.
- Der Empfänger sollte `1-0:1.6.0` als Zustand lesen und nicht als Ereignis.

## So werden die Viertelstundenwerte berechnet

Das Maximum entsteht ausschließlich aus Zählerstand und Zählerzeit des Telegramms, die ESP-Zeit wird nicht verwendet.

1. Das Raster sind die vollen Viertelstunden der **Zählerzeit**.
2. Zwischen dem letzten Telegramm vor und dem ersten Telegramm nach einer Viertelstunden-Grenze wird der Zählerstand **linear interpoliert**. Das gilt nur, wenn die beiden Telegramme höchstens `max_luecke_s` Sekunden auseinanderliegen.
3. Die Leistung der Viertelstunde ist die Differenz der interpolierten Zählerstände an zwei aufeinanderfolgenden Grenzen mal 4.
4. Das Monatsmaximum ist der höchste dieser Werte. Es wird im Flash gesichert und übersteht Neustarts.

Einstellungen:

| Substitution | Bedeutung |
|---|---|
| `max_luecke_s` | Größter erlaubter Abstand der beiden Telegramme für die Interpolation. Muss größer als das Sendeintervall des Zählers sein (z. B. 20 bei 5 s). Kleiner ist genauer, größer verträgt mehr verlorene Telegramme. |
| `min_laufend_s` | Die laufende Viertelstunde wird erst nach so vielen Sekunden angezeigt, weil der Zählerstand nur in ganzen Wh vorliegt und der Wert zu Beginn stark springt. |

## Bekannte Einschränkungen

- **Auflösung:** Der Zählerstand wird in ganzen Wh gerundet, daher liegt die Auflösung der Viertelstundenleistung bei etwa 4 W. Bei einem Vergleich mit dem Maximum des Zählers (1860 W) zeigte die Berechnung 1864 W.
- **Nur Bezug, nur Kalendermonat.** Die Einspeisung und Tarifzeiten werden nicht berücksichtigt.
- **Lücken:** Fehlen Telegramme über eine Viertelstunden-Grenze hinweg, wird diese Grenze nicht ausgewertet, und es fallen zwei Viertelstunden aus.
- **Monatswechsel:** Die Viertelstunde 23:45 bis 00:00 zählt zum alten Monat. Das Maximum wird erst nach der ersten abgeschlossenen Viertelstunde des neuen Monats zurückgesetzt, also bis zu 15 Minuten nach dem Wechsel. Ein Zähler stellt sein Maximum dagegen sofort zurück.
- **Zeitumstellung:** Springt die Zählerzeit, wird ein Intervall übersprungen. Danach läuft die Berechnung normal weiter.
- **Zeitformat:** Das Skript erwartet die Zählerzeit als `JJJJ-MM-TT hh:mm:ss`.

## Fehlersuche

| Problem | Ursache und Abhilfe |
|---|---|
| `Component not found: logger` beim Kompilieren | Die externe Komponente bringt ein eigenes `esp32`-Verzeichnis mit. Importiere nur `wmbus_radio`, `wmbus_meter`, `wmbus_common` (`components:`-Liste), so wie in der Datei. |
| Es kommt kein Telegramm an | Front-End aktiv? (GPIO 7 und 2 an, GPIO 46 aus), Antenne angeschlossen, Modus (`T1`/`C1`) passt zum Zähler. |
| `Telegram not handled by any handler` | Zählernummer in `zaehler_id` stimmt nicht. Die Adresse steht in der Meldung. |
| Telegramm kommt, aber keine Werte | AES-Key falsch, oder der Zählertyp passt nicht zum Treiber. |
| `Zaehlerzeit nicht lesbar` | Zähler sendet `device_date_time` in anderem Format oder gar nicht. Log-Level mindestens `WARN`. |
| Gerät erscheint nicht in Home Assistant | Eintrag wurde früher ignoriert: Einstellungen, ESPHome, „Ignorierte anzeigen". Oder IP und API-Key manuell eintragen. |
| Gerät startet alle 15 Minuten neu | `reboot_timeout: 0s` bei `api`, `wifi` und `mqtt` fehlt. |
| Aktivität der Zählerzeit in Home Assistant lädt langsam | Der Sensor schreibt jede Sekunde einen Zustand. Mit `recorder: exclude` in configuration.yaml für diese Entität verschwinden die gespeicherten Einträge. |     
| Display bleibt dunkel | Vext (GPIO 36, invertiert) und I2C-Scan im Log prüfen. Erwartete Adresse `0x3C`. |

## Sicherheit

- Der AES-Schlüssel liegt im Klartext im Flash des Geräts. Bei Geräten in fremder Hand solltest du die Flash-Verschlüsselung prüfen.
- Das Web-Interface hat standardmäßig keinen Login. Setze `auth:` mit Benutzer und Passwort.
- OTA-Updates sollten ein Passwort haben. Ohne Home Assistant kannst du zusätzlich `ota: - platform: web_server` verwenden.
- Veröffentliche niemals `secrets.yaml`, den API-Key oder den AES-Schlüssel.

## Hinweise zur Wartung

- Die externe Komponente ist auf einen Commit festgenagelt (`refresh: never`). Aktualisiere sie bewusst und teste danach, denn neuere Stände können mit der ESPHome-Version brechen.

## Lizenz

MIT, siehe LICENSE. Beachte die Lizenz der eingebundenen Komponente.

## Danksagung

- [SzczepanLeon/esphome-components](https://github.com/SzczepanLeon/esphome-components) für die wM-Bus-Komponenten
- [wmbusmeters](https://wmbusmeters.org) für die Telegramm-Treiber
- [ESPHome](https://esphome.io)


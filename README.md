# YAIFM - IKEA Fridans Rollo Motorisierung

ESPHome Firmware für motorisierte IKEA Fridans Raffstores mit ESP32-WROOM-32D.

## Features

- **Kein Endstop** — beide Richtungen stoppen über Encoder-Werte + Stall-Detection
- **Positionsspeicherung im NVS** — Kalibrierung und Referenz überleben Stromausfall
- **Sicherheits-Sperre** — Schließen blockiert wenn Referenz ungültig (kein Endlosfahrt)
- **Home Assistant Integration** — Cover Entity mit Positionsanzeige (0-100%)
- **Konfigurierbar über HA** — Schritte bis geschlossen, Motorgeschwindigkeit
- **Auto-Level** — beim ersten Boot automatisch hoch bis Stall, Referenz wird gesetzt
- **Stall-Detection** — 250ms keine Encoder-Bewegung = Motor stoppt
- **Slow-Down** — 25% Speed im letzten Tick-Bereich (Schonung des Mechanismus)

## Hardware

| Komponente | Spezifikation |
|------------|---------------|
| MCU | ESP32-WROOM-32D |
| Motortreiber | DRV8833 H-Bridge |
| Motor | GA12-N20 DC mit Rotary Encoder (39RPM @ 6V) |
| Stromversorgung | 5V USB-C |

### Pinout

```
ESP32 GPIO    ->  DRV8833 / Motor
GPIO14        ->  DRV8833 IN1 (Motor forward PWM)
GPIO27        ->  DRV8833 IN2 (Motor backward PWM)
GPIO16        ->  Encoder C1 (Phase A)
GPIO4         ->  Encoder C2 (Phase B)
GPIO26        ->  DRV8833 EEP (Sleep/Enable)
3.3V          ->  Encoder VCC
GND           ->  Encoder GND + DRV8833 GND
```

## Installation

1. **secrets.yaml anlegen:**
   ```bash
   cp secrets.yaml.example secrets.yaml
   ```
   Trage deine WLAN-Daten und generierte Keys ein:
   ```bash
   esphome generate-key  # für api_encryption_key
   ```

2. **Flashen:**
   ```bash
   esphome run yaifm-rollo.yaml
   ```

3. **Einrichtung (Erststart):**
   - Beim ersten Boot läuft Auto-Level automatisch (fährt hoch bis Stall)
   - Rollo manuell runterfahren bis es stoppt
   - In Home Assistant: Button "Position als Unten speichern" drücken
   - Fertig — Rollo ist einsatzbereit

## Home Assistant Entities

| Entity | Typ | Beschreibung |
|--------|-----|--------------|
| `cover.rollo` | Cover | Position 0-100%, Open/Close/Stop/Position |
| `number.schritte_bis_ganz_geschlossen` | Number | Encoder-Schritte für komplettes Schließen |
| `number.motorgeschwindigkeit` | Number | Motor-PWM 20-100% |
| `button.auto_level` | Button | Referenz neu kalibrieren |
| `button.position_als_unten_speichern` | Button | Unten-Position speichern |
| `button.test_motor_hoch` | Button | Test: 2s hochfahren |
| `button.test_motor_runter` | Button | Test: 2s runterfahren |
| `button.not_aus` | Button | Motor sofort stoppen + Driver aus |
| `button.neustart` | Button | ESP32 neustarten |
| `sensor.encoder_position` | Sensor | Roher Encoder-Wert |
| `sensor.initialisiert` | Sensor | True/False — Referenz gültig? |

## Sicherheitsfeatures

- **Schließen blockiert** wenn Referenz ungültig (`blind_encoder_abs_open_fully == -999999999`)
- **Position blockiert** wenn Referenz ungültig
- **Bewegung blockiert** wenn Kalibrierung 0 oder negativ (NVS korrupt)
- **Öffnen immer erlaubt** — oben ist die sichere Position
- **NOT-AUS** schaltet Motor + Driver sofort aus

## Stromausfall-Verhalten

- Kalibrierung (`blind_encoder_rel_closed_fully`) → in NVS gespeichert (überlebt)
- Referenz oben (`blind_encoder_abs_open_fully`) → in NVS gespeichert (überlebt)
- Letzte Position (`blind_encoder_rel_current`) → in NVS gespeichert (überlebt)
- Nach Reboot: wenn Referenz gültig → Auto-Level wird übersprungen, Werte synchronisiert
- Nach Reboot: wenn Referenz ungültig → Auto-Level läuft automatisch

## Lizenz

GPL-3.0 — siehe [LICENSE](LICENSE)

## Danksagung

Dies ist ein Fork/Neuimplementierung basierend auf zwei Projekten:

- **[AndBu/YAIFM](https://github.com/AndBu/YAIFM)** — Das Original-Projekt. Verwendet ESP8266 + Endstop-Schalter für die Referenzierung. Wir haben das Konzept übernommen, aber auf ESP32 portiert und den Endstop komplett entfernt.
- **[ned14/YAIFM](https://github.com/ned14/YAIFM)** — Unsere Architektur-Vorlage. Diese Version kommt bereits ohne Endstop aus und nutzt reine Encoder-basierte Stall-Detection für beide Richtungen. Die gesamte Bewegungslogik (`do_position_blind` Script, Auto-Level, Slow-Down) basiert auf ned14's Arbeit.

### Unterschiede zum Original (AndBu)
| Feature | AndBu (Original) | Diese Version |
|---------|-----------------|---------------|
| MCU | ESP8266 (Wemos D1 Mini) | ESP32-WROOM-32D |
| Endstop | Creality Ender 3 Schalter (Pflicht) | Keiner — Stall-Detection |
| Referenzierung | Endstop oben | Auto-Level: hoch bis Stall |
| Positionsspeicherung | Neu kalibrieren nach Reboot | NVS (Flash) — überlebt Stromausfall |
| Framework | Arduino | ESPHome (ESP-IDF) |
| HA Integration | Cover (basic) | Cover mit Position 0-100% + Buttons |
| Sicherheits-Sperre | Keine | Schließen blockiert wenn Referenz ungültig |

### Unterschiede zur Vorlage (ned14)
| Feature | ned14 | Diese Version |
|---------|-------|---------------|
| NVS-Persistenz | Encoder-Wert | Encoder-Wert + Referenz + Kalibrierung |
| Boot-Verhalten | Immer Auto-Level | Auto-Level nur bei ungültiger Referenz |
| Close-Sperre | Keine | Blockiert wenn Referenz ungültig |
| HA Toleranz | 0.1% | 2% (verhindert falsche "offen" Anzeige) |
| EEP Driver | Nicht verwendet | DRV8833 Sleep/Enable Pin (GPIO26) |
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
| MCU | ESP32 |
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

### Verdrahtung (komplette Anschluss-Anleitung)

#### Benötigte Bauteile

| Bauteil | Menge | Hinweis |
|---------|-------|---------|
| ESP32 DevKit | 1 | Beliebiges ESP32 Board mit GPIO-Pins |
| DRV8833 H-Bridge | 1 | Motortreiber, 2x H-Brücke |
| GA12-N20 DC Motor mit Encoder | 1 | 39RPM @ 6V, mit Rotary Encoder |
| USB-C Kabel | 1 | 5V Stromversorgung für ESP32 |
| Dupont-Kabel (weiblich-weiblich) | ~10 | Für Verbindungen |
| IKEA Fridans Raffstore | 1 | Das zu motorisierende Rollo |

#### Schritt-für-Schritt Verkabelung

**1. ESP32 → DRV8833 (Motortreiber)**

```
ESP32 Pin     ->  DRV8833 Pin
GPIO14        ->  IN1 (Motor Vorwärts / Runter)
GPIO27        ->  IN2 (Motor Rückwärts / Hoch)
GPIO26        ->  EEP (Sleep/Enable — Driver aktivieren/deaktivieren)
3.3V          ->  VINT (Logik-Spannung, falls vorhanden)
GND           ->  GND
```

**2. DRV8833 → Motor**

```
DRV8833 Pin   ->  Motor Pin
OUT1          ->  Motor Klemme A (Encoder-Seite)
OUT2          ->  Motor Klemme B
VMOT (VIN)    ->  5V (vom ESP32 5V Pin oder USB)
GND           ->  GND (gemeinsam mit ESP32)
```

**3. ESP32 → Encoder (Motor-Encoder)**

```
ESP32 Pin     ->  Encoder Pin
3.3V          ->  VCC (Encoder Stromversorgung)
GND           ->  GND (Encoder Masse)
GPIO4         ->  C2 (Phase B — gelb/weiß)
GPIO16        ->  C1 (Phase A — grün/blau)
```

GPIO4 und GPIO16 haben interne Pull-Up-Widerstände aktiviert (`pullup: true` in der YAML). Externe Pull-Ups sind nicht erforderlich.

**4. Stromversorgung**

```
USB-C Kabel   ->  ESP32 (5V, mindestens 1A)
ESP32 5V Pin  ->  DRV8833 VMOT/VIN (Motor-Strom)
ESP32 3.3V    ->  Encoder VCC
GND           ->  Gemeinsame Masse für alle Komponenten
```

> ⚠️ **Wichtig:** Alle GND-Pins müssen miteinander verbunden sein (ESP32 GND = DRV8833 GND = Encoder GND). Eine gemeinsame Masse ist zwingend erforderlich, sonst funktioniert die Encoder-Abfrage nicht zuverlässig.

**5. Motor im Rollo einbauen**

Der GA12-N20 Motor wird in das IKEA Fridans Rollo-Rohr eingesetzt. Die originale manuelle Kurbel wird durch den Motor ersetzt. Der Encoder sitzt direkt am Motor und erfasst jede Umdrehung.

#### Warum der EEP-Pin (Sleep/Enable) — und nicht einfach IN1/IN2 direkt

Der DRV8833 hat einen EEP-Pin (Enable/Sleep). Wenn EEP LOW ist, ist der Treiber **komplett deaktiviert** — beide H-Brücken-Ausgänge gehen auf High-Z (hochohmig), der Motor bekommt keinen Strom.

Wir verwenden GPIO26 als EEP mit `restore_mode: ALWAYS_OFF`. Das bedeutet:

1. **Beim Booten ist der Treiber sofort OFF** — der ESP32 bootet, GPIO26 ist LOW, der DRV8833 schläft. Erst nach 2 Sekunden (bewusst verzögert) wird der Driver aktiviert.
2. **PWM-Ausgänge (IN1/IN2) können beliebige Zustände haben** — solange EEP LOW ist, passiert nichts. Der Motor bewegt sich nicht.

> ⚠️ **Das Problem das wir hatten:** Ohne EEP hatten die PWM-Pins (GPIO14, GPIO27) beim Booten kurzzeitig undefinierte Zustände. Der DRV8833 interpretierte das als Ansteuerung und der Motor fuhr **Vollgas in eine zufällige Richtung** — unkontrolliert, bis ESPHome fertig initialisiert war. Bei einem Rollo das oben an der Decke hängt ist das nicht ideal.
>
> **Die Lösung:** EEP-Pin auf GPIO26, `ALWAYS_OFF` beim Boot. Der Treiber ist tot bis wir ihn bewusst aktivieren. Erst wenn `on_boot` durchgelaufen ist (Motor aus → 2s warten → Driver an → 1s warten → Auto-Level), bekommt der Motor Strom. Keine unkontrollierten Fahrten mehr.

Diese Lösung ist zuverlässiger als Software-PWM auf 0 zu setzen, weil der ESP32 während des Boot-Prozesses GPIO-Pins nicht garantiert kontrollieren kann. Hardware-Seitig den Treiber deaktivieren ist die saubere Lösung.

#### Verdrahtungsplan (Übersicht)

```
                    +-----------+
                    |   ESP32   |
                    |           |
  USB-C 5V -------> | 5V    3V3 | ---> Encoder VCC
                    |           |
  DRV8833 IN1 <---- | GPIO14    |
  DRV8833 IN2 <---- | GPIO27    |
  DRV8833 EEP <---- | GPIO26    |
  Encoder C1  <--- | GPIO16    |
  Encoder C2  <--- | GPIO4     |
                    |           |
  GND (alle) <----- | GND       |
                    +-----------+
                         |
                    +-----------+
                    |  DRV8833  |
                    |           |
  ESP32 IN1 -----> | IN1   OUT1|---> Motor A
  ESP32 IN2 -----> | IN2   OUT2|---> Motor B
  ESP32 EEP -----> | EEP       |
  ESP32 5V  -----> | VMOT  GND |---> GND (gemeinsam)
                    +-----------+
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

## Bekannte Einschränkungen

### Positions-Drift bei häufigen Teilfahrten

Bei häufigem Positionieren auf Zwischenpositionen (z.B. 25%, 50%) kann es zu einer **leichten Verschiebung der Endposition** kommen. Ursachen:

- **Slow-Down Stall**: Der Motor fährt im letzten Tick-Bereich mit 25% Speed. Bei manchen Rollos hat der Motor bei 25% nicht genug Kraft → Stall wird fälschlich erkannt → Motor stoppt wenige Ticks vor dem Ziel. Über viele Fahrten summiert sich dieser Fehler.
- **Mechanisches Spiel**: Der Stoff dehnt/spannt sich unterschiedlich je nach Richtung und Position. Der Encoder misst die Motorwelle, nicht die Stoffposition — kleine Abweichungen sind möglich.
- **Stall bei normalem Positionieren**: Wenn der Motor beim Erreichen der Zielposition stalled (z.B. durch Widerstand im Mechanismus), wird die aktuelle Position gespeichert — die kann einige Ticks vom echten Ziel abweichen.

**Abhilfe:**
- Regelmäßiges **Auto-Level** (Button in HA drücken) korrigiert die Referenz neu
- Komplett Öffnen (0%) startet automatisch ein Auto-Level und referenziert neu
- Bei präzisen Anwendungen: Nach mehreren Teilfahrten einmal komplett öffnen und wieder schließen

> ⚠️ In der Praxis (3 Rollos über mehrere Wochen getestet) ist die Drift minimal (<1%) und kaum merklich. Die Chance besteht theoretisch, ist aber selten.

## Technische Erklärung: Motor-Pipsen am Ziel

Wenn der Motor sein Ziel erreicht (besonders beim Schließen = unten), hörst du ein kurzes
Summen/Pipsen. Das ist normal und kein Defekt — hier die Erklärung:

### Was passiert im Code

1. Motor fährt mit voller Geschwindigkeit (80% default) zum Ziel
2. Innerhalb der letzten 1000 Encoder-Ticks schaltet der Code auf **25% PWM** (Slow-Down)
3. Der GA12-N20 bei 5V hat bei 25% PWM **nicht genug Drehmoment** um das Rollo
   (besonders wenn voll ausgerollt = maximales Gewicht) noch zu bewegen
4. Der Encoder dreht sich nicht weiter
5. Die Stall-Erkennung (250ms keine Encoder-Bewegung) schaltet den Motor ab
6. Das kurze Pipsen = die 250ms zwischen "25% PWM wird angelegt" und "Software
   erkennt dass der Encoder stillsteht und schaltet ab"

### Warum 25% und nicht mehr?

Der Slow-Down schont den Mechanismus — der Motor ruckt nicht mit voller Wucht ins Ziel
sondern bremst sanft ab. ned14 (Original-Autor) nutzt ebenfalls 25%, hat aber einen
kräftigeren 50-RPM Motor bei 6V. Unser 39-RPM bei 5V ist schwächer und reicht bei 25%
nicht mehr zum Drehen am Ende.

### DRV8833 Fast Decay (Datenblatt-Hintergrund)

Der DRV8833 kennt zwei PWM-Modi (TI Datenblatt SLVSAR1C, Table 3):
- **Fast Decay** (was wir nutzen): Ein Pin PWM, der andere 0% → Motorwicklung
  "floating" während PWM off → Strom klingt schnell ab → weniger effektiver Strom
- **Slow Decay**: Ein Pin HIGH, der andere PWM → Wicklung kurzgeschlossen → Strom
  zirkuliert weiter → mehr effektives Drehmoment bei gleicher PWM-Rate

Der Code nutzt Fast Decay (Pin A = PWM, Pin B = 0%). Bei 25% PWM ist der effektive
Motorstrom dadurch niedriger als bei Slow Decay — ein Grund warum der Motor am Ende
nicht mehr dreht. Das ist bewusst so gelassen: das Pipsen dient als akustische
Bestätigung dass das Rollo angekommen ist.

### DRV8833 interne Overcurrent Protection (OCP)

Das DRV8833 Datenblatt beschreibt eine interne Überstromschutz-Schaltung die bei
Stall-Strom automatisch ein- und ausschaltet (OCP retry period). Das HW-627
Breakout-Board hat Kondensatoren und Widerstände die den OCP-Schwellwert möglicherweise
niedrig setzen. Das kann ebenfalls zum Pipsen beitragen — der Chip schaltet den Strom
zyklisch ein/aus bis die Software den Motor endgültig abschaltet.

Quelle: TI DRV8833 Datenblatt SLVSAR1C, Section "Protection Circuits" + ned14
Build-Anleitung (Beobachtung der HW-627 OCP bei Stall).

**Fazit:** Das Pipsen ist kein Bug sondern ein Feature — es bestätigt dass das Rollo
korrekt am Ziel angekommen ist. Keine Code-Änderung nötig.

## Lizenz

Dieses Projekt verwendet eine doppelte Lizenz:

- **Software** (ESPHome YAML, Skripte, Dokumentation): **GPL-3.0** — siehe [LICENSE](LICENSE)
- **3D-Druck-Dateien** (STL im Ordner `/stl/`): **CC-BY-4.0** — Original-Designs von [AndBu/YAIFM](https://github.com/AndBu/YAIFM)

## 3D-Druck-Dateien (STL)

Die STL-Dateien liegen im Ordner [`stl/`](stl/) und können direkt herunterladen und gedruckt werden.

> ⚠️ **WICHTIG: Diese STL-Dateien sind NICHT von uns!** Sie stammen vom Original-Projekt [AndBu/YAIFM](https://github.com/AndBu/YAIFM) und wurden von uns unverändert übernommen. Wir hosten sie hier nur als Bequemlichkeit damit man alles an einem Ort hat. Die Designs stehen unter **CC-BY-4.0** (Creative Commons Attribution 4.0 International). Alle Urheberrechte liegen beim Original-Autor AndBu.

| Datei | Beschreibung |
|-------|--------------|
| `YAIFM_motor_holder.stl` | Halterung für den GA12-N20 Motor im Rollo-Rohr |
| `YAIFM_hub.stl` | Adapter-Hub für die Motorwelle auf das Rollo |
| `YAIFM_lid.stl` | Deckel für die Motor-Halterung |
| `YAIFM_clip.stl` | Befestigungs-Clip |
| `YAIFM_PCB_holder.stl` | Halterung für eine kleine Platine (optional) |
| `YAIFM_switch_holder.stl` | Halterung für den Endstop-Schalter (nur beim Original, wir verwenden keinen) |
| `motor_holder_for_switch_holder.stl` | Motor-Halterung mit Endstop-Integration |
| `YAIFM_tolerance_checker__hub.stl` | Toleranz-Tester für den Hub (vor dem Druck testen!) |

### Hub-Toleranzen (`stl/different_hub_hole_clearances/`)

Da jeder 3D-Drucker etwas unterschiedliche Maße produziert, gibt es den Hub in verschiedenen Toleranzen:

| Datei | Toleranz |
|-------|----------|
| `YAIFM_hub_minus_0.05mm_hole_clearance.stl` | 0.05mm enger |
| `YAIFM_hub_plus_0.05mm_hole_clearance.stl` | 0.05mm weiter |
| `YAIFM_hub_plus_0.1mm_hole_clearance.stl` | 0.1mm weiter |

> 💡 **Tipp:** Erst den `YAIFM_tolerance_checker__hub.stl` drucken und testen welche Toleranz am besten auf euren Drucker und euer Rollo passt. Dann den passenden Hub auswählen.

### Druck-Empfehlungen

- **Material:** PLA oder PETG
- **Schichtdicke:** 0.2mm
- **Füllung:** 30-40% (Motor-Halterung), 20% (Hub/Deckel)
- **Support:** Nur bei der Motor-Halterung falls nötig

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
| NOT-AUS | Keine | Motor + Driver sofort aus |
| Test-Buttons | Keine | Motor Hoch/Runter (2s Testlauf) |
| Motorgeschwindigkeit | Fest | Konfigurierbar 20-100% über HA |
| EEP Driver Enable | Nicht verwendet | DRV8833 Sleep/Enable (GPIO26) |
| Neustart-Button | Keine | ESP32 Remote-Reboot über HA |

### Unterschiede zur Vorlage (ned14)
| Feature | ned14 | Diese Version |
|---------|-------|---------------|
| NVS-Persistenz | Encoder-Wert | Encoder-Wert + Referenz + Kalibrierung |
| Boot-Verhalten | Immer Auto-Level | Auto-Level nur bei ungültiger Referenz |
| Close-Sperre | Keine | Blockiert wenn Referenz ungültig |
| Position-Sperre | Keine | Blockiert wenn Referenz ungültig |
| Bewegungs-Sperre | Keine | Blockiert wenn Kalibrierung 0 (NVS korrupt) |
| HA Toleranz | 0.1% (99.9% = CLOSED) | 2% (98% = CLOSED) — verhindert falsche "offen" Anzeige |
| EEP Driver | Nicht verwendet | DRV8833 Sleep/Enable Pin (GPIO26) |
| NOT-AUS | Keine | Motor + Driver sofort aus |
| Test-Buttons | Keine | Motor Hoch/Runter (2s Testlauf) |
| Neustart-Button | Keine | ESP32 Remote-Reboot über HA |
| Motorgeschwindigkeit | Fest | Konfigurierbar 20-100% über HA Number-Entity |
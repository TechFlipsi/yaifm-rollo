# Troubleshooting — Bekannte Probleme und Lösungen

Diese Dokumentation enthält alle Probleme die während der Entwicklung und im Praxisbetrieb aufgetreten sind, inklusive der gefundenen Lösungen. Hilft anderen beim Nachbau falls sie ähnliche Probleme haben.

---

## 1. Cover zeigt OFFEN obwohl Rollo physikalisch geschlossen ist

**Symptom:** Rollo fährt komplett runter, Motor stallt am Boden, aber HA zeigt weiterhin OPEN statt CLOSED.

**Ursache:** Beim Auto-Level (Öffnen bis Stall) wird die Referenz auf Encoder 0 gesetzt, BEVOR der Motor vollständig ausläuft. Der Stoff spannt sich und dreht den Motor nach dem PWM-Off noch ~400-500 Ticks weiter. Die Referenz ist bereits auf 0 gespeichert, aber der echte Encoder-Wert steht auf ~442. Beim Schließen fehlen diese 442 Ticks am Ziel — das Rollo stallt zu früh und die Positionsberechnung ergibt <98% → wird als OPEN angezeigt.

**Log-Erkennung:**
```
Stall bei Encoder -24 -> Motor aus
Neue offen-Referenz gesetzt. Encoder = 0.
Encoder Position >> 420.0    ← Motor nachgelaufen!
```

**Lösung:** Motor PWM sofort aus, 800ms warten bis der Stoff sich entspannt hat, DANN Encoder auf 0 resetten und Referenz setzen. Siehe `do_position_blind` Stall-Handler: `auto_level_active && old_current_blind_operation < 0`.

---

## 2. Referenz-Reset bei normalem Positionieren (nicht nur Auto-Level)

**Symptom:** Rollo auf 25% gefahren. Motor stalled beim Slow-Down (25% Speed reicht nicht). Plötzlich steht das Rollo auf 0% (OPEN) — Referenz wurde fälschlich auf 0 gesetzt.

**Ursache:** Die Bedingung für den Referenz-Reset war `old_current_blind_operation < 0` (Öffnen). Beim Positionieren auf 25% schaltet der Motor Richtung, `old_current_blind_operation` wurde beim Öffnen gesetzt (negativ). Als der Motor dann beim Schließen stalled, hat der Code gedacht "das war ein Öffnungs-Stall" → Referenz auf 0.

**Lösung:** Bedingung erweitert auf `auto_level_active && old_current_blind_operation < 0`. Referenz-Reset passiert NUR noch beim Auto-Level — nie bei normalem Positionieren.

---

## 3. OTA Rollback nach Flash (Brownout)

**Symptom:** Nach dem OTA-Flashen startet der ESP32, der Auto-Level beginnt sofort, der Motor zieht Strom → Spannung bricht ein → Brownout → ESP32 resettet → OTA Rollback zur alten Firmware. Der neue Code läuft nie.

**Log-Erkennung:**
```
[W][safe_mode:094]: OTA rollback detected! Rolled back from partition 'app1'
[W][safe_mode:099]: Last reset was due to brownout - check your power supply!
```

**Ursache:** ESPHome Safe-Mode markiert die Firmware erst nach 60 Sekunden als "successful". Wenn der Motor innerhalb dieser 60s Strom zieht und die Spannung einbricht, resettet der ESP32 → Rollback.

**Lösung:** Auto-Level startet erst 80 Sekunden nach Boot (`delay: 80s` im `on_boot`). Die Firmware ist da bereits als "successful" markiert. Ein Brownout führt nicht mehr zum Rollback.

**Log nach Fix:**
```
[I][safe_mode:142]: Boot seems successful; resetting boot loop counter
[D][main:056]: Boot: Safe-Mode vorbei -> starte Auto-Level...
```

---

## 4. ESP32 stürzt ab wenn Motor läuft (Brownout bei Volllast)

**Symptom:** Motor startet (Auto-Level oder Schließen), läuft 2-4 Sekunden, dann disconnectet die API. ESP32 ist gecrasht (Brownout). Nach Reboot stimmen die Encoder-Werte nicht mehr.

**Ursache:** Die Stromversorgung ist zu schwach für ESP32 + DRV8833 + Motor gleichzeitig. Beim Motorstart bricht die 5V-Schiene ein → Brownout-Detector löst aus → ESP32 resettet.

**Lösung (Hardware):**
- **Elko (ab 220µF, ideal 470µF)** direkt an DRV8833 VCC/GND löten — puffert Stromspitzen
- **Separates 5V-Netzteil** für den Motor (nicht vom ESP32 durchschleifen)
- **Mindestens 2A** USB-Netzteil (nicht 1A)

> ⚠️ Software kann Brownouts nicht fixen. Das ist ein reines Hardware-Problem.

---

## 5. Encoder zählt negativ (falsche Richtung)

**Symptom:** Beim Hochfahren (Öffnen) zählt der Encoder in den negativen Bereich statt nach oben. Cover-Position zeigt falsche Werte.

**Ursache:** Encoder-Kabel C1 und C2 sind vertauscht. Der Encoder zählt in die falsche Richtung.

**Lösung:** `pin_a` und `pin_b` im `rotary_encoder` Block tauschen:
```yaml
# Vorher (negativ):
pin_a:
  number: GPIO16
pin_b:
  number: GPIO4

# Nachher (positiv):
pin_a:
  number: GPIO4
pin_b:
  number: GPIO16
```

---

## 6. Motor dreht beim Booten unkontrolliert (Vollgas, zufällige Richtung)

**Symptom:** Beim Einschalten der Stromversorgung dreht der Motor kurz Volllast in eine zufällige Richtung, bevor ESPHome initialisiert ist.

**Ursache:** PWM-Pins (GPIO14, GPIO27) sind beim Boot undefined. Der DRV8833 interpretiert floating Pins als Ansteuerung → Motor dreht.

**Lösung:** DRV8833 EEP-Pin (Sleep/Enable) an GPIO26 mit `restore_mode: ALWAYS_OFF`. Der Treiber ist beim Boot komplett deaktiviert. Erst nach `on_boot` (2s warten) wird der Driver aktiviert. J1 Brücke auf dem DRV8833 Board muss dafür GEÖFFNET sein.

---

## 7. Motor dreht nicht (Driver Enable fehlt)

**Symptom:** Cover-Slider in HA bewegt sich, aber Motor dreht nicht. Encoder ändert sich nicht.

**Ursache:** Wenn EEP über GPIO26 gesteuert wird (J1 offen), MUSS `switch.turn_on: driver_enable` vor JEDEM `fan.turn_on` / Motorbefehl stehen. Ohne Driver Enable schläft der DRV8833.

**Lösung:** In ALLEN Code-Pfaden die den Motor ansteuern (position_action, close_action, open_action, auto_level, Test-Buttons) MUSS `switch.turn_on: driver_enable` + `delay: 100ms` VOR dem Motorbefehl stehen.

---

## 8. Cover zeigt CLOSED bei 98% statt 100%

**Symptom:** Rollo ist physikalisch zu, aber HA zeigt 98% oder 99% statt 100% CLOSED.

**Ursache:** HA-Toleranz war zu eng eingestellt (0.1% = 99.9% für CLOSED). Encoder-Toleranzen und Stall-Abweichungen führen dazu dass nie genau 100% erreicht werden.

**Lösung:** Toleranz auf 2% erweitert in `do_publish_blind`:
```yaml
if (current <= 0.02f) return COVER_OPEN;    # 0-2% = OPEN
if (current >= 0.98f) return COVER_CLOSED;   # 98-100% = CLOSED
```

---

## 9. "Wert muss kleiner als oder gleich 500000 sein" in HA

**Symptom:** HA Number-Entity "Schritte bis ganz geschlossen" zeigt rote Fehlermeldung.

**Ursache:** Nach Brownout-Crashes kann der NVS-Wert korrupt sein (z.B. 150932 oder höher). Das `max_value` der Number-Entity war auf 500000 begrenzt — Werte darüber lösen die HA-Validierung aus.

**Lösung:** `max_value` auf 1000000 erhöht. Der Standardwert wurde von 70000 auf 152000 geändert (gemessener Wert für IKEA Fridans Rollos).

---

## 10. ESP32-C3 SuperMini Boot-Loop (USB-C Defekt)

**Symptom:** ESP32-C3 SuperMini Boards haben ständigen USB-Sound-Loop am PC (Gerät verbunden/getrennt im Sekundentakt). Power-LED flackert.

**Ursache:** Bekannter Hardware-Defekt am USB-C Connector des SuperMini. Betrifft alle Exemplare dieser Baureihe.

**Lösung:** Auf ESP32-WROOM-32D (Micro-USB) umsteigen. Funktioniert einwandfrei, gleiche GPIO-Pin-Belegung möglich.

---

## 11. Gehäusehälften trennen sich (Motor verklemmt)

**Symptom:** Motor bekommt Strom (hört man "piepen"), aber Encoder bewegt sich nicht. Cover geht sofort auf IDLE.

**Ursache:** Die zwei 3D-gedruckten Gehäusehälften die den GA12-N20 Motor halten, springen auseinander. Der Motor hat Spiel und verklemmt beim Drehen.

**Lösung:**
- Temporär: Sekundenkleber an die Fugen
- Dauerhaft: M3-Gewinde durch beide Hälften bohren und mit kurzer M3-Schraube sichern
- PLA alleine hält nicht gegen die Torsion des Motors

> 💡 **Diagnose-Regel:** Wenn Motor Strom bekommt aber Encoder 0 bleibt — ERST Mechanik prüfen (Gehäuse, Verklemmen), DANN Encoder-Kabel, ERST DANN Software.

---

## 12. Positions-Drift bei häufigen Teilfahrten

**Symptom:** Nach vielen Fahrten auf Zwischenpositionen (25%, 50%) verschiebt sich die Endposition leicht.

**Ursachen:**
- Slow-Down bei 25% Speed: Motor hat nicht genug Kraft → fälschlicher Stall → stoppt wenige Ticks vor Ziel
- Mechanisches Spiel: Stoff dehnt/spannt sich, Encoder misst Motorwelle nicht Stoffposition
- Stall bei normalem Positionieren speichert aktuelle Position die einige Ticks vom Ziel abweicht

**Lösung:** Regelmäßiges Auto-Level (Button in HA) oder einmal komplett öffnen (0% startet automatisch Auto-Level). In der Praxis ist die Drift minimal (<1%).

---

## 13. Auto-Level bricht bei Encoder 0 sofort ab

**Symptom:** Auto-Level startet, aber der Motor dreht nicht. Log zeigt "Ziel erreicht" sofort.

**Ursache:** Die `while`-Bedingung `abs(current - pos) > 0.001f` bricht ab wenn Encoder bereits 0 ist und `pos: 0.0` (Öffnen) angefordert wird. Da `abs(0 - 0) = 0` ist die Bedingung false — der Loop wird nie betreten.

**Lösung:** `auto_level_active` Flag. Wenn true, überspringt die `while`-Bedingung die Positionsprüfung und fährt immer weiter bis Stall. Das Flag wird bei `pos <= 0.001f && speed > 0.0f` automatisch gesetzt.

---

## 14. Encoder springt nach Auto-Level von 0 auf -7

**Symptom:** Auto-Level setzt Encoder auf 0, aber der nächste Encoder-Tick überschreibt die 0 wieder mit dem alten Sensor-Wert.

**Ursache:** Separate `encoder_position` globale Variable wird im `on_value` Callback bei jedem Tick überschrieben. Nullung der Variable wird vom nächsten Tick sofort überschrieben.

**Lösung:** Direkt `id(blind_encoder).state` lesen statt separate Variable. `sensor.rotary_encoder.set_value: 0` zum Nullen des Sensors selbst verwenden. `isnan()` Fallback falls der Encoder nie gedreht hat.

---

## 15. Boot-Sequenz blockiert Test-Buttons

**Symptom:** Test-Buttons (Motor Hoch/Runter) reagieren nicht nach dem Boot.

**Ursache:** Boot-Sequenz mit `script.execute` + `script.wait` blockiert alle anderen Aktionen. Das Boot-Script überschreibt den Motor-Output alle 250ms.

**Lösung:** Boot-Sequenz vereinfacht — nur `button.press: auto_level_button` statt inline `script.execute` + `script.wait`. Test-Buttons rufen `script.stop: do_position_blind` auf bevor sie den Motor ansteuern.

---

## Hardware-Empfehlungen

| Problem | Lösung | Kosten |
|---------|--------|--------|
| Brownout bei Motorstart | 470µF Elko an DRV8833 VCC/GND | ~1€ |
| Brownout bei Motorstart | Separates 5V Netzteil (2A+) für Motor | ~5€ |
| Gehäuse trennt sich | M3 Schraube durch beide Hälften | ~0,50€ |
| Motor dreht beim Boot | J1 Brücke öffnen, EEP an GPIO26 | 0€ (Kabel) |
| PLA weich am Fenster | ABS oder ASA drucken | Materialkosten |

---

*Letzte Aktualisierung: 26.07.2026*
*Getestet mit: 3x IKEA Fridans Rollos, ESP32-WROOM-32D, DRV8833, GA12-N20 Motor, ESPHome 2026.7.2*
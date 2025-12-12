# EZHI Batterie-Regelung v1.0.0 für Home Assistant

Intelligente Nulleinspeisung mit Batterie-Strategie für **APsystems EZHI Hybrid-Wechselrichter**.

![Home Assistant](https://img.shields.io/badge/Home%20Assistant-2024.x-blue)
![HACS](https://img.shields.io/badge/HACS-Required-orange)
![License](https://img.shields.io/badge/License-MIT-green)

## 🌟 Features

### 5W-Netzbilanz-Regelung mit adaptiver Peak-Erkennung
- Hält die Netzbilanz nahe 0W (konfigurierbar)
- **2-stufige Peak-Erkennung**: Kleine Peaks (200W+) und große Peaks (600W+)
- Unterschiedliche Reaktionsstärken je nach Peak-Größe

### Asymmetrische Rampe und Anti-Oszillations-Logik
- **Das Kernfeature**: Sofort runter, langsam hoch
- Post-Peak-Dämpfung verhindert Oszillationen nach Lastspitzen
- Intelligente Puffer-Zone für stabile Regelung

### SOC-Schutz mit PV-Laden
- **Kein Netzladen!** - Batterie wird nur mit PV-Überschuss geladen
- Sicherheitspuffer von 40W verhindert ungewolltes Netzladen
- WR-Timeout-Erkennung bei Kommunikationsproblemen

### Solarprognose-basierte Entladetiefe
- Nutzt Solcast-Prognose für den nächsten Tag
- Bei guter Prognose: Tiefere Entladung erlaubt
- Batterie wird morgen wieder voll geladen

---

## 📋 Voraussetzungen

### Hardware
- APsystems EZHI Hybrid-Wechselrichter
- Smart Meter (z.B. Shelly Pro 3EM, SDM630)
- Home Assistant Installation

### Integrationen

| Integration | Beschreibung | Link |
|-------------|--------------|------|
| **APsystems EZHI** | Wechselrichter-Steuerung | [kamilkosek/EZHI](https://github.com/kamilkosek/EZHI) |
| **Smart Meter** | Netzleistungs-Messung | Je nach Gerät |
| **Solcast** (optional) | Solarprognose | [BJReplay/ha-solcast-solar](https://github.com/BJReplay/ha-solcast-solar) |

### HACS Frontend (für Dashboard)

| Karte | Link |
|-------|------|
| **Mushroom Cards** | [piitaya/lovelace-mushroom](https://github.com/piitaya/lovelace-mushroom) |
| **Power Flow Card Plus** | [flixlix/power-flow-card-plus](https://github.com/flixlix/power-flow-card-plus) |
| **Fold Entity Row** | [thomasloven/lovelace-fold-entity-row](https://github.com/thomasloven/lovelace-fold-entity-row) |

---

## ⚙️ Polling-Einstellungen

Für optimale Regelung sind schnelle Update-Raten wichtig:

| Gerät | Empfohlenes Intervall |
|-------|----------------------|
| **EZHI Integration** | 2 Sekunden |
| **Smart Meter** | 1 Sekunde |

### EZHI Integration konfigurieren
In der Integration unter Einstellungen → Geräte & Dienste → APsystems EZHI:
- `scan_interval: 2` setzen

### Smart Meter
Je nach Gerät unterschiedlich. Bei Shelly: Native Integration mit 1s Update.

---

## 📥 Installation

### Option 1: Automation direkt importieren

1. **Helfer erstellen** (siehe [helpers.yaml](helpers.yaml))
   - Manuell über UI: Einstellungen → Geräte & Dienste → Helfer
   - Oder in `configuration.yaml` einbinden

2. **Automation importieren**
   - Einstellungen → Automatisierungen → Neue Automatisierung
   - YAML-Modus aktivieren
   - Inhalt von `automation_commented.yaml` oder `automation_minimal.yaml` einfügen

3. **Entity-IDs anpassen** (falls nötig)
   - `sensor.sm63_net_power` → Dein Netzleistungs-Sensor
   - `sensor.battery_soc` → Dein Batterie-SOC-Sensor
   - `sensor.pv_produktion_gesamt` → Dein PV-Sensor

### Option 2: Blueprint verwenden

1. **Blueprint kopieren**
   ```bash
   # In deinem Home Assistant config-Ordner:
   mkdir -p blueprints/automation/ezhi
   # Kopiere blueprints/ezhi_battery_control.yaml dorthin
   ```

2. **Automation aus Blueprint erstellen**
   - Einstellungen → Automatisierungen → Neue Automatisierung
   - "Aus Blueprint erstellen"
   - "EZHI Batterie-Regelung v1.0.0" auswählen
   - Deine Entities zuordnen

### Option 3: LIGHT Blueprint (nur 1 Helfer!)

Für alle die **möglichst wenig Helfer** anlegen möchten:

1. **Einen Helfer erstellen**: `input_datetime.last_peak` (mit Datum + Zeit)
2. **Blueprint kopieren**: `blueprints/ezhi_battery_control_light.yaml`
3. **4 Sensoren + 1 Helfer auswählen** - fertig!

Alle anderen Parameter werden direkt im Blueprint konfiguriert. Ideal wenn man Werte nur 2-3x pro Jahr ändert.

**Nicht enthalten:** Solarprognose, Notifications, Dashboard-steuerbare Parameter

---

## 🔧 Benötigte Helfer

### Pflicht-Helfer

```yaml
input_number:
  lade_limit:
    name: Lade-Limit Batterie
    min: 50
    max: 100
    step: 1
    unit_of_measurement: "%"

  entlade_limit:
    name: Entlade-Limit Batterie
    min: 5
    max: 50
    step: 1
    unit_of_measurement: "%"

  puffer_leistung:
    name: Netzbilanz-Puffer
    min: 0
    max: 500
    step: 10
    unit_of_measurement: "W"

  peak_schwelle_niedrig:
    name: Peak-Schwelle Niedrig
    min: 50
    max: 500
    step: 25
    unit_of_measurement: "W"

  peak_schwelle_hoch:
    name: Peak-Schwelle Hoch
    min: 200
    max: 1500
    step: 50
    unit_of_measurement: "W"

input_datetime:
  last_peak:
    name: Letzter Peak Zeitstempel
    has_date: true
    has_time: true
```

### Optional (für Vollversion mit Notifications)

```yaml
input_boolean:
  entlade_limit_notification_sent:
    name: Entlade-Limit Notification gesendet

  pv_laden_notification_sent:
    name: PV-Laden Notification gesendet

  lade_limit_notification_sent:
    name: Lade-Limit Notification gesendet
```

### Optional (für Solarprognose)

```yaml
input_number:
  min_solarprognose_schwelle:
    name: Min. Solarprognose für tiefere Entladung
    min: 0
    max: 20
    step: 0.5
    unit_of_measurement: "kWh"

  zusaetzliche_entladung_prozent:
    name: Zusätzliche Entladung bei guter Prognose
    min: 0
    max: 20
    step: 1
    unit_of_measurement: "%"
```

Die vollständige Helfer-Konfiguration findest du in [helpers.yaml](helpers.yaml).

---

## 🚀 Erste Schritte nach der Installation

### ⚠️ Wichtig: Manuelle Erstausführung

Nach der Installation muss die Automation **einmal manuell gestartet** werden, um den Peak-Zeitstempel zu initialisieren:

1. Einstellungen → Automatisierungen
2. "EZHI Batterie-Regelung" finden
3. Auf die drei Punkte klicken → "Ausführen"

### Empfohlene Anfangswerte

### Empfohlene Anfangswerte

| Parameter | Empfohlener Wert | Scharfe Einstellung |
|-----------|------------------|---------------------|
| Lade-Limit | 95% | 100% |
| Entlade-Limit | 20% | 12% |
| Puffer-Leistung | 50W | 10W |
| Peak-Schwelle Niedrig | 200W | 250W |
| Peak-Schwelle Hoch | 600W | 600W |
| Min. Solarprognose | 3 kWh | 0kWh |
| Zusätzliche Entladung | 5% | 0% |

---

## 📊 Dashboard einrichten

1. **HACS-Karten installieren** (siehe oben)
2. **Dashboard erstellen**
   - Einstellungen → Dashboards → Dashboard hinzufügen
3. **YAML-Modus**
   - Dashboard öffnen → Drei Punkte → Dashboard bearbeiten → Drei Punkte → Raw-Konfigurationseditor
   - Inhalt von [dashboard.yaml](dashboard.yaml) einfügen
4. **Entity-IDs anpassen** (falls nötig)

---

## 📁 Dateiübersicht

| Datei | Beschreibung |
|-------|--------------|
| `ezhi_automation_commented.yaml` | Vollständige Automation v1.0.0 mit Kommentaren |
| `ezhi_automation_minimal.yaml` | v1.0.0 ohne Logging und Notifications |
| `dashboard.yaml` | Lovelace Dashboard |
| `helpers.yaml` | Alle benötigten Helfer |
| `blueprints/ezhi_battery_control.yaml` | Blueprint v1.0.0 mit Logging/Notifications |
| `blueprints/ezhi_battery_control_minimal.yaml` | Blueprint v1.0.0 minimal |
| `blueprints/ezhi_battery_control_light.yaml` | **LIGHT: Nur 1 Helfer nötig!** |

---

## 🔍 Funktionsweise im Detail

### Regelungslogik

```
┌─────────────────────────────────────────────────────────────┐
│                    TRIGGER: Netzleistung ändert sich        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│     MINDESTABSTAND: Min. 0.3s seit letzter Ausführung       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     SOC ≤ Entlade-Limit?      │
              └───────────────────────────────┘
                     │               │
                    JA              NEIN
                     │               │
                     ▼               ▼
         ┌───────────────┐   ┌───────────────────────┐
         │  SOC-SCHUTZ   │   │   NORMALE REGELUNG    │
         │               │   │                       │
         │ PV > 10W und  │   │ 1. Peak-Level prüfen  │
         │ Einspeisung?  │   │ 2. Faktor berechnen   │
         │      │        │   │ 3. Delta berechnen    │
         │   JA │ NEIN   │   │ 4. Neuen Wert setzen  │
         │      ▼        │   │                       │
         │ PV-Laden mit  │   │                       │
         │ Sicherheits-  │   │                       │
         │ puffer (40W)  │   │                       │
         └───────────────┘   └───────────────────────┘
```

### Peak-Reaktion

| Peak-Level | Schwelle | Faktor | Beschreibung |
|------------|----------|--------|--------------|
| 0 | < 200W | 0.25 | Normal, langsame Anpassung |
| 1 | 200-600W | 0.6 | Kleiner Peak, moderate Reaktion |
| 2 | > 600W | 1.0 | Großer Peak, sofortige Reaktion |

### Post-Peak-Dämpfung

Nach einem Peak wird die Reaktion gedämpft:

| Zeit nach Peak | Dämpfung |
|----------------|----------|
| 0-15s | 40% |
| 15-45s | 70% |
| 45-90s | 90% |
| > 90s | 100% |

---

## 🐛 Troubleshooting

### Automation startet nicht
- Prüfe ob alle Entity-IDs korrekt sind
- Prüfe ob alle Helfer erstellt wurden
- Führe die Automation einmal manuell aus

### Oszillationen / Pendeln
- Erhöhe `puffer_leistung` (z.B. auf 150W)
- Reduziere Polling-Intervall des Smart Meters
- Prüfe ob WR-Timeout vorliegt

### WR reagiert nicht
- Prüfe EZHI Integration
- Prüfe Netzwerkverbindung zum Wechselrichter
- Logs prüfen: `[SOC-SCHUTZ] ... [TIMEOUT]`

### Netzladen trotz SOC-Schutz
- Sollte nicht passieren durch Sicherheitspuffer
- Prüfe ob `soc_schutz_puffer` (40W) ausreicht
- Bei Bedarf in Automation erhöhen

---

## 📄 Lizenz

MIT License - siehe [LICENSE](LICENSE)

---

## 🤝 Beitragen

Pull Requests sind willkommen! Bitte erstelle zuerst ein Issue für größere Änderungen.

---

## 🙏 Credits

- [kamilkosek](https://github.com/kamilkosek) - APsystems EZHI Integration
- [piitaya](https://github.com/piitaya) - Mushroom Cards
- [flixlix](https://github.com/flixlix) - Power Flow Card Plus

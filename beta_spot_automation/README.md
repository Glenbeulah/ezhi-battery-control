# EZHI Batterie-Regelung v1.1.0-beta - Spot-Preis-Steuerung

## ⚠️ BETA VERSION - Zum Testen!

Diese Version erweitert die Basisautomation um:
- **Dynamische Stromtarife** (EPEX Spot)
- **PV-Prognose** (Solcast - nächste 4 Stunden)
- **Verbrauchsprognose** (letzte 7 Tage, stündlich)

## Betriebsmodi

Der Helfer `input_select.ezhi_modus` steuert das Verhalten der Automation:

### 🔵 Normal
Die Standard-Regelung aus v1.0.0:
- Nulleinspeisung mit Peak-Detection
- SOC-Schutz bei niedrigem Batteriestand
- **Ignoriert Spot-Preise komplett**
- Ideal für: Feste Stromtarife, Einspeisung unerwünscht

### 🟢 Spot-Optimiert
Volle Spot-Preis-Steuerung:
- **Billige Stunden (Rang 1-4):** Batterie aus Netz laden
- **Teure Stunden (Rang 21-24):** Batterie entladen, tieferes Limit
- **Mittlere Preise:** Normal regeln oder halten (je nach PV-Prognose)
- Berücksichtigt PV-Forecast und Verbrauchsprofil
- Ideal für: Dynamische Tarife (Tibber, aWATTar, EPEX)

### 🔴 Manuell
Automation deaktiviert:
- Keine automatischen Änderungen am Inverter
- Volle manuelle Kontrolle
- Ideal für: Wartung, Tests, Urlaub

## Dateien

| Datei | Typ | Einbinden unter |
|-------|-----|-----------------|
| `ezhi_hausverbrauch_template.yaml` | Template Sensor | `template:` (optional) |
| `ezhi_spot_sql.yaml` | SQL Sensor | `sql:` |
| `ezhi_spot_templates.yaml` | Template Sensoren | `template:` |
| `ezhi_spot_helpers.yaml` | Helfer (input_select) | `input_select:` |
| `ezhi_spot_automation.yaml` | Automation | Import via UI |
| `ezhi_spot_dashboard.yaml` | Dashboard | Raw-Editor |

## Architektur

```
┌─────────────────────────────────────────────────────────────────┐
│               HAUSVERBRAUCH TOTAL (optional)                    │
│  (ezhi_hausverbrauch_template.yaml)                            │
├─────────────────────────────────────────────────────────────────┤
│  Berechnet echten Verbrauch aus kumulativen Wh-Zählern:        │
│  Netzbezug + PV + Batterie_Out - Einspeisung - Batterie_In     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SQL SENSOR                                   │
│  (ezhi_spot_sql.yaml)                                          │
├─────────────────────────────────────────────────────────────────┤
│  Verbrauchsprofil 7 Tage → JSON {"00": 63, "01": 58, ...}      │
│  (Wh pro Stunde, Durchschnitt der letzten 7 Tage)              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    TEMPLATE SENSOREN                            │
│  (ezhi_spot_templates.yaml)                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Verbrauch 4h ────────────────────────────────┐                │
│  (aus SQL-Profil)                              │                │
│                                                │                │
│  EPEX Spot ──┬── Rang (1-24)                  │                │
│              └── Quantil (0-1)    ────────────┼──► EZHI        │
│                                                │    Spot        │
│  Solcast ────── PV nächste 4h ────────────────┤    Action      │
│                 (aus detailedHourly)           │                │
│                                                │                │
│                    ↓                           │                │
│              Energie-Bilanz 4h ────────────────┘                │
│              (PV 4h - Verbrauch 4h)                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AUTOMATION                                   │
│  (ezhi_spot_automation.yaml)                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Prüft input_select.ezhi_modus:                                │
│                                                                 │
│  - "Normal"         → Standard-Regelung (v1.0.0)               │
│  - "Spot-Optimiert" → Spot-Preis-Steuerung                     │
│  - "Manuell"        → Keine Aktion                             │
│                                                                 │
│  Bei Spot-Optimiert liest sensor.ezhi_spot_action:             │
│  - "laden"    → Netzladen mit max. Leistung                    │
│  - "entladen" → Tieferes Entlade-Limit                         │
│  - "halten"   → Batterie schonen                               │
│  - "normal"   → Standard-Regelung                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Installation

### 0. Hausverbrauch-Sensor (empfohlen)

Für ein genaues Verbrauchsprofil solltest du den **echten Hausverbrauch** messen, nicht nur den Netzbezug. Der Netzbezug zeigt tagsüber zu niedrige Werte, weil PV-Eigenverbrauch fehlt.

**Option A: Eigenen Sensor erstellen (empfohlen)**

Kopiere `ezhi_hausverbrauch_template.yaml` nach `/config/` und passe die Sensor-Namen an:

```yaml
# configuration.yaml
template: !include ezhi_hausverbrauch_template.yaml
```

Die Datei enthält Beispiele für verschiedene Wechselrichter (Fronius, SMA, Shelly, APsystems).

**Option B: Nur Netzbezug (einfacher, aber ungenauer)**

Verwende direkt deinen Smart Meter Import-Sensor in `ezhi_spot_sql.yaml`.

### 1. SQL Sensor

```yaml
# configuration.yaml
sql: !include ezhi_spot_sql.yaml
```

**Wichtig:** Passe die `statistic_id` an:
- Mit Hausverbrauch-Sensor: `sensor.hausverbrauch_total`
- Ohne: `sensor.DEIN_SMARTMETER_IMPORT_WH`

### 2. Template Sensoren

```yaml
# configuration.yaml
template: !include ezhi_spot_templates.yaml
```

### 3. Helfer

```yaml
# configuration.yaml
input_select: !include ezhi_spot_helpers.yaml
```

**Oder über UI:** Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Dropdown

### 4. Automation importieren

Einstellungen → Automatisierungen → ⋮ → Aus YAML importieren → `ezhi_spot_automation.yaml`

### 5. Dashboard (optional)

1. Einstellungen → Dashboards → Dashboard hinzufügen → "EZHI Spot"
2. Dashboard öffnen → Bearbeiten → Raw-Konfigurationseditor
3. Inhalt von `ezhi_spot_dashboard.yaml` einfügen

**Empfohlene HACS-Cards:**
- `apexcharts-card` - Für Verbrauchsprofil-Grafik
- `mushroom-cards` - Für schönere Status-Karten

### 6. Home Assistant neustarten

Nach dem Neustart sollten die Sensoren erscheinen.

## Verbrauchsprofil erklärt

### Was wird gemessen?

Das SQL-Query analysiert die letzten 7 Tage und berechnet den **durchschnittlichen Verbrauch pro Stunde**.

### Warum Hausverbrauch statt Netzbezug?

```
Beispiel um 12:00 im Sommer:
┌─────────────────────────────────────────┐
│ Echter Verbrauch:     400 Wh           │
│ PV-Eigenverbrauch:    350 Wh           │
│ Netzbezug (gemessen):  50 Wh  ← FALSCH │
└─────────────────────────────────────────┘
```

Der Netzbezug zeigt nur 50 Wh, obwohl 400 Wh verbraucht werden. Das führt zu falschen Prognosen!

### Formel für Hausverbrauch

```
Hausverbrauch = Netzbezug + PV + Batterie_Entladung - Einspeisung - Batterie_Ladung
```

Diese Formel nutzt **nur kumulative Wh-Zähler** - keine Riemann-Summe, robust gegen Sensorausfälle.

## Entscheidungslogik (Spot-Optimiert)

### Inputs

| Quelle | Sensor | Beschreibung |
|--------|--------|--------------|
| EPEX Spot | `sensor.epex_spot_data_rank` | Rang 1-24 (1=billigste Stunde) |
| EPEX Spot | `sensor.epex_spot_data_quantile` | 0-1 (0=billigste) |
| Solcast | `sensor.solcast_pv_forecast_prognose_heute` | detailedHourly Attribut |
| SQL | `sensor.ezhi_verbrauchsprofil_7_tage` | Durchschnitt pro Stunde |

### Berechnete Sensoren

| Sensor | Beschreibung |
|--------|--------------|
| `sensor.ezhi_erwarteter_verbrauch_stunde` | Verbrauch jetzt (Wh) |
| `sensor.ezhi_pv_prognose_4h` | PV nächste 4 Stunden (Wh) |
| `sensor.ezhi_energie_bilanz_4h` | PV 4h minus Verbrauch 4h (Wh) |
| `sensor.ezhi_spot_action` | Empfehlung: laden/entladen/halten/normal |
| `sensor.ezhi_spot_grund` | Begründung als Text |
| `sensor.ezhi_spot_score` | -100 bis +100 |
| `sensor.ezhi_dynamisches_entlade_limit` | Angepasstes Limit (%) |
| `sensor.ezhi_spot_netzladen_max_soc` | Max SOC fürs Netzladen (%) |

### Entscheidungsbaum

```
PREIS BILLIG (Rang ≤4 oder Quantil <0.2)?
├── JA: Bilanz 4h > +2000 Wh?
│       ├── JA: → NORMAL (genug PV in nächsten 4h)
│       └── NEIN: → LADEN (aus Netz!)
│
PREIS TEUER (Rang ≥21 oder Quantil >0.8)?
├── JA: → ENTLADEN (Batterie nutzen)
│
SONST (mittlerer Preis):
├── Bilanz 4h > +1000 Wh? → HALTEN (PV kommt bald)
├── Nach 18 Uhr UND morgen >5kWh PV? → HALTEN (morgen laden)
├── Bilanz 4h < -500 Wh? → NORMAL (Defizit decken)
└── Sonst → NORMAL
```

## Hinweise zum SQL-Sensor

### Warum ist der State leer?

Das JSON ist zu lang für den State (max 255 Zeichen). **Das ist korrekt!** 

Die Daten liegen im **Attribut** `hourly_json`. Die Template-Sensoren lesen von dort.

### Debug: Welche statistic_id hast du?

Falls `unknown`, prüfe verfügbare IDs:

```yaml
sql:
  - name: "Debug Statistics IDs"
    db_url: sqlite:////config/home-assistant_v2.db
    query: "SELECT statistic_id FROM statistics_meta WHERE statistic_id LIKE '%meter%' OR statistic_id LIKE '%energy%' OR statistic_id LIKE '%verbrauch%' LIMIT 20"
    column: "statistic_id"
```

## Voraussetzungen

| Integration | Link | Hinweis |
|-------------|------|---------|
| EPEX Spot | [GitHub](https://github.com/mampfes/ha_epex_spot) | HACS, Marktgebiet DE-LU |
| Solcast | [GitHub](https://github.com/oziee/ha-solcast-solar) | HACS, kostenloser Account |

## Beispiel-Szenarien

### Winter-Nacht (02:00)
```
Modus: Spot-Optimiert
Rang: 2 (billig), PV 4h: 0 Wh, Verbrauch 4h: 250 Wh
Bilanz: -250 Wh

→ Action: LADEN
→ Max SOC: 80% (wenig PV morgen)
→ Grund: "Billigste Stunden (Rang 2, 2.3ct) - Laden!"
```

### Sommer-Vormittag (08:00)
```
Modus: Spot-Optimiert
Rang: 8 (mittel), PV 4h: 3800 Wh, Verbrauch 4h: 350 Wh
Bilanz: +3450 Wh

→ Action: HALTEN
→ Entlade-Limit: 35%
→ Grund: "PV-Überschuss in 4h (+3.5kWh) - Batterie schonen"
```

### Abend-Peak (18:30)
```
Modus: Spot-Optimiert
Rang: 23 (teuer!), PV 4h: 0 Wh, Verbrauch 4h: 300 Wh
Bilanz: -300 Wh

→ Action: ENTLADEN
→ Entlade-Limit: 10%
→ Grund: "Teuerste Stunden (Rang 23, 38ct) - Entladen!"
```

## Troubleshooting

| Problem | Lösung |
|---------|--------|
| SQL Sensor `unknown` | statistic_id prüfen, min. 1 Tag Historie nötig |
| State leer, Attribut hat Daten | **Korrekt!** Template liest aus Attribut |
| Template zeigt 0 | SQL Sensor prüfen |
| EPEX Sensoren fehlen | HACS Integration installieren |
| ApexCharts zeigt "Loading" | Browser-Cache leeren, HACS-Card neu installieren |
| Verbrauch tagsüber zu niedrig | Hausverbrauch-Sensor verwenden (PV-Eigenverbrauch fehlt) |

## Nächste Schritte / TODO

- [ ] Wochenende vs. Wochentag unterscheiden
- [ ] Tibber/aWATTar als Alternative
- [ ] Minimale Lade-/Entlade-Dauer (Hysterese)
- [ ] Notification bei Spot-Aktionen

## Feedback

Issues und Feedback gerne auf GitHub!

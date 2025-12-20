# EZHI Batterie-Regelung v1.1.0-beta - Spot-Preis-Steuerung

## ⚠️ BETA VERSION - Zum Testen!

Diese Version erweitert die Basisautomation um:
- **Dynamische Stromtarife** (EPEX Spot)
- **PV-Prognose** (Solcast)
- **Verbrauchsprognose** (letzte 7 Tage, stündlich)

## Architektur

```
┌─────────────────────────────────────────────────────────────────┐
│                    SQL SENSOR                                   │
│  (sql_sensors.yaml)                                             │
├─────────────────────────────────────────────────────────────────┤
│  Verbrauchsprofil 7 Tage → JSON {"00": 180, "01": 150, ...}    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    TEMPLATE SENSOREN                            │
│  (templates_spot.yaml)                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Verbrauchsprofil ─── Erwarteter Verbrauch (Wh) ───┐           │
│                                                     │           │
│  EPEX Spot ──┬── Rang (1-24)                       │           │
│              └── Quantil (0-1)     ────────────────┼──► EZHI   │
│                                                     │    Spot   │
│  Solcast ───┬── PV Rest heute                      │    Action │
│             └── PV morgen          ────────────────┘           │
│                                                                 │
│                    ↓                                            │
│              Energie-Bilanz 4h                                  │
│              (PV - Verbrauch)                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AUTOMATION                                   │
│  (ezhi_automation_spot_v1.1.0-beta.yaml)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Liest sensor.ezhi_spot_action und entscheidet:                │
│                                                                 │
│  - "laden"    → Netzladen mit max. Leistung                    │
│  - "entladen" → Tieferes Entlade-Limit                         │
│  - "halten"   → Batterie schonen                               │
│  - "normal"   → Standard-Regelung                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Entscheidungslogik

### Inputs

| Quelle | Sensor | Beschreibung |
|--------|--------|--------------|
| EPEX Spot | `sensor.epex_spot_data_rank` | Rang 1-24 (1=billigste Stunde) |
| EPEX Spot | `sensor.epex_spot_data_quantile` | 0-1 (0=billigste) |
| Solcast | `sensor.solcast_pv_forecast_prognose_verbleibende_leistung_heute` | Erwartete PV in kWh |
| Solcast | `sensor.solcast_pv_forecast_prognose_morgen` | PV morgen in kWh |
| SQL | `sensor.ezhi_verbrauchsprofil_7_tage` | Durchschnitt pro Stunde |

### Berechnete Werte

| Sensor | Beschreibung |
|--------|--------------|
| `sensor.ezhi_erwarteter_verbrauch_stunde` | Verbrauch jetzt (Wh) |
| `sensor.ezhi_energie_bilanz_4h` | PV minus Verbrauch nächste 4h (Wh) |
| `sensor.ezhi_spot_action` | Empfehlung: laden/entladen/halten/normal |
| `sensor.ezhi_dynamisches_entlade_limit` | Angepasstes Limit (%) |
| `sensor.ezhi_spot_netzladen_max_soc` | Max SOC fürs Netzladen (%) |

### Entscheidungsbaum

```
PREIS BILLIG (Rang ≤4)?
├── JA: Energie-Bilanz > +2kWh UND vor 14 Uhr?
│       ├── JA: → NORMAL (genug PV kommt)
│       └── NEIN: → LADEN (aus Netz!)
│
PREIS TEUER (Rang ≥21)?
├── JA: → ENTLADEN (Batterie nutzen)
│
SONST (mittlerer Preis):
├── Bilanz > +1kWh UND vor 12 Uhr? → HALTEN (PV kommt)
├── Nach 18 Uhr UND morgen >5kWh PV? → HALTEN (morgen laden)
├── Bilanz < -500Wh? → NORMAL (Defizit decken)
└── Sonst → NORMAL
```

## Installation

### 1. SQL Sensor einbinden

```yaml
# configuration.yaml
sensor: !include sensors/ezhi_sql_sensors.yaml
```

Kopiere `sql_sensors.yaml` nach `/config/sensors/ezhi_sql_sensors.yaml`

**WICHTIG:** Passe die `statistic_id` an deinen Sensor an:
```yaml
WHERE statistic_id = 'sensor.smart_meter_ac_meter_total_watt_hours_imported'
```

### 2. Template Sensoren einbinden

```yaml
# configuration.yaml
template: !include templates/ezhi_spot_sensors.yaml
```

Kopiere `templates_spot.yaml` nach `/config/templates/ezhi_spot_sensors.yaml`

### 3. Helfer erstellen

```yaml
input_select:
  ezhi_modus:
    name: EZHI Betriebsmodus
    options:
      - Normal
      - Spot-Optimiert
      - Manuell
    icon: mdi:auto-fix
```

### 4. Automation importieren

Die Datei `ezhi_automation_spot_v1.1.0-beta.yaml` in Home Assistant importieren.

### 5. Home Assistant neustarten

Nach dem Neustart sollten die neuen Sensoren erscheinen.

## Dateien

| Datei | Typ | Beschreibung |
|-------|-----|--------------|
| `sql_sensors.yaml` | SQL Sensor | Verbrauchsprofil aus Statistics DB |
| `templates_spot.yaml` | Template | Alle berechneten Sensoren |
| `ezhi_automation_spot_v1.1.0-beta.yaml` | Automation | Hauptlogik |
| `README.md` | Doku | Diese Datei |

## Neue Sensoren

### Verbrauchsprognose

| Sensor | Beschreibung |
|--------|--------------|
| `sensor.ezhi_verbrauchsprofil_7_tage` | JSON mit allen 24 Stunden |
| `sensor.ezhi_erwarteter_verbrauch_stunde` | Aktuell erwarteter Verbrauch |
| → Attribut `naechste_stunde` | Verbrauch nächste Stunde |
| → Attribut `naechste_4_stunden` | Summe nächste 4 Stunden |

### Energie-Bilanz

| Sensor | Beschreibung |
|--------|--------------|
| `sensor.ezhi_energie_bilanz_4h` | PV - Verbrauch (Wh) |
| → Attribut `bewertung` | "Überschuss" / "Defizit" |

### Spot-Steuerung

| Sensor | Beschreibung |
|--------|--------------|
| `sensor.ezhi_spot_action` | laden / entladen / halten / normal |
| `sensor.ezhi_spot_grund` | Begründung als Text |
| `sensor.ezhi_spot_score` | -100 bis +100 |
| `sensor.ezhi_dynamisches_entlade_limit` | Angepasstes Limit |
| `sensor.ezhi_spot_netzladen_max_soc` | Max SOC fürs Netzladen |

## Dashboard-Beispiel

```yaml
type: entities
title: EZHI Spot-Steuerung
entities:
  - entity: input_select.ezhi_modus
  - type: divider
  - entity: sensor.ezhi_spot_action
    name: Empfehlung
  - entity: sensor.ezhi_spot_grund
    name: Grund
  - type: divider
  - entity: sensor.epex_spot_data_market_price
    name: Aktueller Preis
    suffix: " ct/kWh"
  - entity: sensor.epex_spot_data_rank
    name: Rang (1=billigste)
  - type: divider
  - entity: sensor.ezhi_erwarteter_verbrauch_stunde
    name: Erwarteter Verbrauch
  - entity: sensor.ezhi_energie_bilanz_4h
    name: Energie-Bilanz 4h
  - type: divider
  - entity: sensor.ezhi_dynamisches_entlade_limit
    name: Entlade-Limit (dynamisch)
  - entity: sensor.ezhi_spot_netzladen_max_soc
    name: Max SOC Netzladen
```

## Beispiel-Szenarien

### Winter-Nacht (02:00)
```
Rang: 2 (billig)
PV morgen: 1.5 kWh
Verbrauch 4h: 600 Wh
Bilanz: -600 Wh (Defizit)

→ Action: LADEN
→ Max SOC: 80% (wenig PV morgen)
→ Grund: "Billigste Stunden (Rang 2, 2.3ct) - Laden!"
```

### Sommer-Vormittag (10:00)
```
Rang: 8 (mittel)
PV Rest: 12 kWh
Verbrauch 4h: 2000 Wh
Bilanz: +10000 Wh (Überschuss!)

→ Action: HALTEN
→ Entlade-Limit: 35% (statt 20%)
→ Grund: "Überschuss erwartet (+10kWh) - Batterie schonen"
```

### Abend-Peak (18:30)
```
Rang: 23 (teuer!)
PV Rest: 0.2 kWh
PV morgen: 8 kWh
Verbrauch 4h: 3500 Wh
Bilanz: -3300 Wh (Defizit)

→ Action: ENTLADEN
→ Entlade-Limit: 10% (statt 20%)
→ Grund: "Teuerste Stunden (Rang 23, 38ct) - Entladen!"
```

## Troubleshooting

### SQL Sensor zeigt "unknown"

1. Prüfe ob `statistic_id` korrekt ist:
   ```sql
   SELECT * FROM statistics_meta WHERE statistic_id LIKE '%import%';
   ```

2. Prüfe ob Daten vorhanden sind (mind. 1 Tag Historie nötig)

### Template Sensoren zeigen 0

1. Warte bis SQL Sensor gültige Daten hat
2. Prüfe Developer Tools → States → `sensor.ezhi_verbrauchsprofil_7_tage`

### EPEX Spot Sensoren fehlen

1. HACS Integration installieren: https://github.com/mampfes/ha_epex_spot
2. Integration einrichten (DE-LU Marktgebiet)

## Nächste Schritte / TODO

- [ ] Wochenende vs. Wochentag unterscheiden
- [ ] Tibber/aWATTar als Alternative
- [ ] Minimale Lade-/Entlade-Dauer
- [ ] ApexCharts Karte für Tagesverlauf

## Feedback

Diese Beta ist für Tests und Diskussion. Was funktioniert? Was fehlt?

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
│  Verbrauchsprofil 7 Tage → JSON {"00": 63, "01": 58, ...}      │
│  (Wh pro Stunde, Durchschnitt der letzten 7 Tage)              │
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
| EPEX Spot | `sensor.epex_spot_data_market_price` | Aktueller Preis in ct/kWh |
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

### 1. SQL Sensor einrichten

⚠️ **Wichtig:** Die SQL-Integration verwendet seit HA 2023.x eine neue Syntax!

Füge in `configuration.yaml` einen **eigenen Block** `sql:` hinzu (nicht unter `sensor:`):

```yaml
sql:
  - name: "EZHI Verbrauchsprofil 7 Tage"
    db_url: sqlite:////config/home-assistant_v2.db
    query: >
      SELECT json_group_object(hour, ROUND(avg_diff, 0)) as hourly_json
      FROM (
        SELECT 
          strftime('%H', start_ts, 'unixepoch', 'localtime') as hour,
          AVG(state - prev_state) as avg_diff
        FROM (
          SELECT 
            start_ts,
            state,
            LAG(state) OVER (ORDER BY start_ts) as prev_state
          FROM statistics
          WHERE metadata_id = (
            SELECT id FROM statistics_meta 
            WHERE statistic_id = 'sensor.smart_meter_ac_meter_total_watt_hours_imported'
          )
          AND start_ts > strftime('%s', 'now', '-7 days')
        )
        WHERE prev_state IS NOT NULL 
          AND (state - prev_state) >= 0 
          AND (state - prev_state) < 5000
        GROUP BY hour
      )
    column: "hourly_json"
```

**Anpassen:** Ersetze `sensor.smart_meter_ac_meter_total_watt_hours_imported` mit deinem Energie-Sensor!

#### Hinweise zum SQL-Sensor

- **Ergebnis:** JSON im Attribut `hourly_json`: `{"00":63,"01":58,"07":179,...}`
- **State ist leer:** Das ist normal! JSON ist zu lang für State (max 255 Zeichen)
- **LAG() Funktion:** Berechnet Differenz für kumulative Zähler automatisch
- **Filter:** Ignoriert negative Werte und Ausreißer >5000 Wh/h

#### Debug: Welche statistic_id hast du?

Falls der Sensor `unknown` zeigt, prüfe verfügbare IDs mit diesem temporären Sensor:

```yaml
sql:
  - name: "Debug Statistics IDs"
    db_url: sqlite:////config/home-assistant_v2.db
    query: "SELECT statistic_id FROM statistics_meta WHERE statistic_id LIKE '%meter%' OR statistic_id LIKE '%energy%' LIMIT 20"
    column: "statistic_id"
```

### 2. Template Sensoren einbinden

```yaml
# configuration.yaml
template: !include templates/ezhi_spot_sensors.yaml
```

Kopiere `templates_spot.yaml` nach `/config/templates/ezhi_spot_sensors.yaml`

Oder füge den Inhalt direkt unter `template:` in deiner configuration.yaml ein.

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

Nach dem Neustart sollten die neuen Sensoren erscheinen:
- `sensor.ezhi_verbrauchsprofil_7_tage` (Attribut: hourly_json)
- `sensor.ezhi_erwarteter_verbrauch_stunde`
- `sensor.ezhi_spot_action` (benötigt EPEX + Solcast)

## Dateien

| Datei | Typ | Beschreibung |
|-------|-----|--------------|
| `sql_sensors.yaml` | SQL Config | Verbrauchsprofil aus Statistics DB |
| `templates_spot.yaml` | Template | Alle berechneten Sensoren |
| `ezhi_automation_spot_v1.1.0-beta.yaml` | Automation | Hauptlogik |
| `dashboard_spot.yaml` | Dashboard | Komplette Dashboard-Seite |
| `README.md` | Doku | Diese Datei |

## Sensoren im Detail

### Verbrauchsprognose

| Sensor | Beschreibung |
|--------|--------------|
| `sensor.ezhi_verbrauchsprofil_7_tage` | JSON mit allen 24 Stunden (im Attribut!) |
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

## Dashboard

Siehe `dashboard_spot.yaml` für ein komplettes Dashboard.

**Installation:**
1. Einstellungen → Dashboards → Dashboard hinzufügen → "EZHI Spot"
2. Dashboard öffnen → Bearbeiten → Raw-Konfigurationseditor
3. Inhalt von `dashboard_spot.yaml` einfügen

**Optionale HACS-Cards für bessere Darstellung:**
- `card-mod` - Für Farben
- `mushroom-cards` - Für schöne Status-Karten  
- `apexcharts-card` - Für Verbrauchsprofil-Grafik

Das Dashboard funktioniert auch ohne diese Cards (Fallback auf Standard-HA-Cards).

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

## Voraussetzungen

### EPEX Spot Integration
- HACS Integration: https://github.com/mampfes/ha_epex_spot
- Marktgebiet: DE-LU (Deutschland/Luxemburg)

### Solcast Integration
- HACS Integration: https://github.com/oziee/ha-solcast-solar
- Kostenloser Account: https://solcast.com/

### Energie-Sensor
- Ein Sensor der den Stromverbrauch in Wh misst
- Muss in der HA Statistics-Datenbank sein (Langzeit-Statistik aktiviert)

## Troubleshooting

### SQL Sensor zeigt "unknown"

1. Prüfe ob `statistic_id` korrekt ist (siehe Debug-Query oben)
2. Mindestens 1 Tag Historie nötig (besser 7 Tage)
3. Neustart nach Änderung erforderlich

### State ist leer, aber Attribut hat Daten

Das ist **korrekt**! JSON ist zu lang für den State. Die Template-Sensoren lesen aus dem Attribut.

### Template Sensoren zeigen 0

1. Warte bis SQL Sensor gültige Daten hat
2. Prüfe: Entwicklerwerkzeuge → Zustände → `sensor.ezhi_verbrauchsprofil_7_tage`
3. Attribut `hourly_json` muss JSON enthalten

### EPEX Spot Sensoren fehlen

1. HACS Integration installieren
2. Integration einrichten mit DE-LU Marktgebiet
3. Sensoren haben Prefix `sensor.epex_spot_data_*`

## Nächste Schritte / TODO

- [ ] Wochenende vs. Wochentag unterscheiden
- [ ] Tibber/aWATTar als Alternative zu EPEX
- [ ] Minimale Lade-/Entlade-Dauer (Hysterese)
- [ ] ApexCharts Karte für Tagesverlauf
- [ ] Notification bei Spot-Aktionen

## Feedback

Diese Beta ist zum Testen gedacht. Was funktioniert? Was fehlt?

Issues und Feedback gerne auf GitHub!

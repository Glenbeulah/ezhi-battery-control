# EZHI Batterie-Regelung v1.1.0-beta - Spot-Preis-Steuerung

## ⚠️ BETA VERSION - Zum Testen!

Diese Version erweitert die Basisautomation um:
- **Dynamische Stromtarife** (EPEX Spot)
- **PV-Prognose** (Solcast)
- **Verbrauchsprognose** (letzte 7 Tage, stündlich)

## Dateien

| Datei | Typ | Einbinden unter |
|-------|-----|-----------------|
| `ezhi_spot_sql.yaml` | SQL Sensor | `sql:` |
| `ezhi_spot_templates.yaml` | Template Sensoren | `template:` |
| `ezhi_spot_helpers.yaml` | Helfer (input_select) | `input_select:` |
| `ezhi_spot_automation.yaml` | Automation | Import via UI |
| `ezhi_spot_dashboard.yaml` | Dashboard | Raw-Editor |

## Architektur

```
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
│  (ezhi_spot_automation.yaml)                                   │
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

## Installation

### 1. SQL Sensor

⚠️ **Wichtig:** Die SQL-Integration verwendet seit HA 2023.x eine neue Syntax!

**Option A: Include (empfohlen)**
```yaml
# configuration.yaml
sql: !include ezhi_spot_sql.yaml
```
Kopiere `ezhi_spot_sql.yaml` nach `/config/`

**Option B: Direkt einfügen**

Füge den Inhalt von `ezhi_spot_sql.yaml` direkt in `configuration.yaml` ein.

**Anpassen:** Ersetze die `statistic_id` mit deinem Energie-Sensor!

### 2. Template Sensoren

```yaml
# configuration.yaml
template: !include ezhi_spot_templates.yaml
```
Kopiere `ezhi_spot_templates.yaml` nach `/config/`

### 3. Helfer

```yaml
# configuration.yaml
input_select: !include ezhi_spot_helpers.yaml
```
Kopiere `ezhi_spot_helpers.yaml` nach `/config/`

**Oder über UI:** Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Dropdown

### 4. Automation importieren

Die Datei `ezhi_spot_automation.yaml` in Home Assistant importieren:
Einstellungen → Automatisierungen → ⋮ → Aus YAML importieren

### 5. Dashboard (optional)

1. Einstellungen → Dashboards → Dashboard hinzufügen → "EZHI Spot"
2. Dashboard öffnen → Bearbeiten → Raw-Konfigurationseditor
3. Inhalt von `ezhi_spot_dashboard.yaml` einfügen

### 6. Home Assistant neustarten

Nach dem Neustart sollten die Sensoren erscheinen.

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

### Berechnete Sensoren

| Sensor | Beschreibung |
|--------|--------------|
| `sensor.ezhi_erwarteter_verbrauch_stunde` | Verbrauch jetzt (Wh) |
| `sensor.ezhi_energie_bilanz_4h` | PV minus Verbrauch nächste 4h (Wh) |
| `sensor.ezhi_spot_action` | Empfehlung: laden/entladen/halten/normal |
| `sensor.ezhi_spot_grund` | Begründung als Text |
| `sensor.ezhi_spot_score` | -100 bis +100 |
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

## Hinweise zum SQL-Sensor

### Warum ist der State leer?

Das JSON ist zu lang für den State (max 255 Zeichen). **Das ist korrekt!** 

Die Daten liegen im **Attribut** `hourly_json`. Die Template-Sensoren lesen von dort:
```jinja2
{% set data = state_attr('sensor.ezhi_verbrauchsprofil_7_tage', 'hourly_json') %}
```

### Debug: Welche statistic_id hast du?

Falls `unknown`, prüfe verfügbare IDs:

```yaml
sql:
  - name: "Debug Statistics IDs"
    db_url: sqlite:////config/home-assistant_v2.db
    query: "SELECT statistic_id FROM statistics_meta WHERE statistic_id LIKE '%meter%' OR statistic_id LIKE '%energy%' LIMIT 20"
    column: "statistic_id"
```

### LAG() für kumulative Zähler

Die Query berechnet automatisch die Differenz pro Stunde. Filter gegen Ausreißer:
- `(state - prev_state) >= 0` → Keine negativen Werte
- `(state - prev_state) < 5000` → Max 5 kWh/Stunde

## Voraussetzungen

| Integration | Link | Hinweis |
|-------------|------|---------|
| EPEX Spot | [GitHub](https://github.com/mampfes/ha_epex_spot) | HACS, Marktgebiet DE-LU |
| Solcast | [GitHub](https://github.com/oziee/ha-solcast-solar) | HACS, kostenloser Account |

## Beispiel-Szenarien

### Winter-Nacht (02:00)
```
Rang: 2 (billig), PV morgen: 1.5 kWh, Bilanz: -600 Wh
→ LADEN, Max SOC: 80%
```

### Sommer-Vormittag (10:00)
```
Rang: 8 (mittel), PV Rest: 12 kWh, Bilanz: +10000 Wh
→ HALTEN, Entlade-Limit: 35%
```

### Abend-Peak (18:30)
```
Rang: 23 (teuer!), PV morgen: 8 kWh, Bilanz: -3300 Wh
→ ENTLADEN, Entlade-Limit: 10%
```

## Troubleshooting

| Problem | Lösung |
|---------|--------|
| SQL Sensor `unknown` | statistic_id prüfen, min. 1 Tag Historie nötig |
| State leer, Attribut hat Daten | **Korrekt!** Template liest aus Attribut |
| Template zeigt 0 | SQL Sensor prüfen |
| EPEX Sensoren fehlen | HACS Integration installieren |

## Nächste Schritte / TODO

- [ ] Wochenende vs. Wochentag unterscheiden
- [ ] Tibber/aWATTar als Alternative
- [ ] Minimale Lade-/Entlade-Dauer (Hysterese)
- [ ] ApexCharts für Tagesverlauf

## Feedback

Issues und Feedback gerne auf GitHub!

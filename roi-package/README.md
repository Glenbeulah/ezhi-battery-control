# 📊 EZHI Batterie ROI-Analyse

Berechnet die Wirtschaftlichkeit und Amortisation deiner APsystems EZHI Batterie in Home Assistant.

## Was wird berechnet?

- **Netto-Ersparnis** = Vermiedener Netzbezug − Entgangene Einspeisung − Standby-Kosten
- **Amortisationsfortschritt** in Prozent und Break-even-Datum
- **Jährliche Rendite** und 10-Jahres-Gewinn-Prognose
- **Wirkungsgrad-Tracking** (DC-zu-DC und leistungsabhängig)
- **Verlust-Analyse** (AC-Verluste beim Laden/Entladen)
- **Strompreis-Snapshot** — bei Preisänderungen bleiben historische Erträge korrekt

---

## Voraussetzungen

| Komponente | Link |
|------------|------|
| **EZHI Integration** | [kamilkosek/EZHI](https://github.com/kamilkosek/EZHI) |
| **Mushroom Cards** (HACS) | [piitaya/lovelace-mushroom](https://github.com/piitaya/lovelace-mushroom) |
| card-mod (optional) | [thomasloven/lovelace-card-mod](https://github.com/thomasloven/lovelace-card-mod) |

### Benötigte EZHI-Sensoren

Diese Sensoren werden von der [EZHI Integration](https://github.com/kamilkosek/EZHI) bereitgestellt:

| Sensor | Beschreibung |
|--------|-------------|
| `sensor.ezhi_battery_total_charge_energy` | Gesamte Ladeenergie (kWh) |
| `sensor.ezhi_battery_total_discharge_energy` | Gesamte Entladeenergie (kWh) |
| `sensor.ezhi_battery_power` | Aktuelle Batterieleistung (W, positiv=Laden, negativ=Entladen) |

---

## Installation (3 Schritte)

### Schritt 1: Package-Datei auf Home Assistant kopieren

Kopiere `ezhi_roi_package.yaml` in den `/config/packages/` Ordner deiner HA-Installation.

**Option A: Über File Editor / Studio Code Server Add-on (empfohlen)**
1. Erstelle den Ordner `packages` unter `/config/` (falls noch nicht vorhanden)
2. Erstelle dort eine neue Datei `ezhi_roi_package.yaml`
3. Kopiere den Inhalt der Datei von GitHub hinein und speichere

**Option B: Per SCP/SSH (wenn SSH-Zugang vorhanden)**
```bash
scp ezhi_roi_package.yaml user@homeassistant:/config/packages/
```

**Option C: Per Samba-Share**
Navigiere zum HA-Share → `config` → `packages` und kopiere die Datei hinein



Falls du noch keinen `packages`-Ordner nutzt, erstelle ihn und aktiviere ihn in der `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

### Schritt 2: Werte anpassen

Öffne `/config/packages/ezhi_roi_package.yaml` und passe diese Werte an:

| Was | Wo ändern | Default |
|-----|-----------|---------|
| **Kaufpreis** | Dashboard → ⚙️ Konfiguration oder `initial:` im Package | 1000 € |
| **Strompreis** | Dashboard → ⚙️ Konfiguration oder `initial:` im Package | 30 ct/kWh |
| **Einspeisevergütung** | Dashboard → ⚙️ Konfiguration oder `initial:` im Package | 8,0 ct/kWh |
| **Inbetriebnahme-Datum** | Dashboard → ⚙️ Konfiguration (input_datetime) | – (muss gesetzt werden) |
| **Standby-Verbrauch** | Im Package: `standby_w = 19` | 19 W |

> **Hinweis:** Die `initial:`-Werte werden **nur beim ersten Start** gesetzt. Danach bleiben deine Änderungen über Neustarts hinweg erhalten. Du kannst alle Werte jederzeit über das Dashboard anpassen.

### Schritt 3: Dashboard einrichten

1. **Home Assistant → Einstellungen → Dashboards → "+ Dashboard hinzufügen"**
2. Wähle "Manuelles Dashboard erstellen"
3. Dashboard öffnen → **Stift-Icon (oben rechts) → ⋮ → Raw-Konfigurationseditor**
4. Gesamten Inhalt von `dashboard_roi.yaml` einfügen
5. Speichern

### Neustart

```
Home Assistant → Einstellungen → System → Neustart
```

Nach dem Neustart:
1. **Inbetriebnahme-Datum setzen**: Entwicklerwerkzeuge → Dienste → `input_datetime.set_datetime` → Entity: `input_datetime.ezhi_roi_start_datum` → Datum eingeben (oder über Dashboard/UI)
2. Initiale Werte im Dashboard unter "⚙️ Konfiguration" prüfen

---

## Strompreis-Snapshot (automatisch)

Wenn sich dein Strompreis ändert, soll die bisherige Ersparnis nicht rückwirkend neu berechnet werden. Das Package enthält eine **Automation**, die bei Änderung von `input_number.ezhi_roi_strompreis` automatisch einen Snapshot erstellt:

```
Neuer Snapshot = bisheriger Snapshot + (aktuelle kWh − Snapshot kWh) × alter Preis
```

**So funktioniert es:**
1. Du änderst den Strompreis im Dashboard (z.B. von 32,33 auf 27,76 ct/kWh)
2. Die Automation sichert automatisch die bisherigen kWh und € als Snapshot
3. Ab sofort werden nur neue kWh mit dem neuen Preis berechnet
4. Du bekommst eine Benachrichtigung mit den Snapshot-Details

**Beispiel:** Batterie hat 500 kWh bei 30 ct entladen (= 150€). Preis wird auf 28 ct geändert.
→ Snapshot: 150€ bei 500 kWh. Ab jetzt: 150€ + (neue kWh − 500) × 0,28 €/kWh.

> **Wichtig:** Die Snapshot-Helfer (`ezhi_roi_brutto_snapshot_*`, `ezhi_roi_standby_snapshot_*`) werden automatisch verwaltet. Nicht manuell ändern!

---

## Anpassung an andere Systeme

Falls du keinen EZHI nutzt, sondern ein anderes Batteriesystem, musst du die **Sensor-Namen** anpassen.

Suche & Ersetze in `ezhi_roi_package.yaml`:

| Original (EZHI) | Ersetze durch deinen Sensor |
|-----------------|---------------------------|
| `sensor.ezhi_battery_total_charge_energy` | Dein Lade-Zähler (kWh, steigend) |
| `sensor.ezhi_battery_total_discharge_energy` | Dein Entlade-Zähler (kWh, steigend) |
| `sensor.ezhi_battery_power` | Deine Batterie-Leistung (W, +Laden/-Entladen) |

Zusätzlich anpassen:
- **Standby-Verbrauch**: Suche `standby_w = 19` und ersetze `19` durch deinen Wert
- **Wirkungsgrad-Kurven**: Die Sensoren `EZHI ROI Wirkungsgrad Laden/Entladen Aktuell` haben EZHI-spezifische Werte. Passe die Leistungs-/Wirkungsgrad-Tabelle an dein System an.
- **Durchschnittliche Wirkungsgrade**: Suche `0.85` (Laden) und `0.95` (Entladen) und ersetze durch deine gemessenen Werte.

---

## Berechnungslogik

```
Netto-Ersparnis = Vermiedener Netzbezug − Entgangene Einspeisung − Standby

Vermiedener Netzbezug = Entladung (kWh) × Strompreis (€/kWh)
Entgangene Einspeisung = Ladung (kWh) × Einspeisevergütung (€/kWh)
Standby-Kosten         = Betriebstage × 24h × Standby (W) / 1000 × Strompreis

Amortisation (%)  = Netto-Ersparnis / Anschaffungskosten × 100
Break-even (Tage) = Verbleibend (€) / Ersparnis pro Tag (€)
Rendite (%/Jahr)  = Ersparnis pro Jahr / Anschaffungskosten × 100
```

### Snapshot-basierte Berechnung

Bei Strompreisänderungen wird die Formel zu:
```
Brutto = Snapshot€ + (aktuelle_kWh − Snapshot_kWh) × neuer_Preis
```
Dadurch werden historische kWh korrekt mit dem damaligen Preis bewertet.

### Korrigierte Berechnung (mit Live-Verlusten)

Der Sensor `EZHI ROI Ersparnis Netto Korrigiert` nutzt eine **Riemann-Summen-Integration** (`sensor.ezhi_roi_verluste_integriert`) über die leistungsabhängige Verlustleistung. Dadurch wird der Wirkungsgrad bei jeder Leistungsstufe korrekt erfasst, statt feste Durchschnittswerte zu verwenden.

**Wichtig:** Die Live-Integration startet erst ab der Installation. Für historische Daten davor wird automatisch auf geschätzte Durchschnittswerte (Laden: 85%, Entladen: 95%) zurückgegriffen. Im Sensor-Attribut `verluste_quelle` ist immer sichtbar, welche Methode gerade aktiv ist.

### Wirkungsgrad-Kurven (EZHI gemessen aus www.photovoltaikforum.com)

| Leistung | Laden | Entladen |
|----------|-------|----------|
| 1200W | 88% | 98% |
| 400W | 86% | 97% |
| 200W | 81% | 94% |
| 100W | 70% | 86% |
| 50W | 50% | 75% |

---

## Enthaltene Sensoren

| Sensor | Beschreibung |
|--------|-------------|
| `sensor.ezhi_roi_betriebstage` | Tage seit Inbetriebnahme |
| `sensor.ezhi_roi_durchsatz_gesamt` | Gesamte Entladeenergie |
| `sensor.ezhi_roi_wirkungsgrad_dc` | Realer DC-zu-DC Wirkungsgrad |
| `sensor.ezhi_roi_standby_verbrauch` | Kumulierter Standby-Verbrauch |
| `sensor.ezhi_roi_standby_kosten` | Standby-Kosten in € (Snapshot-basiert) |
| `sensor.ezhi_roi_ersparnis_brutto` | Vermiedener Netzbezug in € (Snapshot-basiert) |
| `sensor.ezhi_roi_entgangene_einspeisung` | Entgangene Einspeisevergütung in € |
| `sensor.ezhi_roi_ersparnis_netto` | Netto-Ersparnis nach Abzügen |
| `sensor.ezhi_roi_amortisation_prozent` | Amortisationsfortschritt in % |
| `sensor.ezhi_roi_verbleibend` | Noch offener Betrag bis Break-even |
| `sensor.ezhi_roi_ersparnis_pro_tag` | Durchschnittliche Ersparnis/Tag |
| `sensor.ezhi_roi_ersparnis_pro_jahr` | Hochrechnung Ersparnis/Jahr |
| `sensor.ezhi_roi_tage_bis_breakeven` | Prognostizierte Tage bis Break-even |
| `sensor.ezhi_roi_breakeven_datum` | Prognostiziertes Break-even Datum |
| `sensor.ezhi_roi_amortisationszeit` | Geschätzte Amortisationszeit in Jahren |
| `sensor.ezhi_roi_rendite_jaehrlich` | Jährliche Rendite in % |
| `sensor.ezhi_roi_gewinn_10_jahre` | Prognostizierter 10-Jahres-Gewinn |
| `sensor.ezhi_roi_status` | Textuelle Zusammenfassung |
| `sensor.ezhi_roi_wirkungsgrad_laden_aktuell` | Live Lade-Wirkungsgrad |
| `sensor.ezhi_roi_wirkungsgrad_entladen_aktuell` | Live Entlade-Wirkungsgrad |
| `sensor.ezhi_roi_verluste_geschaetzt` | Geschätzte historische Verluste |
| `sensor.ezhi_roi_verlustkosten` | Verlustkosten in € |
| `sensor.ezhi_roi_ersparnis_netto_korrigiert` | Netto-Ersparnis mit AC-Verlusten |

---

## Lizenz

MIT – Nutzung und Anpassung frei erlaubt.

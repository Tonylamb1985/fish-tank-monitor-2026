# 🐟 Freshwater Fish Tank Monitor — Home Assistant Package
## Modular Install Guide

---

## 📁 File Structure

```
config/
├── configuration.yaml          ← add packages + themes lines here
├── packages/
│   ├── fish_tank_helpers.yaml      📦 All input helpers (thresholds, logs, flags)
│   ├── fish_tank_sensors.yaml      📦 Template sensors (status, health, countdowns)
│   ├── fish_tank_automations.yaml  📦 All automations (alerts, schedules, reminders)
│   ├── fish_tank_scripts.yaml      📦 All scripts (feed, log, emergency actions)
│   └── fish_tank_dashboard.yaml    📊 Lovelace dashboard (paste into UI separately)
└── themes/
    └── aquarium.yaml               🎨 Aquarium theme (optional - applies globally)
```

---

## ⚡ Step 1 — Enable Packages and Themes in configuration.yaml

Add this to your `configuration.yaml` (skip if already present):

```yaml
homeassistant:
  packages: !include_dir_named packages

frontend:
  themes: !include_dir_merge_named themes
```

---

## ⚡ Step 2 — Drop the Package Files

Copy these 4 files into `config/packages/`:
- `fish_tank_helpers.yaml`
- `fish_tank_sensors.yaml`
- `fish_tank_automations.yaml`
- `fish_tank_scripts.yaml`

**Install order doesn't matter** — HA loads them all at startup.

---

## ⚡ Step 3 — Restart Home Assistant

Settings → System → Restart

After restart, all helpers, sensors, automations and scripts will be live.

---

## ⚡ Step 4 — Add the Dashboard

1. Settings → Dashboards → **Add Dashboard**
2. Name it `Freshwater Tank`, icon `mdi:fishbowl-outline`
3. Open the dashboard → **⋮ menu → Edit Dashboard**
4. **⋮ → Raw Configuration Editor**
5. Paste the contents of `fish_tank_dashboard.yaml` → **Save**

---

## ⚡ Step 5 — Install HACS Frontend Cards

Go to HACS → Frontend → search and install:

| Card | Purpose |
|------|---------|
| `mini-graph-card` | History sparkline graphs |
| `button-card` | Equipment toggle buttons |
| `apexcharts-card` | Radial gauge charts |
| `mushroom-template-card` | Header status banner |
| `bar-card` | Nitrogen cycle bars |

---

## ⚡ Step 6 — (Optional) Apply the Aquarium Theme Globally

To give **all your dashboards** the underwater aquarium aesthetic:

1. **Copy** `themes/aquarium.yaml` into `config/themes/`
2. **Restart** Home Assistant
3. **Click your profile icon** (bottom left)
4. **Theme** → Select **Aquarium** from dropdown
5. **Refresh** the page

✨ The theme applies globally:
- Deep ocean gradient backgrounds on all views
- Frosted glass panels on all cards
- Cyan/aqua accents everywhere
- Soft aquarium glow effects

📖 See `themes/THEME_INSTALL.md` for customization options.

**Note:** The fish tank dashboard already has aquarium styling built-in, so it looks great with or without the global theme. The theme just extends that look to all your other dashboards too.

---

## ⚡ Step 7 — Connect Your Hardware Sensors

Replace entity IDs in the dashboard with your actual hardware sensors.
Expected entity names (rename yours to match, or find/replace in the YAML):

| Entity | Source |
|--------|--------|
| `sensor.fish_tank_temperature` | DS18B20 / Atlas EZO-RTD via ESPHome |
| `sensor.fish_tank_ph` | Atlas Scientific EZO-pH |
| `sensor.fish_tank_ammonia` | Atlas Scientific EZO-NH3 |
| `sensor.fish_tank_nitrite` | Manual `input_number` or Hanna meter |
| `sensor.fish_tank_nitrate` | Manual `input_number` or Hanna meter |
| `sensor.fish_tank_dissolved_o2` | Atlas Scientific EZO-DO |
| `sensor.fish_tank_gh` | Manual `input_number` (drop test) |
| `sensor.fish_tank_kh` | Manual `input_number` (drop test) |
| `sensor.fish_tank_tds` | TDS meter via ESPHome |
| `sensor.fish_tank_co2` | CO₂ sensor (optional, planted tanks) |
| `sensor.fish_tank_turbidity` | DF Robot Turbidity Sensor |
| `sensor.fish_tank_water_level` | HC-SR04 ultrasonic via ESPHome |
| `switch.fish_tank_filter` | Smart plug or ESPHome relay |
| `switch.fish_tank_heater` | Smart plug or ESPHome relay |
| `switch.fish_tank_air_pump` | ESPHome relay |
| `switch.fish_tank_co2_regulator` | ESPHome solenoid relay |
| `switch.fish_tank_uv_sterilizer` | Smart plug or ESPHome relay |
| `switch.fish_tank_led_lights` | Smart plug / WLED |
| `switch.fish_tank_auto_feeder` | ESPHome relay |

---

## ⚡ Step 8 — Set Your Notify Target

In `fish_tank_automations.yaml` and `fish_tank_scripts.yaml`, replace:
```yaml
service: notify.mobile_app_your_phone
```
With your actual notifier. Find yours at:
**Developer Tools → Services → search `notify`**

---

## 🔧 Customisation

### Change Tank Thresholds
Go to **Settings view** on the dashboard and adjust sliders/boxes for:
- Temperature range, pH range, ammonia/nitrite/nitrate limits
- GH/KH hardness range, CO₂ range, O₂ minimum

### Enable Planted Tank Mode
Toggle **Planted Tank Mode** in Settings → Tank Mode.
This activates CO₂ schedule automations and fertiliser reminders.

### Vacation Mode
Toggle **Vacation Mode** — reduces feeding to once daily at 12:00 PM.

### Cycling Mode
Toggle **Cycling Mode** while establishing a new tank — suppresses ammonia
and nitrite alerts that are expected during the nitrogen cycle.

---

## 📦 What Each Package Does

### `fish_tank_helpers.yaml`
All the input helpers HA needs:
- `input_datetime` — maintenance log timestamps
- `input_number` — alert thresholds and maintenance intervals
- `input_boolean` — feature flags (alerts, planted mode, vacation, cycling)
- `input_select` — tank type, water source, filter type dropdowns

### `fish_tank_sensors.yaml`
Template sensors derived from raw hardware readings:
- Per-parameter status (`optimal` / `warning` / `critical`)
- Overall tank health score (0–100%)
- Overall status + message (used by header banner)
- Days-since and due-in countdown sensors for maintenance tasks
- Tank age in days

### `fish_tank_automations.yaml`
All automations, each independently togglable from Settings view:
- 9x water quality alerts (NH₃, NO₂, NO₃, temp ×2, pH ×2, O₂, CO₂)
- 2x lighting schedule (on/off)
- 2x CO₂ schedule (planted only)
- 3x feeding (morning, evening, vacation)
- 3x maintenance reminders (water change, filter clean, fertiliser)
- 2x vacation mode notification

### `fish_tank_scripts.yaml`
Callable scripts for dashboard buttons and voice assistants:
- Feed now + log
- Log water change / filter clean / glass clean / fert dose / water test / substrate vac
- Emergency aeration (cuts CO₂, maxes air pump)
- All equipment on / all equipment off

---

## 🐟 Freshwater Parameter Reference

| Parameter | Safe Range | Critical if |
|-----------|-----------|-------------|
| Temperature | 72–80°F (tropical) | < 68°F or > 83°F |
| pH | 6.5–7.5 | < 6.0 or > 8.0 |
| Ammonia | 0 ppm | > 0.25 ppm |
| Nitrite | 0 ppm | > 0.25 ppm |
| Nitrate | < 20 ppm | > 40 ppm |
| Dissolved O₂ | 6–9 mg/L | < 5 mg/L |
| GH | 4–12 dGH | < 2 or > 20 dGH |
| KH | 4–8 dKH | < 2 dKH (pH crash risk) |
| TDS | 100–300 ppm | > 500 ppm |
| CO₂ (planted) | 15–30 ppm | > 40 ppm |

# 🌡️ Multi-Point Temperature Monitoring — Session Summary

**Topic:** Raspberry Pi 3B+ multi-zone temperature monitoring with MQTT, calibration dashboard, and Celsius conversion  
**Date:** February 2026  

---

## 📋 What Was Built

A 3-zone waterproof temperature monitoring system using a Raspberry Pi 3B+ running ESPHome, publishing to Home Assistant via MQTT with per-sensor calibration controls built into the Settings dashboard view.

### Zones Monitored
- **Main Tank** — Primary fish environment
- **Sump** — Filtration system
- **ATO Reservoir** — Auto top-off water supply

---

## 📦 Files Created

| File | Location | Purpose |
|------|----------|---------|
| `fish_tank_temps.yaml` | `~/esphome/` (on Pi) | ESPHome sensor config |
| `fish_tank_temperature_multipoint.yaml` | `config/packages/` | HA package (sensors, automations, scripts) |
| `TEMPERATURE_DASHBOARD_CARDS.yaml` | Reference file | Dashboard cards for Overview + Settings views |
| `TEMPERATURE_MULTIPOINT_GUIDE.md` | Reference file | Full hardware + software setup guide |

---

## 🔌 Hardware

### Components
- **Raspberry Pi 3B+** — Runs ESPHome, already owned
- **3× DS18B20 waterproof probes** — $6–8 each, stainless steel, 3m cable
- **1× 4.7kΩ resistor** — Pull-up for 1-wire bus
- **Jumper wires** — Female-female connectors
- **Total cost:** ~$25

### Wiring (all 3 sensors share same 3 wires)
```
DS18B20 RED    → Pi Pin 1  (3.3V)
DS18B20 BLACK  → Pi Pin 6  (GND)
DS18B20 YELLOW → Pi Pin 7  (GPIO4)
4.7kΩ resistor → Between Pin 1 (3.3V) and Pin 7 (GPIO4)
```

### Sensor Labelling
- Red ribbon = Main Tank
- Blue ribbon = Sump
- Green ribbon = ATO Reservoir

---

## ⚙️ ESPHome Configuration (`fish_tank_temps.yaml`)

### Key Settings
```yaml
platform: rpi
board: rpi3
dallas:
  pin: GPIO4
  update_interval: 10s
mqtt:
  broker: 192.168.1.50    # ← HA IP address
  port: 1883
  discovery: true
```

### Sensors Defined
- `fish_tank_temperature` — Main tank (calibrated)
- `sump_temperature` — Sump (calibrated)
- `ato_reservoir_temperature` — ATO reservoir (calibrated)
- `tank_sump_temperature_delta` — Gradient between tank and sump
- `pi_cpu_temperature` — Pi health monitoring
- WiFi signal strength

### Calibration API Services
ESPHome exposes 3 callable services from HA:
```yaml
esphome.fish_tank_temps_calibrate_tank_temp  (offset: float)
esphome.fish_tank_temps_calibrate_sump_temp  (offset: float)
esphome.fish_tank_temps_calibrate_ato_temp   (offset: float)
```
Offsets are stored in flash memory and survive reboots.

### Raw Sensor Handling
- Raw sensors marked `internal: true` (hidden from HA)
- Calibrated template sensors exposed publicly
- 5-sample sliding window average to reduce noise

---

## 📦 HA Package (`fish_tank_temperature_multipoint.yaml`)

### Calibration Helpers (input_number)
```yaml
input_number.fish_tank_temp_calibration       # ±3°C, step 0.1
input_number.fish_tank_sump_temp_calibration  # ±3°C, step 0.1
input_number.fish_tank_ato_temp_calibration   # ±3°C, step 0.1
input_number.fish_tank_temp_gradient_threshold  # 1–10°C, default 1.5°C
```

### Template Sensors Created
| Sensor | Description |
|--------|-------------|
| `fish_tank_temperature` | Calibrated main tank |
| `fish_tank_sump_temperature` | Calibrated sump |
| `fish_tank_ato_temperature` | Calibrated ATO |
| `fish_tank_temperature_average` | Average across all 3 zones |
| `fish_tank_temperature_max` | Highest of the 3 zones |
| `fish_tank_temperature_min` | Lowest of the 3 zones |
| `fish_tank_temperature_range` | Max minus min |
| `fish_tank_tank_sump_gradient` | Tank temp minus sump temp |
| `fish_tank_ato_tank_gradient` | ATO temp minus tank temp |

### Binary Sensor Alerts
| Sensor | Triggers When |
|--------|--------------|
| `fish_tank_temperature_stratification` | Tank-sump gradient > threshold (default 1.5°C) |
| `fish_tank_ato_water_cold_warning` | ATO is >3°C colder than tank (shock risk) |
| `fish_tank_temperature_sensor_error` | Any sensor reads <0°C or >50°C |

### Automations (6)
1. **Stratification alert** — Push notification when gradient exceeds threshold
2. **ATO cold water alert** — Warns before cold top-off causes temperature shock
3. **Sensor failure alert** — Alerts on unrealistic readings (high priority)
4. **Pi offline alert** — Fires after 5 min offline
5. **Apply tank calibration** — Sends offset to ESPHome when slider changes
6. **Apply sump calibration** — Sends offset to ESPHome when slider changes
7. **Apply ATO calibration** — Sends offset to ESPHome when slider changes

### Scripts
- `fish_tank_reset_temp_calibrations` — Resets all 3 offsets to 0°C

---

## 📊 Dashboard Cards

### Overview View Cards
1. **Main tank radial gauge** — Shows temperature as dial (range: 15–32°C)
2. **Temperature zones entity card** — All 3 temps + stats + alerts in one card
3. **24h history graph** — All 3 zones overlaid, colour coded (orange/blue/green)
4. **Gradient bar chart** — Visual colour-coded gradient indicators (green/orange/red)

### Settings View Cards
1. **Calibration sliders** — ±3°C per sensor with current calibrated reading shown
2. **Reset all calibrations button** — With confirmation dialog
3. **Monitoring settings** — Gradient threshold, alert toggles
4. **Raspberry Pi diagnostics** — Online status, IP, WiFi signal, CPU temp, ESPHome version, restart button
5. **Calibration instructions** — Markdown guide inline on the dashboard

---

## 🔧 Calibration Procedure

1. Place a trusted reference thermometer in tank, wait 5 minutes
2. Compare readings:
   - Reference: **24.7°C**
   - Sensor shows: **25.1°C**
   - Difference: **+0.4°C** (sensor reads high)
3. In Settings view, set Tank Calibration Offset to **-0.4**
4. Sensor now displays: 25.1 − 0.4 = **24.7°C** ✓
5. Repeat for Sump and ATO sensors
6. Calibration saved in Pi flash — survives power cuts

---

## 🚨 Alerts Explained

### Temperature Stratification
- **Trigger:** Tank and sump differ by more than 1.5°C (configurable)
- **Cause:** Poor circulation, return pump failure, heater in wrong location
- **Action:** Check return pump flow rate and heater placement

### ATO Cold Water Warning
- **Trigger:** ATO reservoir is >3°C colder than main tank
- **Cause:** Reservoir stored in cold area (garage, basement)
- **Action:** Warm reservoir to room temperature before top-off runs
- **Why it matters:** Sudden cold water addition stresses fish

### Sensor Error
- **Trigger:** Any probe reads below 0°C or above 50°C
- **Cause:** Loose wire, broken probe, water ingress into connector
- **Action:** Check wiring and probe connections

---

## 🖥️ Raspberry Pi Setup Summary

### Installation Steps
1. Flash Raspberry Pi OS Lite (64-bit) to microSD
2. Enable SSH, set hostname `fish-tank-temps`, configure WiFi
3. SSH in, add `dtoverlay=w1-gpio,gpiopin=4` to `/boot/config.txt`
4. Install ESPHome: `pip3 install esphome`
5. Create `~/esphome/secrets.yaml` with WiFi + MQTT credentials
6. Deploy config: `esphome run fish_tank_temps.yaml`
7. Find sensor addresses in logs, update config with correct addresses
8. Re-deploy with correct addresses
9. Add ESPHome integration in HA (Settings → Integrations → ESPHome → Pi IP)

### Auto-Start on Boot
```bash
sudo nano /etc/systemd/system/esphome-fish-tank.service
sudo systemctl enable esphome-fish-tank.service
sudo systemctl start esphome-fish-tank.service
```

### Finding Sensor Addresses
Run `esphome logs fish_tank_temps.yaml` and look for:
```
[dallas:074]: Found device at address 0x1C0000031EDD2A28
[dallas:074]: Found device at address 0x2D0000031F4A3B28
[dallas:074]: Found device at address 0x3E0000031D2F4C28
```
Identify which address = which zone by warming each probe in your hand.

---

## 🌡️ Celsius Conversion (Applied Later)

All files were updated from Fahrenheit to Celsius. Summary of changes:

### ESPHome (`fish_tank_temps.yaml`)
- Removed `lambda: return x * 9.0 / 5.0 + 32.0` conversion filters
- Changed `unit_of_measurement` from `°F` to `°C`

### Dashboard (`fish_tank_dashboard.yaml`)
- Gauge formatter: `val/100*30+60 °F` → `val/100*17+15 °C`
- Gauge transform: `(x - 60) / 30` → `(x - 15) / 17`
- History graph markers: `72`, `80` → `22`, `27`
- Settings labels updated to `°C`

### Helpers (`fish_tank_helpers.yaml`)
- Defaults: `72°F` / `80°F` → `22°C` / `27°C`
- Slider ranges: `60–90°F` → `10–35°C`

### Sensors (`fish_tank_sensors.yaml`)
- Alert thresholds: `72`, `80` → `22`, `27`

### Multipoint Package
- Calibration range: `±5°F` → `±3°C`
- Gradient threshold default: `3°F` → `1.5°C`
- ATO cold threshold: `>5°F` → `>3°C`
- All notification messages updated

---

## 🐛 Issues & Fixes During Session

### No issues were raised specific to the temperature system  
All files validated successfully with zero `°F` references remaining after conversion.

---

## 🔗 Integration with Main Fish Tank System

The multipoint temperature system integrates with the broader fish tank monitor:

- **Overview dashboard** — Temperature cards slotted into existing grid layout
- **Settings dashboard** — Calibration cards added to existing Settings view
- **Existing temperature alerts** — Multipoint data feeds into main water quality health score
- **Notifications** — Uses same `persistent_notification.create` pattern as rest of system
- **MQTT broker** — Shares same Mosquitto broker as Frigate camera system

---

## 📐 Sensor Placement Guide

| Sensor | Mount Method | Location | Avoid |
|--------|-------------|----------|-------|
| Main Tank | Suction cup on glass | Mid-tank height | Near heater, filter outlet |
| Sump | Zip-tie to baffle | Return pump chamber | Air stones |
| ATO | Float or suction cup | Always submerged | Dry areas |

---

## 💰 Cost Summary

| Item | Qty | Cost |
|------|-----|------|
| DS18B20 waterproof probes | 3 | ~$20 |
| 4.7kΩ resistor | 1 | $0.10 |
| Jumper wires | pack | ~$3 |
| Raspberry Pi 3B+ | 1 | owned |
| **Total** | | **~$23** |

---

## ✅ Final File Status

| File | YAML Valid | °F Remaining |
|------|-----------|--------------|
| `fish_tank_temps.yaml` | ✅ | 0 |
| `fish_tank_temperature_multipoint.yaml` | ✅ | 0 |
| `TEMPERATURE_DASHBOARD_CARDS.yaml` | ✅ | 0 |
| `fish_tank_dashboard.yaml` | ✅ | 0 |
| `fish_tank_helpers.yaml` | ✅ | 0 |
| `fish_tank_sensors.yaml` | ✅ | 0 |

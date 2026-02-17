# 💧 ATO System — Setup & Installation Guide

Complete guide for the Auto Top-Off system using Raspberry Pi 3B+, float valve, 8-channel optocoupler relay, and ATO pump.

---

## 🛒 Shopping List

| Item | Specification | Cost |
|------|--------------|------|
| Float valve switch | NC (Normally Closed), side-mount or vertical | £3–8 |
| 8-channel optocoupler relay | 5V, active LOW, opto-isolated | £5–10 |
| ATO pump | Submersible or inline, 12V DC or mains, ~100–300 L/h | £8–25 |
| 10kΩ resistor | Through-hole, 1/4W | £0.10 |
| Jumper wires | Female-female and female-male | £3 |
| Heat shrink / waterproof connectors | For pump wiring | £3 |

**Total: ~£20–50 depending on pump choice**

> The Raspberry Pi 3B+ and DS18B20 temperature sensors are already installed from the temperature monitoring setup.

---

## 🔌 Wiring

### Pi 3B+ GPIO Pin Reference

```
Pi Pin  GPIO    Current Use           New Use
──────────────────────────────────────────────────
1       3.3V    DS18B20 power         —
2       5V      —                     Relay board VCC  ← NEW
4       5V      —                     (spare)
6       GND     DS18B20 ground        —
7       GPIO4   DS18B20 1-wire data   —
11      GPIO17  —                     Float valve NC   ← NEW
12      GPIO18  —                     Relay CH1 IN     ← NEW
13      GPIO27  —                     RESERVED (future sump sensor)
14      GND     —                     Relay board GND + Float common ← NEW
```

### Float Valve Wiring

```
Float valve NC leg ──── GPIO17 (Pin 11)
                              │
                           10kΩ resistor
                              │
                           3.3V (Pin 1)

Float valve COM ──── GND (Pin 14)
```

**Why NC (Normally Closed)?**
- Float UP (tank full) = circuit OPEN = GPIO17 pulled HIGH by resistor = no trigger
- Float DOWN (tank low) = circuit CLOSED = GPIO17 pulled LOW = trigger pump
- If wire breaks = circuit open = pump stays OFF ✅ fail-safe

### Relay Board Wiring

```
Relay VCC  → Pi Pin 2  (5V)
Relay GND  → Pi Pin 14 (GND)
Relay IN1  → Pi GPIO18 (Pin 12)

Relay COM1 → Pump Live (brown wire for mains, + for DC)
Relay NO1  → Pump supply (Normally Open — pump OFF when relay inactive)
```

> ⚠️ **Mains voltage warning:** If using a mains-powered pump, use a proper enclosure, strain relief, and ensure the relay board is rated for mains switching. Consider a 12V DC pump instead for safety.

### Full Wiring Diagram (ASCII)

```
Pi 3B+                    Relay Board (8ch)
┌─────────┐               ┌──────────────────┐
│ Pin 2   │──── 5V ───────│ VCC              │
│ Pin 14  │──── GND ──────│ GND              │
│ GPIO18  │──── IN1 ──────│ IN1              │
│ (Pin12) │               │                  │
│         │               │ COM1 ──┐         │
│         │               │ NO1  ──┤──► Pump │
│         │               └────────┘         │
│ Pin 1   │──── 3.3V ──┐
│ (3.3V)  │            │ 10kΩ
│ GPIO17  │────────────┘
│ (Pin11) │──── Float NC leg
│ Pin 14  │──── Float COM (GND)
└─────────┘
```

---

## 🖥️ ESPHome Setup

### Step 1 — Merge addon into existing ESPHome config

Open `~/esphome/fish_tank_temps.yaml` on the Pi and add each section from `fish_tank_ato_addon.yaml`:

| Section in addon file | Where to merge |
|----------------------|----------------|
| `globals:` entries | Inside existing `globals:` block |
| `api: services:` entries | Inside existing `api: services:` list |
| `sensor:` entries | Inside existing `sensor:` block |
| `binary_sensor:` | New top-level section |
| `switch:` | New top-level section |
| `interval:` | New top-level section |
| `on_boot:` | New top-level section |

### Step 2 — Redeploy to the Pi

```bash
cd ~/esphome
esphome run fish_tank_temps.yaml
```

### Step 3 — Verify in Home Assistant

After redeployment, check these new entities appear:
- `binary_sensor.ato_float_valve`
- `binary_sensor.ato_pump_running_status`
- `binary_sensor.ato_pump_overtime_warning`
- `switch.ato_relay_pump`
- `sensor.ato_pump_runtime_this_cycle`

---

## 📦 Home Assistant Setup

### Step 1 — Install the package

Copy `fish_tank_ato.yaml` to `config/packages/fish_tank_ato.yaml`

Confirm packages are enabled in `configuration.yaml`:
```yaml
homeassistant:
  packages: !include_dir_named packages
```

### Step 2 — Restart Home Assistant

**Settings → System → Restart**

### Step 3 — Add dashboard cards

Open your fish tank dashboard in edit mode and paste the three card sections from `ATO_DASHBOARD_CARDS.yaml`:

| Section | Where to paste |
|---------|---------------|
| Section 1 — Overview Card | Existing Overview view |
| Section 2 — Full ATO View | New view called `ato` with path `/lovelace/ato` |
| Section 3 — Calibration Card | Existing Settings view |

### Step 4 — Initial configuration

Set these values on first run:

| Helper | Value | Where |
|--------|-------|-------|
| `ato_reservoir_capacity` | `25` (your container size in litres) | Settings calibration card |
| `ato_reservoir_minimum` | `5` (minimum before alert) | Settings calibration card |
| `ato_ml_per_activation_manual` | Your best estimate in ml (see below) | Settings calibration card |
| `ato_max_pump_runtime` | `60` (seconds — adjust to your typical cycle) | Settings calibration card |
| `ato_enabled` | Turn on | ATO view controls card |

**Setting initial ml estimate:**
Run the pump over a measuring jug for a typical top-off duration and measure the output. Enter this as the manual override. Auto-learn will replace it after 3 refill periods.

---

## 🧠 Auto-Learn Calibration

The system automatically learns the average ml per activation over time. No ongoing manual measurement needed.

### How it works

```
You refill reservoir → press "Mark Reservoir Refilled"
  ↓
HA calculates: total ml used ÷ total cycles = this period's average
  ↓
Adds to rolling history (default: last 3 refill periods)
  ↓
New learned value = average of rolling history
  ↓
Resets counters for next period
```

### Confidence progression

| State | Description |
|-------|-------------|
| **Not calibrated** | No data yet — set a manual estimate to start |
| **Low** | Fewer than 5 cycles this period |
| **Medium** | 5+ cycles, good estimate forming |
| **High** | Learned from multiple refill periods — most accurate |
| **Manual** | You have set a manual override — this always wins |

### First few days

For the first 1–3 refill periods set a rough manual override so the reservoir level tracking works from day one:
1. Guess ~150ml as starting point (adjust to your pump)
2. Auto-learn will build accuracy over refills
3. Once learned value appears and looks accurate, set manual back to `0`

---

## 🔒 Safety Features

### Layer 1 — ESPHome hardware (always active, even if HA is offline)

| Safety | Mechanism |
|--------|-----------|
| Pump OFF on boot | `restore_mode: ALWAYS_OFF` |
| 120s hard cutoff | Interval counter kills relay at 120s |
| Fail-safe wiring | NC float — wire break = pump stops |
| Active LOW relay | Pi reboot = relay de-energised = pump off |

### Layer 2 — Home Assistant software (configurable)

| Safety | Trigger | Action |
|--------|---------|--------|
| Overtime shutoff | Pump exceeds `ato_max_pump_runtime` | Kills relay + pauses ATO |
| Reservoir low | Level ≤ minimum | Blocks pump, sends alert |
| Reservoir critical | Level ≤ 2L | Auto-disables ATO entirely |
| Cold water block | ATO temp vs tank temp > 3°C | Blocks pump until warm |
| Cycle interval | Too soon since last cycle | Blocks pump |

---

## 🔧 Testing

### Test 1 — Relay wiring

1. Go to **ATO View → Controls**
2. Press **Test Pump (5 sec)**
3. Pump should run for 5 seconds then stop
4. Check for flow from pump outlet

### Test 2 — Float valve

1. In Developer Tools, check `binary_sensor.ato_float_valve`
2. Lift the float by hand → state should change to `on`
3. Release float → state should return to `off`

### Test 3 — Full ATO cycle

1. Manually trigger float valve (lift float) with ATO enabled
2. Pump should start within 5 seconds
3. Release float → pump should stop within 3 seconds
4. Check `sensor.ato_reservoir_level_litres` has decreased slightly

### Test 4 — Safety cutoff

1. Set `ato_max_pump_runtime` to `10` seconds temporarily
2. Trigger a cycle and let it run
3. Pump should stop at 10 seconds and ATO should pause
4. Press **Clear Safety Fault** to resume
5. Set `ato_max_pump_runtime` back to normal value

---

## 🐛 Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Float valve always `on` | NC/NO wiring wrong | Swap to NC leg on float or set `inverted: false` in ESPHome |
| Pump doesn't start | ATO disabled or paused | Check `ato_enabled` and `ato_paused` booleans |
| Pump runs continuously | Float not resetting | Check float is free to move, check wiring |
| Safety shutoff immediately | Max runtime too short | Increase `ato_max_pump_runtime` |
| Reservoir level wrong | ml per activation miscalibrated | Measure with jug, set manual override |
| Relay clicks but pump doesn't run | COM/NO wiring wrong | Swap COM and NO on relay terminal |
| ESPHome entities missing | Config not deployed | Run `esphome run fish_tank_temps.yaml` again |

---

## 📁 File Summary

| File | Location | Purpose |
|------|----------|---------|
| `fish_tank_ato_addon.yaml` | `~/esphome/` (Pi) | Merge into `fish_tank_temps.yaml` |
| `fish_tank_ato.yaml` | `config/packages/` | HA package — sensors, automations, scripts |
| `ATO_DASHBOARD_CARDS.yaml` | Reference | Dashboard cards (3 sections) |
| `ATO_SETUP_GUIDE.md` | Reference | This file |

---

## 📐 Reservoir Placement Tips

- Mount float valve so it triggers when sump is ~2–3cm below target level
- Ensure pump outlet tube reaches sump return section (not display tank directly)
- Keep reservoir out of direct sunlight (prevents algae, slows evaporation)
- If reservoir is cold (garage/cabinet), the cold water temperature block will fire — warm it or move it closer to the tank
- Label the reservoir clearly: **ATO — Do not use for water changes**

# 💧 ATO System — Complete Installation & Usage Guide

**Auto Top-Off System with Auto-Learning Reservoir Tracking**

---

## 📖 Table of Contents

1. [Overview](#overview)
2. [Shopping List](#shopping-list)
3. [Hardware Wiring](#hardware-wiring)
4. [ESPHome Installation](#esphome-installation)
5. [Home Assistant Installation](#home-assistant-installation)
6. [Initial Setup & Calibration](#initial-setup--calibration)
7. [Manual Calibration (New Feature)](#manual-calibration-new-feature)
8. [Dashboard Setup](#dashboard-setup)
9. [Daily Usage](#daily-usage)
10. [Safety Features](#safety-features)
11. [Troubleshooting](#troubleshooting)
12. [Maintenance & Tips](#maintenance--tips)

---

## 📋 Overview

### What This System Does

The ATO system automatically maintains your sump water level by:
1. **Float valve** in sump detects when water level is low
2. **Pump** adds water from a 25L reservoir
3. **Auto-learning algorithm** tracks exactly how much water is left in the reservoir
4. **Safety systems** prevent overflow, cold water shock, and pump failures

### Key Features

✅ **Auto-learning reservoir tracking** — no manual measurements after initial setup  
✅ **Manual calibration option** — for maximum accuracy from day one  
✅ **Temperature shock prevention** — blocks pump if reservoir water is too cold  
✅ **Dual safety layers** — ESPHome hardware + HA software redundancy  
✅ **Low reservoir alerts** — warns when refill is needed  
✅ **Daily evaporation tracking** — monitors water loss trends  

### How Reservoir Tracking Works

Instead of guessing based on pump flow rate (LPH), the system **measures actual usage**:

```
Normal operation:
  → Pump cycles multiple times per day
  → Each cycle: water used calculated from runtime
  → Reservoir level updated after each cycle
  
You refill reservoir:
  → Press "Mark Reservoir Refilled" button
  → System calculates: total_ml_used ÷ total_cycles = avg ml per cycle
  → Stores this in rolling history (last 3 refill periods)
  → Uses rolling average for future tracking
  
Result:
  → Accuracy improves automatically over time
  → Adapts to seasonal evaporation changes
  → Accounts for pump aging and wear
```

---

## 🛒 Shopping List

| Item | Specification | Approx Cost | Notes |
|------|--------------|-------------|-------|
| **Float valve switch** | NC (Normally Closed), side-mount or vertical | £3–8 | Get aquarium-specific if possible |
| **8-channel optocoupler relay** | 5V, active LOW, opto-isolated | £5–10 | 8-channel gives room for expansion |
| **ATO pump** | Submersible or inline, 12V DC or mains, 100–300 L/h | £8–25 | 12V DC safer than mains |
| **10kΩ resistor** | Through-hole, 1/4W | £0.10 | Pull-up for float valve |
| **Jumper wires** | Female-female and female-male, 20cm+ | £3 | Get a variety pack |
| **Heat shrink tubing** | Various sizes | £3 | For waterproofing connections |
| **25L reservoir container** | Food-safe plastic bucket with lid | £5–10 | Opaque to prevent algae |
| **Airline tubing** | 6mm ID, 2-3 meters | £3 | From reservoir to sump |
| **Check valve** (optional) | 6mm, one-way | £2 | Prevents siphon if pump fails |
| **Measuring jug** | 500ml or 1L, clear graduated | £3 | For calibration |

**Total: ~£30–60**

> The Raspberry Pi 3B+ is already installed from the temperature monitoring system.

---

## 🔌 Hardware Wiring

### GPIO Pin Allocation

Your Pi 3B+ already has temperature sensors on GPIO4. The ATO system uses GPIO17 and GPIO18:

```
Pi Pin   GPIO     Temperature Use          ATO Use
─────────────────────────────────────────────────────────────
1        3.3V     DS18B20 power            Float pull-up
2        5V       —                        Relay board VCC  ← NEW
6        GND      DS18B20 ground           —
7        GPIO4    DS18B20 1-wire data      —
11       GPIO17   —                        Float valve NC   ← NEW
12       GPIO18   —                        Relay CH1        ← NEW
13       GPIO27   —                        RESERVED (future)
14       GND      —                        Relay + Float GND ← NEW
```

### Float Valve Wiring

**Components needed:**
- Float valve with NC (Normally Closed) contacts
- 10kΩ resistor (external pull-up)
- 2× jumper wires

**Wiring diagram:**

```
Float Valve NC leg ──────────── GPIO17 (Pin 11)
                                     │
                                  10kΩ resistor (pull-up)
                                     │
                                  3.3V (Pin 1)

Float Valve COM ─────────────── GND (Pin 14)
```

**How it works:**
- Float UP (water OK) → NC contact OPEN → GPIO17 pulled HIGH → no trigger
- Float DOWN (needs water) → NC contact CLOSED → GPIO17 pulled LOW → trigger pump
- Wire break → contact OPEN → GPIO17 HIGH → **pump stays OFF** ✅ fail-safe

**Why external 10kΩ resistor?**
The Pi's internal pull-ups are weak (~50kΩ). An external 10kΩ resistor gives a stronger, more reliable pull-up signal.

### Relay Board Wiring

**Components needed:**
- 8-channel optocoupler relay board (active LOW)
- 3× jumper wires (VCC, GND, IN1)

**Wiring diagram:**

```
Relay Board              Raspberry Pi 3B+
────────────────         ─────────────────
VCC         ←────────    Pin 2  (5V)
GND         ←────────    Pin 14 (GND)
IN1         ←────────    GPIO18 (Pin 12)

Relay Terminal           ATO Pump
────────────────         ─────────────────
COM1        ←────────    Live wire (brown if mains / + if DC)
NO1         ←────────    Supply (black if mains / - if DC)
```

**Safety notes:**
- **NO = Normally Open** — pump is OFF when relay is inactive ✅
- **Active LOW** — GPIO18 LOW = relay ON (optocoupler board logic)
- If using **mains pump:** Use proper enclosure, ensure relay rated for mains (10A+)
- **Recommended:** Use 12V DC pump for safety

### ATO Pump & Tubing Setup

**Pump placement:**
- Submersible pump → inside reservoir
- Inline pump → outside reservoir with intake tube

**Tubing routing:**
1. **Pump outlet** → airline tubing (6mm)
2. **Check valve** (optional but recommended) → prevents siphon
3. **Tubing end** → sump return section (NOT display tank directly)

**Reservoir placement:**
- Keep reservoir out of direct sunlight (prevents algae)
- Position **below** sump level if possible (prevents siphon if tube breaks)
- Close to tank (shorter tubing = less head pressure = more accurate flow)

**Temperature considerations:**
The system monitors reservoir temperature and blocks pump if water is >3°C colder than tank. If reservoir is in a cold room/garage, consider:
- Insulating the reservoir
- Moving it to a warmer location
- Using a small aquarium heater in the reservoir

---

## 🖥️ ESPHome Installation

### Step 1 — Merge ATO Addon into Existing ESPHome Config

You already have `fish_tank_temps.yaml` running on the Pi for temperature monitoring. Now add the ATO hardware sections.

**On the Raspberry Pi:**

```bash
cd ~/esphome
nano fish_tank_temps.yaml
```

**Merge these sections from `fish_tank_ato_addon.yaml`:**

| Section in addon | Where to paste in fish_tank_temps.yaml |
|-----------------|----------------------------------------|
| `globals:` (2 entries) | Inside existing `globals:` block |
| `api: services:` (2 entries) | Inside existing `api: services:` list |
| `sensor:` (1 entry) | Inside existing `sensor:` block |
| `binary_sensor:` | **New top-level section** (paste at bottom) |
| `switch:` | **New top-level section** (paste at bottom) |
| `interval:` | **New top-level section** (paste at bottom) |
| `on_boot:` | **New top-level section** or merge with existing |

**Critical:** The addon file has clear `# MERGE INTO:` comments showing exactly where each section goes.

### Step 2 — Deploy to Raspberry Pi

```bash
cd ~/esphome
esphome run fish_tank_temps.yaml
```

Choose **Wirelessly** if Pi is already running ESPHome, or **USB** if this is first install.

Wait for compilation and upload (~2-3 minutes).

### Step 3 — Verify in Home Assistant

After deployment, these entities should appear in HA:

**Binary Sensors:**
- `binary_sensor.ato_float_valve` — Float valve state
- `binary_sensor.ato_pump_running_status` — Pump on/off
- `binary_sensor.ato_pump_overtime_warning` — Safety alert

**Switch:**
- `switch.ato_relay_pump` — Pump control

**Sensor:**
- `sensor.ato_pump_runtime_this_cycle` — Current cycle seconds

**Check:** Developer Tools → States → search "ato"

If entities are missing, check ESPHome logs:
```bash
esphome logs fish_tank_temps.yaml
```

---

## 📦 Home Assistant Installation

### Step 1 — Install Base ATO Package

Copy `fish_tank_ato.yaml` to your packages folder:

```bash
cp fish_tank_ato.yaml /config/packages/
```

Verify packages are enabled in `configuration.yaml`:
```yaml
homeassistant:
  packages: !include_dir_named packages
```

### Step 2 — Install Manual Calibration Addon (Optional but Recommended)

Merge sections from `ATO_MANUAL_CALIBRATION_ADDON.yaml` into `fish_tank_ato.yaml`:

| Section | Where to paste |
|---------|---------------|
| `input_number:` (2 entries) | Into existing `input_number:` block |
| `input_boolean:` (1 entry) | Into existing `input_boolean:` block |
| `input_datetime:` (1 entry) | Into existing `input_datetime:` block |
| `template: - sensor:` (2 entries) | Into existing `template: - sensor:` block |
| `script:` (3 scripts) | Into existing `script:` block |

### Step 3 — Restart Home Assistant

**Settings → System → Restart**

Wait for HA to fully restart (~1 minute).

### Step 4 — Verify Installation

Check these helpers exist:

**Developer Tools → States:**
- `input_number.ato_reservoir_capacity`
- `input_number.ato_ml_per_activation_manual`
- `input_boolean.ato_enabled`
- `sensor.ato_ml_per_activation`
- `binary_sensor.ato_safe_to_run`

If missing, check **Settings → System → Logs** for YAML errors.

---

## 🎯 Initial Setup & Calibration

### First-Time Configuration Checklist

**1. Set Reservoir Parameters**

Go to **Settings → ATO Calibration** (or use Developer Tools → Services)

```yaml
service: input_number.set_value
data:
  entity_id: input_number.ato_reservoir_capacity
  value: 25  # Your reservoir size in litres

service: input_number.set_value
data:
  entity_id: input_number.ato_reservoir_minimum
  value: 5  # Minimum level before alert (litres)
```

**2. Set Safety Limits**

```yaml
service: input_number.set_value
data:
  entity_id: input_number.ato_max_pump_runtime
  value: 60  # Max seconds per cycle (adjust to your typical cycle)

service: input_number.set_value
data:
  entity_id: input_number.ato_min_cycle_interval
  value: 5  # Min minutes between cycles
```

**3. Perform Manual Calibration** (Recommended)

See [Manual Calibration](#manual-calibration-new-feature) section below.

**4. Mark Reservoir as Full**

After filling the 25L reservoir:

```yaml
service: script.ato_reservoir_refilled
```

Or press the **"Mark Reservoir Refilled"** button on the dashboard.

**5. Enable ATO**

```yaml
service: input_boolean.turn_on
data:
  entity_id: input_boolean.ato_enabled
```

Or toggle **ATO System Enabled** in the dashboard.

---

## 🧪 Manual Calibration (New Feature)

Manual calibration gives the auto-learn system an accurate starting point. You have two options:

### Option 1: Guided Wizard (Easiest)

**Step-by-step automated process:**

1. **Settings → ATO Calibration → "Start Calibration Wizard"**
2. Read the prompt — you'll need a measuring jug (500ml or larger)
3. Place jug under the pump outlet
4. Wizard automatically runs a 30-second test
5. Measure the water collected in the jug
6. Enter ml value in **Settings → ATO Calibration → "Calibration Test Output"**
7. Wizard calculates and applies the result automatically
8. Done!

### Option 2: Manual Process (More Control)

**For users who want to customize test duration:**

1. **Set test duration** in Settings → ATO Calibration → "Test Duration"
   - Default: 30 seconds
   - Shorter (15s): quicker but less accurate
   - Longer (60s): more accurate, uses more reservoir water

2. **Prepare measuring jug**
   - Place under pump outlet
   - Ensure pump outlet is disconnected from sump tubing

3. **Press "Run Test"** button
   - Pump runs for configured duration
   - Measure water collected in jug

4. **Enter measured ml**
   - Go to Settings → ATO Calibration → "Measured Output"
   - Enter the ml value (e.g., 75ml)

5. **Check calculation**
   - "Calculated ml Per Activation" shows the result
   - Formula: `(measured_ml ÷ test_runtime) × max_runtime`
   - Example: `(75ml ÷ 30s) × 60s = 150ml per activation`

6. **Apply calibration**
   - Press "Apply Calibration" button
   - Manual override is now set
   - System uses this value for all reservoir tracking

### Understanding the Calculation

**The formula:**
```
ml_per_activation = (measured_ml ÷ test_runtime) × max_runtime
```

**Why this works:**
- Your test measured flow rate in ml/second
- Multiply by max_runtime to get ml per full cycle
- Even if cycles vary in length, the system calculates proportionally

**Example scenarios:**

| Test Duration | Measured ml | Max Runtime | Result |
|--------------|-------------|-------------|---------|
| 30s | 75ml | 60s | 150ml per activation |
| 30s | 60ml | 60s | 120ml per activation |
| 15s | 35ml | 60s | 140ml per activation |
| 60s | 150ml | 60s | 150ml per activation |

### When to Use Manual vs Auto-Learn

**Use manual calibration when:**
- ✅ First installation — gives accurate baseline from day one
- ✅ After pump replacement — flow rate changed
- ✅ Tube length or height changed — affects head pressure
- ✅ Auto-learn seems inaccurate — manual as ground truth

**Switch to auto-learn after:**
- 3-4 refill periods have passed
- Auto-learn has built a solid rolling average
- Set manual override to `0` to activate auto-learn

**Best practice workflow:**
1. **Day 1:** Manual calibration → 150ml
2. **Week 1-4:** Use manual value (accurate from start)
3. **After 4 refills:** Auto-learn has 3+ data points
4. **Set manual to 0:** Auto-learn takes over
5. **Ongoing:** Auto-learn adapts to pump wear, seasonal changes

---

## 🎨 Dashboard Setup

### Install Dashboard Cards

The dashboard has **3 sections** to add:

**From `ATO_DASHBOARD_CARDS.yaml`:**

| Section | View | Card Type |
|---------|------|-----------|
| **Section 1** | Overview | Summary status card |
| **Section 2** | ATO (new tab) | 7 cards — full view |
| **Section 3** | Settings | Calibration + settings |

**From `ATO_MANUAL_CALIBRATION_ADDON.yaml`:**

| Section | View | Card Type |
|---------|------|-----------|
| **Calibration card** | Settings | Manual calibration controls |
| **Calibration guide** | Settings | Instructions |

### Add Overview Card (Section 1)

1. Edit dashboard (⋮ → Edit dashboard)
2. Navigate to **Overview** view
3. Click **+ ADD CARD**
4. Switch to **code editor** (⋮ → Show code editor)
5. Paste the card from Section 1
6. Save

The card shows:
- Current status (Standby / Pumping / Low / etc)
- Reservoir level %
- Tap → navigate to full ATO view
- Hold → mark reservoir refilled (with confirmation)

### Add Full ATO View (Section 2)

1. Edit dashboard
2. Click **+ ADD VIEW**
3. Configure:
   - Title: `ATO`
   - Path: `ato`
   - Icon: `mdi:water-pump`
   - Type: `Masonry`
4. Switch to code editor for this view
5. Paste **all cards** from Section 2
6. Save

The view includes:
- Reservoir level gauge (radial % display)
- Status card (all sensors, cycle info)
- System controls (enable, pause, refill, test)
- Safety status (all alerts + safe_to_run reason)
- Temperature context (ATO vs tank temp)
- 7-day level history graph
- 14-day daily top-off chart

### Add Settings Cards (Section 3 + Calibration)

1. Edit dashboard
2. Navigate to **Settings** view
3. Add cards from:
   - `ATO_DASHBOARD_CARDS.yaml` Section 3
   - `ATO_MANUAL_CALIBRATION_ADDON.yaml` calibration cards

Settings view shows:
- All configuration sliders
- Manual calibration wizard
- Test pump buttons
- Setup instructions

---

## 📅 Daily Usage

### Normal Operation

**The system runs automatically:**

1. **Float detects low water** → triggers after 5 seconds
2. **Safety checks pass** → `ato_safe_to_run` = true
3. **Pump starts** → relay activated
4. **Float satisfied** → triggers after 3 seconds off
5. **Pump stops** → relay deactivated
6. **Tracking updated:**
   - Cycle runtime recorded
   - ml dispensed calculated
   - Reservoir level decremented
   - Daily total incremented
   - Live average updated

**You do nothing** — system handles it all.

### When to Refill Reservoir

**Watch for alerts:**
- **"Reservoir Low"** — level at minimum (5L default)
- **"Reservoir Critical"** — level ≤2L, ATO auto-disabled

**Dashboard indicators:**
- Reservoir % card turns orange/red
- Days remaining approaches 0
- Cycles remaining shows low number

**Refill process:**
1. Fill reservoir to 25L (or your capacity)
2. Press **"Mark Reservoir Refilled"** button
3. Auto-learn triggers (if 3+ cycles since last refill)
4. ATO re-enabled automatically
5. Counters reset for next period

**Refill frequency:**
- Depends on evaporation rate
- Typical: 10-20 days for 25L reservoir
- Dashboard shows "Days Remaining" estimate

### Monitoring Dashboard

**Check daily:**
- **Status** — should show "Standby" most of the time
- **Reservoir %** — trending down slowly
- **Daily top-off** — should be consistent day-to-day

**Check weekly:**
- **7-day level graph** — smooth downward slope
- **14-day daily chart** — consistent daily totals
- **Days remaining** — plan refill accordingly

**Red flags:**
- Rapid level drop → possible leak or siphon
- Pump running frequently → float valve stuck
- No cycles for days → float valve stuck or wiring issue
- Daily totals increasing — evaporation rising (temperature or humidity change)

---

## 🔒 Safety Features

The system has **dual safety layers** protecting against failures:

### Layer 1 — ESPHome Hardware (Always Active)

**These run on the Pi, work even if Home Assistant is offline:**

| Feature | Implementation | Protection |
|---------|---------------|------------|
| **120s hard cutoff** | Interval counter in ESPHome | Pump killed at exactly 120s runtime |
| **Pump OFF on boot** | `restore_mode: ALWAYS_OFF` | Never auto-starts after power cut |
| **Fail-safe float wiring** | NC (Normally Closed) circuit | Wire break = pump stops |
| **Active LOW relay** | Optocoupler board logic | Pi crash = relay de-energised = pump off |

**Test Layer 1:**
```yaml
# Let pump run and watch it auto-stop at 120s
# Check ESPHome logs for "ATO EMERGENCY STOP" message
```

### Layer 2 — Home Assistant Software (Configurable)

**These run in HA automations, more flexible but depend on HA being online:**

| Feature | Trigger | Action |
|---------|---------|--------|
| **Overtime shutoff** | Runtime > `ato_max_pump_runtime` | Pump off + ATO paused + notification |
| **Reservoir minimum** | Level ≤ minimum (5L default) | Pump blocked via `ato_safe_to_run` |
| **Reservoir critical** | Level ≤ 2L | ATO auto-disabled + urgent notification |
| **Temperature shock** | ATO temp vs tank > 3°C | Pump blocked + notification |
| **Cycle interval** | Too soon since last cycle | Pump blocked (prevents rapid cycling) |

**The `ato_safe_to_run` Gate:**

All conditions must pass before pump can start:
- ✅ `ato_enabled` = on
- ✅ `ato_paused` = off
- ✅ `ato_reservoir_low` = off
- ✅ `ato_pump_overtime` = off
- ✅ `fish_tank_ato_water_cold_warning` = off
- ✅ Time since last cycle > 5 minutes (configurable)

If ANY fails, the `reason` attribute shows which one.

### Temperature Shock Prevention

The system uses the multi-point temperature monitoring to prevent cold water shock:

**How it works:**
1. `sensor.fish_tank_ato_temperature` — reservoir water temp
2. `sensor.fish_tank_temperature` — main tank temp
3. `sensor.fish_tank_ato_tank_gradient` — difference calculated
4. `binary_sensor.fish_tank_ato_water_cold_warning` — triggers if >3°C difference
5. `ato_safe_to_run` includes this check

**If blocked:**
- Notification explains: "ATO water too cold"
- Shows both temperatures
- ATO waits until reservoir warms up

**Solutions:**
- Move reservoir to warmer location
- Add small aquarium heater to reservoir
- Insulate reservoir container
- Wait for room temperature to rise

### Safety Testing Checklist

**Test 1 — Float valve:**
```
1. Lift float manually
2. Check binary_sensor.ato_float_valve changes to "on"
3. Release float
4. Check binary_sensor.ato_float_valve returns to "off"
✓ Pass if state changes correctly
```

**Test 2 — Pump control:**
```
1. Press "Test Pump (5 sec)" button
2. Pump should run exactly 5 seconds
3. Check for water flow from outlet
✓ Pass if pump runs and stops automatically
```

**Test 3 — Safety cutoff:**
```
1. Temporarily set ato_max_pump_runtime to 10s
2. Trigger a cycle
3. Pump should stop at 10s
4. ATO should pause automatically
5. Check notification appears
6. Press "Clear Safety Fault" button
7. Set ato_max_pump_runtime back to 60s
✓ Pass if pump stops and ATO pauses
```

**Test 4 — Reservoir minimum:**
```
1. Set ato_reservoir_current_level to 4.5L (below 5L minimum)
2. Try to trigger float
3. Pump should NOT start
4. Check ato_safe_to_run shows "Reservoir at minimum"
5. Set reservoir level back to normal
✓ Pass if pump blocked when reservoir low
```

**Test 5 — Cold water block:**
```
1. Check sensor.fish_tank_ato_temperature
2. If >3°C colder than tank, block should be active
3. Check binary_sensor.fish_tank_ato_water_cold_warning = on
4. Try to trigger float — pump should not start
5. Check notification explains temperature difference
✓ Pass if pump blocked when water too cold
```

---

## 🐛 Troubleshooting

### Float Valve Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| Float always shows "on" | NC/NO wiring reversed | Swap NC and NO connections on float valve |
| Float never triggers | Wire break or resistor missing | Check continuity, verify 10kΩ resistor present |
| Float intermittent | Loose connection | Re-solder connections, add heat shrink |
| Float stuck | Debris or calcium buildup | Clean float mechanism, check it moves freely |

**Test float directly:**
```bash
# SSH into Pi running ESPHome
# Watch GPIO17 state while moving float
gpio mode 11 in
gpio mode 11 up  # enable pull-up
watch -n 0.1 gpio read 11
# Should show 1 when float up, 0 when float down
```

### Pump Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| Pump won't start | ATO disabled or paused | Check `ato_enabled` = on, `ato_paused` = off |
| Pump runs continuously | Float not resetting | Check float moves freely, verify wiring |
| Relay clicks but no pump | COM/NO wiring swapped | Swap COM and NO terminals on relay |
| Weak flow | Clogged intake or check valve | Clean pump intake filter, check valve |
| No flow but pump running | Airlock or dry pump | Prime pump — submerge in water, run until water flows |

**Test pump directly:**
```yaml
# Bypass all automation, manually control relay
service: switch.turn_on
entity_id: switch.ato_relay_pump
# Wait 5 seconds, check for flow
service: switch.turn_off
entity_id: switch.ato_relay_pump
```

### Reservoir Tracking Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| Level drops too fast | ml_per_activation too high | Run manual calibration again |
| Level drops too slow | ml_per_activation too low | Run manual calibration again |
| Level incorrect after refill | Forgot to press refill button | Press "Mark Reservoir Refilled" |
| Auto-learn not updating | Fewer than 3 cycles per period | Use system normally, wait for more cycles |
| Days remaining shows "0" | Insufficient data | Wait until daily total > 10ml |

**Reset reservoir tracking:**
```yaml
# If tracking is completely wrong
service: script.ato_reset_autolearn
# Then run manual calibration
# Then mark reservoir as refilled
```

### Safety System Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| Overtime shutoff immediately | max_runtime too short | Increase `ato_max_pump_runtime` slider |
| Pump runs too long | max_runtime too high | Decrease `ato_max_pump_runtime` slider |
| Cold water block constant | Reservoir in cold location | Move reservoir, add heater, or insulate |
| Rapid cycling blocked | min_interval too high | Decrease `ato_min_cycle_interval` |

**Check safe_to_run status:**
```yaml
# View which safety check is failing
entity: binary_sensor.ato_safe_to_run
attributes:
  reason: "Reservoir at minimum"  # example
# Fix the issue listed in "reason"
```

### ESPHome Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| Entities not appearing in HA | ESPHome not deployed | Run `esphome run fish_tank_temps.yaml` |
| "Unavailable" in HA | Pi offline or ESPHome crashed | Check Pi is powered, SSH in and check logs |
| Sensors stuck | ESPHome frozen | Restart Pi: `sudo reboot` |
| Upload fails | Wrong IP or Pi not responding | Check Pi IP, try USB cable upload |

**Check ESPHome logs:**
```bash
ssh pi@192.168.1.150  # Your Pi IP
cd ~/esphome
esphome logs fish_tank_temps.yaml
# Watch for errors, warnings, or crash messages
```

---

## 🛠️ Maintenance & Tips

### Weekly Checks

**Every Sunday (or your preferred day):**

1. **Check reservoir level** — compare dashboard % with actual level in container
   - If off by >10%, run manual calibration
2. **Inspect float valve** — ensure moves freely, no debris
3. **Check pump intake** — clean filter if clogged
4. **Review 7-day graph** — level should drop smoothly
5. **Review 14-day chart** — daily totals should be consistent

### Monthly Maintenance

**Once per month:**

1. **Clean float valve**
   - Remove from sump
   - Rinse in fresh water
   - Check for calcium buildup
   - Reinstall and verify operation

2. **Inspect relay board**
   - Check for corrosion on terminals
   - Verify no water damage
   - Test relay clicks audibly when triggered

3. **Check all connections**
   - Verify jumper wires secure
   - Check pump connections waterproof
   - Inspect tubing for cracks or algae

4. **Reservoir deep clean**
   - Empty reservoir completely
   - Scrub with vinegar to remove any biofilm
   - Rinse thoroughly
   - Refill with fresh RO/DI water
   - Press "Mark Reservoir Refilled"

### Seasonal Adjustments

**Summer (higher evaporation):**
- Daily top-off will increase
- Refill reservoir more frequently
- Auto-learn adapts automatically over 2-3 refills
- Consider larger reservoir (50L) if refilling too often

**Winter (lower evaporation):**
- Daily top-off will decrease
- Refill less frequently
- Auto-learn adapts automatically
- Watch for cold water blocking if reservoir in garage/cold room

### Optimizing Performance

**Fine-tune max_runtime:**

The `ato_max_pump_runtime` setting affects accuracy:
- **Too short** (e.g., 15s) — pump cuts off mid-cycle, multiple cycles per top-off
- **Too long** (e.g., 120s) — pump rarely hits timeout, but fails to protect against stuck float
- **Sweet spot** — Set to ~90% of your typical cycle duration

**How to find your typical cycle:**
1. Watch a few natural cycles
2. Note runtime in `sensor.ato_current_cycle_runtime`
3. Typical range: 20-60 seconds
4. Set max_runtime to (typical + 10-20 seconds)

**Example:**
- Cycles typically run 35-45 seconds
- Set max_runtime to 60 seconds
- Gives 20s safety margin

**Optimize reservoir placement:**

Best practices:
- ✅ Close to tank (short tubing = less head pressure)
- ✅ Below sump level (gravity prevents siphon)
- ✅ Out of direct sunlight (prevents algae)
- ✅ Temperature-controlled room (prevents cold water blocking)
- ✅ Easy access for refilling (you'll do this often)
- ✅ Opaque container (light-tight prevents algae)

**Refill schedule optimization:**

Use dashboard data to plan refills:
1. Check "Days Remaining" sensor
2. Schedule refill when showing 3-5 days
3. Don't wait until "critical" — wastes auto-learn data
4. Consistent refill intervals → better auto-learn accuracy

### Advanced Tips

**Use historical data:**

The `ato_refill_history_json` helper stores rolling average data:
```json
["127.3", "134.5", "119.8"]
```
This is ml per activation for last 3 refill periods.

**Customize auto-learn window:**

Default: 3 refill periods

Increase to 5-7 if:
- Evaporation highly variable (seasonal, weather-dependent)
- Want smoother averaging

Decrease to 1-2 if:
- Evaporation very consistent
- Want faster adaptation to changes

**Manual override strategy:**

Good reasons to use manual override:
1. **First month** — gives accurate baseline while auto-learn builds
2. **After pump replacement** — new pump, new flow rate
3. **Precision testing** — you've done multiple timed tests and have exact value

Bad reasons:
- Auto-learn "seems off" but haven't verified with actual measurement
- Impatience — auto-learn needs 3 refills to stabilize

**Monitoring evaporation trends:**

Track `sensor.ato_daily_topoff_ml` over weeks:
- Increasing → tank getting warmer or more air circulation
- Decreasing → tank cooling or less air movement
- Sudden spike → possible leak or siphon
- Sudden drop → possible float valve sticking

**Integration with other systems:**

The ATO provides data other automations can use:
- If reservoir critical → pause fertilizer dosing (no point if no water to dilute)
- If daily top-off doubles → send alert (possible leak)
- If reservoir refilled → good time to log full tank parameters

---

## 📚 Additional Resources

### Files Included

| File | Purpose |
|------|---------|
| `fish_tank_ato.yaml` | Main HA package |
| `fish_tank_ato_addon.yaml` | ESPHome hardware config (merge into fish_tank_temps.yaml) |
| `ATO_MANUAL_CALIBRATION_ADDON.yaml` | Manual calibration feature (merge into fish_tank_ato.yaml) |
| `ATO_DASHBOARD_CARDS.yaml` | All dashboard cards |
| `ATO_SYSTEM_SUMMARY.md` | Technical summary |
| `ATO_SETUP_GUIDE.md` | Original setup guide |

### Related Documentation

- **Multi-point temperature system** — provides ATO reservoir temperature sensor
- **Frigate camera system** — can monitor sump visually for overflow
- **Fish tank sensors** — water quality data integrates with ATO tracking

### Getting Help

**Check logs first:**
- ESPHome: `esphome logs fish_tank_temps.yaml`
- Home Assistant: Settings → System → Logs

**Common log messages:**
```
ERROR ATO EMERGENCY STOP — pump exceeded 120s hard limit
  → ESPHome safety triggered, check float valve

WARNING ATO cycle blocked — cold water risk
  → Reservoir too cold, warm it up

INFO ATO auto-learn updated — new learned value: 126.3ml
  → Normal operation after refill
```

**Test each component:**
1. Float valve (manual lift test)
2. Relay (direct GPIO test)
3. Pump (bypass automation test)
4. Safety system (deliberate trigger test)
5. Tracking (manual calibration verification)

---

## ✅ Quick Reference

### Daily Commands

**Check status:**
```yaml
entity: sensor.ato_status
# Should show: Standby
```

**View reservoir:**
```yaml
entity: sensor.ato_reservoir_level_litres
entity: sensor.ato_reservoir_level_percent
entity: sensor.ato_reservoir_days_remaining
```

**Manual control (emergency):**
```yaml
# Turn pump on
service: switch.turn_on
entity_id: switch.ato_relay_pump

# Turn pump off
service: switch.turn_off
entity_id: switch.ato_relay_pump
```

### Refill Procedure

1. Fill reservoir to capacity
2. Press "Mark Reservoir Refilled" button
3. Verify `ato_reservoir_current_level` = capacity
4. Check auto-learn notification (if 3+ cycles)
5. Done!

### Calibration Procedure

**Guided wizard:**
1. Settings → ATO Calibration → "Start Calibration Wizard"
2. Follow prompts
3. Measure water in jug
4. Enter ml value
5. Done!

**Manual:**
1. Set test duration
2. Press "Run Test"
3. Measure ml in jug
4. Enter in "Measured Output"
5. Press "Apply Calibration"
6. Done!

### Emergency Procedures

**Pump won't stop:**
```yaml
# Immediate
service: switch.turn_off
entity_id: switch.ato_relay_pump

# If that fails, SSH into Pi:
ssh pi@192.168.1.150
# Kill ESPHome process
sudo systemctl stop esphome

# Last resort: power off Pi physically
```

**Sump overflowing:**
```yaml
# Immediately disable ATO
service: input_boolean.turn_off
entity_id: input_boolean.ato_enabled

# Then troubleshoot float valve
```

**Reservoir empty but HA shows full:**
```yaml
# Reset to actual level
service: input_number.set_value
data:
  entity_id: input_number.ato_reservoir_current_level
  value: 0  # or actual measured level

# Then refill and press refill button
```

---

**System is now fully documented and ready for installation!**

# 🌡 Multi-Point Temperature Monitoring — Complete Setup Guide
**Raspberry Pi 3B+ with MQTT Integration**

---

## What You're Building

A professional-grade multi-zone temperature monitoring system that tracks:
- **Main Tank** — Primary fish environment
- **Sump** — Filtration system temperature
- **ATO Reservoir** — Top-off water temperature

With features:
✅ **Individual calibration** for each sensor (adjustable from dashboard)  
✅ **Gradient monitoring** — Detects circulation problems  
✅ **Cold water warnings** — Prevents temperature shock from ATO  
✅ **Sensor failure detection** — Alerts if sensor malfunctions  
✅ **MQTT integration** — Real-time updates to Home Assistant  
✅ **Persistent calibration** — Survives power outages  

---

## Hardware Shopping List

| Item | Quantity | Cost | Link |
|------|----------|------|------|
| Raspberry Pi 3B+ | 1 | $35-45 | Already owned ✓ |
| DS18B20 Waterproof Temp Probe | 3 | $6-8 each | Amazon/AliExpress |
| 4.7kΩ Resistor | 1 | $0.10 | Electronics store |
| MicroSD Card (16GB+) | 1 | $8 | If not included with Pi |
| Jumper Wires (Female-Female) | 6 | $3 | Electronics kit |
| **Total** | | **~$45** | (excluding owned Pi) |

### Recommended DS18B20 Kit
Search Amazon for: "DS18B20 waterproof temperature sensor kit 3m cable"
- Comes with waterproof stainless steel probe
- 3-meter cable (enough to reach anywhere in tank)
- Pre-wired connector

---

## Part 1: Hardware Wiring

### DS18B20 Wire Colors

```
RED    → Power (3.3V)
BLACK  → Ground (GND)
YELLOW → Data (GPIO4)
```

### Raspberry Pi GPIO Pinout

```
        ┌─────────────┐
    3.3V │1  ╔═══╗  2│ 5V
        │3  ║ R ║  4│ 5V
        │5  ║ P ║  6│ GND  ←── All BLACK wires
 GPIO4  │7  ║ i ║  8│
     GND │9  ║   ║ 10│
        │11 ║ 3 ║ 12│
        │13 ║ B ║ 14│ GND
        │15 ║ + ║ 16│
    3.3V │17 ║   ║ 18│
        └───╚═══╝────┘
```

### Complete Wiring Diagram

```
                  ┌─ 4.7kΩ Resistor ─┐
                  │                   │
    [Pi Pin 1] ───┴──── RED ──────────┴──── [All 3 Sensors RED]
    [Pi Pin 6] ───────── BLACK ──────────── [All 3 Sensors BLACK]
    [Pi Pin 7] ───────── YELLOW ─────────── [All 3 Sensors YELLOW]
                          ↑
                       GPIO4
```

**Pull-up Resistor Placement:**
- One end of 4.7kΩ resistor connects to **3.3V (Pin 1)**
- Other end connects to **YELLOW wire (GPIO4 / Pin 7)**

### Step-by-Step Wiring

1. **Power off** Raspberry Pi completely
2. **Connect RED wires:**
   - All 3 sensor RED wires → Pi Pin 1 (3.3V)
   - You can use a breadboard or twist wires together
3. **Connect BLACK wires:**
   - All 3 sensor BLACK wires → Pi Pin 6 (GND)
4. **Connect YELLOW wires:**
   - All 3 sensor YELLOW wires → Pi Pin 7 (GPIO4)
5. **Add pull-up resistor:**
   - 4.7kΩ resistor between Pin 1 (3.3V) and Pin 7 (GPIO4/YELLOW)

### Sensor Labeling

Before sealing everything up, **label each sensor** so you know which is which:
- **RED ribbon** → Main Tank probe
- **BLUE ribbon** → Sump probe
- **GREEN ribbon** → ATO Reservoir probe

Take a photo of your labels for reference!

---

## Part 2: Raspberry Pi OS Setup

### Option A: Fresh Install (Recommended)

1. **Download** Raspberry Pi Imager: https://www.raspberrypi.com/software/
2. **Choose OS**: Raspberry Pi OS (64-bit) Lite (no desktop needed)
3. **Configure** (click gear icon):
   - Set hostname: `fish-tank-temps`
   - Enable SSH
   - Set username: `pi`, password: (your choice)
   - Configure WiFi SSID and password
   - Set timezone
4. **Write** to microSD card
5. **Insert** card into Pi and power on
6. **Find Pi IP** address:
   ```bash
   # From your computer:
   ping fish-tank-temps.local
   # OR check your router's DHCP leases
   ```

### Option B: Existing Pi Setup

If Pi is already running, just make sure:
- SSH is enabled: `sudo raspi-config` → Interface Options → SSH → Enable
- 1-Wire is enabled (we'll do this in next step)

---

## Part 3: Enable 1-Wire Bus

SSH into your Pi:
```bash
ssh pi@192.168.1.150  # Use your Pi's IP
```

Enable 1-wire interface:
```bash
# Edit boot config
sudo nano /boot/config.txt

# Add this line at the end:
dtoverlay=w1-gpio,gpiopin=4

# Save: Ctrl+X, Y, Enter

# Reboot
sudo reboot
```

After reboot, SSH back in and verify 1-wire is working:
```bash
# Load kernel modules
sudo modprobe w1-gpio
sudo modprobe w1-therm

# Check for sensors
ls /sys/bus/w1/devices/

# You should see 3 folders like:
# 28-000008a1b234  28-000008a1c567  28-000008a1d890
# These are your 3 DS18B20 sensors!
```

---

## Part 4: Install ESPHome on Raspberry Pi

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python pip
sudo apt install python3-pip -y

# Install ESPHome
pip3 install esphome

# Add to PATH
echo 'export PATH=$PATH:~/.local/bin' >> ~/.bashrc
source ~/.bashrc

# Verify installation
esphome version
```

---

## Part 5: Create ESPHome Configuration

```bash
# Create ESPHome config directory
mkdir -p ~/esphome

# Create secrets file
nano ~/esphome/secrets.yaml
```

**Paste this into `secrets.yaml`:**
```yaml
wifi_ssid: "YourWiFiName"
wifi_password: "YourWiFiPassword"
api_encryption_key: "GENERATE_NEW_KEY_HERE"  # See note below
ota_password: "fishtank123"
mqtt_username: "homeassistant"
mqtt_password: "your_mqtt_password"
```

**Generate API encryption key:**
```bash
esphome wizard fish_tank_temps.yaml
# Follow prompts, it will generate a key for you
# Copy the key into secrets.yaml
```

**Create main config:**
```bash
cd ~/esphome
```

**Copy the `fish_tank_temps.yaml` file** from the package files into `~/esphome/fish_tank_temps.yaml`

---

## Part 6: Find Sensor Addresses

**Compile and run ESPHome to discover sensor addresses:**

```bash
cd ~/esphome
esphome run fish_tank_temps.yaml
```

Watch the logs — you'll see lines like:
```
[dallas:074]: Found device at address 0x1C0000031EDD2A28
[dallas:074]: Found device at address 0x2D0000031F4A3B28
[dallas:074]: Found device at address 0x3E0000031D2F4C28
```

**Copy these addresses!**

Now **identify which sensor is which:**
1. Hold **RED-ribbon sensor** (tank) in your hand — watch which address temperature rises
2. Note: That address = Main Tank
3. Repeat for BLUE (sump) and GREEN (ato) sensors

**Edit `fish_tank_temps.yaml`** and replace placeholder addresses on lines 79, 98, 117:
```yaml
# Line 79 - Main Tank
address: 0x1C0000031EDD2A28  # ← YOUR TANK SENSOR ADDRESS

# Line 98 - Sump
address: 0x2D0000031F4A3B28  # ← YOUR SUMP SENSOR ADDRESS

# Line 117 - ATO
address: 0x3E0000031D2F4C28  # ← YOUR ATO SENSOR ADDRESS
```

---

## Part 7: Deploy to Raspberry Pi

```bash
cd ~/esphome
esphome run fish_tank_temps.yaml
```

This will:
1. Compile the firmware
2. Upload to the Pi
3. Start logging

You should see temperature readings every 10 seconds!

---

## Part 8: Home Assistant Integration

### Enable MQTT in Home Assistant

If not already enabled:
1. **Supervisor → Add-on Store → Mosquitto broker**
2. **Install** and **Start**
3. **Settings → Integrations → Add MQTT**
4. Accept defaults (auto-discovers broker)

### Add ESPHome Integration

1. **Settings → Integrations → Add Integration**
2. Search: **ESPHome**
3. **Host**: `192.168.1.150` (your Pi IP)
4. **Port**: `6053` (default)
5. **Submit**

**New entities appear:**
```
sensor.fish_tank_temperature
sensor.sump_temperature
sensor.ato_reservoir_temperature
sensor.tank_sump_temperature_delta
sensor.pi_cpu_temperature
sensor.temperature_monitor_wifi_signal
binary_sensor.temperature_monitor_status
binary_sensor.temperature_stratification_alert
switch.temperature_monitor_restart
```

---

## Part 9: Install HA Package

1. **Copy** `fish_tank_temperature_multipoint.yaml` → `config/packages/`
2. **Restart** Home Assistant

**New entities appear:**
```
# Calibration helpers
input_number.fish_tank_temp_calibration
input_number.fish_tank_sump_temp_calibration
input_number.fish_tank_ato_temp_calibration

# Calibrated sensors
sensor.fish_tank_temperature (calibrated)
sensor.fish_tank_sump_temperature (calibrated)
sensor.fish_tank_ato_temperature (calibrated)

# Statistics
sensor.fish_tank_temperature_average
sensor.fish_tank_temperature_range
sensor.fish_tank_tank_sump_gradient
sensor.fish_tank_ato_tank_gradient

# Alerts
binary_sensor.fish_tank_temperature_stratification
binary_sensor.fish_tank_ato_water_cold_warning
binary_sensor.fish_tank_temperature_sensor_error
```

---

## Part 10: Add Dashboard Cards

**Overview View:**
1. Edit dashboard → ⋮ → Raw Config Editor
2. Find the `cards:` section in the Overview view
3. Paste cards from `TEMPERATURE_DASHBOARD_CARDS.yaml` (Overview section)
4. Save

**Settings View:**
1. Scroll to Settings view in Raw Config Editor
2. Find the `cards:` section
3. Paste cards from `TEMPERATURE_DASHBOARD_CARDS.yaml` (Settings section)
4. Save

---

## Part 11: Calibrate Sensors

**You'll need:** A trusted reference thermometer

**Procedure:**
1. Place reference thermometer in main tank
2. Wait 5 minutes for stabilization
3. Compare readings:
   - Reference: **76.5°F**
   - Dashboard sensor: **77.2°F**
   - Difference: **+0.7°F** (sensor reads high)
4. Go to **Settings view** → Temperature Calibration
5. Set "Tank Temp Calibration Offset" to **-0.7**
   - (negative because sensor reads high)
6. Dashboard now shows: 77.2 - 0.7 = **76.5°F** ✓

Repeat for Sump and ATO sensors.

**Calibration is saved** in ESPHome and persists across reboots.

---

## Part 12: Set Up Automations

**Update notification service** in the package file:

Edit `fish_tank_temperature_multipoint.yaml` and replace:
```yaml
notify.mobile_app_your_phone
```

With your actual notifier (find at Developer Tools → Services).

**Restart HA** to activate automations.

---

## Testing & Verification

### Test 1: Sensor Readings

All 3 temperatures should be within **2°F** of each other (if all in same room temp):
```
Tank: 76.5°F
Sump: 76.8°F
ATO:  75.2°F  ← May be cooler if in unheated area
```

### Test 2: Calibration

Adjust a calibration slider → sensor value should update within 1 second.

### Test 3: Stratification Alert

Temporarily set gradient threshold to 0.5°F → alert should trigger.

### Test 4: Hot/Cold Test

Hold a sensor in your hand → temperature should rise → test alerts.

---

## Mounting Sensors

### Main Tank
- **Suction cup mount** on inside glass (mid-tank height)
- Keep away from heater and filter outflow (false readings)
- Secure cable with aquarium-safe clips

### Sump
- **Zip-tie** to return pump chamber baffle
- Submerge probe fully
- Keep cable organized with cable clips

### ATO Reservoir
- **Float** probe in water using foam collar
- OR suction cup to bucket side
- Ensure probe is always submerged

---

## Troubleshooting

### No sensors detected

**Check:**
```bash
ls /sys/bus/w1/devices/
```

Should show 3 folders `28-xxxxx`. If empty:
- Verify wiring (especially 4.7kΩ resistor)
- Check `/boot/config.txt` has `dtoverlay=w1-gpio,gpiopin=4`
- Reboot Pi

### Sensors show -127°C or 185°F

**Bad connection** — check:
- All solder joints secure
- Wires not broken
- Sensor probe is fully waterproof (no water ingress)

### ESPHome won't connect to HA

**Check:**
- Pi and HA on same network
- Port 6053 not blocked by firewall
- API encryption key matches in both configs

### MQTT not working

**Check:**
- Mosquitto broker is running
- MQTT credentials match in `secrets.yaml` and HA
- Pi can reach HA: `ping 192.168.1.50`

### One sensor always reads higher

**This is normal!** Sensors have ±0.5°C accuracy. Use calibration to correct.

---

## Advanced: Auto-Start ESPHome on Boot

Make ESPHome run automatically when Pi boots:

```bash
# Create systemd service
sudo nano /etc/systemd/system/esphome-fish-tank.service
```

**Paste:**
```ini
[Unit]
Description=ESPHome Fish Tank Temperature Monitor
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/esphome
ExecStart=/home/pi/.local/bin/esphome run fish_tank_temps.yaml
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Enable and start:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable esphome-fish-tank.service
sudo systemctl start esphome-fish-tank.service

# Check status
sudo systemctl status esphome-fish-tank
```

Now ESPHome runs on boot automatically!

---

## Maintenance

### Monthly
- Check sensor readings against reference thermometer
- Verify Pi is online (check dashboard)
- Clean any algae off probes

### Every 6 Months
- Re-calibrate all sensors
- Check cable connections for corrosion
- Update ESPHome: `pip3 install -U esphome`

### Yearly
- Replace thermal paste on sensor probes (if readings drift)
- Consider replacing sensors (typical lifespan: 2-3 years in aquarium)

---

## Cost Summary

| Item | Cost |
|------|------|
| 3× DS18B20 Sensors | $18-24 |
| 4.7kΩ Resistor | $0.10 |
| Jumper Wires | $3 |
| **Total** | **~$25** |

**Plus Raspberry Pi 3B+ you already own!**

---

## What You've Gained

✅ **Professional monitoring** — Lab-grade multi-zone temperature tracking  
✅ **Early failure detection** — Heater stuck on/off, circulation issues  
✅ **ATO safety** — Prevents cold water shock  
✅ **Data logging** — Full temperature history for diagnostics  
✅ **Calibration control** — Precision ±0.1°F accuracy  
✅ **Expandable** — Can add more sensors (up to ~10 on one bus)  

Your fish tank now has better temperature monitoring than most research aquariums!

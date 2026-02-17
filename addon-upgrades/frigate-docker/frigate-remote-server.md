# 📹 Frigate on Remote Server — Setup Guide

This guide covers setting up Frigate NVR on a **separate server** (not as a Home Assistant add-on) and integrating it with your HA fish tank monitor.

---

## Why Run Frigate on a Separate Server?

✅ **Better performance** — Offloads AI processing from HA  
✅ **Dedicated hardware** — Use a machine with GPU/Coral TPU  
✅ **More storage** — Dedicated NVR drives for recordings  
✅ **Scalability** — Add more cameras without impacting HA  
✅ **Coral TPU support** — Easier to pass through USB on bare metal  

---

## Hardware Options

### Option A: Dedicated Mini PC
- Intel NUC or similar (i3/i5 with Intel QuickSync)
- 8GB RAM minimum
- 256GB SSD (OS) + 1-2TB HDD (recordings)
- Optional: Google Coral TPU USB accelerator
- **Cost**: $200-400

### Option B: Existing Linux Server
- Ubuntu 22.04 / Debian 12
- Docker installed
- Network accessible to Home Assistant
- **Cost**: Free if you have spare hardware

### Option C: Raspberry Pi 4 (8GB)
- Works, but limited performance
- Must use Coral TPU for acceptable speed
- 64-bit OS required
- **Cost**: $75 + $60 Coral

---

## Architecture Overview

```
┌─────────────────┐         ┌─────────────────┐
│  FRIGATE SERVER │         │ HOME ASSISTANT  │
│  192.168.1.75   │◄───────►│  192.168.1.50   │
│                 │  MQTT   │                 │
│  - Frigate NVR  │  5000   │  - Dashboard    │
│  - Recordings   │  1883   │  - Automations  │
│  - AI Detection │         │  - Alerts       │
└────────┬────────┘         └─────────────────┘
         │
         │ RTSP
         ▼
   ┌──────────┐
   │  CAMERA  │
   │192.168.1.│
   │   100    │
   └──────────┘
```

**Data Flow:**
1. Camera streams RTSP → Frigate server
2. Frigate processes video, runs AI detection
3. Frigate sends events → HA via MQTT
4. HA displays camera feed from Frigate HTTP server (port 5000)
5. HA triggers automations based on Frigate events

---

## Installation Steps

### Step 1 — Install Docker on Frigate Server

**Ubuntu/Debian:**
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group
sudo usermod -aG docker $USER

# Log out and back in, then verify
docker --version
```

---

### Step 2 — Create Frigate Directory Structure

```bash
# Create base directory
mkdir -p ~/frigate
cd ~/frigate

# Create subdirectories
mkdir -p config
mkdir -p media/frigate
mkdir -p go2rtc

# Set permissions
sudo chown -R $USER:$USER ~/frigate
```

Your structure should be:
```
~/frigate/
├── config/          ← Frigate config goes here
├── media/
│   └── frigate/     ← Recordings stored here
└── go2rtc/
```

---

### Step 3 — Copy Frigate Config

1. Download `frigate.yml` from the fish tank package
2. **Edit these lines:**
   - Line 21: `host: 192.168.1.50` ← Your HA IP
   - Line 24: `user: mqtt_user` ← Your MQTT username
   - Line 25: `password: mqtt_pass` ← Your MQTT password
   - Line 80: `path: rtsp://...` ← Your camera RTSP URL

3. **Copy to server:**
   ```bash
   scp frigate.yml user@192.168.1.75:~/frigate/config/config.yml
   ```
   Or use SFTP/WinSCP to copy the file

4. **Verify:**
   ```bash
   cat ~/frigate/config/config.yml
   ```

---

### Step 4 — Create Docker Compose File

Create `~/frigate/docker-compose.yml`:

```yaml
version: "3.9"
services:
  frigate:
    container_name: frigate
    privileged: true
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:stable
    shm_size: "256mb"
    
    # Mount coral TPU (if you have one)
    devices:
      - /dev/bus/usb:/dev/bus/usb  # Coral TPU
      # - /dev/dri/renderD128       # Intel QuickSync (if using)
    
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./config:/config
      - ./media/frigate:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000  # 1GB tmpfs
    
    ports:
      - "5000:5000"   # Web UI
      - "8554:8554"   # RTSP feeds
      - "8555:8555"   # WebRTC
      - "1935:1935"   # RTMP
    
    environment:
      FRIGATE_RTSP_PASSWORD: "password"
      TZ: "America/New_York"  # ← Change to your timezone
```

**For Raspberry Pi, use this image instead:**
```yaml
    image: ghcr.io/blakeblackshear/frigate:stable-rpi
```

---

### Step 5 — Start Frigate

```bash
cd ~/frigate
docker compose up -d
```

**Check logs:**
```bash
docker logs -f frigate
```

You should see:
```
[INFO] Starting go2rtc...
[INFO] Starting Frigate...
[INFO] Camera fish_tank: ffmpeg sent a broken frame
[INFO] Capture process started for fish_tank
```

---

### Step 6 — Verify Frigate Web UI

1. Open browser: `http://FRIGATE_SERVER_IP:5000`
2. You should see the Frigate interface
3. Click **fish_tank** camera
4. Verify live feed shows your tank

**If no video:**
- Check `docker logs frigate` for errors
- Verify RTSP URL is correct in config.yml
- Test RTSP stream: `ffplay rtsp://admin:pass@camera_ip/stream`

---

### Step 7 — Configure MQTT in Home Assistant

**If you don't have MQTT broker yet:**

1. **Supervisor → Add-on Store → Mosquitto broker**
2. **Install** and **Start**
3. **Configuration** tab → Enable:
   ```yaml
   logins:
     - username: mqtt_user
       password: mqtt_pass
   ```
4. **Save** and **Restart** the add-on

5. **Settings → Integrations → Add MQTT**
   - Accept defaults (auto-discovers local broker)

**Get your HA IP:**
```
Settings → System → Network → look for IPv4 address
Example: 192.168.1.50
```

This is the IP you put in frigate.yml line 21.

---

### Step 8 — Add Frigate Integration to HA

1. **Settings → Integrations → Add Integration**
2. Search: **Frigate**
3. **URL**: `http://192.168.1.75:5000`
   - Replace `192.168.1.75` with your Frigate server IP
4. **Submit**

**New entities appear:**
```
camera.fish_tank
camera.fish_tank_detect
sensor.fish_tank_bird_count
sensor.fish_tank_person_count
binary_sensor.fish_tank_motion
switch.fish_tank_detect
switch.fish_tank_recordings
```

---

### Step 9 — Install Fish Tank Camera Package

1. Copy `fish_tank_camera.yaml` → `config/packages/`
2. **Restart Home Assistant**

**New entities appear:**
```
sensor.fish_tank_fish_visibility
sensor.fish_tank_activity_score
binary_sensor.fish_tank_feeding_event
sensor.fish_tank_camera_status
```

---

### Step 10 — Add Camera Cards to Dashboard

1. **Install Frigate Card** via HACS:
   - HACS → Frontend → Search "Frigate Card"
   - Install `frigate-hass-card`

2. **Edit fish tank dashboard**:
   - ⋮ → Edit Dashboard → ⋮ → Raw Config Editor
   - Paste contents of `CAMERA_DASHBOARD_CARDS.yaml` at end of cards list
   - Save

---

## Network Configuration

### Firewall Rules (if applicable)

**On Frigate server**, allow these ports from HA:
```bash
sudo ufw allow from 192.168.1.50 to any port 5000   # Web UI
sudo ufw allow from 192.168.1.50 to any port 8554   # RTSP
```

**On Home Assistant**, allow MQTT from Frigate server:
```bash
# If using HA OS, no firewall config needed
# If running HA Core/Supervised on Linux:
sudo ufw allow from 192.168.1.75 to any port 1883
```

### Port Summary

| Port | Service | Direction |
|------|---------|-----------|
| 5000 | Frigate Web UI | HA → Frigate |
| 8554 | RTSP restream | HA → Frigate |
| 1883 | MQTT | Frigate → HA |
| 554  | Camera RTSP | Frigate → Camera |

---

## Storage Management

### Monitor Disk Usage

```bash
# Check recordings size
du -sh ~/frigate/media/frigate/

# Check available space
df -h ~/frigate/media/
```

### Adjust Retention

Edit `~/frigate/config/config.yml`:

```yaml
record:
  retain:
    days: 7    # Reduce to 3 or 5 to save space
```

Then restart:
```bash
docker restart frigate
```

### Storage Recommendations

| Cameras | Resolution | Days Retained | Storage Needed |
|---------|------------|---------------|----------------|
| 1 | 1080p | 7 days | 100 GB |
| 1 | 1080p | 14 days | 200 GB |
| 1 | 2K | 7 days | 150 GB |
| 1 | 4K | 7 days | 300 GB |

---

## Performance Tuning

### Using Coral TPU

**Verify Coral is detected:**
```bash
lsusb | grep Google
```

Should show: `Bus 001 Device 004: ID 1a6e:089a Global Unichip Corp.`

**Update docker-compose.yml:**
```yaml
detectors:
  coral:
    type: edgetpu
    device: usb
```

**Restart:**
```bash
docker restart frigate
```

**Verify in logs:**
```
[INFO] Coral TPU detected
```

### Using Intel QuickSync (Hardware Acceleration)

**Edit config.yml:**
```yaml
ffmpeg:
  hwaccel_args: preset-vaapi
```

**Edit docker-compose.yml:**
```yaml
devices:
  - /dev/dri/renderD128
```

**Restart:**
```bash
docker restart frigate
```

---

## Troubleshooting

### Camera entities show "unavailable" in HA

**Check:**
1. Frigate is running: `docker ps` (should show frigate container)
2. HA can reach Frigate: `ping 192.168.1.75` from HA Terminal
3. Integration URL is correct: Settings → Integrations → Frigate
4. MQTT is connected: Check Frigate logs for "MQTT connected"

### High CPU usage

**Solutions:**
1. Lower detection FPS in config.yml: `fps: 5` → `3`
2. Use substream instead of mainstream
3. Enable hardware acceleration (QuickSync or Coral)
4. Reduce resolution

### MQTT not connecting

**Check:**
1. MQTT broker is running in HA
2. Frigate server can reach HA: `ping 192.168.1.50`
3. MQTT credentials match in both systems
4. Frigate logs: `docker logs frigate | grep MQTT`

### Recordings filling disk

**Solutions:**
1. Reduce retention: `days: 7` → `3`
2. Use continuous recording only during hours: 
   ```yaml
   record:
     enabled: true
     expire_interval: 60
     retain:
       days: 5
       mode: motion  # Only record when motion detected
   ```
3. Add larger drive and remount

---

## Maintenance

### Update Frigate

```bash
cd ~/frigate
docker compose pull
docker compose up -d
```

### View Logs

```bash
# Live logs
docker logs -f frigate

# Last 100 lines
docker logs --tail 100 frigate
```

### Restart Frigate

```bash
docker restart frigate
```

### Backup Config

```bash
# Backup
cp ~/frigate/config/config.yml ~/frigate/config/config.yml.backup

# Restore
cp ~/frigate/config/config.yml.backup ~/frigate/config/config.yml
docker restart frigate
```

---

## Remote Access (Optional)

### Via Cloudflare Tunnel (Recommended)

1. Install cloudflared on Frigate server
2. Create tunnel pointing to `localhost:5000`
3. Access from anywhere via: `https://frigate.yourname.com`

### Via VPN (Secure)

1. Set up WireGuard/Tailscale
2. Connect to VPN
3. Access via internal IP: `http://192.168.1.75:5000`

⚠️ **Do NOT expose Frigate directly to internet** — no built-in authentication!

---

## Further Reading

- Frigate docs: https://docs.frigate.video
- Docker install: https://docs.docker.com/engine/install/
- Coral TPU setup: https://coral.ai/docs/accelerator/get-started/
- MQTT debugging: https://www.home-assistant.io/integrations/mqtt/

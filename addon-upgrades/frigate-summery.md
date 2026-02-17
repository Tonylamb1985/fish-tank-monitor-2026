# 📹 Frigate Camera Integration — Session Summary

**Topic:** AI-powered fish tank camera monitoring via Frigate NVR  
**Two setups covered:** HAOS Add-on AND separate Docker server  
**Date:** February 2026

---

## 📋 What Was Built

An AI-powered 24/7 camera monitoring system for the fish tank using Frigate NVR, providing fish counting, activity scoring, feeding detection, auto-snapshots, and timelapse generation — integrated fully into the fish tank dashboard.

---

## 📦 Files Created

| File | Purpose |
|------|---------|
| `frigate.yml` | Frigate NVR configuration (cameras, detection, recording, zones) |
| `fish_tank_camera.yaml` | HA package (template sensors, automations, scripts) |
| `CAMERA_DASHBOARD_CARDS.yaml` | Dashboard cards for live view, stats, history, events |
| `CAMERA_SETUP_GUIDE.md` | HAOS add-on installation guide |
| `FRIGATE_REMOTE_SERVER.md` | Separate Docker server installation guide |
| `FIX_NOTIFICATIONS.md` | Guide for replacing `notify.mobile_app_your_phone` |

---

## ⚙️ `frigate.yml` — Key Configuration

### MQTT (Bridges Frigate → Home Assistant)
```yaml
mqtt:
  enabled: true
  host: 192.168.1.50      # ← HA IP (remote setup)
  port: 1883
  topic_prefix: frigate
  client_id: frigate_fish_tank
  user: mqtt_user
  password: mqtt_pass
```
- **Add-on setup:** host = `core-mosquitto` (internal HA broker hostname)
- **Remote setup:** host = your Home Assistant's LAN IP

### Camera Stream
```yaml
cameras:
  fish_tank:
    ffmpeg:
      inputs:
        - path: rtsp://admin:password@192.168.1.100:554/stream1
          roles: [detect, record, rtmp]
    detect:
      width: 1920
      height: 1080
      fps: 5
```
Common RTSP formats documented for Reolink, Amcrest, Hikvision, generic ONVIF.

### Object Detection
```yaml
objects:
  track:
    - bird     # Proxy for fish (no fish-specific model in default YOLO)
    - person   # Detects feeding events
  filters:
    bird:
      min_area: 500
      threshold: 0.6
```
**Note:** Uses `bird` detection as a proxy for fish — the default YOLOv8 model has no fish class. Works for activity/counting purposes.

### Detection Zones
```yaml
zones:
  surface:         # Top 25% of tank — detects feeding
    coordinates: 0.1,0.05, 0.9,0.05, 0.9,0.25, 0.1,0.25
  bottom:          # Bottom 25% — detects hiding/sick fish
    coordinates: 0.1,0.75, 0.9,0.75, 0.9,0.95, 0.1,0.95
```

### Recording
```yaml
record:
  retain:
    days: 7
    mode: all       # 24/7 recording
  events:
    retain:
      default: 30   # Event clips kept 30 days
```

### Detector
```yaml
detectors:
  cpu1:
    type: cpu
    num_threads: 3
  # coral:          # Uncomment if you have Coral TPU
  #   type: edgetpu
  #   device: usb
```

---

## 📦 `fish_tank_camera.yaml` — HA Package

### Template Sensors
| Sensor | Description |
|--------|-------------|
| `fish_tank_feeding_event` | Binary — fish at surface + person detected |
| `fish_tank_fish_active` | Binary — motion + fish visible |
| `fish_tank_fish_visibility` | % of total fish currently visible |
| `fish_tank_activity_score` | 0–100 composite activity level |
| `fish_tank_camera_status` | online / offline / detection_disabled |
| `fish_tank_last_motion` | Timestamp of last detected motion |

### Fish Visibility Calculation
```yaml
visible_count / total_fish * 100
# total_fish hardcoded to 4 — change to your actual count
```

### Activity Score Calculation
```yaml
(fish_count * 20) + (motion * 30) + (surface_activity * 10)
# Capped at 100
```

### Automations (6)
| Automation | Trigger | Condition |
|-----------|---------|-----------|
| Camera offline | Unavailable 5 min | Alerts enabled |
| All fish hidden | Visibility <10% for 2hr | Daytime only (08:00–22:00) |
| Low activity | Score <20 for 3hr | Daytime, lights on |
| Auto-log feeding | Feeding event on for 30s | >3hr since last feed |
| Snapshot on alert | Ammonia/nitrite/nitrate = critical | Always |
| Pi offline | Status off for 5 min | Alerts enabled |

### Scripts (2)
- `fish_tank_capture_snapshot` — Saves to `/config/www/tank_snapshots/`
- `fish_tank_timelapse_today` — Generates MP4 via ffmpeg from day's recordings

### Required Directories
```bash
mkdir -p /config/www/tank_snapshots
mkdir -p /config/www/timelapses
```

---

## 🖥️ Setup Option A — HAOS Add-on

### Installation Steps
1. **Supervisor → Add-on Store** → search **Frigate NVR** → Install
2. Edit `frigate.yml` — set camera RTSP URL
3. Copy `frigate.yml` to `config/frigate.yml`
4. Start add-on, check logs for `Capture process started`
5. Verify live feed at **Frigate add-on → Open Web UI**
6. **Settings → Integrations → Add Frigate**
   - URL: `http://localhost:5000`
7. Copy `fish_tank_camera.yaml` to `config/packages/`
8. Restart HA

### New HA Entities (add-on)
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

## 🐳 Setup Option B — Separate Docker Server

### Why Use a Separate Server
- Offloads AI processing from Home Assistant
- Easier Coral TPU USB passthrough
- Dedicated storage for recordings
- Better performance for multiple cameras
- No impact on HA stability

### Architecture
```
Camera (RTSP) ──────────────► Frigate Server (192.168.1.75)
                                      │
                              AI Detection + Recording
                                      │
                              MQTT (port 1883) ──► Home Assistant (192.168.1.50)
                              HTTP (port 5000) ◄── HA Integration
```

### Key Difference in `frigate.yml`
```yaml
# Add-on:
mqtt:
  host: core-mosquitto    # internal HA hostname

# Separate server:
mqtt:
  host: 192.168.1.50      # HA's LAN IP address
```

### Docker Compose (`~/frigate/docker-compose.yml`)
```yaml
version: "3.9"
services:
  frigate:
    container_name: frigate
    privileged: true
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:stable
    shm_size: "256mb"
    devices:
      - /dev/bus/usb:/dev/bus/usb    # Coral TPU
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./config:/config
      - ./media/frigate:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "5000:5000"    # Web UI
      - "8554:8554"    # RTSP restream
      - "8555:8555"    # WebRTC
      - "1935:1935"    # RTMP
    environment:
      TZ: "Europe/London"
```

**Raspberry Pi image:**
```yaml
image: ghcr.io/blakeblackshear/frigate:stable-rpi
```

### Directory Structure on Server
```
~/frigate/
├── config/
│   └── config.yml        ← frigate.yml renamed
├── media/
│   └── frigate/          ← Recordings stored here
└── docker-compose.yml
```

### Server Installation Steps
1. Install Docker: `curl -fsSL https://get.docker.com | sh`
2. Create directory structure
3. Copy and edit `frigate.yml` (rename to `config.yml`)
4. Create `docker-compose.yml`
5. `docker compose up -d`
6. Check logs: `docker logs -f frigate`
7. Verify Web UI at `http://192.168.1.75:5000`
8. **In HA:** Settings → Integrations → Add Frigate
   - URL: `http://192.168.1.75:5000` ← server IP, not localhost

### Firewall Rules (if needed)
```bash
# On Frigate server — allow HA to reach it
sudo ufw allow from 192.168.1.50 to any port 5000
sudo ufw allow from 192.168.1.50 to any port 8554

# On HA server — allow Frigate MQTT
sudo ufw allow from 192.168.1.75 to any port 1883
```

### Port Reference
| Port | Service | Direction |
|------|---------|-----------|
| 5000 | Frigate Web UI | HA → Frigate |
| 8554 | RTSP restream | HA → Frigate |
| 1883 | MQTT events | Frigate → HA |
| 554 | Camera RTSP | Frigate → Camera |

---

## 📊 Dashboard Cards Added

### Overview View
- **Live camera feed** — `custom:frigate-card` with detection overlay
- **Camera stats card** — Fish count, visibility %, activity score, feeding event, motion, detection/recording toggles, snapshot button, timelapse button
- **24h activity graph** — Activity score + fish count history
- **Recent events gallery** — Clips of detected motion/objects

### Required HACS Card
```
HACS → Frontend → search "Frigate Card"
Repository: frigate-hass-card (by dermotduffy)
```

**Fallback if not using frigate-hass-card:**
```yaml
- type: picture-entity
  entity: camera.fish_tank
  camera_view: live
```

---

## 🚨 Alerts & What They Mean

| Alert | Trigger | Action |
|-------|---------|--------|
| Camera offline | No frames for 5 min | Check Frigate add-on / Docker container |
| All fish hidden | <10% visible for 2hr (daytime) | Check fish health and water quality |
| Low activity | Score <20 for 3hr (daytime, lights on) | Check water params, inspect fish |
| Feeding auto-logged | Person + surface activity for 30s | Informational — feeding timestamp updated |
| Snapshot captured | Water quality parameter hits critical | Photo saved for visual diagnosis |

---

## 🤖 AI Detection Notes

### Current Limitation
Default YOLOv8 model has no `fish` class — uses `bird` as a proxy. Works adequately for:
- Motion/activity detection ✅
- Approximate fish counting ✅
- Person detection for feeding events ✅

### Custom Fish Model (Advanced)
For true fish detection:
1. Collect 500+ labelled fish images (Roboflow)
2. Train YOLOv8 model (Google Colab — free)
3. Export to TFLite format
4. Place at `/config/custom_models/fish.tflite`
5. Update `frigate.yml` model section
6. Restart Frigate

### Hardware Acceleration Options
| Hardware | Config | CPU Savings |
|----------|--------|-------------|
| Google Coral TPU ($60) | `type: edgetpu` | ~90% reduction |
| Intel QuickSync | `hwaccel_args: preset-vaapi` | ~60% reduction |
| None | `type: cpu` | baseline |

---

## 💾 Storage Requirements

| Retention | Resolution | Storage |
|-----------|------------|---------|
| 7 days | 1080p | ~100 GB |
| 14 days | 1080p | ~200 GB |
| 7 days | 4K | ~300 GB |

Snapshots (30 days): +5 GB  
Event clips (30 days): +10 GB

---

## 🔧 Maintenance Commands (Docker setup)

```bash
# Update Frigate
cd ~/frigate && docker compose pull && docker compose up -d

# View live logs
docker logs -f frigate

# Restart
docker restart frigate

# Check disk usage
du -sh ~/frigate/media/frigate/

# Backup config
cp ~/frigate/config/config.yml ~/frigate/config/config.yml.backup
```

---

## ⚠️ Security Note

Do **not** expose Frigate directly to the internet — it has no built-in authentication. Use Cloudflare Tunnel or VPN for remote access.

---

## 🐛 Common Issues & Fixes

| Problem | Cause | Fix |
|---------|-------|-----|
| No video in HA | Wrong RTSP URL | Test with `ffplay rtsp://...` |
| `camera.fish_tank` unavailable | Integration URL wrong | Check add-on vs server IP |
| No fish detected | Low light / low confidence | Lower `threshold: 0.6` → `0.4` |
| High CPU | Too many FPS | Lower `fps: 5` → `3` |
| False positives | Bubbles/filter movement | Add `motion: mask:` polygon |
| MQTT not connecting | Wrong broker host | Check `host:` in frigate.yml |
| Recordings filling disk | Retention too long | Reduce `days: 7` → `3` |

---

## ✅ Final File Status

| File | Valid | Notes |
|------|-------|-------|
| `frigate.yml` | ✅ | Updated for remote server (MQTT host = HA IP) |
| `fish_tank_camera.yaml` | ✅ | All notify replaced with persistent_notification |
| `CAMERA_DASHBOARD_CARDS.yaml` | ✅ | Aquarium theme styling applied |
| `CAMERA_SETUP_GUIDE.md` | ✅ | Both setup options documented |
| `FRIGATE_REMOTE_SERVER.md` | ✅ | Full Docker install guide |

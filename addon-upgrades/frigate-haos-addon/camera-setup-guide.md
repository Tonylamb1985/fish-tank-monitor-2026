# 📹 Fish Tank Camera Monitor — Installation Guide

Complete guide to setting up AI-powered fish tank camera monitoring with Frigate NVR.

**Choose your installation:**
- **Option A**: Frigate as Home Assistant add-on (this guide)
- **Option B**: Frigate on separate server → See `FRIGATE_REMOTE_SERVER.md`

---

## What This Does

✅ **24/7 Recording** — Continuous recording of your tank with motion detection  
✅ **AI Fish Detection** — Tracks fish movement, counts visible fish  
✅ **Activity Monitoring** — Detects active swimming vs hiding/sick behavior  
✅ **Feeding Events** — Auto-detects when you feed (person + surface activity)  
✅ **Anomaly Alerts** — Low activity, all fish hidden, camera offline  
✅ **Snapshots** — Auto-capture on water quality alerts  
✅ **Timelapse** — Generate daily time-lapse videos  

---

## Hardware Requirements

### Camera Options

**Option A: IP Camera (Recommended)**
- Any RTSP camera (Reolink, Amcrest, Hikvision, Dahua)
- 1080p resolution minimum
- Good low-light performance (fish tanks are often dim)
- Wide angle lens (to capture entire tank)
- **Cost**: $30-80

**Option B: USB Webcam**
- Logitech C920 or similar 1080p webcam
- Requires go2rtc setup (included in Frigate)
- **Cost**: $40-70

**Option C: Old Android/iPhone**
- Use "IP Webcam" app (Android) or "AtHome Camera" (iOS)
- Free, but less reliable than dedicated camera

### Optional: Coral TPU

- Google Coral USB Accelerator ($60)
- **Dramatically** speeds up AI detection (10x faster)
- Not required, but highly recommended for smooth operation

### Recommended Camera Placement

```
     [ROOM]
        │
    ┌───┴───┐
    │CAMERA │  ← Angled slightly down
    └───────┘
        ↓
    ┌───────────────┐
    │               │
    │   FISH TANK   │  ← Full front view
    │               │
    └───────────────┘
```

- **Position**: Front-center of tank, at mid-height
- **Angle**: Slight downward tilt (reduces glass reflections)
- **Distance**: 3-5 feet from tank
- **Lighting**: Tank LED lights ON for detection (camera needs light)
- **Avoid**: Bright room lights behind camera (creates glare)

---

## Installation Steps

### Step 1 — Install Frigate Add-on

1. **Home Assistant → Supervisor → Add-on Store**
2. Search: **Frigate NVR**
3. Click **Install** (this takes 5-10 minutes)
4. **Do NOT start yet** — we need to configure first

---

### Step 2 — Configure Camera Connection

1. **Find your camera's RTSP stream URL**

   Common formats:
   ```
   Reolink:    rtsp://admin:password@192.168.1.100:554/h264Preview_01_main
   Amcrest:    rtsp://admin:password@192.168.1.100:554/cam/realmonitor?channel=1&subtype=0
   Hikvision:  rtsp://admin:password@192.168.1.100:554/Streaming/Channels/101
   Wyze:       rtsp://username:password@192.168.1.100:8554/unicast
   ```

   **To find yours:**
   - Check camera manual or manufacturer website
   - Or use ONVIF Device Manager (free tool)
   - Or try generic: `rtsp://admin:password@CAMERA_IP:554/stream1`

2. **Open** `frigate.yml` (provided in files)
3. **Find** line 80: `path: rtsp://admin:password@192.168.1.100:554/stream1`
4. **Replace** with your camera's RTSP URL
5. **Save**

---

### Step 3 — Upload Frigate Config

1. Copy `frigate.yml` to: **`config/frigate.yml`**
   - Use File Editor add-on, or
   - Use Samba share to access config folder, or
   - Use SSH/Terminal to copy file

2. Verify location:
   ```
   config/
   ├── configuration.yaml
   ├── frigate.yml          ← must be here
   └── packages/
   ```

---

### Step 4 — Start Frigate

1. **Supervisor → Frigate NVR → Start**
2. Wait 30 seconds
3. Check logs: **Log tab** — should see "Camera fish_tank: ffmpeg sent a broken frame"
   - This is normal during startup
   - After ~1 min you should see: "Capture process started"

---

### Step 5 — Verify Camera Feed

1. **Frigate NVR → Open Web UI** (button at top of add-on page)
2. You should see your tank live!
3. Click **fish_tank** camera
4. Check:
   - ✅ Live feed shows your tank
   - ✅ Green detection boxes appear when fish move
   - ✅ FPS shows ~5 (in top left)

**Troubleshooting if no video:**
- Check RTSP URL is correct
- Verify camera IP is reachable: `ping 192.168.1.100`
- Check camera username/password
- Try main stream vs substream (see camera docs)

---

### Step 6 — Add Frigate Integration to HA

**For Home Assistant Add-on:**
1. **Settings → Integrations → Add Integration**
2. Search: **Frigate**
3. URL: `http://localhost:5000` (accept default)
4. Submit

**For Remote Server (on separate machine):**
1. **Settings → Integrations → Add Integration**
2. Search: **Frigate**
3. URL: `http://FRIGATE_SERVER_IP:5000`
   - Example: `http://192.168.1.75:5000`
   - Replace with your actual Frigate server IP
4. Submit

**New entities appear:**
```
camera.fish_tank
camera.fish_tank_detect         (with AI overlay)
sensor.fish_tank_bird_count     (fish count)
binary_sensor.fish_tank_motion
switch.fish_tank_detect
switch.fish_tank_recordings
```

---

### Step 7 — Install Camera Package

1. Copy `fish_tank_camera.yaml` → **`config/packages/`**
2. **Restart Home Assistant**

**New entities appear:**
```
sensor.fish_tank_fish_visibility        (% of fish visible)
sensor.fish_tank_activity_score         (0-100 activity level)
binary_sensor.fish_tank_feeding_event   (feeding detected)
sensor.fish_tank_camera_status          (online/offline)
```

**New automations:**
- Camera offline alert
- All fish hidden alert (2hr+ no visibility)
- Low activity alert (3hr+ low movement)
- Auto-log feeding events
- Snapshot on water quality alerts

---

### Step 8 — Create Snapshot Directories

Run in Terminal add-on or SSH:

```bash
mkdir -p /config/www/tank_snapshots
mkdir -p /config/www/timelapses
```

These folders store captured snapshots and timelapse videos.

---

### Step 9 — Add Camera Cards to Dashboard

1. Open your fish tank dashboard
2. **⋮ → Edit Dashboard → ⋮ → Raw Configuration Editor**
3. Scroll to the **Overview view**, find the **cards:** section
4. **Paste the contents** of `CAMERA_DASHBOARD_CARDS.yaml` at the end
5. **Save**

**Required HACS Card:**
- HACS → Frontend → search **"Frigate Card"**
- Install `frigate-hass-card`
- Refresh browser

---

### Step 10 — Customize for Your Tank

Edit `fish_tank_camera.yaml`:

1. **Fish count** (line 98):
   ```yaml
   total_fish: 4    ← change to your actual fish count
   ```

2. **Notification service** (multiple lines):
   ```yaml
   notify.mobile_app_your_phone    ← replace with your notifier
   ```

3. **Detection zones** (optional):
   Adjust coordinates in `frigate.yml` lines 156-186 to match your tank layout

---

## Using the Camera Monitor

### Live View
- Dashboard shows live camera feed with AI detection boxes
- Green boxes = detected fish
- Blue boxes = detected person (you, during feeding)

### Fish Counting
- `sensor.fish_tank_bird_count` shows currently visible fish
- Uses "bird" detection as proxy for fish (close enough for the AI)
- Updates every ~5 seconds

### Activity Monitoring
- `sensor.fish_tank_activity_score` (0-100)
  - 0-20: Very low (sick/hiding)
  - 20-50: Low (calm)
  - 50-80: Normal
  - 80-100: High (feeding/active)

### Feeding Detection
- Auto-detects when you approach tank and fish come to surface
- Auto-logs feeding time
- Prevents duplicate logs (3hr cooldown)

### Snapshots
- Manually: Press "📸 Capture Snapshot" button on dashboard
- Auto: Captured on ammonia/nitrite/nitrate critical alerts
- Saved to: `/config/www/tank_snapshots/`
- View at: `http://homeassistant.local:8123/local/tank_snapshots/`

### Timelapse Videos
- Press "🎬 Generate Today's Timelapse" button
- Creates 30fps video from today's recordings (sped up 20x)
- Saved to: `/config/www/timelapses/`
- View at: `http://homeassistant.local:8123/local/timelapses/`

---

## Alerts You'll Receive

📹 **Camera Offline** — Camera hasn't sent frames for 5 minutes  
⚠️ **All Fish Hidden** — 0 fish visible for 2+ hours (daytime only)  
⚠️ **Low Activity** — Activity score < 20 for 3+ hours (daytime, lights on)  
🍽 **Feeding Detected** — Person + surface activity detected  
📸 **Snapshot Captured** — Auto-snapshot on water quality alert  

---

## Troubleshooting

### "No fish detected" but fish are clearly visible

**Cause**: AI model doesn't have fish-specific training (uses "bird" as proxy)

**Solutions**:
1. Increase detection area (frigate.yml line 149: `min_area: 500` → lower to `300`)
2. Lower confidence threshold (line 153: `threshold: 0.6` → `0.4`)
3. Ensure tank lights are on (AI needs light to see)
4. Add more lighting to tank

### High CPU usage / laggy video

**Solutions**:
1. Lower detection FPS (frigate.yml line 148: `fps: 5` → `3`)
2. Reduce resolution (switch from main stream to substream)
3. Buy a Coral TPU ($60) — reduces CPU by 90%

### Camera shows "unavailable"

**Check**:
1. RTSP URL is correct in frigate.yml
2. Camera is powered on and network-connected
3. Frigate add-on is running (Supervisor → Frigate → should be green)
4. Frigate logs (Frigate add-on → Log tab)

### Detection boxes jitter / false positives

**Cause**: Bubbles, filter movement, reflections

**Solution**: Add motion masks in frigate.yml
```yaml
motion:
  mask:
    # Mask bottom-right corner (air stone bubbles)
    - 0.8,0.8,1.0,0.8,1.0,1.0,0.8,1.0
```

### Recordings use too much disk space

**Solution**: Reduce retention in frigate.yml
```yaml
record:
  retain:
    days: 3    # was 7 — change to 3 days
```

---

## Advanced: Custom Fish Detection Model

The default model uses "bird" as a proxy for fish. For true fish-specific detection:

1. **Collect training data** — 500+ images of your fish
2. **Label fish** using LabelImg or Roboflow
3. **Train YOLOv8 model** using Google Colab (free)
4. **Export to TFLite** format
5. **Upload to** `/config/custom_models/fish.tflite`
6. **Update frigate.yml** model section
7. **Restart Frigate**

Guides available at: https://docs.frigate.video/guides/custom_models

---

## Storage Requirements

| Setting | Storage Used |
|---------|--------------|
| 1 camera, 1080p, 7 days retention | ~50-100 GB |
| + Snapshots (30 day retention) | +5 GB |
| + Event clips (30 day retention) | +10 GB |

**Total**: ~65-115 GB for 7 days of 24/7 recording

Use a dedicated SSD or USB drive for Frigate storage.

---

## Privacy & Security

⚠️ **Your camera feed stays local** — Frigate runs entirely on your HA instance  
⚠️ **No cloud uploads** — All recordings stored locally  
⚠️ **Secure your camera** — Change default passwords  
⚠️ **Network isolation** — Put camera on isolated VLAN if possible  

---

## Further Reading

- Frigate docs: https://docs.frigate.video
- Frigate community: https://github.com/blakeblackshear/frigate
- Camera compatibility: https://docs.frigate.video/cameras

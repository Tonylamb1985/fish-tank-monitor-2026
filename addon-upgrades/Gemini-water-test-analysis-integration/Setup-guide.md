# 🐟 Aquarium Water Test — Home Assistant Setup Guide

## Overview
AI-powered water quality analysis using Google Gemini. Upload a photo of your test results, get sensor values back in 10–15 seconds.

**Supports:**
- API 5-in-1 Strip — pH, KH, GH, Nitrite, Nitrate
- API Liquid Test Kit — adds Ammonia
- NT Labs Liquid Test Kit — full parameter set
- Seachem Ammonia Alert badge — reads colour from photo (Safe/Alert/Alarm/Toxic)

---

## File Locations
```
config/
├── configuration.yaml              ← add packages line here
└── packages/
    ├── aquarium_sensors.yaml       ← helpers + template sensors + Seachem badge sensor
    ├── aquarium_scripts.yaml       ← AI analysis scripts (all 3 test kits)
    └── aquarium_automations.yaml   ← 6 automations incl. Seachem alert
```
`aquarium_dashboard.yaml` → paste into your dashboard as a new view (YAML editor)

---

## Setup Steps

### Step 1 — Install HACS Frontend Integrations
Install all three from HACS → Frontend:
- **Browser Mod** (thomasloven/hass-browser_mod) — upload popup & confirmation dialogs
- **card-mod** (thomasloven/lovelace-card-mod) — colour-coded card borders
- **mini-graph-card** (kalkih/lovelace-mini-graph-card) — history graphs

After installing Browser Mod, open its panel in the sidebar and click **Register** on every device you will use the dashboard on.

### Step 2 — Get Your Free Gemini API Key
1. Go to https://aistudio.google.com
2. Click **Get API Key** → **Create API key in new project**
3. Copy the key (starts with AIzaSy...)
4. In HA: **Settings → Devices & Services → Add Integration → Google Generative AI Conversation**
5. Paste the key — leave all other settings as default

### Step 3 — Enable Packages
Add to `config/configuration.yaml` (merge with existing `homeassistant:` section if present):
```yaml
homeassistant:
  packages: !include_dir_named packages
```
Create the `config/packages/` folder if it doesn't exist.

### Step 4 — Copy Package Files
Place in `config/packages/`:
- `aquarium_sensors.yaml`
- `aquarium_scripts.yaml`
- `aquarium_automations.yaml`

### Step 5 — Create Media Folder
In HA: **Media → Create folder → name it `aquarium`**
Photos go to: `/media/aquarium/`

### Step 6 — Update Your Notify Service Name
In `aquarium_automations.yaml`, replace ALL occurrences of:
```
notify.mobile_app_your_phone
```
With your actual service name. Find it at: **Settings → Devices & Services → Companion App**
Example: `notify.mobile_app_johns_iphone`

There are **5 automations** that use the notify service — use Find & Replace.

### Step 7 — Add Dashboard Tab
1. Open your HA dashboard → Edit (pencil) → Add View
2. Switch to YAML editor
3. Paste the full contents of `aquarium_dashboard.yaml`
4. Save

### Step 8 — Restart Home Assistant

---

## How to Use (Phone Workflow)
1. Do your water test as normal (wait full development time)
2. **Photograph** results — include Seachem badge in same frame if you have one
3. In HA Companion App: **Media → Upload → aquarium folder**
4. On the **Aquarium** dashboard tab:
   - Select your **test kit** from the dropdown
   - Tap **📷 Set Image Path** → enter `/media/aquarium/your_filename.jpg` → Save
   - Tap **🔬 Analyse Test Now**
5. Wait 5–15 seconds — all sensors update automatically

---

## Seachem Ammonia Alert Badge

The badge lives permanently in your tank. The AI reads its colour from your photo:

| Badge Colour | Status | Free NH₃ | Action |
|---|---|---|---|
| 🟡 Yellow | Safe | 0 ppm | None |
| 🟢 Green | Alert | ~0.05 ppm | Monitor — 3–5 days tolerance |
| 🔵 Blue-green | Alarm | ~0.2 ppm | Act within 1–3 days |
| ⬜ Grey/Blue-grey | Toxic | ~0.5 ppm | Immediate water change |

**Notes:**
- Badge detects FREE ammonia (NH₃) only — more toxic than total ammonia
- If pH < 7.0, free ammonia cannot exist — badge will read Safe regardless
- Do not touch the circular sensor with fingers
- Badge lasts 1+ year — store in RO water when not in use

---

## Automations Included
| Automation | Trigger |
|---|---|
| DANGER Water Quality Alert | Water status → Danger |
| WARNING Water Quality Alert | Water status → Warning |
| Ammonia Detected Alert | Ammonia sensor → WARNING or DANGER |
| Seachem Badge Alert | Seachem badge → Alert, Alarm, or Toxic |
| Weekly Test Reminder | Every Sunday 10:00 AM |
| Overdue Test Reminder | Daily check — fires if no test in 10+ days |

---

## Photography Tips
**Test strips:** Flat on white paper, natural daylight, all pads visible, wait 30 seconds first
**Liquid tests:** Tubes in rack with colour chart, photograph from front, tube labels visible
**Seachem badge:** Include in same frame as tests, or photograph through tank glass

---

## Safe Ranges — Freshwater Tropical Fish
| Parameter | Safe | Warning | Danger |
|---|---|---|---|
| pH | 6.8–7.6 | 6.0–6.8 or 7.6–8.0 | <6.0 or >8.0 |
| Ammonia | 0 ppm | 0.01–0.25 ppm | ≥0.5 ppm |
| Nitrite | 0 ppm | 0.01–0.25 ppm | ≥0.5 ppm |
| Nitrate | <20 ppm | 20–40 ppm | ≥40 ppm |
| GH | 70–150 ppm | 30–70 or 150–300 | <30 or >300 |
| KH | 70–140 ppm | 40–70 or >140 | <40 ppm |

*API 5-in-1 strip: no ammonia test. Use API Liquid, NT Labs Liquid, or Seachem badge for ammonia.*

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Sensors show N/A | No test run yet, or JSON parse failed — check image path is correct |
| Analysis fails / spinner stuck | Verify image path exists, check Gemini integration is working |
| Wrong values | Retake photo with better lighting on white background |
| Seachem shows Not Present | Badge not visible in photo — include it in the frame |
| No notifications | Check notify service name in automations file |
| Browser Mod popup not opening | Register your browser in the Browser Mod sidebar panel |
| Coloured borders missing | Reinstall card-mod from HACS, clear browser cache |
| History graphs missing | Install mini-graph-card from HACS, graphs fill in over time |
| Cards say Entity not found | Packages not loaded — check HA logs for YAML errors after restart |

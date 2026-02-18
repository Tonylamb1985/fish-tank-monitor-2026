# 🤖 Gemini AI Integration — Complete Setup Guide

**Module:** Gemini AI Vision + Text Analysis  
**Platform:** Home Assistant + Extended Text Integration  
**API:** Google Gemini 2.0 Flash (free tier)

---

## 📋 What Gemini Does For Your Fish Tank

### 1. 🐟 Fish Health Analysis (Vision Model)
- Analyzes camera snapshots every 12 hours (configurable)
- Assesses fish coloration, fins, swimming behavior
- Detects visible signs of stress or disease
- Reports on tank cleanliness (algae, debris, glass clarity)
- Provides actionable recommendations

### 2. 📊 Daily Summary & Predictions
- Generates AI summary every evening (21:00)
- Analyzes all sensor data and trends
- Predicts when water changes will be needed
- Recommends maintenance actions for tomorrow
- Tracks parameter trends over time

### 3. 💬 Natural Language Queries
- Ask any question about your tank status
- Get answers based on real sensor data
- Examples: "Why are my fish hiding?", "Is my pH stable?", "When should I change water?"

### 4. 📋 Weekly Reports
- Comprehensive tank health report on demand
- Parameter trends, maintenance summary
- Fish behavior observations
- Equipment performance notes

---

## 🛒 Requirements

| Item | Status | Notes |
|------|--------|-------|
| Home Assistant | ✅ Already installed | Any version 2023+ |
| Extended Text integration | ⬇️ Install via HACS | Supports Gemini despite "OpenAI" name |
| Gemini API key | 🔑 Free from Google | No credit card needed |
| Frigate camera system | ✅ Already installed | For snapshot analysis |
| fish_tank_sensors.yaml | ✅ Already installed | For water quality data |
| fish_tank_camera.yaml | ✅ Already installed | For camera entities |

**Total cost: £0** — Everything uses free tier

---

## 📥 Installation Steps

### Step 1 — Install Extended Text Integration (HACS)

1. **HACS → Integrations**
2. Search: **"Extended OpenAI Conversation"**
3. Click **Install**
4. **Restart Home Assistant**

> Despite the name, this integration supports Google Gemini perfectly.

---

### Step 2 — Get Gemini API Key

1. Go to: **[https://aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey)**
2. Sign in with Google account
3. Click **"Create API Key"**
4. Select **"Create API key in new project"** (or use existing project)
5. **Copy the key** (starts with `AIza...`)

> Keep this key private — treat it like a password.

---

### Step 3 — Configure Extended Text

Open `configuration.yaml` and add:

```yaml
extended_openai_conversation:
  - name: Gemini Fish Tank
    api_key: AIzaSyD...YOUR_KEY_HERE...
    api_base: https://generativelanguage.googleapis.com/v1beta
    model: gemini-2.0-flash-exp
    vision_model: gemini-2.0-flash-exp
    max_tokens: 2048
```

**Important notes:**
- The `api_base` URL is specific to Gemini — do not use OpenAI's URL
- `model` and `vision_model` both use `gemini-2.0-flash-exp` (the latest free model)
- `max_tokens` limits response length to control API usage

---

### Step 4 — Install the Package

Copy `fish_tank_gemini.yaml` to `config/packages/fish_tank_gemini.yaml`

Confirm packages are enabled in `configuration.yaml`:
```yaml
homeassistant:
  packages: !include_dir_named packages
```

---

### Step 5 — Add Dashboard Cards

Open your fish tank dashboard in edit mode:

| Section | Where to paste |
|---------|---------------|
| Section 1 — Overview card | Existing Overview view |
| Section 2 — Full Gemini view | New view: `gemini` at `/lovelace/gemini` |
| Section 3 — Settings card | Existing Settings view |

---

### Step 6 — Restart Home Assistant

**Settings → System → Restart**

Wait for HA to come back online (~1 minute).

---

### Step 7 — Test the Integration

1. Go to **Gemini AI view** in the dashboard
2. Press **"Analyze Snapshot Now"** button
3. Wait ~5–10 seconds
4. Check the notification bell (top right)
5. You should see **"📸 Gemini Snapshot Analysis"** with fish health report

If you see an error:
- Check your API key is correct in `configuration.yaml`
- Ensure `api_base` URL is exactly as shown above
- Check logs: **Settings → System → Logs**

---

## ⚙️ Configuration

### Health Check Interval

Default: **12 hours**

Change via dashboard:
- **Settings → Gemini AI Settings → Health Check Interval**
- Range: 1–168 hours

Or via YAML:
```yaml
input_number.gemini_health_check_interval: 6  # Run every 6 hours
```

### Enable/Disable Features

| Feature | Helper | Default |
|---------|--------|---------|
| Auto fish health analysis | `input_boolean.gemini_fish_health_analysis_enabled` | On |
| Auto daily summary (21:00) | `input_boolean.gemini_daily_summary_enabled` | On |
| Auto water predictions | `input_boolean.gemini_water_predictions_enabled` | On |

Toggle these in **Settings → Gemini AI Settings** or via automations.

---

## 🔧 How Each Feature Works

### 🐟 Fish Health Analysis

**Trigger:** Every N hours (configurable) OR manual button press

**Process:**
1. Automation fires: `gemini_auto_fish_health_check`
2. Calls `script.fish_tank_capture_snapshot` to save fresh image
3. Sends image + current sensor data to Gemini vision API
4. Gemini analyzes: fish health, tank cleanliness, visible issues
5. Response stored in `input_text.gemini_last_health_analysis`
6. Notification sent with summary
7. Timestamp updated in `input_datetime.gemini_last_health_check`

**Prompt sent to Gemini:**
```
You are an expert aquarium consultant analyzing a fish tank camera image.

Current tank parameters:
[Full sensor summary]

Please analyze this tank snapshot and provide:
1. Fish health assessment (coloration, fins, swimming behavior)
2. Visible signs of stress or disease
3. Tank cleanliness (algae, debris, glass clarity)
4. Any recommendations for immediate action

Be concise but specific. Focus on actionable observations.
```

**Example response:**
```
Fish appear healthy with good coloration and active swimming patterns. 
No visible signs of disease or fin damage. Tank glass shows minor algae 
buildup on rear wall — recommend cleaning during next water change. 
Filter return flow appears strong. Overall: tank in good condition.
```

---

### 📊 Daily Summary & Predictions

**Trigger:** Daily at 21:00 (9 PM)

**Process:**
1. Automation fires: `gemini_daily_summary`
2. Collects all sensor data + today's activity metrics
3. Sends to Gemini text API (no vision)
4. Gemini generates: health summary, parameter trends, tomorrow's actions, water change prediction
5. Response stored in `input_text.gemini_last_daily_summary_text`
6. Notification sent
7. Timestamp updated

**Prompt includes:**
- All water parameters (temp, pH, ammonia, nitrite, nitrate, O2)
- Today's ATO top-off volume
- Camera activity score
- Fish visibility %
- Days since last water change/feeding

**Example response:**
```
Tank Status: Stable
Water parameters all within safe ranges. Nitrate trending upward 
(currently 28ppm) — water change recommended within 5-7 days. 
Temperature stable at 24.5°C. Fish activity high (score: 78), good 
sign of health. Today's evaporation: 180ml (normal). 

Tomorrow's Actions:
- Monitor nitrate
- Clean glass during evening (minor algae visible)
- Consider 25% water change this weekend

Water Change Prediction: 6 days (NO3 will approach 40ppm threshold)
```

---

### 🔮 Water Quality Predictions

**Trigger:** After every water test OR water change OR daily at noon

**Process:**
1. Automation fires: `gemini_water_quality_prediction`
2. Sends current parameters + historical context to Gemini
3. Gemini predicts: days until next water change, which parameter will trigger it, trend direction
4. Response stored in `input_text.gemini_water_prediction`

**Prompt includes:**
- Current ammonia, nitrite, nitrate, pH, O2 levels
- Safe thresholds for each parameter
- Tank age
- Time since last water change
- Configured water change interval

**Example response:**
```
Estimated Days Until Water Change: 7-8 days
Trigger Parameter: Nitrate (currently 28ppm, rising ~5ppm/week)
Current Trend: Gradually degrading (expected for mature tank)
Immediate Concerns: None

Nitrate is accumulating at a steady rate typical for an established 
tank with your bioload. Current trajectory suggests reaching the 40ppm 
threshold in approximately one week. pH remains stable. All other 
parameters excellent. No urgent action needed.
```

---

### 💬 Ask Gemini Custom Questions

**Trigger:** Manual via script call

**Usage:**
```yaml
service: script.gemini_ask_about_tank
data:
  question: "Why are my fish hiding?"
```

**Process:**
1. Gemini receives your question
2. Gemini gets full tank context (all sensors + latest health analysis)
3. Gemini provides specific answer based on actual data
4. Response sent as notification

**Example:**
```
Question: "Why are my fish hiding?"

Answer: Based on your tank data, several factors could explain this:

1. Recent water change (2 days ago) can temporarily stress fish
2. Temperature is slightly elevated at 26°C vs your usual 24.5°C — 
   check heater setting
3. Latest vision analysis noted strong filter flow — fish may be 
   seeking calmer areas
4. Lighting may be too bright — fish often hide when lights first 
   turn on

Recommendations:
- Verify heater thermostat (should be ~24.5°C)
- Observe if hiding decreases as day progresses
- Consider adding more plants/decor for cover
- No water parameter concerns detected
```

---

### 📋 Weekly Reports

**Trigger:** Manual via button or script call

**Process:**
1. Script fires: `gemini_generate_weekly_report`
2. Collects comprehensive weekly data:
   - Current status
   - Week's ATO activity (cycles, ml added, evaporation rate)
   - Maintenance history
   - Latest AI health analysis
3. Gemini generates structured report
4. Sent as notification (can be very long, check full content)

**Report structure:**
1. Executive summary (2-3 sentences)
2. Parameter trends over the week
3. Fish health and behavior observations
4. Maintenance recommendations for next week
5. Equipment performance notes
6. Any concerning patterns

---

## 💰 Gemini API Free Tier Limits

| Limit | Value | Fish Tank Usage |
|-------|-------|-----------------|
| Requests per minute | 15 | ~0.5/min average (well under limit) |
| Requests per day | 1,500 | ~20/day typical (1.3% of limit) |
| Tokens per day | 1,000,000 | ~50,000/day typical (5% of limit) |
| Vision requests | Included | Free with flash model |
| Credit card | Not required | True free tier |

**Typical daily usage:**
- 2× health analyses (12h interval) = 2 requests
- 1× daily summary = 1 request
- 2× water predictions = 2 requests
- ~5× manual queries = 5 requests
- **Total: ~10 requests/day = 0.7% of free tier**

You'd have to run 150 analyses per day to hit the limit.

---

## 📊 Data Sent to Gemini

### For Vision Analysis (Images)
- Latest camera snapshot (JPG, ~500KB)
- Current sensor readings (text)
- Tank age and maintenance history (text)

### For Text Analysis (No Images)
- All sensor values
- Historical timestamps
- Maintenance logs
- Camera-derived metrics (visibility %, activity score)

### Privacy Notes
- Images and data sent to Google's Gemini API
- Google's policy: data not used to train models
- Data retained temporarily for abuse prevention
- No personal identifying information sent
- If privacy is critical, disable vision analysis and use text-only predictions

---

## 🐛 Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| "API key invalid" | Wrong key or typo | Check key in configuration.yaml, ensure no spaces |
| "Model not found" | Wrong model name | Use exactly `gemini-2.0-flash-exp` |
| "Connection failed" | Wrong api_base URL | Use `https://generativelanguage.googleapis.com/v1beta` |
| No analysis running | Helper disabled | Check `input_boolean.gemini_fish_health_analysis_enabled` = on |
| Analysis says "failed" | Snapshot missing | Ensure Frigate camera is working, check `/config/www/tank_snapshots/latest.jpg` exists |
| Rate limit hit | Too many requests | Wait 1 minute, reduce health check frequency |
| Response truncated | Max tokens too low | Increase `max_tokens` in configuration.yaml |
| Notification not appearing | Notification service issue | Check HA notifications in bell icon top-right |

---

## 🔧 Advanced Customization

### Change Analysis Prompt

Edit `fish_tank_gemini.yaml`, find automation `gemini_auto_fish_health_check`, modify the `prompt:` section.

Example — focus on specific species:
```yaml
prompt: >
  You are an expert on Discus fish. Analyze this tank for Discus-specific 
  health indicators...
```

### Add More Sensors to Context

Edit `sensor.gemini_tank_state_summary`, add more attributes:
```yaml
attributes:
  gh: "{{ states('sensor.fish_tank_gh') }}"
  kh: "{{ states('sensor.fish_tank_kh') }}"
```

### Run Reports on Schedule

Add automation to call `script.gemini_generate_weekly_report` every Sunday:
```yaml
automation:
  - alias: "Weekly Gemini Report"
    trigger:
      - platform: time
        at: "09:00:00"
    condition:
      - condition: time
        weekday:
          - sun
    action:
      - service: script.gemini_generate_weekly_report
```

---

## 📁 File Summary

| File | Location | Purpose |
|------|----------|---------|
| `fish_tank_gemini.yaml` | `config/packages/` | HA package — sensors, automations, scripts |
| `GEMINI_DASHBOARD_CARDS.yaml` | Reference | Dashboard cards (3 sections) |
| `GEMINI_SETUP_GUIDE.md` | Reference | This guide |
| `configuration.yaml` | `config/` | Add Extended Text integration config here |

---

## ✅ Final Checklist

- [ ] Extended Text integration installed via HACS
- [ ] Gemini API key obtained from Google AI Studio
- [ ] `extended_openai_conversation:` block added to configuration.yaml
- [ ] `fish_tank_gemini.yaml` copied to packages folder
- [ ] Home Assistant restarted
- [ ] Dashboard cards added to Overview, Gemini, and Settings views
- [ ] "Analyze Snapshot Now" tested successfully
- [ ] Daily summary automation confirmed (check logs next day)
- [ ] Auto fish health analysis enabled and interval configured

---

## 🚀 Next Steps

Once everything is working:

1. **Monitor for a week** — let the daily summaries and health checks build a baseline
2. **Review predictions** — see how accurate Gemini's water change predictions are
3. **Ask questions** — use the custom query feature when you notice unusual behavior
4. **Generate weekly report** — every Sunday to see comprehensive trends
5. **Adjust intervals** — increase health check frequency if you notice issues often, decrease if tank is very stable

Gemini will learn your tank's patterns and provide increasingly relevant insights over time.

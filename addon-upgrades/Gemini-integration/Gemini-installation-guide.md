# 🤖 Gemini AI — Complete Installation Guide

**Integration Strategy:** Use BOTH integrations together for best results
- **Google Generative AI Conversation** (already installed) → voice assistant, general queries
- **Extended OpenAI Conversation** (install via HACS) → fish tank AI with vision support

This gives you full text + vision capabilities for your fish tank.

---

## 📦 Files Included

| File | Location | Purpose |
|------|----------|---------|
| `fish_tank_gemini.yaml` | `config/packages/` | HA package — automations, scripts, sensors |
| `GEMINI_DASHBOARD_CARDS.yaml` | Reference | Dashboard cards for 3 views |
| `GEMINI_INSTALLATION_GUIDE.md` | Reference | This file — complete setup instructions |
| `GEMINI_SETUP_GUIDE.md` | Reference | Detailed usage guide and examples |

---

## 🚀 Installation Steps

### Step 1 — Install Extended OpenAI Conversation (HACS)

This is for the **fish tank vision analysis**. Your existing Google AI integration stays as-is.

1. **HACS → Integrations**
2. Click **⊕ EXPLORE & DOWNLOAD REPOSITORIES**
3. Search: **"Extended OpenAI Conversation"**
4. Click the result (by jekalmin)
5. Click **DOWNLOAD**
6. **Restart Home Assistant** when prompted

### Step 2 — Get Gemini API Key (If You Don't Have It)

You might already have this from your Google AI integration. If not:

1. Go to: **https://aistudio.google.com/app/apikey**
2. Sign in with Google account
3. Click **"Create API Key"**
4. Select **"Create API key in new project"** (or existing)
5. **Copy the key** (starts with `AIza...`)

> You can reuse the same API key for both integrations — no extra cost.

### Step 3 — Configure Extended OpenAI Conversation

Open `configuration.yaml` and add:

```yaml
extended_openai_conversation:
  - name: Gemini Fish Tank
    api_key: AIzaSyD...YOUR_GEMINI_KEY_HERE...
    api_base: https://generativelanguage.googleapis.com/v1beta
    model: gemini-2.0-flash-exp
    vision_model: gemini-2.0-flash-exp
    max_tokens: 2048
```

**Critical details:**
- `api_base` must be exactly as shown above (Gemini URL, not OpenAI)
- `model` and `vision_model` both use `gemini-2.0-flash-exp`
- Can use the same API key as your Google Generative AI Conversation integration

### Step 4 — Restart Home Assistant

**Settings → System → Restart**

### Step 5 — Install the Package

Copy `fish_tank_gemini.yaml` to `config/packages/fish_tank_gemini.yaml`

Verify packages are enabled in `configuration.yaml`:
```yaml
homeassistant:
  packages: !include_dir_named packages
```

### Step 6 — Restart Home Assistant Again

**Settings → System → Restart**

### Step 7 — Add Dashboard Cards

Open your fish tank dashboard in edit mode:

**From `GEMINI_DASHBOARD_CARDS.yaml`:**

1. **Section 1** → Paste into **Overview** view
2. **Section 2** → Create new view:
   - Title: `Gemini AI`
   - Path: `gemini`
   - Icon: `mdi:robot`
   - Type: `masonry`
   - Paste all cards from Section 2
3. **Section 3** → Paste into **Settings** view

### Step 8 — Test the Integration

**Test 1 — Weekly Report (Text Only)**
1. Go to **Gemini AI view**
2. Press **"📋 Generate Weekly Report"**
3. Wait ~5-10 seconds
4. Check notification bell (top right)
5. Should see comprehensive report

**Test 2 — Vision Analysis**
1. Press **"📸 Analyze Snapshot Now"**
2. Wait ~10 seconds
3. Check notification bell
4. Should see fish health analysis based on camera image

**Test 3 — Custom Query**
1. Go to Developer Tools → Services
2. Run:
   ```yaml
   service: script.gemini_ask_about_tank
   data:
     question: "What's the current water quality status?"
   ```
3. Check notifications for AI response

If all 3 work → **fully operational!** ✅

---

## 🔧 Configuration Options

### Health Check Interval

Default: **12 hours** (automatic vision analysis every 12h)

Change via:
- **Settings → Gemini AI Settings → Health Check Interval**
- Range: 1–168 hours

### Enable/Disable Features

| Feature | Helper | Default |
|---------|--------|---------|
| Auto fish health analysis (vision) | `input_boolean.gemini_fish_health_analysis_enabled` | ✅ On |
| Auto daily summary (21:00) | `input_boolean.gemini_daily_summary_enabled` | ✅ On |
| Auto water quality predictions | `input_boolean.gemini_water_predictions_enabled` | ✅ On |

Toggle in **Gemini AI view → Status card** or via automations.

---

## 🎯 What Each Integration Does

### Google Generative AI Conversation (Already Installed)
- Voice assistant integration
- General HA queries via voice/chat
- **Not used by fish tank package** (but keeps working as normal)

### Extended OpenAI Conversation (New Install)
- **Used by fish tank package** for all AI features
- Supports vision (image analysis)
- Easier image handling than official integration
- **fish_tank_gemini.yaml calls this integration**

Both can run simultaneously with the same API key — no conflicts.

---

## 📊 Features Overview

| Feature | Trigger | What It Does |
|---------|---------|--------------|
| 🐟 **Fish Health Analysis** | Auto every 12h (or manual) | Analyzes camera snapshot — fish health, behavior, tank cleanliness |
| 📊 **Daily Summary** | Auto at 21:00 daily | Text analysis of all sensors, trends, tomorrow's actions |
| 🔮 **Water Predictions** | After water test/change + noon | Predicts days until water change needed |
| 💬 **Custom Queries** | Manual script call | Ask anything — "Why are my fish hiding?" |
| 📋 **Weekly Report** | Manual button | Comprehensive report with trends |

---

## 💰 API Usage & Costs

### Free Tier Limits
- **15 requests/minute**
- **1,000,000 tokens/day**
- **Vision included** (no extra cost)
- **No credit card required**

### Your Expected Usage
- ~20 requests/day
- ~15,000 tokens/day
- **1.5% of free tier** — massive headroom

You could run **65 more tanks** before hitting limits.

---

## 📂 File Locations

After installation, you should have:

```
config/
├── configuration.yaml                      ← add extended_openai_conversation block
└── packages/
    └── fish_tank_gemini.yaml              ← copy here
```

---

## 🐛 Troubleshooting

### "Service not found: extended_openai_conversation.query"
**Fix:** Extended OpenAI Conversation not installed or not restarted
1. Check HACS → Integrations → Extended OpenAI Conversation shows as installed
2. Restart HA

### "API key invalid" or "401 Unauthorized"
**Fix:** Wrong API key or typo
1. Verify key in configuration.yaml has no spaces/newlines
2. Generate new key at https://aistudio.google.com/app/apikey
3. Restart HA after updating

### "Model not found" or "404 error"
**Fix:** Wrong model name or api_base URL
1. Use exactly: `model: gemini-2.0-flash-exp`
2. Use exactly: `api_base: https://generativelanguage.googleapis.com/v1beta`
3. Restart HA

### Vision analysis returns "failed" or "image not found"
**Fix:** Snapshot not being saved
1. Check Frigate camera is working: **Media → Frigate**
2. Verify `/config/www/tank_snapshots/` folder exists
3. Test snapshot script: **Developer Tools → Services → script.fish_tank_capture_snapshot**
4. Check file appears: **File editor → www/tank_snapshots/latest.jpg**

### Rate limit errors (429)
**Fix:** Too many requests
1. Wait 1 minute
2. Reduce health check frequency in settings
3. Check for automation loops in HA logs

### Responses are cut off
**Fix:** max_tokens too low
1. Increase in configuration.yaml: `max_tokens: 4096`
2. Restart HA

### Notifications not appearing
**Fix:** Notification issue, not AI issue
1. Check bell icon (top right) manually
2. Enable persistent notifications: **Settings → Notifications**

---

## 🎓 Example Use Cases

### Morning Routine
```yaml
service: script.gemini_ask_about_tank
data:
  question: "Give me a quick morning status check — anything urgent?"
```

### After Water Change
```yaml
service: script.gemini_ask_about_tank
data:
  question: "I just did a 30% water change. What should I monitor closely over the next 24 hours?"
```

### Troubleshooting
```yaml
service: script.gemini_ask_about_tank
data:
  question: "My nitrate is 35ppm and rising. When should I do the next water change?"
```

### Planning
```yaml
service: script.gemini_generate_weekly_report
```
Then check notification for full report.

---

## 🔐 Privacy & Data

### What Gets Sent to Google
- Camera snapshots (JPG, ~500KB)
- Sensor readings (text)
- Maintenance timestamps (text)
- Tank age, capacity, settings (text)
- Your questions/prompts (text)

### Google's Policy
- Data **not used** to train AI models
- Retained temporarily for abuse prevention
- No personal identifying information in fish tank data
- See: https://ai.google.dev/gemini-api/terms

### If Privacy Is Critical
- Disable vision analysis (`input_boolean.gemini_fish_health_analysis_enabled` → off)
- Use text-only predictions and summaries
- Or self-host using Ollama (advanced, outside scope)

---

## ✅ Post-Installation Checklist

- [ ] Extended OpenAI Conversation installed via HACS
- [ ] Gemini API key added to configuration.yaml
- [ ] `extended_openai_conversation:` block added to configuration.yaml
- [ ] Home Assistant restarted (twice — after HACS install and after config)
- [ ] `fish_tank_gemini.yaml` copied to `config/packages/`
- [ ] Dashboard cards added to Overview, Gemini, Settings views
- [ ] Weekly report test successful
- [ ] Vision analysis test successful
- [ ] Custom query test successful
- [ ] Auto health check interval configured (Settings card)
- [ ] Daily summary enabled (check logs tomorrow at 21:00)

---

## 🚀 Next Steps

**First 24 Hours:**
1. Let automations run — daily summary will fire at 21:00
2. Check first health analysis appears (within 12 hours)
3. Try asking a custom question via Developer Tools

**First Week:**
1. Generate weekly report on Sunday
2. Review prediction accuracy against actual water changes
3. Adjust health check interval if needed

**Ongoing:**
- Use custom queries when you notice unusual behavior
- Review daily summaries for trend warnings
- Let auto-learn build baseline over 2-3 weeks
- Gemini's predictions improve with more historical data

---

## 📚 Additional Resources

- **Gemini API Docs:** https://ai.google.dev/gemini-api/docs
- **Extended OpenAI Conversation GitHub:** https://github.com/jekalmin/extended_openai_conversation
- **HA Conversation Integration:** https://www.home-assistant.io/integrations/conversation/
- **GEMINI_SETUP_GUIDE.md** (included) — detailed feature explanations and examples
- **GEMINI_DASHBOARD_CARDS.yaml** (included) — all dashboard YAML

---

## 💡 Pro Tips

1. **First week:** Set health check interval to 6 hours to build good baseline data
2. **After baseline:** Increase to 12-24 hours to conserve API quota
3. **Ask specific questions:** "Why did my pH drop overnight?" works better than "How's my tank?"
4. **Use weekly reports:** Best way to spot slow-developing issues
5. **Trust the predictions:** After 2-3 weeks, Gemini's water change predictions are typically ±1 day accurate
6. **Cross-reference:** If Gemini says something unusual, verify with your sensors — AI can misinterpret edge cases

---

**You're all set!** Both integrations are configured and the fish tank AI package will use Extended OpenAI Conversation for all features including vision.

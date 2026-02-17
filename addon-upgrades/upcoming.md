# 🚀 Fish Tank Monitor — Advanced Upgrades

Enhance your fish tank monitoring system with these powerful upgrades, from simple additions to advanced automation.

---

## 🎯 Quick Wins (Easy, High Impact)

### 1. **Smart Leak Detection**
**What**: Water sensors that alert before disaster strikes  
**Cost**: $15-30  
**Impact**: ⭐⭐⭐⭐⭐

**Hardware:**
- Aqara Water Leak Sensor (Zigbee) - $15
- Shelly Flood (WiFi) - $20
- Multiple sensors: under tank, near sump, filter area

**Placement:**
- Base of tank stand (catches leaks early)
- Under sump/canister filter
- Near water change buckets

**Automations:**
```yaml
- Immediate push notification + audible alarm
- Turn off auto-top-off system
- Turn off return pump (prevent overflow)
- Flash all lights in house
- Send photo from tank camera
```

---

### 2. **Automated Top-Off (ATO)**
**What**: Maintains water level automatically as it evaporates  
**Cost**: $30-150  
**Impact**: ⭐⭐⭐⭐⭐

**Options:**

**DIY ATO ($30):**
- HC-SR04 ultrasonic sensor (water level)
- ESPHome-controlled peristaltic pump
- Freshwater reservoir (5-10 gallon bucket)

**Commercial ATO ($80-150):**
- Tunze Osmolator
- AutoAqua Smart ATO
- Integrates with HA via smart plug

**HA Integration:**
- Monitor reservoir level
- Track daily evaporation rate
- Alert when reservoir needs refill
- Detect ATO failures (pump running too long)
- Graph evaporation trends (seasonal changes)

---

### 3. **PAR Meter (Light Intensity)**
**What**: Measures photosynthetically active radiation for planted tanks  
**Cost**: $200-500 (meter) or $50 (DIY sensor)  
**Impact**: ⭐⭐⭐ (planted tanks only)

**DIY Option:**
- TSL2591 lux sensor (I2C)
- Convert lux to PAR using calibration factor
- ESPHome integration

**Commercial:**
- Apogee MQ-500 (gold standard)
- Seneye Reef/Pond (continuous monitoring)

**HA Automations:**
- Adjust LED brightness based on PAR target
- Dim lights as bulbs age (maintain consistent PAR)
- Alert when PAR drops (bulb replacement needed)
- Create PAR heatmap across tank (multiple sensors)

---

### 4. **Multi-Point Temperature Monitoring**
**What**: Monitor temperature gradients across tank  
**Cost**: $5 per sensor  
**Impact**: ⭐⭐⭐⭐

**Why:** Detects heater failures, dead spots, stratification

**Setup:**
- DS18B20 sensors (waterproof, $5 each)
- Place at: top, middle, bottom, near heater, far corner
- ESPHome with 1-wire bus (supports 10+ sensors)

**Alerts:**
- Temperature delta >3°F (indicates stratification/circulation issue)
- One zone cold (heater failure)
- One zone hot (heater stuck on — DANGEROUS)

---

### 5. **Power Monitoring for Equipment**
**What**: Track power usage of each device, detect failures  
**Cost**: $10-15 per outlet  
**Impact**: ⭐⭐⭐⭐

**Hardware:**
- Sonoff S31 (Tasmota/ESPHome) - $12
- TP-Link Kasa plugs - $15
- Shelly Plug S - $15

**Monitor:**
- Filter (detect clog = higher wattage)
- Heater (stuck on = continuous power)
- Lights (bulb failure = lower wattage)
- UV sterilizer (verify it's actually running)

**Automations:**
- Alert if filter power 30% above baseline (clog)
- Alert if heater runs 24/7 (thermostat stuck)
- Alert if pump power drops to zero (failure)
- Monthly cost tracking per device

---

## 🧪 Water Chemistry Upgrades

### 6. **Continuous ORP (Oxidation-Reduction Potential)**
**What**: Real-time water quality indicator  
**Cost**: $80-150  
**Impact**: ⭐⭐⭐⭐

**Hardware:**
- Atlas Scientific EZO-ORP kit ($120)
- ORP probe
- ESPHome I2C integration

**Why it matters:**
- ORP 200-400 mV = healthy freshwater
- Sudden drop = ammonia spike, organic waste
- Gradual decline = filter needs cleaning
- Validates water changes (ORP should rise)

**Automations:**
- Alert if ORP <150 mV (water quality crash)
- Trigger ozone generator (advanced setups)
- Log ORP recovery after water changes

---

### 7. **Conductivity / TDS Sensor (Continuous)**
**What**: Monitor dissolved solids in real-time  
**Cost**: $100  
**Impact**: ⭐⭐⭐⭐

**Hardware:**
- Atlas Scientific EZO-EC kit
- Conductivity probe K1.0
- ESPHome/I2C

**Use cases:**
- Detect TDS creep (gradually rising minerals)
- Verify RO/DI water quality (should be ~0 ppm)
- Monitor remineralization dosing
- Track evaporation vs water loss

---

### 8. **Automated Water Change System**
**What**: Scheduled partial water changes, hands-free  
**Cost**: $150-300  
**Impact**: ⭐⭐⭐⭐⭐

**Components:**
- 2x peristaltic pumps (waste out, fresh in)
- Solenoid valves (water supply, drain)
- Float switches (safety)
- ESPHome control

**HA Integration:**
```yaml
script:
  fish_tank_auto_water_change_25_percent:
    sequence:
      - Turn off heater (prevent dry heating)
      - Open drain valve
      - Run waste pump for 15 min (remove 25%)
      - Close drain valve
      - Open fresh water valve
      - Run fill pump for 15 min
      - Close fresh water valve
      - Turn on heater
      - Log water change timestamp
```

**Safety:**
- High-level float kills fill pump (prevent overflow)
- Low-level float kills drain pump (prevent dry tank)
- Max runtime limits (30 min fail-safe)

---

### 9. **Sump Camera (Secondary)**
**What**: Monitor equipment, check for leaks/issues  
**Cost**: $25 (old phone) or $40 (IP camera)  
**Impact**: ⭐⭐⭐

**Use cases:**
- Verify protein skimmer is producing foam
- Detect sump water level issues
- Check for leaks under tank
- Monitor refugium (macroalgae growth)
- AI detection: detect overflowing skimmer cup

---

## 🤖 Advanced Automation

### 10. **Predictive Maintenance AI**
**What**: ML model predicts equipment failures before they happen  
**Cost**: Free (software)  
**Impact**: ⭐⭐⭐⭐⭐

**Using HA's built-in tools:**
```yaml
# Trend sensor - detects gradual changes
- platform: trend
  sensors:
    filter_clogging:
      entity_id: sensor.filter_power
      sample_duration: 86400  # 1 day
      max_samples: 7          # 7 days
      min_gradient: 0.05      # 5% increase/day
```

**Predictions:**
- Filter media lifespan (based on flow rate decay)
- Heater failure (power consumption pattern changes)
- pH probe degradation (drift over time)
- Light bulb end-of-life (PAR decline)

**Automations:**
- "Filter flow decreased 15% over 2 weeks — clean in 3 days"
- "Heater power cycling increasing — replace within 1 month"
- "pH readings drifting +0.1 per week — recalibrate probe"

---

### 11. **Voice Control & Announcements**
**What**: Talk to your tank, get voice updates  
**Cost**: Free (if you have smart speaker)  
**Impact**: ⭐⭐⭐

**Alexa/Google Assistant examples:**
- "Alexa, what's my tank temperature?" → "77.2 degrees"
- "Hey Google, feed the fish" → triggers feeder
- "Alexa, how many fish are visible?" → "3 out of 4 fish detected"
- "Hey Google, when was the last water change?" → "5 days ago"

**Proactive announcements:**
```yaml
automation:
  - trigger:
      - platform: numeric_state
        entity_id: sensor.fish_tank_ammonia
        above: 0.5
    action:
      - service: tts.google_translate_say
        data:
          message: "Warning: Fish tank ammonia is critical at 0.8 PPM. Water change required immediately."
          entity_id: media_player.bedroom_speaker
```

---

### 12. **Feeding Schedule Optimizer**
**What**: AI adjusts feeding based on activity, water quality  
**Cost**: Free (software)  
**Impact**: ⭐⭐⭐⭐

**Logic:**
```yaml
# Reduce feeding if:
- Ammonia >0.1 ppm (overfeeding)
- Nitrate >40 ppm (excess nutrients)
- Fish activity <30% (sick/stressed, won't eat)
- Water change overdue (prevent further nutrient buildup)

# Increase feeding if:
- Fish activity >80% for 3+ days (healthy, active)
- Nitrate <5 ppm (low nutrients, fish hungry)
- Recent water change (fresh water, safe to feed more)
```

**HA Implementation:**
```yaml
sensor:
  - platform: template
    sensors:
      fish_tank_feeding_amount_adjustment:
        value_template: >
          {% set base = 2.0 %}  {# grams #}
          {% set amm = states('sensor.fish_tank_ammonia')|float(0) %}
          {% set no3 = states('sensor.fish_tank_nitrate')|float(0) %}
          {% set activity = states('sensor.fish_tank_activity_score')|float(50) %}
          
          {% if amm > 0.1 %}{% set base = base * 0.5 %}{% endif %}
          {% if no3 > 40 %}{% set base = base * 0.7 %}{% endif %}
          {% if activity < 30 %}{% set base = base * 0.6 %}{% endif %}
          {% if activity > 80 %}{% set base = base * 1.2 %}{% endif %}
          
          {{ base|round(1) }}
```

---

### 13. **Moonlight Simulation (Circadian Rhythm)**
**What**: Gradual sunrise/sunset, lunar cycle  
**Cost**: Free (if you have smart lights)  
**Impact**: ⭐⭐⭐⭐

**Benefits:**
- Reduces fish stress (no sudden lights on/off)
- Encourages spawning (follows natural cycles)
- Looks beautiful

**Implementation:**
```yaml
automation:
  # Sunrise (60 min fade-in)
  - alias: "Fish Tank: Sunrise"
    trigger:
      - platform: sun
        event: sunrise
        offset: "-00:30:00"
    action:
      - service: light.turn_on
        data:
          entity_id: light.fish_tank_led
          brightness: 0
          kelvin: 2700  # Warm
      - delay: 00:05:00
      - repeat:
          count: 12
          sequence:
            - service: light.turn_on
              data:
                entity_id: light.fish_tank_led
                brightness: "{{ repeat.index * 21 }}"  # 0→255 over 60min
                kelvin: "{{ 2700 + (repeat.index * 200) }}"  # Warm→Cool
            - delay: 00:05:00
  
  # Moonlight (10% blue LED, all night)
  - alias: "Fish Tank: Moonlight"
    trigger:
      - platform: sun
        event: sunset
        offset: "01:00:00"
    action:
      - service: light.turn_on
        data:
          entity_id: light.fish_tank_led
          brightness: 25
          rgb_color: [100, 150, 255]  # Blue
```

**Advanced:** Sync to actual lunar cycle
```yaml
sensor:
  - platform: moon
template:
  - sensor:
      - name: fish_tank_moonlight_brightness
        state: >
          {% set phase = states('sensor.moon') %}
          {% if phase == 'full_moon' %}50
          {% elif phase == 'new_moon' %}5
          {% else %}25{% endif %}
```

---

### 14. **Vacation Mode Automation**
**What**: Reduces maintenance, extends time between servicing  
**Cost**: Free  
**Impact**: ⭐⭐⭐⭐⭐

**When activated:**
```yaml
script:
  fish_tank_vacation_mode_on:
    sequence:
      # Reduce feeding
      - service: input_number.set_value
        data:
          entity_id: input_number.fish_tank_daily_feeds
          value: 1  # Once daily instead of 2x
      
      # Reduce lighting (algae control)
      - service: input_number.set_value
        data:
          entity_id: input_number.fish_tank_light_hours
          value: 8  # 8h instead of 10h
      
      # Increase alert sensitivity
      - service: input_boolean.turn_on
        entity_id: input_boolean.fish_tank_notify_email
      
      # Enable extra monitoring
      - service: automation.turn_on
        entity_id:
          - automation.fish_tank_all_fish_hidden_alert
          - automation.fish_tank_low_activity_alert
          - automation.fish_tank_camera_offline_alert
      
      # Notification
      - service: notify.mobile_app_your_phone
        data:
          title: "🏖 Vacation Mode Active"
          message: >
            Tank maintenance reduced. Feeding 1x/day.
            Extra monitoring enabled. Safe travels!
```

---

## 📊 Advanced Analytics

### 15. **Long-term Trend Analysis & Reporting**
**What**: Weekly/monthly reports with insights  
**Cost**: Free  
**Impact**: ⭐⭐⭐⭐

**Using HA's history stats:**
```yaml
sensor:
  # Average weekly temperature
  - platform: statistics
    entity_id: sensor.fish_tank_temperature
    state_characteristic: mean
    sampling_size: 10080  # 7 days in minutes
    name: fish_tank_temp_weekly_avg
  
  # Ammonia spikes this month
  - platform: history_stats
    name: fish_tank_ammonia_spikes_monthly
    entity_id: sensor.fish_tank_ammonia_status
    state: "critical"
    type: count
    start: "{{ now().replace(day=1, hour=0, minute=0, second=0) }}"
    end: "{{ now() }}"
```

**Weekly Email Report:**
```yaml
automation:
  - alias: "Fish Tank: Weekly Report"
    trigger:
      - platform: time
        at: "09:00:00"
      - condition: template
        value_template: "{{ now().weekday() == 6 }}"  # Sunday
    action:
      - service: notify.email
        data:
          title: "🐟 Weekly Tank Report"
          message: |
            Tank Health This Week:
            
            Temperature: {{ states('sensor.fish_tank_temp_weekly_avg') }}°F avg
            pH Range: {{ state_attr('sensor.fish_tank_ph', 'min_value') }} - {{ state_attr('sensor.fish_tank_ph', 'max_value') }}
            
            Water Changes: {{ states('sensor.fish_tank_water_changes_this_month') }}
            Feedings: {{ states('sensor.fish_tank_feedings_this_week') }}
            
            Alerts: {{ states('sensor.fish_tank_ammonia_spikes_monthly') }} ammonia spikes
            
            Fish Activity: {{ states('sensor.fish_tank_activity_score') }}% average
            Visible Fish: {{ states('sensor.fish_tank_fish_visibility') }}% average
            
            Action Items:
            - Water change due in {{ states('sensor.fish_tank_water_change_due_in') }} days
            - Filter clean due in {{ states('sensor.fish_tank_filter_clean_due_in') }} days
```

---

### 16. **Cost Tracking & Efficiency**
**What**: Monitor electricity, water, supply costs  
**Cost**: Free  
**Impact**: ⭐⭐⭐

**Track:**
- Power usage (kWh/month) × electricity rate = monthly cost
- Water changes (gallons/month) × water rate = water cost
- Supply costs (food, chemicals, test kits)

```yaml
sensor:
  - platform: template
    sensors:
      fish_tank_monthly_power_cost:
        value_template: >
          {% set heater_w = states('sensor.fish_tank_heater_power')|float(0) %}
          {% set filter_w = states('sensor.fish_tank_filter_power')|float(0) %}
          {% set light_w = states('sensor.fish_tank_light_power')|float(0) %}
          {% set total_w = heater_w + filter_w + light_w %}
          {% set kwh_month = (total_w * 24 * 30) / 1000 %}
          {% set rate = 0.12 %}  {# $/kWh - adjust to your rate #}
          {{ (kwh_month * rate)|round(2) }}
        unit_of_measurement: "$"
```

---

## 🔬 Experimental / Cutting Edge

### 17. **Computer Vision Fish Health Analysis**
**What**: AI detects sick fish from camera footage  
**Cost**: Free (software) + $60 (Coral TPU recommended)  
**Impact**: ⭐⭐⭐⭐⭐

**Train custom model to detect:**
- Ich (white spots)
- Fin rot (damaged fins)
- Bloat (swollen abdomen)
- Clamped fins (stress indicator)
- Erratic swimming (poisoning/ammonia)
- Gasping at surface (low O₂)

**Using TensorFlow + Frigate:**
1. Collect 500+ images of healthy vs sick fish
2. Label in Roboflow
3. Train YOLOv8 model
4. Deploy to Frigate
5. Alert when sick fish detected

---

### 18. **Aquaponics Integration**
**What**: Use tank water to grow plants, return filtered water  
**Cost**: $100-500  
**Impact**: ⭐⭐⭐⭐⭐

**HA Monitoring:**
- Plant bed moisture
- Grow light schedule
- Pump flow rate
- Nutrient levels (NO₃ should be consumed by plants)

**Benefits:**
- Natural nitrate removal (planted bed acts as biofilter)
- Grow herbs/veggies using tank waste
- Reduced water change frequency

---

### 19. **Bluetooth Mesh Water Sensors**
**What**: Wireless sensors throughout tank (no wires!)  
**Cost**: $80-150  
**Impact**: ⭐⭐⭐

**Using ESPHome Bluetooth Proxy:**
- Xiaomi Mi Temperature/Humidity sensors (submersible mods)
- Inkbird Bluetooth pH probes
- Ruuvi Tag environmental sensors

**No wires into tank** = cleaner, safer

---

### 20. **UV-C Dosing Control**
**What**: Auto-adjust UV sterilizer runtime based on water clarity  
**Cost**: Free (if you have UV already)  
**Impact**: ⭐⭐⭐

```yaml
automation:
  - alias: "Fish Tank: UV Sterilizer Smart Control"
    trigger:
      - platform: state
        entity_id: sensor.fish_tank_turbidity
    action:
      - choose:
          # High turbidity → Run UV 24/7
          - conditions:
              - condition: numeric_state
                entity_id: sensor.fish_tank_turbidity
                above: 10
            sequence:
              - service: switch.turn_on
                entity_id: switch.fish_tank_uv_sterilizer
          
          # Medium turbidity → Run 12h/day
          - conditions:
              - condition: numeric_state
                entity_id: sensor.fish_tank_turbidity
                above: 5
                below: 10
            sequence:
              - service: switch.turn_on
                entity_id: switch.fish_tank_uv_sterilizer
              - delay: 12:00:00
              - service: switch.turn_off
                entity_id: switch.fish_tank_uv_sterilizer
          
          # Clear water → UV off (save bulb life)
          - conditions:
              - condition: numeric_state
                entity_id: sensor.fish_tank_turbidity
                below: 5
            sequence:
              - service: switch.turn_off
                entity_id: switch.fish_tank_uv_sterilizer
```

---

## 💰 Cost vs Impact Summary

| Upgrade | Cost | Impact | Difficulty |
|---------|------|--------|------------|
| Leak Detection | $15 | ⭐⭐⭐⭐⭐ | Easy |
| Auto Top-Off | $30-150 | ⭐⭐⭐⭐⭐ | Medium |
| Multi-temp Sensors | $25 | ⭐⭐⭐⭐ | Easy |
| Power Monitoring | $50 | ⭐⭐⭐⭐ | Easy |
| ORP Sensor | $120 | ⭐⭐⭐⭐ | Medium |
| Conductivity Sensor | $100 | ⭐⭐⭐⭐ | Medium |
| Auto Water Change | $250 | ⭐⭐⭐⭐⭐ | Hard |
| PAR Meter | $50-500 | ⭐⭐⭐ | Easy |
| Sump Camera | $25 | ⭐⭐⭐ | Easy |
| Voice Control | Free | ⭐⭐⭐ | Easy |
| Moonlight Sim | Free | ⭐⭐⭐⭐ | Easy |
| Vacation Mode | Free | ⭐⭐⭐⭐⭐ | Easy |
| Trend Analysis | Free | ⭐⭐⭐⭐ | Medium |
| Cost Tracking | Free | ⭐⭐⭐ | Easy |
| Fish Health AI | $60 | ⭐⭐⭐⭐⭐ | Hard |

---

## 🎯 Recommended Upgrade Path

**Phase 1 - Safety First (Week 1):**
1. Leak detection sensors ($15)
2. Multi-point temperature ($25)
3. Power monitoring ($50)

**Phase 2 - Automation (Month 1):**
4. Auto top-off ($80)
5. Voice control (free)
6. Vacation mode (free)
7. Moonlight simulation (free)

**Phase 3 - Chemistry Precision (Month 2):**
8. ORP sensor ($120)
9. Conductivity sensor ($100)

**Phase 4 - Advanced (Month 3+):**
10. Auto water change system ($250)
11. Fish health AI ($60 Coral TPU)
12. PAR meter for planted tanks ($50 DIY)

**Total for all phases**: ~$770 (hardware) + endless free software automation

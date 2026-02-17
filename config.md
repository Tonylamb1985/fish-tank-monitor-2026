###############################################################################
# configuration.yaml — Fish Tank Monitor Setup
# ─────────────────────────────────────────────────────────────────────────────
# HOW TO USE THIS FILE
# ─────────────────────────────────────────────────────────────────────────────
#
# OPTION A — Fresh / simple config (no existing configuration.yaml customisations)
#   1. Copy the entire fish_tank_ha/ folder into your HA config directory:
#        /config/fish_tank_ha/
#        /config/packages/   ← copy the packages/ folder here
#   2. Add ONLY the homeassistant: block below into your existing configuration.yaml
#      (do NOT paste the whole file if you already have a configuration.yaml)
#
# OPTION B — You already use packages
#   Just copy the 4 package YAML files into your existing packages/ folder.
#   No changes to configuration.yaml needed — HA will auto-discover them.
#
# ─────────────────────────────────────────────────────────────────────────────
# STEP 1 — Add this to your configuration.yaml
# ─────────────────────────────────────────────────────────────────────────────
# If you already have a homeassistant: section, just add the packages line.
# If you don't, paste the whole block below.
#
# NOTE: !include_dir_named means HA will load every .yaml file in the
#       packages/ folder as a named package automatically — no need to
#       list them individually.

homeassistant:
  packages: !include_dir_named packages

# ── FRONTEND THEMES ───────────────────────────────────────────────────────────
# Enable custom themes (like the Aquarium theme)
frontend:
  themes: !include_dir_merge_named themes

# ─────────────────────────────────────────────────────────────────────────────
# STEP 2 — Verify your config/ folder structure looks like this:
# ─────────────────────────────────────────────────────────────────────────────
#
#   config/
#   ├── configuration.yaml          ← contains the homeassistant: block above
#   └── packages/
#       ├── fish_tank_helpers.yaml
#       ├── fish_tank_sensors.yaml
#       ├── fish_tank_automations.yaml
#       └── fish_tank_scripts.yaml
#
# The dashboard file (fish_tank_dashboard.yaml) is NOT a package —
# it gets pasted into the Lovelace Raw Config Editor (see Step 4 below).
#
# ─────────────────────────────────────────────────────────────────────────────
# STEP 3 — Restart Home Assistant
# ─────────────────────────────────────────────────────────────────────────────
#   Settings → System → Restart
#
#   After restart, verify everything loaded:
#   Settings → System → Logs — look for errors related to fish_tank_*
#
#   Also check Developer Tools → States and search "fish_tank" —
#   you should see all the input helpers and template sensors appear.
#
# ─────────────────────────────────────────────────────────────────────────────
# STEP 4 — Add the Dashboard
# ─────────────────────────────────────────────────────────────────────────────
#   1. Settings → Dashboards → Add Dashboard
#      Name: Freshwater Tank
#      Icon: mdi:fishbowl-outline
#   2. Open the new dashboard
#   3. Click ⋮ (top right) → Edit Dashboard
#   4. Click ⋮ again → Raw Configuration Editor
#   5. Select all existing text → delete it
#   6. Paste the full contents of fish_tank_dashboard.yaml → Save
#
# ─────────────────────────────────────────────────────────────────────────────
# STEP 5 — Set your notification target
# ─────────────────────────────────────────────────────────────────────────────
#   Open fish_tank_automations.yaml and fish_tank_scripts.yaml
#   Find every line that says:
#       service: notify.mobile_app_your_phone
#   Replace "mobile_app_your_phone" with your actual notifier name.
#
#   To find your notifier:
#   Developer Tools → Services → search "notify" → look for your device
#   Common examples:
#       notify.mobile_app_johns_iphone
#       notify.mobile_app_samsung_galaxy
#       notify.persistent_notification   ← always works, shows in HA sidebar
#
# ─────────────────────────────────────────────────────────────────────────────
# STEP 6 — Connect your hardware sensors
# ─────────────────────────────────────────────────────────────────────────────
#   Your physical sensors must expose entities with these exact IDs,
#   OR open the dashboard YAML and do a find+replace with your real IDs.
#
#   Expected entity IDs:
#     sensor.fish_tank_temperature      DS18B20 / Atlas EZO-RTD via ESPHome
#     sensor.fish_tank_ph               Atlas Scientific EZO-pH
#     sensor.fish_tank_ammonia          Atlas Scientific EZO-NH3
#     sensor.fish_tank_nitrite          Hanna meter or manual input_number
#     sensor.fish_tank_nitrate          Hanna meter or manual input_number
#     sensor.fish_tank_dissolved_o2     Atlas Scientific EZO-DO
#     sensor.fish_tank_gh               Manual input_number (drop test result)
#     sensor.fish_tank_kh               Manual input_number (drop test result)
#     sensor.fish_tank_tds              TDS meter via ESPHome
#     sensor.fish_tank_co2              CO2 sensor (optional, planted tanks)
#     sensor.fish_tank_turbidity        DF Robot Turbidity Sensor
#     sensor.fish_tank_water_level      HC-SR04 ultrasonic via ESPHome
#     switch.fish_tank_filter           Smart plug or ESPHome relay
#     switch.fish_tank_heater           Smart plug or ESPHome relay
#     switch.fish_tank_air_pump         ESPHome relay
#     switch.fish_tank_co2_regulator    ESPHome solenoid relay
#     switch.fish_tank_uv_sterilizer    Smart plug or ESPHome relay
#     switch.fish_tank_led_lights       Smart plug / WLED
#     switch.fish_tank_auto_feeder      ESPHome relay
#
# ─────────────────────────────────────────────────────────────────────────────
# STEP 7 — Install HACS Frontend Cards (required for dashboard)
# ─────────────────────────────────────────────────────────────────────────────
#   HACS → Frontend → search and install each:
#     mini-graph-card
#     button-card
#     apexcharts-card
#     mushroom-template-card (mushroom-strategy)
#     bar-card
#
# ─────────────────────────────────────────────────────────────────────────────
# TROUBLESHOOTING
# ─────────────────────────────────────────────────────────────────────────────
#
#   "Could not load packages"
#     → Make sure the packages/ folder is directly inside config/
#     → Check spelling: packages: !include_dir_named packages
#     → No trailing slash: packages  NOT  packages/
#
#   Template sensor shows "unknown"
#     → Your hardware sensor entity ID doesn't match what's expected
#     → Go to Developer Tools → Template, paste:
#          {{ states('sensor.fish_tank_temperature') }}
#       to test if the entity exists
#
#   Automations not appearing
#     → Check Settings → System → Logs for YAML errors in the package files
#     → Settings → Automations — search "Fish Tank" to confirm they loaded
#
#   "Integration 'packages' not found"
#     → packages is a core HA feature, not an integration
#     → Make sure you're on Home Assistant OS or Supervised (not Core in Docker
#       with a stripped config)
###############################################################################

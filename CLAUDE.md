# Technical Documentation

## !!!! CRITICAL - DO NOT UPLOAD FIRMWARE !!!!

**NEVER USE `esphome upload` OR `esphome run` COMMANDS**

The user will handle all firmware uploads manually. Claude's role is LIMITED TO:
- Compiling firmware (`esphome compile`)
- Reading logs (`esphome logs`)
- Code analysis and editing

**DO NOT:**
- Upload firmware to any device
- Run `esphome upload`
- Run `esphome run`
- Flash any device via USB or OTA

**ALWAYS wait for the user to upload firmware themselves.**

## !!!! END CRITICAL SECTION !!!!

---

ESPHome-based water softener salt level monitoring system using M5Stack ATOM hardware and VL53L0X ToF distance sensor.

## Hardware

### ATOM S3 (ESP32-S3)
- ESP-IDF framework (persistence issues resolved in ESPHome 2025.10+)
- Web server enabled (http://water-softener-monitor.local or -s3-dev.local for dev)
- I2C: SDA=GPIO2, SCL=GPIO1
- Button: GPIO41 (internal pullup)
- RGB LED: GPIO35 (SK6812)
- Configuration via web UI or Home Assistant

## Project Structure

**Core:**
- `src/water-softener-s3-core.yaml` - Main sensor logic and configuration

**Web Installer:**
- `src/water-softener-s3-webinstall.yaml` - For web-based initial flash

**Development:**
- `src/water-softener-s3-dev.yaml` - ATOM S3 (water-softener-dev.lan.ram6.com)

**Production:**
- Managed via ESPHome Dashboard (water-softener-prod.lan.ram6.com)

**Dev Config Strategy:**
- Dev config uses `!include` for fast local iteration on core logic
- Production uses ESPHome Dashboard with GitHub package refs (`@latest` tag)

## Core Logic

### Sensor Update Strategy
Distance sensor uses `update_interval: never` and is manually triggered by interval component. Enables dynamic intervals via `update_interval_seconds` parameter without reflashing.

### Out-of-Range Handling
- `last_valid_status` - Persisted across reboots
- `sensor_out_of_range` binary sensor - Indicates sensor reading issues

### Lambda Functions
1. Out of range detection - Checks sensor validity
2. Regeneration detection - Derivative-based cycle detection
3. Status text generation - Simple threshold-based status with hysteresis

### Calibration Mode
Fast sensor polling (100ms) for setup/testing. Auto-disables after 5 minutes.

## Water Softener Regeneration Cycle Behavior

### Background
Water softeners run a nightly regeneration cycle (typically 2-3 AM) that significantly affects ToF sensor readings. Understanding this behavior is critical for proper sensor configuration and filtering strategy.

### Regeneration Cycle Stages

**Typical timing: 02:10-03:10 AM PDT (09:10-10:10 UTC)**

1. **Brine Draw Stage (~30-60 min)**
   - Concentrated brine is **pumped OUT** of the salt tank
   - Water level in brine tank **drops significantly** (nearly empties)
   - Brine flows to resin tank to regenerate the softener resin
   - ToF sensor measures **longer distances** as water level drops
   - Salt level reading drops 15-20% (appears as false "depletion")

2. **Backwash/Rinse Stages (~10-20 min)**
   - Resin tank is flushed, brine tank remains mostly empty
   - Sensor continues reading longer distances

3. **Refill Stage (~5-15 min)**
   - Fresh water flows back into brine tank
   - Water level **rises back up** to prepare brine for next cycle
   - ToF sensor measures **shorter distances** as water level rises
   - Salt level reading "recovers" back to baseline

4. **Post-Cycle Settling (~30-60 min)**
   - Water surface stabilizes
   - Readings gradually return to pre-cycle values

### Observed Data Patterns (InfluxDB Analysis, Oct 24 2025)

**5-minute sample data:**
```
08:00-09:05 AM UTC: Normal stable readings (~59-60%)
09:10 AM UTC:       Sudden drop begins (59.81% → 57.37%)
09:15 AM UTC:       Major drop (44.72%)
09:20-10:00 AM UTC: Sustained low readings (~41-42%) - BRINE DRAW ACTIVE
10:10 AM UTC:       Recovery begins (43% → 52%)
10:15-11:30 AM UTC: Gradual return to baseline
11:30+ AM UTC:      Back to normal (~57%)
```

**Impact on readings:**
- **~28 affected samples** out of 288 daily (10% of data)
- **15-20% apparent drop** in salt level during brine draw
- **1-2 hour recovery period** after cycle completes
- **Actual salt consumption:** ~2% per cycle (masked by water level changes and ±1-2% measurement noise)

### Physical Explanation

The ToF distance sensor measures distance to the **water surface**, not the salt bed. During regeneration:

- **Brine Draw**: Water level drops → sensor reads longer distance → calculates lower salt percentage
- **Refill**: Water level rises → sensor reads shorter distance → calculates higher salt percentage
- **Reality**: Salt dissolves slowly (~2% consumed per cycle), but water level changes dramatically

The "recovery" in sensor readings is **not salt regenerating** - it's the brine tank refilling with fresh water for tomorrow's cycle.

### Implications for Sensor Configuration

**Key Constraints:**
1. Cannot accurately measure salt consumption during regeneration cycle
2. ~2.5 hours of unreliable data daily (02:00-04:30 AM local time)
3. Sensor tracks **water level changes** more than **salt depletion**

**Recommended Strategies:**
1. **Longer update intervals** (5 min - 1 hour) to reduce database load
2. **Heavy filtering** (median + exponential moving average) to smooth fluctuations
3. **Optional blackout period** (01:30-04:30 AM) to ignore regeneration cycle data
4. **Backwash detection** via sudden drops >10% can confirm cycle execution

**Update Frequency Trade-offs:**
- 20 seconds (v1.4.0): 4,320 readings/day, excessive database load, captures surface noise
- 5 minutes (1.5.0+): 288 readings/day, good balance, can detect backwash cycle
- 1 hour: 24 readings/day, minimal load, stable readings, misses regeneration details

## Regeneration Detection

### Binary Sensor: "Regeneration Cycle Active"

**Derivative-based detection** - monitors rate of change (cm/min) rather than absolute thresholds.

**Detection Logic:**
- **START**: Sustained rising derivative above threshold (default 0.30 cm/min) for 6 consecutive readings
- **END**: Sustained stable derivative below threshold (default 0.02 cm/min) for 10 consecutive readings
- **Safety timeout**: 4 hours maximum
- **Minimum duration**: Configurable (default 10 min) - cycles ending earlier are treated as false alarms

**Configurable Parameters (via Home Assistant):**
- Regen Detection Rise Threshold (cm/min)
- Regen Detection Stable Threshold (cm/min)
- Regen Minimum Duration (minutes)

**Status Sensor Behavior:**
- "Salt Status" text sensor freezes during regeneration to prevent false alerts

### Home Assistant Automation Examples

**1. Simple Refill Alert**
```yaml
automation:
  - alias: "Water Softener - Low Salt Alert"
    trigger:
      - platform: state
        entity_id: text_sensor.water_softener_salt_status
        to: "Low - Add Salt Soon"
    action:
      - service: notify.mobile_app
        data:
          title: "Water Softener"
          message: "Salt level is low. Please refill soon."
```

**2. Missed Regeneration Cycle (Time-Based Softeners)**
```yaml
automation:
  - alias: "Water Softener - Missed Regeneration"
    trigger:
      - platform: state
        entity_id: binary_sensor.water_softener_regeneration_cycle_active
        to: "off"
        for:
          hours: 36  # Adjust based on your softener schedule
    action:
      - service: notify.mobile_app
        data:
          title: "Water Softener Alert"
          message: "No regeneration detected in 36 hours. Check softener operation."

# For demand-based softeners, use longer delay:
# hours: 168  # 7 days
```

**3. Abnormally Long Regeneration Cycle**
```yaml
automation:
  - alias: "Water Softener - Long Regeneration"
    trigger:
      - platform: state
        entity_id: binary_sensor.water_softener_regeneration_cycle_active
        to: "on"
        for:
          hours: 3
    action:
      - service: notify.mobile_app
        data:
          title: "Water Softener Warning"
          message: "Regeneration cycle has been running for over 3 hours."
```

**4. Malfunction Detection (No Recovery)**
```yaml
automation:
  - alias: "Water Softener - Malfunction Detection"
    trigger:
      - platform: state
        entity_id: binary_sensor.water_softener_regeneration_cycle_active
        to: "on"
        for:
          hours: 2
    condition:
      - condition: numeric_state
        entity_id: sensor.water_softener_salt_level
        below: 45  # Still very low after 2 hours
    action:
      - service: notify.mobile_app
        data:
          title: "Water Softener Malfunction"
          message: "Salt level not recovering during regeneration. Possible system malfunction."
          data:
            priority: high
```

**5. Regeneration Cycle Notification (Informational)**
```yaml
automation:
  - alias: "Water Softener - Regeneration Started"
    trigger:
      - platform: state
        entity_id: binary_sensor.water_softener_regeneration_cycle_active
        to: "on"
    action:
      - service: notify.mobile_app
        data:
          title: "Water Softener"
          message: "Regeneration cycle started."

  - alias: "Water Softener - Regeneration Completed"
    trigger:
      - platform: state
        entity_id: binary_sensor.water_softener_regeneration_cycle_active
        to: "off"
    action:
      - service: notify.mobile_app
        data:
          title: "Water Softener"
          message: "Regeneration cycle completed."
```

**Note:** Adjust thresholds and timing based on your specific water softener model and usage patterns. Time-based softeners typically regenerate daily, while demand-based softeners may run every 3-7 days.

## Status Hysteresis

Current implementation uses simple distance-based hysteresis (±2cm dead zone) to prevent status bouncing between "Good" and "Refill" states.

## Configuration Parameters

**Measurement (via Home Assistant numbers):**
- Tank Height (30-150cm, default 100cm)
- Refill Threshold Distance (10-120cm, default 43cm)
- Update Interval (30-300s, default 60s)

**Regeneration Detection:**
- Rise Threshold (0.01-0.50 cm/min, default 0.30)
- Stable Threshold (0.005-0.10 cm/min, default 0.02)
- Minimum Duration (1-30 min, default 10)

**Status:**
- "Good" when distance < threshold
- "Refill" when distance >= threshold

## Version Management

**Version String Format**: `X.Y.Z-variant` (e.g., `1.9.3-s3`)

**How Versioning Works:**
1. Version defined in webinstall config (`src/water-softener-s3-webinstall.yaml`):
   ```yaml
   substitutions:
     version: "1.9.3-s3"
   ```

2. Version passed to core config via `${version}` substitution

3. Exposed via `text_sensor.water_softener_firmware_version` in Home Assistant

**Important**: Git tags use NO "v" prefix (use `1.9.3`, NOT `v1.9.3`)

## Development Commands

**CRITICAL: NEVER flash/upload firmware without explicit user approval**

**ABSOLUTELY FORBIDDEN without user approval:**
- `esphome upload` (network or USB)
- `esphome run` (compiles + uploads automatically)
- Any serial flashing commands
- Any OTA update operations
- ANY method that deploys firmware to physical hardware

**Why this is critical:**
- Cannot verify which device is physically connected (USB)
- Cannot verify device state (in use, running automations, production vs dev)
- Network addresses may point to production devices users depend on
- Wrong device could be bricked or have critical functionality disrupted
- Firmware deployment is irreversible and affects physical hardware

**Safe operations (no approval needed):**
- `esphome compile` - Build firmware only
- `esphome logs` - Read-only monitoring
- File reads and code analysis

**Required workflow:**
1. Compile firmware
2. Show compilation results to user
3. Wait for explicit approval with device confirmation
4. Only then proceed with upload/flash

```bash
# ATOM S3 Dev
esphome compile src/water-softener-s3-dev.yaml  # SAFE - compile only
esphome logs src/water-softener-s3-dev.yaml --device water-softener-dev.lan.ram6.com  # SAFE - read only

# FORBIDDEN - Never run these without explicit user approval:
# esphome upload src/water-softener-s3-dev.yaml --device water-softener-dev.lan.ram6.com
# esphome run src/water-softener-s3-dev.yaml --device water-softener-dev.lan.ram6.com
```

## Release Workflow

1. Update version in `src/water-softener-s3-webinstall.yaml`
2. Update `dashboard_import` URL to reference new tag
3. Compile firmware: `esphome compile src/water-softener-s3-webinstall.yaml`
4. Copy firmware to docs/: `cp src/.esphome/build/.../firmware.factory.bin docs/water-softener-monitor-s3-vX.Y.Z.bin`
5. Update `docs/manifest-s3.json` with new version and filename
6. Commit changes
7. Create and push git tag (NO "v" prefix): `git tag -a 1.9.3 -m "Release 1.9.3"`
8. Update `latest` tag: `git tag -f latest && git push origin latest -f`
9. Push to GitHub

## Deployment Model

1. Web installer firmware ships with no encryption
2. User configures WiFi via BLE Improv (no button press required)
3. ESPHome Dashboard adoption adds encryption/OTA password
4. Updates: Users manually re-flash via web installer for project updates

## Technical Notes

- BLE Improv uses `authorizer: none` for frictionless setup
- Production device uses `@latest` tag for GitHub package import
- ESPHome Dashboard manages production device updates

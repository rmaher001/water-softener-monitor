# Technical Documentation

ESPHome-based water softener salt level monitoring system using M5Stack ATOM hardware and VL53L0X ToF distance sensor.

## Hardware Variants

### ATOM Lite (ESP32-PICO-D4)
- ESP-IDF framework
- Web server enabled (http://water-softener-monitor-lite.local or -lite-dev.local for dev)
- I2C: SDA=GPIO26, SCL=GPIO32
- Button: GPIO39 (no internal pullup)
- RGB LED: GPIO27 (WS2812B)

### ATOM S3 (ESP32-S3)
- Arduino framework (ESP-IDF has persistence issues on S3)
- Web server disabled (socket exhaustion)
- I2C: SDA=GPIO2, SCL=GPIO1
- Button: GPIO41 (internal pullup)
- RGB LED: GPIO35 (SK6812)
- Configuration via Home Assistant only

## Project Structure

**Core:**
- `src/water-softener-lite-core.yaml` - ATOM Lite functionality
- `src/water-softener-s3-core.yaml` - ATOM S3 functionality

**Web Installer:**
- `src/water-softener-lite-webinstall.yaml`
- `src/water-softener-s3-webinstall.yaml`

**Development:**
- `src/water-softener-lite-dev.yaml` - ATOM Lite (192.168.86.32)
- `src/water-softener-s3-dev.yaml` - ATOM S3 (192.168.86.104)

**Dev Config Strategy (v1.4.0+):**
- Dev configs use GitHub package refs (not `!include`) to mirror production workflow
- Enables testing ESPHome Dashboard adoption and Home Assistant update mechanisms
- Device names include hardware suffix to avoid mDNS conflicts during testing
- For fast iteration on sensor logic, temporarily switch to `!include` in dev configs

## Core Logic

### Sensor Update Strategy
Distance sensor uses `update_interval: never` and is manually triggered by interval component. Enables dynamic intervals via `update_interval_seconds` parameter without reflashing.

### Out-of-Range Handling
Maintains last valid reading when sensor out of range (< 3cm or > 150cm):
- `last_valid_percentage` - Persisted across reboots
- `last_valid_status` - Persisted across reboots
- `sensor_out_of_range` binary sensor - Indicates stale readings

### Lambda Functions
1. Salt level calculation - Validates distance, converts to percentage
2. Out of range detection - Checks sensor validity
3. Status text generation - Maps percentage to status strings
4. Update interval control - Custom timing with globals and millis()

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

## Regeneration Detection (1.5.0+)

### Binary Sensor: "Regeneration Cycle Active"

**Automatic detection of water softener regeneration cycles using salt level patterns.**

**Detection Logic:**
- **START**: >3% drop in 5 minutes OR >5% drop in 10 minutes
- **END**: 50% recovery from drop + stability (±2% for 30 minutes)
- **Safety timeout**: 4 hours maximum
- **False alarm reset**: Auto-clear if <5% total drop after 1 hour

**Metrics Available in Home Assistant:**
- Binary sensor state (ON/OFF) with timestamps
- Users can compute additional metrics using Home Assistant template sensors:
  - Baseline level: Capture `salt_level_percent` when cycle starts
  - Drop amount: Track min/max of `salt_level_percent` during cycle
  - Duration: Calculate from `last_changed` timestamp

**Status Sensor Behavior:**
- "Salt Status" text sensor freezes during regeneration to prevent false alerts
- Status changes respond immediately (no confirmation delay as of v1.6.0)

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

## Manual Refill Confirmation & Status Hysteresis (1.6.0+)

### Problem: Water Level Fluctuations vs. Actual Salt Changes

**Background:**
- Brine tanks maintain standing water that fluctuates ±5cm around regeneration cycles
- ToF sensor measures distance to water surface, not salt level
- Water level changes after regeneration make status appear to improve (e.g., "Moderate" → "Good")
- This is incorrect - salt was consumed during regeneration, not added

**Example:**
```
Before regen: Distance 47cm, Status "Moderate" (45%)
After regen:  Distance 41cm, Status "Good" (52%) ← WRONG!
Reality:      Salt was consumed, water level changed
```

### Solution: Manual Refill Confirmation Button

**Hardware Button (ATOM Lite only):**
- Hold button on ATOM Lite for 3-5 seconds to confirm salt refill
- LED flashes 3 times during hold, then solid for 2 seconds to confirm
- Records refill timestamp in device memory (persisted across reboots)

**Status Hysteresis Rule:**
- Status can **ONLY improve** (Critical→Low, Low→Good, Good→Full) if refill button was pressed in last 24 hours
- Status can always **degrade** (Good→Low, Low→Critical) without button press
- Prevents false status improvements from water level fluctuations

**Behavior:**
```
Day 1: Status = "Moderate" (45%)
Day 2: Regen runs, water settles, reading shows 52%
       Status: STAYS "Moderate" (can't improve without refill)
Day 3: Reading drops to 38%
       Status: Changes to "Low" (degradation always allowed)
Day 4: User refills salt, holds button 3-5 seconds
       LED flashes, confirms, reading shows 75%
       Status: Changes to "Good" (improvement allowed after refill)
```

**Benefits:**
- Eliminates false status improvements from water noise
- User has definitive control over status changes
- Salt level trend always moves downward between refills (as expected)
- Simple physical confirmation - no network or HA required

**LED Feedback:**
- Button press detected → LED flashes 3 times (600ms total)
- Refill confirmed → LED solid for 2 seconds
- No LED feedback → Button press too short or too long (must be 3-5 seconds)

**Logs (ESPHome):**
```
[refill] Manual refill confirmed - distance: 41.2cm, level: 75%
```

**Technical Details:**
- Refill timestamp stored in NVS (persisted across reboots and power loss)
- Hysteresis check happens before any status change
- 24-hour window resets on each button press
- Works entirely on-device, no Home Assistant connection required

**Hardware Support:**
- **ATOM Lite**: Button on GPIO39, RGB LED on GPIO27 (WS2812B)
- **ATOM S3**: Button on GPIO41, RGB LED on GPIO35 (SK6812)

## Measurement Accuracy & Lid Positioning

### Physical Constraints

**Hardware Layout:**
- ToF sensor mounted **2 inches off-center** in round tank lid
- Water surface not perfectly level across tank (±3 cm variation)
- Lid rotation changes which part of water surface is measured
- Tight lid fit prevents accidental rotation but allows manual adjustment

**Rotational Sensitivity:**
- Rotating lid across full range: **10% variance** (53-63% salt level)
- Distance variation from rotation: **±6 cm** from center to edge
- Critical issue: Reopening lid without alignment causes false readings

### Lid Alignment Procedure

**Required for Consistent Readings:**
1. Mark both lid and tank rim at baseline position
2. Baseline position: **41.5-41.8 cm distance** / **64-65% salt level**
3. After opening lid (inspection/refill), align marks before closing
4. Verify reading returns to expected range after realignment

**Calibration Process:**
1. Press calibration button for fast polling (100ms updates)
2. Slowly rotate lid while monitoring readings
3. Stop at baseline distance (41.5-41.8 cm)
4. Mark lid edge and tank rim with permanent marker
5. Document: "Lid must align with marks - misalignment causes ±5% error"

### Measurement Error Budget

**Normal Operating Conditions (with proper lid alignment):**
- Sensor precision: ±0.24 cm std dev (excellent)
- Peak-to-peak variation: 1.26 cm over 5 days
- Coefficient of variation: 0.59% (stable)
- **Realistic accuracy: ±1-2%** from water surface fluctuations

**Implications:**
- Day-to-day variations <3% are measurement noise, not real changes
- Salt consumption (~2% per cycle) at edge of measurement resolution
- Regeneration cycle drops (15-20%) easily detected above noise floor
- Long-term trends reliable, single-cycle tracking unreliable

### Salt Consumption Data

**Confirmed from Regeneration Cycles:**
- Actual salt consumption: **~2% per regeneration cycle**
- First cycle observation: 2% drop (real consumption)
- Second cycle observation: 6% rise (lid misalignment, not real)

**Current Status (as of Nov 2, 2025):**
- Lid marked and positioned at baseline (41.8 cm / 64.8%)
- Collecting clean baseline data with proper lid alignment
- **Need 3-5 regeneration cycles (3-5 weeks)** to confirm consumption pattern
- Data required before implementing v1.7.0 malfunction detection

**System Purpose:**
- **Primary goal**: Alert when salt is low (prevent running out)
- Regeneration detection: Confirms system operation
- Consumption tracking: Unreliable due to ±1-2% noise vs 2% consumption
- Manual refill button: Prevents false status improvements

## Configuration Parameters

**Measurement:**
- Tank Height (30-150cm, default 100cm)
- Full Level Distance (5-50cm, default 20cm)
- Update Interval (1-86400s, default 300s / 5 minutes)

**Thresholds:**
- Critical: 20%, Low: 40%, Good: 60%, Full: 80%

## Version Management

**Version String Format**: `X.Y.Z-variant` (e.g., `1.5.0-s3`, `1.5.0-lite`)

**How Versioning Works:**
1. Version defined in webinstall configs (`src/water-softener-{s3|lite}-webinstall.yaml`):
   ```yaml
   substitutions:
     version: "1.5.0-s3"  # or "1.5.0-lite"
   ```

2. Version passed to core configs via `${version}` substitution

3. Exposed in two places:
   - **Home Assistant**: `text_sensor.water_softener_firmware_version`
   - **Device Info**: Settings → Devices → Water Softener → "Project Version"

**Verifying Active Version:**
- Check `text_sensor.water_softener_firmware_version` in Home Assistant
- Or check device info in HA: Settings → Devices → Water Softener Monitor
- Or look for version-specific features (e.g., 1.5.0 has `binary_sensor.water_softener_regeneration_cycle_active`)

**Important**: Git tags use NO "v" prefix (use `1.5.0`, NOT `v1.5.0`)

## Development Commands

```bash
# ATOM Lite
esphome compile src/water-softener-lite-dev.yaml
esphome upload src/water-softener-lite-dev.yaml --device 192.168.86.32
esphome logs src/water-softener-lite-dev.yaml --device 192.168.86.32

# ATOM S3
esphome compile src/water-softener-s3-dev.yaml
esphome upload src/water-softener-s3-dev.yaml --device 192.168.86.104
esphome logs src/water-softener-s3-dev.yaml --device 192.168.86.104
```

## Release Workflow

1. Update version in webinstall configs (e.g., `1.3.1-s3`, `1.3.1-lite`)
2. Commit changes
3. Create and push git tag (NO "v" prefix): `git tag -a 1.5.0 -m "Release 1.5.0"`
   - **IMPORTANT**: Use `1.5.0` format, NOT `v1.5.0` (this project does not use "v" prefix)
4. Update `dashboard_import` URLs to reference new tag
5. Compile both firmware variants
6. Copy to docs/ and update manifest files
7. Commit, push, and test ESPHome Dashboard adoption

## Deployment Model

1. Web installer firmware ships with no encryption
2. User configures WiFi via BLE Improv (no button press required)
3. ESPHome Dashboard adoption adds encryption/OTA password
4. Updates: Users manually re-flash via web installer for project updates

## Technical Notes

- BLE Improv uses `authorizer: none` for frictionless setup
- Version tags in `dashboard_import` ensure reproducible builds
- Web installer is standard deployment path (not ESPHome Dashboard)
- See DEVELOPMENT.md for detailed workflow documentation

## ESPHome Update Notifications

**Observed Behavior (ESPHome 2025.10.x):**

Home Assistant's update list for ESPHome devices has limitations that can cause devices to not appear:

**Key Findings:**
- Update list shows only **online/discoverable** devices that need updates
- Offline or undiscoverable devices are excluded (even if they need updates)
- No fixed limit on number of devices - varies based on connectivity
- Observed: 8 devices initially → 10 devices after powering on 2 monitors

**Related Issue:**
- GitHub issue #6775: Reports 26 devices in ESPHome but only 7 shown in HA update list
- Suggests network/discovery issues rather than hard limits

**Recommended Update Workflow:**
1. **ESPHome Dashboard "Update All"** - Most reliable, updates all devices in dashboard
2. **Home Assistant update list** - Only shows reachable devices needing updates

**For Users:**
- Devices must be online/reachable to appear in HA update notifications
- ESPHome Dashboard adoption recommended for consistent update visibility
- Offline devices won't show update prompts even if they need platform updates

# Technical Documentation

ESPHome-based water softener salt level monitoring system using M5Stack ATOM hardware and VL53L0X ToF distance sensor.

## Hardware Variants

### ATOM Lite (ESP32-PICO-D4)
- ESP-IDF framework
- Web server enabled (http://water-softener-monitor.local)
- I2C: SDA=GPIO26, SCL=GPIO32
- Button: GPIO39 (no internal pullup)

### ATOM S3 (ESP32-S3)
- Arduino framework (ESP-IDF has persistence issues on S3)
- Web server disabled (socket exhaustion)
- I2C: SDA=GPIO2, SCL=GPIO1
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

## Configuration Parameters

**Measurement:**
- Tank Height (30-150cm, default 100cm)
- Full Level Distance (5-50cm, default 20cm)
- Update Interval (1-86400s, default 20s)

**Thresholds:**
- Critical: 20%, Low: 40%, Good: 60%, Full: 80%

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
3. Create and push git tag: `git tag -a 1.3.1 -m "Release v1.3.1"
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

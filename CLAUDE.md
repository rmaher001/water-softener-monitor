# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an ESPHome-based water softener salt level monitoring system running on M5Stack ATOM hardware (S3 or Lite) with a VL53L0X Time-of-Flight distance sensor. The device measures the distance to the salt level in a water softener tank and calculates percentage full, with configurable thresholds for different alert levels.

## Hardware Configuration

**VERSION 1.3.0+ supports two hardware variants:**

### ATOM S3 Lite (ESP32-S3)
- **Board**: M5Stack ATOM S3 Lite
- **Chip**: ESP32-S3
- **Framework**: Arduino (ESP-IDF has persistence issues on S3)
- **Sensor**: VL53L0X ToF distance sensor via I2C (Grove port)
  - SDA: GPIO2
  - SCL: GPIO1
- **Web Server**: Disabled (Arduino framework socket exhaustion)
- **Configuration**: Via Home Assistant only

### ATOM Lite (ESP32-PICO-D4) - RECOMMENDED
- **Board**: M5Stack ATOM Lite
- **Chip**: ESP32-PICO-D4
- **Framework**: ESP-IDF (works perfectly with persistence and web server)
- **Sensor**: VL53L0X ToF distance sensor via I2C (Grove port)
  - SDA: GPIO26
  - SCL: GPIO32
- **Button**: GPIO39 (no internal pullup)
- **Web Server**: Enabled (http://water-softener-monitor.local)
- **Configuration**: Via Home Assistant OR web server
- **Advantages**: Cheaper than S3, standalone web configuration, same WiFi performance

## Key Architecture

### Configuration Files

**Core Packages (hardware-specific):**
- `src/water-softener-s3-core.yaml` - ATOM S3 core functionality (Arduino framework)
- `src/water-softener-lite-core.yaml` - ATOM Lite core functionality (ESP-IDF framework, web server enabled)

**Web Installer Configs:**
- `src/water-softener-s3-webinstall.yaml` - ATOM S3 web installer (no encryption, uses Improv BLE)
- `src/water-softener-lite-webinstall.yaml` - ATOM Lite web installer (no encryption, uses Improv BLE)

**Development/Testing:**
- `src/water-softener-dev.yaml` - Development configuration for local testing (ATOM S3)
- `src/water-softener-atom-lite.yaml` - Development configuration for ATOM Lite testing

**Archived (see archive/README.md):**
- Multi-device configurations removed in v1.3.0 (simplified for single-device use case)

### Core Logic Flow

1. **Interval-based Updates**: A 1-second interval checks if it's time to update based on the configurable `update_interval_seconds` parameter
2. **Distance Measurement**: The VL53L0X sensor measures distance to salt surface
3. **Percentage Calculation**: Lambda function in `salt_level_percent` sensor converts distance to percentage using:
   - Tank height (configurable)
   - Full level distance (configurable baseline)
   - Formula: `(tank_height - current_distance) / (tank_height - full_level) * 100`
4. **Status Determination**: Text sensor evaluates percentage against thresholds to display status

### Configurable Parameters

Parameters are organized into two groups using ESPHome's web_server sorting groups:

**Measurement Settings Group:**
- `Tank Height`: Total internal tank height (30-200cm, default 84cm)
- `Full Level Distance`: Distance from sensor to "full" salt level (5-50cm, default 15cm)
- `Update Interval`: How often to poll the sensor (1-300s, default 30s)
- `Reset to Defaults`: Button to restore all parameters to initial values

**Alert Thresholds Group:**
- `Critical Alert Threshold`: Minimum percentage for "Critical" alert (0-100%, default 10%)
- `Full Threshold`: Minimum percentage for "Full" status (0-100%, default 75%)
- `Good Threshold`: Minimum percentage for "Good" status (0-100%, default 50%)
- `Low Alert Threshold`: Minimum percentage for "Low" alert (0-100%, default 25%)

All thresholds support the full 0-100% range for maximum flexibility.

### Out-of-Range Handling

The system maintains the last valid reading when the sensor is out of range:

**Global Variables:**
- `last_valid_percentage`: Stores last good percentage reading (restored on reboot)
- `last_valid_status`: Stores last good status text (restored on reboot)
- `last_update_time`: Tracks update timing (not restored)

**Binary Sensor:**
- `Sensor Out of Range`: Indicates if current reading is valid or stale
  - `true` when: distance < 3cm OR distance > 150cm OR reading is NaN
  - `false` when: sensor reading is valid (3-150cm range)

**Behavior:**
- When in range: Calculate and store new values, update percentage and status
- When out of range: Return stored values, set binary sensor to true

### Lambda Functions

The project uses four lambda functions:
1. **Salt level calculation** (salt_level_percent sensor): Validates distance, converts to percentage, stores last valid value
2. **Out of range detection** (sensor_out_of_range binary sensor): Checks if current reading is valid
3. **Status text generation** (salt_status_text sensor): Maps percentage to status strings, stores last valid status
4. **Update interval control** (interval component): Custom timing logic using globals and millis()

### Calibration Mode

**Calibration Mode** (`calibration_mode` switch) enables fast sensor polling for device setup and testing:
- Turns on: Distance sensor updates every 100ms (instead of configured interval)
- Auto-off: Automatically disables after 5 minutes to prevent excessive power use
- Use for: Measuring initial distance values, troubleshooting sensor readings, testing threshold adjustments
- Via UI: Toggle the "Calibration Mode" switch in Home Assistant or web interface

### BLE Improv Configuration (v1.2.20+)

**Finding**: The original `authorizer: setup_button` requirement created friction during web installer setup - users had to press the physical button to authorize WiFi configuration via Bluetooth.

**Solution**: Changed to `authorizer: none` for seamless web installer experience while maintaining security:
- BLE WiFi setup works immediately without button presses
- Improv Serial provides alternative setup path if needed
- WiFi hotspot fallback available ("Water Softener Hotspot")
- Security enforced via ESPHome Dashboard adoption (adds encryption/passwords)
- Tested and verified: BLE Improv works, Dashboard adoption successful

**Known Issue - Stuck Discovery Notifications After Device Deletion**:

If you delete a device from Home Assistant and then try to re-add it via BLE Improv, the discovery notification in the Home Assistant web interface may become stuck and won't dismiss. Clicking "Add" shows a manual IP address entry dialog instead of the BLE Improv WiFi configuration flow. This is a Home Assistant caching issue, not a firmware problem.

**Workaround - Use Mobile App**:
1. Use the **Home Assistant mobile app** (iOS/Android) instead of the web interface
2. In the mobile app: **Settings → Devices & Services → Add Integration → ESPHome**
3. The app will detect the device via Bluetooth and show BLE Improv setup
4. Enter WiFi credentials when prompted (no button press required thanks to `authorizer: none`)
5. Complete the WiFi setup through the mobile app interface

**Alternative**: Restart Home Assistant (Settings → System → Restart) to clear the stuck notification cache, then try adding the device again via the web interface.

## Common Commands

**ESPHome CLI Path**: `~/esphome/venv/bin/esphome` (or create an alias for faster access)

### For Development (using dev config)
```bash
# Validate Configuration
~/esphome/venv/bin/esphome config src/water-softener-dev.yaml

# Compile Firmware
~/esphome/venv/bin/esphome compile src/water-softener-dev.yaml

# Upload via OTA (to dev device at 192.168.86.104)
~/esphome/venv/bin/esphome upload src/water-softener-dev.yaml --device 192.168.86.104

# View Logs
~/esphome/venv/bin/esphome logs src/water-softener-dev.yaml --device 192.168.86.104

# Run Full Pipeline (compile + upload + logs)
~/esphome/venv/bin/esphome run src/water-softener-dev.yaml --device 192.168.86.104

# Flash via USB (if OTA fails)
~/esphome/venv/bin/esphome run src/water-softener-dev.yaml --device /dev/tty.usbmodem*
```

### For Web Installer Build
```bash
# Compile ATOM S3 Web Installer
~/esphome/venv/bin/esphome compile src/water-softener-s3-webinstall.yaml

# Compile ATOM Lite Web Installer
~/esphome/venv/bin/esphome compile src/water-softener-lite-webinstall.yaml

# Copy firmware to docs directory for web installer release
cp src/.esphome/build/water-softener-monitor/.pioenvs/water-softener-monitor/firmware.factory.bin docs/firmware-s3.factory.bin   # for S3
cp src/.esphome/build/water-softener-monitor/.pioenvs/water-softener-monitor/firmware.factory.bin docs/firmware-lite.factory.bin # for Lite
```

## WiFi Configuration

Requires a `secrets.yaml` file in the ESPHome config directory with:
```yaml
wifi_ssid: "YourSSID"
wifi_password: "YourPassword"
```

Fallback AP: "Softener-Fallback" / "12345678"

## Integrations

- **Home Assistant**: API enabled for native integration (no encryption in webinstaller, added during ESPHome adoption)
- **OTA Updates**: Enabled for wireless firmware updates (no password in webinstaller, added during ESPHome adoption)
- **Improv BLE/Serial**: Enabled for easy WiFi configuration

**Web Server:**
- **ATOM S3**: Web server **disabled** to prevent socket exhaustion issues with Arduino framework. Configuration via Home Assistant only.
- **ATOM Lite**: Web server **enabled** (ESP-IDF framework supports it without issues). Configuration via Home Assistant OR web interface at http://water-softener-monitor.local

## Deployment Model

1. Web installer firmware has NO encryption/passwords for easy setup
2. User flashes device and configures WiFi via Improv BLE or hotspot
3. ESPHome Dashboard discovers and adopts the device
4. Adoption process adds API encryption and OTA password automatically
5. Device runs securely with user's own keys

## Sensor Update Strategy

The distance sensor uses `update_interval: never` and is manually triggered by the interval component. This allows for dynamic update intervals controlled by the `update_interval_seconds` parameter without recompiling firmware.

## Development Workflow

For comprehensive development guidance, see **DEVELOPMENT.md**, which covers:
- **Local Development**: Fast iteration using `!include` for instant testing on dev hardware (192.168.86.104)
- **Feature Branches**: Testing from GitHub branches before merging to `master`
- **Deployment**: Releasing changes to all users via GitHub package imports
- **Web Installer**: Building and releasing firmware binaries for the web installer
- **Recovery**: Procedures for unbricking devices or fixing flashing issues
- **Git Operations**: Branching, merging, and cleanup workflow

**Key Principle**: Feature changes go in the core config files (`src/water-softener-s3-core.yaml` or `src/water-softener-lite-core.yaml`). Changes that apply to both hardware variants should be duplicated in both files. Web installer configs are just entry points that reference these core packages.

## CRITICAL: Firmware Update Notifications

**THERE ARE NO AUTOMATIC UPDATE NOTIFICATIONS FOR CUSTOM PROJECT FIRMWARE VERSIONS.**

ESPHome update notifications are ONLY for ESPHome platform updates (e.g., ESPHome 2024.8.0 → 2024.9.0), NOT for custom project firmware updates (e.g., water-softener-monitor 1.0.0 → 1.1.0).

**The `dashboard_import` and `project` sections:**
- Are for device adoption and importing from web installers
- Do NOT provide update notifications for custom firmware versions
- Only help ESPHome Dashboard discover and adopt devices initially

**How users update to newer firmware versions:**
- Users MUST manually check ESPHome Dashboard
- Users MUST manually click "Install" to pull latest version from GitHub
- There is NO mechanism to notify users "water softener v1.1.0 is available"

**Version numbering (e.g., 1.1.0) is for:**
- Documentation and release tracking
- Manual identification by users
- NOT for automatic update detection

## Package Import Strategy and Updates

**VERSION 1.2.20+ USES VERSION TAGS FOR REPRODUCIBLE BUILDS**

### Configuration Approach

The web installer configs use **version tags** in `dashboard_import`:

```yaml
# water-softener-s3-webinstall.yaml (example)
dashboard_import:
  package_import_url: github://rmaher001/water-softener-monitor/src/water-softener-s3-webinstall.yaml@1.3.0
  import_full_config: false

packages:
  water_softener: !include water-softener-s3-core.yaml
```

**Why version tags?**
- Immutable - `@1.2.20` always points to the same git commit
- Reproducible - users get tested, known-good configurations
- No caching ambiguity - ESPHome knows exactly what to fetch
- Testable - we can verify the exact config users will get during adoption

**Why `!include` for packages?**
- Ensures core package is fetched alongside webinstall file
- Avoids separate cache layer for the core package
- Simplifies dependency chain

### How Updates Work

**ESPHome Platform Updates (automatic):**
- Home Assistant notifies users 2-3x/month about new ESPHome platform releases
- Users update ESPHome add-on in HA
- Users click "Update" on devices in HA or "Update All" in ESPHome Dashboard
- Devices get recompiled with new ESPHome platform version
- **Uses locally cached project config** - no project feature updates

**Project Updates (manual via web installer):**
- Project updates (v1.2.20 → v1.2.21) are NOT automatically notified
- Users must visit https://rmaher001.github.io/water-softener-monitor/
- Re-flash device via web installer to get new firmware
- Reconfigure WiFi via BLE Improv if needed
- Device now runs new project version

**This is the standard model used by Apollo Automation and other commercial ESPHome projects.**

### Release Workflow

When releasing a new version (e.g., v1.3.1):

1. Update version in `src/water-softener-s3-webinstall.yaml` and `src/water-softener-lite-webinstall.yaml`
   - ATOM S3: `version: "1.3.1-s3"`
   - ATOM Lite: `version: "1.3.1-lite"`
2. Commit changes to repository
3. Create git tag: `git tag -a 1.3.1 -m "Release v1.3.1: <description>"`
4. Push tag: `git push origin 1.3.1`
5. Update `dashboard_import` URL to reference new tag: `@1.3.1`
6. Compile web installer firmware binaries for both hardware variants
7. Copy firmware to docs/ directory:
   - `docs/firmware-s3.factory.bin`
   - `docs/firmware-lite.factory.bin`
8. Update manifest files with new version:
   - `docs/manifest-s3.json`
   - `docs/manifest-lite.json`
9. Commit and push to GitHub
10. Test adoption in ESPHome Dashboard with new tag reference

### Why Users Don't Get Project Update Notifications

ESPHome has a device registry at esphome.io for per-device update notifications, but:
- Most projects (including Apollo Automation) don't use it
- Requires maintaining registry entries and infrastructure
- Web installer re-flashing works well for major updates
- Users expect to update via web installer for firmware changes

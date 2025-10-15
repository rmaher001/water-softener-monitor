# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an ESPHome-based water softener salt level monitoring system running on an M5Stack ATOM S3 Lite with a VL53L0X Time-of-Flight distance sensor. The device measures the distance to the salt level in a water softener tank and calculates percentage full, with configurable thresholds for different alert levels.

## Hardware Configuration

- **Board**: M5Stack ATOM S3 (ESP32-S3)
- **Sensor**: VL53L0X ToF distance sensor via I2C (Grove port)
  - SDA: GPIO2
  - SCL: GPIO1
- **Framework**: Arduino

## Key Architecture

### Configuration Files

- `src/water-softener-core.yaml` - Core functionality with all sensors and logic
- `src/water-softener-webinstall-simple.yaml` - Web installer configuration for single devices (no encryption, uses Improv BLE)
- `src/water-softener-webinstall-multi.yaml` - Web installer configuration for multiple devices (adds MAC suffix)
- `src/water-softener-dev.yaml` - Development configuration for local testing

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

## Common Commands

### For Development (using dev config)
```bash
# Validate Configuration
esphome config src/water-softener-dev.yaml

# Compile Firmware
esphome compile src/water-softener-dev.yaml

# Upload via USB
esphome upload src/water-softener-dev.yaml --device /dev/ttyUSB0

# View Logs
esphome logs src/water-softener-dev.yaml

# Run Full Pipeline (compile + upload + logs)
esphome run src/water-softener-dev.yaml
```

### For Web Installer Build
```bash
# Compile Web Installer Firmware
esphome compile src/water-softener-webinstall.yaml
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

**Note**: Web server component is **disabled** to prevent socket exhaustion issues with Arduino framework. All configuration is done through Home Assistant UI.

## Deployment Model

1. Web installer firmware has NO encryption/passwords for easy setup
2. User flashes device and configures WiFi via Improv BLE or hotspot
3. ESPHome Dashboard discovers and adopts the device
4. Adoption process adds API encryption and OTA password automatically
5. Device runs securely with user's own keys

## Sensor Update Strategy

The distance sensor uses `update_interval: never` and is manually triggered by the interval component. This allows for dynamic update intervals controlled by the `update_interval_seconds` parameter without recompiling firmware.

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

## CRITICAL: Package Import Strategy to Avoid Caching Issues

**VERSION 1.2.18+ USES !include AND @main TO AVOID ESPHOME PACKAGE CACHING PROBLEMS**

### Why This Matters

ESPHome caches GitHub package imports using `@master` branch references. The `@master` cache becomes stale and doesn't refresh when updates are pushed. However, `@main` references are handled differently and fetch fresh content reliably.

### The Solution

**1. Use `@main` in dashboard_import (not `@master`):**
```yaml
# water-softener-webinstall-simple.yaml
dashboard_import:
  package_import_url: github://rmaher001/water-softener-monitor/src/water-softener-webinstall-simple.yaml@main
```

**2. Use `!include` for the core package:**
```yaml
packages:
  water_softener: !include water-softener-core.yaml
```

### Why This Works

- `@main` references don't get cached the same way `@master` does in ESPHome
- Even if the repo's default branch is `master`, using `@main` in URLs works and bypasses cache
- `!include` ensures the core package file is fetched alongside the webinstall file (no separate cache layer)
- When ESPHome Dashboard adopts a device, the adopted config gets `@main`, which continues to work for updates

### How It Works

1. User flashes device via web installer
2. ESPHome Dashboard discovers device via `dashboard_import` with `@main`
3. Dashboard fetches webinstall-simple.yaml and water-softener-core.yaml from GitHub
4. Adopted config inherits `@main` reference which doesn't cache
5. Updates work when user clicks "Install" - ESPHome fetches fresh from GitHub

### DO NOT Use These Approaches (They Cause Caching Issues)

```yaml
# WRONG - @master gets cached and becomes stale
dashboard_import:
  package_import_url: github://rmaher001/water-softener-monitor/src/water-softener-webinstall-simple.yaml@master
```

```yaml
# WRONG - Direct package import with @master also caches
packages:
  water_softener: github://rmaher001/water-softener-monitor/src/water-softener-core.yaml@master
```

```yaml
# WRONG - Mapping syntax with ref: master caches indefinitely
packages:
  water_softener:
    url: https://github.com/rmaher001/water-softener-monitor
    files: [src/water-softener-core.yaml]
    ref: master
```

### Adopted Device Configuration

After adoption, ESPHome Dashboard creates a config like:
```yaml
packages:
  rmaher001.water-softener-monitor: github://rmaher001/water-softener-monitor/src/water-softener-webinstall-simple.yaml@main
```

The `@main` reference ensures ESPHome fetches fresh content from GitHub on each compile, avoiding the caching issues that affect `@master`.

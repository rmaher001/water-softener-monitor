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

- `src/water-softener-package.yaml` - Core functionality with all sensors and logic
- `src/water-softener-webinstall.yaml` - Web installer configuration (no encryption, uses Improv BLE)
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
- **Web Server**: Available on port 80 for direct browser access
- **OTA Updates**: Enabled for wireless firmware updates (no password in webinstaller, added during ESPHome adoption)
- **Improv BLE/Serial**: Enabled for easy WiFi configuration

## Deployment Model

Follows the Apollo MSR-2 pattern:
1. Web installer firmware has NO encryption/passwords
2. User flashes device and configures WiFi via Improv BLE or hotspot
3. ESPHome Dashboard discovers and adopts the device
4. Adoption process adds API encryption and OTA password automatically
5. Device runs securely with user's own keys

## Sensor Update Strategy

The distance sensor uses `update_interval: never` and is manually triggered by the interval component. This allows for dynamic update intervals controlled by the `update_interval_seconds` parameter without recompiling firmware.

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

### Main Configuration File

All configuration is in `src/water-softener.yaml` - this is a single-file ESPHome YAML configuration with no custom components.

### Core Logic Flow

1. **Interval-based Updates**: A 1-second interval checks if it's time to update based on the configurable `update_interval_seconds` parameter
2. **Distance Measurement**: The VL53L0X sensor measures distance to salt surface
3. **Percentage Calculation**: Lambda function in `salt_level_percent` sensor converts distance to percentage using:
   - Tank height (configurable)
   - Full level distance (configurable baseline)
   - Formula: `(tank_height - current_distance) / (tank_height - full_level) * 100`
4. **Status Determination**: Text sensor evaluates percentage against thresholds to display status

### Configurable Parameters (number entities)

- `update_interval_seconds`: How often to poll the sensor (1-300s, default 30s)
- `tank_height_cm`: Total internal tank height (30-200cm, default 84cm)
- `full_level_cm`: Distance from sensor to "full" salt level (5-50cm, default 15cm)
- `full_threshold`: Minimum percentage for "Full" status (50-95%, default 75%)
- `good_threshold`: Minimum percentage for "Good" status (30-90%, default 50%)
- `low_threshold`: Minimum percentage for "Low" alert (10-50%, default 25%)
- `critical_threshold`: Minimum percentage for "Critical" alert (5-25%, default 10%)

### Lambda Functions

The project uses three lambda functions:
1. **Salt level calculation** (salt_level_percent sensor): Converts raw distance to percentage
2. **Status text generation** (salt_status_text sensor): Maps percentage to status strings
3. **Update interval control** (interval component): Custom timing logic using globals and millis()

## Common Commands

### Validate Configuration
```bash
esphome config src/water-softener.yaml
```

### Compile Firmware
```bash
esphome compile src/water-softener.yaml
```

### Upload to Device (OTA)
```bash
esphome upload src/water-softener.yaml
```

### Upload via USB
```bash
esphome upload src/water-softener.yaml --device /dev/ttyUSB0
```

### View Logs
```bash
esphome logs src/water-softener.yaml
```

### Run Full Pipeline (compile + upload + logs)
```bash
esphome run src/water-softener.yaml
```

## WiFi Configuration

Requires a `secrets.yaml` file in the ESPHome config directory with:
```yaml
wifi_ssid: "YourSSID"
wifi_password: "YourPassword"
```

Fallback AP: "Softener-Fallback" / "12345678"

## Integrations

- **Home Assistant**: API enabled for native integration
- **Web Server**: Available on port 80 for direct browser access
- **OTA Updates**: Enabled for wireless firmware updates

## Sensor Update Strategy

The distance sensor uses `update_interval: never` and is manually triggered by the interval component. This allows for dynamic update intervals controlled by the `update_interval_seconds` parameter without recompiling firmware.

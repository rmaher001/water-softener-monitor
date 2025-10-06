# Water Softener Salt Monitor

ESPHome-based salt level monitor for water softener brine tanks using a VL53L0X ToF distance sensor.

## Hardware

- M5Stack ATOM S3 Lite (ESP32-S3)
- VL53L0X ToF sensor (0-200cm range)
- 10cm Grove cable (sensor to ATOM)
- Long USB-C cable for power

## Physical Installation

1. **Sensor mounting**: Attach VL53L0X to tank lid using M2.5 nylon screws and nuts through sensor mounting holes
2. **Cable routing**: Drill 8-10mm hole in lid, install rubber grommet, route Grove cable through
3. **ATOM mounting**: Attach ATOM S3 to top of lid using 3M Velcro strips
4. **Power**: Connect long USB-C cable to ATOM (disconnect when refilling tank)

## Installation

### For End Users (Recommended - Auto-updates from GitHub)

**Your ESPHome config directory is:**
- Home Assistant add-on: `/config/esphome/`
- Standalone ESPHome: Your working directory where you run `esphome` commands

1. **Create or update `secrets.yaml`** in your ESPHome config directory (skip if you already have WiFi credentials):
   ```yaml
   # secrets.yaml
   wifi_ssid: "YourWiFiSSID"
   wifi_password: "YourWiFiPassword"
   ```

2. **Copy `water-softener.yaml`** from this repo to your ESPHome config directory and customize if needed:
   ```yaml
   substitutions:
     device_name: "water-softener"
     friendly_name: "Water Softener Monitor"
     wifi_ssid: !secret wifi_ssid
     wifi_password: !secret wifi_password

   packages:
     water_softener: github://rmaher001/water-softener-monitor/src/water-softener-package.yaml@master
   ```

3. **Flash via USB first time**:
   ```bash
   esphome run water-softener.yaml --device /dev/ttyUSB0
   ```

4. **Future updates**: Device will show in Home Assistant ESPHome dashboard. Click "Update" to pull latest version from GitHub and flash OTA.

### For Development

1. Clone this repository
2. Copy `secrets.yaml.example` to `secrets.yaml` and add WiFi credentials
3. Edit `src/water-softener.yaml` directly
4. Flash: `esphome run src/water-softener.yaml`

## Configuration

All parameters adjustable via web interface (no reflashing needed):

- **Tank Height**: Total internal tank height in cm
- **Full Level Distance**: Distance from sensor to "full" salt level
- **Update Interval**: How often to poll the sensor (1-300 seconds)
- **Thresholds**: Full, Good, Low, Critical alert levels (percentages)

## Status Levels

- **Full**: ≥75% (default)
- **Good**: ≥50% (default)
- **Low**: ≥25% (default)
- **Critical**: <10% (default)

## Integration

- Home Assistant API enabled
- Web interface on port 80
- OTA updates supported

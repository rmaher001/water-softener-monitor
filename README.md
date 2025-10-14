# Water Softener Salt Monitor

ESPHome-based salt level monitor for water softener brine tanks using a VL53L0X ToF distance sensor.

## Hardware

- **M5Stack ATOM S3 Lite** (ESP32-S3)
  - [Official M5Stack Store](https://shop.m5stack.com/products/atoms3-lite-esp32s3-dev-kit)
  - Also available at Mouser, DigiKey, Adafruit
- **M5Stack ToF Sensor Unit** (VL53L0X, 0-200cm range)
  - [Official M5Stack Store](https://shop.m5stack.com/products/tof-sensor-unit)
  - Also available at Mouser, DigiKey, Adafruit
  - Includes 20cm Grove cable
- **Long USB-C cable** for power

## Physical Installation

1. **Sensor mounting**: Attach M5Stack ToF sensor unit to underside of tank lid using 3M Velcro or double-sided foam tape
2. **Cable routing**: Drill 8-10mm hole in lid, install rubber grommet, route Grove cable through
3. **ATOM mounting**: Attach ATOM S3 to top of tank lid using 3M Velcro strips
4. **Power**: Connect long USB-C cable to ATOM (disconnect when refilling tank)

**Note**: The M5Stack ToF sensor comes in a plastic case - Velcro/foam tape mounting works best and avoids drilling into the unit.

## Installation

### Quick Start - Web Installer (Recommended!)

Flash firmware directly from your browser - no software installation required:

**👉 [Open Web Installer](https://rmaher001.github.io/water-softener-monitor/)**

**Two Installation Options:**

1. **Simple Install** (Recommended for most users)
   - For users with **ONE** water softener
   - Clean entity IDs without MAC suffix
   - Example: `sensor.water_softener_monitor_salt_level`
   - No renaming needed!

2. **Multi-Device Install**
   - For users with **MULTIPLE** water softeners
   - Unique entity IDs with MAC suffix
   - Example: `sensor.water_softener_monitor_954910_salt_level`
   - Prevents conflicts between multiple devices

**Setup Process:**
1. Connect ATOM S3 via USB-C
2. Choose your installation type (Simple or Multi-Device)
3. Flash the device using the web installer
4. Configure WiFi using Home Assistant app (Bluetooth) or connect to "Water Softener Hotspot"
5. Device auto-discovers in Home Assistant - works immediately!

---

### Optional: ESPHome Dashboard Management

If you want to manage your device from ESPHome Dashboard or customize entity IDs:

1. **Adopt in ESPHome Dashboard** - Device appears, click "Adopt"
2. **For Clean Entity IDs** (if you used Multi-Device install):
   - Delete device from Home Assistant first
   - Edit in ESPHome Dashboard, remove MAC suffix
   - Reinstall and re-add to Home Assistant
   - See detailed instructions in our setup guide

---

### Manual Installation (Advanced Users)

**For Development/Testing:**

1. Clone this repository
2. Update `/Users/yourusername/esphome/secrets.yaml` with your WiFi credentials:
   ```yaml
   wifi_ssid: "YourWiFiSSID"
   wifi_password: "YourWiFiPassword"
   ```
3. Flash the development config:
   ```bash
   esphome run src/water-softener-dev.yaml --device /dev/ttyUSB0
   ```

**Note**: The webinstaller approach is recommended for most users as it handles encryption and configuration automatically through ESPHome Dashboard adoption.

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

- **Home Assistant**: Auto-discovery with ESPHome integration (no API key required after adoption)
- **Web Interface**: Available on port 80 for direct browser access
- **OTA Updates**: Supported through ESPHome Dashboard (no password required after adoption)
- **Bluetooth**: Improv BLE for easy WiFi configuration

## Project Structure

- `src/water-softener-package.yaml` - Core functionality (sensors, thresholds, logic)
- `src/water-softener-webinstall-simple.yaml` - Simple web installer (no MAC suffix)
- `src/water-softener-webinstall-multi.yaml` - Multi-device web installer (with MAC suffix)
- `src/water-softener-dev.yaml` - Development/testing configuration
- `docs/` - Web installer files
  - `firmware-simple.factory.bin` - Single device firmware
  - `firmware-multi.factory.bin` - Multi-device firmware

## How It Works

1. **Initial Setup**: Device ships with no encryption (like commercial products)
2. **WiFi Configuration**: Uses Improv BLE or fallback hotspot for easy setup
3. **Adoption**: ESPHome Dashboard discovers and adopts the device, adding encryption
4. **Operation**: Runs securely with automatic updates from GitHub

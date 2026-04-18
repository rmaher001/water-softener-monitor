# Water Softener Salt Monitor

Monitor your water softener salt level from Home Assistant. A distance sensor measures the salt level in your brine tank and reports the percentage full. Use Home Assistant automations to send notifications when salt is low, so you never run out.

**[Web Installer](https://rmaher001.github.io/water-softener-monitor/)** | Uses M5Stack ATOM Lite or S3 hardware with VL53L0X ToF sensor

## Hardware

**Controller (choose one):**
- **M5Stack ATOM Lite** (ESP32-PICO-D4) - ESP-IDF framework, web server enabled
  - [Official M5Stack Store](https://shop.m5stack.com/products/atom-lite-esp32-development-kit)
  - Also available at Mouser, DigiKey, Adafruit
- **M5Stack ATOM S3 Lite** (ESP32-S3) - ESP-IDF framework, web server enabled
  - [Official M5Stack Store](https://shop.m5stack.com/products/atoms3-lite-esp32s3-dev-kit)
  - Also available at Mouser, DigiKey, Adafruit

**Sensor:**
- **M5Stack ToF Sensor Unit** (VL53L0X, 0-200cm range)
  - [Official M5Stack Store](https://shop.m5stack.com/products/tof-sensor-unit)
  - Also available at Mouser, DigiKey, Adafruit
  - Includes 20cm Grove cable

**Power:**
- Long USB-C cable for power

## Physical Installation

1. **Sensor mounting**: Attach M5Stack ToF sensor unit to underside of tank lid using 3M Velcro or double-sided foam tape
2. **Cable routing**: Drill 8-10mm hole in lid, install rubber grommet, route Grove cable through
3. **Controller mounting**: Attach ATOM to top of tank lid using 3M Velcro strips
4. **Power**: Connect long USB-C cable to ATOM (disconnect when refilling tank)

**Note**: The M5Stack ToF sensor comes in a plastic case - Velcro/foam tape mounting works best and avoids drilling into the unit.

## Installation

### Quick Start - Web Installer (Recommended!)

Flash firmware directly from your browser - no software installation required:

**👉 [Open Web Installer](https://rmaher001.github.io/water-softener-monitor/)**

**Setup Process:**
1. Connect your ATOM device via USB-C
2. Choose your hardware (ATOM Lite or ATOM S3)
3. Flash the firmware using the web installer
4. Configure WiFi using Home Assistant app
5. Device auto-discovers in Home Assistant

---

### Optional: ESPHome Dashboard Management

For advanced management and customization:

1. **Adopt in ESPHome Dashboard** - Device appears automatically, click "Adopt"
2. **Customize Configuration** - Edit YAML to add custom features
3. **OTA Updates** - Deploy changes wirelessly after adoption

**Note**: It appears that some of the devices may not receive update notifications in Home Assistant if they are not adopted in ESPHome Dashboard.

---

### Manual Installation (Advanced Users)

**For Development/Testing:**

1. Clone this repository
2. Update `/Users/yourusername/esphome/secrets.yaml` with your WiFi credentials:
   ```yaml
   wifi_ssid: "YourWiFiSSID"
   wifi_password: "YourWiFiPassword"
   ```
3. Flash the development config for your hardware:
   ```bash
   # For ATOM Lite
   esphome run src/water-softener-lite-dev.yaml --device /dev/ttyUSB0

   # For ATOM S3
   esphome run src/water-softener-s3-dev.yaml --device /dev/ttyUSB0
   ```

**Note**: The web installer approach is recommended for most users as it handles encryption and configuration automatically through ESPHome Dashboard adoption.

## Configuration

All parameters are adjustable in Home Assistant (no reflashing needed).

**Required — set these for your tank:**
- **Tank Height** — total internal tank height in cm (default 100)
- **Refill Threshold Distance** — distance at which "Refill" status triggers (default 43 cm)

**Optional — defaults work well, leave them alone unless you need to tune:**
- **Update Interval** — sensor poll rate in seconds (30–300, default 60)
- **Regen Step Threshold** — permanent distance jump that signals a regeneration cycle completed (default 2.0 cm)
- **Regen Confirmation Hours** — how long the step must persist before being confirmed as a regen (default 6 h)
- **Regen Overdue Days** — days without a regen before the "Regeneration Overdue" alert fires (default 20)

The regen-detection settings are optional — start with the defaults and only tune them if your softener's behavior differs (e.g. very small tanks, multiple cycles per day).

**Note**: Both ATOM Lite and S3 include a web interface at http://water-softener-monitor.local (with MAC suffix) for standalone configuration.

## Status

The salt status is reported via `text_sensor.water_softener_salt_status`:
- **Good** — measured distance is below the refill threshold (tank has enough salt)
- **Refill** — distance is at or above the refill threshold (time to add salt)

## Integration

- **Home Assistant**: Auto-discovery with ESPHome integration (no API key required after adoption)
- **Web Interface**: Available on both ATOM Lite and S3 (http://water-softener-monitor.local with MAC suffix)
- **OTA Updates**: Supported through ESPHome Dashboard (no password required after adoption)
- **Bluetooth**: Improv BLE for easy WiFi configuration

## Project Structure

**Core Packages:**
- `src/water-softener-lite-core.yaml` - ATOM Lite core functionality (ESP-IDF, web server)
- `src/water-softener-s3-core.yaml` - ATOM S3 core functionality (ESP-IDF, web server)

**Web Installer Configs:**
- `src/water-softener-lite-webinstall.yaml` - ATOM Lite web installer
- `src/water-softener-s3-webinstall.yaml` - ATOM S3 web installer

**Development Configs:**
- `src/water-softener-lite-dev.yaml` - ATOM Lite development/testing
- `src/water-softener-s3-dev.yaml` - ATOM S3 development/testing

**Web Installer Files:**
- `docs/firmware-lite.factory.bin` - ATOM Lite firmware binary
- `docs/firmware-s3.factory.bin` - ATOM S3 firmware binary
- `docs/manifest-lite.json` - ATOM Lite web installer manifest
- `docs/manifest-s3.json` - ATOM S3 web installer manifest

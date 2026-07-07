# Water Softener Salt Monitor

Monitor your water softener salt level from Home Assistant. A distance sensor on the tank lid measures the distance down to the salt surface — as salt is used up, the distance grows. It gives you two honest signals: **"add salt"** when the level gets low, and **"your softener may be broken"** when no salt has been consumed for too long (it caught a stuck brine float in the wild). No percentages, no predictions — just distance and status.

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

**The one setting you need:**
- **Refill Threshold Distance** — the distance (cm from the sensor) at which "Refill" status triggers. Pick it from a fresh fill: read **Distance to Salt** right after filling, then add the depletion you're comfortable with before an alert (e.g. fresh fill reads 30 cm → set threshold to 40–45 cm).

**Optional — defaults work well, leave them alone unless you need to tune:**
- **Tank Height** — total internal tank height in cm. Only used by the *Set Default Thresholds* button (which sets the refill threshold to 43% of tank height). Nothing else reads it, so you can ignore it and set the threshold directly.
- **Update Interval** — sensor poll rate in seconds (30–300, default 60)
- **Regen Step Threshold** — permanent distance jump that signals a regeneration cycle completed (default 2.0 cm)
- **Regen Confirmation Hours** — how long the step must persist before being confirmed as a regen (default 6 h)
- **Regen Overdue Days** — days without a regen before the "Regeneration Overdue" alert fires (default 20)

The regen-detection settings are optional — start with the defaults and only tune them if your softener's behavior differs (e.g. very small tanks, multiple cycles per day).

**Note**: Both ATOM Lite and S3 include a web interface at http://water-softener-monitor.local (with MAC suffix) for standalone configuration.

## Salt & Refilling

- **What salt:** slow-dissolve pellets work best.
- **How much:** about two-thirds full. Don't overfill — keep the salt **above the water line** (10–15 cm above is a good target).
- **When status shows Refill:** just add salt. The sensor sees the new, closer salt surface and the status returns to **Good** on its own — **no recalibration needed.** Only revisit **Refill Threshold Distance** if you change how full you normally fill the tank.

## What the entities mean

| Entity | Meaning |
|---|---|
| **Distance to Salt** | Raw distance (cm) from sensor to salt surface. Bigger = less salt. |
| **Salt Status** | `Good` (distance below the refill threshold) or `Refill` (at/above it — time to add salt) |
| **Last Regeneration** | Days since the softener last consumed salt |
| **Regeneration Overdue** | ON if no salt has been consumed in `Regen Overdue Days` (default 20) — your softener may not be working |
| **Sensor Out of Range** | ON if the sensor reads <5 cm or >120 cm — check mounting/moisture |

## How regeneration detection works

When a softener regenerates, it consumes salt and the distance takes a small permanent step up (typically 2–5 cm). The firmware watches for these steps. If none happen for `Regen Overdue Days`, the **Regeneration Overdue** alert turns on.

**Take that alert seriously.** A softener can appear to run normally — motor turning, water flowing — while consuming no salt at all (stuck brine float, clogged brine line, salt bridge). Overdue + salt level flat for weeks = call your water softener tech.

## Integration

- **Home Assistant**: Auto-discovery with ESPHome integration (no API key required after adoption)
- **Web Interface**: Available on both ATOM Lite and S3 (http://water-softener-monitor.local with MAC suffix)
- **OTA Updates**: Supported through ESPHome Dashboard (no password required after adoption)
- **Bluetooth**: Improv BLE for easy WiFi configuration

## Troubleshooting

- **Status stuck on Refill after adding salt** — the salt surface may be below the sensor's aim point, or your threshold is tighter than your fill level. Check Distance to Salt vs your threshold.
- **Sensor Out of Range** — moisture or condensation on the sensor, or the lid was moved. Wipe the sensor and reseat the lid.
- **Regeneration Overdue but softener sounds fine** — a running softener can still consume no salt. Poke the salt with a broom handle (salt bridge?), and check whether the brine tank is actually being refilled with water after a regeneration. If in doubt, call your tech.
- **Reading jumped after opening the lid** — normal. Moving the lid shifts the sensor's reference point; it settles back once the lid is seated in its usual position.

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

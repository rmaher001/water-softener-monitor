# Water Softener Salt Monitor

ESPHome-based salt level monitor for water softener brine tanks using a VL53L0X ToF distance sensor.

## Hardware

- M5Stack ATOM S3 Lite (ESP32-S3)
- VL53L0X ToF sensor (0-200cm range)
- Grove cable connection

## Setup

1. Copy `secrets.yaml.example` to `secrets.yaml` and add your WiFi credentials
2. Install ESPHome: `pip install esphome`
3. Compile and upload: `esphome run src/water-softener.yaml`

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

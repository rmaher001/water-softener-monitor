# Development Workflow

This document describes the Git-based development workflow for the ESPHome Water Softener Monitor project.

## Repository Structure

```
src/
  ├── water-softener-package.yaml  # Main package - edit this for all changes
  ├── water-softener-dev.yaml      # Dev unit config (uses !include for local testing)
  └── water-softener.yaml          # Prod unit config (uses @master from GitHub)
water-softener.yaml                # User template (uses @master from GitHub)
```

## Hardware Setup

- **Dev Unit**: Current hardware at `192.168.86.62`
  - Device name: `water-softener-dev`
  - Config: `src/water-softener-dev.yaml`
  - Uses local `!include` for instant testing

- **Prod Unit**: Future/second hardware
  - Device name: `water-softener-prod`
  - Config: `src/water-softener.yaml`
  - Uses `@master` from GitHub (same as all users)

## Development Workflow

### 1. Local Development (Fast Iteration)

```bash
# Start feature/fix branch
git checkout -b feature/my-feature

# Edit the main package file
# vim src/water-softener-package.yaml

# Test on dev unit via command line (uses !include - instant)
~/esphome/venv/bin/esphome run src/water-softener-dev.yaml --device 192.168.86.62

# Iterate: edit, flash, test, repeat
```

### 2. Testing from GitHub Branch (Optional)

```bash
# Commit and push your branch
git add src/water-softener-package.yaml
git commit -m "Description of changes"
git push -u origin feature/my-feature

# Update HA ESPHome dashboard config to test from GitHub:
# packages:
#   water_softener: github://rmaher001/water-softener-monitor/src/water-softener-package.yaml@feature/my-feature

# Flash from HA dashboard to verify GitHub package works correctly
```

### 3. Release to Production

```bash
# Merge to master
git checkout master
git merge feature/my-feature
git push

# Revert HA dashboard config back to @master (if you changed it):
# packages:
#   water_softener: github://rmaher001/water-softener-monitor/src/water-softener-package.yaml@master

# All users (and your prod unit) automatically get the update via Home Assistant ESPHome dashboard
```

### 4. Cleanup

```bash
# Delete feature branch
git branch -d feature/my-feature
git push origin --delete feature/my-feature
```

## File Configurations

### Local Dev Config (`src/water-softener-dev.yaml`)

```yaml
substitutions:
  device_name: "water-softener-dev"
  friendly_name: "Water Softener Dev"
  wifi_ssid: !secret wifi_ssid
  wifi_password: !secret wifi_password

packages:
  water_softener: !include water-softener-package.yaml  # Local include for fast testing
```

### HA ESPHome Dashboard Config

```yaml
substitutions:
  device_name: "water-softener-dev"
  friendly_name: "Water Softener Dev"
  wifi_ssid: !secret wifi_ssid
  wifi_password: !secret wifi_password

packages:
  water_softener: github://rmaher001/water-softener-monitor/src/water-softener-package.yaml@master
  # Change to @feature/branch-name when testing branches
```

## Common Commands

### Flash via Command Line (Development)
```bash
# Compile only
~/esphome/venv/bin/esphome compile src/water-softener-dev.yaml

# Flash via OTA
~/esphome/venv/bin/esphome upload src/water-softener-dev.yaml --device 192.168.86.62

# Compile + Upload + Logs
~/esphome/venv/bin/esphome run src/water-softener-dev.yaml --device 192.168.86.62

# View logs only
~/esphome/venv/bin/esphome logs src/water-softener-dev.yaml --device 192.168.86.62
```

### Git Operations
```bash
# Create feature branch
git checkout -b feature/description

# Create fix branch
git checkout -b fix/issue-description

# View current branch
git branch

# Switch branches
git checkout master
git checkout feature/my-feature

# Merge and push
git checkout master
git merge feature/my-feature
git push
```

## Key Principles

1. **All changes go in `src/water-softener-package.yaml`** - This is the single source of truth
2. **Use feature/fix branches** - Never commit directly to master during development
3. **Test locally first** - Use `!include` for instant feedback
4. **Verify from GitHub** - Optionally test branch before merging to ensure package works remotely
5. **You're a user too** - Your prod unit gets updates the same way as all users (from `@master`)

## Home Assistant Integration

### Entity IDs
After flashing with clean discovery, entities are named:
- `sensor.water_softener_dev_salt_level`
- `sensor.water_softener_dev_distance_to_salt`
- `sensor.water_softener_dev_salt_status`
- `binary_sensor.water_softener_dev_sensor_out_of_range`
- `binary_sensor.water_softener_dev_calibration_active`
- `switch.water_softener_dev_calibration_mode`
- `button.water_softener_dev_reset_to_defaults`

### Deleting Device from HA
If you need to delete the device from Home Assistant for a clean start:
1. Go to **Settings** → **Devices & Services** → **ESPHome** (integration card, not device)
2. Find device in the ESPHome integration view
3. Click **three dots** → **Delete**
4. Reflash to rediscover with clean entity IDs

## Troubleshooting

### Device shows offline in ESPHome dashboard
- The ESPHome add-on dashboard and Home Assistant device discovery are separate
- Device may show in HA but not in ESPHome dashboard - this is normal if you haven't added the config to the dashboard
- Add it manually via "Empty Configuration" if needed

### Entity IDs don't match device name
- Delete device from Home Assistant completely
- Reflash to get fresh discovery with correct entity IDs
- Delete option is in ESPHome integration view, not individual device view

### Changes not appearing
- If using `!include`: Changes are instant (just recompile)
- If using GitHub reference: Changes only appear after pushing to that branch
- Clear `.esphome` cache if needed: `rm -rf src/.esphome`

## Recovery: Unbricking M5Stack ATOM S3

If the ATOM S3 gets bricked (USB connect/disconnect bootloop, won't flash), follow this recovery procedure:

### Symptoms of Bricked Device
- USB connects and disconnects repeatedly (~1 second cycle)
- Cannot flash new firmware
- No LED activity
- Device not recognized by computer

### Recovery Procedure

**Step 1: Enter Download Mode**

The key is holding the RST button long enough:

1. **Unplug** the USB cable from ATOM S3
2. **Press and hold the RST button** (side button)
3. **While holding RST**, plug in the USB cable
4. **Keep holding RST for 5-8 seconds** after plugging in (not just 2 seconds as labeled)
5. **Release** the RST button
6. The device should now stay connected without cycling

**Note:** The label says "Hold 2s → G0=0" but in practice you need to hold it longer (5-8 seconds) for reliable entry into download mode.

**Step 2: Erase Corrupted Firmware**

With the device in download mode and connected:

```bash
# Check device is connected
ls /dev/tty.* | grep usb

# Erase the corrupted flash
~/esphome/venv/bin/python -m esptool --chip esp32s3 --port /dev/tty.usbmodem* erase_flash
```

You should see:
```
Flash memory erased successfully in 2.1 seconds.
```

**Step 3: Flash Good Firmware**

```bash
# Flash the working dev firmware
~/esphome/venv/bin/esphome run src/water-softener-dev.yaml --device /dev/tty.usbmodem*
```

The device should flash successfully and boot normally. I2C errors about the VL53L0X sensor are normal if the sensor isn't connected.

### Prevention

- **Do NOT use web-based ESP flashers** - they can cause hard bricks on ATOM S3
- **Always use ESPHome CLI** or Home Assistant ESPHome dashboard for flashing
- **Test on dev hardware first** before flashing production units
- **Keep USB cable connected** throughout the entire flash process

### If Recovery Fails

If the RST button hold method doesn't work:

1. Try different USB cables (must be data-capable, not charge-only)
2. Try different USB ports or a powered USB hub
3. Try holding **both buttons** (RST + top button) while plugging in
4. As a last resort, physical GPIO0 to GND shorting may be required (advanced)

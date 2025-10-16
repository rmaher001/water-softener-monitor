# Development Workflow

Git-based development workflow for ESPHome Water Softener Monitor with dual hardware support.

## Hardware Setup

### ATOM S3 (Development)
- IP: `192.168.86.104`
- Config: `src/water-softener-s3-dev.yaml`
- Framework: Arduino
- Features: No web server (socket exhaustion)

### ATOM Lite (Development)
- IP: `192.168.86.32`
- Config: `src/water-softener-lite-dev.yaml`
- Framework: ESP-IDF
- Features: Web server enabled at http://water-softener-monitor-lite-dev.local

## Project Architecture

```
src/
  ├── water-softener-s3-core.yaml          # ATOM S3 core package
  ├── water-softener-lite-core.yaml        # ATOM Lite core package
  ├── water-softener-s3-webinstall.yaml    # ATOM S3 web installer
  ├── water-softener-lite-webinstall.yaml  # ATOM Lite web installer
  ├── water-softener-s3-dev.yaml           # ATOM S3 dev config
  └── water-softener-lite-dev.yaml         # ATOM Lite dev config

docs/
  ├── firmware-s3.factory.bin              # ATOM S3 web installer firmware
  ├── firmware-lite.factory.bin            # ATOM Lite web installer firmware
  ├── manifest-s3.json                     # ATOM S3 manifest
  └── manifest-lite.json                   # ATOM Lite manifest
```

**Key Principle**: All feature changes go in the core packages (`*-core.yaml`). Other configs are entry points that reference these cores.

## Development Workflow

### 1. Local Development (Testing Integration/Adoption)

**Dev configs use GitHub refs** to test the full adoption workflow that users experience:

```bash
# Create feature branch
git checkout -b feature/my-feature

# Edit core package for your hardware
vim src/water-softener-s3-core.yaml    # or water-softener-lite-core.yaml

# Commit and push to test from GitHub
git add src/
git commit -m "Description of changes"
git push -u origin feature/my-feature

# Test adoption workflow from GitHub
~/esphome/venv/bin/esphome run src/water-softener-s3-dev.yaml --device 192.168.86.104

# Or for ATOM Lite:
~/esphome/venv/bin/esphome run src/water-softener-lite-dev.yaml --device 192.168.86.32

# Test ESPHome Dashboard adoption and Home Assistant updates
```

**For fast sensor logic iteration**, temporarily use local files:
- Edit dev config to use `!include` instead of GitHub ref
- Test changes instantly without git operations
- Revert to GitHub ref before committing

### 2. Testing from GitHub Branch (Optional)

```bash
# Commit and push feature branch
git add src/
git commit -m "Description of changes"
git push -u origin feature/my-feature

# Update dashboard_import in webinstall config to test from branch:
# github://rmaher001/water-softener-monitor/src/water-softener-s3-webinstall.yaml@feature/my-feature

# Flash from HA ESPHome dashboard to verify GitHub package works
```

### 3. Release to Production

```bash
# Merge to master
git checkout master
git merge feature/my-feature

# Update version tags in webinstall configs
vim src/water-softener-s3-webinstall.yaml    # Update @main to @1.3.1
vim src/water-softener-lite-webinstall.yaml  # Update @main to @1.3.1

# Compile web installer firmware
~/esphome/venv/bin/esphome compile src/water-softener-s3-webinstall.yaml
~/esphome/venv/bin/esphome compile src/water-softener-lite-webinstall.yaml

# Copy firmware to docs directory
cp src/.esphome/build/water-softener-monitor-s3/.pioenvs/water-softener-monitor-s3/firmware.factory.bin docs/firmware-s3.factory.bin
cp src/.esphome/build/water-softener-monitor-lite/.pioenvs/water-softener-monitor-lite/firmware.factory.bin docs/firmware-lite.factory.bin

# Update manifest versions
vim docs/manifest-s3.json    # Update version field
vim docs/manifest-lite.json  # Update version field

# Create git tag
git tag -a 1.3.1 -m "Release v1.3.1: description"
git push origin 1.3.1

# Push all changes
git add -A
git commit -m "Release v1.3.1: description"
git push
```

### 4. Cleanup

```bash
# Delete feature branch
git branch -d feature/my-feature
git push origin --delete feature/my-feature
```

## Common Commands

### ATOM S3 Development
```bash
# Compile only
~/esphome/venv/bin/esphome compile src/water-softener-s3-dev.yaml

# Upload via OTA
~/esphome/venv/bin/esphome upload src/water-softener-s3-dev.yaml --device 192.168.86.104

# Compile + Upload + Logs
~/esphome/venv/bin/esphome run src/water-softener-s3-dev.yaml --device 192.168.86.104

# View logs
~/esphome/venv/bin/esphome logs src/water-softener-s3-dev.yaml --device 192.168.86.104
```

### ATOM Lite Development
```bash
# Compile only
~/esphome/venv/bin/esphome compile src/water-softener-lite-dev.yaml

# Upload via OTA
~/esphome/venv/bin/esphome upload src/water-softener-lite-dev.yaml --device 192.168.86.32

# Compile + Upload + Logs
~/esphome/venv/bin/esphome run src/water-softener-lite-dev.yaml --device 192.168.86.32

# View logs
~/esphome/venv/bin/esphome logs src/water-softener-lite-dev.yaml --device 192.168.86.32
```

### Web Installer Build
```bash
# Compile both firmware variants
~/esphome/venv/bin/esphome compile src/water-softener-s3-webinstall.yaml
~/esphome/venv/bin/esphome compile src/water-softener-lite-webinstall.yaml

# Copy to docs directory
cp src/.esphome/build/water-softener-monitor-s3/.pioenvs/water-softener-monitor-s3/firmware.factory.bin docs/firmware-s3.factory.bin
cp src/.esphome/build/water-softener-monitor-lite/.pioenvs/water-softener-monitor-lite/firmware.factory.bin docs/firmware-lite.factory.bin
```

## Web Installer

### Building Firmware

Use ESPHome's pre-built `firmware.factory.bin` (includes bootloader, partition table, boot_app0, firmware):

```bash
# Compile web installer configs
~/esphome/venv/bin/esphome compile src/water-softener-s3-webinstall.yaml
~/esphome/venv/bin/esphome compile src/water-softener-lite-webinstall.yaml

# Copy factory binaries to docs directory
cp src/.esphome/build/water-softener-monitor-s3/.pioenvs/water-softener-monitor-s3/firmware.factory.bin docs/firmware-s3.factory.bin
cp src/.esphome/build/water-softener-monitor-lite/.pioenvs/water-softener-monitor-lite/firmware.factory.bin docs/firmware-lite.factory.bin

# Update manifest versions
vim docs/manifest-s3.json    # Increment version
vim docs/manifest-lite.json  # Increment version

# Commit and push
git add docs/
git commit -m "Update web installer firmware to v1.3.1"
git push
```

### Manifest Format

```json
{
  "name": "Water Softener Monitor (ATOM S3)",
  "version": "1.3.0",
  "home_assistant_domain": "esphome",
  "new_install_prompt_erase": true,
  "new_install_improv_wait_time": 0,
  "builds": [
    {
      "chipFamily": "ESP32-S3",
      "improv": true,
      "parts": [
        { "path": "firmware-s3.factory.bin", "offset": 0 }
      ]
    }
  ]
}
```

### User Workflow

1. **Flash Firmware**: User visits web installer, flashes firmware for their hardware
2. **Configure WiFi**: BLE Improv via Home Assistant app (no button press required)
3. **Add to Home Assistant**: Device auto-discovers via ESPHome integration
4. **Configure**: Adjust tank settings in Home Assistant device page

## Version Strategy

Web installer configs use version tags for reproducible builds:

```yaml
dashboard_import:
  package_import_url: github://rmaher001/water-softener-monitor/src/water-softener-s3-webinstall.yaml@1.3.0
  import_full_config: false

packages:
  water_softener: !include water-softener-s3-core.yaml
```

**Why version tags?**
- Immutable - `@1.3.0` always points to same git commit
- Reproducible - users get tested configurations
- No caching ambiguity - ESPHome knows exactly what to fetch

**Update workflow:**
- ESPHome platform updates: Automatic via Home Assistant ESPHome dashboard
- Project updates: Manual via web installer re-flash

## Troubleshooting

### Device offline in ESPHome dashboard
- ESPHome add-on dashboard and HA device discovery are separate
- Device may show in HA but not ESPHome dashboard - normal if not added to dashboard
- Add manually via "Empty Configuration" if needed

### Changes not appearing
- If using `!include`: Changes are instant (just recompile)
- If using GitHub reference: Changes appear after pushing to that branch
- Clear cache: `rm -rf src/.esphome`

### Web installer crashes Chrome
1. Verify using `firmware.factory.bin` (not manual merge)
2. Check `logger` has `deassert_rts_dtr: true` for ESP32-S3
3. Increment manifest version to avoid cache
4. Wait 2-3 minutes for GitHub Pages rebuild

## Recovery: Unbricking ATOM S3

### Symptoms
- USB connects/disconnects repeatedly (~1 second cycle)
- Cannot flash firmware
- No LED activity

### Recovery Procedure

**Enter Download Mode:**
1. Unplug USB cable
2. Press and hold RST button (side button)
3. While holding RST, plug in USB cable
4. Keep holding RST for 5-8 seconds (label says 2s, but needs longer)
5. Release RST button
6. Device should stay connected

**Erase Corrupted Firmware:**
```bash
# Check device connection
ls /dev/tty.* | grep usb

# Erase flash
~/esphome/venv/bin/python -m esptool --chip esp32s3 --port /dev/tty.usbmodem* erase_flash
```

**Flash Good Firmware:**
```bash
~/esphome/venv/bin/esphome run src/water-softener-s3-dev.yaml --device /dev/tty.usbmodem*
```

### Prevention
- Do NOT use web-based ESP flashers (can cause hard bricks)
- Always use ESPHome CLI or HA ESPHome dashboard
- Test on dev hardware first
- Keep USB connected throughout flash process

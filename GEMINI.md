# Gemini Project Context: ESPHome Water Softener Monitor

## Project Overview

This is an ESPHome-based project to monitor the salt level in a water softener brine tank.

- **Core Technology**: ESPHome (declarative YAML for firmware).
- **Hardware**: M5Stack ATOM S3 Lite (ESP32-S3) and a VL53L0X Time-of-Flight (ToF) distance sensor.
- **Functionality**: It measures the distance to the salt level, calculates the remaining percentage, and reports a status (Full, Good, Low, Critical).
- **Integration**: Integrates seamlessly with Home Assistant via the native ESPHome API, with auto-discovery.
- **User Interface**: Provides a web server on the device for direct configuration and a browser-based web installer for initial setup.
- **Architecture**: The project is highly modular. The core logic is encapsulated in `src/water-softener-package.yaml`, which is designed to be reusable. Different entry-point YAML files (`-dev.yaml`, `-webinstall-simple.yaml`, etc.) include this core package to build specific firmware versions.

## Building and Running

### Local Development

The primary workflow for development uses a dedicated configuration that loads the local package for rapid testing.

1.  **Prerequisites**:
    *   Python environment with ESPHome installed.
    *   A `secrets.yaml` file in the project root containing WiFi credentials (`wifi_ssid`, `wifi_password`). A `secrets.yaml.example` is provided.

2.  **Core Files**:
    *   **`src/water-softener-package.yaml`**: The single source of truth. All logic, sensor definitions, and UI components go here.
    *   **`src/water-softener-dev.yaml`**: The development entry point. It uses `!include` to load the local `water-softener-package.yaml`.

3.  **Build/Run Commands**:
    *   **Compile**: `esphome compile src/water-softener-dev.yaml`
    *   **Run (Compile, Upload, and Logs)**: `esphome run src/water-softener-dev.yaml --device <DEVICE_IP_OR_PATH>`
    *   **View Logs**: `esphome logs src/water-softener-dev.yaml --device <DEVICE_IP_OR_PATH>`

### Web Installer Firmware

The project uses a GitHub Pages site to host a web installer.

1.  **Build**: The factory firmware binaries are generated from `src/water-softener-webinstall-simple.yaml` and `src/water-softener-webinstall-multi.yaml`.
    *   `esphome compile src/water-softener-webinstall-simple.yaml`
2.  **Deploy**: The resulting `firmware-simple.factory.bin` is copied to the `docs/` directory. The `docs/manifest-simple.json` is updated, and the changes are pushed to GitHub, which deploys the installer via GitHub Pages.

## Development Conventions

- **Single Source of Truth**: All functional changes must be made in `src/water-softener-package.yaml`. The other `.yaml` files are just entry points for different build targets.
- **Git Workflow**:
    1.  Create a feature or fix branch (e.g., `feature/new-sensor`, `fix/status-logic`).
    2.  Make changes in `src/water-softener-package.yaml`.
    3.  Test locally using the `src/water-softener-dev.yaml` configuration.
    4.  Push the branch to GitHub.
    5.  (Optional) Test the branch directly from GitHub by updating the `packages` URL in an ESPHome dashboard config.
    6.  Merge the branch into `master`. Production users who point to the `@master` branch will get the update automatically.
- **Configuration Management**:
    *   Device-specific substitutions (like `device_name`) are handled in the entry-point YAML files.
    *   Secrets are externalized to `secrets.yaml` and are not committed to the repository.
    *   User-configurable parameters (Tank Height, Thresholds) are exposed as `number` entities in the web interface, allowing for runtime changes without reflashing.

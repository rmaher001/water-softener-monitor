# Archived Configurations

This directory contains configurations that are no longer actively maintained but preserved for reference.

## Multi-Device Support (Archived v1.2.20)

**Files:**
- `water-softener-s3-webinstall-multi.yaml` - ATOM S3 multi-device web installer config
- `firmware-s3-multi.factory.bin` - ATOM S3 multi-device firmware binary
- `manifest-s3-multi.json` - ATOM S3 multi-device web installer manifest

**Why archived:**
- Most users have a single water softener
- Multi-device support adds complexity for minimal benefit
- Power users can enable multi-device manually after adoption

**How to use multi-device (manual method):**
1. Flash standard firmware via web installer
2. Adopt device in ESPHome Dashboard
3. Edit YAML to add `name_add_mac_suffix: true`
4. Recompile and upload

**To restore these configs:**
Move files back to their original locations and update version numbers to match current release.

# Repository Guidelines

## Project Structure & Module Organization
- `src/` — ESPHome YAML configs: `water-softener-package.yaml` (main package), `water-softener-dev.yaml` (local testing), and web‑installer variants.
- `docs/` — Web Installer site (`index.html`), manifests, and release `firmware.factory.bin` binaries.
- `.esphome/` — ESPHome build cache (ignored); do not commit.
- `water-softener.yaml` — Example user config; `secrets.yaml.example` — template for Wi‑Fi credentials.

## Build, Test, and Development Commands
- Validate config: `esphome config src/water-softener-dev.yaml` — fast schema/lint check.
- Compile dev build: `esphome compile src/water-softener-dev.yaml` — builds without flashing.
- Flash/run dev unit: `esphome run src/water-softener-dev.yaml --device <ip|serial>` — uploads and tails logs.
- View logs: `esphome logs src/water-softener-dev.yaml --device <ip|serial>` — check sensors and status.
- Build Web Installer binaries: `esphome compile src/water-softener-webinstall-simple.yaml` (and `-multi`) — use resulting `firmware.factory.bin` from `.esphome/build/.../` and place in `docs/` (see DEVELOPMENT.md).

## Coding Style & Naming Conventions
- YAML: 2‑space indent, no tabs; wrap long lines thoughtfully; keep comments.
- Filenames: kebab‑case with `.yaml` (e.g., `water-softener-package.yaml`).
- ESPHome IDs: `snake_case` for `id:`; user‑facing `name:` in Title Case (e.g., "Salt Level").
- Group related components and preserve section order for readability.
- Lint via `esphome config`; no additional linters required.

## Testing Guidelines
- Create `~/esphome/secrets.yaml` (see `secrets.yaml.example`) or ensure `src/secrets.yaml` symlink resolves.
- Sanity checks: device boots, web_server groups render, distance updates, status text changes with thresholds.
- Include a short `esphome logs` snippet in PRs to demonstrate behavior.

## Commit & Pull Request Guidelines
- Commits: imperative, concise, scope optional. Examples: "Add Restart button", "Fix dashboard adoption", "Document update flow".
- PRs must include: summary, rationale, testing steps/device used, before/after notes or screenshots for UI/log changes, and linked issues.
- Update README/DEVELOPMENT when user or release workflows change.
- Do not commit secrets or transient build artifacts (except curated `docs/*` release files).

## Security & Configuration Tips
- Never commit Wi‑Fi credentials; use `secrets.yaml` only locally.
- Prefer factory binaries for Web Installer; adoption via ESPHome Dashboard sets encryption and clean entity IDs.

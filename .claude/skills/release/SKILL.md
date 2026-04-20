---
name: release
description: Full release pipeline for the water-softener ESPHome project — pre-flight safety gate (PII/dev-artifact scan), code review, compile S3+Lite, stage binaries, update manifests, commit, tag, push, publish GitHub release with auto-drafted notes, verify Pages deploy.
disable-model-invocation: false
argument-hint: <version> [--variants s3,lite] [--dry-run]
---

# Water Softener Release Pipeline

Release a new version of the water-softener firmware end-to-end. User provides the version number (e.g. `2.0.1`); skill applies `-s3` / `-lite` suffixes automatically.

**Autonomous by default.** Stops only on:
- Pre-flight safety failure
- Compile failure
- Code-reviewer flagging substantive issues (non-trivial fixes)
- Push/tag conflict
- Pages deploy failure

**Dry-run mode** (`--dry-run`): print the full plan including commit message and release notes, but make no edits, no commits, no pushes.

## Arguments

- `<version>` — required. Format `X.Y.Z` (no `v` prefix, no variant suffix). Example: `2.0.1`.
- `--variants` — optional, comma-separated. Default: `s3,lite`. Example: `--variants s3` to skip Lite.
- `--dry-run` — optional flag. No side effects.

## Branch detection

Before running, check `git branch --show-current`:
- On `master`: master-direct release flow (build + commit + tag + push).
- On any other branch: **stop and tell the user to merge first.** The project uses a no-PR workflow (Claude Code Desktop sandbox can't unlock keychain for GitHub API writes, so `gh pr create` won't work). Print:
  ```
  You're on branch <name>, not master.
  The release pipeline only runs on master. To finalize:
    git checkout master && git merge <name> && git push origin master
  Then re-run /release <version>.
  ```
  Do NOT attempt `gh pr create` — it will fail with a 401 in this environment.

## Step 1 — Pre-flight safety gate (STOP on any failure)

Run all checks. On failure: print a clear report grouped by severity and STOP. Do not proceed to step 2.

### 1a. Working tree

- `git status --porcelain` — must be clean (no uncommitted/untracked files).

### 1b. Forbidden public-repo files

- `git ls-files | grep -E '^(DEVELOPMENT|CLAUDE|ANALYSIS|CHANGELOG)\.md$|\.env$|\.vscode/|\.idea/|_test\.py$'` — must return empty.
- If any tracked: STOP, show which, ask user to resolve (either `git rm` them first or add to `.gitignore` — skill does not auto-delete).

### 1c. PII and secrets scan (tracked files only)

Grep tracked files in `src/`, `docs/`, root for:
- Email addresses: `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}` — allow-list any GitHub no-reply addresses already present.
- Private IP ranges: `\b(10\.|172\.(1[6-9]|2[0-9]|3[01])\.|192\.168\.)\d+\.\d+\b`
- MAC addresses: `([0-9A-Fa-f]{2}[:-]){5}[0-9A-Fa-f]{2}`
- Internal hostnames: `\.local\b|\.lan\b|ram6\.com`
- Generic secrets: `password\s*[:=]\s*["'][^"']+["']` (allow-list the known fallback `configme123` — it's the intentional Improv setup AP password, documented).

On any hit NOT on the allow-list: STOP with the file + line. Ask user to redact before proceeding.

### 1d. Version sanity

- Parse arg `<version>` — must match `^\d+\.\d+\.\d+$`.
- Compare against existing git tags — must not duplicate (`git tag -l '<version>'` empty).
- New version > last release version (semver compare).

### 1e. `.gitignore` sanity

Must contain at minimum: `secrets.yaml`, `src/secrets.yaml`, `.esphome/`, `CLAUDE.md`, `ANALYSIS.md`, `DEVELOPMENT.md`, `.vscode/`, `.idea/`.

### 1f. Report format

If any check fails, output:
```
PRE-FLIGHT FAILED
=================
[BLOCKING] <check-name>: <details>
[WARNING]  <check-name>: <details>
```
Then STOP.

## Step 2 — Apply version bump

For each variant in `--variants`:
- Edit `src/water-softener-<variant>-webinstall.yaml`: change the `version: "X.Y.Z-<variant>"` substitution line.
- Edit `src/water-softener-<variant>-core.yaml` line 1: update the `# ESPHome Water Softener Salt Monitor - vX.Y.Z` header comment.

If values are already at the target version: skip the edit silently (idempotent).

## Step 3 — Code review (launch agent, wait for result)

Launch `code-reviewer` agent with the diff so far:

```
Review the diff of branch <current-branch> vs master for a release of v<version>.
Files touched so far: <list>.
Goal: catch regressions, race conditions, and stylistic drift from the rest of the codebase.
Report BLOCKING issues, WARNINGS, and cosmetic nits separately.
```

Wait for agent output.

### Handling feedback

- If reviewer reports no BLOCKING or WARNING issues: continue to step 4.
- If reviewer reports only **cosmetic nits** (typos in comments, header version bumps, whitespace): auto-apply them if the edits are obvious one-liners, then continue.
- If reviewer reports BLOCKING or WARNING issues requiring judgment: STOP. Show the report to the user. Do not auto-fix.

## Step 4 — Compile firmware

For each variant in `--variants`:

```bash
source ~/esphome/venv/bin/activate && esphome compile src/water-softener-<variant>-webinstall.yaml
```

Wait for SUCCESS. On failure: STOP, print the last 30 lines of output.

**Build-path quirk** — both S3 and Lite compile to the same output directory because they share `name: water-softener-monitor` in `esphome:`. Both `firmware.factory.bin` AND `firmware.ota.bin` get overwritten by the next compile. To avoid one overwriting the other:
1. Compile S3 first.
2. Copy BOTH `firmware.factory.bin` AND `firmware.ota.bin` from the build dir → staging (see step 5).
3. THEN compile Lite.
4. Copy both Lite binaries from same path → staging.

If `--variants` has only one, no conflict — just compile and copy.

## Step 5 — Stage binaries in `docs/`

After each compile, copy BOTH the factory bin (for web installer) AND the OTA bin (for HA update-entity OTA path, added in 2.0.2):

```bash
BUILD=src/.esphome/build/water-softener-monitor/.pioenvs/water-softener-monitor
cp "$BUILD/firmware.factory.bin" "docs/water-softener-monitor-<variant>-v<version>.bin"
cp "$BUILD/firmware.ota.bin"     "docs/water-softener-monitor-<variant>-v<version>.ota.bin"
```

**Size sanity check** — compare new factory bin size to the previous same-variant factory bin in `docs/`:
- If new is >20% smaller or >20% larger: print a WARNING and ask user to acknowledge before continuing.
- Otherwise continue silently.

**Compute MD5 of the OTA bin** (required for the manifest `ota.md5` field in Step 6):

```bash
OTA_MD5=$(md5 -q "docs/water-softener-monitor-<variant>-v<version>.ota.bin" 2>/dev/null \
       || md5sum "docs/water-softener-monitor-<variant>-v<version>.ota.bin" | awk '{print $1}')
```

Log both `shasum` (for audit of the factory bin) and `md5` (persisted in the manifest for OTA verification) for each variant.

## Step 6 — Update manifest JSON files

For each variant in `--variants`, extend `docs/manifest-<variant>.json`:

1. Top-level `"version": "<version>-<variant>"` — **must include the variant suffix** (e.g. `"2.0.3-s3"`, `"2.0.3-lite"`). This matches the device's `project.version` exactly so `update.http_request` sees `installed == latest` and doesn't flag a phantom update. Learned the hard way in 2.0.2: manifest `"2.0.2"` vs device `"2.0.2-s3"` triggered an infinite "Install → fails because post-install version mismatch" loop.
2. `builds[0].parts[0].path` → `water-softener-monitor-<variant>-v<version>.bin` (web installer — unchanged).
3. `builds[0].ota` block (added/updated — drives HA update-entity notifications):

```json
"ota": {
  "md5": "<OTA_MD5 from step 5>",
  "path": "water-softener-monitor-<variant>-v<version>.ota.bin",
  "release_url": "https://github.com/rmaher001/water-softener-monitor/releases/tag/<version>",
  "summary": "<first line of the release notes summary from step 7 — one sentence>"
}
```

Preserve all other fields (`chipFamily`, `improv`, top-level metadata). If the `ota` block doesn't exist yet (pre-2.0.2 manifests), add it alongside `parts`.

**Why both `parts` and `ota` coexist**: `parts` is used by the ESP-Web-Tools web installer for fresh flashing (factory bin). `ota` is used by ESPHome's `update.http_request` platform for in-place OTA upgrades (OTA bin). Both schemas fit in the same builds[0] object.

## Step 7 — Release notes

Auto-draft notes from `git log <last-tag>..HEAD --pretty=format:'- %s'`.

Also scan the diff for user-visible changes:
- New HA entities? (new `name: "..."` lines under `sensor:`, `binary_sensor:`, `number:`, etc.)
- Removed entities?
- New configurable parameters? (new `number:` entries)

Output a drafted release-notes block:
```markdown
## v<version>

### Changes
<bulleted list from git log>

### User-visible
- New/removed HA entities
- New/removed config parameters
- Behavior changes
```

Save to `/tmp/release-notes-<version>.md` for use in step 10.

## Step 8 — Release commit

Stage ALL release artifacts:
```bash
git add \
  src/water-softener-<variant>-webinstall.yaml \        # for each variant
  src/water-softener-<variant>-core.yaml \              # for each variant (if header was bumped)
  docs/water-softener-monitor-<variant>-v<version>.bin \ # for each variant
  docs/manifest-<variant>.json                          # for each variant
git diff --staged --stat
```

Commit with auto-generated message:
```
Release <version> - <one-line summary from release notes>

<release notes body>
```

If dry-run: stop here, print what would happen.

## Step 9 — Tags

```bash
git tag -a <version> -m "Release <version>"
git tag -f latest        # force-move (this is the mutable release pointer)
```

**Master-direct only.** On a feature branch, skip tag creation — tags are applied after merge.

## Step 10 — Push to origin (always via SSH)

Origin is HTTPS but the stored creds are keychain-locked in Claude Code Desktop. Push via SSH URL directly every time:

```bash
git push git@github.com:rmaher001/water-softener-monitor.git master
git push git@github.com:rmaher001/water-softener-monitor.git <version>      # new annotated tag
git push git@github.com:rmaher001/water-softener-monitor.git latest --force # move the 'latest' pointer
```

Do NOT try `git push origin ...` first — it will fail with `errSecInteractionNotAllowed` and waste time. Do NOT change the stored remote either — the SSH URL is used ad-hoc.

## Step 11 — Publish GitHub Release (best-effort)

```bash
gh release create <version> \
  --title "v<version>" \
  --notes-file /tmp/release-notes-<version>.md \
  --verify-tag
```

**Auth failure is expected in Claude Code Desktop** (sandbox can't unlock keychain, so token reads for API writes fail with 401). Do NOT stop the pipeline on this. Instead:
- Print a single-line warning: `⚠ gh release create skipped (keychain-locked token). Release tag is pushed; create notes manually if needed at https://github.com/rmaher001/water-softener-monitor/releases/new?tag=<version>`
- Continue to step 12.

Read-only `gh` calls (`gh run list`, `gh run watch`) work fine — those don't need write scope.

## Step 12 — Verify GitHub Pages deploy

```bash
sleep 10  # give Actions a moment to register the push
gh run list --workflow=pages.yml --limit 1 --json status,conclusion,databaseId --jq '.[0]'
```

If status is `in_progress`: poll every 30s up to 5 minutes using `gh run watch <id>`.

On `success`: print the web installer URL (`https://rmaher001.github.io/water-softener-monitor/`).
On `failure`: STOP with run ID + link, leave everything else in place (release is tagged/published; only Pages failed).

## Step 13 — Summary report

Print a final summary block:
```
RELEASE v<version> COMPLETE
===========================
✓ Tag: <version>
✓ latest pointer moved to: <short-sha>
✓ GitHub Pages: https://rmaher001.github.io/water-softener-monitor/
✓ Release page: https://github.com/rmaher001/water-softener-monitor/releases/tag/<version>

Binaries (factory — web installer):
  s3:   docs/water-softener-monitor-s3-v<version>.bin    (shasum: <hash>)
  lite: docs/water-softener-monitor-lite-v<version>.bin  (shasum: <hash>)

OTA binaries (HA update-entity path, 2.0.2+):
  s3:   docs/water-softener-monitor-s3-v<version>.ota.bin    (md5: <hash>)
  lite: docs/water-softener-monitor-lite-v<version>.ota.bin  (md5: <hash>)

Next steps (user-driven — Claude does NOT flash devices):
  - Devices on 2.0.2+ will auto-surface the new version as an HA update
    entity within 6 hours of the next poll.
  - Prod can also be OTA'd via ESPHome Dashboard as before.
  - Verify firmware version reads <version>-s3 / <version>-lite in HA.
```

## Notes on this project's conventions

- Tags use **no `v` prefix** (e.g., `2.0.1`, not `v2.0.1`).
- `latest` is a **mutable lightweight tag** — force-moving it is intentional, not destructive.
- Firmware version substitution `X.Y.Z-<variant>` is the SOURCE of truth; manifest/release names use `X.Y.Z` without suffix.
- S3 and Lite share `esphome.name: water-softener-monitor` — their compile outputs clobber each other in the build dir. Skill handles this by compiling serially and staging between.
- **Never** run `esphome upload` or `esphome run`. `esphome compile` only. Firmware deployment is strictly user-driven per project CLAUDE.md.
- `@latest` GitHub tag is what ESPHome Dashboard resolves for OTA — moving it is how existing devices get the update.
- **No-PR workflow**: Claude Code Desktop can't unlock keychain, so GitHub API writes (`gh pr create`, `gh release create`) return 401. Git operations work fine via SSH. User merges feature branches into master manually before invoking this skill.
- **SSH for all git pushes**: the origin remote is HTTPS (keychain-locked), but the SSH URL `git@github.com:rmaher001/water-softener-monitor.git` works. Skill uses SSH URL directly without modifying the stored remote.
- **Two binaries per variant per release** (2.0.2+): `firmware.factory.bin` for fresh web installs, `firmware.ota.bin` for in-place HA update-entity OTAs. Both compiled in a single `esphome compile` pass; both must be published to `docs/` with versioned filenames. Manifest's `builds[0].parts[]` points at the factory bin; `builds[0].ota.path` points at the OTA bin.
- **Manifest schema** combines ESP-Web-Tools (for web installer) and ESPHome `update.http_request` (for HA update entity) formats in the same JSON. `parts[]` = factory bin for fresh installs; `ota{}` = OTA bin + MD5 + release_url + summary for in-place OTA.

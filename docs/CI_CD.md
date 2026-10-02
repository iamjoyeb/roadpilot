# CI/CD Design — RoadPilot

| Field | Value |
|---|---|
| Project | RoadPilot |
| Status | **Live and green** — all four stages pass on `main` (first fully green run: `37008619949`, 2026-10-02; MSI content fixed in `37009320259`). Every push produces retained Windows folder, MSI, and Android APK artifacts (§4) |
| Last updated | 2026-10-01 |
| Related | [DEVELOPMENT_WORKFLOW.md](DEVELOPMENT_WORKFLOW.md), [TECH_STACK.md](TECH_STACK.md), [TESTING.md](TESTING.md) |

---

## 1. Goals

1. Every push to `main` produces **Windows** and **Android** artifacts that any machine can install — the dev machine is never required to build.
2. Artifacts are **identifiable**: commit SHA + build number embedded in names and metadata.
3. Failures are **loud and logged**; no artifacts from failed runs.
4. The pipeline stays **fast and cacheable** on GitHub-hosted runners (4 GB-friendly project ⇒ builds must not be bloated either).

## 2. Triggers

| Trigger | Runs | Purpose |
|---|---|---|
| Push to `main` | Full pipeline (all stages, artifacts) | Required behavior; produces retained build artifacts |
| Pull requests (recommended) | Fast gates only: lint + headless tests | Catch breakage before merge |
| `workflow_dispatch` (recommended) | Full pipeline on demand | Rebuild an old commit / debug pipeline |
| Tags (future, §9) | Optional publication of existing artifacts | Not planned — OQ6 resolved: no semantic versions/tags; artifacts are identified by SHA + run number |

## 3. Pipeline stages

```
push to main
    │
    ▼
┌─────────────────────────────────────────────┐
│ STAGE 1 — Quality gates        (ubuntu)    │
│   • install pinned Godot 4.7.2 (headless)  │
│   • gdlint + gdformat --check              │
│   • GUT headless tests (unit/integration/  │
│     scenarios)                             │
└───────────────┬─────────────────────────────┘
                │ success required
        ┌───────┴────────┐
        ▼                ▼
┌──────────────────┐  ┌──────────────────────────┐
│ STAGE 2A —       │  │ STAGE 2B —               │
│ Windows build    │  │ Android build            │
│ (windows-latest) │  │ (ubuntu-latest)          │
│ Godot export →   │  │ JDK 17 + Android SDK +   │
│ Windows x64      │  │ Godot export → APK       │
│ folder           │  │ (debug-signed)           │
└────────┬─────────┘  └──────────┬───────────────┘
         ▼                       │
┌──────────────────┐             │
│ STAGE 3 —        │             │
│ MSI packaging    │             │
│ (WiX, same       │             │
│ Windows runner)  │             │
└────────┬─────────┘             │
         ▼                       ▼
┌─────────────────────────────────────────────┐
│ STAGE 4 — Artifacts + metadata (per target) │
│   upload-artifact with build-info.json      │
└─────────────────────────────────────────────┘
```

Stages 2A and 2B run **in parallel** after Stage 1. Stage 3 runs within the Windows job (no artifact re-upload round-trip needed between export and packaging).

**Jobs are independent per target** so an Android-only failure does not hide a good Windows build (and vice versa) — but any failed job marks the run failed (§6).

### 3.0 Stage 0 — preflight (readiness gate)

Every run starts with a `preflight` job that reports which prerequisites exist:

| Prerequisite | Enables |
|---|---|
| `src/`, `tests/` script dirs | Quality gates run lint/tests; otherwise they skip with a notice |
| `project.godot` + `export_presets.cfg` | Build jobs (2A/2B/3) run; otherwise they are skipped |

While the repository is docs-only (or partially scaffolded), the workflow stays **green** and the preflight job summary states exactly what is missing — builds turn on automatically, no workflow edit needed. This replaces the manual ordering of §8 bring-up steps.

**Contracts the workflow expects from `export_presets.cfg`** (to be honored when presets are created):

- Preset names must be exactly **`Windows Desktop`** and **`Android`** (Godot defaults; the CLI export flags reference them).
- Windows output: `build/windows/RoadPilot.exe`; Android output: `build/android/roadpilot.apk`.
- Android preset: debug keystore fields may be **empty** — CI injects them via `GODOT_ANDROID_KEYSTORE_DEBUG_*` environment variables (§3.4). Package id/ABI/min-SDK are preset-only concerns (OQ2 resolved: `arm64-v8a`, API 24+).

### 3.1 Stage 1 — Quality gates (ubuntu-latest)

1. Checkout.
2. Install pinned **gdtoolkit 4.5.0** (pip).
3. Run `gdlint` and `gdformat --check` over `src/` and `tests/` (skipped with a notice while those directories are empty/absent).
4. Install pinned **Godot 4.7.2** Linux editor binary (cached) and run **GUT headless** when `addons/gut/` and `project.godot` exist; otherwise skip with a notice (bring-up step 2, §8).
5. Fail the pipeline on any lint/test failure.

*Bring-up note:* guards keep the workflow green while inputs don't exist yet; by design every push to `main` requires all stages green once scaffolding is complete (PRD AC11/AC12).

### 3.2 Stage 2A — Windows build (windows-latest)

1. Checkout.
2. Install pinned Godot 4.7.2 **Windows** editor binary (cached) + matching **export templates** (cached per OS, staged into `%APPDATA%\Godot\export_templates\4.7.2.stable\`).
3. Run Godot in headless mode: `--import`, then export the preset named **`Windows Desktop`** → `build/windows/RoadPilot.exe`. Three runner-specific safeguards (found in runs 1–2 of CI): the job creates `build/windows/` first (**Godot fails if the export target's parent directory is missing**); Godot is invoked via **`Start-Process -Wait -PassThru`** because the Windows editor is a GUI-subsystem exe (`&` does not wait for it and leaves `$LASTEXITCODE` empty); and the preset name is passed **with embedded quotes** in `-ArgumentList` (Start-Process does not quote elements — an unquoted `Windows Desktop` was parsed as preset `Windows`).
4. Verify outputs exist (executable + `*.pck`, expected directory structure).
5. Generate `build-info.json` (§5) inside `build/windows/`.
6. Proceed to Stage 3 in the same job.

Export presets exist (`export_presets.cfg`, scaffolded 2026-10-01; preset names + config validated locally against Godot 4.7.2); design decision: export **release** builds for artifacts (debug symbols stripped by template), consistent across MSI and folder build.

### 3.3 Stage 3 — MSI packaging (same Windows job)

1. Install **WiX Toolset 5.0.2** (`dotnet tool install --global wix --version 5.0.2`).
2. Generate a compact `.wxs` (in the workflow) and compile the MSI. Harvesting uses the WiX v5 **`<Files Include="…\build\windows\**" />`** element (built-in directory harvesting — no Heat step). **Path gotcha (run 3):** `<Files>` resolves its `Include` **relative to the `.wxs` file's directory** (`build/installer/`), not the repo root — a relative `build\windows\**` harvested nothing (`WIX8601` warning, exit 0, empty 28 KB MSI). The workflow therefore passes an **absolute** `$env:GITHUB_WORKSPACE\build\windows\**` path. Requirement: **installable MSI that places a working build under `Program Files\RoadPilot` on a clean Windows machine.**
3. Version: **`0.0.<run_number>`** (build-number versioning per OQ6 — not a release version). Stable `UpgradeCode` constant lives in the workflow (product identity across builds). `build-info.json` sits next to and inside the MSI payload.
4. Smoke validation on the runner: MSI file exists and is **at least 1 MB** — enforced by a size guard in the workflow (`WIX8601` empty harvests only warn, so exit codes are not enough; full install test happens manually per TESTING.md).
5. First-run validation note: the harvested install layout (prefix stripping of `build\windows\`) is verified on the first green Windows run and adjusted in the workflow if needed.

**Note:** no Windows code signing (OQ4 resolved: none). Expect SmartScreen warnings — acceptable for personal use.

### 3.4 Stage 2B — Android build (ubuntu-latest)

1. Checkout.
2. Setup **JDK 17** (Temurin). The Android SDK is **preinstalled on GitHub's Ubuntu runner images** (`ANDROID_HOME`): the job accepts licenses and installs `platform-tools`, platforms 34/35 + matching build-tools using the image's `cmdline-tools/latest` `sdkmanager` (adjust to Godot's target API if needed). `android-actions/setup-android` was **removed** after first-run CI showed it failing on `sdkmanager tools` — a package Google removed from the repositories.
3. Install pinned Godot 4.7.2 Linux editor + export templates (same cache keys as the other jobs).
4. **Debug keystore (OQ5 resolved: debug-only):** use the optional `ANDROID_KEYSTORE_B64` GitHub secret if present (alias `roadpilot`, store/key password `roadpilot` — keeps the APK signature stable across runs); otherwise the workflow generates an ephemeral keystore with `keytool` and prints a notice that the device requires uninstall-before-reinstall when the signature changes.
5. Headless export: `--import`, then `--export-debug "Android"` → `build/android/roadpilot.apk` (the job creates `build/android/` first — same parent-directory rule as §3.2), with keystore settings injected via `GODOT_ANDROID_KEYSTORE_DEBUG_PATH/_USER/_PASSWORD` env vars (no credentials committed in `export_presets.cfg`; env-var names verified against the Godot 4.7.2 binary).
6. Verify APK exists; generate `build-info.json`.
7. **No custom Gradle build** in MVP (TECH_STACK §12) — uses prebuilt templates only. No release/store signing (OQ5: personal project, no publishing).

ABI/min-SDK belong to the export preset (OQ2 resolved: `arm64-v8a`, min API 24 — recorded in PRD §16); the workflow itself is ABI-agnostic. The project setting `rendering/textures/vram_compression/import_etc2_astc=true` (in `project.godot`) is a **hard precondition** for Android export — Godot refuses to export without it.

## 4. Artifact naming and retention

| Artifact | Name pattern (example) |
|---|---|
| Windows folder build | artifact `roadpilot-windows-x64-<shortsha>-<run_number>` (folder build incl. `build-info.json`) |
| MSI installer | artifact `roadpilot-msi-x64-<shortsha>-<run_number>`, file `RoadPilot-0.0.<run_number>.msi` |
| Android APK | artifact `roadpilot-android-<shortsha>-<run_number>`, file `roadpilot.apk` |
| Build metadata | `build-info.json` included **in every artifact** |
| Failure diagnostics | `roadpilot-*-logs-<shortsha>-<run_number>`, 7-day retention, only on failure |

- `<shortsha>` = first 7 characters of the triggering commit; `<run_number>` = GitHub Actions run number (monotonic build number).
- **Retention:** 30 days per artifact (project setting per workflow) — enough for test rounds without accumulating gigabytes; adjust if quota (OQ1) matters.
- Failed runs publish **no** artifacts.

## 5. Build metadata (`build-info.json`)

Every artifact carries:

```json
{
  "project": "RoadPilot",
  "version": "0.0.<run_number>",
  "commit": "<full sha>",
  "short_sha": "<7 chars>",
  "branch": "main",
  "build_number": <github run number>,
  "run_id": "<github run id>",
  "built_at": "<UTC ISO timestamp>",
  "godot_version": "4.7.2",
  "target": "windows-x64 | android-apk"
}
```

`version` is **build-number versioning only** (OQ6 resolved: no semantic versioning; the `0.0.<run>` form exists because Windows installers require a numeric product version). Identity for humans = `commit` + `build_number`.

Purpose: a tester on a different machine can tell exactly what build they hold (PRD AC9, US10).

## 6. Caching strategy

| Cache | Key basis | Why |
|---|---|---|
| Godot editor binary (Linux + Windows) | `godot-4.7.2-editor-<os>` | ~50 MB download avoided per run |
| Export templates (unpacked in workspace, staged per OS) | `godot-4.7.2-templates-<os>` | ~1 GB download avoided |

Not cached in v1 (installs are fast; add later if they show up in run time): gdtoolkit (pip), WiX (`dotnet tool`), Android SDK packages.

- Caches are **immutable on key hit** (GitHub caches cannot be updated in place) — keys include the version so upgrades invalidate correctly.
- No cache is required for *correctness*; a cold run must still succeed (cache miss = slower, not broken).

## 7. Failure handling and logs

| Situation | Behavior |
|---|---|
| Lint or test failure | Stage 1 fails → no artifact stages run → run marked failed |
| Export failure (Godot nonzero exit, missing outputs) | That target's job fails; the other target's job still completes (its artifact remains available for inspection), but the **overall run is failed** |
| MSI/APK packaging failure | Same as above: job fails, run fails |
| Infrastructure flake (runner/network) | Rerun failed jobs via GitHub UI or `workflow_dispatch`; no auto-retry loops (keep it predictable) |
| Red `main` | Treated as top priority: fix-forward with a small PR (DEVELOPMENT_WORKFLOW §2) |
| **Logs** | Standard GitHub Actions logs per job/step; Godot headless output captured to console; on failure, an optional diagnostics artifact (`roadpilot-*-logs-<sha>-<run>`, produced `*.log` files) with 7-day retention, guarded by `if: failure()` |

Concurrency: a `concurrency` group per branch cancels superseded in-flight runs on rapid pushes to `main` (saves minutes; OQ1).

## 8. Bring-up order and CI run log

The workflow (`.github/workflows/build.yml`) contains every stage with **guards** that skip missing prerequisites and say so in the preflight summary. Status:

1. ~~**Step 1:** lint on push to `main`~~ — **done** (now linting the scaffolded `src/`; `gdlint` + `gdformat --check` pass locally with the pinned gdtoolkit 4.5.0).
2. **Step 2:** activates when `tests/` + GUT (`addons/gut/`) exist.
3. ~~**Step 3:** `project.godot` + `export_presets.cfg`~~ — **done** (scaffolded 2026-10-01); build jobs activate on the next push to `main`.
4. ~~**Step 4:** MSI packaging~~ — **done** (31.2 MB MSI in run 4; the run-3 empty harvest was found by artifact audit and fixed — see §3.3). Clean-machine install test remains manual (TESTING §9).
5. ~~**Step 5:** Android export~~ — **done** (28.3 MB APK in run 4, debug keystore per OQ5). Device install test remains manual (TESTING §8).
6. ~~**Step 6:** PR fast gates + `workflow_dispatch`~~ — **done** (PRs run lint/tests only; builds run on `main` pushes + manual dispatch).

**Local pre-validation (2026-10-01, Linux, real Godot 4.7.2 editor):** headless `--import` clean; boot scene runs; **Windows export produces `RoadPilot.exe` + `RoadPilot.pck`**; Android preset passes config checks up to the SDK lookup (CI provides SDK); keystore env-var names verified against the binary; WiX `.wxs` well-formed.

**First CI run (2026-10-01, run `36868939265`):** preflight ✅, lint & tests ✅; two runner-only failures, both fixed immediately:

1. **Windows** — `& $exe` on the GUI-subsystem Godot exe returned without waiting (`$LASTEXITCODE` empty → false export failure). Fixed: `Start-Process -Wait -PassThru` + `.ExitCode` (parse-checked with PowerShell 7.4 locally).
2. **Android** — `android-actions/setup-android@v3` failed on `sdkmanager tools` (removed package). Fixed: action dropped; the preinstalled runner SDK is used directly (path resolution mock-tested for both `latest/` and versioned `cmdline-tools` layouts).

**Second CI run (run `37007644588`):** both mechanisms fixed (real exit codes, proper log sequencing); two deeper errors surfaced and were fixed:

3. **Windows** — `-ArgumentList` does not quote elements: `Windows Desktop` reached Godot as preset `Windows` (invalid). Fixed: embedded quotes around the preset argument.
4. **Android** — Godot rejected the export: `ETC2/ASTC texture compression is required`. Fixed: `textures/vram_compression/import_etc2_astc=true` added to `project.godot` (setting key verified against the 4.7.2 binary).

Still to validate on the next run: WiX harvest/install layout (Windows runner), full Android export end-to-end (ETC2 was the last config error listed), templates/keystore steps green in sequence. *(Outcome: see runs 3–4 below.)*

**Third CI run (run `37008619949`): all four jobs green — first fully green pipeline.** All three artifacts produced with correct `<shortsha>-<run>` naming and 30-day retention; `build-info.json` matches the §5 contract exactly; Android keystore notice emitted as designed. **Artifact content audit caught one silent defect:** the MSI was 28 KB (empty) because of the WiX `Files` relative-path gotcha (`WIX8601` only warns). Fixed: absolute harvest path + mandatory ≥1 MB MSI size guard (§3.3).

**Fourth CI run (run `37009320259`, 2026-10-02): all green with content-verified artifacts** — Windows folder 38.9 MB, **MSI 31.2 MB** (guard passed, `WIX8601` gone), Android APK 28.3 MB. Pipeline bring-up is complete for stages 0–3; what remains is **manual validation** (clean-machine MSI install, Vivo device install — TESTING.md §8/§9) and **bring-up step 2** (GUT tests activate when `tests/` + `addons/gut/` land).

## 9. Optional future: release workflow

Not planned (OQ6: no version tags/releases; personal project, no publishing). If ever needed: a `workflow_dispatch`-triggered job that publishes *existing* artifacts (already SHA-identified) to a GitHub Release — with Android release keystore (OQ5) and MSI code signing (OQ4) decided at that time.

## 10. Explicit non-goals (CI)

- No deployment/CD targets (no stores, no hosting).
- No multi-OS build matrix beyond the two required targets.
- No performance benchmarking gate in MVP (perf is manual per TESTING.md; automated FPS gates are experimental).
- This document remains the **specification**; the executable version is `.github/workflows/build.yml`. If workflow and this doc disagree, the divergence is a bug — fix both in one change.

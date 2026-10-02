# Tech Stack — RoadPilot

| Field | Value |
|---|---|
| Project | RoadPilot |
| Status | Active — planning complete (2026-10-01); living document |
| Last updated | 2026-10-01 |
| Related | [DECISIONS.md](DECISIONS.md), [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md), [CI_CD.md](CI_CD.md) |

> Every entry answers **what** and **why**. Alternatives considered are in [DECISIONS.md](DECISIONS.md).

---

## 1. Recommended engine

| Item | Choice |
|---|---|
| Engine | **Godot Engine** |
| Version | **4.7.2 (stable)** — exact version pinned for editor, export templates, and CI |
| License | MIT (engine itself) |

**Why Godot for this project:**

| Criterion | Godot | Unity | Effect on this decision |
|---|---|---|---|
| Linux development | First-class official Linux editor | Linux editor exists but is secondary/support-tier | Dev machine is Zorin OS 18.1 → Godot native fit |
| 4 GB RAM dev machine | Lightweight editor (~100 MB installer, modest runtime) | Editor + services typically several GB of RAM/disk; heavy IDE | Godot keeps the 4 GB machine usable |
| GitHub Actions / headless CI | `--headless` first-class; download a single binary + export templates | Requires game-ci Docker images (multi-GB) + license activation steps | Godot CI is simple, fast, cacheable |
| Android export | Official one-click export templates; works from Linux CI | Solid, but adds Unity build-server/licensing overhead on CI | Godot meets the APK requirement with less machinery |
| Windows export from Linux | Official cross-export (no Wine needed) | Possible but same licensing/CI weight | Windows artifact on every push stays cheap |
| 3D on Intel HD 4600 | Compatibility renderer = OpenGL 3.3; runs on integrated GPUs | Generally heavier baseline expectations, larger builds | Godot matches low-end-first requirement |
| Licensing | MIT, no runtime fees, no revenue gates, fully open source | Personal license terms historically changed (runtime-fee episode); proprietary | Open source = long-term maintainability, no vendor risk for a personal project |
| AI/vibe coding friendliness | GDScript: small files, fast iteration, simple syntax, headless-runnable logic | C# + heavier project scaffolding, compile waits | GDScript suits small reviewable AI-generated diffs |
| Long-term maintainability | Community-driven; project fully self-hostable (even building engine from source) | Depends on vendor account/terms continuity | MIT wins for an indefinitely-lived hobby project |

**Not chosen:** Unity (CI/licensing weight, heavier on 4 GB RAM, Linux second-class), Unreal (far too heavy for Intel HD 4600 and this scope), custom engine (reinventing rendering/physics/platform exports is anti-goal per PRD non-goals).

**Version policy:** pin **4.7.2** everywhere (dev editor, export templates, CI download). Upgrade only to another stable 4.x patch/minor as a deliberate, tested change — never to dev/RC builds. The version is a single source of truth referenced by CI (see CI_CD.md) and by `project.godot`.

## 2. Programming language

| Item | Choice |
|---|---|
| Language | **GDScript**, statically typed where practical (`type hints` on public APIs) |

**Why:** native to Godot (zero build step), fast iteration, minimal ceremony, trivially lintable (`gdlint`/`gdformat`), and its readability keeps AI-generated diffs small and reviewable. C# (.NET) was rejected for MVP: adds a compile step, larger templates, and heavier CI for no requirement we have. If a subsystem ever needs C#, Godot supports mixed projects — that would be an ADR, not a default.

## 3. Physics technology

| Item | Choice |
|---|---|
| Physics engine | **Godot built-in physics (GodotPhysics3D)** — no third-party physics lib |
| Vehicle model | Lightweight **raycast vehicle layer** on `RigidBody3D`, behind the `VehicleController` interface |

**Why:** free, integrated, headless-testable, and enough for "believable and controllable" (PRD non-goal: not simulation-grade). Godot 4 no longer ships Bullet, so external C++ physics would fight the architecture for zero MVP gain. The raycast layer gives tunable, data-driven behavior while `VehicleController` keeps the implementation swappable (ADR-003).

## 4. Rendering approach

| Item | Choice |
|---|---|
| Renderer | **Compatibility** (OpenGL 3.3) |
| Quality model | Scalable presets: **low / medium** (MVP), resolution scale, shadow toggle, draw-distance clamp |
| Art direction | Low-poly models, **texture atlases** for repeated world elements, textures ≤ 1024² in MVP, few unique materials |

**Why:** the Compatibility renderer is the only renderer that treats Intel HD 4600 as a first-class citizen, and it maps cleanly to the Windows/Android low-end targets. Forward+/Mobile (Vulkan) are optional upgrades *later* if hardware allows — behind the graphics preset, never as a requirement.

## 5. Input system

| Item | Choice |
|---|---|
| Mapping | Godot **InputMap** actions (`steer_left`, `steer_right`, `throttle`, `brake`, `auto_drive_engage` (hold-to-confirm), `pause`, …) |
| Consuming | `ManualInputDriver` normalizes actions → `DriveCommand` + intents |
| MVP devices | Keyboard (Windows/Linux), plus Android build validated with defaults |
| Later devices | Gamepad and touch layers (post-MVP) plug into the same `ManualInputDriver` role |

**Why:** InputMap gives remapping, per-platform bindings, and deadzone handling for free, while the driver role keeps device specifics out of the vehicle and Auto Drive (same decoupling pattern as everything else).

## 6. UI technology

| Item | Choice |
|---|---|
| Framework | Godot **Control nodes** (anchors, containers), scenes under `src/ui/` |
| Layout baseline | 1366×768; scale-aware so other resolutions degrade gracefully |
| No third-party UI kits | — |

**Why:** built-in, lightweight, themeable, export-safe on both targets; a UI library would add dependency surface (NFR-8) for a HUD + one settings screen.

## 7. Audio system

| Item | Choice |
|---|---|
| Playback | Godot `AudioStreamPlayer`/`AudioStreamPlayer3D` + **audio buses** (Master/SFX/Music) |
| Triggering | EventBus events (+ snapshot read for engine pitch later) |
| MVP content | Placeholders only; real audio design is future scope (PRD) |

**Why:** integrated, cheap, bus architecture gives volume settings for free when config lands. No middleware (FMOD/Wwise) — unjustified dependency for this scope (NFR-8).

## 8. Asset formats

| Kind | Format | Why |
|---|---|---|
| 3D models | **glTF 2.0 (.glb)** | Godot's native-ish interchange, compact, industry standard, importable headlessly |
| Textures | **PNG** source → imported/compressed (atlases) | Universal, diff-tool friendly, atlas workflow for draw-call/memory savings |
| Audio | **OGG Vorbis** (loops/music), **WAV** (short SFX) | Good size/quality balance; WAV for latency-sensitive one-shots |
| Fonts | **TTF/OTF** | Standard, embeddable |
| Routes/data | **JSON** | Tool-agnostic, human-editable, diffable in git, validated on load (ADR-004) |
| Scenes/prefs | Godot **.tscn / .tres** | Engine-native text formats (git-friendly) |

**Asset policy:** original or properly licensed only (README principle 5); low-poly and atlased by default (NFR-7); every asset's provenance documented when added.

## 9. Version control

| Item | Choice |
|---|---|
| VCS | **Git**, hosted on **GitHub** (`roadpilot` repository) |
| Branching | `main` = stable integration; short-lived `feature/*`, `fix/*`, `chore/*` branches; PRs for merges |
| Triggers | Every push to `main` runs CI and produces retained artifacts |
| Binary policy | Keep repo light: no huge binaries; LFS only if a real need appears (decision point if assets grow) |

**Why:** GitHub gives us PR review + Actions in one place; the simple mainline workflow fits a small/one-person project and keeps AI-assisted changes reviewable (see DEVELOPMENT_WORKFLOW.md).

## 10. CI/CD

| Item | Choice |
|---|---|
| Platform | **GitHub Actions** |
| Trigger | Push to `main` (plus PR runs for lint/test — see CI_CD.md) |
| Stages | lint + headless tests → Windows export → MSI packaging → Android APK |
| Godot on CI | Pinned 4.7.2 Linux editor binary + matching export templates (cached) |
| Artifacts | Retained per run, named with commit SHA + build number, with `build-info.json` |

**Why:** native to the hosting platform, supports Linux and Windows runners (Android builds on Linux), and caching makes Godot downloads cheap. Design in [CI_CD.md](CI_CD.md).

## 11. Windows build system

| Item | Choice |
|---|---|
| Game build | Godot **export templates** → Windows x86_64 folder build (runner: `windows-latest`) |
| Installer | **WiX Toolset** (v4 or later, `dotnet` tool) compiling an MSI from the exported folder |
| Version/metadata | Embedded in artifact name + `build-info.json`; MSI product version `0.0.<run_number>` (build-number versioning — OQ6 resolved: no semantic versions) |

**Why WiX:** the requirement is specifically an **MSI**; WiX is free, scriptable, and runs headless on a Windows CI runner. Alternatives: Inno Setup (produces .exe installer, not MSI), MSIX (code-signing expectations), AdvancedInstaller (commercial) — all rejected for the MSI requirement (ADR-006). Exact harvesting approach is chosen at implementation time (kept out of this doc to avoid inventing details).

## 12. Android build system

| Item | Choice |
|---|---|
| Packaging | Godot **Android export templates** → APK (runner: `ubuntu-latest`) |
| Toolchain | JDK **17** + Android SDK (command-line tools, platform/build-tools) installed on the runner |
| Build mode | Default Godot export (**no custom Gradle build** in MVP) → fewer moving parts, faster CI |
| Signing | **Debug keystore only** (OQ5 resolved: personal project, no store publishing): CI generates one per run, or uses the optional `ANDROID_KEYSTORE_B64` secret for a stable signature; no release keystore |
| ABI focus | arm64 primary (universal APK acceptable) — final ABI set is OQ2 |

**Why:** the default export path avoids Gradle entirely for MVP, which is the single biggest CI complexity saver on the Android side. Gradle/custom build only becomes necessary when a native plugin is needed (not planned).

## 13. Testing strategy

| Level | Tool/means | Where |
|---|---|---|
| Unit tests | **GUT** (Godot Unit Test framework), pinned version compatible with Godot 4.7 | `tests/unit/`, headless in CI |
| Integration tests | GUT, wiring loader→runner→agent, control mode, config round-trip | `tests/integration/` |
| Auto Drive scenario tests | Headless physics/logic runs of scripted routes (straight/curve/stop/…) | `tests/scenarios/` |
| Build validation | CI checks: artifacts exist, `build-info.json` SHA matches trigger, lint clean | CI pipeline |
| Manual/gameplay | Checklist-based sessions on Windows + Android builds | TESTING.md |
| Low-end/perf | Session on the dev machine (HD 4600 @ 1366×768, low preset, ≥30 FPS) | TESTING.md |

**Why GUT:** mature, Godot-native, runs headless for CI, no external runtime. Alternatives (gdUnit4, Wisetest) are fine too — GUT chosen for longevity/simplicity; recorded as ADR-008. Headless-first design (NFR-5) keeps tests runnable on the 4 GB machine and CI.

## 14. Static analysis / linting

| Tool | Purpose |
|---|---|
| **gdlint** (gdtoolkit) | Style/convention lint for GDScript — runs in CI as a gate |
| **gdformat** (gdtoolkit) | Canonical formatting — enforced to end style debates in AI-generated code |
| Editor | Godot's built-in script warnings kept visible (unused code, shadowing, etc.) |

**Why:** gdtoolkit is the community-standard, pip-installable, headless-friendly toolchain; formatting-by-tool keeps diffs minimal and reviewable — critical for AI-assisted workflows. Both are dev/CI-time only, not runtime dependencies.

## 15. Documentation approach

- All docs are **Markdown in `/docs`**, linked from `README.md`; one document per concern (PRD, design, roadmap, workflow, CI, testing, decisions).
- **ADRs in `DECISIONS.md`** record decisions with context/alternatives/consequences — the file an AI agent must consult before changing architecture.
- Rule: **architecture changes update docs in the same PR** (README principle 7). Docs use plain language, avoid enterprise jargon, and stay implementation-oriented.
- No generated doc sites (MkDocs etc.) — overkill for now; plain Markdown renders natively on GitHub.

## 16. Dependency summary

Runtime dependencies: **Godot 4.7.2 only.**

Dev/CI-time (pinned, not shipped): GUT, gdtoolkit (gdlint/gdformat), WiX (CI), JDK 17 + Android SDK (CI), GitHub Actions (CI).

Any addition to either list requires justification and a `DECISIONS.md` entry (AI coding principle 6).

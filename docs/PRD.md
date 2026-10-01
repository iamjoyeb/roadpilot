# Product Requirements Document (PRD) — RoadPilot

| Field | Value |
|---|---|
| Project | RoadPilot |
| Status | Draft — planning phase |
| Last updated | 2026-10-01 |
| Related | [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md), [FEATURES.md](FEATURES.md), [DECISIONS.md](DECISIONS.md) |

---

## 1. Product overview

RoadPilot is a lightweight 3D driving simulator focused on buses. The player drives a bus manually on a simple 3D road network, or hands control to **Auto Drive**, an AI driver that follows a predefined waypoint route from origin to destination.

The distinguishing feature is Auto Drive: automation is a core system with its own architecture, not a cheat or cutscene. It uses the same driving interface as the player, which keeps the vehicle simulation honest and makes both systems testable in isolation.

The project is personal/experimental. All code, assets, branding, UI, maps, sounds, and other content are original or properly licensed. It is inspired by the general concept of bus-driving simulators only.

## 2. Problem / opportunity

- Bus-driving simulators are popular, but existing titles are heavy, proprietary, or closed to modification. There is no small, open, hackable project where a developer can explore **vehicle automation as a first-class game feature** on modest hardware.
- Integrating an AI driver is usually bolted on and tightly coupled to a specific vehicle. This makes it hard to test, hard to extend, and impossible to reuse.
- Development hardware is constrained (Intel HD 4600, 4 GB RAM), which rules out "throw GPU at it" solutions and forces a disciplined, lightweight architecture — a useful constraint that also benefits end users on low-end devices.

## 3. Vision

A small, original, MIT-licensed bus simulator where:

- Manual driving feels responsive and understandable.
- Auto Drive genuinely drives the same bus, on the same roads, through the same commands the player uses.
- The whole thing runs on a laptop with integrated graphics, and builds itself on every push to `main`.

## 4. Goals

| ID | Goal |
|---|---|
| G1 | Deliver a playable MVP: manual driving + Auto Drive on one bus and one route. |
| G2 | Keep Auto Drive architecturally independent of the vehicle implementation. |
| G3 | Run at an acceptable frame rate on Intel HD 4600 at 1366×768 (see NFR-1). |
| G4 | Produce Windows and Android artifacts automatically on every push to `main`. |
| G5 | Keep the codebase small, modular, and legible to a human or AI reviewer. |
| G6 | Use only original or properly licensed content. |

## 5. Non-goals

- Not a commercial product; no revenue targets, no store publishing requirements in scope.
- Not a vehicle-physics research project. Physics is "believable and controllable", not simulation-grade.
- Not a full open-world city. One small map for MVP.
- Not a multiplayer game.
- Not feature-parity with any existing commercial bus simulator.
- Not a custom engine or framework build — we use an existing engine (Godot) as-is.
- MVP does **not** require: passengers, bus stops with boarding, traffic, traffic lights, weather, day/night, multiple vehicles, save/load of game progress, mobile touch controls, or gamepad support (these are future/post-MVP — see [FEATURES.md](FEATURES.md)).

## 6. Target experience

A player launches the game, spawns on a short road with a defined route, and can either:

1. Drive the bus themselves (steer, accelerate, brake) with a third-person follow camera, or
2. Engage Auto Drive and watch the bus follow the route, reaching the destination and reporting route completion — then take over at any time.

The experience should feel calm and readable: clear HUD (speed, control mode, route progress), no clutter, no stutters caused by asset overload.

## 7. Target platforms

| Platform | Priority | Notes |
|---|---|---|
| Windows x86_64 | Primary target (MVP) | CI-built, packaged as MSI installer |
| Android (arm64 focus) | Primary target (MVP builds) | CI-built APK; touch controls are post-MVP |
| Linux | Development only (for now) | Dev runs on Zorin OS 18.1; no Linux build artifact required by CI at this stage |
| iOS, Web, Console | Out of scope | Not planned |

**Note (avoiding contradiction):** the MVP requires an Android *build* as a CI artifact and basic validation (boots, runs, acceptable performance), but **not** full mobile touch UI — touch controls are a post-MVP feature. The MVP Android build is validated with default/auto input mapping where applicable.

## 8. User stories

| ID | Story |
|---|---|
| US1 | As a player, I can steer, accelerate, and brake the bus so that I can drive it manually along a road. |
| US2 | As a player, I can switch from manual to Auto Drive so that the AI takes over driving. |
| US3 | As a player, I can take over from Auto Drive at any moment by using the driving controls, so that I always stay in charge. |
| US4 | As a player, I can switch back to Auto Drive so that the AI resumes following the route. |
| US5 | As a player, I can see my speed, current control mode, and route progress on a HUD so that I understand the game state. |
| US6 | As a player, I can see the route completed when the bus reaches the destination, so that I know the trip is finished. |
| US7 | As a player, I can adjust basic graphics settings so the game runs on low-end hardware. |
| US8 | As a player, I follow the bus with a third-person camera so that I can see the road and the vehicle. |
| US9 | As a developer, I can install a built MSI/APK on a different machine than the dev machine, so that testing is representative. |
| US10 | As a developer, I can identify which commit and build number produced an artifact, so that bugs can be traced. |

## 9. MVP definition

The MVP is the smallest coherent release proving the core concept: **one bus, one route, manual driving, Auto Drive.**

**In scope for MVP:**

1. One playable bus (original/low-poly placeholder-quality model is acceptable).
2. One basic 3D road scene (simple straight + curves sufficient).
3. Manual driving: steering, acceleration, braking (keyboard).
4. Basic vehicle physics: acceleration, braking, steering, friction, no toppling.
5. Third-person follow camera.
6. Waypoint-based route (at least one route, loaded from data).
7. Auto Drive stages 1–5 (waypoint following, steering control, speed control, braking, route completion — see [FEATURES.md](FEATURES.md)).
8. Manual ↔ Auto Drive switching: immediate takeover on manual input; **hold-to-confirm** to engage Auto Drive.
9. Route completion feedback (HUD/event).
10. Minimal HUD: speed, control mode, route progress.
11. Basic graphics quality setting (low/medium preset).
12. Start menu: a first screen with a start action leading to the driving scene.
13. CI pipeline: Windows build + MSI, Android APK, artifacts tagged with commit SHA/build number.

**Explicitly not in MVP:** bus stops behavior, passengers, traffic, traffic lights, intersections behavior, weather, day/night, multiple buses, gamepad, touch controls, save/load of progress, audio design beyond minimal placeholders (audio system exists architecturally; content is minimal).

## 10. Functional requirements

### 10.1 Manual driving requirements

| ID | Requirement | MVP |
|---|---|---|
| MR1 | The player can steer left/right with bounded steering input (−1..1). | Yes |
| MR2 | The player can apply throttle (0..1) and brake (0..1) independently. | Yes |
| MR3 | Input is processed as discrete drive commands fed to the vehicle controller (same path as Auto Drive). | Yes |
| MR4 | Input actions are defined in a configurable input map (remappable later). | Yes |
| MR5 | Manual input during Auto Drive triggers an immediate takeover (switch to manual mode). | Yes |
| MR6 | Gamepad and touch input are supported. | No (post-MVP) |

### 10.2 Vehicle requirements

| ID | Requirement | MVP |
|---|---|---|
| VR1 | Exactly one bus type in MVP. | Yes |
| VR2 | Vehicle physics: accelerate, brake to stop, steer, maintain stability (no flipping under normal driving). | Yes |
| VR3 | Speed is measurable and exposed as read-only state (for HUD and Auto Drive). | Yes |
| VR4 | Physics parameters (mass, power, brake strength, steering limits) are data/tunable, not hard-coded into Auto Drive. | Yes |
| VR5 | Vehicle exposes position, velocity, speed, heading as a read-only snapshot for consumers (Auto Drive, camera, UI). | Yes |
| VR6 | Advanced physics (tyre wear, suspension realism, damage) | No (future) |

### 10.3 Route requirements

| ID | Requirement | MVP |
|---|---|---|
| RR1 | A route is an ordered list of waypoints with 3D positions. | Yes |
| RR2 | Waypoints carry optional properties: type (default/stop), target speed. | Yes |
| RR3 | Routes are stored as data files (JSON), not embedded in code. | Yes |
| RR4 | Routes are validated on load (non-empty, monotonic positions, valid types); invalid routes are rejected with a clear error and never crash the game. | Yes |
| RR5 | Route runner tracks the current waypoint index and advances on arrival. | Yes |
| RR6 | Route completion is detected and announced as an event. | Yes |
| RR7 | Multiple routes, route selection UI, route editor tooling. | No (post-MVP) |
| RR8 | Missing/invalid waypoint at runtime is handled gracefully (error event + fallback behavior). | Yes |

### 10.4 Auto Drive requirements

Detailed staged breakdown in [FEATURES.md](FEATURES.md). Summary:

| ID | Requirement | MVP |
|---|---|---|
| AR1 | Auto Drive can be **engaged by holding** the engagement action for a confirmation duration (hold-to-confirm); a short press cancels. It disengages immediately on manual takeover (MR5) or on a press of the engagement action while active. | Yes |
| AR2 | When enabled, Auto Drive steers, throttles, and brakes to follow the active route (stages 1–5). | Yes |
| AR3 | Auto Drive produces `DriveCommand` values through the same driving interface as manual input; it does not access vehicle physics internals. | Yes |
| AR4 | Auto Drive reads vehicle state only through the read-only vehicle snapshot. | Yes |
| AR5 | Auto Drive reads route context from the route runner (current waypoint, target speed). | Yes |
| AR6 | Manual input takes over immediately from Auto Drive (AR/mr5). | Yes |
| AR7 | Reactivating Auto Drive resumes route following from the current route position. | Yes |
| AR8 | Route completion: Auto Drive stops the bus at the final waypoint and route-complete state is entered. | Yes |
| AR9 | Auto Drive gracefully handles degenerate states (missing waypoint, invalid route) by reporting an error instead of producing garbage commands. | Yes |
| AR10 | Bus-stop dwell, intersections, traffic awareness, traffic lights, stuck recovery. | No (stages 6–10, post-MVP) |

### 10.5 Camera requirements

| ID | Requirement | MVP |
|---|---|---|
| CR1 | Third-person follow camera trails the bus with smoothing. | Yes |
| CR2 | Camera follows via a read-only target reference; it does not modify vehicle state. | Yes |
| CR3 | Camera works in both manual and Auto Drive modes without mode-specific logic. | Yes |
| CR4 | Additional camera modes (interior, cinematic), camera cycling. | No (post-MVP) |

### 10.6 UI requirements

| ID | Requirement | MVP |
|---|---|---|
| UR1 | HUD shows: current speed (km/h display), active control mode (MANUAL/AUTO), route progress (e.g., waypoint n of N). | Yes |
| UR2 | Route completion is visibly indicated. | Yes |
| UR3 | Engaging Auto Drive requires holding the engagement action (hold-to-confirm, AR1); the active control mode is always visible on the HUD. | Yes |
| UR4 | Basic settings screen with graphics quality preset (low/medium) and reset option. | Yes |
| UR5 | Start menu shown at launch, with a start action that enters the driving scene. | Yes (minimal) |
| UR6 | Rich menus, map selection UI, pause menu options, localization, touch control overlays. | No (post-MVP/future) |

### 10.7 Save/configuration requirements

| ID | Requirement | MVP |
|---|---|---|
| SC1 | Graphics and input settings persist between sessions (JSON in user data dir). | Yes |
| SC2 | Missing/corrupt settings file falls back to defaults without crashing. | Yes |
| SC3 | Saving/loading of game progress (position, route state). | No (future) |

## 11. Non-functional requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | Frame rate on Intel HD 4600 @ 1366×768, low preset | ≥ 30 FPS sustained in MVP scene |
| NFR-2 | Memory footprint of the game process | < 1 GB (target: < 500 MB) |
| NFR-3 | APK size | < 150 MB for MVP |
| NFR-4 | Startup to drivable state | < 15 s on dev-class hardware |
| NFR-5 | Physics/logic determinism of route logic | Route runner must be testable headless without rendering |
| NFR-6 | Build reproducibility | Any `main` commit builds on CI without dev-machine state |
| NFR-7 | Art budget | Low-poly models, texture atlases, no texture larger than 1024² in MVP |
| NFR-8 | Dependency count | Engine + pinned test/lint tooling only; no other runtime dependencies without a recorded decision |
| NFR-9 | Crash resistance | Invalid user data (routes, settings) must produce an error state, not a crash |

## 12. Accessibility considerations

- All primary actions bindable to keyboard (input map, not hard-coded keys).
- HUD text readable at 1366×768 (minimum sensible font size; contrast against background).
- Graphics preset toggle for low-end hardware; resolution scaling option where feasible.
- No flashing/high-frequency visual effects (none in MVP by design).
- Full accessibility features (remapping UI, color-blind modes, subtitles) are **future** scope; the architecture must not preclude them.

## 13. Performance requirements

(See also NFR-1..NFR-8 and [TECH_STACK.md](TECH_STACK.md).)

- Compatibility renderer (OpenGL 3.3) is the baseline; no Vulkan requirement.
- Assume no dedicated GPU anywhere: dev machine, target test machine, and low-end users.
- Low-poly/optimized assets; texture atlasing for repeated world elements (road, props).
- Scalable graphics settings designed in from the start (quality presets), not retrofitted.
- Per-frame logic avoids allocations; Auto Drive computations are cheap math on snapshots (no physics queries per frame unless justified).
- Memory is budgeted from the beginning: no unbounded caches, no loading whole asset packs.

## 14. Future features (explicitly not MVP)

Bus stops with behavior (Auto Drive stage 6), passengers, intersections (stage 7), traffic (stage 8), traffic lights (stage 9), route error recovery (stage 10), multiple buses, improved vehicle physics, weather, day/night, map expansion, audio improvements, vehicle customization, save/load of progress, mobile touch controls, controller support, route editor tooling, multiple route selection.

Full roadmap with priorities: [FEATURES.md](FEATURES.md).

## 15. Risks

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | Auto Drive quality is poor (oscillating steering, missed waypoints) | Undermines flagship feature | Staged rollout (stages 1–5), dedicated test scenarios in TESTING.md, tunable parameters in data |
| R2 | Godot vehicle physics tuning takes longer than expected | MVP slip | Keep physics minimal; encapsulate behind `VehicleController` so implementation can be swapped |
| R3 | CI complexity (Godot headless export, Android SDK, MSI) consumes disproportionate time | Delayed builds | Documented pipeline design (CI_CD.md), incremental bring-up, caching |
| R4 | 4 GB dev machine limits editor multitasking (editor + Android SDK + browser) | Slow development | Keep project small, CI does heavy builds, avoid huge assets |
| R5 | Scope creep from "later features" list | Never ships MVP | MVP boundary in section 9; FEATURES.md priorities enforced |
| R6 | Accidental inclusion of proprietary code/assets | Legal/ethical violation | Original/licensed-only rule in README principles; review checklist in PR template |
| R7 | GitHub Actions minutes limits (private repo: Windows runners cost multiplier) | CI throttling | **Resolved:** repository is public → free unlimited Actions minutes; concurrency guard still cancels superseded runs |
| R8 | Single maintainer / bus factor | Stalls | Documentation-first approach, small changes, ADRs |

## 16. Open questions (resolved)

| ID | Question | Resolution (2026-10-01) |
|---|---|---|
| OQ1 | Will the GitHub repository be public (free unlimited Actions minutes) or private (limited quota)? | **Public** — Actions minutes are free; concurrency guard retained. |
| OQ2 | Minimum Android version and target ABIs (arm64 only vs universal APK)? | **Android 7.0 (API 24)+, `arm64-v8a` only** — smallest APK, covers the 64-bit Vivo test phone; recorded for the Android export preset at scaffold time (see note below). |
| OQ3 | Who creates original art/branding (bus model, logo, icons), and what style? | **Project owner** creates all original branding/art; no third-party assets in the repo. |
| OQ4 | Windows code signing for the MSI (unsigned is acceptable for personal use?) | **No code signing** — unsigned MSI accepted (personal use; SmartScreen warning documented). |
| OQ5 | Android release keystore: create now and store as GitHub secret, or debug-only keystore for MVP? | **Debug-only signing** — personal project, no store publishing. CI generates a debug keystore (or uses the optional `ANDROID_KEYSTORE_B64` secret for a stable signature). |
| OQ6 | Versioning scheme (semver starting 0.1.0?) and where the single version source lives. | **No semantic versions.** Build identity = commit SHA + GitHub run number. MSI technical product version = `0.0.<run_number>` (installer requirement only). |
| OQ7 | Test machine details (Windows laptop? Android device model/API level?) | **Android Vivo phone** is the primary test device; Windows builds validated on any available Windows machine. |
| OQ8 | Auto Drive re-engagement UX: instant toggle vs hold-to-confirm? | **Hold-to-confirm** to engage; manual takeover remains immediate (AR1, MR5). |
| OQ9 | Should the MVP launch directly into the driving scene or show a start menu? | **Start menu kept** (UR5, MVP item 12). |

**Note on OQ2 (recorded for the export preset):**

- **Minimum Android version** = the oldest Android version the APK will install on. Higher minimum = simpler/leaner, but excludes old phones. Godot 4's default is **API 24 (Android 7.0)**, which covers effectively all phones from ~2017 onward — including any modern Vivo.
- **Target ABI** = CPU architecture the APK is compiled for. Options: `arm64-v8a` (modern 64-bit phones — nearly all phones since ~2017, almost certainly your Vivo), `armeabi-v7a` (older 32-bit phones), `x86_64` (emulators/Chromebooks). More architectures = larger APK.
- **Decision (OQ2, confirmed 2026-10-01):** min API 24 + `arm64-v8a` only. Revisit only if the Vivo reports **32-bit** (*Settings → About phone*) — then add `armeabi-v7a` to the preset.

## 17. Acceptance criteria (MVP)

The MVP is accepted when all of the following are demonstrably true:

| ID | Criterion |
|---|---|
| AC1 | On Windows (CI-built MSI install), the player can drive the bus manually: steer, accelerate, brake, and come to a stop. |
| AC2 | The follow camera keeps the bus in view in third person while driving manually. |
| AC3 | Engaging Auto Drive makes the bus follow the loaded waypoint route through curves without leaving the road in the designated test scenarios (straight, gentle curve, sharp curve). |
| AC4 | Auto Drive stops the bus at the final waypoint and the UI indicates route completion. |
| AC5 | Manual input while Auto Drive is active immediately returns control to the player; engaging Auto Drive requires the hold-to-confirm action (AR1), and once engaged it resumes route following. |
| AC6 | HUD shows speed, control mode (MANUAL/AUTO), and route progress at all times during driving. |
| AC7 | An invalid/missing route file produces a visible error and no crash. |
| AC8 | On the dev machine (Intel HD 4600, 1366×768, low preset), the MVP scene sustains ≥ 30 FPS. |
| AC9 | Every push to `main` produces retained Windows and Android artifacts named with commit SHA and build number; `build-info.json` matches the triggering commit. |
| AC10 | The Android APK installs and launches on a test device/emulator, boots to the scene, and runs without crashing. |
| AC11 | Automated tests for route logic and Auto Drive core math pass headless in CI. |
| AC12 | All automated tests and lint checks pass in CI on `main`. |
| AC13 | No proprietary code or assets are present in the repository. |

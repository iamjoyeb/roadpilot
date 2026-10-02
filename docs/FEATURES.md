# Feature Roadmap — RoadPilot

| Field | Value |
|---|---|
| Project | RoadPilot |
| Status | Active — planning complete (2026-10-01); living document |
| Last updated | 2026-10-01 |
| Related | [PRD.md](PRD.md), [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) |

**Flagship feature: Auto Drive** (staged, Section 2).

**Status values:** `Not started` · `In progress` · `Done` · `Deferred`
**Complexity:** `S` small (localized, well-understood) · `M` medium (multi-module, some tuning) · `L` large (new subsystem or heavy tuning) · `XL` research-heavy/tuning-dominated
**Priority:** `P0` MVP-blocking · `P1` post-MVP important · `P2` future · `P3` experimental/optional

All features are `Not started` at the time of writing (documentation-only phase).

---

## 1. Feature lists by tier

### 1.1 MVP (P0)

| Feature | Description | Priority | Dependencies | Complexity | Acceptance criteria (summary) | Status |
|---|---|---|---|---|---|---|
| **Core game skeleton** | Root scene, EventBus, GameState (LOADING, DRIVING, ROUTE_COMPLETE, PAUSED), control-mode arbiter | P0 | Engine setup | M | Game boots to a drivable scene headless and in editor; events flow between subsystems | Not started |
| **Manual driving** | Keyboard steering/throttle/brake via InputMap → `DriveCommand` | P0 | Core, vehicle | S | PRD MR1–MR5; drive and stop the bus reliably | Not started |
| **Basic vehicle physics** | RigidBody3D + raycast layer: accelerate, brake, steer, stable, data-tunable | P0 | Core | L | PRD VR1–VR5; no flipping in normal driving; parameters editable without touching Auto Drive | Not started |
| **Bus model (placeholder)** | Original low-poly bus, GLB, single mesh/material set | P0 | Asset creation (OQ3) | S | Visible bus, reasonable poly/texture budget (NFR-7) | Not started |
| **Basic 3D road scene** | Original low-poly road: straight segment + curves, ground, light | P0 | World module, assets | M | Drivable loop for test routes; atlased road textures | Not started |
| **Follow camera** | Third-person SpringArm3D-based follow with smoothing | P0 | Vehicle snapshot | S | PRD CR1–CR3; bus stays framed at speed | Not started |
| **Route system (data)** | JSON route schema + loader + validation | P0 | Routes module | M | PRD RR1–RR4; invalid routes rejected with error, no crash (scenario T11) | Not started |
| **Route runner** | Progress tracking, arrival radius, events, completion | P0 | Route loader | S | PRD RR5–RR8; `waypoint_reached`/`route_completed` fire correctly | Not started |
| **Auto Drive stages 1–5** | Waypoint following, steering, speed, braking, completion (Section 2) | P0 | Routes, vehicle snapshot, drive interface | L | Auto Drive stages table, Section 2; PRD AR1–AR9; test scenarios T1–T10 pass | Not started |
| **Manual ↔ Auto switching** | Hold-to-confirm engage + takeover-on-input via control-mode arbiter | P0 | Core, input, auto_drive | S | PRD MR5, AR1, AR6, AR7; hold-to-confirm engage; takeover ≤1 tick; resume works | Not started |
| **Minimal HUD** | Speed (km/h), control mode, route progress, completion notice | P0 | Snapshots, events | S | PRD UR1–UR3; readable at 1366×768 | Not started |
| **Settings (basic)** | Graphics preset low/medium + persistence with defaults fallback | P0 | Config module | S | PRD UR4, SC1–SC2; survives restart | Not started |
| **Start menu (minimal)** | First screen with a start action → driving scene (PRD UR5) | P0 | UI module | S | Launching enters the driving scene; works on Windows + Android | Not started |
| **CI pipeline** | Push-to-main: lint, headless tests, Windows+MSI, Android APK, SHA metadata (`.github/workflows/build.yml` exists) | P0 | Repo, export presets | M | CI_CD.md; PRD AC9–AC12 | **Done** — all four stages green on every push (runs 3–4); GUT stage activates when `tests/` lands (AC11/AC12) |
| **MSI installer** | WiX MSI from Windows export (step present in workflow) | P0 | Windows export | M | Installs/runs on a non-dev Windows machine (AC1) | **In progress** — 31 MB MSI built by CI (run 4); clean-machine install test pending (TESTING §9) |
| **Android APK build** | Godot Android export on Linux runner, debug-signed (step present in workflow) | P0 | Export presets, SDK on runner | M | APK installs/boots on test device (AC10) | **In progress** — 28 MB APK built by CI (run 4); Vivo install test pending (TESTING §8) |

### 1.2 Post-MVP (P1)

| Feature | Description | Priority | Dependencies | Complexity | Acceptance criteria (summary) | Status |
|---|---|---|---|---|---|---|
| **Auto Drive stage 6: bus-stop behavior** | Dwell at stop waypoints, doors hook (with passenger system) | P1 | Stages 1–5, passenger feature | M | Bus stops precisely, waits, resumes | Not started |
| **Auto Drive stage 7: intersection behavior** | Yield/slow logic at intersection waypoints | P1 | Stage 5, world intersection data | M | Safe traversal without player intervention on test map | Not started |
| **Auto Drive stage 10: route error recovery** | Stuck/off-route detection, re-route or request takeover | P1 | Stages 1–5 | L | Stuck bus recovers or hands back cleanly; no oscillation | Not started |
| **Bus stops (world content)** | Stop markers, schedules | P1 | World, routes | M | Stops rendered and routed | Not started |
| **Passengers** | Spawn/board/alight counts, simple scoring | P1 | Bus stops, doors | L | Boarding cycle works end-to-end | Not started |
| **Route selection** | Menu listing `data/routes/*.json` to play | P1 | Loader, UI | S | Selecting a route loads and starts it | Not started |
| **Touch controls (Android)** | On-screen steering/pedals + auto-drive engage button | P1 | Input driver, UI | M | Fully drivable on phone/tablet | Not started |
| **Gamepad support** | Controller bindings through InputMap | P1 | Input | S | Playable with a gamepad | Not started |
| **Save/load (progress)** | Persist route/session progress via save subsystem | P1 | Save module, GameState | M | Progress restored after restart | Not started |
| **Audio pass** | Engine loop tied to RPM/speed, skid, UI/complete cues | P1 | Audio module, assets | M | Audible feedback without mix issues on low-end | Not started |
| **Graphics preset expansion** | High preset, finer toggles, resolution scale UI | P1 | Renderer settings | S | User-tunable within perf budget | Not started |
| **Vehicle tuning v2** | Better physics feel (grip curves, ABS-ish braking) | P1 | Vehicle module | L | Measurably better handling, still data-driven | Not started |

### 1.3 Future (P2)

| Feature | Description | Priority | Dependencies | Complexity | Acceptance criteria (summary) | Status |
|---|---|---|---|---|---|---|
| **Auto Drive stage 8: traffic awareness** | Sense nearby vehicles/obstacles, slow/stop/yield | P2 | World query interface, traffic | L | No collisions in scripted traffic scenario | Not started |
| **Auto Drive stage 9: traffic-light awareness** | Obey signal states at signal waypoints | P2 | Traffic light system | M | Correct stop/go at signals | Not started |
| **Traffic system** | AI vehicles on lanes/routes | P2 | Driver pattern, world | XL | Stable traffic without perf regression ≥30 FPS | Not started |
| **Traffic lights** | Signal cycles + world state exposure | P2 | World | M | Signals visible, state machine correct | Not started |
| **Multiple buses** | More bus models, selectable, optional AI-driven | P2 | Vehicle modularization | L | Second bus drivable/switchable | Not started |
| **Day/night + weather** | Lighting states, rain/fog | P2 | Renderer, effects | L | Toggleable, within low-end budget | Not started |
| **Map expansion** | Larger road network, districts | P2 | World streaming (maybe) | XL | New areas without memory blow-up (NFR-2) | Not started |
| **Vehicle customization** | Livery/color options | P2 | Assets, UI | M | Cosmetic change persists | Not started |
| **Route editor tooling** | In-game/editor tool to author routes | P2 | Routes, tools | L | Authoring a route without hand-editing JSON | Not started |
| **Localization** | UI string tables | P2 | UI | M | ≥2 languages switchable | Not started |
| **Linux build artifact** | Add Linux export to CI matrix | P2 | CI | S | Linux artifact retained like others | Not started |

### 1.4 Experimental (P3)

| Feature | Description | Priority | Dependencies | Complexity | Acceptance criteria (summary) | Status |
|---|---|---|---|---|---|---|
| **Alternative Auto Drive controllers** | Compare pure-purity vs. other steering controllers (e.g., lookahead variants) | P3 | Stage 2 interface | M | Can swap controller and measure route metrics | Not started |
| **Telemetry overlay** | Debug HUD: waypoint index, command values, errors | P3 | Auto Drive, UI | S | Toggleable debug info while driving | Not started |
| **Route metrics report** | Automated scoring: completion time, max deviation, smoothness | P3 | Scenario tests | M | Headless run produces a metrics summary | Not started |
| **Replay/ghost** | Record drive commands, replay for regression comparison | P3 | Drive interface | L | Deterministic-ish replay of a recorded route | Not started |
| **Shader/LOD experiments** | Cheap visual upgrades tested on HD 4600 | P3 | Renderer | M | Either adopted within budget or discarded | Not started |

---

## 2. Auto Drive — staged breakdown (flagship)

Each stage builds on the previous; **Stages 1–5 are MVP (P0)**, **6–10 are post-MVP/future**. All stages share the same architecture: the agent reads `VehicleState` + `RouteContext` and emits `DriveCommand` (SYSTEM_DESIGN §5.4). Stage logic is internal to `src/auto_drive/` — the vehicle and UI never change to accommodate a new stage.

| Stage | Name | Description | Priority | Depends on | Complexity | Acceptance criteria | Status |
|---|---|---|---|---|---|---|---|
| 1 | **Basic waypoint following** | Advance through the ordered waypoint list using an arrival radius; neutral commands otherwise | P0 | Route runner, snapshot | S | Correct index advancement in headless tests; arrival radius respected | Not started |
| 2 | **Steering control** | Steer toward a look-ahead point on the active waypoint path; bounded, smooth steering (no oscillation) | P0 | Stage 1 | M | Traverses gentle and sharp curves of test route within road bounds (T2, T3) | Not started |
| 3 | **Speed control** | Regulate speed toward waypoint `target_speed` / route default using throttle | P0 | Stage 1, snapshot speed | M | Reaches and holds target speed; accelerates from stop (T1, T5) | Not started |
| 4 | **Braking** | Decelerate before curves and stop waypoints; brake to a full stop when target speed is 0 | P0 | Stages 2–3 | M | Stops within tolerance of stop waypoints; no overshoot off road (T4, T6) | Not started |
| 5 | **Route completion** | On final waypoint: full stop, emit completion, Auto Drive idles with neutral commands | P0 | Stages 1–4 | S | `route_completed` fires once; UI shows completion; bus stationary (T7) | Not started |
| 6 | **Bus-stop behavior** | Dwell at stop waypoints for a configured time, then resume; (later) door/passenger interaction | P1 | Stage 5, bus stops | M | Dwell timing correct; resumes automatically; matches schedule data | Not started |
| 7 | **Intersection behavior** | Slow/stop and proceed rules at intersection waypoints (simplified right-of-way) | P1 | Stage 4, world intersection data | M | Crosses intersections safely in scripted scenarios without collision | Not started |
| 8 | **Traffic awareness** | Perceive nearby dynamic obstacles via world queries; slow/stop/yield behind them | P2 | World query interface | L | Maintains headway; no collision in scripted traffic test | Not started |
| 9 | **Traffic-light awareness** | Read signal state at signal waypoints; stop on red, proceed on green | P2 | Traffic light system, stage 7 | M | Correct behavior over full signal cycle | Not started |
| 10 | **Recovery from route errors** | Detect stuck/off-route/degenerate states; attempt recovery (rejoin nearest waypoint) or escalate to manual takeover | P1 | Stages 1–5 | XL | Stuck scenario (T12) recovers or requests takeover cleanly; no command garbage; faults reported via `auto_drive_fault` | Not started |

**Cross-stage rules (apply to every stage):**

- A stage never bypasses `DriveCommand`; it only computes values for it.
- A stage only reads sanctioned snapshots/contexts — no vehicle internals, no per-frame physics queries unless the stage's design explicitly budgets one (stages 8+).
- Enabling/disabling stages is configuration, so partial capability sets remain testable (stage gating supports headless scenario tests).
- Degenerate input (missing waypoint, invalid route) → neutral command + `auto_drive_fault` (never invalid values).

---

## 3. Milestone mapping

| Milestone | Contents |
|---|---|
| **M0 — Planning** (current) | Documentation set in `/docs`; decisions recorded; no code |
| **M1 — Drivable** | Core skeleton, manual driving, vehicle physics, follow camera, basic road scene |
| **M2 — Routable** | Route JSON + loader + validation, route runner, HUD progress, events |
| **M3 — Auto (MVP complete)** | Auto Drive stages 1–5, switching/takeover, settings, all PRD acceptance criteria met |
| **M4 — Shippable builds** | CI pipeline green on `main`: lint, tests, Windows+MSI, Android APK with SHA/build metadata |
| **M5+ — Post-MVP** | Stages 6/10, bus stops, passengers, touch/gamepad, save/load, audio pass (Section 1.2) |

Milestones M1–M4 are sequenced so that each is independently demoable; CI (M4) can start in parallel once M1 exists, per [CI_CD.md](CI_CD.md) bring-up notes.

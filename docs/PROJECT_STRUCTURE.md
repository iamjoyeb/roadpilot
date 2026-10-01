# Project Structure — RoadPilot

| Field | Value |
|---|---|
| Project | RoadPilot |
| Status | Active — scaffold landed 2026-10-01 (structure in effect; new directories still appear as their first real file lands) |
| Last updated | 2026-10-01 |
| Related | [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md), [TECH_STACK.md](TECH_STACK.md) |

---

## 1. Design rules for the layout

1. **Organized by domain, not by type.** There is no giant `scripts/` folder. Each subsystem owns its own directory with its scripts and scenes together, so a change to "routes" touches `src/routes/` (and only its declared dependencies).
2. **Runtime code vs. data vs. assets vs. tooling are physically separate.** Code in `src/`, game data in `data/`, media in `assets/`, automation in `tools/`, tests in `tests/`.
3. **The Godot project root is the repository root** (`project.godot` at top level). This keeps CI export commands, addon installs, and relative paths simple.
4. **Directories are created lazily** — a directory appears when its first real file lands, not in advance as an empty shell.
5. **Third-party code is quarantined** in `addons/` with its license documented; first-party code never lives there.

## 2. Proposed directory tree

```
roadpilot/
├── README.md                     # Project overview, principles, AI coding rules
├── LICENSE                       # MIT
├── .gitignore                    # .godot/, build/, export credentials, OS noise
│
├── docs/                         # All planning & architecture documentation
│   ├── PRD.md                    # Product requirements, MVP, acceptance criteria
│   ├── SYSTEM_DESIGN.md          # Architecture, subsystems, data flow
│   ├── PROJECT_STRUCTURE.md      # This file
│   ├── TECH_STACK.md             # Technology choices + rationale
│   ├── FEATURES.md               # Roadmap: MVP / post-MVP / future / experimental
│   ├── DEVELOPMENT_WORKFLOW.md   # Git + AI-assisted workflow
│   ├── CI_CD.md                  # GitHub Actions pipeline design
│   ├── TESTING.md                # Test strategy and Auto Drive scenarios
│   └── DECISIONS.md              # Architecture Decision Records (ADRs)
│
├── project.godot                 # Godot 4.7.2 project file (scaffolded: input map, main scene)
├── export_presets.cfg            # Export presets: Windows Desktop x64, Android arm64 (scaffolded)
│
├── addons/                       # Third-party Godot addons only (pinned, licensed)
│   └── gut/                      # GUT test framework (when tests are added)
│
├── src/                          # First-party runtime source, organized by domain
│   ├── core/                     # Bootstrap, EventBus, GameState, ControlMode arbiter
│   │   ├── core.tscn             #   root scene wiring subsystems together
│   │   └── *.gd
│   ├── vehicle/                  # Vehicle physics, VehicleController, VehicleState,
│   │   ├── bus.tscn              #   vehicle tuning data, bus scene
│   │   └── *.gd
│   ├── input/                    # ManualInputDriver, action mapping helpers
│   ├── auto_drive/               # AutoDriveAgent + per-stage logic (stages 1–10)
│   ├── routes/                   # Route/Waypoint models, RouteLoader, RouteRunner
│   ├── world/                    # Map scene, road mesh, static environment
│   │   └── world.tscn
│   ├── camera/                   # FollowCamera (SpringArm3D-based)
│   ├── ui/                       # HUD, settings screen, route-complete overlay
│   │   ├── hud.tscn
│   │   └── *.gd
│   ├── audio/                    # Audio manager, event-driven playback
│   ├── save/                     # Save/load subsystem (post-MVP for progress)
│   └── config/                   # Settings model + JSON persistence
│
├── assets/                       # Original/licensed media only
│   ├── models/                   # Low-poly GLB/GLTF meshes (bus, road pieces, props)
│   ├── textures/                 # Atlases and textures (≤1024² in MVP)
│   ├── audio/                    # SFX and loops (OGG/WAV)
│   └── fonts/                    # UI fonts
│
├── data/                         # Versioned game data (JSON)
│   └── routes/                   # route_*.json waypoint routes
│
├── tests/                        # Automated tests (run headless in CI)
│   ├── unit/                     # Pure logic: routes, auto_drive math, config
│   ├── integration/              # Combinations: loader→runner→agent, control mode
│   └── scenarios/                # Auto Drive drive-scenarios (headless sim runs)
│
├── tools/                        # Developer/CI helper scripts (bash/python)
│   └── (e.g., export helpers, data validation, atlas checks)
│
└── .github/
    ├── workflows/
    │   └── build.yml             # Push-to-main pipeline (preflight → lint/tests →
    │                             #   Windows+MSI, Android); builds activate when
    │                             #   project.godot + export_presets.cfg exist
    └── PULL_REQUEST_TEMPLATE.md  # PR checklist (docs updated, tests, no proprietary assets)
```

## 3. Directory responsibilities

| Directory | Responsibility | Key contents |
|---|---|---|
| `docs/` | Single source of truth for planning/architecture. Updated alongside architectural code changes. | The 9 documents listed above |
| `src/core/` | Game bootstrap and cross-cutting contracts: root scene, event bus, game state, control-mode arbitration. Knows about everyone; owned by no feature. | `core.tscn`, event bus, `GameState`, `ControlMode` |
| `src/vehicle/` | Everything about *this bus's physics*: rigid body setup, raycast vehicle layer, `VehicleController` (DriveCommand consumer), `VehicleState` producer, tuning data, bus scene. **Must not** import auto_drive/input/ui. | `bus.tscn`, controller, physics, snapshot |
| `src/input/` | Translating device actions into `DriveCommand` + intents (engage auto-drive, takeover). Deadzones/normalization live here. | `ManualInputDriver` |
| `src/auto_drive/` | The AI driver: consumes `VehicleState` + `RouteContext`, emits `DriveCommand`. One module per stage's logic as stages land (steering controller, speed controller, stop logic, later recovery, etc.). | `AutoDriveAgent`, stage logic |
| `src/routes/` | Route data model, JSON loading/validation, `RouteRunner` progress tracking, route events. No knowledge of who follows the route. | loader, runner, models |
| `src/world/` | The map: road mesh, ground, static props, lighting for the scene. Future: traffic signals, obstacles (via read-only queries). | `world.tscn` |
| `src/camera/` | Third-person follow camera. Reads the vehicle snapshot/transform only. | follow camera script |
| `src/ui/` | HUD (speed, control mode, route progress), settings screen, completion overlay. View + intents only. | `hud.tscn`, menus |
| `src/audio/` | Event-driven playback and bus (volume) routing. MVP has placeholder content. | audio manager |
| `src/save/` | Serialization of game progress (post-MVP) to `user://`. Owns file format. | save API |
| `src/config/` | Settings model + persistence (`user://settings.json`) with defaults fallback. | settings model |
| `assets/` | All media. Original or properly licensed only. Kept small: low-poly, atlased, compressed. | GLB, PNG/OGG, fonts |
| `data/` | Editable game data versioned in git (routes). Untrusted at runtime → validated on load. | route JSON |
| `tests/` | Automated tests mirroring `src/` domains; run headless locally and in CI. | GUT tests |
| `tools/` | Repo scripts for building/validating (export helpers, data checks). Executable documentation for repeatable dev tasks. | shell/python scripts |
| `addons/` | Pinned third-party Godot addons (test framework). Never first-party code. | GUT |
| `.github/` | CI workflow definitions and PR template. | `workflows/` |

## 4. Scene organization

- Scenes live **next to the scripts of their domain** (`src/vehicle/bus.tscn`, `src/ui/hud.tscn`), not in a global `scenes/` folder — co-location keeps a domain's change surface in one place.
- The root scene (`src/core/core.tscn`) composes the world, bus, camera, and UI at the top level; subsystems are wired there or via their own scenes.
- One playable scene for MVP; menu/other scenes added only when needed.

## 5. Naming conventions

| Kind | Convention | Examples |
|---|---|---|
| Directories | `snake_case` | `auto_drive/`, `src/routes/` (domain areas are written `auto_drive` on disk, referred to as "Auto Drive" in prose) |
| Script files | `snake_case.gd`, named after the principal class | `vehicle_controller.gd`, `route_runner.gd` |
| Scene files | `snake_case.tscn`, named after the root node's role | `bus.tscn`, `hud.tscn`, `core.tscn` |
| Classes | `PascalCase`, declared with `class_name` when reusable across files | `VehicleController`, `RouteRunner`, `DriveCommand` |
| Functions/variables | `snake_case` | `current_waypoint_index`, `apply_command()` |
| Constants/enums | `SCREAMING_SNAKE_CASE` for true constants; `PascalCase` enum types with `PascalCase` members | `ARRIVAL_RADIUS`, `ControlMode.AUTO` |
| Signals | past-tense / occurrence names | `waypoint_reached`, `route_completed`, `control_mode_changed` |
| Input actions | `snake_case` verbs | `steer_left`, `throttle`, `brake`, `auto_drive_engage` |
| Data files | `snake_case.json` with stable `id` field | `route_demo_01.json` |
| Tests | `test_*.gd` under `tests/<level>/`, named for behavior | `test_route_validation.gd` |
| Git branches | `feature/<short-name>`, `fix/<short-name>`, `chore/<short-name>` | `feature/auto-drive-steering` |
| Commits | imperative mood, scoped subject | `vehicle: bound steering input` |

## 6. File-count discipline

- Prefer **one principal class per script file**; split only when a file becomes hard to review (~a few hundred lines is a smell, not a rule).
- Avoid "helper dumping grounds" (`utils.gd`, `misc.gd`). A helper is placed with the domain that owns its meaning; a genuinely shared helper goes to `src/core/` only after a second real consumer exists.
- Any third-party file retains its original license file inside `addons/<name>/`.

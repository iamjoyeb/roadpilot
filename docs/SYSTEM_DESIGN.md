# System Design — RoadPilot

| Field | Value |
|---|---|
| Project | RoadPilot |
| Status | Draft — planning phase |
| Last updated | 2026-10-01 |
| Engine | Godot 4.7.2 (GDScript) |
| Related | [PRD.md](PRD.md), [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md), [TECH_STACK.md](TECH_STACK.md), [DECISIONS.md](DECISIONS.md) |

> **Scope of this document:** architecture only. It describes subsystems, responsibilities, interfaces, and data flows. It intentionally contains no implementation code.

---

## 1. Architecture overview

RoadPilot is organized as a set of domain subsystems around a small core. Subsystems communicate through **narrow, defined interfaces** (direct calls for per-frame data) and an **event bus** (for discrete, cross-cutting events). Dependencies point in one direction; nothing reaches "downward" into physics internals.

The central idea of the whole architecture:

```
Player Input ────────┐
                     ├──> Drive Command ──> Vehicle Controller ──> Vehicle Physics
Auto Drive AI ────────┘
```

Both the player and the AI produce the **same artifact** — a `DriveCommand` — and a single `VehicleController` applies it. Auto Drive never touches vehicle internals, and the vehicle never knows who is driving.

### 1.1 Subsystem map

```
                        ┌───────────────────────────────┐
                        │            CORE               │
                        │  bootstrap · EventBus ·       │
                        │  GameState · ControlMode      │
                        └──────┬───────────────┬────────┘
                               │ (used by all) │
        ┌──────────────────────┼───────────────┼──────────────────────┐
        │                      │               │                      │
        ▼                      ▼               ▼                      ▼
  ┌──────────┐          ┌───────────┐   ┌──────────────┐        ┌──────────┐
  │  INPUT   │─command─▶│  VEHICLE  │◀──│  AUTO DRIVE  │─command│   UI     │
  │ (manual) │          │ controller│   │   (AI driver)│        │ (HUD etc)│
  └──────────┘          │  physics  │   └──────┬───────┘        └────┬─────┘
                        └─────┬─────┘          │ reads               │ observes
                              │                ▼                     │ (snapshots
                              │          ┌─────────────┐             │  + events)
                              │          │   ROUTES    │─────────────┘
                              │          │ waypoints · │
                              │          │ route runner│
                              │          └──────┬──────┘
                              │                 │ world queries (future)
                              ▼                 ▼
                        ┌───────────────────────────────┐
                        │             WORLD             │
                        │  map · roads · environment    │
                        └───────────────────────────────┘

  Cross-cutting: CAMERA (follows vehicle read-only) · AUDIO (reacts to events)
                 SAVE/CONFIG (persists settings) · TESTS/TOOLS/CI (non-runtime)
```

## 2. Major subsystems and responsibilities

| Subsystem | Directory | Responsibility | Owns state? |
|---|---|---|---|
| **Core** | `src/core/` | Game bootstrap, event bus, high-level game state, control-mode arbitration (MANUAL/AUTO) | Game state, control mode |
| **Vehicle** | `src/vehicle/` | Vehicle physics (RigidBody3D + raycast layer), `VehicleController` applying `DriveCommand`, read-only `VehicleState` snapshot | Vehicle physics state |
| **Input** | `src/input/` | Reads Godot InputMap actions and produces `DriveCommand` + control intents (engage auto-drive, takeover) | Raw device state (transient) |
| **Auto Drive** | `src/auto_drive/` | AI driver: consumes route context + vehicle snapshot, emits `DriveCommand`. Staged capability set (see FEATURES.md) | Its own internal stage/state only |
| **Routes** | `src/routes/` | Waypoint/route data model, JSON loading + validation, `RouteRunner` tracking progress, route events | Active route + progress index |
| **World** | `src/world/` | Map, road mesh, static environment, (future) traffic signals/obstacles | World/scene graph |
| **Camera** | `src/camera/` | Third-person follow camera; follows a read-only target reference | Camera pose (transient) |
| **UI** | `src/ui/` | HUD, settings screen, route-complete overlay; renders state, emits intents | View state only |
| **Audio** | `src/audio/` | Audio stream playback via Godot audio buses; reacts to events | Playback state |
| **Save** | `src/save/` | Persist/load game data (post-MVP for progress) | Save files |
| **Configuration** | `src/config/` | Settings model (graphics, input), load/save with defaults fallback | Settings |

Non-runtime areas: `tests/`, `tools/`, `assets/`, `data/`, `docs/` (see [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md)).

## 3. Core interfaces and data contracts

These are the contracts that keep subsystems decoupled. Names are proposed; exact signatures are finalized at implementation, but **the shape of these contracts is an architectural decision** (see DECISIONS.md ADR-005).

### 3.1 DriveCommand

The single command structure flowing from any driver to the vehicle.

| Field | Type | Range | Meaning |
|---|---|---|---|
| `steer` | float | −1..1 | Left/right steering demand |
| `throttle` | float | 0..1 | Acceleration demand |
| `brake` | float | 0..1 | Braking demand |

Reserved for later (not in MVP, but the structure is designed to grow): `handbrake`, `horn`, `doors`, `lights` — added only when a feature needs them.

`DriveCommand` is **produced by exactly two producers**: the manual input provider and the Auto Drive agent. It is **consumed by exactly one consumer**: `VehicleController`.

### 3.2 VehicleState (read-only snapshot)

Produced by the vehicle, consumed by Auto Drive, camera, and UI. Contains at minimum:

- `position`, `heading` (basis), `velocity`, `speed` (scalar, m/s)
- Later: wheel contact/surface info (only when a consumer needs it)

Consumers receive a snapshot value; they never hold a reference into the physics body and never write to it.

### 3.3 RouteContext (read-only)

Produced by the route runner, consumed by Auto Drive (and UI for progress display):

- Current waypoint index, total count, active route id/name
- Current target waypoint (position + type + `target_speed` in m/s)
- Route status: `ACTIVE` | `COMPLETED` | `FAILED`

### 3.4 Driver interface (conceptual)

Both manual and auto driving implement the same conceptual role:

```
Driver
 ├── ManualInputDriver : reads InputMap → DriveCommand + intents
 └── AutoDriveAgent    : reads VehicleState + RouteContext → DriveCommand
```

The `ControlMode` arbiter (core) decides **whose command is forwarded** to `VehicleController` this tick:

```
ControlMode = MANUAL → forward ManualInputDriver's DriveCommand
ControlMode = AUTO   → forward AutoDriveAgent's DriveCommand
```

**Decoupling rule (the key architectural constraint):** `auto_drive/` may depend on `routes/` and on the *read-only* `VehicleState` contract. It must **not** import vehicle physics classes, access the rigid body, or read private vehicle nodes. If Auto Drive needs new information from the vehicle, the vehicle must expose it through `VehicleState` (or a dedicated read-only query), making the dependency explicit and reviewable.

### 3.5 EventBus (discrete events)

A small set of cross-cutting signals, owned by `core`:

| Event | Emitted by | Consumed by (examples) |
|---|---|---|
| `control_mode_changed(from, to)` | Core | UI (HUD indicator), Audio |
| `route_loaded(route_id)` | Routes | UI, Audio |
| `waypoint_reached(index, waypoint)` | Route runner | Auto Drive, UI |
| `route_completed(route_id)` | Route runner | UI, Audio, game state |
| `route_failed(reason)` | Route runner / loader | UI |
| `auto_drive_fault(reason)` | Auto Drive (degenerate state) | UI, core |

**Rule:** the event bus carries *discrete, low-frequency* events only. Per-frame values (speed, steering) are **not** broadcast as events; consumers read snapshots directly. This avoids signal-spam and keeps frame cost predictable on low-end hardware.

## 4. Dependency direction

Allowed dependencies (arrow = "may depend on"):

```
ui          ──▶ core (events, state reads), config
input       ──▶ core (intents), vehicle (DriveCommand type only)
auto_drive  ──▶ core, routes, vehicle (VehicleState + DriveCommand types only)
routes      ──▶ core (events)          [no dependency on vehicle or auto_drive]
vehicle     ──▶ core                   [no dependency on input, auto_drive, ui]
camera      ──▶ vehicle (read-only transform/snapshot)
audio       ──▶ core (events)
world       ──▶ (none beyond engine)
config/save ──▶ (none beyond engine)
core        ──▶ engine only
```

**Forbidden dependencies (explicit):**

- `vehicle` → `auto_drive`, `input`, `ui` (the vehicle must not know who drives it)
- `auto_drive` → vehicle physics internals (only the snapshot/contracts)
- `routes` → `auto_drive` (routes are data; the AI consumes them, not the reverse)
- `ui` → `vehicle` internals or `auto_drive` internals (UI observes state/events only)
- Any subsystem → another subsystem's private nodes by hard-coded path (use contracts/events)

This is the rule an AI coding agent must check before adding any `load()`/`preload()`/type reference across domains.

## 5. Runtime flow

### 5.1 Frame overview

```
                (render frame)
┌──────────────────────────────────────────────────────────┐
│ 1. Poll device input (buffered; consumed by the next      │
│    physics tick's command collection + UI intents)        │
│ 2. UI update from snapshots/events                       │
│ 3. Camera follow update                                  │
│ 4. Render                                               │
└──────────────────────────────────────────────────────────┘
                (fixed physics tick, 60 Hz)
┌──────────────────────────────────────────────────────────┐
│ 1. Collect DriveCommand (manual OR auto, per ControlMode)│
│ 2. VehicleController applies command → physics forces    │
│ 3. Physics integrates (GodotPhysics3D)                   │
│ 4. VehicleState snapshot refreshed                       │
│ 5. RouteRunner checks waypoint arrival                   │
│ 6. Auto Drive (if AUTO) computes next DriveCommand       │
│    from refreshed snapshot + route context               │
└──────────────────────────────────────────────────────────┘
```

Design rules:

- **All driving decisions run on the fixed physics tick** (60 Hz), never on render frames — consistent behavior regardless of FPS, which also makes Auto Drive testable headless.
- One `DriveCommand` is produced and applied **per physics tick**.
- Order per tick: *collect → apply → integrate → refresh state → advance route → compute next command*. Auto Drive always works from the last completed state (no feedback loop within one tick).

### 5.2 Input flow (manual)

```
Device → InputMap actions (steer_left/right, throttle, brake,
        auto_drive_engage, pause) → ManualInputDriver
        → DriveCommand (values) + intents (engage, takeover)
```

- Values are normalized (deadzones handled here, not in the vehicle).
- A **takeover intent** is raised when a driving action is pressed while `ControlMode == AUTO`.
- Input is remappable through the InputMap (MR4) — no hard-coded key codes in logic.

### 5.3 Manual driving flow

```
ManualInputDriver ──DriveCommand──▶ ControlMode arbiter (MANUAL)
                                    ──▶ VehicleController
                                        ──▶ wheel/steering + throttle/brake
                                            forces on RigidBody3D
                                        ──▶ GodotPhysics3D integrates
                                        ──▶ VehicleState refreshed
                                             ├─▶ UI (speed)
                                             └─▶ Camera (follow target)
```

### 5.4 Auto Drive flow

```
RouteRunner ──RouteContext──┐
                            ├──▶ AutoDriveAgent ──DriveCommand──▶ Arbiter (AUTO)
Vehicle ◀──VehicleState─────┘                                     │
  (snapshot)                                                     ▼
                                                          VehicleController
                                                          (identical path
                                                           as manual)
```

Steps per physics tick while `AUTO`:

1. Read `RouteContext` (current waypoint, type, target speed).
2. Read `VehicleState` (position, heading, speed).
3. Compute command: steering toward a look-ahead point, throttle/brake toward target speed and upcoming waypoint constraints (stages 2–4).
4. Emit `DriveCommand`; arbiter forwards it; vehicle applies it exactly as it would a human's.

Degenerate states (no active route, out-of-range waypoint index) → emit `auto_drive_fault(reason)` and emit **zero commands** (neutral), never garbage values.

**Takeover:** any driving input while `AUTO` → core sets `ControlMode = MANUAL` immediately and emits `control_mode_changed`. **Re-engage (hold-to-confirm, OQ8 resolved):** the player must **hold** the `auto_drive_engage` action for a confirmation duration (tunable constant) to set `ControlMode = AUTO`; a short press cancels. Once engaged, Auto Drive resumes from current route progress (no reset of the bus). Pressing `auto_drive_engage` while active disengages immediately — and no button press is ever *required* for takeover: manual driving input always wins instantly (PRD AR1/MR5/AR7).

### 5.5 Route / waypoint flow

```
data/routes/*.json ──▶ RouteLoader ──validate──▶ Route (model)
                                                  │
                                            RouteRunner
                                                  │ progress index
                            events ───────────────┼──────────────▶ UI/Audio/Core
                            RouteContext ─────────┴──▶ Auto Drive
```

- **Arrival rule (MVP):** advance when horizontal distance to current waypoint < arrival radius (tunable per route); emit `waypoint_reached`; on last waypoint → `route_completed` and status `COMPLETED`.
- **Validation on load:** non-empty waypoint list, well-formed positions, known waypoint types, speed ranges. Failure → `route_failed(reason)`, no crash (NFR-9, AC7).
- **Runtime gap:** if the runner is asked for a waypoint beyond the list, it clamps/flags `route_failed` rather than returning null positions.

Proposed route JSON shape (proposal — finalized at implementation):

```json
{
  "id": "route_demo_01",
  "name": "Depot Loop",
  "waypoints": [
    { "position": [0.0, 0.0, -10.0], "type": "default", "target_speed": 8.0 },
    { "position": [0.0, 0.0, -60.0], "type": "stop",   "target_speed": 0.0 }
  ]
}
```

`target_speed` in m/s; HUD displays km/h (conversion at UI layer only).

### 5.6 Vehicle physics flow

```
DriveCommand ──▶ VehicleController ──▶ steering demand → front wheel/steer angle
                                      throttle demand  → forward force (speed-limited)
                                      brake demand     → opposing force (to stop)
                GodotPhysics3D (RigidBody3D) integrates motion
                ──▶ VehicleState snapshot
```

- Implementation is a **lightweight raycast vehicle layer** on `RigidBody3D` (suspension/contact rays + simplified tire response), started as simple as possible and tuned by data (VR4).
- Encapsulated entirely behind `VehicleController`. Swapping the physics implementation (e.g., to Godot's `VehicleBody3D`, or a kinematic model) requires **no changes** to input, Auto Drive, routes, or UI — this is why the interface exists (DECISIONS.md ADR-003).
- Physics parameters live in resource/data files under the vehicle's own domain, not in Auto Drive.

### 5.7 Camera architecture

```
VehicleState.position/basis ──(read-only)──▶ FollowCamera (SpringArm3D-based)
                                             smoothing lag → camera transform
```

- No mode-specific logic: identical behavior under MANUAL and AUTO.
- Never writes to the vehicle. Loose coupling means the camera keeps working if the vehicle implementation changes.

### 5.8 UI architecture

```
Snapshots (VehicleState, RouteContext) ──read──▶ UI (HUD, overlays)
Events (EventBus) ────────────────────────read──▶
User actions (button/key) ──intents──▶ Core (engage Auto Drive, settings changes)
```

- UI is a **view**: it renders state and forwards intents. It contains no driving logic and never calls physics.
- Control-mode toggling goes through core (`ControlMode` arbiter), not directly to Auto Drive or the vehicle.
- Godot `Control` nodes with anchors/responses; one scalable layout target (1366×768 baseline) (TECH_STACK.md).

### 5.9 Audio architecture

- MVP: minimal placeholder content (engine loop, events like route complete) — the **system** exists, content is thin.
- Audio reacts to **events** (`route_completed`, `control_mode_changed`) and may read a snapshot for engine pitch; it drives nothing.
- Mixing via Godot audio buses (Master/SFX/Music) so volume settings fit into config later.

### 5.10 Save / configuration architecture

- **Config (MVP):** settings model (graphics preset, later volumes/keybinds) serialized as JSON under Godot's per-user data directory (`user://`). Load: missing/corrupt file → defaults + log warning (SC2).
- **Save (post-MVP):** game progress serialized through the same save subsystem; UI/game core write, save subsystem owns file format.
- Neither subsystem is imported by vehicle/auto_drive — settings are consumed via core/config only.

## 6. State management

| State | Owner | Notes |
|---|---|---|
| Physics state (transform, velocity) | Vehicle | Only vehicle writes |
| `DriveCommand` producers' state | Input / Auto Drive | Each owns its own internals |
| Control mode (MANUAL/AUTO) | Core (`ControlMode` arbiter) | Single source of truth; all others read it |
| Route + progress | Route runner | Only route runner mutates index/status |
| Game phase (LOADING, DRIVING, ROUTE_COMPLETE, PAUSED) | Core `GameState` | Small explicit state set; UI renders it |
| Settings | Config | Written on change; read by UI/render setup |
| World geometry | World | Static in MVP |

**Principle:** every piece of mutable state has exactly one writer. Everyone else reads snapshots or listens to events. This is what prevents the "who changed this?" class of bugs in AI-assisted development.

## 7. Event/message communication strategy

1. **Per-frame data → direct snapshot reads / direct calls.** No events for speed, steering, transforms.
2. **Discrete, cross-cutting occurrences → EventBus signals.** Route progress, control-mode changes, faults.
3. **Within a subsystem → direct calls** (private helpers don't need indirection).
4. **New event?** Add it to the core event contract, document it here, keep payload small (IDs/enums/reason strings, not scene objects).
5. Signals use past-tense/occurrence names (`waypoint_reached`, `route_completed`, `control_mode_changed`).

## 8. Data ownership

| Data | File location | Written by | Read by |
|---|---|---|---|
| Routes | `data/routes/*.json` | Designers/tools (repo) | Route loader → runner → auto_drive, UI |
| Vehicle tuning | vehicle domain data (resource/data file) | Vehicle domain | Vehicle controller/physics |
| Settings | `user://settings.json` | Config subsystem (via UI intents) | Config, UI, render setup |
| Saves (future) | `user://saves/*.json` | Save subsystem | Save subsystem, core |
| Assets | `assets/**` | Asset pipeline (repo) | World/vehicle/audio/ui |
| Build metadata | generated `build-info.json` artifact | CI | Testers/support |

Repo data files are versioned; runtime files (`user://`) are not.

## 9. Error handling

| Error class | Handling |
|---|---|
| Invalid route JSON / schema | Loader rejects → `route_failed(reason)` → UI message; game stays running (AC7) |
| Route index out of range at runtime | Runner flags `route_failed`; Auto Drive goes neutral and emits `auto_drive_fault` |
| Auto Drive degenerate state (no route, NaN guard) | Neutral command + `auto_drive_fault`; never emit invalid values |
| Corrupt settings file | Fall back to defaults + log (SC2) |
| Missing asset/resource | Fail loudly in dev (visible error), graceful message in build; never silent null-deref |
| Physics instability (e.g., flipped bus) | MVP: manual reset/recovery not required; detected stuck/off-route handling is Auto Drive stage 10 (post-MVP) |
| CI failure | Pipeline fails; no artifacts published from a failed run; logs retained (see CI_CD.md) |

**Principle:** user-generated data (routes, settings) is untrusted input; engine-authored content failures are development-time bugs. Neither may hard-crash the game.

## 10. Performance considerations

Constraints: Intel HD 4600, 1366×768, 4 GB RAM machine (NFR-1, NFR-2).

- **Renderer:** Compatibility (OpenGL 3.3); no Vulkan dependency. Quality presets (low/medium) with toggles for shadows, draw distance, resolution scale.
- **Assets:** low-poly models, texture atlases for road/props, ≤1024² textures in MVP, minimal unique materials.
- **Frame budget:** driving logic is simple math on snapshots; no physics queries per frame in Auto Drive MVP stages (world queries arrive with stage 8+ and will be explicitly budgeted).
- **Allocation discipline:** reuse command/snapshot objects; no per-frame array/dictionary creation in hot paths.
- **Events:** discrete only (Section 7) — avoids signal storms.
- **Memory:** single scene, no streaming in MVP; unbounded caches forbidden (NFR-2).
- **Testable performance:** headless route/Auto Drive logic must not require rendering (NFR-5), so tests run cheaply on CI and on the 4 GB dev machine.

## 11. Extensibility strategy

Design for evolution without speculative machinery:

1. **New driver sources** (traffic AI vehicles later): implement the same `Driver`/DriveCommand pattern; each vehicle has its own arbiter instance. No Auto Drive changes required for other vehicles.
2. **Auto Drive stages:** capabilities are added as internal strategies behind one agent entry point; stage toggles are data/config (FEATURES.md stages 1–10). UI/vehicle unaffected.
3. **New waypoint types** (`stop`, `intersection`, `signal`): extend waypoint `type` enum + loader validation; route runner and Auto Drive check types they understand and ignore the rest until their stage exists.
4. **New commands** (handbrake/doors/horn): grow `DriveCommand` additively with neutral defaults; existing producers keep compiling.
5. **World growth** (traffic, lights): world subsystem exposes read-only queries/services; Auto Drive stage 8/9 consumes them through an interface, still without vehicle coupling.
6. **Platforms:** new export targets are CI/export-preset concerns, not architecture changes.

The test for any extension: *does it add a new producer/consumer of an existing contract, or a new contract?* Prefer the former; new contracts require a DECISIONS.md entry.

## 12. What deliberately is NOT in this design

- No ECS framework, no custom service locator beyond a couple of autoloads (core event bus, config), no DI container.
- No networking/multiplayer layer.
- No plugin system or mod-loading.
- No generic "behavior tree framework" until a stage actually needs one (stages 6–10 revisit this).

Kept intentionally boring so that the AI driving feature — the actual experiment — gets the attention.

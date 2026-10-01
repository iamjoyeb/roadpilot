# Testing Strategy — RoadPilot

| Field | Value |
|---|---|
| Project | RoadPilot |
| Status | Draft — planning phase |
| Last updated | 2026-10-01 |
| Related | [PRD.md](PRD.md), [FEATURES.md](FEATURES.md), [CI_CD.md](CI_CD.md), [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) |

---

## 1. Testing principles

1. **Headless-first logic.** Route logic, Auto Drive math, config, and validation must run **without rendering** (NFR-5) — that is what makes them testable in CI and on the 4 GB dev machine.
2. **Test the contracts, not the scenes.** Unit/integration tests target `DriveCommand`, `RouteContext`, `VehicleState` producers/consumers, and control-mode arbitration — the seams that keep subsystems decoupled.
3. **Auto Drive is the flagship ⇒ it gets the most scenario coverage** (Section 5).
4. **CI is the safety net; manual sessions are the truth.** Automated tests gate merges; human play sessions on real builds validate feel and performance.
5. **Test machines ≠ dev machine.** Builds must be validated on a second, different machine (Windows + Android) to catch dev-environment assumptions (PRD US9, AC1).

## 2. Test levels

### 2.1 Unit tests (automated, headless, CI)

Scope: pure logic with no scene tree.

| Area | Examples |
|---|---|
| Route validation | Empty route rejected; bad waypoint types rejected; speed clamping; malformed JSON → `route_failed` (never crash) — AC7 |
| Route runner | Arrival-radius advancement; index clamping; completion fires exactly once; out-of-range request → failure state (RR8) |
| Auto Drive math | Steering controller output within [−1..1]; speed controller converges; braking distance logic; neutral command on fault |
| Control mode | Takeover rule: driving input during AUTO → MANUAL; hold-to-confirm engage (short press cancels); press-while-active disengages; command source selection |
| Config | Defaults on missing/corrupt file; save/load round-trip (SC1–SC2) |
| Build metadata helper | `build-info.json` generation contains required fields (if implemented as a tool) |

Runner: **GUT**, invoked headless; runs on every PR and push to `main` (CI_CD §3.1).

### 2.2 Integration tests (automated, headless, CI)

Wiring between subsystems using lightweight/simulated doubles where scenes are unnecessary:

- Loader → Route model → RouteRunner → AutoDriveAgent command loop (headless tick simulation over a scripted route).
- ManualInputDriver (simulated actions) → arbiter → command-source assertions (AR6/AR7).
- Event sequence assertions: `route_loaded` → `waypoint_reached`* → `route_completed` order and single emission.
- Snapshot contract: a fake vehicle state feeding the agent produces bounded commands (no NaN, no out-of-range).

### 2.3 Auto Drive scenario tests (automated, headless, CI — Section 5)

Scripted routes run in a **headless physics simulation** with scripted assertions (bus stays within corridor, stops within tolerance, completes route). These are the regression net for the flagship feature.

### 2.4 Gameplay / manual tests (human, per milestone)

Exploratory sessions on built artifacts: driving feel, camera comfort, HUD readability, mode switching UX, obvious edge cases. Checklist in Section 6. Run before merging milestone-level PRs (M1–M4 in FEATURES §3) and after any vehicle/Auto Drive tuning change.

### 2.5 Build validation (automated + manual)

| Check | When | Pass criteria |
|---|---|---|
| Lint + format gate | Every PR/push | `gdlint`/`gdformat --check` clean (AC12) |
| Tests gate | Every PR/push | All suites green headless (AC11/AC12) |
| Artifact presence | Every push to `main` | Windows zip, MSI, APK all present (AC9) |
| Metadata match | Every push to `main` | `build-info.json` commit == triggering SHA (AC9) |
| Artifact integrity | On download | Non-zero size; expected structure (exe+pck / valid APK) |
| MSI install test | Per release candidate / milestone | Installs and launches on the **test Windows machine** (AC1) |
| APK install test | Per release candidate / milestone | Installs and launches on the **test Android device** (AC10) |

## 3. Physics tests

| Type | Approach |
|---|---|
| Headless smoke | Simulated bus: apply throttle N seconds → speed increases; apply brake → speed reaches 0; steering produces bounded yaw change (VR2) |
| Stability | Scripted maneuver (steer at speed) does not flip/launch the bus under normal parameters (VR2) |
| Parameter sanity | Tuning values within documented valid ranges (guards bad data commits) |
| Feel/tuning validation | **Manual only** — "believable" is a human judgment; done on the low-end machine and recorded in the PR |

Full physics accuracy validation is explicitly out of scope (PRD non-goals).

## 4. Route tests

- **Schema/load:** valid route loads; each invalid variant (empty, malformed positions, unknown type, negative speeds) → rejected with `route_failed` and no crash (RR4, AC7).
- **Progress:** correct index sequence on a known route; arrival radius boundary behavior; final waypoint → `route_completed` once (RR5/RR6).
- **Degenerate runtime:** requesting waypoint beyond list → failure state, agent neutral (RR8, AR9).
- **Data lint (tooling):** all `data/routes/*.json` in repo validate against the schema (CI step or `tools/` script).

## 5. Auto Drive test scenarios (flagship)

Scenario suite lives in `tests/scenarios/`. Each has: setup, stimulus, expected result, and level (A = automated headless, M = manual). **T1–T11 are required for MVP sign-off** (T12 becomes required only when Auto Drive stage 10 lands, post-MVP); stage mapping per FEATURES §2.

| ID | Scenario | Setup | Stimulus / steps | Expected result | Level | Stage |
|---|---|---|---|---|---|---|
| T1 | **Straight road** | Straight route, AUTO engaged from stop | Full run | Bus accelerates to target speed, stays centered in lane, no oscillation | A | 1–3 |
| T2 | **Gentle curve** | Route with wide-radius curve | Drive through curve | Speed maintained or slightly reduced; lateral deviation stays within road corridor; steering smooth | A | 2 |
| T3 | **Sharp curve** | Route with tight curve | Approach and traverse | Decelerates before curve (stage 4 interplay); completes curve within corridor; no spin/overshoot | A + M | 2, 4 |
| T4 | **Stop (waypoint)** | Route containing `target_speed: 0` stop waypoint | Approach stop | Decelerates to full stop at/near waypoint (within tolerance); holds until stage rules say resume (MVP: route may end there) | A | 4 |
| T5 | **Acceleration** | AUTO from standstill on straight | Observe speed profile | Monotonic speed increase to target; no jerky command chatter (bounded d(throttle)/tick) | A | 3 |
| T6 | **Braking** | AUTO at cruise speed, stop waypoint ahead | Braking segment | Speed reaches 0 before waypoint tolerance; brake command in range; no reverse creep | A | 4 |
| T7 | **Route completion** | Full short route | Run to end | `route_completed` emitted once; bus stationary; UI shows completion; agent commands neutral (AC4) | A + M | 5 |
| T8 | **Manual takeover** | AUTO active, bus moving | Player presses throttle/steer | Control switches to MANUAL **within one physics tick**; player command takes effect immediately; `control_mode_changed` emitted (AC5) | A + M | — |
| T9 | **Auto Drive reactivation** | MANUAL, mid-route | Hold the engagement action for the confirmation duration (hold-to-confirm) | AUTO engages, resumes from current route progress (no route reset); bus continues to next waypoint (AC5) | A + M | 1 |
| T10 | **Missing waypoint** | Route with a gap/removed waypoint at runtime (constructed test fixture) | Runner/agent access | `route_failed`/`auto_drive_fault` emitted; agent issues neutral commands; no crash; UI shows error (AC7) | A | — (PRD AR9) |
| T11 | **Invalid route** | Corrupt/empty route file | Attempt to load | Load rejected with clear error; game continues; no garbage state (AC7) | A | — |
| T12 | **Vehicle stuck** | Bus blocked/against obstacle on route (or wheels off surface) | Auto Drive continues attempting | **Post-MVP (stage 10):** detect stuck → recover by rejoining route or escalate to manual takeover with `auto_drive_fault`; no command garbage. **MVP:** not covered — player takes over manually (documented limitation) | A + M | 10 (post-MVP) |

**Scenario harness notes:**

- Automated scenarios run headless with a **simulated or minimal real vehicle** state, asserting on `DriveCommand` streams and position traces — not screenshots.
- Corridor/lane tolerance and stop tolerance are **named tunable constants** so tests and tuning share one source (documented in the test fixture).
- Every stage added later (6–9) ships with at least one new scenario in the same PR (ties FEATURES stages to coverage).

## 6. Manual testing checklist (per milestone build)

Run on **built artifacts** (not the editor):

1. Install via MSI / APK per Section 2.5.
2. Manual drive: steer left/right, accelerate, brake to stop; repeat at speed — no loss of control, no flipping.
3. Camera: bus framed correctly; no jitter at speed; works in both modes.
4. HUD: speed, MANUAL/AUTO mode, route progress update correctly.
5. Auto Drive: engage → follows route (T1–T3 corridor feel) → completes (T7).
6. Takeover (T8) and reactivation (T9) feel immediate and predictable.
7. Invalid route (T11) shows a readable error, game keeps running.
8. Settings: change preset, restart, setting persisted.
9. Quit/restart cleanly; no crash dialogs/errors in logs.
10. Notes recorded (machine, OS, build SHA from `build-info.json`).

## 7. Low-end hardware testing

The development machine **is** the low-end benchmark:

| Check | Target | How |
|---|---|---|
| Frame rate | ≥ 30 FPS sustained in MVP scene, 1366×768, **low** preset (NFR-1, AC8) | In-engine FPS counter during the manual checklist; note min/avg over a full route run |
| Memory | < 1 GB process footprint (target < 500 MB) (NFR-2) | OS process monitor during a session |
| Thermals/stability | No crash/throttle-induced freeze over a 15-minute session | Manual soak |
| Asset budget | Textures ≤ 1024², low-poly budgets (NFR-7) | `tools/` asset checks + human review |
| Startup | Drivable < 15 s (NFR-4) | Stopwatch on cold start |

Any perf-affecting PR (renderer, assets, per-frame work) re-runs this checklist before merge. Later, an automated FPS smoke (headless FPS proxy metrics) is *experimental*, not a gate.

## 8. Android testing

| Scope | MVP expectation |
|---|---|
| CI | APK exists, metadata correct, build succeeds every `main` push (AC9/AC10 pipeline-side) |
| Device smoke (per milestone) | Install, launch, reach scene, no crash within 5 minutes; orientation/render correctness; audio not required |
| Performance | Best-effort note on the test device (no fixed FPS gate for MVP on Android — device variability) |
| Touch controls | **Not tested in MVP** — post-MVP feature (PRD §7 note, FEATURES 1.2) |
| Coverage | `arm64-v8a`, min API 24 per OQ2 (PRD §16); **primary test device: the project's Vivo phone** (OQ7) |
| Signature | CI keystore is ephemeral unless the `ANDROID_KEYSTORE_B64` secret is set — if an install fails with a signature conflict, uninstall first (CI_CD §3.4) |

## 9. Windows testing

| Scope | Expectation |
|---|---|
| CI | Export + MSI succeed; artifacts retained with SHA metadata (AC9) |
| Clean-machine install | MSI installs on **any available Windows machine** (OQ7: primary test device is the Android phone; Windows validation is done opportunistically): launches, drivable, uninstall leaves no obvious residue (AC1) |
| Full checklist | Section 6 run on the test machine at each milestone; results noted with build SHA |
| SmartScreen | Expected warning (unsigned, OQ4) — document, not a defect |

## 10. Test ownership and cadence

| Test type | Runs when | Owner |
|---|---|---|
| Unit + integration + scenarios | Every PR / push to `main` (CI) | Author of the change |
| Build validation | Every push to `main` | CI |
| Manual checklist (Section 6) | Before merging milestone PRs; after vehicle/Auto Drive tuning | Author |
| Low-end perf (Section 7) | Milestones + perf-affecting changes | Author |
| Windows clean-machine + Android device | Per milestone build (M1–M4+): Android on the **Vivo test phone**, Windows on any available Windows machine | Author (human) |

**Definition of done for a feature:** code + tests (where logic exists) + docs updated + relevant manual checks on a **built artifact**, with the artifact's SHA recorded in the PR.

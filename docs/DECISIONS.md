# Architecture Decision Records — RoadPilot

| Field | Value |
|---|---|
| Project | RoadPilot |
| Status | Draft — planning phase (initial decision set) |
| Last updated | 2026-10-01 |

**Format:** each record uses Context / Decision / Alternatives / Reasoning / Consequences. Records are **append-only**: superseded decisions get a "Superseded by ADR-xxx" note; they are never silently rewritten.

**How AI agents must use this file:** consult the relevant ADR before changing anything it governs. Reversing or materially altering a decision requires a new ADR (or an explicit amendment here) in the same PR as the code change.

---

## ADR-001: Engine choice — Godot 4.7.2

**Status:** Accepted · **Date:** 2026-10-01

**Context.**
RoadPilot needs 3D driving, Windows + Android exports, CI-friendly headless builds, and must run comfortably while being developed on a 4 GB RAM / Intel HD 4600 / Linux (Zorin OS 18.1) machine. The project is personal, MIT-licensed, long-lived, and AI-assisted. The engine decision constrains everything downstream (language, physics, CI, tooling).

**Decision.**
Use **Godot Engine, exact version 4.7.2 (stable)** — pinned for the editor, export templates, and CI alike.

**Alternatives considered.**
- **Unity (6.x line):** strong ecosystem and Android/Windows tooling, but: Linux editor is second-class; CI requires game-ci Docker images plus license activation (heavy on hosted runners and on a 4 GB machine); editor RAM/disk footprint far larger; proprietary licensing with a history of term changes; C# adds compile steps. Rejected as disproportionately heavy for this scope.
- **Unreal Engine:** far beyond hardware and scope (GPU assumptions, huge builds). Rejected.
- **Custom/engine-less (OpenGL/Vulkan by hand):** would reinvent scene, physics, input, audio, and both platform exporters — directly contradicts PRD non-goals. Rejected.
- **Godot 3.x:** older 3D/CI ergonomics, no future. Rejected in favor of the current 4.x line.

**Reasoning.**
- Official Linux editor → native fit for the dev machine.
- MIT/open source → no vendor account, no revenue gates, self-hostable forever; ideal for an indefinitely-lived hobby project.
- `--headless` CLI is first-class → cheap GitHub Actions (single binary download + cached export templates, no multi-GB Docker or license steps).
- Cross-export to Windows from Linux CI, and Android export from Linux runners, both official.
- Compatibility (OpenGL 3.3) renderer treats Intel HD 4600 as a first-class target.
- Small, readable project format + GDScript → well-suited to small, reviewable AI-generated diffs.
- 4.7.2 chosen as the latest stable maintenance release of the current feature line at project start; dev/RC builds (4.8-dev) explicitly excluded.

**Consequences.**
- ✅ Low-friction CI and dev loop; minimal RAM pressure; full control via MIT.
- ✅ Exports for both required targets from hosted runners.
- ⚠️ 3D ceiling below Unity/Unreal (lighting, large-world tech) — acceptable: low-end-first art direction.
- ⚠️ Must stay disciplined about version pinning and stable-only upgrades across dev/CI/templates.
- ⚠️ GDScript performance limits → hot paths must stay allocation-light (baked into SYSTEM_DESIGN §10).

---

## ADR-002: Language — GDScript (typed)

**Status:** Accepted · **Date:** 2026-10-01

**Context.**
Godot supports GDScript, C# (.NET), and (via extensions) other languages. The team is AI-assisted, the machine has 4 GB RAM, and iteration speed matters more than raw language performance.

**Decision.**
Use **GDScript** with type hints on public APIs (arguments, returns, exported/public fields), enforced style via `gdformat` + `gdlint`.

**Alternatives considered.**
- **C# / .NET:** stronger typing and tooling, but adds a compile step, larger runtime/templates, heavier CI, and slower iteration for no current requirement. Godot allows mixed projects later if a specific subsystem justifies it.
- **Rust/C++ GDExtension:** max performance, but build complexity and toolchain burden are anti-goals for MVP.
- **Untyped GDScript:** fastest to write, but silent type errors and weaker AI-review signal; rejected in favor of typed-by-default on contracts.

**Reasoning.**
Native, zero-build, fast headless execution for tests; readable diffs; linters/formatters exist headless for CI; keeps dependency count at engine-only at runtime (NFR-8).

**Consequences.**
- ✅ Fastest iteration path; trivially lintable; contracts (`DriveCommand`, `VehicleState`, …) stay explicit.
- ⚠️ Weaker static guarantees than C# → mitigated by type hints + tests on logic.
- ⚠️ If a hot spot ever needs more speed, the escape hatch (C#/GDExtension) requires an ADR — not a default.

---

## ADR-003: Physics approach — Godot built-in physics with a raycast vehicle layer behind `VehicleController`

**Status:** Accepted · **Date:** 2026-10-01

**Context.**
The bus needs believable, controllable driving physics for manual control and Auto Drive, on low-end hardware, headlessly testable. Godot 4 no longer ships Bullet. Simplicity and swappability matter more than simulation fidelity (PRD non-goal).

**Decision.**
Use **Godot's built-in physics (GodotPhysics3D)** with a **lightweight raycast vehicle layer** (contact/suspension rays + simplified tire response) built on `RigidBody3D`. All of it is encapsulated behind the **`VehicleController`** interface, which is the sole consumer of `DriveCommand` and the sole producer of `VehicleState`. Physics parameters are data-tunable (VR4).

**Alternatives considered.**
- **`VehicleBody3D` (built-in raycast vehicle):** quicker start, but less control over tuning and behavior; retained as a possible *internal* swap later — the interface makes this a local change.
- **Full tire-model physics (e.g., external libraries / Pacejka-style):** over-engineered for MVP; adds dependencies (violates NFR-8 spirit) and tuning cost.
- **Kinematic-only (no rigid body):** simplest and most deterministic, but loses believable mass/inertia/suspension behavior; rejected for feel, still available as a swap option behind the same interface.
- **Third-party physics engine:** unnecessary dependency for this scope.

**Reasoning.**
The interface — not the physics flavor — is the architectural guarantee. Starting simple keeps M1 ("Drivable") achievable, while `VehicleController` isolation guarantees Auto Drive/input/UI are untouched if the internals change (their contracts are snapshots + commands).

**Consequences.**
- ✅ No external physics dependency; headless-testable; tunable via data.
- ✅ Any future physics upgrade is contained inside `src/vehicle/`.
- ⚠️ Physics "feel" will need tuning iterations (risk R2) — mitigated by data-driven parameters and manual feel checks (TESTING §3).
- ⚠️ Raycast models can misbehave off-road; acceptable for MVP's defined test routes.

---

## ADR-004: Route representation — ordered waypoints in versioned JSON files

**Status:** Accepted · **Date:** 2026-10-01

**Context.**
Routes are the data backbone of Auto Drive. They must be editable without touching code, diffable in git, validated at runtime (untrusted input), and consumable headlessly by tests.

**Decision.**
Represent routes as **ordered waypoint lists in JSON files** under `data/routes/*.json` (schema proposal in SYSTEM_DESIGN §5.5: position, `type`, `target_speed`, plus optional per-type fields). Loaded by a `RouteLoader` that **validates** and converts into an in-memory `Route` model consumed by the `RouteRunner`.

**Alternatives considered.**
- **Godot resources (`.tres`):** engine-native, but harder to edit/diff outside the editor, weaker tool-agnostic story, and awkward for AI/test tooling.
- **Hard-coded routes in scripts:** fastest for one route, but violates data/code separation and blocks route selection tooling later.
- **CSV / custom binary:** no benefit at this scale; binary hurts diffability.
- **GIS/imported maps (OpenDRIVE etc.):** powerful but heavyweight overkill for one small map; revisit only with a map-expansion decision.

**Reasoning.**
JSON is tool-agnostic (hand-edit, scripts, AI, tests), git-diff friendly, self-describing, and cheap to parse at MVP scale. Validation-at-load gives AC7 (invalid route ⇒ error, no crash) a single enforcement point. Future editors (FEATURES: route editor tooling) can emit the same schema.

**Consequences.**
- ✅ Routes are testable and validated in one place; easy for `tools/` to lint all route files in CI.
- ✅ Schema evolution is explicit (adding waypoint types/stages 6–9 fields is additive).
- ⚠️ Schema must be versioned/validated carefully — silent schema changes are forbidden (never silently change public interfaces); changes require loader + tests + docs in the same PR.
- ⚠️ No visual route editing in MVP → hand-authored JSON is fine for one route.

---

## ADR-005: Auto Drive architecture — driver pattern over a single `DriveCommand` interface

**Status:** Accepted · **Date:** 2026-10-01

**Context.**
The flagship requirement: Auto Drive must control steering/throttle/braking through the **same interface** as manual input, and must **not** be coupled to the bus. Player takeover must be immediate. The AI must be stage-able (10 stages) and testable headless.

**Decision.**
Model both controllers as **drivers** producing a `DriveCommand` (`steer` ∈ [−1,1], `throttle` ∈ [0,1], `brake` ∈ [0,1]):

- `ManualInputDriver` (input subsystem) and `AutoDriveAgent` (auto_drive subsystem) each produce exactly one command per **physics tick**.
- A **control-mode arbiter in `core`** (`MANUAL` / `AUTO`) forwards only the active driver's command to `VehicleController`.
- Auto Drive reads only sanctioned read-only contracts: **`VehicleState`** (vehicle snapshot) and **`RouteContext`** (route runner). It has no access to physics internals; the vehicle has no knowledge of either driver.
- Takeover: any driving input while `AUTO` ⇒ arbiter switches to `MANUAL` within one tick; **engagement (OQ8 resolved, 2026-10-01) requires hold-to-confirm** — the player holds the engagement action for a confirmation duration to enter `AUTO`; a short press cancels; once engaged, Auto Drive resumes route following from current progress.
- Degenerate states ⇒ neutral command + `auto_drive_fault` event (never invalid values).

**Alternatives considered.**
- **Auto Drive writes directly into the vehicle (sets forces/steering on the rigid body):** fastest to prototype, but exactly the forbidden coupling — untestable in isolation, fragile to vehicle changes. Rejected.
- **Full behavior-tree/utility framework from day one:** premature for stages 1–5; adds a dependency/concept cost before it pays off. Deferred (revisit at stages 6–10 if warranted).
- **Event-bus-driven commands** (auto drive publishes commands as signals): harder to reason about per-tick ordering and testing than a direct arbiter call; events reserved for discrete notifications (SYSTEM_DESIGN §7).
- **Two separate control paths** (manual path + AI path applying physics differently): violates the requirement that AI uses the player's interface. Rejected.

**Reasoning.**
The arbiter gives exactly one writer of control per tick (matches "single writer" state principle), makes takeover a trivial state flip, and lets both drivers be unit-tested with fake snapshots/commands. Stage gating stays *inside* the agent, invisible to the rest of the game.

**Consequences.**
- ✅ Auto Drive ↔ vehicle decoupling is structural, not a convention — enforced by dependency rules (SYSTEM_DESIGN §4).
- ✅ Testable: agent logic runs headless against scripted snapshots (TESTING §5).
- ✅ Reusable for future AI traffic vehicles (each vehicle has its own driver + arbiter).
- ⚠️ Two drivers must be kept behaviorally consistent in edge cases (deadzones, tick ordering) → documented in SYSTEM_DESIGN §5.1 and covered by T8/T9.
- ⚠️ Future commands (handbrake/doors/horn) grow `DriveCommand` additively — a deliberate, tested schema change.
- ⚠️ Takeover-policy UX is a tunable constant (confirmation duration) owned by input/core — changing it requires no subsystem redesign (adopted: hold-to-confirm, PRD AR1/UR3).

---

## ADR-006: CI/CD strategy — GitHub Actions, push-to-main artifacts, WiX MSI, default Android export

**Status:** Accepted · **Date:** 2026-10-01

**Context.**
Every push to `main` must yield retained Windows (eventually MSI) and Android APK artifacts, identifiable by commit SHA/build number, buildable without the dev machine, on hosted runners.

**Decision.**
- **GitHub Actions** with a pipeline: quality gates (lint + headless tests) → parallel jobs for **Windows export** (`windows-latest`) and **Android APK** (`ubuntu-latest`) → **MSI packaging via WiX Toolset** in the Windows job.
- **Godot headless export** using pinned 4.7.2 editor + export templates, cached.
- Android uses **default Godot export templates (no custom Gradle build)** with a debug keystore in MVP.
- Artifacts named with **short SHA + run number**, each containing `build-info.json`; failed runs publish nothing; artifact retention ~30 days. Build identity only — **no semantic versioning** (OQ6 resolved): MSI product version is `0.0.<run_number>`.
- The workflow **exists at** `.github/workflows/build.yml` (created 2026-10-01, ahead of the game code); [CI_CD.md](CI_CD.md) remains its specification, with a preflight gate that activates build stages as prerequisites (`project.godot`, `export_presets.cfg`) appear.

**Alternatives considered.**
- **Unity + game-ci:** multi-GB Docker images, license activation, heavier runners — rejected with the engine decision (ADR-001).
- **Self-hosted runner on the dev machine:** violates "builds must not depend on the dev machine"; device is also the perf benchmark and would be blocked by long builds. Rejected (could be a *supplement*, never a requirement).
- **Other CI (GitLab CI, Buildkite):** hosting is already GitHub; no benefit to splitting. Rejected.
- **MSI alternatives — Inno Setup:** produces .exe installers, not MSI (requirement is MSI). **MSIX:** signing/store expectations beyond scope. **AdvancedInstaller:** commercial. → **WiX** chosen: free, scriptable, headless on Windows runners.
- **Android release signing / custom Gradle:** unnecessary for MVP artifacts; rejected — OQ5 resolved: debug-only signing, no store publishing.
- **No MSI in MVP (zip only):** contradicts the stated MSI direction → MSI is in scope per requirements (staged: zip first if needed, MSI in same pipeline).

**Reasoning.**
Native fit with the repo host; Linux runners handle Godot Linux export and Android; Windows runner handles Windows export + WiX without Wine hacks; caching keeps ~1 GB templates cheap; two independent target jobs give partial signal even on failure.

**Consequences.**
- ✅ PRD AC9–AC12 mechanically satisfied; any tester machine can consume artifacts.
- ⚠️ Windows runner minutes multiply on private repos (risk R7 / OQ1) → concurrency cancellation + PR fast-gates only.
- ⚠️ Contract now concrete (implementation detail validated on first green run): WiX 5.0.2 with `<Files Include="build\windows\**"/>` harvesting; export preset names `Windows Desktop` / `Android`; keystore via `GODOT_ANDROID_KEYSTORE_DEBUG_*` env vars (no committed credentials).
- ⚠️ Debug-signed APK + unsigned MSI → platform warnings are expected and documented (OQ4 resolved: no code signing; OQ5 resolved: debug-only signing, optional `ANDROID_KEYSTORE_B64` secret for signature stability).
- ⚠️ Pipeline bring-up is staged (CI_CD §8) — guards keep `main` green while prerequisites are still missing.

---

## ADR-007: Repository workflow — `main`-trunk with feature branches and PRs

**Status:** Accepted · **Date:** 2026-10-01

**Context.**
Solo/personal project with AI-assisted contributions. Needs a stable integration point that always builds, plus a review habit that keeps changes small and traceable.

**Decision.**
- **`main` is the stable integration branch**, protected in practice: merges via PR (default), short-lived `feature/*`, `fix/*`, `chore/*` branches.
- **Every push to `main` triggers CI** with retained artifacts (ADR-006); PRs run fast gates (lint + tests).
- Squash-merge preferred for feature branches; commits use imperative, scoped subjects.
- AI agents follow the small-diff workflow in DEVELOPMENT_WORKFLOW §4 and README's AI coding principles.

**Alternatives considered.**
- **Commit-directly-to-main (trunk with no PRs):** fewer ceremony steps, but loses the review checklist that guards architecture/docs/tests — especially valuable for AI-generated changes. Rejected as default (trivial one-liners may still go direct by judgment).
- **Git-flow (develop branch, releases):** heavyweight ceremony for one contributor; adds a second integration branch to keep green. Rejected.
- **Long-lived per-feature branches with late merges:** merge-risk and divergence on a project that moves in small steps. Rejected.

**Reasoning.**
PRs are where the checklist (scope, docs, tests, no proprietary content, no silent API changes) is enforced; CI on `main` makes the integration branch's health objective; squash history stays legible.

**Consequences.**
- ✅ `main` always ships buildable, tested, SHA-stamped artifacts; easy bisect of regressions.
- ✅ Review habit bounds AI-agent blast radius (one domain per PR).
- ⚠️ Slight ceremony vs. direct pushes — acceptable; kept light for docs-only/trivial changes.
- ⚠️ Requires discipline on branch lifetime (rule of hours-to-days) to avoid a soft "develop" branch reappearing.

---

## ADR-008: Test framework — GUT, headless

**Status:** Accepted · **Date:** 2026-10-01

**Context.**
Route logic, Auto Drive math, arbitration, and config need automated coverage running headless in CI and on the 4 GB machine (NFR-5, AC11).

**Decision.**
Use **GUT (Godot Unit Test framework)**, pinned to a version compatible with Godot 4.7, installed as a Godot addon (`addons/gut/`), run headless in CI and locally. Tests live in `tests/{unit,integration,scenarios}`.

**Alternatives considered.**
- **gdUnit4:** comparable, well-maintained; viable fallback. Chose GUT for longevity and simpler assertion/API surface for AI-generated tests.
- **Wisetest/AutoTest-style recorders:** wrong fit for logic-first testing.
- **External harness (pytest + engine glue):** can't exercise Godot objects properly; rejected.
- **No framework (manual asserts in scripts):** no CI gating; rejected.

**Reasoning.**
Godot-native, addon-based (isolated in `addons/`), headless CI support, mature community. Matches the "engine + pinned tooling only" dependency rule (TECH_STACK §16).

**Consequences.**
- ✅ AC11 test gate implementable; scenario suite (TESTING §5) has a home.
- ⚠️ Scene-dependent tests stay thin — heavy coverage relies on headless logic tests (by design).
- ⚠️ Framework version pinning needed on each Godot upgrade; switching to gdUnit4 later is possible but is a decision, not a drive-by.

---

## ADR-009: Renderer — Compatibility (OpenGL 3.3), scalable presets

**Status:** Accepted · **Date:** 2026-10-01

**Context.**
Intel HD 4600 @ 1366×768 and low-end Android are first-class targets (NFR-1); the dev machine must stay responsive with 4 GB total RAM.

**Decision.**
Baseline renderer is Godot's **Compatibility** renderer (OpenGL 3.3), with **low/medium quality presets** (shadows, resolution scale, draw distance) persisted via config. Art direction: low-poly + texture atlases, textures ≤1024² in MVP.

**Alternatives considered.**
- **Forward+ / Mobile (Vulkan):** better high-end features, but Vulkan on Haswell-class integrated GPUs is unreliable and raises the floor — rejected as baseline; could become an *opt-in* preset later on capable hardware (never required).
- **Fixed single quality level:** simpler, but ignores the explicit "scalable graphics settings" requirement.

**Reasoning.**
Only Compatibility honestly targets the stated hardware everywhere (Windows/Linux/Android), keeps CI/dev identical to what users run, and makes the ≥30 FPS gate (AC8) plausible.

**Consequences.**
- ✅ Low-end is the *default*, so performance regressions surface on the dev machine immediately.
- ⚠️ Visual ceiling (no fancy lighting) — aligned with project scope.
- ⚠️ Quality-preset code must exist from early on so higher tiers can be added without rework (config-driven, not hard-coded branches).

---

## Decision index

| ADR | Decision | Governs |
|---|---|---|
| 001 | Godot 4.7.2 (stable, pinned) | Engine, all tooling |
| 002 | GDScript (typed) + gdtoolkit | All first-party code |
| 003 | Built-in physics + raycast layer behind `VehicleController` | `src/vehicle/` |
| 004 | JSON waypoint routes + validation at load | `src/routes/`, `data/routes/` |
| 005 | Driver pattern: `DriveCommand` + core arbiter | `src/auto_drive/`, `src/input/`, `src/core/`, contracts |
| 006 | GitHub Actions pipeline; WiX MSI; default Android export | CI/CD, packaging |
| 007 | `main`-trunk + feature branches + PRs | Repository workflow |
| 008 | GUT headless testing | `tests/` |
| 009 | Compatibility renderer + quality presets | Rendering, config, assets |

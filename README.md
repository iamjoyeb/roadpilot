# RoadPilot

A lightweight 3D bus-driving simulator with **Auto Drive** — an AI driver that can take control of the bus and follow predefined waypoint routes through the same control interface used by manual input.

## Current status

**Planning complete, scaffold + CI live.** The repository contains full documentation ([`docs/`](docs/)), the MIT license, a Godot **4.7.2** scaffold (`project.godot`, `export_presets.cfg`, boot scene `src/core/`), and a **green CI pipeline**: every push to `main` lints, runs headless checks, and produces retained **Windows folder (≈39 MB), MSI (≈31 MB), and Android APK (≈28 MB)** artifacts named with commit SHA + run number (see [`docs/CI_CD.md`](docs/CI_CD.md) for the bring-up log). Next phase: gameplay (M1 — world + vehicle), then GUT tests (CI's test stage activates when `tests/` lands).

## Vision

RoadPilot is a personal/experimental project that explores realistic-but-accessible bus driving in 3D, with automation as a first-class feature rather than an afterthought. It is designed from day one to run acceptably on low-end hardware (integrated graphics, 4 GB RAM class machines) and to be developed in small, reviewable steps.

## Core feature: Auto Drive

Auto Drive is the flagship feature. The player can switch between **manual control** and **Auto Drive** at any time. Auto Drive follows a waypoint-based route — steering, accelerating, braking, and completing the route — by emitting the exact same kind of drive commands as manual input. It never talks to vehicle internals directly; it is a *driver* plugged into the driving interface, not a modification of the vehicle.

```
Player Input ────────┐
                     ├──> Drive Command ──> Vehicle Controller ──> Vehicle Physics
Auto Drive AI ────────┘
```

## Technology decision

| Item | Choice |
|---|---|
| Engine | **Godot 4.7.2** (stable, exact version pinned) |
| Language | **GDScript** (typed) |
| Renderer | **Compatibility** (OpenGL 3.3) — first-class low-end target |
| Physics | Godot built-in physics with a lightweight raycast vehicle layer |
| CI/CD | GitHub Actions → Windows build + MSI, Android APK |

Rationale and alternatives are recorded in [`docs/DECISIONS.md`](docs/DECISIONS.md) and [`docs/TECH_STACK.md`](docs/TECH_STACK.md).

## Supported platforms

| Platform | Role |
|---|---|
| Linux (Zorin OS 18.1, x86_64) | Development platform |
| Windows x86_64 | Build target, packaged as MSI |
| Android (arm64 focus) | Build target, APK artifact |

Builds are produced by CI and are not dependent on the development machine. The dev machine (i5-4200M, Intel HD 4600, 4 GB RAM) is treated as a **first-class low-end target**, not just an authoring environment.

## Development workflow (intended)

- `main` is the stable integration branch.
- Work happens on short-lived **feature branches**, merged via **pull requests**.
- Every push to `main` triggers CI, which builds and retains **Windows** and **Android** artifacts.
- Every artifact carries the **Git commit SHA** and **build (run) number**.
- AI-assisted changes must be small and reviewable — see [AI coding principles](#ai-coding-principles).

Details: [`docs/DEVELOPMENT_WORKFLOW.md`](docs/DEVELOPMENT_WORKFLOW.md), [`docs/CI_CD.md`](docs/CI_CD.md).

## Repository structure (proposed)

```
roadpilot/
├── README.md
├── LICENSE
├── docs/                  # All planning & architecture documentation
├── project.godot          # Godot 4.7.2 config: main scene, input map, renderer
├── export_presets.cfg     # Export presets: Windows x64, Android arm64 (API 24+)
├── .gitignore             # Ignores .godot/, build/, export credentials
├── addons/                # Pinned third-party Godot addons (test framework only)
├── src/                   # Source code, organized by domain
│   ├── core/              # Bootstrap, events, game state
│   ├── vehicle/           # Vehicle physics & drive controller
│   ├── input/             # Manual input providers
│   ├── auto_drive/        # Auto Drive AI driver
│   ├── routes/            # Waypoints, route loading, route runner
│   ├── world/             # Map, roads, environment
│   ├── camera/            # Follow camera
│   ├── ui/                # HUD and menus
│   ├── audio/             # Sound playback & buses
│   ├── save/              # Save/load
│   └── config/            # Settings/configuration
├── assets/                # Models, textures, audio, fonts (low-poly/optimized)
├── data/                  # JSON game data (routes, etc.)
├── tests/                 # Automated tests
├── tools/                 # Build/dev helper scripts
└── .github/workflows/     # CI pipeline (build.yml: lint/tests + Windows/MSI + Android)
```

Full explanation and naming conventions: [`docs/PROJECT_STRUCTURE.md`](docs/PROJECT_STRUCTURE.md).

## Documentation

| Document | Purpose |
|---|---|
| [`docs/PRD.md`](docs/PRD.md) | Product requirements: goals, MVP, user stories, acceptance criteria |
| [`docs/SYSTEM_DESIGN.md`](docs/SYSTEM_DESIGN.md) | Architecture, subsystems, data flow, runtime diagrams |
| [`docs/PROJECT_STRUCTURE.md`](docs/PROJECT_STRUCTURE.md) | Repository layout and naming conventions |
| [`docs/TECH_STACK.md`](docs/TECH_STACK.md) | Technology choices and why each was selected |
| [`docs/FEATURES.md`](docs/FEATURES.md) | Feature roadmap: MVP / post-MVP / future / experimental |
| [`docs/DEVELOPMENT_WORKFLOW.md`](docs/DEVELOPMENT_WORKFLOW.md) | Git workflow and AI/vibe-coding workflow |
| [`docs/CI_CD.md`](docs/CI_CD.md) | GitHub Actions pipeline design |
| [`docs/TESTING.md`](docs/TESTING.md) | Testing strategy and Auto Drive test scenarios |
| [`docs/DECISIONS.md`](docs/DECISIONS.md) | Architecture Decision Records |

## Important project principles

1. **Separation of subsystems.** Game/application, vehicle simulation, manual input, Auto Drive, routes, world, UI, audio, save/config, and build infrastructure stay separate and communicate through defined interfaces/events.
2. **Auto Drive is a driver, not a vehicle feature.** It produces drive commands; it never reaches into vehicle physics internals.
3. **Low-end hardware is first-class.** Lightweight architecture, small assets, low-poly art, texture atlases, scalable graphics settings, and a Compatibility-renderer baseline are constraints, not afterthoughts.
4. **Simple systems that evolve.** No speculative frameworks, no unnecessary dependencies, no over-abstraction. Prefer the simplest design that satisfies the current requirement.
5. **Original or properly licensed content only.** No proprietary simulator source code, assets, branding, UI, maps, or sounds. This project is inspired by the *genre*, not by any specific product.
6. **Small, reviewable changes.** Especially for AI-assisted work. Never rewrite working systems wholesale.
7. **Documentation follows architecture.** When an architectural decision changes, the relevant document is updated in the same change.
8. **Builds are CI-built and machine-independent.** The development machine must never be required to produce a shippable build.

## AI coding principles

Rules for AI coding agents working on this repository:

1. **Read the relevant documentation first** (`docs/`) before modifying code. Follow the architecture described in `docs/SYSTEM_DESIGN.md` and decisions in `docs/DECISIONS.md`.
2. **Do not rewrite unrelated systems.** Change only what the task requires.
3. **Make small changes.** One concern per change; keep diffs reviewable.
4. **Preserve existing architecture.** Respect module boundaries and dependency direction. Do not couple Auto Drive to the vehicle, or UI to physics internals.
5. **Update documentation when architecture changes.** Code and docs change together.
6. **Do not introduce dependencies without justification.** State the reason and record the decision.
7. **Do not copy proprietary code or assets.** All content must be original or under a permissive license compatible with this project's MIT license.
8. **Prefer simple implementations.** Solve the stated problem, not a hypothetical future one.
9. **Keep performance appropriate for low-end hardware.** No per-frame allocations in hot paths, no heavy shaders or huge assets unless justified and budgeted.
10. **Add tests for important logic** (route logic, Auto Drive decisions, pure math) using the project's test framework.
11. **Never silently change public interfaces** (class names, signal signatures, data schemas, exported APIs). Renaming or changing a contract requires updating callers and docs, and calling it out in the PR.
12. **Explain significant architectural changes.** If a change alters subsystem responsibilities or data flow, write down why in the PR description and update `docs/DECISIONS.md`.

## License

MIT — see [`LICENSE`](LICENSE).

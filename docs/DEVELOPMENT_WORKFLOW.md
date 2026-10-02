# Development Workflow — RoadPilot

| Field | Value |
|---|---|
| Project | RoadPilot |
| Status | Active — planning complete (2026-10-01); living document |
| Last updated | 2026-10-01 |
| Related | [CI_CD.md](CI_CD.md), [TESTING.md](TESTING.md), [README.md](../README.md) |

---

## 1. Git workflow

### 1.1 Branch model

```
main  ── stable integration branch; CI runs on every push; artifacts retained
  │
  ├── feature/<short-name>   new work
  ├── fix/<short-name>       bug fixes
  └── chore/<short-name>     tooling/docs/CI maintenance
```

- **`main` is always in a buildable, testable state.** No broken merges.
- Branches are **short-lived** (hours to a few days). Long-lived branches are where divergence bugs come from.
- Branch names are `snake_case` after the type prefix: `feature/auto-drive-steering`, `fix/route-validation-crash`.

### 1.2 Pull requests

- Use a PR for anything beyond trivial changes (docs typo, one-line fix may go direct — judgment call, but PRs are the default).
- Every PR must satisfy the checklist (`.github/PULL_REQUEST_TEMPLATE.md`, to be added with CI):
  1. **Scope:** one concern; diff is reviewable end-to-end.
  2. **Docs:** if architecture/behavior changed, the relevant `docs/` file is updated in the same PR.
  3. **Tests:** important logic has tests; existing tests still pass.
  4. **Lint/format:** `gdlint` + `gdformat` clean.
  5. **Content:** no proprietary code/assets; only original or properly licensed material.
  6. **Public API:** no silent changes to interfaces/signals/schemas — called out explicitly if intentionally changed.
- Squash-merge or merge-commit are both acceptable; prefer **squash** for feature branches to keep `main` history linear and legible.

### 1.3 Commits

- Imperative subject, scoped prefix: `vehicle: bound steering input`, `auto_drive: add lookahead steering`, `docs: update Auto Drive stages`.
- Small commits that build (locally) on their own — this is what makes bisecting and AI review practical.

### 1.4 Releases / versioning

- **Resolved (OQ6): no versioning scheme.** The project is personal (source public, game not published); builds are identified by commit SHA + GitHub run number only. Windows installers use the technical product version `0.0.<run_number>` — that is a packaging requirement, not a release number.
- Tagged releases and a GitHub Release workflow are **future** (see CI_CD.md §8).

## 2. CI integration

- **Every push to `main`** triggers the build pipeline: lint → headless tests → Windows build + MSI → Android APK → retained artifacts with commit SHA + build number ([CI_CD.md](CI_CD.md)).
- **PRs** run the fast gates (lint + headless tests) so problems are caught *before* merge. Artifact builds on PRs are optional/cheaper-scope (decided when the workflow is written).
- A red `main` is treated as a top priority: fix-forward immediately with a small PR; do not pile new features on a broken branch.

## 3. Local development loop

1. Read the relevant docs before touching code (see §4).
2. Create a branch (`feature/…`).
3. Make the smallest change that satisfies the requirement.
4. Run locally: **lint** (`gdlint`, `gdformat --check`), **headless tests** (GUT), and a quick in-editor smoke of the affected scene.
5. Check performance-relevant changes against the low-end budget (dev machine is the benchmark: ≥30 FPS, low preset).
6. Update docs if architecture moved.
7. Open a PR (checklist), address review, merge.
8. Confirm the post-merge `main` run is green and artifacts exist.

No local environment should need secret setup beyond what the README documents; heavy builds (MSI/APK) belong to CI — local runs are optional and faster scoped (e.g., export without MSI).

## 4. AI / vibe-coding workflow

AI coding agents are expected contributors. The workflow exists to keep their output **small, correct, and reviewable**.

### 4.1 The loop

```
1. READ      relevant docs (PRD scope, SYSTEM_DESIGN contracts,
             DECISIONS for architecture, FEATURES for the stage being built)
             + the files you are about to change
2. PLAN      the smallest change that satisfies the task
3. BRANCH    feature/<short-name>
4. EDIT      only the files this task owns; respect dependency rules
5. VERIFY    lint + headless tests + targeted manual smoke
6. DOCS      update docs if architecture/behavior changed
7. PR        explain what/why; list any interface changes explicitly
8. MERGE     only after checklist passes; CI proves main stays green
```

### 4.2 Hard rules for AI agents (full list: README → AI coding principles)

- Read docs before modifying code; **preserve the existing architecture**.
- **Small changes.** Never rewrite unrelated or working systems "while you're here".
- **Do not invent implementation details** that contradict the docs (schemas, command fields, event names come from SYSTEM_DESIGN).
- **Never silently change public interfaces** — signals, class names, JSON schemas, exported APIs are contracts; changing one requires updating all callers, tests, docs, and calling it out in the PR.
- **No new dependencies** without a stated justification and a `DECISIONS.md` entry.
- **No proprietary code/assets** — original or permissively licensed only.
- **Low-end budget respected**: no per-frame allocations in hot paths, no heavy textures/shaders beyond budget (NFR-7).
- **Add tests** for important logic you touch (route math, Auto Drive control math, validation).
- **Explain significant architectural changes** in the PR; architecture changes go through `DECISIONS.md`.

### 4.3 Anti-patterns to reject in review

- "While refactoring" sweeps across multiple subsystems in one PR.
- Cross-domain shortcuts (e.g., Auto Drive grabbing the rigid body directly, UI calling physics).
- New autoloads/utilities added without a second consumer or a decision.
- Copy-pasted code/assets from other simulators or tutorials with unclear licensing.
- Test-skipping as a way to make CI green; deleting tests instead of fixing behavior.
- Unexplained changes to route JSON schema or command/event names.

### 4.4 Sizing guidance

A good AI-assisted PR touches **one domain** (`src/<domain>/`) plus, at most, its direct dependency contracts and tests/docs. If the diff spans 4+ domains, it is almost certainly two PRs pretending to be one.

## 5. Code review standards

Even solo, PRs are reviewed (self-review with a checklist, or an AI reviewer pass):

- Correctness against the task and PRD requirement IDs (e.g., "implements AR6/AC5").
- Dependency direction respected (SYSTEM_DESIGN §4).
- Performance sanity for the low-end target.
- Docs/tests included where required.
- No scope creep; no undocumented interface changes.

## 6. Issue / bug handling

- Bugs get a `fix/*` branch from `main`; the PR references the requirement/acceptance criterion it restores (e.g., "restores AC7").
- Regressions found by CI → fix-forward on `main` promptly (§2).
- Nice-to-haves → recorded in FEATURES.md tiers rather than ad-hoc branches, to protect the MVP boundary (PRD §5, R5).

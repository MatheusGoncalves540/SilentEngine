# design/ — SilentEngine Design System

Complements `docs/spec/` (what the engine does) and `EXECUTION.md` (how to run
the work). This folder is **how the editor looks and behaves**. Same rules as
the spec: no open questions; when in doubt apply `docs/spec/01-mission.md`
priorities and log an `adr-pending` in `NOTES.md`.

Visual reference (canvas with every board):
https://claude.ai/artifact/9BQw8zsrdskMEYEoH2qNi7 — private until shared.

## Decisions (fixed)

| Topic | Decision |
|---|---|
| Structure | Pro-technical: dense data, typed colors per data kind, mono numbers |
| Palette | Warm organic neutrals (clay), no pure grey, no pure black |
| Accent | One violet. Means focus, selection, AI. Never decoration |
| Density | Dense rows (22 px, 12 px text) but calm surface: progressive disclosure (§04.3) |
| Themes | Data-driven theme files like VS Code / Zed. Built-in: Silent Ember (dark, default), Silent Linen (light) |
| Scale | Everything is a registered contribution (commands, menus, panels, tokens, themes). Adding a feature never means editing a central layout |

## Files

| File | Contents |
|---|---|
| 01-principles.md | Six rules every screen obeys |
| 02-tokens.md | Semantic roles, color values, type, space, sizes, radius, elevation |
| 03-components.md | Widget specs: anatomy, sizes, states, tokens |
| 04-layout.md | Editor shell, docking, panels, disclosure levels, command registry |
| 05-themes.md | Theme file format, loader, egui mapping, plugin themes |
| themes/*.theme.ron | Built-in themes. **Source of truth for values** |

If a value here disagrees with `themes/*.theme.ron`, the `.ron` wins and this
doc is the bug.

## Where it plugs into the roadmap

Editor UI is **Phase 2** (`docs/spec/11-roadmap.md`). Design work is not a new
phase; it rides on Phase 2 tasks:

| Phase 2 task | Design files it must read |
|---|---|
| egui editor shell | 04-layout §1–2, 02-tokens, 05-themes §4 |
| auto-inspector from reflect | 03-components §6–7 |
| agent loop UI (AI chat) | 03-components §10, 04-layout §2 |
| metrics panel | 03-components §11, 02-tokens §1.4 |
| theme loader (new task, add to roadmap via ADR) | 05-themes |

Phase 1's "editor waveform panel" uses the same tokens; if it lands before the
theme loader, hard-code **Silent Ember** values through a `theme::builtin()`
function, never literals in widgets.

## Verification (gates, per EXECUTION.md §4)

Design adds no new gate. It adds tests to the **correctness** gate and inputs
to the **visual** gate:

| Check | How | Pass when |
|---|---|---|
| Theme schema | `cargo test -p editor theme::` parses every `themes/*.theme.ron` | every semantic role resolves; no unknown keys |
| Contrast | same test computes WCAG ratio for pairs in 02-tokens §1.5 | all ≥ listed minimum, both built-in themes |
| No literal colors | `cargo clippy` lint / grep `Color32::from_rgb` outside `crates/editor/src/theme/` | zero hits |
| Editor visual | critic subagent gets editor screenshots (Ember + Linen) at 2560×1440 next to the canvas boards 05/06 | no finding above `minor` |

Metrics (spec §08.8 style, mandatory): `editor.theme.apply_ms` (histogram),
`editor.theme.load_errors` (counter), `editor.command.invocations` (counter,
label = source: menu | palette | shortcut | ai).

# 11 — Roadmap

Each phase has a Definition of Done (DoD). Do not start a phase before the
previous one's DoD is green in CI. Phases are ordered to de-risk the hardest
decisions early (ECS, assets pipeline, metrics) and push scope to the end.

## Phase 0 — Foundation

Goal: workspace, App/Module, registry, RON, cooker v0, CLI skeleton, metrics
registry, allocation tracker.

DoD:

- `cargo test --workspace` green.
- `engine new demo && engine run demo` opens a window with a colored triangle.
- `engine build demo` produces a self-contained binary that runs the same scene.
- `bench/reports/` exists with a first `frames.jsonl` for the triangle scene.
- CI: module without `Metrics` impl fails build.

## Phase 1 — 2D playable

Goal: winit + wgpu sprite batcher, input, cpal audio, rapier2d, camera, RON scene
loading, hot-reload.

DoD:

- A 2D game scene loads from `.scene.ron`, plays, hot-reloads while running.
- Bench: 128 K sprites at 60 fps on target, zero per-frame allocs.
- `engine build` produces a binary that plays the scene.
- Audio: play a looping track, editor waveform panel shows it.

## Phase 2 — Editor + AI + Plugins

Goal: egui editor, auto-inspector from reflect, agent loop with tool registry,
permission manifest, undo stack, WASM plugin host.

DoD:

- AI can, from chat: read a scene, mutate an entity, undo it, and re-run.
- AI can write a minimal WASM plugin, build it, and it loads when the user
  clicks Install.
- Editor and AI share the same undo stack.
- Metrics panel shows all registered families.

## Phase 3 — Build pipeline

Goal: `engine build` produces static Rust binary, cooker packs `assets.bin`,
smoke test runs.

DoD:

- Dist binary runs on a clean machine (no Rust installed).
- `dist/` is under 100 MB for the demo game (excluding user assets).
- Editor symbols verified absent from dist binary (CI symbol check).
- Correctness gate (§10.4) passes: dist == editor render.

## Phase 4 — 3D

Goal: clustered forward renderer, PBR, shadows, cascades, camera 3D, rapier3d,
skeletal animation.

DoD:

- 3D scene with a lit mesh, shadows, and a moving camera.
- Same bench protocol, same pass criteria (§10.1).
- Hot-reload for 3D scenes.

## Phase 5 — Advanced debug

Goal: tracy integration, RenderDoc capture, deterministic replay, audio/video
inspectors, visual regression suite.

DoD:

- Tracy shows every system's span for a frame.
- Replay produces identical state hashes (§10.3).
- Visual regression suite runs in CI, diffs 6 poses per PR.

## Phase 6 — Release v0.1

Goal: docs, examples, `engine new` templates, signed binaries.

DoD:

- Two example games ship (a 2D and a 3D).
- Docs cover: Module trait, RON format, cooker, AI tools, plugin ABI.
- All four gates (§10) pass on tag.

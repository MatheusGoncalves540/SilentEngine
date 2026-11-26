# 01 — Mission

## Mission

Rust game engine with three non-conflicting goals:

1. **Radical modularity** — everything is a registered module. Editor, runtime,
   cooker, and AI agent mount the same `App` with different subsets. If a module
   can't be removed without breaking the build, it isn't a module.
2. **Integrated AI as first-class** — the agent uses the same scene graph, files,
   and tools the editor does. It writes real plugins that run.
3. **Mandatory instrumentation** — no system enters main without a metric. If it
   doesn't measure, it doesn't exist.

## Priorities (on conflict, top wins)

1. **Observable correctness** — editor == binary.
2. **Stable frame pacing** — p99 ≤ budget, max ≤ 2× budget. Never "nice average,
   hitches in the tail".
3. **Raw performance and build time**
4. **Final binary size**
5. **Author ergonomics** (human and AI)

## Target

The target has two layers: a **meta** (fixed thresholds) and a **baseline**
(runtime telemetry snapshot, §09.11). The gate checks both.

### Meta (fixed)

- Windows 11 25H2, Linux 24.04, macOS 15+. Desktop only for v0.x.
- GPU: RTX 5070 Ti / AMD equivalent. CI perf on RTX 4070 as baseline.
- 2560×1440 @ 180 Hz. Budget: **5.5 ms GPU-bound**, **8 ms CPU-bound**.
- **p99 ≤ 11 ms**, **max ≤ 16.7 ms** over 60 s scripted fly-through.
- Benchmark scene: `bench/flythrough.ron` — versioned, changed only on approval.

### Baseline (runtime snapshot)

Every run of the engine — dev, bench, or build-verify — writes a
`target.ron` snapshot to `bench/targets/<run_id>/` (full history, versioned)
and to `target/` (local rotation, last 10 runs). The gate compares against the
last snapshot that **passed** the gate. A run worse than baseline by >5 % on
any gated metric fails, even if it is inside the meta thresholds. See §09.11
for format, retention, and the `targets.*` tools.

**No procedural-only restriction.** Binary assets (png, ogg, glb, ktx2 source)
are first-class citizens of `assets/`. What is *not* allowed is binary files
without a sibling `.asset.ron` metadata file, or binary scene/prefab/save data.
Text is the source of truth for structure and metadata, not for pixel/audio
payloads.

## Constraints

- **No external engine** (no full Bevy/Fyrox/Godot). Allowed deps: `bevy_ecs`
  (ECS+reflect), `wgpu`, `winit`, `cpal`, `egui`, `wasmtime`, `rapier2d/3d`,
  `serde`/`ron`, `tracing`.
- **Text as source of truth** for structure and metadata: RON for scene,
  prefab, material, animation, shader, config, save. Binary assets (png, ogg,
  glb, ktx2 sources) are allowed and normal; each has a sibling `.asset.ron`
  metadata file. No binary scene/prefab/save data. No binary without metadata.
- **Final build is a static Rust binary.** `engine build` emits `./dist/<game>`
  with no runtime dependency on `cargo`.
- **Zero heap allocations per frame** in hot paths (render, ECS tick, audio
  callback). Enforced by allocator tracking (§09.4). Buffers pre-allocated.
- **Determinism**: same seed + input log → identical state hash per frame.
- **AI never executes `unsafe` or native dylib.** AI-generated plugins run in
  WASM sandbox with declared capabilities (§07).
- **Every module exposes metrics.** CI rejects PRs of modules without `Metrics`.

## Done

`10-verification.md` defines four gates (performance, visual, determinism,
correctness). All must pass with no findings above **minor**, and the latest
`target.ron` must be within meta thresholds **and** within 5 % of the accepted
baseline.

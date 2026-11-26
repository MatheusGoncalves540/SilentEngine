# 02 — Architecture

## 1. Crate layout

```
engine/crates/
  core/       App, Module, TypeMap, event bus, time, RNG, profile flags
  ecs_ext/    thin helpers over bevy_ecs
  assets/     RON, cooker, hot-reload, cache
  render/     wgpu backend, 2D+3D pipelines
  audio/      cpal backend, mixer graph
  input/      winit adapter, action map (RON)
  physics/    rapier wrapper
  script/     Rhai for mods
  ai/         agent loop, Tool API, permissions
  plugin/     wasmtime host, manifest
  editor/     egui, inspector, viewport, undo
  debug/      profiler, replay, viewports
  metrics/    registry, exporters (jsonl, csv, tracy)
  cli/        engine new|run|build|test|bench
```

Each crate is a Cargo package. A crate depends only on crates it declares in
`deps()`.

## 1.1 CLI mirrors UI — non-negotiable

> **Every action reachable through the UI must be executable from the CLI.**
> No exceptions, no "internal only", no "power user flag".

This is a **structural** rule, not a feature. It is how we guarantee that the
AI can drive the engine, that CI can reproduce any manual step, and that no
behavior lives only in a mouse handler.

Consequences, all mandatory:

- Every `Command` that a UI panel pushes (undo stack, §06.4) has a CLI
  subcommand that pushes the *same* `Command`. Same code path, same effect.
- Every panel that mutates state must expose its mutations as `Command`s and
  register them in a `CommandRegistry` (see `docs/spec/08-editor.md` §9).
- Every entry in `CommandRegistry` is auto-exposed:
  - as a CLI subcommand `engine <domain> <action> [--flags]`,
  - as an AI tool `command.<domain>.<action>` (§06.2),
  - as a keybindable editor action.
- The CLI subcommand and the editor action call **the exact same function**.
  No reimplementation, no "CLI variant". If a divergence is needed, it's a
  new `Command` with its own subcommand.
- A module that adds a panel but does not register its commands fails CI
  (`cli-parity` check, §2 below).

### 1.1.1 `cli-parity` CI check

`engine cli verify` walks the `CommandRegistry` and asserts:

1. Every registered command has a CLI subcommand (`--help` includes it).
2. Every registered command has an AI tool registration.
3. Every registered command has an undo/redo implementation.
4. No CLI subcommand exists that is not backed by a registered command
   (no orphan subcommands).

Any missing entry → exit non-zero, CI blocks the merge.

### 1.1.2 Why

- **AI parity** — the agent drives the engine through the same surface a
  human does. If a UI action exists, the AI can do it (`docs/spec/06-ai.md`).
- **Reproducibility** — any bug report, any gate, any bench is a CLI script.
- **Determinism** — CI never clicks. It types.
- **Discoverability** — `engine --help` is the authoritative list of what the
  engine can do. If it's not there, it doesn't exist.

## 2. Module contract

```rust
pub trait Module: Send + Sync + 'static {
    fn name(&self) -> &'static str;
    fn deps(&self) -> &[&'static str] { &[] }
    fn register(&self, app: &mut App);
    fn metrics(&self) -> &dyn Metrics;                          // REQUIRED
    fn commands(&self) -> &[Box<dyn Command>] { &[] }            // REQUIRED by §1.1
}
```

A module with `commands()` returning empty is allowed **only if it owns no UI**
(and therefore no mutations). Adding a panel requires adding commands.

- `App::add_module(m)` inserts `m` into a topological sort keyed by deps.
- Cycle → build fails, error points at the cycle.
- Missing dep → build fails with `module X requires Y`.
- `register()` runs once per App build.

## 3. Frame order (fixed)

1. **Input** — drain winit events, update action state
2. **Time** — delta, frame counter, fixed accumulator
3. **FixedUpdate** — 0+ ticks at fixed dt (60 Hz default)
4. **Update** — variable dt
5. **Render** — encode, submit, present
6. **Metrics** — flush per-frame counters (feature-gated)

Any new module must hook into one of these phases explicitly. No async spawn in
hot paths.

Commands and CLI subcommands follow the same phase model: a CLI invocation
produces a `Command` that is applied at the module's declared phase, never
inline. This keeps CLI and UI bit-identical by construction.

## 4. Resources

`App` holds a `TypeMap` over `dyn Any + Send + Sync`. `insert_resource`,
`resource`, `resource_mut`. `resource_mut` is exclusive; no interior mutability
in the common case.

## 5. Events

Double-buffered typed bus. `send(Ev)` writes to next-frame buffer.
`read::<Ev>()` returns last frame's batch. Ordering is deterministic per phase.

## 6. Determinism

- No `SystemTime::now()` in gameplay. Use `Time` resource.
- No `HashMap` iteration in gameplay systems without sorted key extraction.
- RNG is per-system seeded. Frame-rate-dependent RNG is a bug.
- `--replay path.ron` re-runs an input log; two replays produce identical state
  hashes per frame (§09.5).

## 7. Feature flags and profiles

- `editor` — egui inspector, viewport. On for run, off for build.
- `metrics-export` — jsonl/csv exporters.
- `tracy` — tracing-tracy.
- `replay` — input record/replay.

Every flag has a compile-time profile in `crates/core/src/profile.rs`. CI asserts
"no editor symbols in release" by reading the symbol table.

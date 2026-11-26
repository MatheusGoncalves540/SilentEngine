# SilentGame — Rust Engine Spec

Auto-executable spec for a modular Rust game engine with integrated AI and
mandatory instrumentation. **No open questions.** When in doubt, apply the
priorities in `01-mission.md` and log the interpretation in an ADR.

## Files

| File | Contents |
|---|---|
| 01-mission.md | Mission, priorities, target, constraints |
| 02-architecture.md | Crate layout, Module trait, App lifecycle |
| 03-assets.md | RON format, cooker, hot-reload |
| 04-render.md | wgpu backend, 2D/3D pipeline rules |
| 05-audio.md | cpal backend, mixer, debug |
| 06-ai.md | Agent loop, Tool API, permissions |
| 07-plugins.md | WASM sandbox, wasmtime, manifest |
| 08-editor.md | egui inspector, viewport, undo |
| 09-metrics.md | Metrics trait, exporters, hot paths |
| 10-verification.md | Gates, critic protocol, done criteria |
| 11-roadmap.md | Phases 0-6, DoD per phase |
| 12-bench-protocol.md | Fly-through, camera poses, report format |

## Running

```
engine new mygame
engine run mygame
engine bench --scene bench/flythrough.ron --out bench/reports/
engine bench --quick                       # 10 s, advisory only, no commit
engine targets latest                      # read latest target.ron
engine targets history                     # list runs + deltas
engine targets set-baseline <run_id>       # promote a baseline (explicit)
engine cli verify                          # CLI-UI parity check (mandatory)
engine test --visual --pose A,B,C,D,E,F
engine build mygame --release
```

## Acceptance (v0.1)

- Bench meta: p99 ≤ 11 ms, max ≤ 16.7 ms on target hardware (§10.1)
- Bench baseline: every gated metric within 5 % of the accepted `target.ron`
  for this `host_slug` (§09.11.6). Better-by->20 % flagged suspicious.
- Visual critic: no finding above minor for all 6 poses (§10.2)
- `cargo test --workspace` green
- `engine build` output runs the same scene as `engine run`
- Latest `bench/targets/<run_id>/target.ron` has `gate: Passed` and is committed
- **CLI parity**: `engine cli verify` exits 0. Every UI-reachable action has a
  CLI subcommand, an AI tool, and an undo entry. No orphans (§02.1.1, §10.4.1)

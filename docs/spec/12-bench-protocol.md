# 12 — Benchmark Protocol

Fixed protocol so results are comparable across commits, phases, and machines.
Changing anything here requires an explicit approval + changelog entry.

## 1. Scene

`bench/flythrough.ron` — a scripted 60 s camera path through a representative
scene. Contains:

- 60 s of camera keyframes at fixed dt (60 Hz).
- Fixed seed for all RNG.
- All modules' features exercised: assets hot-reload, physics, audio loop,
  particles, AI tool (one scripted call at t=10 s to prove tool cost), plugin
  (one scripted WASM call at t=20 s).

Camera path is a Bezier spline through the scene, with a fixed yaw/pitch curve.
No user input during bench.

## 2. Camera poses (for visual critic)

6 fixed poses, saved as PNG at 2560×1440. Each pose is a keyframe of the
fly-through, so the same frame is used in both gates.

| Pose | t (s) | Purpose |
|---|---|---|
| A | 0 | Hero shot — first impression |
| B | 12 | Depth/fog test — long corridor |
| C | 24 | Overdraw test — dense foliage |
| D | 36 | Motion test — mid-flight, particles visible |
| E | 48 | Detail test — close-up of one asset |
| F | 60 | Frame-boundary test — moving out of densest area |

Each pose rendered twice (once with editor overlay off, once on) to prove the
editor is not altering the render.

## 3. Video

60 s fly-through recorded at 60 fps (frame-locked to bench), 2560×1440, lossless
FFV1 in `.mkv`. Used by the visual critic to catch temporal artifacts the 6
stills can miss: shimmer, popping, banding during motion, frame pacing.

## 4. Hardware protocol

- Fresh boot, no other GPU apps.
- Power plan: performance.
- GPU driver: latest stable at the time of the run (recorded in `summary.md`).
- OS build recorded.
- Clock speed sensors logged alongside.
- Warmup: 5 s of the fly-through before measurements begin.
- Run 3 times; report all three + median.

## 5. Report format

Each run writes two top-level artifacts:

- `bench/reports/run_<date>/summary.md` — human-readable.
- `bench/targets/<run_id>/target.ron` — runtime snapshot (§09.11.4), read by
  the gate. A copy is also placed in `bench/reports/run_<date>/target.ron`.

`bench/reports/run_<date>/summary.md`:

```
# Bench run <date>

## Hardware
- GPU: ...
- CPU: ...
- RAM: ...
- OS: ...
- Driver: ...

## Results (median of 3)
- GPU: avg 4.12 ms, p99 8.90 ms, max 12.4 ms
- CPU: avg 5.20 ms, p99 9.10 ms, max 14.1 ms
- Allocs per frame (max): 128 bytes
- Audio underruns: 0
- Audio clips: 0
- Sprite overflows: 0
- Plugin traps: 0

## Regressions vs baseline (<baseline_run_id>)
- render.frame.gpu_ms: -0.3 ms (better, inside 5%)
- render.draw.calls: +12 (worse, +8.2%, EXCEEDS 5% → gate FAIL)
- render.alloc_bytes_per_frame_max: 128 (unchanged)

## Gate verdict
- Meta ceilings: PASS
- Baseline check: FAIL (render.draw.calls +8.2%)
- Overall: FAIL

## Attachments
- frames.jsonl
- target.ron
- histogram_gpu.png
- histogram_cpu.png
- poses/A.png ... F.png
- flythrough.mkv
- critic_final.md
```

The gate verdict section is **generated**, not written by hand — same code
path as `engine bench` exit status. If gate is FAIL, exit code is non-zero.

## 6. CI policy

- Bench runs **only** on the dedicated perf machine (never on dev laptops).
- Devs can run `engine bench --quick` (10 s) locally; results are advisory and
  land in `target/targets/` only, never committed.
- Gate is the dedicated machine's median-of-3 against the last accepted
  baseline for that `host_slug` (§09.11.7).
- Regression threshold: any gated metric worse by >5 % vs baseline blocks the
  release; better-by->20 % is flagged suspicious and requires signoff.
- Baseline updates only via `targets.set_baseline(run_id)` — an explicit commit
  that touches `bench/targets/_default.ron` and adds a `BASELINE.md` note.

## 7. Visual critic workflow

1. Bench run produces 6 PNGs + 1 `.mkv`.
2. `engine bench critic` uploads them to the critic subagent.
3. Critic returns `critic_final.md` with findings.
4. If any finding > minor: block, fix, re-render, repeat.
5. `critic_final.md` is committed with the release.

Critic sees **only** the images and video. Never the code, spec, or reasoning.

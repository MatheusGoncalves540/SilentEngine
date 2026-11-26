# 10 — Verification

Two gates. Both must pass before a release. No exceptions, no "ship it and fix
later". If it's a build, it's gated.

## 1. Performance gate

### Protocol

1. `engine bench --scene bench/flythrough.ron --out bench/reports/run_<date>/`
2. Runs the scripted fly-through (§12) at 2560×1440, windowed, on target
   hardware, with a fresh boot.
3. Captures:
   - GPU timestamp queries per frame (wgpu `TIMESTAMP_QUERY`).
   - CPU frame time via `Instant` around the frame.
   - `frames.jsonl` per-frame log (§09.7).
   - `target.ron` runtime snapshot (§09.11.4).
4. Reports:
   - `summary.md` — human-readable avg / p99 / max for GPU, CPU, per subsystem.
   - `frames.jsonl` — raw per-frame data.
   - `target.ron` — the runtime snapshot, read by the gate.
   - `histogram_*.png` — per-metric histograms.

### Pass criteria

All of the following must hold. Meta ceilings come from §01; baseline
comparison comes from §09.11.6.

**Meta ceilings (hard, cannot regress past these even if baseline is worse):**

- GPU avg ≤ 5.5 ms, p99 ≤ 11 ms, max ≤ 16.7 ms
- CPU avg ≤ 8 ms, p99 ≤ 11 ms, max ≤ 16.7 ms
- Zero allocations in `render()` per frame (§04.7)
- Zero audio underruns, clips, sprite overflows, plugin traps (§09.8)
- No single frame > 16.7 ms

**Baseline check (vs the last accepted `target.ron`, §09.11.6):**

- Every gated metric is within 5 % of baseline, direction-aware (lower is
  always fine).
- Better-by-more-than-20 % is flagged *suspicious* and requires human signoff
  (catch accidental no-ops).
- A missing metric in the snapshot fails with `IncompleteSnapshot`.

The run's `target.ron` is written to `bench/targets/<run_id>/` and is the
artifact the gate reads.

### Failure modes and responses

| Symptom | Likely cause | First check |
|---|---|---|
| GPU avg high | shader too heavy, overdraw | per-draw-call timing in tracy |
| GPU p99 spike | pipeline creation, texture upload | search logs for `create_` during frame |
| CPU spike | `Vec::push` overflow, GC, syscall | allocator tracker per-frame |
| Audio underrun | mixer node too heavy, buffer too small | callback timing histogram |
| Hitch every N frames | texture atlas rebuild, save-to-disk | metric `assets.hotreload.*` |

## 2. Visual gate

### The critic

A **separate subagent** that sees **only the renders**. Not the code, not the
reasoning, not the spec. It receives:

- 6 PNGs at 2560×1440, one per fixed camera pose (§12.2).
- 1 video (60 s, 60 fps) of the scripted fly-through.

It returns **prioritized, concrete defects**, each with:

- severity: `blocker` / `major` / `minor` / `nit`
- location: pose id + screen region (top-left, center, right-quarter, etc.)
- description: what's wrong
- suggested fix, at the visual level (not code level)

### Pass criteria

- No finding above `minor`.
- All `minor` findings documented as accepted.

### Iteration loop

```
while any finding > minor:
  fix highest-severity finding
  re-render 6 poses + fly-through video
  send to critic
```

### Critic protocol (system prompt, fixed)

```
You are a visual critic. You are given 6 still frames at 2560x1440 and a 60 s
fly-through video. You do NOT see the code or the reasoning behind the scene.
Your job: identify visual defects. Focus on:

1. Aliasing (shimmering edges, moire on distance)
2. Banding (in gradients, fog, sky)
3. Popping (LOD, culling, lighting changes)
4. Artifacts (clipping, z-fighting, broken normals)
5. Composition (does the frame read? is there focus? is depth legible?)
6. Frame pacing (from the video: stutter, jitter)

For each defect return: severity (blocker/major/minor/nit), pose id + region,
description, suggested fix (visual). Do not list preferences as defects.
```

## 3. Determinism gate

Nightly CI:

- Record `bench/flythrough.ron` once → `bench/replays/base.ron`.
- Replay three times → identical state hash sequence (§09.5).
- Any divergence → fail, diff first divergent frame.

## 4. Correctness gate

- `cargo test --workspace` green.
- Editor == binary: `engine build` then run the same scene → screenshot diff
  against `engine run` (allowed diff: tiny AA variance; must fail otherwise).

## 4.1 CLI parity gate (mandatory, per `docs/spec/02-architecture.md` §1.1)

- `engine cli verify` exits 0.
- `editor.commands.registered` == number of CLI subcommands under `engine`.
- `editor.commands.cli_parity_errors` == 0 in the run's `target.ron`.
- No orphan CLI subcommand (`engine <x> <y>` without a registered
  `command.<x>.<y>`).
- No orphan UI command (button/panel mutation without a registered command).

Any violation blocks the release. This gate exists because the CLI is the
**only** surface the AI and CI can drive deterministically.

## 5. Done

All four gates pass, and the run's `target.ron` is committed at
`bench/targets/<run_id>/target.ron` with `gate: Passed`. Release artifacts:

- `dist/<game>.exe` (or platform binary)
- `dist/assets.bin` + `dist/assets.idx`
- `bench/reports/<date>/summary.md`
- `bench/reports/<date>/target.ron` (copy of the committed snapshot)
- `bench/reports/<date>/critic_final.md`
- `CHANGELOG.md` entry
- If baseline changed (only via `targets.set_baseline`): an explicit
  `BASELINE.md` note in the same commit, listing old and new `run_id`.

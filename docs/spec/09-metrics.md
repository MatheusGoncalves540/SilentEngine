# 09 — Metrics (mandatory)

## 1. Why a document

"No system enters main without a metric." This file defines the contract, the
registry, the exporters, and the CI checks that make it enforceable.

## 2. Metric primitives

```rust
pub enum Metric {
    Counter(u64),       // monotonic, per-frame delta logged
    Gauge(f64),         // last value
    Histogram(Histogram), // reservoir or log-bucket, 128 buckets default
}
```

Every metric has:

- **name**: `module.subsystem.name` (dot-separated, lowercase)
- **unit**: `ms`, `bytes`, `count`, `pct`, `hz`
- **description**: one line, plain English

Naming rules:

- First segment = module name, matching `Module::name()`.
- `.ms` suffix means milliseconds (histogram).
- `.bytes` suffix means bytes (gauge or counter).
- `.count` suffix means counter.
- No unit suffix on gauges with arbitrary scale — use `.ratio` or `.pct`.

## 3. Registry

Global `MetricsRegistry` in `crates/metrics`. Populated by modules in
`register()`. Zero-allocation `record()` on hot paths:

```rust
metrics.counter("render.draw.calls").add(1);
metrics.histogram("render.frame.gpu_ms").record(gpu_ms);
```

Each metric is an index into a static array; recording is a single store.
No allocation, no lookup, no string format.

## 4. Allocation tracking

In debug builds, `#[global_allocator]` wraps the system allocator:

- Counts bytes per frame.
- Exposes `mem.alloc_bytes_per_frame` gauge.
- CI bench asserts this is **≤ 256 bytes** in `render()` (§04.7).

In release builds the wrapper compiles out.

## 5. Replay + determinism check

`--replay path.ron` re-runs an input log at fixed dt. State hash:

- All components serialized with `rkyv`, hashed with xxhash.
- Per frame: `replay.state_hash` counter.
- Two replays must produce identical hash sequences; CI runs this nightly.
- Mismatch → fail with the first divergent frame + entity diff.

## 6. Exporters

| Exporter | Output | When |
|---|---|---|
| `jsonl` | one line per metric per frame | `--metrics jsonl` |
| `csv` | per-metric CSV | `--metrics csv` |
| `tracy` | tracy timeline | feature `tracy` |
| `prometheus` | `/metrics` HTTP endpoint | editor or dev server |

All exporters are feature-gated. `jsonl` and `csv` are the ones used in CI.

## 7. Frame log format

```json
{"frame": 42, "dt_ms": 5.12, "gpu_ms": 4.01, "cpu_ms": 3.8,
 "systems": {"render": 1.2, "update": 1.5, "physics": 0.3},
 "metrics": {"render.draw.calls": 84, "render.verts": 128000}}
```

One line per frame. Written to `bench/reports/<run>/frames.jsonl`.

## 8. Bench assertions (CI)

For a 60 s fly-through at 2560×1440 on target hardware:

- `render.frame.gpu_ms` avg ≤ 5.5, p99 ≤ 11, max ≤ 16.7
- `render.frame.cpu_ms` avg ≤ 8, p99 ≤ 11, max ≤ 16.7
- `render.alloc_bytes_per_frame` ≤ 256 for every frame
- `audio.underruns` == 0
- `audio.clip.count` == 0
- `render.sprite.overflow` == 0
- `plugins.traps` == 0

Any violation fails the gate.

## 9. Adding a metric

1. Add it to the owning module's `metrics()` impl.
2. Document it in the module's spec section.
3. If it's a hot-path metric, verify via allocation test that recording is free.

## 11. Targets — runtime telemetry snapshot

### 11.1 Why

The target must be a **living artifact**, not a number in a document. Every
run of the engine samples the real hardware and the real frame data, writes it
to a snapshot, and the gate compares against the last snapshot that passed.
This is how "measure everything" becomes "measure everything and keep it".

### 11.2 What is collected

| Field | Source |
|---|---|
| `host.gpu`, `host.gpu_driver` | wgpu adapter info at boot |
| `host.cpu` | `sysinfo` crate |
| `host.os`, `host.monitor_refresh_hz` | winit + `sysinfo` |
| `host.resolution` | surface config |
| `durations.*` | aggregated histograms from §09 registry |
| `counters.*` | final counter values for the run |
| `allocations.*` | allocator tracker (debug builds; `0` in release) |

### 11.3 Where it lives

```
bench/targets/
  _default.ron                    # committed bootstrap baseline (see 11.7)
  <run_id>/target.ron             # one per gated run, committed
  <run_id>/frames.jsonl           # per-frame (usually gitignored, size)
target/
  targets/                        # local rotation, last 10 runs, gitignored
```

`run_id` = `<YYYY-MM-DD>_<git_sha_short>_<host_slug>` — deterministic, sorts
chronologically, unique per host+commit+day.

### 11.4 Format (RON)

```ron
TargetV1(
    run_id: "2026-11-25_a1b2c3d_rtx5070ti-win",
    host: (
        gpu: "NVIDIA GeForce RTX 5070 Ti",
        gpu_driver: "560.94",
        cpu: "AMD Ryzen 7 7800X3D",
        os: "Windows 11 25H2 (build 26100)",
        monitor_refresh_hz: 180,
        resolution: (2560, 1440),
    ),
    started_at: "2026-11-25T18:42:11Z",
    frame_count: 3600,
    durations: {
        "render.frame.gpu_ms": (n: 3600, min: 3.8, p50: 4.1, p90: 4.6, p99: 8.9, max: 12.4, sum: 14796.0, sum_sq: 61234.5),
        "render.frame.cpu_ms": (n: 3600, min: 4.1, p50: 5.0, p90: 5.8, p99: 9.1, max: 14.1, sum: 18720.0, sum_sq: 97811.2),
        "audio.callback.ms":   (n: 703,  min: 0.1, p50: 0.2, p90: 0.3, p99: 0.5, max: 0.9,  sum: 140.6,   sum_sq: 40.1),
    },
    counters: {
        "audio.underruns": 0,
        "audio.clip.count": 0,
        "render.sprite.overflow": 0,
        "plugins.traps": 0,
        "render.alloc_bytes_per_frame_max": 128,
    },
    baseline: Some("2026-11-24_9f8e7d6_rtx5070ti-win"),
    gate: Passed,   // Passed | Failed(reason) | Bootstrap
)
```

`gate` is `Bootstrap` for the run that seeded `_default.ron`.

### 11.5 Retention

- **Committed** in `bench/targets/`: every run that passed a gate, plus
  `_default.ron`. History is auditable and diffable.
- **Local only** in `target/targets/`: last 10 runs, rotating. Cheap during dev.
- **Failed runs** are never committed; they land in `target/targets/` and are
  surfaced in the report of the next run as "previous failure".

### 11.6 Gate comparison

For every metric in §09.8 pass list:

1. **Meta check** — value must be inside §01 meta thresholds.
2. **Baseline check** — value must be within 5 % of the corresponding value in
   the baseline snapshot. "Within" is direction-aware: lower is always fine.
   Better-by-more-than-20 % is flagged as *suspicious* and requires a human
   signoff, to catch accidental no-ops.

The gate emits `gate: Passed` only if both checks pass for every gated metric.

### 11.7 Bootstrap on a new machine

`bench/targets/_default.ron` is committed and represents the reference run on
target hardware. A new machine's first run compares against `_default.ron` and
is marked `gate: Bootstrap` if it is within meta thresholds. The first gated
run on that host then becomes the baseline for that host, looked up by
`host_slug`.

### 11.8 Tool API (for AI and editor)

| Tool | Capability | Returns |
|---|---|---|
| `targets.latest` | `metrics:read` | Latest snapshot (RON) |
| `targets.history` | `metrics:read` | List of run_ids + 1-line delta summary |
| `targets.compare(a, b)` | `metrics:read` | Per-metric diff between two runs |
| `targets.set_baseline(run_id)` | `metrics:write` | Promote a run as baseline (undoable) |

These tools are how the AI "consults the target" when running the gate.

### 11.9 Required metrics in snapshot

At minimum, the snapshot must contain every metric referenced by §09.8. Missing
metric in snapshot = gate fails with `IncompleteSnapshot`.

## 10. Required metric families (per module)

| Module | Must include |
|---|---|
| core | `core.frame.ms`, `core.systems.ms` |
| assets | §03.7 |
| render | §04.9 |
| audio | §05.5 |
| input | `input.events.count`, `input.latency.ms` |
| physics | `physics.step.ms`, `physics.bodies`, `physics.contacts` |
| script | `script.eval.ms`, `script.errors` |
| ai | §06.7 |
| plugin | §07.7 |
| editor | §08.8 |
| debug | `debug.replay.frame`, `debug.overlay.ms` |

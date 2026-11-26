# 05 — Audio

## 1. Backend

`cpal` for output. No DSP crate in core. Mixing is our own graph.

## 2. Mixer graph

```
Sources (voices) → Bus → Master → Device
```

- A **Source** is a decoded PCM stream (wav/ogg), fully pre-loaded. No
  streaming decode in v0.x.
- A **Bus** is a summing node with gain, pan, and up to 4 insert effects
  (reverb, delay, filter, EQ).
- **Master** applies final gain + limiter, writes to device buffer.

Graph is declared in `mixer.ron`, hot-reloadable:

```ron
MixerV1(
    buses: [
        (id: "music", gain: 0.7, effects: [Reverb(room: 0.8, wet: 0.2)]),
        (id: "sfx",   gain: 1.0, effects: []),
        (id: "amb",   gain: 0.6, effects: [Filter(cutoff: 800.0)]),
    ],
)
```

## 3. Audio callback

Audio runs on a dedicated OS thread with a callback that fills N frames
(256 by default, configurable 64–1024). **Zero allocation in callback.** All
buffers preallocated at startup.

- The callback drains a lock-free ring buffer of commands (play, stop, set
  param). If the ring is empty, silence.
- Summing uses SIMD-friendly layout (`f32` interleaved stereo).
- Underrun is a bug, logged and counted (`audio.underruns`); never silently
  produce garbage.

## 4. Debug UI

Editor has an **Audio** tab:

- **Live waveform** of every bus (rolling 1 s window).
- **Spectrogram** of the master bus.
- **Node graph view** — click a bus, see its connections and effect chain.
- **Per-source** cards: elapsed time, remaining time, gain, pan, loop flag.
- **Latency** — one-way device latency, buffer size, callbacks/sec.
- **Recording** — dump the last 10 s of master bus to `.wav`. Used by tests
  and critic.

## 5. Metrics (mandatory)

- `audio.callback.ms` (histogram) — must stay < 50 % of buffer period
- `audio.underruns` (counter) — must be 0 in bench
- `audio.voices.active` (gauge)
- `audio.bus.peak` (gauge, per bus)
- `audio.clip.count` (counter) — must be 0 in bench
- `audio.latency.ms` (gauge)

# 04 — Render

## 1. Backend

`wgpu` (Vulkan/Metal/DX12). No fallback to GL.

- Single device, single queue, single surface.
- All pipelines created at startup or asset-load time. **No pipeline creation in
  hot loop.** Missing pipeline in hot path → panic (caught in dev).
- Bindless-style: growable descriptor sets, per-frame arena, no allocations.

## 2. Frame budget

- GPU-bound budget: **5.5 ms** at 2560×1440 on target hardware (§01 meta).
- CPU-encode budget: **2.5 ms**.
- Submit cost measured via wgpu timestamp queries when available, else CPU wall
  clock around `queue.submit`.
- Every frame's `gpu_ms` and `cpu_ms` are recorded into the runtime target
  snapshot (§09.11). The snapshot is the baseline for the gate; the meta
  numbers above are hard ceilings that a regression cannot exceed even if the
  baseline is worse.

## 3. 2D pipeline (Phase 1)

- Batched sprite renderer. One draw call per texture atlas per frame in target
  scenes.
- Vertex buffer: fixed 128 K sprites (2 MB) allocated at startup, ring-cycled per
  frame. Overflow → metric `render.sprite.overflow` and drop (never realloc).
- Instanced quads. Texture atlas stored in a 2D array, capacity 256 layers.
- Camera: orthographic, pixel-perfect option, world units configurable per scene.

## 4. 3D pipeline (Phase 4)

Deferred for Phase 4. When built:

- Clustered forward shading (better for transparency-heavy scenes).
- Shadow atlas 4096². Cascades: 3 for directional light.
- Post: bloom → tonemap (ACES) → optional DOF.

## 5. Shaders

- WGSL only. No GLSL, no SPIR-V source.
- `#include` handled by a tiny preprocessor in `crates/render/src/shader_pre.rs`.
- Hot reload of `.wgsl` re-compiles pipeline; if compile fails, keep last
  working, log to console.
- Shader code is text in the repo. Yes, they diff.

## 6. Command encoding

- One `CommandEncoder` per frame, one `RenderPass` per pipeline stage.
- No `create_*` in `render()`.
- All dynamic buffers (uniforms, instance data) come from a per-frame `BufferArena`.
  Arena capacity is 32 MB by default, growable only between frames.

## 7. Zero-allocation rule

Per frame in `render()`:

- **No** `Vec::push` beyond pre-reserved capacity.
- **No** `String` construction.
- **No** `format!`.
- **No** iterator `.collect()`.

Enforced by:

- `#[global_allocator]` tracking bytes-per-frame in debug builds.
- CI test: run fly-through 300 frames in debug, assert per-frame allocation
  bytes ≤ 256 (some wgpu internal bookkeeping allowed; tight budget).

## 8. Debug viewports

Every pipeline registers a debug name. Inspector lets you:

- Blit any render target to screen (mips, depth, normals, albedo, AO).
- Pause frame, inspect every draw call, every buffer binding.
- Save `.png` snapshot per pose (used by visual critic §10.2).

## 9. Metrics (mandatory)

- `render.frame.gpu_ms` (histogram)
- `render.frame.cpu_ms` (histogram)
- `render.draw.calls` (counter)
- `render.verts` (counter)
- `render.pipeline.swaps` (counter)
- `render.sprite.overflow` (counter) — must be 0 in bench
- `render.bufferarena.used_bytes` (gauge)
- `render.alloc_bytes_per_frame` (gauge) — must be ≤ 256 in bench

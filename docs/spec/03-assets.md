# 03 — Assets

## 1. Golden rule

**Text is the source of truth. Binary is a build artifact.**

| Layer | Format | Committed? |
|---|---|---|
| Scene | `.scene.ron` | yes |
| Prefab | `.prefab.ron` | yes |
| Material | `.mat.ron` | yes |
| Animation | `.anim.ron` | yes |
| Shader | `.wgsl` (WGSL, text) | yes |
| Config | `.toml` (engine config only) | yes |
| Binary asset | `foo.png` + `foo.asset.ron` | yes (raw) |
| Cooked cache | `target/cooked/*.rkyv` | no (gitignored) |
| Dist bundle | `dist/<game>/assets.bin` | no (built) |

No opaque binary in the repo unless it is a genuinely binary source (png, ogg,
glb, ktx2 source). Its `.asset.ron` describes import settings.

## 2. RON schema (v1)

All RON files start with a versioned root tag. Example `player.prefab.ron`:

```ron
PrefabV1(
    name: "player",
    entities: [
        (
            id: "root",
            components: {
                "Transform": (pos: (0.0, 0.0, 0.0), rot: (0.0, 0.0, 0.0, 1.0), scale: (1.0, 1.0, 1.0)),
                "Sprite": (
                    texture: "assets/tex/player.png",
                    size: (32, 48),
                    pivot: (0.5, 0.0),
                ),
                "Health": (max: 100, current: 100),
            },
        ),
    ],
)
```

Rules:

- Unknown fields **error** on load (deny_unknown_fields). Prevents silent typos.
- Component names are strings, resolved through a `ComponentRegistry` populated by
  `register_component::<T>()` in each module.
- Field order is stable (BTreeMap internally). Diff-friendly.
- No tabs. 4-space indent. Max line width 100.

## 3. Asset metadata (`.asset.ron`)

```ron
TextureAssetV1(
    source: "player.png",
    filter: Nearest,
    wrap: Clamp,
    mips: true,
    color_space: Srgb,
)
```

The cooker reads the source + metadata, emits cooked form. Hot-reload watches
both files.

## 4. Cooker

```
engine cook <project_dir> [--release]
```

Pipeline:

1. **Scan** — walk `assets/`, collect every `.ron` and binary source.
2. **Validate** — parse every RON file, resolve every reference, error on missing.
3. **Convert** — png→ktx2, ogg→ogg (passthrough), glb→rkyv-packed mesh.
4. **Pack** — concatenate into `assets.bin` + `assets.idx`. Zero-copy rkyv.
5. **Verify** — read back, hash, compare against manifest.

Output is deterministic: same input → same bytes. Enables reproducible builds.

Cooked format uses `rkyv` for zero-copy reads. Runtime mmaps `assets.bin`,
no deserialization for large assets.

## 5. Hot reload

- `notify` watcher on `assets/`.
- Debounce 100 ms.
- On change: re-cook the *changed* asset only, swap handle via `ArcSwap`.
- Scene RON: re-parse, diff against live ECS, apply patch (add/remove/update
  entities) instead of reloading from scratch. Preserves runtime state for
  unchanged entities.
- Any hot-reload error → keep last-known-good, surface in editor console and
  metrics `assets.hotreload.errors`.

## 6. Asset handles

Typed generational handles: `Handle<Texture>`, `Handle<Mesh>`, etc.
`AssetServer::load::<T>(path) -> Handle<T>`. Misses return a magenta placeholder
and increment `assets.missing`.

## 7. Metrics (mandatory)

- `assets.load.count` (counter)
- `assets.load.bytes` (counter)
- `assets.load.ms` (histogram)
- `assets.hotreload.ms` (histogram)
- `assets.hotreload.errors` (counter)
- `assets.missing` (counter)
- `assets.cook.ms` (histogram, per-asset-kind)

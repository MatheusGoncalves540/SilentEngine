# 07 — Plugins

## 1. Philosophy

**AI can author plugins. Plugins run sandboxed. Native dylib is out of scope for v0.x.**

Plugins extend the engine through a stable, versioned ABI. They are the only way
the AI's generated code touches the runtime.

## 2. Two plugin kinds

| Kind | Where | Capability |
|---|---|---|
| **WASM** | `wasmtime`, sandboxed | Declared in manifest, host-enforced |
| **Native** (post v0.x) | `libloading` dylib | Full process privileges |

v0.1 ships **WASM only**. Native ABI is designed but disabled behind a build
flag; enabling it requires a UI confirmation every session.

## 3. Plugin manifest

Every plugin directory:

```
plugins/my_plugin/
  plugin.ron
  src/
    lib.rs
  Cargo.toml
```

`plugin.ron`:

```ron
PluginV1(
    name: "my_plugin",
    version: "0.1.0",
    api_version: 1,
    entry: "on_load",
    capabilities: [
        FsRead(glob: "assets/**"),
        SceneRead,
        Log,
    ],
    provides: [
        Component("MyComponent"),
        System("my_system"),
        Tool("my_tool"),
    ],
)
```

`api_version` mismatch → plugin refuses to load. Host records metric
`plugins.load_errors`.

## 4. Host ABI (WASM)

The host exposes a small, stable set of imports:

```
log(level, msg_ptr, msg_len)
fs_read(path_ptr, path_len) -> (buf_ptr, buf_len)
scene_query(filters_json_ptr, filters_len) -> entities_json_ptr
component_get(entity, name_ptr, name_len) -> bytes_ptr
component_set(entity, name_ptr, name_len, bytes_ptr)
metrics_record(name_ptr, name_len, value)
```

Every import checks the manifest's `capabilities` before executing. Denied
calls return an error code and increment `plugins.capability_denied`.

## 5. Build pipeline

```
engine plugin build plugins/my_plugin/
```

1. Compile the plugin crate to `wasm32-wasip1` via `cargo` in a subprocess.
2. Assert no `unsafe` blocks (source scan; plugins must be safe).
3. Validate `plugin.ron` against `api_version`.
4. Emit `target/plugins/my_plugin.wasm`.

AI-generated plugin flow:

1. AI writes `plugins/<name>/src/lib.rs` + `plugin.ron`.
2. `plugin build` is invoked through the Tool API (permission-gated).
3. If build succeeds, plugin is *staged* but not loaded.
4. User clicks "Install" in editor → loaded into wasmtime instance.
5. Hot-reload on edit: rebuild + swap the `wasmtime::Instance`, no process restart.

## 6. Safety rules

- WASM instance is given a **linear-memory cap** (default 64 MB).
- **No filesystem** outside declared globs. Paths are canonicalized and checked
  against the manifest before the host does the read.
- **No network** in v0.x. `net:*` capability exists in the schema but is always
  denied; when added, it will go through an explicit allowlist.
- **No threads** in v0.x. Plugins are single-threaded on a dedicated executor.
- **Panic isolation**: a panicking plugin traps in wasmtime; host catches,
  reports via metrics, keeps the game running.

## 7. Metrics (mandatory)

- `plugins.loaded` (gauge)
- `plugins.load_errors` (counter)
- `plugins.capability_denied` (counter, per capability)
- `plugins.call.ms` (histogram, per plugin)
- `plugins.traps` (counter)
- `plugins.memory_bytes` (gauge, per plugin)

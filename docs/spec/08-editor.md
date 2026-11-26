# 08 — Editor

## 1. Built on egui

The editor is a feature-gated module inside the same binary as the game.
`engine run --editor` opens it. `engine build` strips it (verified by symbol
check in CI).

## 2. Panels

- **Scene tree** — hierarchical view of ECS. Click → selects entity.
- **Inspector** — auto-generated from `bevy_reflect` + `ComponentRegistry`.
  Every registered component gets an editable UI **for free**. New module → new
  component → inspector row.
- **Viewport** — live render target with gizmos (translate/rotate/scale).
- **Asset browser** — directory tree of `assets/`, drag into scene.
- **Console** — filterable log, `tracing` events.
- **Metrics** — live graphs from `crates/metrics` (§09). Every module's gauges
  and histograms.
- **AI chat** — the agent loop UI. Tool calls visible inline with undo buttons.
- **Audio** — waveform/spectrogram per bus (§05).
- **Profiler** — frame timeline with per-system spans (tracy integration when
  enabled).

## 3. Inspector contract

For a component to appear in the inspector it must:

1. Be registered via `register_component::<T>()`.
2. Implement `bevy_reflect::Reflect` (mostly derive).
3. Optionally implement `InspectorUi` for custom widgets.

**No component is inspector-invisible.** If it's in the ECS and registered, it
has a UI. This is how the AI and the human see the same data.

## 4. Viewport

The viewport renders the game camera to an offscreen texture. Editor overlays
use a separate egui pass. Only one wgpu device is used; editor UI and viewport
share the same queue.

Frame layout:

- Game viewport texture (scene render) → present
- egui overlay (panels + gizmos) → composite

**No extra GPU pass** unless a debug viewport is active (§04.8).

## 5. Undo / redo

Editor mutations go through the same `CommandStack` as the AI (§06.4).
Ctrl-Z / Ctrl-Shift-Z bound to it. All panels that mutate state push a command.

## 9. CommandRegistry — CLI-UI parity

This section exists because of `docs/spec/02-architecture.md` §1.1 ("every UI
action is a CLI action").

### 9.1 Shape

```rust
pub struct CommandRegistry {
    cmds: BTreeMap<CommandId, CommandEntry>,
}

pub struct CommandEntry {
    pub id: CommandId,                 // "scene.spawn", "asset.reimport"
    pub domain: &'static str,          // "scene"
    pub action: &'static str,          // "spawn"
    pub args_schema: JsonSchema,
    pub apply: fn(&mut App, Value) -> Result<Command, CommandError>,
    pub undo:  fn(&mut App, &Command) -> Result<(), CommandError>,
}
```

### 9.2 Registration

Every panel that mutates state calls, in its `init()`:

```rust
commands.register(
    CommandId::new("scene", "spawn"),
    schema_of::<SpawnArgs>(),
    apply_spawn, undo_spawn,
);
```

### 9.3 What registration grants, automatically

A single `register` call yields all of:

1. A CLI subcommand: `engine scene spawn --prefab player --at 10,0`.
2. An AI tool: `command.scene.spawn` (visible to the agent, §06.2).
3. A keybindable action in `keymap.ron`.
4. An undo/redo entry in `CommandStack`.
5. A row in the editor's command palette (`Ctrl-P`).

If any of these is missing, `engine cli verify` (02-architecture §1.1.1) fails.

### 9.4 Rule for panel authors

A panel is a **view** over commands. It does not mutate state directly. Every
button, drag, toggle, and text field in a panel must:

1. Build a `Command` from user input.
2. Push it via `CommandStack::push`.
3. Let the framework apply it at the module's phase.

Direct `world.insert_component(...)` in a panel is a bug and will be rejected
in review.

## 6. Scene save

Saving writes `.scene.ron` (or `.prefab.ron` for selected subtree). Diffs are
textual and small — every commit is reviewable.

## 7. Layout state

Window layout persisted in `~/.config/silentgame/editor.ron`. Not committed.

## 8. Metrics (mandatory)

- `editor.frame.ms` (histogram)
- `editor.egui.pass_ms` (histogram)
- `editor.inspector.reflect_ms` (histogram)
- `editor.ai.requests` (counter)
- `editor.ai.undo_from_ai` (counter)
- `editor.commands.registered` (gauge) — must equal CLI subcommand count
- `editor.commands.cli_parity_errors` (counter) — must be 0 in CI

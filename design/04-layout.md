# 04 — Layout and scale

## 1. Editor shell (top → bottom)

| Region | Height | Contents |
|---|---|---|
| Menu bar | `h.menubar` 30 | app mark · menus (Arquivo/File … Ajuda/Help) · palette field (center, 340 px) · branch + phase |
| Toolbar | `h.toolbar` 32 | gizmo tools Q W E R · space · snap · play/pause/step (center) · frame stats (right) |
| Dock area | fill | panels (§2) |
| Status bar | `h.status` 22 | gate dots (correctness, perf, det, visual) · CommandStack depth · theme · `editor.frame.ms` |

Default dock (canvas boards 05/06):

```
┌──────────┬──────────────────────────────┬────────────┐
│ Scene    │ Viewport                      │ Inspector  │
│ │Assets  │                               │            │
│ 260 px   ├──────────────────────────────┤────────────┤
│          │ Console│Metrics│Profiler│Audio│ AI chat    │
│          │ 220 px                        │ 320 px     │
└──────────┴──────────────────────────────┴────────────┘
```

Minimum window 1180×720; below that, side docks collapse to icon rails.

## 2. Panels and docking

- Docking via `egui_dock` (`adr-pending` vs. in-house). Any panel can be a tab
  in any dock node, split, or floated.
- Panel = registered contribution (§4): `id`, title, icon, kind color,
  default dock slot, `ui(&mut self, ctx)`. Removing a module removes its
  panels; nothing else changes (spec mission §1, radical modularity).
- Each panel header: tabs + ⋯ (panel menu: settings, close, move to …).
- Layout persisted in `~/.config/silentengine/editor.ron` (spec §08.7).
  `Window ▸ Reset layout` restores the default above.
- Viewport overlays (view mode, shading, gizmo toggles, stats) are chips at
  the viewport corners, `rgba(surface.base, 0.88)`, not toolbar buttons.

## 3. Progressive disclosure (the density rule)

Dense, but not everything visible. Every action is placed at the **highest
level its frequency justifies**, never higher.

| Level | Where | Rule |
|---|---|---|
| N0 visible | toolbar, panel header | used every minute; ≤ 7 controls per bar |
| N1 hover / focus | row trailing slot | per-item actions (eye, lock, ⋯); also shown for keyboard focus |
| N2 overflow ⋯ | panel or component menu | grouped by separators, with shortcuts |
| N3 submenu / context | right-click, `▸` | variants and rare options; max 2 deep |
| N4 palette | `Ctrl+Shift+P` | everything, always, searchable; AI uses the same list |

Every N0–N3 item must also exist at N4. Shortcuts are shown wherever an
action appears. Promoting an action to a higher level is a design change
(log it in `NOTES.md`).

## 4. Command registry and contribution points

The thing that keeps the UI scalable. Like VS Code `contributes`:

```rust
app.register_command(Command {
    id: "scene.add_light.point",         // stable, dotted, lowercase
    title: "Add Point Light",            // localizable
    category: "scene",
    icon: Some(Icon::Light),
    kind: Some(Kind::Light),             // swatch color
    shortcut: None,
    enabled: |ctx| ctx.scene_loaded(),   // + reason when false
    run: |ctx| ctx.commands.push(AddEntity::point_light()),  // CommandStack
});

app.contribute_menu("Entity/Add/Light", "scene.add_light.point", Order(10));
app.contribute_toolbar(...);   // rare: needs N0 justification
app.contribute_context("scene_tree.row", ...);
app.contribute_panel(...);
app.contribute_theme_tokens(...);   // 05-themes §5
```

Rules:

1. `run` always goes through `CommandStack` (EXECUTION.md §6.3).
2. Menus, context menus, palette and AI tool list are **generated** from the
   registry; order by `Order`, group by path segment.
3. Plugin commands are prefixed `plugin.<id>.` and appear under their own
   submenu unless they contribute to a core path.
4. Shortcut conflicts are detected at registration and logged to Console;
   the core binding wins.
5. Metric: `editor.command.invocations{source}`.

## 5. Keyboard

Every interactive element is reachable by Tab; focus ring always visible on
keyboard navigation. Core bindings: Ctrl+Z / Ctrl+Shift+Z (undo/redo),
Ctrl+Shift+P (palette), Ctrl+P (go to asset), Ctrl+K Ctrl+T (theme picker),
Q W E R (gizmo), F (frame selection), F2 (rename), Del (remove), Ctrl+D
(duplicate), Ctrl+S (save scene). Remappable in `editor.ron`.

## 6. Language

UI strings are keys, not literals (`t!("menu.entity.add")`); English and
Portuguese ship first. Design widths assume +30 % text growth; labels
truncate with ellipsis and full text in tooltip.

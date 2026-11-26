# 05 — Theme system

Like VS Code / Zed: a theme is a **data file**, not code. Built-in, user and
plugin themes go through one loader.

## 1. Layers

```
palette (primitives) ──► tokens (semantic roles) ──► components (optional overrides)
                                          │
                                          ▼
                     ResolvedTheme (flat struct, built once per change)
                                          │
                                          ▼
                       egui::Visuals + egui::Style + editor widgets
```

Per-frame cost of theming: zero (one struct read). Switching theme rebuilds
`ResolvedTheme` once; metric `editor.theme.apply_ms`.

## 2. File format — `<id>.theme.ron`

```ron
Theme(
    id: "silent-ember",          // unique, [a-z0-9-]
    name: "Silent Ember",
    kind: Dark,                  // Dark | Light | HighContrast
    extends: None,               // or Some("silent-ember")
    schema: 1,
    palette: { "clay_900": "#1C1A17", ... },
    tokens:  { "surface.base": "clay_900", "text.muted": "#A39886", ... },
    components: { "gizmo.axis.x": "#C4553D" },
    metrics: (density: Compact, radius_scale: 1.0, ui_scale: 1.0),
)
```

- A token value is a palette key or `#RRGGBB` / `#RRGGBBAA`.
- `tokens` must define **every role** listed in 02-tokens §1 unless
  `extends` provides it.
- `components` keys are a closed list owned by widgets (`tree.row.selected`,
  `gizmo.axis.x`, `console.level.warn`, …); unknown key = error.
- `metrics.ui_scale` multiplies egui `pixels_per_point`; `radius_scale`
  multiplies `r.*`; `density` per 02-tokens §4.

Full examples: `themes/silent-ember.theme.ron`, `themes/silent-linen.theme.ron`.

## 3. Loader rules

1. Search order: built-in (embedded with `include_str!`) → plugin themes →
   `~/.config/silentengine/themes/*.theme.ron`. Same `id`: last wins.
2. `extends`: copy the base, apply declared keys. One level only; cycles or
   missing base = error.
3. Closed schema: missing role or unknown key → theme rejected, error in
   Console, `editor.theme.load_errors += 1`, previous theme stays. Never
   fall back to random colors.
4. Contrast (02-tokens §1.5): warning for user/plugin themes, **test failure**
   for built-ins.
5. Hot reload: file watcher reapplies on save within one frame.
6. Selection: `theme: "silent-ember"` or `theme_mode: System(dark:
   "silent-ember", light: "silent-linen")` in `editor.ron` (not committed).
7. Theme picker: `Ctrl+K Ctrl+T`, live preview while arrowing, Esc reverts.

## 4. Mapping to egui

`ResolvedTheme::apply(&self, ctx: &egui::Context)`:

| egui field | token |
|---|---|
| `visuals.panel_fill` | `surface.base` |
| `visuals.window_fill` | `surface.overlay` |
| `visuals.extreme_bg_color` | `surface.sunken` |
| `visuals.faint_bg_color` | `surface.raised` |
| `visuals.selection.bg_fill` | `selection.bg` |
| `visuals.selection.stroke` | `accent.fill` |
| `visuals.hyperlink_color` | `accent.text` |
| `visuals.warn_fg_color` / `error_fg_color` | `status.warn` / `status.error` |
| `widgets.noninteractive.bg_stroke` | `border.subtle` |
| `widgets.inactive.bg_fill` / `bg_stroke` / `fg_stroke` | `surface.raised` / `border.default` / `text.secondary` |
| `widgets.hovered.*` | `state.hover` / `border.strong` / `text.primary` |
| `widgets.active.*` | `state.active` / `focus.ring` / `text.primary` |
| `widgets.*.corner_radius` | `r.sm × radius_scale` |
| `window_shadow` / `popup_shadow` | e3 / e2 with `shadow.color` |
| `style.spacing.interact_size.y` | `h.control` |
| `style.spacing.item_spacing` | (`sp.2`, `sp.2`) |
| `style.text_styles` | 02-tokens §2 (Body=12 Inter, Monospace=12 JetBrains Mono, Small=11, Heading=13/600) |

Anything egui doesn't cover (kind swatches, axis colors, tool cards) is read
from `ResolvedTheme` by our widgets directly.

## 5. Plugins and modules

- A module that adds a data kind contributes `kind.<name>` with dark and
  light values: `app.contribute_theme_tokens(ThemeTokens { ... })`. Themes may
  override it; if they don't, the module's default is used.
- Plugin tokens live under `plugin.<id>.*`. Plugins **cannot** override core
  roles (`surface.*`, `text.*`, `accent.*`, `status.*`, `focus.*`).
- A plugin may ship whole themes (`themes/*.theme.ron` in its package); they
  go through §3 like user themes. Themes are data, so they need no
  capability (spec §07).

## 6. Adding a built-in theme (checklist)

1. Copy a built-in, change `id`, `name`, values.
2. `cargo test -p editor theme::` — schema + contrast green.
3. Screenshot the default layout at 2560×1440 → visual critic (README
   Verification). No finding above `minor`.
4. Add a line to `NOTES.md`.

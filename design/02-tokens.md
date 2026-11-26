# 02 — Tokens

Widgets read **semantic roles** only. Values live in `themes/*.theme.ron`.
Tables below are a mirror for reading; the `.ron` files win.

## 1. Color

### 1.1 Surfaces and borders

| Role | Ember (dark) | Linen (light) | Use |
|---|---|---|---|
| `surface.sunken` | #161411 | #EAE3D7 | inputs, console, wells |
| `surface.base` | #1C1A17 | #F4EFE6 | panel body (e0) |
| `surface.raised` | #24211D | #FBF8F2 | panel headers, tab bars, status bar (e1) |
| `surface.overlay` | #2B2823 | #FFFFFF | menus, popovers, palette (e2/e3) |
| `state.hover` | #322E28 | #E6DED0 | hovered row / button |
| `state.active` | #3B362F | #D3C8B6 | pressed |
| `selection.bg` | #3A3050 | #E3DAF7 | selected rows, active tool, AI card header |
| `border.subtle` | #2F2B26 | #E2D9CA | panel separators |
| `border.default` | #3B362F | #D3C8B6 | control outlines |
| `border.strong` | #4D463D | #B8AB96 | overlays, dashed "add" buttons |

### 1.2 Text and accent

| Role | Ember | Linen | Use |
|---|---|---|---|
| `text.primary` | #EDE6DA | #2A251F | values, names |
| `text.secondary` | #C2B8A8 | #4A4237 | labels, inactive tabs |
| `text.muted` | #A39886 | #625849 | shortcuts, counts, hints |
| `text.disabled` | #6B6256 | #A89C8A | disabled only (exempt from contrast) |
| `text.on_accent` | #FFFFFF | #FFFFFF | text on `accent.fill` |
| `accent.fill` | #7A5AD6 | #6B4BC9 | primary button, active tab bar, slider fill |
| `accent.fill_hover` | #8A6BE3 | #5A3BB8 | hover of the above |
| `accent.text` | #B9A3F5 | #5A3BB8 | links, modified-value labels, AI tool names |
| `focus.ring` | #9D80EE | #7A5AD6 | 2 px keyboard focus outline, offset 1 px |

### 1.3 Status

| Role | Ember | Linen | Means |
|---|---|---|---|
| `status.ok` | #8FB573 | #3F6629 | gate Passed, success |
| `status.warn` | #E0A458 | #82510F | near budget, WARN log |
| `status.error` | #E07E66 | #9E3B27 | failure, blocker, ERROR log, validation |
| `status.info` | #6FA3C7 | #2E5F82 | INFO log, N/A gate, hint |

Status is never color alone: pair with a dot + label or an icon.

### 1.4 Data kinds

Small swatch (8×8, r=2) before a name in tree, inspector headers, asset
browser, metrics legends. One color per **kind**, not per instance.

| Role | Ember | Linen |
|---|---|---|
| `kind.entity` | #D8CFC0 | #6B5F4F |
| `kind.transform` | #6FA3C7 | #2E5F82 |
| `kind.mesh` | #8FB573 | #3F6629 |
| `kind.material` | #C08AB8 | #8E4F86 |
| `kind.light` | #E0C068 | #7A6010 |
| `kind.audio` | #5FB3A6 | #1F6E63 |
| `kind.plugin` | #E0A458 | #82510F |
| `kind.ai` | #B9A3F5 | #5A3BB8 |

New kinds come from modules (05-themes §5) and must declare both values.
Metrics charts with > 8 series reuse kinds cyclically **plus** a dash pattern.

### 1.5 Contrast contract (tested)

WCAG 2.x ratio, computed by the theme test for both built-ins:

| Foreground | On | Minimum |
|---|---|---|
| `text.primary`, `text.secondary` | every surface + `state.hover` + `selection.bg` | 4.5 |
| `text.muted` | sunken, base, raised, overlay, hover | 4.5 |
| `accent.text` | every surface + `selection.bg` | 4.5 |
| `text.on_accent` | `accent.fill` | 4.5 |
| `status.*` | sunken, base, raised, overlay | 4.5 |
| `kind.*`, `axis.*`, `focus.ring` | base | 3.0 (non-text) |

Rule that follows: on a selected row, secondary info uses `text.secondary`,
never `text.muted`.

### 1.6 Viewport

`viewport.bg`, `viewport.grid`, `axis.x|y|z` (terracotta / sage / dusty blue,
never pure red/green). Gizmo selection handle uses `accent.fill`.

## 2. Typography

Families: **Inter** (UI), **JetBrains Mono** (numbers, paths, IDs, logs, tool
calls). Both OFL, embedded in the editor binary; editor feature only, so
`engine build` strips them with the editor.

| Token | Size / line / weight | Family | Use |
|---|---|---|---|
| `type.display` | 20 / 26 / 600 | Inter | dialog title, first-run |
| `type.title` | 15 / 20 / 600 | Inter | section heading in a panel |
| `type.heading` | 13 / 18 / 600 | Inter | component header, group |
| `type.body` | 12 / 16 / 400 | Inter | **default**: labels, menus, rows |
| `type.body.strong` | 12 / 16 / 500 | Inter | active tab, modified value |
| `type.caption` | 11 / 14 / 400 | Inter | shortcuts, counts, status bar |
| `type.mono` | 12 / 16 / 400 | Mono | numeric fields, paths |
| `type.mono.small` | 11 / 14 / 400 | Mono | console, metrics labels |

Rules: weights 400/500/600 only. Nothing below 11 px. Uppercase only for
11 px section labels with 0.06 em tracking. Truncate paths in the middle,
names at the end. Numbers use tabular figures.

## 3. Space (4 px grid)

| Token | px | Use |
|---|---|---|
| `sp.1` | 2 | icon ↔ dense text |
| `sp.2` | 4 | between sibling controls |
| `sp.3` | 6 | button horizontal padding, row gap |
| `sp.4` | 8 | panel padding |
| `sp.5` | 12 | between groups |
| `sp.6` | 16 | between sections |
| `sp.7` | 24 | dialog padding |

Tree indent: 16 px per level.

## 4. Sizes

| Token | px | Use |
|---|---|---|
| `h.xs` | 18 | chip, badge, inline button in card |
| `h.row` | 22 | tree row, menu item, inspector field |
| `h.control` | 24 | button, input, dropdown |
| `h.header` | 26 | component header in inspector |
| `h.tabbar` | 28 | panel tab bar |
| `h.menubar` | 30 | top menu bar |
| `h.toolbar` | 32 | main toolbar |
| `h.status` | 22 | status bar |
| `h.touch` | 44 | dialogs / first-run only |
| `icon.sm` / `icon.md` | 12 / 14 | icons, 1.25 px stroke |

`metrics.density` in the theme: `Compact` (values above) or `Comfortable`
(+2 px on row/control/header, +1 px on type.body). Nothing else changes.

## 5. Radius and elevation

| Token | px | Use |
|---|---|---|
| `r.xs` | 2 | swatches, sparkline bars |
| `r.sm` | 3 | controls, rows |
| `r.md` | 5 | cards, menus |
| `r.lg` | 8 | dialogs, command palette |

| Level | Surface | Border | Shadow | Use |
|---|---|---|---|---|
| e0 | base | subtle | none | panels |
| e1 | raised | subtle | none | headers, tab bars |
| e2 | overlay | strong | 0 8 24 `shadow.color` | menus, popovers, toasts |
| e3 | overlay | strong | 0 16 40 `shadow.color` | dialogs, palette |

Depth comes from luminance first; shadows only at e2+.

## 6. Motion

Hover/press: instant. Menus/popovers: 80 ms fade. Panel collapse: 120 ms.
Toasts: 160 ms in, auto-dismiss 6 s (never for errors). `prefers reduced
motion` (editor setting) sets all to 0. No animation inside the viewport UI.

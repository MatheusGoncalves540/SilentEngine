# 03 — Components

Every component: anatomy, size tokens, states, roles. States always in this
order of precedence: disabled > pressed > focused > hovered > selected > rest.
Implementation lives in `crates/editor/src/widgets/`; each widget takes
`&ResolvedTheme`, never a color.

Board 04 on the canvas shows all of these in both themes.

## 1. Button

| Variant | Rest | Hover | Use |
|---|---|---|---|
| primary | `accent.fill` bg + border, `text.on_accent` | `accent.fill_hover` | one per region (Apply, Install) |
| secondary | `surface.raised` bg, `border.default` | `state.hover`, `border.strong` | default |
| ghost | transparent | `state.hover` | toolbars, low-emphasis |
| danger | transparent, `status.error` border + text | `status.error` 12 % bg | destructive (prefer Undo toast) |

Height `h.control`, padding `sp.3`, radius `r.sm`, `type.body` (primary uses
500). Focus: 2 px `focus.ring`, offset 1. Disabled: `text.disabled`, 60 %
opacity, tooltip says **why**.

**Icon button**: 24×24, icon 14, ghost by default; toggled = `selection.bg` +
`accent.fill` border + `accent.text` icon. Mandatory tooltip after 500 ms:
`Name  Shortcut`. Groups of related icon buttons are separated by a 1×16
`border.default` divider.

## 2. Text field

`h.control`, `surface.sunken`, `border.default`, radius `r.sm`, padding
`0 8`. Focus: border → `focus.ring`. Error: border + label → `status.error`,
message below in `type.caption` `status.error`. Placeholder `text.muted`.

## 3. Numeric drag field

Mono, right aligned, tabular. Drag horizontally to change; Shift ×0.1;
Ctrl snaps to the theme-independent snap step; double-click → text edit;
Esc reverts. Each change during a drag coalesces into **one**
`CommandStack` entry on release (spec §08.5).

**Vector field** (Vec2/3/4, Quat as Euler): one field per axis, each with a
14–16 px axis tag colored `axis.x|y|z`, letter in `text.on_accent`.

**Modified vs prefab**: label in `accent.text` + trailing `•`. Context menu:
Revert to prefab.

## 4. Dropdown / select

Looks like a secondary button with a chevron. Opens a menu (§8) anchored
below. Over 12 options → menu gets a filter field at the top.

## 5. Toggle, checkbox, slider

- Toggle: 28×16 track, 12 px knob. On = `accent.fill`; off = `border.strong`.
  Real `role=switch` semantics in AccessKit.
- Checkbox: 12 px, check in `accent.fill`.
- Slider: 4 px track `border.default`, fill `accent.fill`, paired with a
  52 px numeric field.

## 6. Tree row (scene tree, asset browser)

`h.row`, indent 16/level, parts: caret (10) · kind swatch (8) · name ·
trailing meta. States: hover = `state.hover` and reveals N1 actions
(visibility, lock, ⋯) in the meta slot; selected = `selection.bg` +
2 px left inset bar `accent.fill`; hidden entity = `text.muted`;
multi-select keeps the bar on every selected row. Drag target: 1 px
`accent.fill` line between rows or `selection.bg` on the row (reparent).
Keyboard: ↑↓ move, ←→ collapse/expand, F2 rename, Del remove (undoable).

## 7. Inspector

Per component (auto-generated from reflect, spec §08.3):

- **Header** `h.header`, `surface.raised`, `type.heading`: caret · kind swatch ·
  name · **source module** (`type.mono.small`, `text.muted`, e.g. `render`,
  `plugin.kelp`) · ⋯ menu (Reset, Copy, Paste values, Remove, Docs).
- **Body** grid `88px | 1fr`, row gap 4, padding 8. Label `text.secondary`,
  value widget from §2–5 chosen by reflect type.
- Transform expanded by default; every other component collapsed; expansion
  state persisted per component type in `editor.ron`.
- Footer: dashed `border.strong` button "+ Add component" → palette scoped
  to components (§9).
- A custom `InspectorUi` must still use these widgets and tokens.

## 8. Menu, submenu, context menu

e2 surface. Item `h.row`, padding `0 8`, radius `r.sm`; parts: check slot (12)
· label · right slot (shortcut in `type.mono.small` `text.muted`, or `▸` for
submenu). Hover/keyboard-active item = `accent.fill` + `text.on_accent`.
Separators 1 px `border.subtle`. Max **2 submenu levels**. Disabled items
stay visible with reason in tooltip. Items are **generated from the command
registry** (04-layout §4); no hand-built menus.

## 9. Command palette

`Ctrl+Shift+P`. e3, width 560, radius `r.lg`, top third of the window.
Input row 34 px; results 26 px rows: label · category (`type.mono.small`) ·
shortcut. Footer hint row. Prefixes: `>` commands (default), `@` entities,
`#` assets, `:` settings, `?` ask the AI. Fuzzy match, recent first. The AI's
tool list is the same registry, so anything in the palette is AI-callable
(subject to spec §06 permissions).

## 10. AI tool-call card

Inline in the AI chat (spec §08.2). Header `h.row` on `selection.bg`: status
dot (ok/warn/error) · tool name in `type.mono` `accent.text` · summary ·
**Undo** (secondary, `h.xs`). Optional body on `surface.sunken`, mono small:
diff summary + `CommandStack #id`. User messages: right-aligned bubble,
`surface.raised`. AI prose: plain text, `text.secondary`. A tool needing
user confirmation (`plugin.install`, spec §06/§07) renders as a confirm
banner (§12), never auto-accepted.

## 11. Metrics and profiler

- Sparkline / histogram bars: `accent.fill`; samples over budget:
  `status.warn`; over 2× budget: `status.error`.
- Budget line: 1 px dashed `status.warn`. Axis labels `type.mono.small`
  `text.muted`.
- Profiler spans colored by **module kind** (§02.1.4); span label in
  `text.primary` on 30 % mix.
- Every number shown has its unit (ms, MB, dBFS).

## 12. Feedback

- **Badge / chip** `h.xs`, `surface.raised`, `border.default`, dot + label.
- **Toast** e2, bottom-right, max 3 stacked, dot + title + detail + one
  action link. Undoable actions show Undo here instead of a confirm dialog.
- **Confirm banner** (user-gated actions): inline in the panel that asked,
  text + secondary "Decline" + primary action. Shows requested capabilities
  in mono.
- **Dialog** e3: only for irreversible or blocking choices. `h.touch`
  buttons, `sp.7` padding.

## 13. Tabs and panel header

Tab bar `h.tabbar`, `surface.raised`. Tab: `type.body`, padding `0 10`;
active = `text.primary` 500 + 2 px bottom `accent.fill`; inactive =
`text.secondary`. Right side: panel ⋯ menu (N2). Tabs are draggable for
docking (04-layout §2).

## 14. Icons

Single family, stroke 1.25 px, 12/14 px, `currentColor`. Candidate:
`egui-phosphor` (MIT) — `adr-pending` until chosen. Never emoji. Every
icon-only control has an accessible name.

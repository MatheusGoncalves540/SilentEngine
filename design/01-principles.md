# 01 — Principles

On conflict, top wins (mirrors `docs/spec/01-mission.md`: observable
correctness first).

1. **The viewport is the hero.** UI chrome recedes: warm neutral surfaces, low
   contrast borders, no gradients, no decorative color. Anything that competes
   with the render for attention must justify itself.
2. **Color is meaning.** Violet = focus / selection / AI. Status colors = gate
   and log state. Data-kind colors (02-tokens §1.4) = what a thing *is*
   (mesh, light, plugin…). If a color carries no meaning, it is a neutral.
3. **Dense data, calm surface.** 22 px rows and 12 px text so a lot fits; but
   only frequent actions are visible. Everything else is one level down
   (04-layout §3). Never "everything on screen".
4. **Every action exists once.** Actions are commands in a registry. Menus,
   toolbars, context menus, shortcuts, the palette and the AI are just entry
   points to the same command. This is what makes the UI scale forever: a new
   module adds commands + contributions, never edits a central menu.
5. **Same data for human and AI.** The inspector shows exactly the reflected
   data the AI sees (spec §08.3). AI actions render inline as tool-call cards
   with Undo (03-components §10). AI never gets a private UI.
6. **Verifiable, not "looks fine".** Tokens are data, contrast is a test, theme
   schema is a test, editor screenshots go to the visual critic. No literal
   colors in widget code.

## Anti-patterns (each is a `major` in review)

- A hex / `Color32::from_rgb` outside `crates/editor/src/theme/`.
- A new top-level toolbar button for an action used less than every few
  minutes — put it in a menu or overflow.
- A submenu deeper than 2 levels.
- An icon-only button without tooltip (name + shortcut).
- Red/green as the only difference between two states (use luminance too).
- A modal dialog for something undoable — do it, show a toast with Undo.
- Shadows on anything below elevation e2.

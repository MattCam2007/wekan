# WeKan Design Tokens — the theme contract

Status: **Plan — not yet implemented** · Related: `docs/Theme/UI-Redesign-Plan.md` (why and
in what order), `docs/Theme/Theme-Nebula.md`, `docs/Theme/Theme-Meridian.md`.

This is the **contract** between component CSS and themes:

- **Component CSS may only reference tokens.** It must never contain a color literal, and no
  bare `border-radius` / shadow / padding literal where a token exists.
- **A theme redefines layer 1, layer 2 and layer 2F (form).** It must never contain a
  selector for a component.

> **Revision 1.** An earlier draft restricted themes to color only. That was wrong — it meant
> a new theme could only recolor a square, flat layout. Shape, elevation, separation and
> density are now first-class themeable tokens (layer 2F below). See
> `docs/Theme/Form-Redesign.md` for the reasoning and the per-component geometry.
>
> **Revision 2.** Four corrections from the review in `UI-Redesign-Plan.md`:
> a **typeface** section (the app has no `font-family` on `body` at all);
> `color-mix()` **banned inside token definitions**, where its documented fallback silently
> does not work; theme blocks written `:root.wk-theme-<name>` so precedence does not depend on
> import order; and Classic's values seeded from the **rendered cascade** (`boardColors.css`
> outranks component CSS on every board) rather than from whichever file declares them.

`tests/themeTokens.test.cjs` enforces that every theme defines every token listed here.
Adding a token means adding it to *every* theme file in the same commit.

Prefix: `--wk-`. Existing public variables (`--theme-accent`, `--theme-accent-2`,
`--list-width`, `--swimlane-height`, `--wekan-ui-*`) keep their names for backward
compatibility and are re-sourced from these tokens — see `UI-Redesign-Plan.md` §3.3.

---

## Layer 1 — Primitives (per theme)

Raw colors with no assigned meaning. Never referenced by component CSS.

```
--wk-palette-neutral-0 … -1000     11 steps, lightest → darkest
--wk-palette-accent-100 … -900      7 steps of the theme's primary hue
--wk-palette-success-{300,500,700}
--wk-palette-warning-{300,500,700}
--wk-palette-danger-{300,500,700}
--wk-palette-info-{300,500,700}
```

A theme may add its own extra primitives (Nebula adds a violet and a cobalt ramp). Extra
primitives are theme-private and never referenced outside their own theme file.

---

## Layer 2 — Semantic (per theme)

The vocabulary component CSS actually speaks. **Every theme must define all of these.**

### Surfaces

| Token | Meaning |
|---|---|
| `--wk-surface-app` | Outermost page background (behind everything) |
| `--wk-surface-canvas` | Board canvas — the area lists sit on |
| `--wk-surface-1` | Default raised surface: cards, popups, panels |
| `--wk-surface-2` | Surface raised above `-1`: list columns, popup headers |
| `--wk-surface-3` | Highest: dropdown menus, drag ghost |
| `--wk-surface-sunken` | Recessed: input wells, code blocks, empty list drop zones |
| `--wk-surface-overlay` | Modal scrim (translucent) |
| `--wk-surface-hover` | Hover wash over `-1` (translucent) |
| `--wk-surface-active` | Pressed/selected wash (translucent) |
| `--wk-surface-selected` | Persistent selected-row background |

Translucent tokens must be `rgb(… / α)` so they compose over any surface.

### Text

| Token | Minimum contrast |
|---|---|
| `--wk-text-primary` | 7:1 on `-1`, `-2`, `canvas` |
| `--wk-text-secondary` | 4.5:1 on the same |
| `--wk-text-muted` | 4.5:1 on the same (metadata, timestamps) |
| `--wk-text-disabled` | 3:1 — non-interactive only |
| `--wk-text-on-accent` | 4.5:1 on `--wk-accent` |
| `--wk-text-link` / `--wk-text-link-hover` | 4.5:1 on `-1` and `canvas` |
| `--wk-text-inverse` | For inverted chips/tooltips |

### Borders

`--wk-border-subtle` (dividers, ≥1.5:1) · `--wk-border-default` (control outlines, ≥3:1) ·
`--wk-border-strong` (emphasis) · `--wk-border-focus` (≥3:1 against adjacent surfaces).

### Accent and state

```
--wk-accent            --wk-accent-hover      --wk-accent-active
--wk-accent-subtle     (tinted background for selected/active nav)
--wk-accent-focus      (focus ring color, ≥3:1 on every surface)
--wk-success / -subtle / -text
--wk-warning / -subtle / -text
--wk-danger  / -subtle / -text
--wk-info    / -subtle / -text
```

`-subtle` is a low-saturation background for badges/banners; `-text` is the AA-passing
foreground to use **on** that `-subtle` background.

### Elevation colors

`--wk-shadow-ambient`, `--wk-shadow-direct` — the two shadow colors composed by
`--wk-elev-*`.

---

## Layer 2F — Form (per theme)

Shape, depth and density. **Every theme must define all of these.** This is the layer that
makes themes structurally different rather than merely recolored — see
`docs/Theme/Form-Redesign.md`.

### Shape — by role, not by pixel value

| Token | Applies to | Classic *(as rendered)* | Meridian | Nebula |
|---|---|---:|---:|---:|
| `--wk-shape-column` | list column, swimlane panel | `0` | `10px` | `16px` |
| `--wk-shape-card` | minicard, board tile | **`7px`** | `8px` | `14px` |
| `--wk-shape-popup` | pop-over, modal, dropdown | `6px` | `10px` | `16px` |
| `--wk-shape-control` | button, input, select | `4px` | `6px` | `10px` |
| `--wk-shape-chip` | label, date, badge, count | `4px` | `6px` | `999px` |
| `--wk-shape-avatar` | avatars only | `50%` | `50%` | `50%` |

**Classic's values are the ones it renders at, which is not always what a component file says.**
`minicard.css` declares `border-radius: 4px`, but every board carries a `board-color-*` class
and `boardColors.css:228` overrides it to `7px` at higher specificity. Seed Classic from the
cascade, not from a single file — `UI-Redesign-Plan.md` §1.3.

The older `--wk-radius-sm|md|lg|full` remain as the raw scale; the `--wk-shape-*` roles are
what component CSS references, so a theme changes a component class without hunting pixels.

### Separation — how a surface reads as raised

```
--wk-sep-strategy      documentation only: shadow | rim | both
--wk-sep-card          box-shadow for a card at rest
--wk-sep-card-hover    … hovered
--wk-sep-card-drag     … dragging
--wk-sep-column        … list column / swimlane panel
--wk-sep-popup         … popup / dropdown / modal
--wk-rim-light         inset top-edge highlight (dark themes; empty on light)
```

**Light themes use `shadow`. Dark themes must use `rim` plus a surface value step** — a black
shadow over a dark surface is close to invisible (2.4% of range on the current dark theme,
1.2% on Nebula, versus 12.9% on light). Encoding this is the fix for "the dark theme looks
flat"; `tests/themeForm.test.cjs` asserts dark themes define a non-empty `--wk-rim-light` and
a measurable lightness step between `--wk-surface-2` and `--wk-surface-1`.

### Density

```css
--wk-density: 1;                          /* comfortable (default) */
.wk-density-compact { --wk-density: 0.75; }
```

Consumed as `calc(var(--wk-space-N) * var(--wk-density))` in padding and gutters. Interactive
targets must stay ≥32px even at `0.75`.

---

## Layer 3 — Component aliases (theme-agnostic)

Defined once in `_tokens.css` in terms of layer 2. Component CSS prefers these; they exist so
a component's intent is legible and so a future theme could special-case one component
without a broad override.

```css
--wk-card-bg:            var(--wk-surface-1);
--wk-card-border:        var(--wk-border-subtle);
--wk-card-shadow:        var(--wk-elev-1);
--wk-list-bg:            var(--wk-surface-2);
--wk-list-header-bg:     var(--wk-surface-2);
--wk-board-header-bg:    var(--wk-accent);
--wk-board-header-text:  var(--wk-text-on-accent);
--wk-sidebar-bg:         var(--wk-surface-1);
--wk-popup-bg:           var(--wk-surface-3);
--wk-input-bg:           var(--wk-surface-sunken);
--wk-input-border:       var(--wk-border-default);
--wk-swimlane-bg:        var(--wk-surface-canvas);
```

---

## Non-color scales (theme-agnostic)

Defined once in `_tokens.css`. Themes may override **only** `--wk-radius-*` and
`--wk-density`.

### Space — 4px base

```css
--wk-space-0: 0;     --wk-space-1: 4px;   --wk-space-2: 8px;   --wk-space-3: 12px;
--wk-space-4: 16px;  --wk-space-5: 24px;  --wk-space-6: 32px;  --wk-space-8: 48px;
```

Replaces the current `3/5/6/8/10/12/14/21/23px` mix. Where a legacy value has no exact match,
Phase 2b keeps the literal (spacing is not a color) and Phase 2c snaps it to the scale.

### Typeface

The app has **no** `font-family` on `body` today, and `client/components/main/fonts.css` is
100% commented out (`UI-Redesign-Plan.md` §1.4). These tokens are that missing default:

```css
--wk-font-sans: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue",
                "Noto Sans", Arial, sans-serif;
--wk-font-mono: ui-monospace, SFMono-Regular, "SF Mono", Menlo, Consolas,
                "Liberation Mono", monospace;
```

- `body { font-family: var(--wk-font-sans); }` in `_tokens.css`. A system stack — no bundled
  asset, no network fetch, no swap-induced layout shift.
- `--wk-font-mono` replaces the ad-hoc `monospace`, `lucida console, monospace` and `Courier`
  in `attachments.css`, `layouts.css`, `originalPositionsView.css` and `globalSearch.css`.
- **`--wekan-ui-font` (#4759) still wins.** `uiFont.css` applies it with `!important` under the
  `has-ui-font` class, so a user's explicit font preference continues to override this default.
- A theme **may** override `--wk-font-sans`. With `--wk-radius-*` / `--wk-shape-*` and
  `--wk-density`, that is the complete list of non-color things a theme may touch. Neither
  Nebula nor Meridian overrides it.

### Type

```css
--wk-text-xs:  12px;  --wk-text-sm:  13px;  --wk-text-base: 14px;
--wk-text-md:  16px;  --wk-text-lg:  18px;  --wk-text-xl:   20px;
--wk-text-2xl: 24px;
--wk-leading-tight: 1.25;  --wk-leading-normal: 1.45;  --wk-leading-relaxed: 1.6;
--wk-weight-normal: 400;   --wk-weight-medium: 500;    --wk-weight-bold: 700;
```

These are the **54** current sizes collapsed to 7 (an earlier draft said ~20; that count
excluded `boardColors.css` and the `!important` variants). Most existing values have no exact
match and must be *snapped* to the nearest step, which moves pixels — that is why the collapse
is its own phase rather than part of the mechanical color migration.

They must remain compatible with the #4759 root font-size preset (`--wekan-ui-font-size` scales
`body`), so **prefer `rem` at the point of use** where a component should scale with the user's
size preset, and `px` only where it must not (icon boxes, 1px rules). Note that
`tests/noViewportSpacing.test.cjs` rejects `clamp()` outright, so a "fluid" type scale is not
available and is not wanted.

### Radius — raw scale (themes override via the `--wk-shape-*` roles above)

```css
--wk-radius-sm: 4px;  --wk-radius-md: 6px;  --wk-radius-lg: 10px;  --wk-radius-full: 999px;
```

### Elevation

```css
--wk-elev-0: none;
--wk-elev-1: 0 1px 2px var(--wk-shadow-ambient);
--wk-elev-2: 0 1px 3px var(--wk-shadow-ambient), 0 2px 6px var(--wk-shadow-direct);
--wk-elev-3: 0 2px 6px var(--wk-shadow-ambient), 0 8px 16px var(--wk-shadow-direct);
--wk-elev-4: 0 4px 12px var(--wk-shadow-ambient), 0 16px 32px var(--wk-shadow-direct);
```

Usage: rest card `-1`, hovered card / list header `-2`, popup / dropdown `-3`, modal / drag
ghost `-4`.

### Motion

```css
--wk-motion-fast: 120ms;  --wk-motion-base: 180ms;  --wk-motion-slow: 260ms;
--wk-ease-standard: cubic-bezier(0.2, 0, 0, 1);
--wk-ease-decelerate: cubic-bezier(0, 0, 0, 1);

@media (prefers-reduced-motion: reduce) {
  :root { --wk-motion-fast: 0ms; --wk-motion-base: 0ms; --wk-motion-slow: 0ms; }
}
```

### Focus

```css
--wk-focus-ring: 0 0 0 2px var(--wk-surface-1), 0 0 0 4px var(--wk-accent-focus);
--wk-focus-ring-offset-canvas: 0 0 0 2px var(--wk-surface-canvas), 0 0 0 4px var(--wk-accent-focus);
```

### Z-index

Collect the existing ad-hoc values into one ladder so stacking bugs stop being guesswork:

```css
--wk-z-base: 0;      --wk-z-sticky: 100;   --wk-z-drag: 200;
--wk-z-sidebar: 300; --wk-z-header: 400;   --wk-z-popup: 500;
--wk-z-modal: 600;   --wk-z-toast: 700;
```

---

## Authoring rules

1. **No color literal in component CSS.** Permitted residue: `transparent`, `currentColor`,
   `#fff`/`#000` *inside* a shadow or gradient definition that a token already colors.
2. **No component selector in a theme file.** A theme is a flat list of custom properties
   inside one `:root` or `:root.wk-theme-<name>` block.
3. **Write the theme selector as `:root.wk-theme-<name>`, never a bare `.wk-theme-<name>`.**
   `:root` and a single class are both specificity (0,1,0), so a bare class merely *ties* with
   the Classic block and the winner falls to source order — i.e. to the order of imports in
   `client/styles.js`. `:root.wk-theme-nebula` is (0,2,0) and wins regardless.
   `tests/themeSelector.test.cjs` enforces this.
4. **`color-mix()` is banned inside custom-property definitions.** It is allowed in component
   longhands, with a static fallback declaration immediately above:

   ```css
   .btn:hover { background: #2471a3; }                                   /* fallback */
   .btn:hover { background: color-mix(in oklab, var(--wk-accent) 88%, black); }
   ```

   The fallback works there because an unsupported `color-mix()` makes the *declaration* a
   parse error, so it is dropped and the previous one wins. **Custom-property values are not
   parsed at declaration time** — they are stored as raw tokens and substituted at `var()`
   time. So `--wk-accent-hover: color-mix(…)` parses fine everywhere, and then
   `background: var(--wk-accent-hover)` becomes invalid-at-computed-value-time on a
   non-supporting browser → `unset` → transparent, with the "fallback" discarded. Theme files
   use **literal derivation** (write the computed value with a comment giving its origin). If a
   derived ramp is genuinely needed, wrap the theme block in
   `@supports (color: color-mix(in oklab, red 50%, blue))` and provide a literal-valued block
   before it. `tests/themeTokens.test.cjs` fails on `color-mix(` inside any `--wk-*` definition
   that is not inside such an `@supports`.
5. **Keep logical properties.** `inset-inline-start`, `margin-inline-end`, `padding-inline` —
   the codebase is RTL-correct today and tokens must not regress that.
6. **No viewport units, no `clamp()`.** `tests/noViewportSpacing.test.cjs` scans every
   stylesheet. Allowed: `100vh`/`100vw` for a full-viewport box, `max-*` caps, and shrink-only
   `min(400px, 52vw)`. Sizes are fixed px or `calc(token * var(--wk-density))`.
7. **Adding a token = adding it to every theme file in the same commit.**
   `tests/themeTokens.test.cjs` fails otherwise.
8. **New foreground/background pairings must be added to
   `tests/themeContrast.test.cjs`** with their required minimum.
9. **Seed Classic from the rendered cascade**, not from whichever component file declares the
   property. `boardColors.css` outranks component rules on every board (§Shape above,
   `UI-Redesign-Plan.md` §1.3).

---

## Worked example

Before (`minicard.css`, current):

```css
.minicard {
  padding: 8px 14px 3px;
  background-color: #fff;
  box-shadow: 0 2px 3px rgba(0,0,0,0.15);
  border-radius: 4px;
  color: #4d4d4d;
}
```

After Phase 2b (color only — geometry untouched, so the render is byte-identical in Classic):

```css
.minicard {
  padding: 8px 14px 3px;
  background-color: var(--wk-card-bg);
  box-shadow: var(--wk-card-shadow);
  border-radius: var(--wk-radius-sm);
  color: var(--wk-text-secondary);
}
```

After Phase 2c with form tokens (geometry tokenized; Classic pins `--wk-shape-card: 7px` — the
value `boardColors.css` actually renders, not `minicard.css`'s shadowed `4px` — so Classic does
not move, while Nebula renders a `14px` rim-lit card and Meridian an `8px` soft-shadowed one
from the *same* rule):

```css
.minicard {
  padding: calc(var(--wk-space-3) * var(--wk-density));
  background-color: var(--wk-card-bg);
  box-shadow: var(--wk-sep-card);
  border-radius: var(--wk-shape-card);
  color: var(--wk-text-secondary);
  transition: box-shadow var(--wk-motion-fast) var(--wk-ease-standard),
              transform  var(--wk-motion-fast) var(--wk-ease-standard);
}
.minicard:hover { box-shadow: var(--wk-sep-card-hover); }
```

The same file now renders correctly in Classic, Nebula and Meridian — with **different shape
and depth per theme** — and no theme-specific rule anywhere. That is the entire point.

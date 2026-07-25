# Form redesign — shape, depth and rhythm

Status: **Plan — not yet implemented** · Related: `docs/Theme/UI-Redesign-Plan.md`,
`docs/Theme/Design-Tokens.md`, `docs/Theme/Theme-Nebula.md`, `docs/Theme/Theme-Meridian.md`.

Color was the easy half. This document is the other half: **why the UI reads as square, boxy
and flat, and the exact geometry that fixes it.**

The first draft of this plan treated a theme as a palette and deferred form to a final
"Phase 6". That was wrong: recoloring a boxy layout produces a boxy layout in a new color.
Shape, depth and rhythm are now **themeable tokens** and ship *with* each theme.

---

## 1. Why it looks flat — measured, not opinion

### 1.1 The columns are hard rectangles with no gutter

```css
/* client/components/lists/list.css — current */
.list {
  background: #dedede;
  border-inline-start: 1px solid #ccc;   /* the ONLY separation between columns */
  padding: 0;
  /* no border-radius, no box-shadow, no margin */
}
.list-header {
  padding: 28px 21px 6px !important;
  background-color: #e4e4e4 !important;  /* flat grey bar, no radius, no separation */
}
```

List columns are full-bleed rectangles butted edge-to-edge, divided by a **1px line**. There
is no radius, no gutter, no elevation. That single declaration is most of the "boxy".

### 1.2 The swimlane header is a full-bleed strip

```css
/* client/components/swimlanes/swimlanes.css — current */
.swimlane .swimlane-header-wrap {
  background-color: #ccc;
  min-height: 33px;                      /* edge-to-edge bar, no radius, centered text */
}
```

Those saturated orange/cyan bands spanning the entire viewport are the loudest thing on
screen while carrying the least information.

### 1.3 The shadow is real but mathematically invisible on dark surfaces

```css
.minicard { box-shadow: 0 2px 3px rgba(0,0,0,0.15); }
```

A **black** shadow at 15% alpha, composited over the surface behind it:

| List background | Shadow composites to | Visible delta |
|---|---|---:|
| Classic light `#dedede` (222) | 189 | **12.9%** of range |
| Current dark theme `~#2a2a2a` (42) | 36 | **2.4%** |
| Nebula list `#141024` (20) | 17 | **1.2%** |

The same shadow is **10× less visible on dark than on light**. This is why the dark screenshot
looks flat: the depth cue is present in the CSS and absent on screen. Dark themes cannot fix
this by darkening the shadow — there is no room below black. They need a **different
separation strategy** (§3.2).

### 1.4 Chips are sharp and starved

```css
.card-date { border-radius: 4px; padding: 1px 3px; }        /* 1px vertical padding */
.minicard .badges .badge.is-finished { padding: 0 6px; border-radius: 6px; }
.card-label { border-radius: 4px; padding: 4px 10px; }
/* minicard label square */                                  height: 25px; width: 10.5%;
```

`padding: 1px 3px` on a date chip is not density, it is starvation — the text touches its own
background edge. Three chip types on one card use three different radii (4/6/4) and three
different paddings.

### 1.5 Every corner disagrees

Measured across the tree: **13 distinct `border-radius` values** — `2, 3, 4, 5, 6, 7, 8, 10,
12, 16px`, `50%`, plus `!important` variants. On a single minicard: card `4px`, finished
badge `6px`, drag placeholder `17px`, "and N other" chip `6px`.

### 1.6 Summary of the real defects

| Symptom | Actual cause |
|---|---|
| "Square / boxy" | Columns and swimlane bars have **no radius and no gutter**; 1px lines do all separation |
| "Flat" | Black shadows are **invisible on dark**; no rim light; no surface value steps |
| "Cramped" | `1px 3px` chips, `0` column padding, no spacing scale |
| "Cheap" | 13 radii, 3 chip styles per card, saturated full-bleed strips |

---

## 2. The corrected architecture

**Revision to `UI-Redesign-Plan.md` §3:** a theme is no longer color-only. Themes redefine
three groups:

```
Layer 1  PRIMITIVES      raw colors                          (per theme)
Layer 2  SEMANTIC        surface / text / border / state     (per theme)
Layer 2F FORM            shape / elevation / density / separation   (per theme)  ← NEW
Layer 3  COMPONENT       aliases consumed by component CSS   (theme-agnostic)
```

Layer 2F is what makes Nebula and Meridian look like **different applications** rather than
the same boxy app in two palettes — while Classic pins today's exact values and does not move.

### 2.1 Shape tokens — by role, not by pixel

The 13 radii collapse into six *roles*. Themes set the values.

| Token | Applies to | Classic | Meridian | Nebula |
|---|---|---:|---:|---:|
| `--wk-shape-column` | list column, swimlane panel | `0` | `10px` | `16px` |
| `--wk-shape-card` | minicard, board tile | `4px` | `8px` | `14px` |
| `--wk-shape-popup` | pop-over, modal, dropdown | `6px` | `10px` | `16px` |
| `--wk-shape-control` | button, input, select | `4px` | `6px` | `10px` |
| `--wk-shape-chip` | label, date, badge, count | `4px` | `6px` | `999px` (pill) |
| `--wk-shape-avatar` | avatars only | `50%` | `50%` | `50%` |

### 2.2 Separation strategy — themeable

How a surface separates from what is behind it. **This is the fix for "flat".**

```css
--wk-sep-strategy         /* documentation only: shadow | rim | both       */
--wk-sep-card             /* the actual box-shadow value for a card        */
--wk-sep-column           /* … for a list column                           */
--wk-sep-popup            /* … for a popup                                 */
--wk-rim-light            /* inset highlight on the top edge (dark themes) */
```

- **Light themes → `shadow`.** Black shadows work (12.9% delta). Soft and shallow.
- **Dark themes → `rim` + value step.** A 1px inset highlight on the top edge simulates a
  light source above, plus a genuine lightness step between `--wk-surface-2` (column) and
  `--wk-surface-1` (card). Optionally a colored glow for the accent.

```css
/* Nebula: rim light instead of an invisible black shadow */
--wk-rim-light: inset 0 1px 0 rgb(255 255 255 / 0.07);
--wk-sep-card:
  inset 0 1px 0 rgb(255 255 255 / 0.07),      /* rim: top edge catches light */
  0 2px 8px rgb(0 0 0 / 0.45);                /* still helps at larger blur  */
```

Nebula's card `#1d1733` on column `#141024` is a real, visible value step — the card is
lighter than what it sits on, which is how depth reads on dark. The current dark theme has
almost no such step, which is the other half of §1.3.

### 2.3 Density — real numbers, one multiplier

```css
--wk-density: 1;          /* comfortable (default) */
.wk-density-compact { --wk-density: 0.75; }
```

Applied as `calc(var(--wk-space-N) * var(--wk-density))`. Compact restores today's tightness
for users who want maximum cards on screen — **density becomes a choice instead of the only
option.**

---

## 3. Per-component specification

Every "before" below is the current declaration, quoted from the tree.

### 3.1 List column — the biggest single win

**Before:** `background:#dedede; border-inline-start:1px solid #ccc; padding:0;` no radius,
no gutter, no elevation.

**After:** columns become *panels* — rounded, inset, separated by a real gutter, with the
1px divider deleted.

```css
.list {
  background: var(--wk-list-bg);
  border-radius: var(--wk-shape-column);
  margin-inline-end: calc(var(--wk-space-2) * var(--wk-density));   /* the gutter */
  padding: calc(var(--wk-space-2) * var(--wk-density));
  box-shadow: var(--wk-sep-column);
  border-inline-start: none;                                        /* delete the 1px line */
}
.list-header {
  padding: calc(var(--wk-space-3) * var(--wk-density))
           calc(var(--wk-space-3) * var(--wk-density))
           calc(var(--wk-space-2) * var(--wk-density));
  background: transparent;              /* no second grey bar; the panel is the surface */
  border-radius: var(--wk-shape-column) var(--wk-shape-column) 0 0;
}
```

Note `padding: 28px 21px 6px !important` becomes a balanced rhythm — the current 28px top
against 6px bottom is what makes the header look detached from its own list.

### 3.2 Swimlane header — strip → inset panel label

**Before:** `background-color:#ccc; min-height:33px;` full-bleed, square, centered text.

**After:** an inset rounded bar with the swimlane color as a **leading accent**, not a
full-width flood:

```css
.swimlane .swimlane-header-wrap {
  margin: calc(var(--wk-space-2) * var(--wk-density));
  border-radius: var(--wk-shape-column);
  background: var(--wk-surface-2);
  border-inline-start: 4px solid var(--swimlane-accent, var(--wk-accent));
  box-shadow: var(--wk-sep-column);
  min-height: 36px;
}
.swimlane .swimlane-header-wrap .swimlane-header {
  text-align: start;                    /* stop centering; titles read left (or RTL start) */
  padding-inline: var(--wk-space-3);
  font-size: var(--wk-text-sm);
  font-weight: var(--wk-weight-medium);
  letter-spacing: 0.01em;
  text-transform: none;
}
```

The user's chosen swimlane color is **preserved** — it moves from a 100%-width flood to a 4px
edge plus an optional 8%-alpha tint. Same information, a tenth of the visual shouting.

### 3.3 Minicard

**Before:** `padding:8px 14px 3px; border-radius:4px; box-shadow:0 2px 3px rgba(0,0,0,0.15);`

**After:**

```css
.minicard {
  padding: calc(var(--wk-space-3) * var(--wk-density));
  border-radius: var(--wk-shape-card);
  background: var(--wk-card-bg);
  box-shadow: var(--wk-sep-card);
  transition: box-shadow var(--wk-motion-fast) var(--wk-ease-standard),
              transform  var(--wk-motion-fast) var(--wk-ease-standard);
}
.minicard:hover     { box-shadow: var(--wk-sep-card-hover); }
.minicard:active,
.minicard.ui-sortable-helper {
  box-shadow: var(--wk-sep-card-drag);
  transform: rotate(2deg) scale(1.02);      /* was rotate(4deg), no depth change */
}
```

The `3px` bottom padding against `8px` top is corrected to a symmetric box — that asymmetry
is why card content currently looks like it is sliding out of the bottom.

### 3.4 Chips — one shape, one rhythm

All three chip types (label, date, badge) converge:

```css
.card-date,
.minicard .badges .badge,
.card-label {
  border-radius: var(--wk-shape-chip);
  padding: var(--wk-space-1) var(--wk-space-2);   /* was 1px 3px / 0 6px / 4px 10px */
  min-height: 20px;
  font-size: var(--wk-text-xs);
  line-height: var(--wk-leading-tight);
  display: inline-flex;
  align-items: center;
  gap: var(--wk-space-1);
}
```

Date states keep their semantics but move from raw saturated fills (`#ff4444`, `#ffaa00`,
`#5ba639`) to the `-subtle` / `-text` token pairs, so an overdue chip is a **tinted** chip
with AA-passing text rather than a fire-engine block.

The minicard label squares (`height:25px; width:10.5%`) become fixed-size rounded chips laid
out in a `flex-wrap` row with `gap: var(--wk-space-1)` — no percentage widths, so they stop
resizing with the card.

### 3.5 Board header and top bar

**Before:** two flat full-width bars; every control the same weight; no grouping.

**After:** one bar, grouped into three zones (identity · view controls · actions) separated by
`var(--wk-space-4)`. Controls become ghost buttons with `--wk-shape-control`; **only the
primary action carries the accent fill.** Height set from the type scale rather than ad-hoc
padding.

### 3.6 Popups

**Before:** `border-radius:6px; box-shadow:0 2px 7px rgba(0,0,0,0.3); width:min(380px,55vw);`

**After:** `--wk-shape-popup`, `--wk-sep-popup` (a genuine `--wk-elev-3`), consistent
`var(--wk-space-4)` padding, and a header/body/footer rhythm. The board-background swatch grid
in the screenshot becomes rounded tiles at `--wk-shape-card` with a check overlay and a real
focus ring, on a `--wk-space-2` grid gap.

### 3.7 Inputs

**Before:** bare white rectangles, mixed focus treatments, several `outline: none`.

**After:** `--wk-shape-control`, `--wk-input-bg`, `--wk-input-border`, one height from the type
scale, and the standard double focus ring from `Design-Tokens.md` §Focus.

---

## 4. What this means for the two themes

The same component CSS, three genuinely different results:

| | Classic | Meridian | Nebula |
|---|---|---|---|
| Column | square, 1px line | `10px`, gutter, soft shadow | `16px`, wide gutter, floating |
| Card | `4px`, faint shadow | `8px`, soft shadow, white on tint | `14px`, rim light + value step |
| Chips | `4px`, `1px 3px` | `6px`, tinted | pill, tinted, glowing on hover |
| Separation | 1px borders | shadow | rim light + value step |
| Swimlane | full-bleed strip | inset bar, 4px accent | inset bar, 4px accent, translucent |
| Character | unchanged | crisp, calm, structured | soft, luminous, floating |

**Classic pins every Layer 2F token to its current value** (`--wk-shape-column: 0`,
`--wk-shape-card: 4px`, the existing shadow), so it renders byte-identically and the visual
regression baselines hold. The original theme is preserved *because* form is tokenized, not
despite it.

---

## 5. Revised phase order

`UI-Redesign-Plan.md` §7 is amended: form is no longer last.

| Phase | Was | Now |
|---|---|---|
| 0 | Tooling | unchanged |
| 1 | Tokens + Classic (color) | Tokens + Classic (**color and form**) |
| 2 | Migrate color literals | Migrate color literals **and geometry to shape/space tokens** |
| 3 | Theme runtime | unchanged |
| 4 | Nebula (color) | Nebula (**color + form**) |
| 5 | Meridian (color) | Meridian (**color + form**) |
| 6 | Form redesign ← *was here* | **Structural work**: §3.1 gutters, §3.2 swimlane, §3.5 header zones |
| 7 | Cleanup | unchanged |

Phases 1–5 stay visually neutral for Classic (its Layer 2F values equal today's). **Phase 6 is
still the only phase that intentionally moves Classic**, because deleting the 1px column
divider and adding gutters changes the default layout — that needs maintainer sign-off and
fresh baselines. Nebula and Meridian, however, get their full character in Phases 4–5, since
their Layer 2F values differ from Classic's from the moment they exist.

---

## 6. Accessibility and behaviour constraints

- **Density must not break touch targets.** Even at `--wk-density: 0.75`, interactive targets
  stay ≥ 32px; the existing invisible hit-area overlay on the multi-selection checkbox
  (`minicard.css`) is the pattern to follow.
- **Rim light is decoration, never the only cue.** Selection and focus keep a border/ring that
  meets 3:1, so depth is never load-bearing for meaning.
- **Radius must not clip content.** Rounded columns need `overflow: hidden` only on the header
  cap; the scroll container must keep `overflow-y: auto` or card drag breaks.
- **Gutters cost horizontal space.** Adding `var(--wk-space-2)` per column reduces visible
  columns on narrow screens; compact density and the existing `--list-width` variable must be
  checked together at 1280px and at mobile breakpoints.
- **RTL.** Every new declaration uses logical properties (`margin-inline-end`,
  `border-inline-start`) — the tree is RTL-correct today.
- **`prefers-reduced-motion`** zeroes the new hover/drag transitions via the motion tokens.
- **Print** disables shadows, rim light and gutters.

## 7. Testing additions

- `tests/themeTokens.test.cjs` — extend to require every **Layer 2F** token in every theme.
- `tests/themeForm.test.cjs` *(new)* — assert Classic's Layer 2F values equal the current
  literals (`--wk-shape-card: 4px`, `--wk-shape-column: 0`, the existing shadow), so Phases
  1–5 cannot silently move the original theme.
- Visual regression gains a **density** axis: 8 screens × 3 themes × 2 densities.
- A dark-separation check: assert dark themes define a non-empty `--wk-rim-light` and a
  measurable lightness step between `--wk-surface-2` and `--wk-surface-1` (the §1.3 bug,
  encoded as a test).

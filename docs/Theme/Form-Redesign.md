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

> **Read this first: the "before" quotes below are the base rules, not always what renders.**
> Every board carries a `board-color-<name>` class (`models/boards.js:426` defaults it to
> `belize`), and `boardColors.css` holds **784** two-class `.board-color-<name> .foo` selectors
> that outrank every single-class rule quoted in this document — plus 126 `border-radius`, 101
> `padding`, 47 `box-shadow` and 62 `font-size` declarations of its own. So `.minicard`'s
> `border-radius: 4px` below is real CSS that **loses**; users see `7px` from
> `boardColors.css:228`.
>
> This does not change the diagnosis — the UI is square, boxy and flat either way, and in
> several cases the legacy file makes it *worse* (three different card radii across the 19
> themes: `7px`, `12px`, `2px`). It changes two things:
>
> - **Classic's Layer 2F values are seeded from the rendered cascade**, not from these quotes.
>   `--wk-shape-card` is `7px`.
> - **`boardColors.css` is fixed first** (`UI-Redesign-Plan.md` Phase 2a). Applying the geometry
>   below to `.minicard` while the legacy file still overrides it accomplishes nothing.
>
> A second caveat on §1.1–§1.2: those quote the **light** default (`#dedede`, `#ccc`). Under a
> dark board color the same rules are overridden with dark values — so if you are looking at a
> dark screenshot, the literals below are not the ones on your screen, though the geometry
> (no radius, no gutter, 1px divider, full-bleed strip) is identical because
> `boardColors.css` does not change it.

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

Measured across the whole tree: **32 distinct `border-radius` values** — `0, 1, 2, 3, 4, 5, 6,
7, 8, 10, 12, 14, 15, 16, 17, 18, 20, 100px`, `50%`, `100%`, `inherit`, `unset`, several
multi-corner forms (`5px 5px 0 0`, `0 0 14px 14px`, `12px 12px 0 0`), plus `!important`
variants. Alongside them: **54** distinct `font-size` values and **70** distinct `box-shadow`
values.

(An earlier draft of this section said 13 / ~20 / 12+. Those came from a sample that excluded
`boardColors.css` and the `!important` variants. The real figures are 2–5× larger, which is why
`UI-Redesign-Plan.md` §7 splits the scale collapse into its own phase — 32 → 4 radii is a
redesign that moves pixels, not a substitution.)

On a single minicard: card `4px` in `minicard.css` but `7px` as rendered, finished badge `6px`,
drag placeholder `17px`, "and N other" chip `6px`.

### 1.6 There is no typeface

`client/components/main/fonts.css` — imported at `client/styles.js:33` — is **entirely
commented out**: every `@font-face` for Roboto and Poppins sits inside one `/* … */`. No rule
in `client/components` sets `font-family` on `body` or `html`. The app renders in the browser
default face, and has for a long time.

`--wekan-ui-font` (#4759) exists but `uiFont.css` applies it only under the `has-ui-font`
class — i.e. only for users who explicitly set a font preference. It is an override on top of
nothing.

This is a large share of the "cheap" impression and it is not a theming problem: it is a
default the app never had. `UI-Redesign-Plan.md` Phase 1a fixes it in ~15 lines, ahead of
everything else in this document.

### 1.7 Summary of the real defects

| Symptom | Actual cause |
|---|---|
| "Square / boxy" | Columns and swimlane bars have **no radius and no gutter**; 1px lines do all separation |
| "Flat" | Black shadows are **invisible on dark**; no rim light; no surface value steps |
| "Cramped" | `1px 3px` chips, `0` column padding, no spacing scale |
| "Cheap" | **No typeface at all**; 32 radii, 54 font sizes, 70 shadows; 3 chip styles per card; saturated full-bleed strips |
| "Nothing I change takes effect" | `boardColors.css` outranks the component rules on every board (see the note at the top of §1) |

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

The 32 radii collapse into six *roles*. Themes set the values.

**Classic's column is the value it renders at**, which for the card is `boardColors.css`'s
`7px`, not `minicard.css`'s `4px` (see the note at the top of §1):

| Token | Applies to | Classic *(as rendered)* | Meridian | Nebula |
|---|---|---:|---:|---:|
| `--wk-shape-column` | list column, swimlane panel | `0` | `10px` | `16px` |
| `--wk-shape-card` | minicard, board tile | **`7px`** | `8px` | `14px` |
| `--wk-shape-popup` | pop-over, modal, dropdown | `6px` | `10px` | `16px` |
| `--wk-shape-control` | button, input, select | `4px` | `6px` | `10px` |
| `--wk-shape-chip` | label, date, badge, count | `4px` | `6px` | `999px` (pill) |
| `--wk-shape-avatar` | avatars only | `50%` | `50%` | `50%` |

Two of the 19 legacy themes disagree with `belize` on the card radius (`12px` and `2px`). Those
two keep their own `--wk-shape-card` override on `.board-color-<name>` after the Phase 2a
regeneration, so their appearance is preserved exactly.

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

Every "before" below is the current declaration, quoted from the tree — subject to the cascade
note at the top of §1.

**Markup requirements.** Three of these items need template edits, which the first draft of
`UI-Redesign-Plan.md` had listed as a non-goal while specifying them here. That contradiction
is resolved: presentational containers and classes are allowed, in Phase 6, with no change to
Blaze helpers, event handlers, data contexts, `js-*` hook classes, or interactive DOM order.

| Item | Template | Change |
|---|---|---|
| §3.4 chips | `client/components/cards/minicard.jade` | A wrapper around the label squares so they can be a `flex-wrap` row |
| §3.5 header zones | `client/components/boards/boardHeader.jade` | Regroup the button flex items into three zones — **see the warning in §3.5** |
| §3.6 popups | `client/components/main/popup.tpl.jade` | Explicit header / body / footer containers |

§3.1, §3.2, §3.3 and §3.7 are CSS-only.

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

> **The existing grouping is load-bearing — do not treat it as arbitrary.** `boardHeader.jade`
> already has structure, and both parts of it were bug fixes, documented in comments in the
> template:
>
> - `.board-header-btns-group` wraps **all** the button groups in one flex item, specifically
>   so that a long board title pushes every button to the second row *together*. As separate
>   flex items, the left group stayed beside the title and only the right group wrapped,
>   splitting the controls across both rows.
> - `.board-header-sidebar-toggle` is a separate flex item placed right after the title in
>   source order, with `order` moving it last on wide screens. That is what puts the hamburger
>   top-right in mobile mode instead of wrapping down with everything else.
>
> Three-zone grouping must preserve both. `tests/mobileModeConsistency.test.cjs` and
> `tests/mobileBoardFit.test.cjs` are the guards. **If the zoning cannot be reached without
> breaking the wrap behaviour, drop the zoning** — a header that regresses to two-row control
> splitting is worse than a header with two zones instead of three.
>
> Note also that "two flat full-width bars" is only half the story: the upper bar is
> `#header-quick-access` (the All Boards / starred-boards strip from `header.jade:11`), a
> different template with a different purpose. This section covers the board header only;
> whether the quick-access strip should merge, collapse or stay is not specified here and is
> out of scope for Phase 6.

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
| Card | `7px`, faint shadow | `8px`, soft shadow, white on tint | `14px`, rim light + value step |
| Chips | `4px`, `1px 3px` | `6px`, tinted | pill, tinted, glowing on hover |
| Separation | 1px borders | shadow | rim light + value step |
| Swimlane | full-bleed strip | inset bar, 4px accent | inset bar, 4px accent, translucent |
| Character | unchanged | crisp, calm, structured | soft, luminous, floating |

**Classic pins every Layer 2F token to the value it currently renders at**
(`--wk-shape-column: 0`, `--wk-shape-card: 7px`, the `2px 2px 4px 0 rgb(0 0 0 / 0.15)` shadow —
all three from `boardColors.css`, not from the component files), so it renders byte-identically
and the visual regression baselines hold. The original theme is preserved *because* form is
tokenized, not despite it.

---

## 5. Revised phase order

`UI-Redesign-Plan.md` §7 is the authority; this table summarises where the work in *this*
document lands. Form is no longer last, and there are now three phases that intentionally move
Classic rather than one.

| Phase | Content | Moves Classic? |
|---|---|---|
| 0 | Tooling | no |
| **1a** | **§1.6 — the typeface** | **yes**, deliberately |
| 1b | Tokens + Classic extraction (color **and** form, seeded from the *rendered* cascade) | no |
| **2a** | **Regenerate `boardColors.css`** — the prerequisite; without it nothing below is visible | no (pure refactor, 19 themes screenshot-identical) |
| 2b | Migrate color literals | no |
| **2c** | Collapse geometry to the shape/space/type/elevation scales (32→4, 54→7, 70→5) | **yes**, expected |
| 3 | Theme runtime | no |
| 4 | Nebula (**color + form**) | n/a |
| 5 | Meridian (**color + form**) | n/a |
| **6** | **Structural + markup**: §3.1 gutters, §3.2 swimlane, §3.4 label row, §3.5 header zones, §3.6 popup containers | **yes**, needs sign-off |
| 7 | Cleanup | no |

Nebula and Meridian get their full character in Phases 4–5, since their Layer 2F values differ
from Classic's from the moment they exist. Classic changes in 1a (typeface), 2c (scale
snapping) and 6 (layout) — each isolated so each can be signed off, or reverted, on its own.

Within Phase 6, **ship §3.1 (the column gutter) first**. It is the item most likely to collide
with the mobile-layout work that landed immediately before this plan, and shipping it alone
keeps the revert cheap.

---

## 6. Accessibility and behaviour constraints

- **Density must not break touch targets.** Even at `--wk-density: 0.75`, interactive targets
  stay ≥ 32px; the existing invisible hit-area overlay on the multi-selection checkbox
  (`minicard.css`) is the pattern to follow.
- **Rim light is decoration, never the only cue.** Selection and focus keep a border/ring that
  meets 3:1, so depth is never load-bearing for meaning.
- **Radius must not clip content.** Rounded columns need `overflow: hidden` only on the header
  cap; the scroll container must keep `overflow-y: auto` or card drag breaks.
- **Gutters cost horizontal space, and land on freshly-stabilised code.** Adding
  `var(--wk-space-2)` per column reduces visible columns on narrow screens. `--list-width` is
  not a stylesheet default — it is an **inline style** written per list from `list.jade:3`
  (`--list-width:{{listWidth}}px`), backed by the `wekan-fixed-list-width` localStorage
  preference in `listHeader.js`, and consumed with `!important` at `list.css:95-97`. A gutter
  is therefore *added to* a width the user may have pinned, not absorbed by it.

  The ten commits immediately preceding this plan are all mobile-layout fixes — `clamp()`
  removal, board-width fit, centred titles, Add List relocation. Six existing tests gate this
  one change: `listWidthDefaults`, `mobileBoardFit`, `mobileModeFullWidth`,
  `mobileModeConsistency`, `mobileAddList`, `sidebarWidth`. Verify at 1280px, at the mobile
  breakpoint, at both densities, **and with a fixed per-list width set**.

- **No viewport units, and no `clamp()`.** `tests/noViewportSpacing.test.cjs` scans every
  stylesheet and rejects `vh`/`vw` for size or spacing, `clamp()` in any form, and viewport
  `min-*`. Allowed: full-viewport boxes (`100vh`/`100vw`), `max-*` caps, and shrink-only
  `min(400px, 52vw)`. Everything specified in this document is fixed-px or
  `calc(token * density)`, so it complies — but any "make it responsive" instinct during
  implementation will trip this test, and the test is right.

- **The always-visible list scrollbar stays.** `tests/scrollbarCss.test.cjs` guards #5439:
  list bodies must keep visible scrollbars on every engine, so overlay/auto-hiding scrollbars
  are not an option for a cleaner look. Themes may restyle them (`scrollbar-color`,
  `::-webkit-scrollbar`) — Nebula already does — but must not hide them.
- **RTL.** Every new declaration uses logical properties (`margin-inline-end`,
  `border-inline-start`) — the tree is RTL-correct today.
- **`prefers-reduced-motion`** zeroes the new hover/drag transitions via the motion tokens.
- **Print** disables shadows, rim light and gutters.

## 7. Testing additions

- `tests/themeTokens.test.cjs` — extend to require every **Layer 2F** token in every theme.
- `tests/themeForm.test.cjs` *(new)* — assert Classic's Layer 2F values equal the values it
  **renders** at (`--wk-shape-card: 7px` from `boardColors.css`, `--wk-shape-column: 0`, the
  `2px 2px 4px 0 rgb(0 0 0 / 0.15)` card shadow), so Phases 1b–5 cannot silently move the
  original theme. Asserting against `minicard.css`'s `4px` would pin a value nobody sees.
- Visual regression gains a **density** axis: 8 screens × 3 themes × 2 densities — plus, as a
  Phase 2a gate, all **19 legacy board colors**, since that phase rewrites the file that
  renders them.
- A dark-separation check: assert dark themes define a non-empty `--wk-rim-light` and a
  measurable lightness step between `--wk-surface-2` and `--wk-surface-1` (the §1.3 bug,
  encoded as a test).
- Full list of the **existing** tests that gate this work, and what each one forbids:
  `UI-Redesign-Plan.md` §8.2.

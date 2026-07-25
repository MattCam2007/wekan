# Theme: Nebula (dark)

Status: **Plan — not yet implemented** · Category: `token` · Class: `:root.wk-theme-nebula`
Related: `docs/Theme/UI-Redesign-Plan.md`, `docs/Theme/Design-Tokens.md`.

Nebula is derived from a deep-space nebula photograph: a near-black void, a hot
magenta/pink core, violet mid-tones bleeding to cobalt blue on one edge and crimson on the
other, over a fine field of stars.

## 1. Source reading

What the source image actually provides, and how each part maps into the UI:

| Region in the image | Color | Role in the theme |
|---|---|---|
| Outer void | near-black, blue-violet tinted | App background, board canvas |
| Deep field between clouds | very dark violet | List columns, cards |
| Hot core | bright magenta/pink | **Primary accent** — header, primary buttons, focus |
| Core highlight | near-white pink | Highlight, selected state, star glow |
| Mid cloud | violet | Secondary accent — links, informational chips |
| Right edge | cobalt blue | Info state, drag/target affordances |
| Left edge | crimson | Danger state |
| Stars | white / cold white | Primary text, decorative star field |

**The image supplies no green.** A kanban needs a success color (checklists, due-soon,
completed). Nebula therefore adds one deliberate palette extension: an aquamarine sitting
between the image's cobalt and its highlights, so it reads as part of the same nebula rather
than as a foreign UI green. This is documented as an extension, not an extraction.

**No raster asset is shipped.** The backdrop is reproduced with CSS radial gradients
(~2 KB) instead of the 2560×1600 JPEG. Users who want the actual photograph can already set
it through the existing board background feature (`docs/Theme/background`).

## 2. Layer 1 — Primitives

```css
:root.wk-theme-nebula {
  /* Void — neutral ramp, blue-violet tinted rather than pure grey */
  --wk-palette-neutral-1000: #05040c;   /* deepest void          */
  --wk-palette-neutral-900:  #0b0918;   /* board canvas          */
  --wk-palette-neutral-800:  #141024;   /* list column           */
  --wk-palette-neutral-700:  #1d1733;   /* card                  */
  --wk-palette-neutral-600:  #2a2145;   /* raised / hover        */
  --wk-palette-neutral-500:  #3a2f5c;   /* borders strong        */
  --wk-palette-neutral-400:  #4a3d72;
  --wk-palette-neutral-300:  #6b5f92;
  --wk-palette-neutral-200:  #968db2;   /* muted text            */
  --wk-palette-neutral-100:  #c4bcdc;   /* secondary text        */
  --wk-palette-neutral-0:    #f2edff;   /* starlight, primary text */

  /* Magenta core — primary accent ramp */
  --wk-palette-accent-100: #ffd6ec;
  --wk-palette-accent-300: #ff92c9;
  --wk-palette-accent-500: #ff5fb0;     /* the core               */
  --wk-palette-accent-600: #ef3d97;
  --wk-palette-accent-700: #d6338d;
  --wk-palette-accent-900: #7d1a52;

  /* Theme-private extras (image-derived) */
  --wk-palette-violet-500: #a78bfa;     /* mid cloud              */
  --wk-palette-violet-700: #7c5cd6;
  --wk-palette-cobalt-500: #5b93ff;     /* right edge             */
  --wk-palette-cobalt-700: #3f6fd8;
  --wk-palette-crimson-500: #ff6b7f;    /* left edge              */
  --wk-palette-crimson-700: #c33049;
  --wk-palette-star-gold:  #ffd48a;     /* warm stars             */
  --wk-palette-aqua-500:   #2fd6a8;     /* EXTENSION — success    */
  --wk-palette-aqua-700:   #1a9c78;
}
```

## 3. Layer 2 — Semantic mapping

```css
:root.wk-theme-nebula {
  /* Surfaces */
  --wk-surface-app:       var(--wk-palette-neutral-1000);
  --wk-surface-canvas:    var(--wk-palette-neutral-900);
  --wk-surface-1:         var(--wk-palette-neutral-700);
  --wk-surface-2:         var(--wk-palette-neutral-800);
  --wk-surface-3:         var(--wk-palette-neutral-600);
  --wk-surface-sunken:    var(--wk-palette-neutral-1000);
  --wk-surface-overlay:   rgb(5 4 12 / 0.72);
  --wk-surface-hover:     rgb(255 255 255 / 0.06);
  --wk-surface-active:    rgb(255 255 255 / 0.10);
  --wk-surface-selected:  rgb(255 95 176 / 0.14);

  /* Text */
  --wk-text-primary:   var(--wk-palette-neutral-0);
  --wk-text-secondary: var(--wk-palette-neutral-100);
  --wk-text-muted:     var(--wk-palette-neutral-200);
  --wk-text-disabled:  var(--wk-palette-neutral-300);
  --wk-text-on-accent: #1a0a14;          /* dark ink on hot magenta */
  --wk-text-link:      var(--wk-palette-violet-500);
  --wk-text-link-hover:var(--wk-palette-accent-300);
  --wk-text-inverse:   var(--wk-palette-neutral-1000);

  /* Borders */
  --wk-border-subtle:  rgb(255 255 255 / 0.08);
  --wk-border-default: var(--wk-palette-neutral-500);
  --wk-border-strong:  var(--wk-palette-neutral-400);
  --wk-border-focus:   var(--wk-palette-accent-500);

  /* Accent */
  --wk-accent:         var(--wk-palette-accent-500);
  --wk-accent-hover:   var(--wk-palette-accent-600);
  --wk-accent-active:  var(--wk-palette-accent-700);
  --wk-accent-subtle:  rgb(255 95 176 / 0.16);
  --wk-accent-focus:   var(--wk-palette-accent-300);

  /* States */
  --wk-success: var(--wk-palette-aqua-500);
  --wk-success-subtle: rgb(47 214 168 / 0.16);   --wk-success-text: var(--wk-palette-aqua-500);
  --wk-warning: var(--wk-palette-star-gold);
  --wk-warning-subtle: rgb(255 212 138 / 0.16);  --wk-warning-text: var(--wk-palette-star-gold);
  --wk-danger:  var(--wk-palette-crimson-500);
  --wk-danger-subtle:  rgb(255 107 127 / 0.16);  --wk-danger-text:  var(--wk-palette-crimson-500);
  --wk-info:    var(--wk-palette-cobalt-500);
  --wk-info-subtle:    rgb(91 147 255 / 0.16);   --wk-info-text:    var(--wk-palette-cobalt-500);

  /* Elevation — dark themes need deeper, cooler shadows */
  --wk-shadow-ambient: rgb(0 0 0 / 0.55);
  --wk-shadow-direct:  rgb(0 0 0 / 0.45);

}
```

## 3F. Layer 2F — Form

**This is what stops Nebula being "the boxy layout in purple".** Character: *soft, luminous,
floating* — panels drifting in the void rather than tiles butted together. Rationale and
per-component geometry in `docs/Theme/Form-Redesign.md`.

```css
:root.wk-theme-nebula {
  /* Shape — the most rounded of the three themes */
  --wk-shape-column:  16px;
  --wk-shape-card:    14px;
  --wk-shape-popup:   16px;
  --wk-shape-control: 10px;
  --wk-shape-chip:    999px;        /* full pills, not sharp rectangles */
  --wk-shape-avatar:  50%;

  /* Separation: RIM LIGHT, not shadow. A black shadow over #141024 shifts the
     surface by ~1.2% of range — invisible. Depth here comes from a top-edge
     highlight plus a genuine value step (card #1d1733 sits ABOVE list #141024). */
  --wk-sep-strategy: rim;
  --wk-rim-light: inset 0 1px 0 rgb(255 255 255 / 0.07);

  --wk-sep-card:
    inset 0 1px 0 rgb(255 255 255 / 0.07),
    0 2px 10px rgb(0 0 0 / 0.45);
  --wk-sep-card-hover:
    inset 0 1px 0 rgb(255 255 255 / 0.12),
    0 4px 18px rgb(0 0 0 / 0.55),
    0 0 0 1px rgb(255 95 176 / 0.22);          /* faint magenta rim on hover */
  --wk-sep-card-drag:
    inset 0 1px 0 rgb(255 255 255 / 0.16),
    0 16px 40px rgb(0 0 0 / 0.65),
    0 0 24px rgb(255 95 176 / 0.28);           /* the card glows while dragged */
  --wk-sep-column:
    inset 0 1px 0 rgb(255 255 255 / 0.05),
    0 8px 32px rgb(0 0 0 / 0.40);
  --wk-sep-popup:
    inset 0 1px 0 rgb(255 255 255 / 0.10),
    0 20px 48px rgb(0 0 0 / 0.70);

  --wk-density: 1;                  /* generous — the void needs room to read */
}
```

### Structural signatures

- **List columns float.** `16px` radius, a `--wk-space-3` gutter (wider than Meridian's), and
  the `1px solid #ccc` divider deleted. The nebula backdrop shows through the gutters, which
  is what makes the columns read as panels *in* the scene rather than windows onto it.
- **Optional translucency.** Columns may use `background: rgb(20 16 36 / 0.72)` with
  `backdrop-filter: blur(12px)` so the nebula glows faintly through. **Perf-gated**: this is
  a compositing cost on a scrolling board, so it ships behind the same toggle as the backdrop
  and falls back to the opaque `--wk-surface-2` when disabled or unsupported.
- **Swimlane headers** become inset rounded bars with a 4px leading accent — never full-bleed
  strips.
- **Chips are pills.** Labels, dates and badges all take `--wk-shape-chip: 999px` with
  `--wk-space-1 --wk-space-2` padding, replacing today's `1px 3px` sharp rectangles.
- **Glow is hover/drag only.** No resting glow anywhere: a board of 40 glowing cards is noise,
  and permanent glow would defeat the contrast work in §4.

## 4. Verified contrast

Computed with the WCAG 2.1 relative-luminance formula against Nebula's three text-bearing
surfaces. **All pass AA (4.5:1); all but `--wk-text-muted` also pass AAA (7:1) on canvas.**

| Foreground | Hex | on canvas `#0b0918` | on list `#141024` | on card `#1d1733` |
|---|---|---:|---:|---:|
| `--wk-text-primary` | `#f2edff` | 17.19 | 16.23 | 15.00 |
| `--wk-text-secondary` | `#c4bcdc` | 10.85 | 10.25 | 9.47 |
| `--wk-text-muted` | `#968db2` | 6.33 | 5.98 | 5.52 |
| `--wk-accent` | `#ff5fb0` | 7.05 | 6.66 | 6.15 |
| link / violet | `#a78bfa` | 7.24 | 6.83 | 6.31 |
| `--wk-info` | `#5b93ff` | 6.62 | 6.25 | 5.78 |
| `--wk-danger` | `#ff6b7f` | 7.18 | 6.78 | 6.27 |
| `--wk-success` | `#2fd6a8` | 10.60 | 10.01 | 9.25 |
| `--wk-warning` | `#ffd48a` | 14.10 | 13.31 | 12.30 |

`--wk-text-on-accent` `#1a0a14` on `--wk-accent` `#ff5fb0` = **6.85** ✓

These pairs go into `tests/themeContrast.test.cjs` with their minimums so the values cannot
silently regress.

## 5. Decorative backdrop (`theme-nebula-decor.css`)

The nebula itself, as CSS. **Optional, off-switchable, and perf-budgeted.**

```css
.wk-theme-nebula .board-canvas::before {
  content: "";
  position: fixed;
  inset: 0;
  z-index: -1;                      /* behind all content, never intercepts pointer */
  pointer-events: none;
  background:
    radial-gradient(60% 45% at 38% 42%, rgb(255 95 176 / 0.22), transparent 70%),
    radial-gradient(45% 40% at 62% 50%, rgb(124 92 214 / 0.20), transparent 72%),
    radial-gradient(38% 34% at 86% 46%, rgb(63 111 216 / 0.18), transparent 70%),
    radial-gradient(30% 30% at 10% 52%, rgb(195 48 73 / 0.16), transparent 72%),
    var(--wk-surface-app);
}
```

Star field: a second fixed layer of 3–4 tiled `radial-gradient` dots at low opacity.

### Rules

- **`position: fixed` + `z-index: -1`, never `background-attachment: fixed`** — the latter
  forces a full repaint on every scroll frame and would visibly hurt board dragging, which is
  WeKan's most performance-sensitive interaction.
- **No animation by default.** Any twinkle/drift must sit behind
  `@media (prefers-reduced-motion: no-preference)` *and* the user toggle.
- **Perf budget:** no measurable regression in list-scroll or card-drag frame time versus
  Classic on the reference hardware. If the budget fails, ship Nebula flat (the semantic
  tokens alone are the theme; the decor is garnish).
- **Never behind text without a surface.** Cards and list columns are opaque, so the backdrop
  is only ever visible in canvas gutters — the contrast table above stays valid.
- **Print:** suppressed under `@media print`.
- **Toggle:** Member Settings → "Nebula backdrop", stored alongside the existing #4759 UI
  preferences; adds/removes `wk-theme-nebula-flat` on `<html>`.

## 6. Signature details

Small touches that make Nebula feel designed rather than merely inverted:

- **Board header** — a magenta→violet gradient (`--wk-accent` → `--wk-palette-violet-700`)
  rather than a flat fill, echoing the core-to-cloud transition.
- **Focus ring** — `--wk-accent-focus` `#ff92c9`, which reads as a glow against the void
  while staying ≥3:1 on every surface.
- **Minicard left border** on selection uses `--wk-accent`, matching the existing
  `border-inline-start: 3px solid` convention in `boardColors.css`.
- **Drag** uses `rotate(2deg) scale(1.02)` with `--wk-sep-card-drag`, so a dragged card lifts
  *and* glows — replacing today's `rotate(4deg)` with no depth change at all.
- **Labels** — the 19 existing label colors are user data and are **not** restyled. Nebula
  only adjusts their *text* color via the existing `models/lib/contrastColor` helper so
  labels stay readable on a dark card.
- **Scrollbars** — `scrollbar-color: var(--wk-palette-neutral-500) var(--wk-surface-2)`.

## 7. Implementation checklist (Phase 4)

**Prerequisite: Phase 2a.** Until `boardColors.css` is regenerated from tokens, its 784
two-class selectors outrank every component rule Nebula's tokens feed, so the theme would
render as a partial repaint over Classic geometry. See `UI-Redesign-Plan.md` §1.3.

- [ ] `client/components/theme/theme-nebula.css` — layers 1, 2 **and 2F** exactly as above,
      in a single `:root.wk-theme-nebula` block (not a bare class — see `Design-Tokens.md`
      authoring rule 3)
- [ ] Every value a **literal**; no `color-mix()` in any `--wk-*` definition
      (`Design-Tokens.md` authoring rule 4)
- [ ] Rim-light separation verified on a real board (the §1.3 flatness bug in
      `Form-Redesign.md` must not reappear) — `tests/themeForm.test.cjs`
- [ ] Translucent columns behind the perf toggle, with an opaque fallback
- [ ] `client/components/theme/theme-nebula-decor.css` — backdrop + star field
- [ ] `'nebula'` in `config/const.js` `ALLOWED_BOARD_COLORS`
- [ ] `'nebula'` in the `token` category in `models/lib/themeCategories.js`
- [ ] i18n keys `theme-nebula`, `theme-nebula-backdrop` and `theme-category-token` in
      `en.i18n.json` **only**
- [ ] Contrast pairs added to `tests/themeContrast.test.cjs`
- [ ] Token completeness passes `tests/themeTokens.test.cjs`
- [ ] Playwright baselines for the 8 reference screens
- [ ] Backdrop toggle wired and defaulted per maintainer decision (`UI-Redesign-Plan.md` §10.2)
- [ ] Verified: no legacy `board-color-*` class co-active (`UI-Redesign-Plan.md` §6.2)

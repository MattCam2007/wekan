# Theme: Meridian (light)

Status: **Plan — not yet implemented** · Category: `token` · Class: `:root.wk-theme-meridian`
Related: `docs/Theme/UI-Redesign-Plan.md`, `docs/Theme/Design-Tokens.md`.

Meridian is the third theme and the **reference implementation of the redesigned form**. Where
Nebula shows that the token layer can carry a dramatic, fully dark repaint, Meridian shows
what the redesign is actually *for*: a calm, high-legibility light theme for people who look
at a board for eight hours.

## 1. Design intent

The original theme is a saturated flat-blue chrome over near-white surfaces, with cramped
padding, 54 font sizes, 32 corner radii, 70 shadow variants and no typeface of its own.
Meridian keeps the same information density target but fixes the rhythm:

- **Cool neutrals, one confident accent.** Color carries *meaning* only — status, selection,
  focus. Chrome is neutral, so user data (labels, card colors, avatars) is what stands out.
- **Layered by value, not by line.** Surfaces separate through small lightness steps and one
  subtle shadow instead of the current mix of borders, shadows and background swaps.
- **Legible by default.** Every text token clears AA on every surface it can land on, with
  primary text well past AAA.
- **Quiet chrome, loud content.** The board header drops from a saturated fill to a neutral
  surface with an accent underline, so the board itself is the brightest thing on screen.

Meridian is *not* "Classic with better colors" — Classic is preserved untouched. Meridian is
the alternative for users who find the current chrome heavy.

## 2. Layer 1 — Primitives

```css
:root.wk-theme-meridian {
  /* Cool neutral ramp */
  --wk-palette-neutral-0:    #ffffff;
  --wk-palette-neutral-50:   #f7f8fa;   /* app background        */
  --wk-palette-neutral-100:  #eef0f4;   /* board canvas          */
  --wk-palette-neutral-200:  #e6e9ef;   /* sunken / input wells  */
  --wk-palette-neutral-300:  #dce0e8;   /* subtle border         */
  --wk-palette-neutral-400:  #c3c9d6;   /* default border        */
  --wk-palette-neutral-500:  #9aa2b3;   /* strong border         */
  --wk-palette-neutral-600:  #5f6778;   /* muted text            */
  --wk-palette-neutral-700:  #4a5163;   /* secondary text        */
  --wk-palette-neutral-900:  #12161f;   /* primary text          */

  /* Accent — an ink blue chosen to clear AA on white AND on the canvas */
  --wk-palette-accent-100: #e8effc;
  --wk-palette-accent-300: #7fa5f0;
  --wk-palette-accent-500: #1f5bd0;
  --wk-palette-accent-600: #1a4db3;
  --wk-palette-accent-700: #153f96;

  --wk-palette-success-600: #0d6e49;
  --wk-palette-success-100: #e2f4ec;
  --wk-palette-warning-600: #96590a;
  --wk-palette-warning-100: #fdf0dd;
  --wk-palette-danger-600:  #b82f26;
  --wk-palette-danger-100:  #fbe9e7;
}
```

## 3. Layer 2 — Semantic mapping

```css
:root.wk-theme-meridian {
  /* Surfaces */
  --wk-surface-app:      var(--wk-palette-neutral-50);
  --wk-surface-canvas:   var(--wk-palette-neutral-100);
  --wk-surface-1:        var(--wk-palette-neutral-0);
  --wk-surface-2:        var(--wk-palette-neutral-50);
  --wk-surface-3:        var(--wk-palette-neutral-0);
  --wk-surface-sunken:   var(--wk-palette-neutral-200);
  --wk-surface-overlay:  rgb(18 22 31 / 0.45);
  --wk-surface-hover:    rgb(18 22 31 / 0.045);
  --wk-surface-active:   rgb(18 22 31 / 0.08);
  --wk-surface-selected: var(--wk-palette-accent-100);

  /* Text */
  --wk-text-primary:    var(--wk-palette-neutral-900);
  --wk-text-secondary:  var(--wk-palette-neutral-700);
  --wk-text-muted:      var(--wk-palette-neutral-600);
  --wk-text-disabled:   var(--wk-palette-neutral-500);
  --wk-text-on-accent:  #ffffff;
  --wk-text-link:       var(--wk-palette-accent-500);
  --wk-text-link-hover: var(--wk-palette-accent-600);
  --wk-text-inverse:    var(--wk-palette-neutral-0);

  /* Borders */
  --wk-border-subtle:  var(--wk-palette-neutral-300);
  --wk-border-default: var(--wk-palette-neutral-400);
  --wk-border-strong:  var(--wk-palette-neutral-500);
  --wk-border-focus:   var(--wk-palette-accent-500);

  /* Accent */
  --wk-accent:        var(--wk-palette-accent-500);
  --wk-accent-hover:  var(--wk-palette-accent-600);
  --wk-accent-active: var(--wk-palette-accent-700);
  --wk-accent-subtle: var(--wk-palette-accent-100);
  --wk-accent-focus:  var(--wk-palette-accent-500);

  /* States */
  --wk-success: var(--wk-palette-success-600);
  --wk-success-subtle: var(--wk-palette-success-100);  --wk-success-text: var(--wk-palette-success-600);
  --wk-warning: var(--wk-palette-warning-600);
  --wk-warning-subtle: var(--wk-palette-warning-100);  --wk-warning-text: var(--wk-palette-warning-600);
  --wk-danger:  var(--wk-palette-danger-600);
  --wk-danger-subtle:  var(--wk-palette-danger-100);   --wk-danger-text:  var(--wk-palette-danger-600);
  --wk-info:    var(--wk-palette-accent-500);
  --wk-info-subtle:    var(--wk-palette-accent-100);   --wk-info-text:    var(--wk-palette-accent-500);

  /* Elevation — light themes need soft, shallow shadows */
  --wk-shadow-ambient: rgb(18 22 31 / 0.08);
  --wk-shadow-direct:  rgb(18 22 31 / 0.06);

}
```

## 3F. Layer 2F — Form

Character: *crisp, calm, structured* — clearly separated panels with soft real shadows.
Meridian is deliberately less rounded than Nebula; it should read as precise, not playful.
Rationale and per-component geometry in `docs/Theme/Form-Redesign.md`.

```css
:root.wk-theme-meridian {
  /* Shape — moderate; crisper than Nebula, far softer than Classic's squares */
  --wk-shape-column:  10px;
  --wk-shape-card:     8px;
  --wk-shape-popup:   10px;
  --wk-shape-control:  6px;
  --wk-shape-chip:     6px;
  --wk-shape-avatar:  50%;

  /* Separation: real shadows. On light surfaces a black shadow shifts the
     background by ~12.9% of range, so it is genuinely visible (unlike on dark). */
  --wk-sep-strategy: shadow;
  --wk-rim-light: none;

  --wk-sep-card:       0 1px 2px rgb(18 22 31 / 0.06), 0 1px 3px rgb(18 22 31 / 0.08);
  --wk-sep-card-hover: 0 2px 4px rgb(18 22 31 / 0.08), 0 4px 10px rgb(18 22 31 / 0.10);
  --wk-sep-card-drag:  0 8px 16px rgb(18 22 31 / 0.12), 0 16px 32px rgb(18 22 31 / 0.16);
  --wk-sep-column:     0 1px 2px rgb(18 22 31 / 0.05), 0 2px 8px rgb(18 22 31 / 0.06);
  --wk-sep-popup:      0 4px 12px rgb(18 22 31 / 0.10), 0 16px 32px rgb(18 22 31 / 0.14);

  --wk-density: 1;                  /* comfortable; compact toggle matters most here */
}
```

### Structural signatures

- **Columns are cards.** `--wk-surface-2` panels at `10px` radius on the tinted canvas, with a
  `--wk-space-2` gutter and the `1px solid #ccc` divider deleted. Separation is by value step
  and a soft shadow — **no vertical rules anywhere**.
- **List headers lose their second grey bar.** Today `.list-header` paints `#e4e4e4` over the
  `#dedede` column, creating a visible seam; in Meridian the header is `transparent` on the
  panel with a balanced `12/12/8` padding rhythm instead of `28px 21px 6px`.
- **Cards are white on tint.** The value step (`#ffffff` on `#eef0f4`) does most of the
  separating; the shadow only confirms it. This is why Meridian's shadows can be so light.
- **Swimlane headers** become inset bars with a 4px leading accent and start-aligned text —
  the saturated full-bleed bands are gone; the user's swimlane color survives as the accent
  edge plus an 8%-alpha tint.
- **Chips** unify at `6px` with `--wk-space-1 --wk-space-2` padding. Due-date states move from
  raw `#ff4444` / `#ffaa00` fills to `--wk-danger-subtle` / `--wk-warning-subtle` backgrounds
  with `-text` foregrounds, so an overdue chip is legible rather than alarming.
- **Board header** is `--wk-surface-1` with a 3px accent underline (see §5), which only works
  because the header is no longer a saturated flood.

## 4. Verified contrast

Computed with the WCAG 2.1 relative-luminance formula against Meridian's three text-bearing
surfaces. **All pass AA (4.5:1).** The accent was specifically darkened from a first-pass
`#2f6feb` (which fell to 4.01 on the canvas) to `#1f5bd0` so it clears AA on *every* surface
it can appear on, including the sunken input well.

| Foreground | Hex | on surface `#ffffff` | on canvas `#eef0f4` | on sunken `#e6e9ef` |
|---|---|---:|---:|---:|
| `--wk-text-primary` | `#12161f` | 18.10 | 15.86 | 14.88 |
| `--wk-text-secondary` | `#4a5163` | 7.93 | 6.95 | 6.52 |
| `--wk-text-muted` | `#5f6778` | 5.68 | 4.98 | 4.67 |
| `--wk-accent` / link | `#1f5bd0` | 6.06 | 5.31 | 4.98 |
| `--wk-accent-hover` | `#1a4db3` | 7.62 | 6.68 | 6.26 |
| `--wk-success` | `#0d6e49` | 6.28 | 5.50 | 5.16 |
| `--wk-warning` | `#96590a` | 5.63 | 4.93 | 4.63 |
| `--wk-danger` | `#b82f26` | 6.05 | 5.30 | 4.97 |

`--wk-text-on-accent` `#ffffff` on `--wk-accent` `#1f5bd0` = **6.06** ✓

These pairs go into `tests/themeContrast.test.cjs` with their minimums.

## 5. Signature details

- **Board header** — `--wk-surface-1` with a 3px `--wk-accent` bottom border and
  `--wk-text-primary` text, instead of a saturated accent fill. Buttons in the header become
  neutral ghost buttons; only the primary action carries the accent.
- **List columns** — `--wk-surface-2` on `--wk-surface-canvas`, separated by value and
  `--wk-elev-1`, with **no** vertical rules.
- **Cards** — pure white on the tinted canvas, `--wk-shape-card`, `--wk-sep-card` at rest and
  `--wk-sep-card-hover` on hover; no border. The value step does the separating.
- **Selection** — `--wk-surface-selected` fill plus a 3px `--wk-accent` inline-start border,
  keeping the existing `boardColors.css` convention.
- **Focus** — the standard double ring from `Design-Tokens.md`, which on Meridian reads as a
  white gap then an ink-blue ring; visible on cards, canvas and inside the header alike.
- **Density** — Meridian ships at `--wk-density: 1` (comfortable) but is the theme where the
  compact toggle matters most; both must be screenshotted.
- **Labels and card colors** — user data, unchanged. Meridian only ensures label text uses
  `models/lib/contrastColor`.

## 6. Implementation checklist (Phase 5)

**Prerequisite: Phase 2a.** Until `boardColors.css` is regenerated from tokens, its 784
two-class selectors outrank every component rule Meridian's tokens feed, so the theme would
render as a partial repaint over Classic geometry. See `UI-Redesign-Plan.md` §1.3.

- [ ] `client/components/theme/theme-meridian.css` — layers 1, 2 **and 2F** exactly as above,
      in a single `:root.wk-theme-meridian` block (not a bare class — see `Design-Tokens.md`
      authoring rule 3)
- [ ] Every value a **literal**; no `color-mix()` in any `--wk-*` definition
      (`Design-Tokens.md` authoring rule 4)
- [ ] `'meridian'` in `config/const.js` `ALLOWED_BOARD_COLORS`
- [ ] `'meridian'` in the `token` category in `models/lib/themeCategories.js`
- [ ] i18n key `theme-meridian` in `en.i18n.json` **only** (Transifex supplies translations)
- [ ] Contrast pairs added to `tests/themeContrast.test.cjs`
- [ ] Token completeness passes `tests/themeTokens.test.cjs`
- [ ] Playwright baselines for the 8 reference screens, comfortable **and** compact density
- [ ] Verified: no legacy `board-color-*` class co-active (`UI-Redesign-Plan.md` §6.2)

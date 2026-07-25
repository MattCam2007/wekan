# Theme: Meridian (light)

Status: **Plan — not yet implemented** · Category: `modern` · Class: `.wk-theme-meridian`
Related: `docs/Theme/UI-Redesign-Plan.md`, `docs/Theme/Design-Tokens.md`.

Meridian is the third theme and the **reference implementation of the redesigned form**. Where
Nebula shows that the token layer can carry a dramatic, fully dark repaint, Meridian shows
what the redesign is actually *for*: a calm, high-legibility light theme for people who look
at a board for eight hours.

## 1. Design intent

The original theme is a saturated flat-blue chrome over near-white surfaces, with cramped
padding, ~20 font sizes and 13 corner radii. Meridian keeps the same information density
target but fixes the rhythm:

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
.wk-theme-meridian {
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
.wk-theme-meridian {
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

  /* Character: crisper corners than Nebula */
  --wk-radius-md: 6px;
  --wk-radius-lg: 10px;
}
```

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
- **Cards** — pure white on the tinted canvas, `--wk-radius-md`, `--wk-elev-1` at rest and
  `--wk-elev-2` on hover; no border. The value step does the separating.
- **Selection** — `--wk-surface-selected` fill plus a 3px `--wk-accent` inline-start border,
  keeping the existing `boardColors.css` convention.
- **Focus** — the standard double ring from `Design-Tokens.md`, which on Meridian reads as a
  white gap then an ink-blue ring; visible on cards, canvas and inside the header alike.
- **Density** — Meridian ships at `--wk-density: 1` (comfortable) but is the theme where the
  compact toggle matters most; both must be screenshotted.
- **Labels and card colors** — user data, unchanged. Meridian only ensures label text uses
  `models/lib/contrastColor`.

## 6. Implementation checklist (Phase 5)

- [ ] `client/components/theme/theme-meridian.css` — layers 1–2 exactly as above
- [ ] `'meridian'` in `config/const.js` `ALLOWED_BOARD_COLORS`
- [ ] `'meridian'` in the `modern` category in `models/lib/themeCategories.js`
- [ ] i18n key `theme-meridian` in `en.i18n.json` **only** (Transifex supplies translations)
- [ ] Contrast pairs added to `tests/themeContrast.test.cjs`
- [ ] Token completeness passes `tests/themeTokens.test.cjs`
- [ ] Playwright baselines for the 8 reference screens, comfortable **and** compact density
- [ ] Verified: no legacy `board-color-*` class co-active (`UI-Redesign-Plan.md` §6.2)

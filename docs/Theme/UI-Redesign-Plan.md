# WeKan UI Redesign + Multi-Theme Plan

Status: **Plan — not yet implemented** · Owner: unassigned · Related: `docs/Theme/Theme.md`
(Select Color categories), `docs/Theme/Design-Tokens.md` (token contract),
`docs/Theme/Theme-Nebula.md`, `docs/Theme/Theme-Meridian.md`, #5778 (global per-user theme),
#4759 (UI font / size / colors).

The function of WeKan is good. The *form* is the problem: the visual layer grew by
accretion, has no shared vocabulary, and every new theme costs hundreds of lines of
copy-pasted CSS. This document is the implementation plan to fix that and to ship two new
themes alongside the original, unchanged one.

Read `docs/Theme/Design-Tokens.md` for the exact token names and `Theme-Nebula.md` /
`Theme-Meridian.md` for the two new palettes. This file covers the *why*, the
*architecture*, and the *order of work*.

> **Revision 2 — after an adversarial review against the tree.** Every claim in the first
> draft was re-measured. The colour audit held up exactly (2 736 literals, 545 hex, 68
> `var()`, 2 200 `!important`, 4 327 lines); much of the rest did not. Corrected here:
>
> | # | Was | Now |
> |---|---|---|
> | 1 | `boardColors.css` out of scope, "not token-driven", optional Phase 7 | **In scope, Phase 2a, prerequisite.** It wins the cascade on every board, so the original plan was inert (§1.3) |
> | 2 | Non-goal: "no markup/template restructuring" — while §5.3 specified header zoning, a label row and popup structure | Template edits **allowed and scoped**; the contradiction is resolved (§2, §5.3, Phase 6) |
> | 3 | No mention of `font-family` anywhere in 2 176 lines | **Phase 1a** sets a real default; `fonts.css` is 100% commented out today (§1.4, §5.0) |
> | 4 | "13 radii, ~20 font sizes, 12+ shadows" | **32 / 54 / 70** — Phase 2 re-costed and split into 2a/2b/2c (§1, §7) |
> | 5 | `color-mix()` with a static fallback line above it | **Banned in theme files** — the fallback does not work inside a custom property (§3.4) |
> | 6 | `.wk-theme-<name>`, "a class beats `:root`" | **`:root.wk-theme-<name>`** — the two actually tie, and source order decided it (§3.2) |
> | 7 | A new category named `modern` | **`token`** — `modern` and `moderndark` are already theme names (§4.1) |
> | 8 | Testing table listed only the 7 invented theme tests | **Nine existing CSS-guarding tests** added as hard gates (§8.2) |
>
> Unchanged and still correct: the three-layer architecture, the Layer 2F form revision, the
> shadow-composite diagnosis of "flat", reuse of the existing theme runtime, and both palettes.

---

## 1. Audit — the measured state of the CSS

All numbers below were measured on the current tree, not estimated.

| Metric | Value | Why it matters |
|---|---|---|
| CSS files (`client/components/**`) | 71 files, **23 890 lines** | All hand-written, no preprocessor |
| Hardcoded color literals (hex + `rgb(a)`) | **2 736** (**903** of them in `boardColors.css`, 1 833 elsewhere) | Every one is a value a theme cannot reach |
| Distinct hex colors | **545** | A 19-theme app needs ~40, not 545 |
| `var(--…)` usages | **68** | Effectively no token layer exists |
| `!important` declarations | **2 200** | Specificity war; overriding needs escalation |
| `boardColors.css` | **4 327 lines** for 19 themes | ~228 lines of duplicated selectors per theme |
| `.board-color-<name> .foo` selectors | **784** | Two-class selectors that outrank every component rule (§1.3) |
| Distinct `border-radius` values | **32** | Corners disagree between adjacent components |
| Distinct `font-size` values | **54** | No type scale |
| Distinct `box-shadow` values | **70** | No elevation model |
| `font-family` on `body` | **none** | The app renders in the browser default face (§1.4) |

Worst offenders by `!important` count: `boardHeader.css` (508), `list.css` (395),
`header.css` (309), `boardColors.css` (201), `popup.css` (145), `cardDetails.css` (137).

> **The geometry counts above are the corrected ones.** The first draft of this audit reported
> 13 radii, ~20 font sizes and "12+" shadows. Those numbers came from a sample that excluded
> `boardColors.css` and the `!important` variants. Measured across the whole tree the real
> figures are 32 / 54 / 70 — between 2× and 5× larger — which is why Phase 2 is re-costed
> in §7 and no longer claims to be purely mechanical.

### 1.1 What is already good (and must be kept)

The *plumbing* for themes is already in place and well built. This plan reuses it wholesale
rather than replacing it:

- **`config/const.js` → `ALLOWED_BOARD_COLORS`** — the canonical theme-name list.
- **`models/lib/themeCategories.js`** — categories (`flat` / `clear` / `dark` / `special`),
  custom-color counts, validation helpers. Pure CommonJS, unit-testable.
- **`models/users.js`** — `profile.globalThemeColor` + `profile.globalThemeCustomColors`
  (hex-validated), with `getGlobalThemeColor()` / `getGlobalThemeCustomColors()`.
- **`client/components/main/globalThemeColor.js`** — a `Tracker.autorun` that puts
  `board-color-<name>` on `<body>` and sets `--theme-accent` / `--theme-accent-2`.
- **`client/components/main/themeColorPicker.{js,jade}`** — the shared picker used by both
  Board Settings and Member Settings.
- **Template hooks** — `header.jade`, `boardBody.jade` already apply
  `currentUser.globalThemeColorClass` falling back to `currentBoard.colorClass`.
- **Logical properties** — the CSS already uses `inset-inline-start` / `margin-inline-end`
  etc., so RTL works. Tokens must not regress this.
- **Existing tests** — `tests/themeCategories.test.cjs`, `tests/globalThemeColor.test.cjs`,
  `tests/themeColorPicker.test.cjs`, `tests/buttonThemeColors.test.cjs`.

**Conclusion: the theme *runtime* is fine. The theme *substrate* is missing.** A theme today
can only reach the ~68 `var()` sites and whatever it re-declares with higher specificity —
which is exactly what `boardColors.css` does, at 4 327 lines, and why the app has no real dark
mode.

That last clause is the important one and is developed in §1.3: the legacy file is not merely
verbose, it is the file that **wins the cascade on every board page**. Tokenizing everything
else while leaving it alone changes nothing on screen.

### 1.2 Build constraints

`rspack.config.js` uses `['style-loader', 'css-loader']` with no PostCSS, no autoprefixer,
and no `browserslist`. So:

- Native CSS custom properties are the only viable token mechanism (they are resolved by the
  browser at runtime — exactly what theme switching needs). **No build changes required.**
- No nesting, no `@custom-media`, no build-time color functions. Any derived color must be
  written literally or computed with `color-mix()` at runtime (see §3.4).

### 1.3 The cascade: `boardColors.css` decides what is actually on screen

**This is the single most important fact in the audit, and the first draft of this plan got it
wrong.** It listed `boardColors.css` under non-goals ("they stay; they are simply not
token-driven") and deferred it to an optional Phase 7. That would have made Phases 2–5 a no-op
on every real board. Three measured facts:

1. **Every board always has a color.** `models/boards.js:426` sets `BOARD_COLORS[0]`
   (`belize`) as an insert `autoValue`. There is no "no color" state. `header.jade:103` and
   `boardBody.jade:26` therefore always emit a `board-color-<name>` class on `#header` and
   `.board-wrapper`.
2. **Those rules outrank component CSS.** `boardColors.css` contains **784** selectors of the
   form `.board-color-<name> .foo` — specificity (0,2,0) — against the (0,1,0) single-class
   rules in `minicard.css`, `list.css` and the rest. No `!important` is involved; the legacy
   file simply wins.
3. **It carries geometry, not just color.** Inside `boardColors.css`: **126** `border-radius`
   (20 distinct values), **101** `padding`, **47** `box-shadow`, **62** `font-size`.

The default theme, today, as rendered:

```css
/* client/components/boards/boardColors.css:228 — and 16 other themes identically */
.board-color-belize .minicard {
  border-radius: 7px;
  padding: 10px 10px 4px 10px;
  box-shadow: 2px 2px 4px 0px rgba(0,0,0,0.15);
}
```

So the "Classic minicard radius" is **7px**, not the `4px` in `minicard.css`. Any plan that
pins `--wk-shape-card: 4px` and calls that preservation is pinning a value no user has ever
seen. 24 such geometry blocks exist across the 19 legacy themes (17 at `7px`, one at `12px`,
one at `2px`).

**Consequences, all of which this revision applies:**

- `boardColors.css` moves **into** Phase 2a scope (§7). Regenerating it from tokens is the
  *enabler* of the whole plan, not a cleanup afterthought.
- The Phase 2b exit criterion may no longer be measured "outside `boardColors.css`" — that
  metric goes green without changing anything a user sees.
- Classic's Layer 2F values must be seeded from the **rendered** cascade (`7px` card,
  `10px 10px 4px 10px` padding), not from the shadowed base rule.
- §9's mitigation "tokens are consumed inside the existing `!important` declarations, so the
  value changes without touching specificity" was answering the wrong objection. `!important`
  was never the problem; selector specificity in an excluded file was.

### 1.4 There is no typeface

`client/components/main/fonts.css` is imported at `client/styles.js:33` and its entire
contents — every `@font-face` for Roboto and Poppins — sit inside one `/* … */` block. No
rule anywhere in `client/components` sets `font-family` on `body` or `html`. The app renders
in whatever the browser defaults to.

`--wekan-ui-font` (#4759) exists but `uiFont.css` only applies it under the `has-ui-font`
class, i.e. **only when a user has explicitly opted in**. It is an override, not a default.

A type *scale* without a typeface fixes half the problem. The first draft of this plan had
seven type steps and no `font-family` anywhere in 2 176 lines of specification. Phase 1a (§7)
now sets a real default stack, and `--wk-font-sans` / `--wk-font-mono` become tokens
(`Design-Tokens.md`).

---

## 2. Goals and non-goals

### Goals

1. **A design-token layer** so a theme is a list of ~120 values, not 228 lines of selectors.
2. **The original theme is preserved exactly** — pixel-identical *as rendered* (i.e. including
   `boardColors.css`, per §1.3), enforced by visual regression, not by eyeball.
3. **Two new themes**, both token-only:
   - **Nebula** — dark, derived from the supplied deep-space nebula image.
   - **Meridian** — light, calm, high-legibility; the reference implementation of the
     redesigned form.
4. **Fix the form**: one typeface, one type scale, one spacing scale, one radius scale, one
   elevation model, one focus-ring system, consistent controls.
5. **Accessibility**: every foreground/background pair in every shipped theme meets WCAG 2.1
   AA (4.5:1 body, 3:1 large text and UI boundaries), verified by an automated test.
6. **All 19 existing board colors keep working** — same names, same appearance, but sourced
   from tokens instead of 4 327 lines of duplicated selectors (§1.3).

### In scope, and previously mis-scoped

- **`boardColors.css` is in scope from Phase 2a.** See §1.3. It is the file that decides what
  renders; excluding it made the rest of the plan inert.
- **Template edits are in scope where a component's geometry needs them.** The first draft
  listed "no markup/template restructuring" as a non-goal while §5.3 simultaneously specified
  header zoning, a `flex-wrap` label row and popup header/body/footer rhythm — none of which
  are reachable from CSS alone. That contradiction is resolved in favour of allowing the
  edits, with the limits below.

### Non-goals (explicitly out of scope)

- **No behavioural or structural template change.** Markup edits are permitted only to add or
  regroup presentational containers (a wrapper, a zone div, a class). No change to Blaze
  helpers, event handlers, data context, `js-*` hook classes, or the order in which
  interactive elements appear in the DOM. Every markup commit states which of these it did
  not touch.
- No CSS framework (Tailwind/Bootstrap), no preprocessor, no build-pipeline change.
- No component-library migration, no Blaze→React work.
- Not deleting the 19 legacy board colors. They keep their names and their appearance.
- Not removing all 2 200 `!important` in one pass (see Phase 7 — it is a follow-up, and the
  plan is deliberately designed so it is *not* a prerequisite).
- No new webfont **asset**. Phase 1a uses a system stack; shipping a bundled face is a
  separate maintainer decision (§10.6).

---

## 3. Architecture — three token layers

```
Layer 1  PRIMITIVES   --wk-palette-*    raw colors, no meaning     (per theme)
              │
Layer 2  SEMANTIC     --wk-surface-*    meaning, no component      (per theme)
              │       --wk-text-*  --wk-border-*  --wk-accent-*  --wk-state-*
              │
Layer 2F FORM         --wk-shape-*      shape, depth, density      (per theme)
              │       --wk-sep-*  --wk-rim-light  --wk-density
              │
Layer 3  COMPONENT    --wk-card-bg      component-scoped aliases   (theme-agnostic)
              │       --wk-list-header-bg  --wk-popup-bg  …
              ▼
         component CSS consumes ONLY layers 2, 2F and 3
```

**The rule that makes this work: component CSS never names a color, a radius or a shadow.**
It references a semantic, form or component token. A theme then redefines layers 1, 2 and 2F.

> **Revision — form is themeable.** The first draft of this plan limited themes to color and
> deferred shape/depth to a final "Phase 6". That was a mistake: recoloring a square, flat
> layout yields a square, flat layout in a new color. **Layer 2F** (shape by role, separation
> strategy, density) is now part of every theme, and the form work moves into Phases 1–5.
> See **`docs/Theme/Form-Redesign.md`** for the measured diagnosis and the per-component
> geometry — including why black shadows are ~10× less visible on dark surfaces, which is the
> real reason the current dark theme reads as flat.

The remaining scales (space, type, motion, z-index) live in a single theme-agnostic file and
are **not** overridden per theme.

### 3.1 File layout

```
client/components/theme/
  _tokens.css              layer 3 aliases + non-color scales (space/type/radius/elevation)
  theme-classic.css        :root  — layer 1+2 for the ORIGINAL look (default, no class)
  theme-nebula.css         .wk-theme-nebula   — layer 1+2
  theme-meridian.css       .wk-theme-meridian — layer 1+2
  theme-nebula-decor.css   Nebula-only backdrop/glow (optional, perf-guarded)
```

Imported from `client/styles.js` **first**, before every component stylesheet, so that
tokens are defined before use and component rules keep their existing cascade order:

```js
// client/styles.js — must be the first imports in the file
import '/client/components/theme/_tokens.css';
import '/client/components/theme/theme-classic.css';
import '/client/components/theme/theme-nebula.css';
import '/client/components/theme/theme-meridian.css';
import '/client/components/theme/theme-nebula-decor.css';
```

### 3.2 Scoping and precedence

Classic is defined on `:root` (unconditional default). Nebula/Meridian are defined on a
`.wk-theme-<name>` class placed on `<html>` by the runtime (§6). Custom properties inherit, so
every descendant picks up the override with no `!important` anywhere.

**Write the theme selector as `:root.wk-theme-<name>`, not `.wk-theme-<name>`.** An earlier
draft claimed "a class selector beats `:root` at equal specificity ordering". That is not how
it works: `:root` is specificity (0,1,0) and a bare `.wk-theme-nebula` is *also* (0,1,0), so
the two tie and the winner is decided by source order — i.e. by the order of the `import`
statements in `client/styles.js` and by however the bundler happens to insert them. Nothing
tests that order, and a future refactor of `styles.js` would silently revert every user to
Classic. `:root.wk-theme-nebula` is (0,2,0) and wins unconditionally:

```css
:root                    { /* Classic — layers 1, 2, 2F */ }
:root.wk-theme-nebula    { /* Nebula overrides           */ }
:root.wk-theme-meridian  { /* Meridian overrides         */ }
```

Put the theme class on `<html>`, **not** `<body>`: `<body>` already carries
`board-color-*`, `has-custom-theme-color`, `has-ui-font`, `has-ui-font-size` etc., and
keeping the axes on separate elements avoids collisions.

Precedence, highest first:

1. `--theme-accent` / `--theme-accent-2` custom colors (existing #5778 behaviour, set inline
   on `documentElement` — inline style wins over any class).
2. `.wk-theme-<name>` on `<html>` — Nebula / Meridian.
3. `board-color-<name>` legacy rules on `<body>` / `#header` / `.board-wrapper`.
4. `:root` Classic defaults.

**Guard:** a legacy `board-color-*` theme and a new `.wk-theme-*` theme must never be active
at once — the legacy rules hardcode colors and would half-override a token theme. The runtime
enforces mutual exclusion (§6.2).

### 3.3 Backward compatibility for existing variables

Do not rename existing public variables. Redefine them *in terms of* the new tokens so
current behaviour and `tests/buttonThemeColors.test.cjs` keep passing:

```css
:root {
  /* --theme-accent stays the #5778 custom-color override hook. Component CSS keeps
     using var(--theme-accent, <fallback>); we only change the fallback to a token. */
  --wk-accent: #2980b9;                 /* per-theme */
  --wk-accent-hover: #2471a3;
  --wk-accent-active: #1f618d;
}
```

`forms.css` currently asserts literal fallbacks such as `var(--theme-accent, #005377)`.
Phase 2b rewrites these to `var(--theme-accent, var(--wk-accent-strong))` **and updates
`tests/buttonThemeColors.test.cjs` in the same commit** — that test matches the literal
string and will otherwise fail. Same for `--list-width`, `--swimlane-height`,
`--wekan-ui-*` (#4759): keep the names, source the defaults from tokens.

### 3.4 Derived colors

No build-time color math is available. Two allowed techniques:

- **Literal derivation** — write the computed value into the theme file with a comment
  giving its origin (`/* accent -12% L */`). Preferred for the ~8 accent ramp steps.
- **`color-mix()`** — for hover/pressed/disabled states and translucent overlays, **in
  component CSS only**, and only with a static fallback declaration immediately above it:

  ```css
  .btn:hover { background: #2471a3; }
  .btn:hover { background: color-mix(in oklab, var(--wk-accent) 88%, black); }
  ```

  Baseline support is fine for WeKan's targets (Chrome/Edge 111+, Firefox 113+, Safari 16.2+).

> **`color-mix()` is BANNED inside custom-property definitions**, i.e. banned from every theme
> file. The fallback trick above works because an unsupported `color-mix()` in a *longhand*
> declaration is a parse error, so the declaration is dropped and the previous one wins.
> Custom-property values are **not** parsed at declaration time — they are stored as raw
> tokens and only substituted at `var()` time. So this:
>
> ```css
> :root.wk-theme-nebula { --wk-accent-hover: color-mix(in oklab, var(--wk-accent) 88%, black); }
> .btn:hover { background: #2471a3; }          /* "fallback" */
> .btn:hover { background: var(--wk-accent-hover); }
> ```
>
> is accepted by *every* browser at parse time, and then on a non-supporting one the
> substitution fails at computed-value time. `background` becomes invalid-at-computed-value-time
> → `unset` → transparent. The fallback is discarded, which is precisely the failure the rule
> claims to prevent. Since `Design-Tokens.md` defines a theme as "a flat list of custom
> properties", that is where essentially every `color-mix()` would otherwise live.
>
> **Theme files use literal derivation only.** Where a derived ramp is genuinely wanted, wrap
> the whole theme block in `@supports (color: color-mix(in oklab, red 50%, blue))` and provide
> a literal-valued block before it. `tests/themeTokens.test.cjs` fails on a `color-mix(` inside
> any `--wk-*` definition that is not inside such an `@supports`.

---

## 4. Theme roster

| Theme | Category | Class | Basis | Custom colors |
|---|---|---|---|---|
| **Classic** (original) | default | *(none — `:root`)* | Today's exact appearance *as rendered* (§1.3) | via `--theme-accent` as today |
| **Nebula** | `token` | `:root.wk-theme-nebula` | The supplied nebula image | 0 initially, 1 in Phase 7 |
| **Meridian** | `token` | `:root.wk-theme-meridian` | Original light design | 0 initially, 1 in Phase 7 |

**Classic is the default and is not a picker entry** — it is what `:root` yields when no theme
class is present, which is exactly today's behaviour. This is how "keep the original theme"
is satisfied strictly: not by recreating it, but by *extracting its current values into
tokens* — the values it actually renders at, including the `boardColors.css` layer — and
proving with screenshots that nothing moved.

### 4.1 Registering the two new themes

Add a **fifth category** rather than extending `dark` / `special`. The existing tests assert
the exact membership of those arrays (`colorsInCategory('dark')` is a `deepStrictEqual`
against a 5-name list), and a new category keeps the story clear: *token-driven, fully themed,
contrast-verified.*

> **The category is called `token`, not `modern`.** `modern` is already taken — twice.
> `ALLOWED_BOARD_COLORS` (`config/const.js:16-17`) contains the theme names `'modern'` and
> `'moderndark'`, and `themeCategories.js` already files `'modern'` under `special` and
> `'moderndark'` under `dark`. A category named `modern` would put two meanings of the same
> string in one module — `categoryOf('modern') === 'special'` while
> `colorsInCategory('modern') === ['nebula','meridian']` — and `themeColorPicker.js:55` builds
> its label key as `theme-category-${key}`, so the picker would show a `theme-category-modern`
> heading directly above an unrelated theme called *modern*.

```js
// models/lib/themeCategories.js
const THEME_CATEGORIES = {
  flat:    [...],
  clear:   ['clearblue'],
  dark:    [...],
  special: [...],                           // still contains the legacy 'modern'
  token:   ['nebula', 'meridian'],          // NEW — token-driven themes
};
const THEME_CATEGORY_ORDER = ['flat', 'clear', 'dark', 'special', 'token'];
const CUSTOM_COLOR_COUNT = { flat: 1, clear: 2 };   // token: 0 until Phase 7
```

A negative test asserts that no category name is also a theme name, so this cannot recur.

Files that must change together (all guarded by existing tests):

1. `config/const.js` — append `'nebula'`, `'meridian'` to `ALLOWED_BOARD_COLORS`.
   `models/metadata/colors.js` derives `BOARD_COLORS` from it automatically.
2. `models/lib/themeCategories.js` — as above.
3. `tests/themeCategories.test.cjs` — update `THEME_CATEGORY_ORDER` and the union assertion.
4. `imports/i18n/data/en.i18n.json` — display names + descriptions (`theme-nebula`,
   `theme-meridian`, `theme-category-token`). English only; Transifex supplies the rest per
   the translation policy in `CLAUDE.md` — **do not hand-fill other languages**.

Because they are registered as ordinary theme names, they inherit *for free*: the picker UI,
server-side validation, the global per-user override (#5778), per-board selection, and
persistence. No new schema, no new publication, no new method.

---

## 5. Fixing the form

> **Full specification: `docs/Theme/Form-Redesign.md`.** That document measures why the UI
> reads as *square, boxy and flat* (columns with no radius and no gutter, separated by a 1px
> line; full-bleed swimlane strips; black shadows that are invisible on dark; `1px 3px` chip
> padding; 32 disagreeing radii) and specifies the replacement geometry per component. This
> section is the summary.

The "horrific form" complaint is about typeface, shape, depth, rhythm and hierarchy. It is
fixed by the typeface below, the scales below, **plus the Layer 2F form tokens** (§3), which
apply to all themes — seeded with Classic's *rendered* values so Classic does not move until
Phase 6.

### 5.0 The typeface (Phase 1a)

Per §1.4 there is no `font-family` on `body` at all, and `fonts.css` is 100% commented out.
This is the cheapest large improvement available and it is not a theme concern — it is a
default the app has simply never had. Set it once, in `_tokens.css`:

```css
:root {
  --wk-font-sans: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue",
                  "Noto Sans", Arial, sans-serif;
  --wk-font-mono: ui-monospace, SFMono-Regular, "SF Mono", Menlo, Consolas,
                  "Liberation Mono", monospace;
}
body { font-family: var(--wk-font-sans); }
```

Rules:

- **A system stack, no bundled asset.** No new webfont file, no network fetch, no CLS from a
  swap. Shipping an actual face (the commented-out Roboto/Poppins, or something else) is a
  separate maintainer decision — §10.6.
- **`--wekan-ui-font` still wins.** `uiFont.css` applies it under `has-ui-font` with
  `!important`, so the #4759 per-user preference keeps overriding this default. The default is
  what users who have *not* set a preference finally get.
- **A theme may override `--wk-font-sans`**, and this is the second non-color scale a theme may
  touch (the first being `--wk-radius-*` / `--wk-shape-*`). Neither Nebula nor Meridian does
  in Phase 4–5; the hook exists for later.
- **`--wk-font-mono` replaces** the ad-hoc `font-family: monospace`, `lucida console, monospace`
  and `Courier` in `attachments.css`, `layouts.css`, `originalPositionsView.css` and
  `globalSearch.css`.
- Changing the default face **moves Classic**, so it lands with baseline updates and needs the
  same sign-off as Phase 6. It is split out as Phase 1a precisely so that sign-off is about one
  small, isolated diff rather than being buried in a 71-file migration.

### 5.1 Scales

- **Space** — 4px base: `--wk-space-1:4px` … `--wk-space-8:48px`. Replaces the current ad-hoc
  `3px/5px/6px/8px/10px/12px/14px/21px/23px` mix.
- **Type** — 7 steps: `12 / 13 / 14 / 16 / 18 / 20 / 24px` mapped to
  `--wk-text-xs … --wk-text-3xl`, with `--wk-leading-tight|normal|relaxed`. The existing **54**
  sizes collapse into these — see the re-costing note in §7, Phase 2b.
- **Radius** — 4 steps: `--wk-radius-sm:4px`, `-md:6px`, `-lg:10px`, `-full:999px`. The **32**
  current values collapse into these; `50%` stays only for avatars.
- **Elevation** — 5 steps `--wk-elev-0…4`, each a *pair* of shadows (ambient + direct).
  Themes override the shadow *color* only (dark themes need much stronger, cooler shadows).
- **Motion** — `--wk-motion-fast:120ms`, `-base:180ms`, `-slow:260ms`, plus
  `--wk-ease-standard`. All wrapped in `@media (prefers-reduced-motion: reduce) { … 0ms }`.

### 5.2 Density and character

Kanban users want density; the current UI is *cramped* rather than dense, which is different.
Introduce a density multiplier consumed by card/list padding:

```css
:root                      { --wk-density: 1; }      /* comfortable */
.wk-density-compact        { --wk-density: 0.75; }
```

Themes may also re-map `--wk-radius-*` to express character (Nebula: softer, `md:8px`;
Meridian: crisper, `md:6px`). This is the *only* non-color scale a theme may touch.

### 5.3 Component work (Phases 2–6)

Ordered by user-visible impact. Full geometry in `Form-Redesign.md` §3. The **Markup** column
is the honest accounting the first draft lacked — it claimed no template work while specifying
several items that cannot be done from CSS:

| # | Component | Work | Markup |
|---|---|---|---|
| 1 | **Minicard** (`minicard.css`, 979 lines) | Align the badge row on one baseline, cap label chips with a consistent height/radius, single truncation strategy for titles, one elevation for rest / one for drag. Today: `border-radius:4px` on the card but `17px` on the drag placeholder and `6px` on the "and N other" chip — and `7px` from `boardColors.css`, which is what actually renders (§1.3). | **Yes** — the label squares become a `flex-wrap` row, which needs a wrapper element in `minicard.jade` |
| 2 | **List column** (`list.css`, 1 596 lines, 395 `!important`) | Consistent header height, one padding rhythm, sticky header that does not shift on scroll. | No |
| 3 | **Board header** (`boardHeader.css`, 1 057 lines, 508 `!important`) | The single worst file; unify button sizes into one `.board-header-btn` spec with real focus states. | **Yes** for the three-zone grouping — see the warning below |
| 4 | **Popups** (`popup.css`) | One width scale, one padding, consistent header/footer. | **Yes** — the header/body/footer rhythm needs those three containers to exist |
| 5 | **Forms** (`forms.css`) | One input height, one border/focus treatment, real `:focus-visible` rings replacing the current mixed outline/box-shadow approaches. | No |
| 6 | **Sidebar**, **card details**, **admin/settings tables** | Apply the same scales. | No |

> **Warning on the board header (§5.3 item 3, `Form-Redesign.md` §3.5).** `boardHeader.jade`'s
> current grouping is not arbitrary and is not decorative. `.board-header-btns-group` exists so
> that *all* buttons wrap together under a long board title instead of splitting across two
> rows, and `.board-header-sidebar-toggle` is a separate flex item ordered last specifically so
> the hamburger can sit top-right in mobile mode. Both facts are recorded in comments in the
> template, and both were bug fixes. Re-zoning into identity · view controls · actions must
> preserve them, and the mobile-mode tests (`tests/mobileModeConsistency.test.cjs`,
> `tests/mobileBoardFit.test.cjs`) are the guard. If the three-zone grouping cannot be reached
> without breaking the wrap behaviour, **drop the zoning and keep the wrap fix** — it is a
> smaller loss.

### 5.4 Focus and keyboard accessibility

Today focus styling is inconsistent (`box-shadow: 0 0 0 2px #666 inset`, various outlines,
several `outline: none`). Standardise:

```css
--wk-focus-ring: 0 0 0 2px var(--wk-surface-1), 0 0 0 4px var(--wk-accent-focus);
:where(a, button, input, select, textarea, [tabindex]):focus-visible {
  outline: none;
  box-shadow: var(--wk-focus-ring);
}
```

The double ring keeps the indicator visible on *any* surface, including inside colored
headers. Requirement: ≥3:1 between `--wk-accent-focus` and the surface behind it, checked by
the contrast test.

---

## 6. Runtime

### 6.1 Applying the class

Extend `client/components/main/globalThemeColor.js` — it already resolves the active theme
name for both the global override and the per-board case. Add a sibling to `applyClass()`:

```js
const TOKEN_THEMES = ['nebula', 'meridian'];   // import from themeCategories: colorsInCategory('token')

function applyThemeClass(name) {
  const root = document.documentElement;
  TOKEN_THEMES.forEach(t => root.classList.remove(`wk-theme-${t}`));
  if (name && TOKEN_THEMES.includes(name)) root.classList.add(`wk-theme-${name}`);
}
```

### 6.2 Mutual exclusion with legacy board colors

In the same autorun, when the resolved theme is a `token` one, **do not** add the
`board-color-<name>` class to `<body>`; when it is legacy, **do not** add `wk-theme-*`.
`header.jade` / `boardBody.jade` must likewise emit no `board-color-*` class for a token
theme — add a `currentBoard.isTokenTheme` / `currentUser.isTokenTheme` helper and guard the
existing `class="…"` expressions. Cover this with a unit test extending
`tests/globalThemeColor.test.cjs`.

**This guard has a product consequence that needs a decision (§10.4).** Because every board
always carries a color (§1.3), suppressing `board-color-*` for a token theme means a user who
picks Nebula globally loses the per-board color distinction everywhere — every board looks
identical. That is defensible for a deliberate whole-app theme, but it is a behaviour change,
not an implementation detail, and it must be stated in the picker UI rather than discovered.

### 6.3 Flash-of-unstyled-theme

The autorun runs after hydration, so a Nebula user sees a white flash on load. Mitigate by
writing the resolved theme name to `localStorage` on change and reading it in a tiny
synchronous snippet injected via `imports/lib/customHeadDefaults.js` before first paint. Must
be defensive: wrap in `try/catch`, validate against the known-theme list (never inject an
unvalidated string into a class name), and fall back silently to Classic.

### 6.4 Follow-ups (not in this plan's phases)

- **Auto mode** — follow `prefers-color-scheme`, mapping dark→Nebula, light→Meridian.
- **Per-board override of a global theme** — currently the global override wins everywhere;
  a per-board opt-out is a separate UX decision (see §10).

---

## 7. Phased implementation

Each phase is independently shippable and independently revertable.

> **Revision — the order changed, and so did the ambition of Phase 2.** The first draft
> sequenced the work for reviewability: tooling, then a 71-file mechanical migration, then the
> runtime, and only then something a user could see. Under that order the first visible
> improvement arrived in Phase 4, and — because `boardColors.css` was excluded (§1.3) — Phases
> 1–3 would have produced no visible change *at all*, on any board, ever. Three changes:
>
> 1. **Phase 1a (typeface) is new and comes first.** It is ~15 lines, it is the largest
>    single visual improvement per line of diff in the entire plan, and it depends on nothing.
> 2. **`boardColors.css` is regenerated in Phase 2a, before the component migration.** It is
>    the cascade authority; migrating the files it overrides first is working backwards.
> 3. **Phase 2 splits into 2a/2b/2c** because collapsing 32 radii, 54 font sizes and 70 shadows
>    into 4/7/5 is not the "mechanically reviewable, screenshot-clean" operation the first
>    draft described. Colour substitution is mechanical. Scale collapsing is a redesign, it
>    *will* produce screenshot diffs, and it is now costed and sequenced as such.

### Phase 0 — Tooling (no visual change)

- `tests/themeTokens.test.cjs` — parse the theme CSS files; assert every theme defines
  **every** token named in `_tokens.css` (no silent fallbacks).
- `tests/themeContrast.test.cjs` — pure-JS WCAG ratio computation over a declared list of
  (foreground, background, minimum) triples per theme. Pure CommonJS, no browser, no network
  — same style as the existing `.cjs` tests.
- Visual-regression harness: Playwright screenshots of 8 reference screens (All Boards, board
  with swimlanes, card details, popup, admin settings, My Cards, global search, login) × each
  theme. Baselines committed under `tests/visual/`.

**Exit:** harness runs green on the untouched tree and produces Classic baselines.

### Phase 1a — The typeface (small, visible, independent)

Per §1.4 and §5.0: define `--wk-font-sans` / `--wk-font-mono`, set `body { font-family }`,
replace the four ad-hoc `monospace` / `Courier` / `lucida console` declarations. Either delete
the fully commented-out `fonts.css` or leave it commented with a note pointing at §10.6.

This **moves Classic** — deliberately, and it is the one place in the plan where that is worth
it before Phase 6. It is isolated so the sign-off is about one 15-line diff.

**Exit:** every screen renders in the stack; `--wekan-ui-font` still overrides it (verify with
a user who has set the #4759 preference); fresh Classic baselines approved.

### Phase 1b — Token layer + Classic extraction

- Author `_tokens.css` and `theme-classic.css`, seeding every value from the colors **and
  geometry Classic actually renders at** — which means reading `boardColors.css` as well as
  the component files (§1.3). Classic pins `--wk-shape-column: 0`, **`--wk-shape-card: 7px`**
  (the `belize` value, not `minicard.css`'s shadowed `4px`), `--wk-shape-popup: 6px` and the
  `2px 2px 4px 0px rgb(0 0 0 / 0.15)` card shadow that `boardColors.css` supplies.
- Change **no** component file yet.

**Exit:** tokens defined and unused; zero screenshot diff by construction.
`tests/themeForm.test.cjs` pins Classic's Layer 2F values — **asserted against the rendered
cascade, not against a single file** — so later phases cannot move them.

### Phase 2a — Regenerate `boardColors.css` from tokens

The cascade authority first (§1.3). Each of the 19 legacy themes becomes a short block of
custom-property overrides instead of ~228 lines of duplicated selectors; the 784
`.board-color-<name> .foo` rules collapse into token definitions on `.board-color-<name>`.

- Every legacy theme keeps its **name** and its **rendered appearance** — this is a pure
  refactor, verified per-theme by screenshot, not a redesign.
- The 126 `border-radius`, 101 `padding`, 47 `box-shadow` and 62 `font-size` declarations in
  the file become Layer 2F token values, which is what makes them reachable at all.
- Where a legacy theme genuinely needs a component-specific rule that no token expresses,
  that rule stays — but it is listed in the commit message as residue, and the count is a
  tracked number.

**Exit:** `boardColors.css` drops from 4 327 lines toward ~600; the 903 literals in it drop
below 100; **all 19 legacy themes screenshot-identical**; component rules are no longer
outranked, so Phase 2b's work becomes visible. This phase carries the highest regression risk
in the plan and gets per-theme baselines for all 19.

### Phase 2b — Migrate component CSS to tokens (color)

One commit per file, largest-impact first, using the measured line counts:
`forms.css` → `minicard.css` → `list.css` → `header.css` → `boardHeader.css` →
`cardDetails.css` → `popup.css` → `sidebar.css` → `boardBody.css` → `swimlanes.css` →
`boardsList.css` → `layouts.css` → the remaining ~50 smaller files.

For each file: replace every color literal with the semantic token whose Classic value is
byte-identical. **Do not** change specificity, do not remove `!important`, do not restructure
layout, **do not yet collapse scales** (that is 2c). This half genuinely is mechanical and
genuinely is screenshot-clean.

**Exit:** hardcoded literals across `client/components/**` drop from 2 736 to <150 (permitted
residue: pure-decorative gradients, spinner keyframes, `#fff`/`#000` inside shadow
definitions). Zero screenshot diff per commit. Note the metric is now measured over the
**whole** tree — the earlier "outside `boardColors.css`" phrasing let the number go green
without changing anything a user sees.

### Phase 2c — Collapse the geometry scales

Substitute `--wk-shape-*` / `--wk-space-*` / `--wk-text-*` / `--wk-elev-*` for the literal
geometry. **This is a redesign, not a substitution**, and it is where the corrected audit
numbers bite: 32 radii → 4, 54 font sizes → 7, 70 shadows → 5. Most values have no exact token
and must be *snapped* to the nearest step, which moves pixels.

- Per-file commits, each with an explicit list of every value that changed and by how much.
- Screenshot diffs are **expected**; each is reviewed and approved rather than asserted to be
  zero. This is the phase the first draft mislabelled as mechanically reviewable.
- Where snapping a value would break a mobile-mode or width test, keep the literal and record
  it; `tests/noViewportSpacing.test.cjs` and the four mobile tests (§8) are hard gates.

**Exit:** the four scales are what component CSS references; the residue list of un-snapped
literals is <50 and each entry has a reason.

### Phase 3 — Theme runtime

`config/const.js`, `themeCategories.js`, the `token` category, `globalThemeColor.js` class
application, template guards, i18n keys, updated tests. Includes the negative test that no
category name is also a theme name (§4.1).

**Exit:** picker shows a `token` category; selecting an entry toggles `wk-theme-*` on
`<html>` and suppresses `board-color-*`; the §6.2 behaviour change is surfaced in the picker
copy. No theme CSS yet, so both render as Classic.

### Phase 4 — Nebula (color **and** form)

Implement `theme-nebula.css` + `theme-nebula-decor.css` per `docs/Theme/Theme-Nebula.md` —
Layer 1, 2 **and 2F**: `16px` columns, `14px` cards, pill chips, rim-light separation.

**Exit:** contrast test green; `themeForm` dark-separation check green; Nebula baselines;
decor within the perf budget. Nebula looks like a different application, not a repaint.

### Phase 5 — Meridian (color **and** form)

Implement `theme-meridian.css` per `docs/Theme/Theme-Meridian.md` — `10px` columns, `8px`
cards, soft shadows, comfortable density.

**Exit:** contrast test green; Meridian baselines at both densities.

### Phase 6 — Structural work (markup)

The changes that alter the **layout box model itself** and therefore move Classic, and the
only phase containing template edits: `Form-Redesign.md` §3.1 (delete the 1px column divider,
add gutters and column radius), §3.2 (swimlane strip → inset panel), §3.4 (the minicard label
row wrapper), §3.5 (header zoning), §3.6 (popup header/body/footer containers).

Together with Phase 1a and Phase 2c this is where Classic's appearance intentionally changes,
so it needs explicit maintainer sign-off and per-component baseline updates. Nebula and
Meridian already carry their own shape from Phases 4–5; this phase lets Classic benefit too.

**Markup discipline** (per the revised §2 non-goals): presentational containers and classes
only. Each commit's message states explicitly that it changed no Blaze helper, no event
handler, no data context, no `js-*` hook class, and no interactive DOM order. Reviewers check
that claim against the diff.

**Highest-risk item — the column gutter.** Adding `var(--wk-space-2)` per column reduces the
number of visible columns and interacts directly with the mobile-mode work that landed
immediately before this plan (`mobileBoardFit`, `mobileModeFullWidth`, `mobileModeConsistency`,
`mobileAddList`, `listWidthDefaults`, `sidebarWidth`) and with the per-list `--list-width`
inline style set from `list.jade:3`. Verify at 1280px, at the mobile breakpoint, at both
densities, and with a fixed per-list width set — before the swimlane and header items, so a
revert is cheap.

**Exit:** one type/space/shape/elevation scale in use; focus rings consistent; baselines
re-approved; column-gutter width verified across the matrix above; all nine CSS-guarding tests
in §8 green.

### Phase 7 — Cleanup

Reduce `!important` now that tokens make overrides unnecessary (target: <500) — Phase 2a
should already have removed a large share of the 201 in `boardColors.css` and much of the
pressure that justified the rest. Enable one custom accent for `token` themes with
contrast-safe on-accent derivation.

(The `boardColors.css` regeneration that used to live here is now Phase 2a — it is a
prerequisite, not a cleanup. See §1.3.)

---

## 8. Testing

### 8.1 New and updated theme tests

| Test | Type | Guards |
|---|---|---|
| `tests/themeTokens.test.cjs` | unit, pure | Every theme defines every token, layers 1/2/2F; **no `color-mix()` in a `--wk-*` definition outside an `@supports`** (§3.4) |
| `tests/themeContrast.test.cjs` | unit, pure | WCAG AA for all declared pairs |
| `tests/themeForm.test.cjs` *(new)* | unit, pure | Classic's Layer 2F pinned to the values it **renders** at (§1.3 — `7px` card, not `4px`); dark themes define `--wk-rim-light` and a real `surface-2`→`surface-1` lightness step |
| `tests/themeCategories.test.cjs` *(update)* | unit, pure | Category union == `ALLOWED_BOARD_COLORS`; **no category name is also a theme name** (§4.1) |
| `tests/globalThemeColor.test.cjs` *(extend)* | unit | Class application + legacy mutual exclusion |
| `tests/buttonThemeColors.test.cjs` *(update)* | unit | `--theme-accent` fallbacks still themed |
| `tests/themeColorPicker.test.cjs` *(extend)* | unit | `token` category renders, no wheels |
| `tests/themeSelector.test.cjs` *(new)* | unit, pure | Theme blocks are written `:root.wk-theme-<name>`, so precedence never depends on `styles.js` import order (§3.2) |
| `tests/visual/*` | Playwright | 8 screens × 3 themes × 2 densities, **plus all 19 legacy board colors for Phase 2a**, no unintended diff |
| Negative tests | unit | Unknown theme name rejected; `wk-theme-` never built from unvalidated input; legacy + token never co-active |

### 8.2 Existing tests this work must not break

The first draft listed only the theme tests it invents. There are **220 tests** in `tests/`,
and Phase 2 (a/b/c) rewrites all 71 stylesheets while Phase 6 changes layout and markup. These nine
are hard gates on every phase from 1a onward, and several of them encode decisions the
maintainer made *deliberately* — a plan that trips them is wrong, not the test:

| Test | What it forbids / requires | Bites in |
|---|---|---|
| `tests/noViewportSpacing.test.cjs` | Scans **every** stylesheet. No `vh`/`vw` for size or spacing, and — since the July 2026 fix — **no `clamp()` and no viewport `min-*`**. Allowed: full-viewport boxes, `max-*` caps, shrink-only `min(400px, 52vw)`. | 1a, 2b, 2c, 6 |
| `tests/scrollbarCss.test.cjs` | List bodies keep always-visible scrollbars (#5439) across all engines. | 2c, 6 |
| `tests/sidebarWidth.test.cjs` | Sidebar is a fixed px width with **no viewport unit at all**. | 2c, 6 |
| `tests/listWidthDefaults.test.cjs` | Per-list width defaults; interacts with the `--list-width` inline style from `list.jade:3`. | 6 (gutters) |
| `tests/mobileBoardFit.test.cjs` | The board fits the viewport width in mobile mode. | 6 (gutters) |
| `tests/mobileModeFullWidth.test.cjs` | Mobile-mode full-width layout. | 6 |
| `tests/mobileModeConsistency.test.cjs` | Same bar on phone as desktop; list title present. | 5.3 item 3, 6 |
| `tests/mobileAddList.test.cjs` | Add List lives in the list header, not a row of its own. | 6 |
| `tests/oauth2LoginStyle.test.cjs` | Login page styling. | 2b, 2c |

**Note the timing.** The ten commits immediately preceding this plan are all mobile-layout
stabilisation — `clamp()` removal, board-width fit, centred titles, the Add List move. Phase 6
lands per-column gutters directly on top of that freshly-settled code. That is the sequencing
risk in §9, not a footnote.

Run per `build.sh` / `docs/Security/Sandboxes/vscode/README.md`; logs land in
`../log/<datetime>/` (see `CLAUDE.md`, "Check newest test logs").

---

## 9. Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| **`boardColors.css` outranks the tokenized rules, so the migration changes nothing on screen** | **Was certain** — the first draft excluded the file | Phase 2a regenerates it *before* the component migration; Phase 2b's exit metric is measured tree-wide, not "outside `boardColors.css`" (§1.3) |
| **Phase 2a regression across all 19 legacy themes** | High | Per-theme screenshots for all 19 as a Phase 2a gate; pure refactor — names and appearance unchanged; revertable as one commit |
| Classic drifts during Phase 2b | Medium | Per-commit screenshot diff; token values seeded from the literals they replace |
| Classic drifts during Phase 2c | **Expected, by design** | 2c snaps 32/54/70 values onto 4/7/5 scales; diffs are reviewed and approved per commit, not asserted to be zero |
| 2 200 `!important` defeat a theme | Low | Tokens are consumed *inside* the existing `!important` declarations, so the value changes without touching specificity. **This was never the real cascade problem** — selector specificity in the excluded `boardColors.css` was (§1.3) |
| Legacy `board-color-*` half-overrides a token theme | Medium | Mutual exclusion in the runtime + template guards + a negative test |
| `color-mix()` unsupported on an old target | Low | Banned inside custom-property definitions, where the fallback silently does not work; permitted in component longhands with a static fallback above (§3.4) |
| Theme precedence flips on a `styles.js` refactor | Low | `:root.wk-theme-<name>` is (0,2,0) and beats `:root` unconditionally; asserted by `themeSelector.test.cjs` (§3.2) |
| **Phase 6 gutters break the just-landed mobile layout work** | **High** | Gutters ship first within Phase 6 so a revert is cheap; verified at 1280px, mobile, both densities, and with a fixed per-list width; six existing tests gate it (§8.2) |
| Header re-zoning breaks the button-wrap fix | Medium | The wrap behaviour and the mobile hamburger position are recorded in `boardHeader.jade` comments and were bug fixes; if zoning cannot preserve them, drop the zoning (§5.3) |
| Nebula backdrop hurts drag/scroll performance | Medium | Fixed single layer, no `background-attachment: fixed`, no animation by default, perf budget in the theme doc, user toggle |
| Token churn breaks `buttonThemeColors.test.cjs` | High (by design) | That test asserts literal fallback strings — update it in the same commit as `forms.css` |
| Scope creep into Blaze/behavioural work | Medium | Revised §2 non-goals: presentational containers only, and each Phase 6 commit states what it did **not** touch |
| Changing the default typeface is unwelcome | Low | Phase 1a is 15 lines and independently revertable; `--wekan-ui-font` still overrides it for anyone who has set a preference |

---

## 10. Open decisions for the maintainer

1. **Should `token` themes accept a custom accent?** Plan assumes no in Phases 4–5, yes in
   Phase 7. Enabling it earlier is a one-line `CUSTOM_COLOR_COUNT` change but needs
   contrast-safe on-accent derivation to stay AA.
2. **Should Nebula's decorative backdrop default on or off?** Plan: **on**, static, with a
   Member Settings toggle. Alternative: off by default.
3. **Auto (`prefers-color-scheme`) mode** — worth a picker entry, or leave to Phase 7?
4. **A token theme flattens per-board colors — is that acceptable?** (§6.2.) Because every
   board always has a color, mutual exclusion means picking Nebula globally makes every board
   look the same; the per-board color signal is gone. Options: accept it and say so in the
   picker; or let a token theme keep a per-board accent tint. Plan assumes accept-and-say-so.
5. **Is Phase 2a's "identical appearance" bar right for all 19 legacy themes?** Regenerating
   `boardColors.css` is now a prerequisite (§1.3), not optional. The plan holds all 19 to
   pixel-identical. If a few of the 19 are considered expendable, 2a gets substantially
   cheaper — but that is a product call, not an engineering one.
6. **Ship a bundled typeface, or stay on the system stack?** (§5.0.) Phase 1a uses a system
   stack and adds no asset. `fonts.css` already contains commented-out Roboto and Poppins
   `@font-face` blocks and the `.woff2` files may still be in `public/`; enabling them is a
   payload, licensing and CLS decision. Recommendation: system stack now, revisit later.
7. **Does the always-visible list scrollbar (#5439) survive the redesign visually?** It is
   deliberate and test-guarded (`scrollbarCss`), but a permanent scrollbar on every column
   fights the "floating rounded panels" direction of both new themes. Options: keep as-is;
   style it per theme via `scrollbar-color` / `::-webkit-scrollbar` (Nebula already proposes
   this); or make always-visible a preference. Plan assumes keep-and-style.

---

## 11. CHANGELOG

This document is a plan; no user-visible behaviour changes with it, so it adds no
`# Upcoming WeKan ® release` entry. Each implementation phase adds its own entry per the
CHANGELOG rules in `CLAUDE.md`, with the one-flowing-sentence subsection headers
(`This release …:` then `and …:`):

| Phase | CHANGELOG section |
|---|---|
| 0 | developer tooling |
| **1a** (typeface) | *new features* — it is user-visible on every screen |
| 1b (tokens) | developer tooling |
| **2a** (`boardColors.css`) | developer tooling, and *bug fixes* for anything it corrects in the 19 legacy themes |
| 2b (color migration) | developer tooling |
| **2c** (scale collapse) | *new features* — it moves Classic |
| 3 (runtime) | developer tooling |
| **4–5** (Nebula, Meridian) | *new features* |
| **6** (structural + markup) | *new features* |
| 7 (cleanup) | *bug fixes* / developer tooling |

Phases that move Classic — 1a, 2c and 6 — say so plainly in their entry rather than being
filed as refactors.

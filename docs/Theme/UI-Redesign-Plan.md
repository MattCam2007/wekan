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

---

## 1. Audit — the measured state of the CSS

All numbers below were measured on the current tree, not estimated.

| Metric | Value | Why it matters |
|---|---|---|
| CSS files (`client/components/**`) | 69 files, **23 890 lines** | All hand-written, no preprocessor |
| Hardcoded color literals (hex + `rgb(a)`) | **2 736** | Every one is a value a theme cannot reach |
| Distinct hex colors | **545** | A 19-theme app needs ~40, not 545 |
| `var(--…)` usages | **68** | Effectively no token layer exists |
| `!important` declarations | **2 200** | Specificity war; overriding needs escalation |
| `boardColors.css` | **4 327 lines** for 19 themes | ~228 lines of duplicated selectors per theme |
| Distinct `font-size` values | ~20 (`16px`,`14px`,`12px`,`18px`,`13px`,`11px`,`1em`,`0.9em`,…) | No type scale |
| Distinct `border-radius` values | 13 (`2,3,4,5,6,7,8,10,12,16px`,`50%`,…) | Corners disagree between adjacent components |
| `box-shadow` variants | 12+ ad-hoc | No elevation model |

Worst offenders by `!important` count: `boardHeader.css` (508), `list.css` (395),
`header.css` (309), `boardColors.css` (201), `popup.css` (145), `cardDetails.css` (137).

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
can only reach the ~68 `var()` sites and whatever it re-declares with higher specificity.
That is why `boardColors.css` is 4 327 lines and why the app has no real dark mode.

### 1.2 Build constraints

`rspack.config.js` uses `['style-loader', 'css-loader']` with no PostCSS, no autoprefixer,
and no `browserslist`. So:

- Native CSS custom properties are the only viable token mechanism (they are resolved by the
  browser at runtime — exactly what theme switching needs). **No build changes required.**
- No nesting, no `@custom-media`, no build-time color functions. Any derived color must be
  written literally or computed with `color-mix()` at runtime (see §3.4).

---

## 2. Goals and non-goals

### Goals

1. **A design-token layer** so a theme is a list of ~120 values, not 228 lines of selectors.
2. **The original theme is preserved exactly** — pixel-identical, enforced by visual
   regression, not by eyeball.
3. **Two new themes**, both token-only:
   - **Nebula** — dark, derived from the supplied deep-space nebula image.
   - **Meridian** — light, calm, high-legibility; the reference implementation of the
     redesigned form.
4. **Fix the form**: one type scale, one spacing scale, one radius scale, one elevation
   model, one focus-ring system, consistent controls.
5. **Accessibility**: every foreground/background pair in every shipped theme meets WCAG 2.1
   AA (4.5:1 body, 3:1 large text and UI boundaries), verified by an automated test.
6. **All 19 existing board colors keep working** unchanged throughout.

### Non-goals (explicitly out of scope)

- No markup/template restructuring beyond what a class or token rename requires.
- No CSS framework (Tailwind/Bootstrap), no preprocessor, no build-pipeline change.
- No component-library migration, no Blaze→React work.
- Not deleting the 19 legacy board colors. They stay; they are simply not token-driven.
- Not removing all 2 200 `!important` in one pass (see Phase 7 — it is a follow-up, and the
  plan is deliberately designed so it is *not* a prerequisite).

---

## 3. Architecture — three token layers

```
Layer 1  PRIMITIVES   --wk-palette-*    raw colors, no meaning     (per theme)
              │
Layer 2  SEMANTIC     --wk-surface-*    meaning, no component      (per theme)
              │       --wk-text-*  --wk-border-*  --wk-accent-*  --wk-state-*
              │
Layer 3  COMPONENT    --wk-card-bg      component-scoped aliases   (theme-agnostic)
              │       --wk-list-header-bg  --wk-popup-bg  …
              ▼
         component CSS consumes ONLY layer 2 + layer 3
```

**The rule that makes this work: component CSS never names a color.** It references a
semantic or component token. A theme then only redefines layers 1–2.

Non-color scales (space, radius, type, elevation, motion, z-index) live in a single
theme-agnostic file and are **not** overridden per theme — except `--wk-radius-*` and
`--wk-density-*`, which themes may re-map to express their character (see §5.2).

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
`.wk-theme-<name>` class placed on `<html>` by the runtime (§6). Because a class selector
beats `:root` at equal specificity ordering *and* custom properties inherit, every descendant
picks up the override with no `!important` anywhere.

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
Phase 2 rewrites these to `var(--theme-accent, var(--wk-accent-strong))` **and updates
`tests/buttonThemeColors.test.cjs` in the same commit** — that test matches the literal
string and will otherwise fail. Same for `--list-width`, `--swimlane-height`,
`--wekan-ui-*` (#4759): keep the names, source the defaults from tokens.

### 3.4 Derived colors

No build-time color math is available. Two allowed techniques:

- **Literal derivation** — write the computed value into the theme file with a comment
  giving its origin (`/* accent -12% L */`). Preferred for the ~8 accent ramp steps.
- **`color-mix()`** — for hover/pressed/disabled states and translucent overlays:
  `background: color-mix(in oklab, var(--wk-accent) 88%, black);`
  Baseline support is fine for WeKan's targets (Chrome/Edge 111+, Firefox 113+, Safari 16.2+).
  **Every `color-mix()` must carry a static fallback declaration immediately above it** so
  older browsers degrade to a solid color rather than to `unset`:

  ```css
  .btn:hover { background: #2471a3; }
  .btn:hover { background: color-mix(in oklab, var(--wk-accent) 88%, black); }
  ```

---

## 4. Theme roster

| Theme | Category | Class | Basis | Custom colors |
|---|---|---|---|---|
| **Classic** (original) | default | *(none — `:root`)* | Today's exact appearance | via `--theme-accent` as today |
| **Nebula** | `modern` | `.wk-theme-nebula` | The supplied nebula image | 0 initially, 1 in Phase 7 |
| **Meridian** | `modern` | `.wk-theme-meridian` | Original light design | 0 initially, 1 in Phase 7 |

**Classic is the default and is not a picker entry** — it is what `:root` yields when no theme
class is present, which is exactly today's behaviour. This is how "keep the original theme"
is satisfied strictly: not by recreating it, but by *extracting its current values into
tokens* and proving with screenshots that nothing moved.

### 4.1 Registering the two new themes

Add a **fifth category** rather than extending `dark` / `special`. The existing tests assert
the exact membership of those arrays (`colorsInCategory('dark')` is a `deepStrictEqual`
against a 5-name list), and a new category keeps the story clear: *`modern` = token-driven,
fully themed, contrast-verified.*

```js
// models/lib/themeCategories.js
const THEME_CATEGORIES = {
  flat:    [...],
  clear:   ['clearblue'],
  dark:    [...],
  special: [...],
  modern:  ['nebula', 'meridian'],          // NEW — token-driven themes
};
const THEME_CATEGORY_ORDER = ['flat', 'clear', 'dark', 'special', 'modern'];
const CUSTOM_COLOR_COUNT = { flat: 1, clear: 2 };   // modern: 0 until Phase 7
```

Files that must change together (all guarded by existing tests):

1. `config/const.js` — append `'nebula'`, `'meridian'` to `ALLOWED_BOARD_COLORS`.
   `models/metadata/colors.js` derives `BOARD_COLORS` from it automatically.
2. `models/lib/themeCategories.js` — as above.
3. `tests/themeCategories.test.cjs` — update `THEME_CATEGORY_ORDER` and the union assertion.
4. `imports/i18n/data/en.i18n.json` — display names + descriptions (`theme-nebula`,
   `theme-meridian`, `theme-category-modern`). English only; Transifex supplies the rest per
   the translation policy in `CLAUDE.md` — **do not hand-fill other languages**.

Because they are registered as ordinary theme names, they inherit *for free*: the picker UI,
server-side validation, the global per-user override (#5778), per-board selection, and
persistence. No new schema, no new publication, no new method.

---

## 5. Fixing the form

Themes only change color. The "horrific form" complaint is about rhythm, hierarchy and
consistency — addressed by the scales below, which apply to **all** themes including Classic
(where they are seeded with Classic's *current* values, so Classic does not move).

### 5.1 Scales

- **Space** — 4px base: `--wk-space-1:4px` … `--wk-space-8:48px`. Replaces the current ad-hoc
  `3px/5px/6px/8px/10px/12px/14px/21px/23px` mix.
- **Type** — 7 steps: `12 / 13 / 14 / 16 / 18 / 20 / 24px` mapped to
  `--wk-text-xs … --wk-text-3xl`, with `--wk-leading-tight|normal|relaxed`. The existing ~20
  sizes collapse into these.
- **Radius** — 4 steps: `--wk-radius-sm:4px`, `-md:6px`, `-lg:10px`, `-full:999px`. The 13
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

### 5.3 Component work (Phase 6)

Ordered by user-visible impact:

1. **Minicard** (`minicard.css`, 979 lines) — align the badge row on one baseline, cap label
   chips with a consistent height/radius, single truncation strategy for titles, one
   elevation for rest / one for drag. Today: `border-radius:4px` on the card but `17px` on
   the drag placeholder and `6px` on the "and N other" chip.
2. **List column** (`list.css`, 1 596 lines, 395 `!important`) — consistent header height,
   one padding rhythm, sticky header that does not shift on scroll.
3. **Board header** (`boardHeader.css`, 1 057 lines, 508 `!important`) — the single worst
   file; unify button sizes into one `.board-header-btn` spec with real focus states.
4. **Popups** (`popup.css`) — one width scale, one padding, consistent header/footer.
5. **Forms** (`forms.css`) — one input height, one border/focus treatment, real
   `:focus-visible` rings replacing the current mixed outline/box-shadow approaches.
6. **Sidebar**, **card details**, **admin/settings tables** — apply the same scales.

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
const TOKEN_THEMES = ['nebula', 'meridian'];   // import from themeCategories: colorsInCategory('modern')

function applyThemeClass(name) {
  const root = document.documentElement;
  TOKEN_THEMES.forEach(t => root.classList.remove(`wk-theme-${t}`));
  if (name && TOKEN_THEMES.includes(name)) root.classList.add(`wk-theme-${name}`);
}
```

### 6.2 Mutual exclusion with legacy board colors

In the same autorun, when the resolved theme is a `modern` one, **do not** add the
`board-color-<name>` class to `<body>`; when it is legacy, **do not** add `wk-theme-*`.
`header.jade` / `boardBody.jade` must likewise emit no `board-color-*` class for a modern
theme — add a `currentBoard.isTokenTheme` / `currentUser.isTokenTheme` helper and guard the
existing `class="…"` expressions. Cover this with a unit test extending
`tests/globalThemeColor.test.cjs`.

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

### Phase 1 — Token layer + Classic extraction

- Author `_tokens.css` and `theme-classic.css`, seeding every value from the colors currently
  hardcoded in component CSS.
- Change **no** component file yet.

**Exit:** tokens defined and unused; zero screenshot diff by construction.

### Phase 2 — Migrate component CSS to tokens

One commit per file, largest-impact first, using the measured line counts:
`forms.css` → `minicard.css` → `list.css` → `header.css` → `boardHeader.css` →
`cardDetails.css` → `popup.css` → `sidebar.css` → `boardBody.css` → `swimlanes.css` →
`boardsList.css` → `layouts.css` → the remaining ~50 smaller files.

For each file: replace every literal with the semantic token whose Classic value is
byte-identical. **Do not** change specificity, do not remove `!important`, do not adjust any
non-color property. This keeps each commit mechanically reviewable and screenshot-clean.

**Exit:** hardcoded literals outside `boardColors.css` and the theme files drop from ~1 833 to
<150 (permitted residue: pure-decorative gradients, spinner keyframes, `#fff`/`#000` in
shadow definitions). Zero screenshot diff per commit.

### Phase 3 — Theme runtime

`config/const.js`, `themeCategories.js`, the `modern` category, `globalThemeColor.js` class
application, template guards, i18n keys, updated tests.

**Exit:** picker shows a `modern` category; selecting an entry toggles `wk-theme-*` on
`<html>` and suppresses `board-color-*`. No theme CSS yet, so both render as Classic.

### Phase 4 — Nebula

Implement `theme-nebula.css` + `theme-nebula-decor.css` per `docs/Theme/Theme-Nebula.md`.

**Exit:** contrast test green; new Nebula baselines; decor within the perf budget.

### Phase 5 — Meridian

Implement `theme-meridian.css` per `docs/Theme/Theme-Meridian.md`.

**Exit:** contrast test green; Meridian baselines.

### Phase 6 — Form redesign

Apply §5.3 component work. **This is the only phase that intentionally changes Classic's
appearance**, so it needs explicit maintainer sign-off and per-component baseline updates.

**Exit:** one type/space/radius/elevation scale in use; focus rings consistent; baselines
re-approved.

### Phase 7 — Cleanup

Reduce `!important` now that tokens make overrides unnecessary (target: <500). Enable one
custom accent for `modern` themes with contrast-safe on-accent derivation. Optionally
regenerate `boardColors.css` from tokens, collapsing 4 327 lines toward ~600.

---

## 8. Testing

| Test | Type | Guards |
|---|---|---|
| `tests/themeTokens.test.cjs` | unit, pure | Every theme defines every token |
| `tests/themeContrast.test.cjs` | unit, pure | WCAG AA for all declared pairs |
| `tests/themeCategories.test.cjs` *(update)* | unit, pure | Category union == `ALLOWED_BOARD_COLORS` |
| `tests/globalThemeColor.test.cjs` *(extend)* | unit | Class application + legacy mutual exclusion |
| `tests/buttonThemeColors.test.cjs` *(update)* | unit | `--theme-accent` fallbacks still themed |
| `tests/themeColorPicker.test.cjs` *(extend)* | unit | `modern` category renders, no wheels |
| `tests/visual/*` | Playwright | 8 screens × 3 themes, no unintended diff |
| Negative tests | unit | Unknown theme name rejected; `wk-theme-` never built from unvalidated input; legacy + modern never co-active |

Run per `build.sh` / `docs/Security/Sandboxes/vscode/README.md`; logs land in
`../log/<datetime>/` (see `CLAUDE.md`, "Check newest test logs").

---

## 9. Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Classic drifts during Phase 2 | High | Per-commit screenshot diff; token values seeded from the literals they replace |
| 2 200 `!important` defeat a theme | Medium | Tokens are consumed *inside* the existing `!important` declarations, so the value changes without touching specificity |
| Legacy `board-color-*` half-overrides a token theme | Medium | Mutual exclusion in the runtime + template guards + a negative test |
| `color-mix()` unsupported on an old target | Low | Static fallback declaration required immediately above every use |
| Nebula backdrop hurts drag/scroll performance | Medium | Fixed single layer, no `background-attachment: fixed`, no animation by default, perf budget in the theme doc, user toggle |
| Token churn breaks `buttonThemeColors.test.cjs` | High (by design) | That test asserts literal fallback strings — update it in the same commit as `forms.css` |
| Scope creep into markup/Blaze work | Medium | Non-goals in §2; Phase 2 forbids non-color edits |

---

## 10. Open decisions for the maintainer

1. **Should `modern` themes accept a custom accent?** Plan assumes no in Phases 4–5, yes in
   Phase 7. Enabling it earlier is a one-line `CUSTOM_COLOR_COUNT` change but needs
   contrast-safe on-accent derivation to stay AA.
2. **Should Nebula's decorative backdrop default on or off?** Plan: **on**, static, with a
   Member Settings toggle. Alternative: off by default.
3. **Auto (`prefers-color-scheme`) mode** — worth a picker entry, or leave to Phase 7?
4. **Does a global theme override still win on a board?** Current #5778 behaviour says yes.
   Token themes make a per-board opt-out cheap; is it wanted?
5. **Regenerate `boardColors.css` from tokens in Phase 7?** Large diff, large payoff
   (4 327 → ~600 lines), but touches all 19 legacy themes at once.

---

## 11. CHANGELOG

This document is a plan; no user-visible behaviour changes with it, so it adds no
`# Upcoming WeKan ® release` entry. Each implementation phase adds its own entry per the
CHANGELOG rules in `CLAUDE.md` — Phases 4–6 under *new features*, Phase 7 under *bug fixes* /
developer tooling, with the one-flowing-sentence subsection headers (`This release …:` then
`and …:`).

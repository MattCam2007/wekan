# Theme docs index

## The UI redesign + multi-theme plan (not yet implemented)

Read in this order:

1. **[UI-Redesign-Plan.md](UI-Redesign-Plan.md)** — the master plan. Measured audit of the
   current CSS, the three-layer token architecture, the theme roster, how it plugs into the
   existing theme runtime, the form redesign, 8 implementation phases with exit criteria,
   testing, risks, and the open decisions for the maintainer.
2. **[Form-Redesign.md](Form-Redesign.md)** — **why the UI reads as square, boxy and flat**,
   measured from the current CSS, and the geometry that fixes it: shape-by-role tokens, the
   dark-theme separation problem (black shadows are ~10× less visible on dark), density, and
   a per-component before/after spec.
3. **[Design-Tokens.md](Design-Tokens.md)** — the contract between component CSS and themes:
   every token name across layers 1, 2, 2F (form) and 3, the scales, and the authoring rules.
4. **[Theme-Nebula.md](Theme-Nebula.md)** — the dark theme derived from a deep-space nebula
   image. Full palette, verified contrast table, form character (floating rounded panels, rim
   light, pill chips), decorative backdrop spec and perf budget.
5. **[Theme-Meridian.md](Theme-Meridian.md)** — the light theme; the reference implementation
   of the redesigned form. Full palette, verified contrast table, form character (crisp
   panels, soft shadows, comfortable density).

Themes carry **both color and form**. A theme that only recolors would leave the boxy, flat
layout intact — see the revision note in `Form-Redesign.md` §2.

The original theme is preserved exactly and is the default — see `UI-Redesign-Plan.md` §4.

## Existing theme docs

- **[Theme.md](Theme.md)** — the shipped "Select Color" theme categories (`flat` / `clear` /
  `dark` / `special`), custom colors, and the picker UX. The redesign adds a fifth
  category, `modern`, on top of this.
- **[Custom-CSS-themes.md](Custom-CSS-themes.md)** — writing your own CSS theme.
- **[Dark-Mode.md](Dark-Mode.md)** — the current dark-mode notes.
- **[Converting-Meteor-Stylus-to-CSS.md](Converting-Meteor-Stylus-to-CSS.md)** — history of
  the Stylus → plain CSS migration.

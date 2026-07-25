# Theme docs index

## The UI redesign + multi-theme plan (not yet implemented)

Read in this order:

1. **[UI-Redesign-Plan.md](UI-Redesign-Plan.md)** — the master plan. Measured audit of the
   current CSS, the three-layer token architecture, the theme roster, how it plugs into the
   existing theme runtime, the form redesign, 8 implementation phases with exit criteria,
   testing, risks, and the open decisions for the maintainer.
2. **[Design-Tokens.md](Design-Tokens.md)** — the contract between component CSS and themes:
   every token name, the space/type/radius/elevation/motion scales, and the authoring rules.
3. **[Theme-Nebula.md](Theme-Nebula.md)** — the dark theme derived from a deep-space nebula
   image. Full palette, verified contrast table, decorative backdrop spec and perf budget.
4. **[Theme-Meridian.md](Theme-Meridian.md)** — the light theme; the reference implementation
   of the redesigned form. Full palette and verified contrast table.

The original theme is preserved exactly and is the default — see `UI-Redesign-Plan.md` §4.

## Existing theme docs

- **[Theme.md](Theme.md)** — the shipped "Select Color" theme categories (`flat` / `clear` /
  `dark` / `special`), custom colors, and the picker UX. The redesign adds a fifth
  category, `modern`, on top of this.
- **[Custom-CSS-themes.md](Custom-CSS-themes.md)** — writing your own CSS theme.
- **[Dark-Mode.md](Dark-Mode.md)** — the current dark-mode notes.
- **[Converting-Meteor-Stylus-to-CSS.md](Converting-Meteor-Stylus-to-CSS.md)** — history of
  the Stylus → plain CSS migration.

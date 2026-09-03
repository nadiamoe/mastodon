# Styles system changes: `4-6-7` to `4-7-1`

Summary of what changed in the Mastodon **styles system** (upstream) between the `4-6-7` and `4-7-1`
branches. Purpose: use this as context when reapplying our custom styles (the espresso/latte theme
overlay, see `espresso-theme-reconstruction.md`) on top of the newer base.

None of these are our commits. They're upstream Mastodon changes that shift the plumbing our overlay
hooks into, so the overlay has to be adapted to match.

## TL;DR of what matters for the overlay

- The token layer moved: `mastodon/theme/` is now `mastodon/tokens/theme/`, and there's a new
  `mastodon/tokens` aggregator. Any `@use` pointing at `mastodon/theme` must become `mastodon/tokens`.
- More SCSS variables were deleted and replaced by CSS custom properties. Specifically the font family
  vars (`$font-sans-serif`, `$font-display`, `$font-monospace`) and the media-modal size vars are
  gone. If the overlay set or read those SCSS vars, switch to the CSS custom properties instead.
- New CSS custom properties exist to target: `--color-bg-blend`, `--color-bg-highlight`,
  `--color-border-strong`, plus a full set of spacing/radius/type tokens and `--font-body` /
  `--font-heading` / `--font-monospace`.
- `--color-bg-overlay-base` was removed. If the overlay referenced it, retarget.
- Popover library swap: `data-popper-*` attributes became `data-popover-*`. Any selector we wrote
  against `[data-popper-placement]` etc. must be renamed.

## 1. Directory restructure: `theme/` becomes `tokens/`

`app/javascript/styles/mastodon/theme/` was moved under a new `tokens/` module:

- `mastodon/theme/index.scss`  -> `mastodon/tokens/index.scss`
- `mastodon/theme/_base.scss`  -> `mastodon/tokens/theme/_base.scss` (unchanged content)
- `mastodon/theme/_dark.scss`  -> `mastodon/tokens/theme/_dark.scss`
- `mastodon/theme/_light.scss` -> `mastodon/tokens/theme/_light.scss`
- `mastodon/theme/_utils.scss` -> `mastodon/tokens/theme/_utils.scss` (unchanged content)

New files added under the module:

- `mastodon/tokens/_shape.scss` (spacing + radius tokens)
- `mastodon/tokens/_type.scss` (font-size, line-height, letter-spacing tokens)

`application.scss` changed its import accordingly:

```diff
-@use 'mastodon/theme';
+@use 'mastodon/tokens';
```

and now also pulls in two font files:

```diff
+@use 'fonts/inter';
+@use 'fonts/syne';
```

`tokens/index.scss` now applies the new token mixins on `html`:

```scss
@use 'theme/base';
@use 'theme/dark';
@use 'theme/light';
@use 'theme/utils';
@use 'shape';
@use 'type';

html {
  @include base.palette;
  @include shape.space;
  @include shape.radius;
  @include type.font-size;
  @include type.line-height;
  @include type.letter-spacing;
}
```

## 2. SCSS variables removed from `mastodon/_variables.scss`

Deleted (10 lines):

```scss
$media-modal-media-max-width: 100%;
$media-modal-media-max-height: 80%;
$font-sans-serif: 'mastodon-font-sans-serif' !default;
$font-display: 'mastodon-font-display' !default;
$font-monospace: 'mastodon-font-monospace' !default;
```

What remains in that file is just language lists and breakpoints. These deletions are the ones most
likely to break an overlay that still spoke the old SCSS-variable dialect.

Replacements:

- Font families are now CSS custom properties defined in `basics.scss` on `html`:

  ```scss
  --font-body: mastodon-font-sans-serif, sans-serif;
  --font-heading: Syne, sans-serif;
  --font-monospace: mastodon-font-monospace, monospace;

  &[data-redesign='true'] {
    --font-body: Inter, sans-serif;
  }
  ```

  So `body` uses `var(--font-body)`, `code`/`samp`/CSS textareas use `var(--font-monospace)`, and the
  `character-counter` uses `var(--font-body)`. There's a `[data-redesign='true']` toggle on `html`
  that swaps the body font to Inter, for the in-progress 5.0 redesign.

- The media-modal size vars became local CSS custom properties scoped to `.zoomable-image`:

  ```scss
  .zoomable-image {
    --media-max-width: 100%;
    --media-max-height: 80%;
    // img and &__preview use var(--media-max-width) / var(--media-max-height)
  }
  ```

## 3. New color tokens (dark and light)

Added to both `tokens/theme/_dark.scss` and `_light.scss`:

- `--color-bg-blend`
  - dark: `css-alpha(var(--color-black), 30%)`
  - light: `css-alpha(var(--color-text-primary), 4%)`
- `--color-bg-highlight`
  - dark: `css-alpha(var(--color-text-primary), 5%)`
  - light: `css-alpha(var(--color-text-primary), 4%)`
- `--color-border-strong`
  - dark: `var(--color-grey-100)`
  - light: `var(--color-grey-950)`

Changed / removed:

- `--color-bg-overlay-base` was **removed**.
- `--color-bg-overlay-highlight` now just aliases the new `--color-bg-highlight` (marked legacy) in
  both themes.

The base grey/indigo/etc. palette (`tokens/theme/_base.scss`) is unchanged.

(These three new tokens come from upstream PR #39786 "Add new theme tokens `bg-blend`,
`bg-highlight`, and `border-strong`".)

## 4. New shape and type tokens

`tokens/_shape.scss` adds spacing and radius scales as CSS custom properties:

- Spacing: `--space-3xs` (2px) through `--space-5xl` (40px).
- Radius: `--radius-xs` (8px), `-sm` (12px), `-md` (16px), `-lg` (20px), `-xl` (28px),
  `--radius-round: calc(1px * infinity)`.

`tokens/_type.scss` adds:

- Font sizes `--fs-3xs` (11px) through `--fs-3xl` (26px).
- Line heights `--lh-tighter` (1.3), `--lh-tight` (1.4), `--lh-normal` (1.6).
- Letter spacing `--ls-tighter` (-0.02em) through `--ls-looser` (0.02em).

## 5. New mixins in `mastodon/_mixins.scss`

110 lines prepended (the existing `search-input` mixin is untouched, just pushed down):

- Type styles: `type-heading-lg/md/sm`, `type-body-lg/strong/compact`, `type-body`,
  `type-label-lg/md/sm`, `type-micro`. Each sets `font-family` (`var(--font-heading)` /
  `var(--font-body)` / `var(--font-monospace)`), `font-size`, `font-weight`, `line-height`,
  `letter-spacing` from the new tokens.
- Elevation styles: `elevation-1`, `elevation-2`. Both use `var(--color-border-strong)` for border
  and box-shadow (1px vs 2px offset).

## 6. Fonts

- **Inter** (`fonts/inter.scss`): switched from a single variable font
  (`inter-variable-font-slnt-wght.woff2`, weight `100 900`) to four static faces: regular, italic,
  semibold and semibold-italic. Note the quirk: the native 600 semibold is mapped to `font-weight:
  700` so that `font-weight: bold` picks it up.
- **Syne** (`fonts/syne.scss`, new): a single bold (700) face, used as `--font-heading`.
- `fonts/inter-mailer.scss` also gained faces (mailer variant).

(These are from upstream PRs #39868 "Add webfonts for 5.0 redesign" and #39848 "Add design tokens for
5.0 redesign".)

## 7. Popover library swap (react-overlays -> Floating UI)

Upstream PR #39621 replaced react-overlays with Floating UI. In the styles this renamed data
attributes:

- `body > [data-popper-placement]`  -> `body > [data-popover-placement]`
- `.hover-card-controller[data-popper-reference-hidden='true']` -> `...[data-popover-reference-hidden='true']`
- `.visibility-dropdown__overlay[data-popper-placement]` rule was removed.

If our overlay styles any dropdown/popover/hover-card via `data-popper-*`, rename to `data-popover-*`.

## 8. Minor

- `tables.scss`: `.critical` and a new `.unsupported` class are now nested selectors (used by the
  "end-of-support date" feature, PR #39732), colored with `--color-text-warning` /
  `--color-text-error`.

## Reapplication checklist for the overlay

When moving the theme overlay onto the `4-7-1` base:

1. Repoint any `@use '.../mastodon/theme'` to `.../mastodon/tokens`.
2. Stop relying on `$font-sans-serif` / `$font-display` / `$font-monospace`; set/override
   `--font-body`, `--font-heading`, `--font-monospace` instead (on `html` or `:root`).
3. Stop relying on `$media-modal-media-max-width/height`; override `--media-max-width` /
   `--media-max-height` on `.zoomable-image` if needed.
4. If the overlay touched `--color-bg-overlay-base`, switch to `--color-bg-highlight` /
   `--color-bg-blend`.
5. Rename any `data-popper-*` selectors to `data-popover-*`.
6. Consider whether the espresso palette wants values for the new `--color-bg-blend`,
   `--color-bg-highlight`, `--color-border-strong` tokens (otherwise they inherit the default
   dark/light values, which are grey, not brown).
7. Decide what to do about `--font-heading: Syne` and the `[data-redesign='true']` Inter toggle;
   the espresso theme may want to keep its own display font rather than Syne.

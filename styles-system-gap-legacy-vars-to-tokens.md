# The bigger gap: legacy `$ui-*` SCSS variables to CSS tokens

This is the migration that sits *underneath* the 4.6.7-to-4.7.1 changes (see
`styles-system-changes-4.6.7-to-4.7.1.md`). Our espresso/latte overlay (see
`espresso-theme-reconstruction.md`) was written against the **old** SCSS-variable system, which no
longer exists in 4.6/4.7. Reapplying the overlay is not a version bump, it's a port across two
different theming architectures. This documents that gap.

## Timeline (verified against upstream stable branches)

| Version | Theming system | `_functions.scss` | `css_variables.scss` | `$ui-*` vars | token dir |
|---------|----------------|-------------------|----------------------|--------------|-----------|
| 4.4     | legacy SCSS vars | yes | yes | yes | none |
| 4.5     | legacy SCSS vars | yes | yes | yes | none |
| 4.6     | CSS tokens | no | no | **gone** | `mastodon/theme/` |
| 4.7     | CSS tokens | no | no | gone | `mastodon/tokens/` |

- The espresso overlay targets **4.4/4.5** (it does `@use '../mastodon/variables' with(...)` and
  `@use '../mastodon/functions'`, both of which only exist there).
- The hard break is **4.5 to 4.6**: the token migration removed the `$ui-*` variables, the custom
  `_functions.scss` (`darken`/`lighten`/`hex-color`), and the `css_variables.scss` bridge.
- 4.6 to 4.7 only *moved and extended* the token layer (`theme/` to `tokens/`, added shape/type
  tokens). That's the easy part and is covered in the other doc.

## How theming worked in 4.4/4.5 (what the overlay assumes)

Everything flowed from a set of SCSS variables in `mastodon/_variables.scss`:

- Base palette vars: `$ui-base-color` (darkest bg), `$ui-base-lighter-color`, `$ui-primary-color`,
  `$ui-secondary-color`, `$ui-highlight-color`, the `$ui-button-*` family, text vars
  (`$primary-text-color`, `$darker-text-color`, `$dark-text-color`, `$secondary-text-color`,
  `$passive-text-color`, `$inverted-text-color`, `$light-text-color`, `$lighter-text-color`),
  `$error-red`, `$base-shadow-color`, `$simple-background-color`.
- Defaults were `classic-*` (dark bluish greys) with a `$blurple-500` (`#6364ff`) brand highlight.
- A theme = re-declaring those vars via `@use 'mastodon/variables' with(...)`. Because components
  referenced the SCSS vars directly, overriding them recolored the whole UI. This is exactly what
  `espresso/variables.scss` does.
- `mastodon/_functions.scss` provided `darken()` / `lighten()` (HSL adjust wrappers) and `hex-color()`.
  The overlay calls `darken(...)` / `lighten(...)` freely.
- `mastodon/css_variables.scss` derived a layer of `--*` CSS custom properties **from** those SCSS
  vars (e.g. `--background-color: darken($ui-base-color, 8%)`). The espresso `diff.scss` `:root`
  block is a near-copy of this file, re-derived from the espresso palette. Reference (4.4/4.5):

  ```scss
  --dropdown-border-color: lighten($ui-base-color, 4%);
  --background-color: darken($ui-base-color, 8%);
  --surface-variant-background-color: $ui-base-color;
  --on-surface-color: color.adjust($ui-base-color, $alpha: -0.5);
  --media-outline-color: rgba(#fcf8ff, 0.15);
  --error-background-color: darken($error-red, 16%);
  // ...etc
  ```

## How theming works in 4.6/4.7 (what we must port to)

- No SCSS color vars. The palette is CSS custom properties defined in
  `mastodon/tokens/theme/_base.scss` as neutral ramps: `--color-grey-50..950`,
  `--color-indigo-50..950`, `--color-red-*`, `--color-yellow-*`, `--color-green-*`, plus
  `--color-white` / `--color-black`.
- Semantic tokens in `_dark.scss` / `_light.scss` map those ramps to meaning:
  `--color-bg-primary`, `--color-text-primary`, `--color-text-brand`, `--color-bg-brand-base`,
  `--color-border-primary`, etc. Applied per color scheme via `[data-color-scheme='dark'|'light']`.
- Components reference the `--color-*` tokens. There is no `darken()`/`lighten()`; use
  `color-mix(...)` or the `css-alpha()` util in `tokens/theme/_utils.scss`.
- The default brand color is indigo (`--color-indigo-*`), not blurple SCSS.

## The overlay's `:root` overrides are mostly dead now

This is the important gotcha. Most CSS custom properties the espresso `diff.scss` overrides were
renamed to `--color-*` and are **no longer consumed** in 4.7. Counted consumers of each in the 4.7
stylesheet:

| Custom property overridden in espresso diff.scss | Still consumed in 4.7? | Replacement token |
|--------------------------------------------------|------------------------|-------------------|
| `--dropdown-shadow`            | yes (19) | same name, now derives from `--color-shadow-primary` |
| `--overlay-icon-shadow`        | yes (5)  | same name, derives from `--color-shadow-primary` |
| `--avatar-border-radius`       | yes (9)  | same name, now set in `basics.scss` (`8px`) |
| `--dropdown-border-color`      | **no**   | `--color-border-primary` |
| `--dropdown-background-color`  | **no**   | `--color-bg-secondary` / overlay tokens |
| `--modal-background-color`     | **no**   | `--color-bg-*` overlay tokens |
| `--modal-border-color`         | **no**   | `--color-border-primary` |
| `--background-filter`          | **no**   | `$backdrop-blur-filter` (still SCSS) |
| `--background-color`           | **no**   | `--color-bg-*` |
| `--background-border-color`    | **no**   | `--color-border-primary` |
| `--background-color-raised`    | **no**   | `--color-bg-secondary` |
| `--background-color-tint`      | **no**   | `--color-bg-*` |
| `--surface-background-color`   | **no**   | `--color-bg-primary` |
| `--surface-variant-background-color` | **no** | `--color-bg-secondary` |
| `--surface-variant-active-background-color` | **no** | `--color-bg-tertiary` / `--color-bg-highlight` |
| `--on-surface-color`           | **no**   | (dropped) |
| `--media-outline-color`        | **no**   | `--color-border-media` (same value `rgb(252 248 255 / 15%)`) |
| `--error-background-color`     | **no**   | `--color-bg-error-base` |
| `--error-active-background-color` | **no** | `--color-bg-error-base-hover` |
| `--on-error-color`             | **no**   | `--color-text-on-error-base` |

So porting the `:root` block verbatim would silently do almost nothing. Keep the three live ones,
translate the rest to the `--color-*` tokens.

## Palette mapping: espresso to the token system

The old overlay recolored the world by overriding `$ui-base-color` and friends. The equivalent in the
token era is to override the **neutral ramp** (so greys become browns) for the dark scheme, and
override the **brand** tokens (so indigo becomes golden). Suggested mapping for a `[data-color-scheme='dark']`
espresso block:

Backgrounds (was driven by `$ui-base-color` family):

| espresso color | old role | token to override | default value replaced |
|----------------|----------|-------------------|------------------------|
| `#21140c` `$bg-darker`   | main/base bg    | `--color-grey-950` (feeds `--color-bg-primary`)   | `#181820` |
| `#34251b` `$bg-dark`     | one step up bg  | `--color-grey-900` (feeds `--color-bg-secondary`) | `#21212c` |
| `#4f3828` `$bg-highlight`| raised / drawer | `--color-grey-800` (feeds `--color-bg-tertiary`)  | `#3a3a50` |

Text (was `$ui-primary/secondary`, `$dark-text-color`):

| espresso color | old role | token to override |
|----------------|----------|-------------------|
| `#dfc29f` `$action-soft` | brightest text | `--color-grey-100` (feeds `--color-text-primary`) |
| `#d9a982` `$action`      | secondary text | `--color-grey-300` (feeds `--color-text-secondary`) |
| `#9c856a` `$text-dim`    | muted text     | `--color-grey-400` / `--color-grey-600` (tertiary/disabled) |

Brand / accent (was `$ui-highlight-color`, `$ui-button-background-color`):

| espresso color | old role | token to override |
|----------------|----------|-------------------|
| `#e5ab14` `$highlight`  | highlight / button bg | `--color-text-brand`, `--color-bg-brand-base`, `--color-border-brand` (or retint `--color-indigo-*`) |
| `#21140c` `$bg-darker`  | text on the golden button | `--color-text-on-brand-base` (default `#fff`; espresso wants dark brown for contrast) |

Errors and shadow:

- `$error-red` overrides map onto the `--color-red-*` ramp / `--color-bg-error-*` and
  `--color-text-error`.
- `$base-shadow-color` (black) is now `--color-black`, consumed via `--color-shadow-primary`.

New tokens with no espresso equivalent yet (added in 4.7, will otherwise inherit grey/indigo defaults,
which look wrong on brown): `--color-bg-blend`, `--color-bg-highlight`, `--color-border-strong`. Give
these brown/gold values if the theme uses surfaces that read them.

## Recommended porting strategy

1. Drop `espresso/variables.scss` as written. It targets the removed SCSS-var system and won't
   compile (no `mastodon/functions`, no configurable `$ui-*` in `mastodon/variables`).
2. Rebuild the theme as a **color-scheme token override**: a `[data-color-scheme='dark']` (and/or
   `:root`) block that redefines the neutral ramp to browns and the brand tokens to gold, per the
   tables above. This is the modern equivalent of the old `with(...)` override and recolors the whole
   UI in one place.
3. From `espresso/diff.scss`, keep the live `:root` overrides (`--dropdown-shadow`,
   `--overlay-icon-shadow`, `--avatar-border-radius`) and drop or retarget the dead ones to their
   `--color-*` replacements.
4. Keep the element-level rules in `diff.scss` (borderless dropdowns, drawer bg, white compose text,
   non-purple quote icon, column background fills, transparent avatar bg, settings-UI tweaks) but
   re-check selectors, several column/header selectors changed across 4.3 to 4.7 and the popover
   attributes were renamed `data-popper-*` to `data-popover-*` (see the other doc).
5. Replace every `darken()` / `lighten()` call with `color-mix(in oklab, ..., black/white x%)` or the
   `css-alpha()` util; the custom SCSS functions are gone.
6. Re-derive the espresso `:root` custom properties (the ones copied from the old
   `css_variables.scss`) straight from the palette values, since there are no `$ui-*` SCSS vars left
   to compute them from.
7. Register `espresso` and `latte` wherever themes are registered on the new base (was
   `config/themes.yml`), and resolve the external `owocafe/*` partials as before.

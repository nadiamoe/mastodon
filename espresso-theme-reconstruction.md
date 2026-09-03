# Espresso (and Latte) theme reconstruction notes

Notes for recreating the theme added in commit `359ed86` ("copy styles from overlay") on a
different state of the codebase. The goal here is to preserve the *spirit* of the theme, mostly the
palette and what each color is for, so it can be rebuilt even if the surrounding SCSS plumbing has
changed.

## What the commit adds

Four files, all under `app/javascript/styles/`:

- `espresso.scss` - the theme entrypoint.
- `espresso/variables.scss` - the palette and the Mastodon variable overrides.
- `espresso/diff.scss` - the CSS overrides that the variables alone can't express.
- `latte.scss` - a one-line light companion theme.

The themes almost certainly also need registering in `config/themes.yml` (in this checkout that file
only lists `default: styles/application.scss`). The overlay this was copied from presumably adds:

```yaml
espresso: styles/espresso.scss
latte: styles/latte.scss
```

If the target codebase registers themes elsewhere, register them there instead.

## Compatibility warning (read first)

The committed patch targets an *older* Mastodon style layer that still exposed:

- Legacy SCSS color variables configured via `@use '../mastodon/variables' with(...)`
  (`$ui-base-color`, `$ui-highlight-color`, `$error-red`, `$base-shadow-color`, etc.).
- A `mastodon/functions` module (used for `darken`/`lighten`/`transparentize`/`rgba` helpers).

Newer Mastodon (4.3+ token work) moved to CSS custom-property tokens in
`app/javascript/styles/mastodon/theme/` and dropped those legacy SCSS variables. If the target tree
is the newer kind, the `with(...)` block and `@use '../mastodon/functions'` won't resolve, and you'll
need to translate the intent below into the token system (`--color-*` overrides) instead of literally
copying. The color values and their *purpose* stay the same regardless; that's what these notes
capture.

## The palette ("Kiara's espresso palette")

An espresso/coffee theme. Dark brown backgrounds, a golden crema highlight, and tan/latte-foam text.

| Variable       | Hex       | Feel                | Used for |
|----------------|-----------|---------------------|----------|
| `$bg-black`    | `#000000` | pure black          | reference only |
| `$bg-darker`   | `#21140c` | near-black brown    | main base background (`$ui-base-color`); also the *text color* placed on highlighted/golden surfaces for contrast |
| `$bg-dark`     | `#34251b` | dark brown          | simple/page background, one step lighter base; compose-form warning background |
| `$bg-highlight`| `#4f3828` | medium brown        | stands in for a `lighten()`-ed `$bg-dark`; dropdown borders, drawer background, dropdown item hover |
| `$highlight`   | `#e5ab14` | golden amber        | THE accent. Highlight color, primary button background, passive/active-passive text |
| `$action`      | `#d9a982` | light tan           | primary UI color, secondary buttons, action buttons, darker modal text |
| `$action-soft` | `#dfc29f` | cream / latte foam  | secondary UI color, tertiary buttons, column links, lighter compose-box text, compose-form warning text |
| `$text-dim`    | `#9c856a` | muted taupe brown   | muted/notification text (`$dark-text-color`), the big quote icon |

Palette intent notes worth keeping:

- The golden highlight is deliberately brighter than the source. Kiara's original was `#ca8f04`
  (kept as a comment: "too light"), bumped to `#e5ab14`. If reconstructing, keep the brighter one.
- Because `$highlight` is bright, text sitting *on top of* highlighted surfaces is forced to
  `$bg-darker` (dark brown) for legibility. This is a recurring theme, see diff.scss.
- `$bg-highlight` source reference: https://www.color-hex.com/color-palette/24950

## Variable overrides (espresso/variables.scss)

Mapping of the palette onto Mastodon's configurable variables. Preserve these *roles*:

- Backgrounds: `$simple-background-color: $bg-dark`, `$ui-base-color: $bg-darker`,
  `$ui-base-lighter-color: $bg-dark`.
- Accents: `$ui-primary-color: $action`, `$ui-secondary-color: $action-soft`,
  `$ui-highlight-color: $highlight`.
- Primary buttons: golden background with dark-brown text.
  `$ui-button-color: $bg-darker`, `$ui-button-background-color: $highlight`, focus states are
  `darken($highlight, 7%)` (background, outline color, and a `solid 2px` outline).
- Secondary buttons: tan outline/text (`$action`), focus text goes dark brown (`$bg-darker`). The
  secondary focus *background* override is commented out in the source, leave it out.
- Tertiary buttons: cream (`$action-soft`) outline/text, cream focus background, dark-brown focus text.
- `$dark-text-color: $text-dim` (muted text, notifications).
- `$action-button-color: $action`.
- `$passive-text-color` and `$active-passive-text-color`: both `$highlight` (golden).
- Inverted-surface text: the theme forces most "inverted" areas to not look inverted, so inverted
  text colors are set to normal-reading values:
  - `$inverted-text-color: #fff` (reply-to window, emoji picker, boost modal)
  - `$light-text-color: $action` (darker modal text)
  - `$lighter-text-color: $action-soft` (some compose-box buttons)

## CSS overrides (espresso/diff.scss)

Things the variable system alone couldn't do. Each block has a clear intent; recreate the intent even
if selectors have drifted.

`:root` custom properties (derive from palette, don't hardcode new colors):

- Dropdowns: border = `$bg-highlight`; background = `darken($ui-base-color, 8%)`; a soft two-layer
  shadow from `rgba($base-shadow-color, 0.25)`.
- Modal / hover card: background = `darken($ui-base-color, 8%)` (deliberately *different* from column
  bg), border = `$ui-base-color`.
- Backgrounds: `--background-filter: blur(10px) saturate(180%) contrast(75%) brightness(70%)`;
  `--background-color: darken($ui-base-color, 8%)`; border set equal to `$ui-base-color` to be
  effectively borderless; `--background-color-raised: $ui-base-color` (bg for "raised" items like
  columns); `--background-color-tint: rgba(darken($ui-base-color, 8%), 0.9)`.
- Surfaces: `--surface-background-color` and `--surface-variant-background-color` = `$ui-base-color`;
  active variant = `lighten($ui-base-color, 4%)`; `--on-surface-color: transparentize($ui-base-color, 0.5)`.
- `--avatar-border-radius: 8px` (rounded-square avatars, not full circles).
- `--media-outline-color: rgba(#fcf8ff, 0.15)` (faint near-white outline).
- `--overlay-icon-shadow: drop-shadow(0 0 8px rgba($base-shadow-color, 0.25))`.
- Errors: background = `darken($error-red, 16%)`, active = `darken($error-red, 12%)`,
  `--on-error-color: #fff`.

Element overrides and their reasons:

- `.compose-form__warning`: `$bg-dark` background, `$action-soft` text (both `!important`).
- Text-on-golden fix: a big selector list (active dropdown button, admin selected tab, active/focused
  privacy and language dropdown options and their children) forced to `color: $bg-darker !important`.
  Reason: the highlight is bright, so text on it must be dark.
- `.column-link`: `$action-soft` so column links read as cream/white by default.
- `.dropdown-menu`: `border: none` (nuke dropdown border).
- `.dropdown-menu__item a:hover`: background `$bg-highlight` (because default would use the border
  color, which we set to match bg).
- Leftmost drawer (`.drawer__inner`, `.drawer__inner__mastodon`, `.drawer__header`): `$bg-highlight`
  background `!important`, to make the drawer a touch lighter. Marked with a TODO to revisit after
  v4.3.0.
- `.compose-form .autosuggest-textarea__textarea`: `color: #fff !important` (white compose text).
- `.status__quote-icon`: `$text-dim !important` (the big quote icon defaults to purple/blurple; kill
  that).
- Column backgrounds: `div.column > div`, `h1.column-header`, `div.column-header__collapsible` get
  `background-color: var(--background-color-raised)`. Reason: 4.3.x dropped the dedicated column
  background in favor of a border; this puts the fill back.
- `div.column-header__wrapper`: `background: none !important` (the wrapper has no border, unlike
  `h1.column-header`, so it must stay transparent).
- `div.account__avatar`: `background-color: transparent !important` (no black box behind avatars).
- `div.status--is-quote`: `overflow-y: hidden` as a workaround for quotes expecting a background
  that isn't there. TODO left to properly fix transparent column backgrounds.
- `div.admin-wrapper`: `var(--background-color-raised)` (settings UI uses the lighter bg).
- `.sidebar ul li a.selected`: `var(--background-color-raised)` (highlight selected settings menu).
- `.simple_form select`: `var(--background-color) !important` (darker bg for select controls).

## espresso.scss (entrypoint)

Composition order matters:

```scss
@use 'espresso/variables';
@use 'espresso/diff';
@use 'application';

@use 'owocafe/zoom-emojis';
@use 'owocafe/no-alt';
```

Note: `owocafe/zoom-emojis` and `owocafe/no-alt` are NOT in this repo. They come from a separate
`owocafe` overlay. If they're absent in the target tree, either bring them over from that overlay or
drop those two `@use` lines. `zoom-emojis` = enlarge emojis on hover; `no-alt` = hide the "ALT" badge
on media (going by their names).

## latte.scss

The light companion, a single line:

```scss
@use 'mastodon-light';
```

It just re-exports Mastodon's built-in light theme under the name "latte". No customization. If
`mastodon-light` doesn't exist as an importable name in the target tree, point it at whatever the
current light theme entrypoint is.

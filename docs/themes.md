---
title: Themes
layout: default
nav_order: 10
parent: Docs
permalink: /docs/themes/
---
# Themes

Themes provide semantic style tokens for templates, views, components, and layouts. Use theme tokens instead of hardcoded colors whenever possible.

## Register Built-In Themes

Six themes ship with the framework: `phosphor`, `catppuccin-mocha`,
`catppuccin-latte`, `gruvbox-dark`, `nord`, and `tokyonight`. Register the ones you
want by name (or loop `Charming::UI::Theme.built_in_names` for all of them):

```ruby
class MyApp::Application < Charming::Application
  theme :phosphor, built_in: "phosphor"
  theme :mocha, built_in: "catppuccin-mocha"
  theme :nord, built_in: "nord"

  default_theme :phosphor
end
```

## Register Custom Themes

Register a custom JSON theme file with `from:`:

```ruby
class MyApp::Application < Charming::Application
  theme :custom, from: "config/themes/custom.json"
  default_theme :custom
end
```

Relative paths resolve from `Application.root` when set, otherwise from the current working directory.

## Derive Themes

`extends:` builds a theme from an already-registered one, merging `overrides:`
(token name → style spec) on top:

```ruby
theme :base, built_in: "phosphor"
theme :high_contrast, extends: :base, overrides: {
  text: {foreground: "#ffffff", bold: true},
  title: {foreground: "#FFD75F", bold: true}
}
```

Override colors are literal (hex strings, named symbols, or 256-color indexes) — the
parent's palette names are resolved when the parent loads. Register the parent before
extending it.

## Color Capability

Themes are written in truecolor and degrade automatically. At boot,
`Charming::UI::ColorSupport` classifies the terminal — `NO_COLOR` wins, then
`COLORTERM` (`truecolor`/`24bit`), then `TERM` — into one of `:truecolor`,
`:color256`, `:color16`, or `:none`. Hex colors downconvert to the nearest 256-color
index or basic ANSI color as needed; at `:none`, color codes are omitted entirely
(text attributes like bold survive).

Force a level when needed (tests pin `:truecolor` for stable output):

```ruby
Charming::UI::ColorSupport.level = :color256
Charming::UI::ColorSupport.level = nil   # back to auto-detection
```

## Use Theme Tokens

Theme tokens return `Charming::UI::Style` objects:

```ruby
text "Welcome", style: theme.title
text "Status", style: theme.muted
text "Alert", style: theme.info
```

Default tokens:

| Token | Meaning |
|-------|---------|
| `text` | Primary text |
| `title` | Bright title text |
| `muted` | Secondary text |
| `border` | Border styling |
| `selected` | Selected/focused item styling |
| `info` | Informational accent |
| `warn` | Warning accent |

Tokens are style objects, so they can be chained:

```ruby
theme.title.align(:center).width(40)
theme.border.border(:rounded).padding(1, 2)
```

## Runtime Theme Switching

Switch themes from a controller with:

```ruby
use_theme :phosphor
```

Generated apps expose a theme picker command:

```ruby
command "Theme", :open_theme_palette
```

`open_theme_palette` opens a command-palette-like list of registered themes.

## Theme JSON Shape

Theme JSON files contain a `styles` object and may contain a `palette` and `background`:

```json
{
  "palette": {
    "cyan": "#00ffff"
  },
  "background": "#101820",
  "styles": {
    "title": { "foreground": "cyan", "bold": true },
    "muted": { "foreground": "#777777" },
    "border": { "foreground": "cyan" }
  }
}
```

Style options mirror `Charming::UI::Style` methods: foreground/background colors, text attributes, padding, border, width, height, and alignment.

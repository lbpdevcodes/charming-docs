---
title: UI & Themes
layout: default
parent: API Reference
nav_order: 6
permalink: /docs/api/ui/
---
# UI & Themes

`Charming::UI` is the low-level styling and layout toolkit: immutable `Style` builders, ANSI-aware measurement and joining, color capability detection, and gradients. Themes sit on top — named token sets (`title`, `muted`, `border`, …) that resolve to styles so app code never hard-codes colors. For a guided tour of the `Style` builder, colors, and borders, see [Styling]({{ '/docs/styling/' | relative_url }}); for theme registration, see [Application & Configuration]({{ '/docs/api/application/' | relative_url }}).

## Layout helpers

`Charming::UI` provides ANSI-aware layout helpers:

- `Charming::UI.style`
- `Charming::UI.adaptive(light:, dark:)` — a color resolved against the terminal background at render time
- `Charming::UI.join_horizontal(*blocks, gap: 0, align: :top)` — `align:` is `:top`/`:center`/`:bottom` or a 0.0–1.0 fraction
- `Charming::UI.join_vertical(*blocks, gap: 0, align: :left)` — pads narrower blocks to the widest; `align:` is `:left`/`:center`/`:right` or a fraction
- `Charming::UI.center(block, width:, height:)`
- `Charming::UI.place(block, width:, height:, top: 0, left: 0, background: nil)` — positions take integers, `:center`, or fractions
- `Charming::UI.overlay(base, overlay, top: :center, left: :center)`
- `Charming::UI.visible_slice(line, start_column, width)`
- `Charming::UI::Width.measure(value)` / `pad_to(line, width)` / `widest(lines)`
- `Charming::UI::Width.strip_ansi(value)`
- `Charming::UI::Truncate.tail(text, width, ellipsis: "…")`
- `Charming::UI::TextWrapper.new(width:).wrap(value)`
- `Charming::UI::Gradient.blend(from, to, amount)` / `steps(from, to, count)` / `colorize(text, from:, to:)`

## Color capability

Color capability (`Charming::UI::ColorSupport`):

- `level` — the active capability: `:truecolor`, `:color256`, `:color16`, or `:none` (auto-detected from `NO_COLOR`, `COLORTERM`, `TERM`).
- `level = value` — force a level; `nil` re-enables detection.
- `at_least?(:color256)` — capability comparison.
- Hex colors in styles and themes downconvert automatically to the active level.

Terminal background (`Charming::UI::Background`):

- `dark?` — whether the background is dark (OSC 11 query at runtime startup, `COLORFGBG` fallback, dark default).
- `assume = :dark | :light | nil` — override or re-enable detection.

## Styles

Styles are immutable builders:

```ruby
style.foreground(:cyan).bold.border(:rounded).padding(1, 2).width(40)
```

Common style methods:

- `foreground` / `fg`
- `background` / `bg`
- `bold`, `faint`, `italic`, `underline`, `reverse`, `strikethrough`
- `padding`, `margin` (CSS shorthand; per-side setters like `padding_left`, `margin_top`)
- `border(style, sides:, foreground:, background:)` — style name or a custom `UI::Border`; `foreground:` takes a color or per-side hash
- `width`, `height`, `max_width`, `max_height`
- `align`, `align_vertical`
- `wrap`, `truncate(ellipsis: "…")` — overflow fit modes at a fixed width (default is clip)
- `render(value)`

## Themes

Theme tokens return `Charming::UI::Style` objects:

- `text`
- `title`
- `muted`
- `border`
- `selected`
- `info`
- `warn`

Themes can be loaded with `theme name, built_in:` or `theme name, from:` on the application class.

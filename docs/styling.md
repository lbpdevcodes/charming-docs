---
title: Styling
layout: default
nav_order: 11
permalink: /docs/styling/
---
# Styling

Charming styles terminal text with `Charming::UI::Style`, a Lip Gloss–inspired
immutable builder. Every method returns a **new** style, so styles are safe to share
and chain. `render(value)` applies the accumulated style to a string.

Most app code reaches styles through [theme tokens]({{ '/docs/themes/' | relative_url }})
(`theme.title`, `theme.muted`, …) — those tokens *are* `Style` objects, so everything
here applies to them too.

## The builder

```ruby
style = Charming::UI.style          # a blank style
styled = style.foreground(:cyan).bold.border(:rounded).padding(1, 2).width(40)
puts styled.render("Hello")
```

Because each call returns a new instance, a base style can be branched safely:

```ruby
base = Charming::UI.style.foreground("#7dcfff")
heading = base.bold.underline
muted   = base.faint
```

## Text attributes

Each attribute toggles one ANSI text effect:

| Method | Effect |
|--------|--------|
| `bold` | Increased intensity. |
| `faint` | Dimmed intensity. |
| `italic` | Italicized (terminal-dependent). |
| `underline` | Underlined. |
| `reverse` | Swap foreground and background. |
| `strikethrough` | Struck-through. |

```ruby
Charming::UI.style.bold.underline.render("Title")
```

## Colors

`foreground` (alias `fg`) and `background` (alias `bg`) accept four forms:

- a **named color** symbol — `:black`, `:red`, `:green`, `:yellow`, `:blue`,
  `:magenta`, `:cyan`, `:white`, and their `:bright_*` variants;
- a **256-color index** integer, `0`–`255`;
- a **hex** string, `"#rrggbb"` or shorthand `"#rgb"`;
- an **adaptive color** that picks a variant per terminal background (below).

```ruby
Charming::UI.style.foreground(:bright_green).render("ok")
Charming::UI.style.fg(81).bg(236).render(" indexed ")
Charming::UI.style.foreground("#ff8800").render("hex")
```

### Adaptive light/dark colors

`Charming::UI.adaptive(light:, dark:)` builds a color that resolves at render time
against the terminal's background, so one style reads well on both dark and light
terminals:

```ruby
ink = Charming::UI.adaptive(light: "#1a1a2e", dark: "#e6e6fa")
Charming::UI.style.foreground(ink).render("readable anywhere")
```

The runtime detects the background at startup by querying the terminal (OSC 11),
falling back to the `COLORFGBG` convention, and finally assuming dark — the common
terminal default. Override or reset detection with:

```ruby
Charming::UI::Background.assume = :light   # or :dark; nil re-enables detection
Charming::UI::Background.dark?             # => true/false
```

The Markdown component's `:auto` style uses the same detection to pick its dark or
light style config.

### Color capability and downconversion

Hex and indexed colors **downconvert automatically** to whatever the terminal
supports, so a truecolor theme degrades gracefully. The active capability lives in
`Charming::UI::ColorSupport`:

| Level | Meaning |
|-------|---------|
| `:truecolor` | 24-bit color. |
| `:color256` | xterm 256-color cube. |
| `:color16` | 16 basic ANSI colors. |
| `:none` | No color (escapes are dropped). |

The level is auto-detected from the environment — `NO_COLOR` forces `:none`,
`COLORTERM=truecolor`/`24bit` forces `:truecolor`, otherwise `TERM` is inspected
(`*-256color` → `:color256`, `dumb`/empty → `:none`). Override or query it:

```ruby
Charming::UI::ColorSupport.level = :color256   # force a level (nil re-enables detection)
Charming::UI::ColorSupport.at_least?(:color256) # => true/false capability check
```

Specs pin the level for stable captures (`spec_helper.rb` sets
`Charming::UI::ColorSupport.level = :truecolor`).

## Borders

`border(style = :normal, sides: nil, foreground: nil, background: nil)` draws a
box. The named styles:

| Style | Looks like |
|-------|-----------|
| `:normal` / `:ascii` | ASCII `+`, `-`, `|` |
| `:square` | `┌─┐ │ └─┘` |
| `:rounded` | `╭─╮ │ ╰─╯` |
| `:thick` | `┏━┓ ┃ ┗━┛` |
| `:double` | `╔═╗ ║ ╚═╝` |
| `:block` | `███ █ ███` |
| `:hidden` | Spaces — keeps the box footprint without drawing anything. |

```ruby
Charming::UI.style.border(:double).padding(0, 1).render("Boxed")
```

- `sides:` restricts the border to specific edges, e.g. `sides: [:top, :bottom]`.
- `foreground:` colors the border independently of the content:
  `border(:rounded, foreground: "#7dcfff")` — or per side with a hash:
  `border(:rounded, foreground: {top: :red, bottom: :blue})` (corners take the
  top/bottom row's color).
- `background:` colors the border cells independently of the box background.
- A custom `Charming::UI::Border` instance works anywhere a style name does:

```ruby
dotted = Charming::UI::Border.new(corners: %w[. . . .], edges: %w[. :])
Charming::UI.style.border(dotted).render("custom")
```

In the [layout DSL]({{ '/docs/layouts/' | relative_url }}), panes take the same
border names: `pane(:content, border: :rounded)`.

## Box model

```ruby
Charming::UI.style.width(40).height(3).align(:center).padding(1, 2).margin(1).render(text)
```

- `padding(*values)` and `margin(*values)` follow CSS shorthand: **1** value → all
  sides, **2** → `[vertical, horizontal]`, **4** → `[top, right, bottom, left]`.
  Padding sits inside the border and takes the style's colors; margin sits outside
  the border and stays unstyled. Per-side setters exist for both:
  `padding_left(2)`, `margin_top(1)`, etc.
- `width(n)` / `height(n)` fix the rendered size in display columns / rows, padding
  content out to fit.
- `max_width(n)` / `max_height(n)` cap the size instead — content narrower or
  shorter than the cap is left alone.
- `align(:left | :center | :right)` aligns content within the width;
  `align_vertical(:top | :middle | :bottom)` positions it within a fixed height.

### Overflow: clip, wrap, or truncate

Content wider than a fixed `width` is **clipped** by default. Two other fit modes:

```ruby
Charming::UI.style.width(20).wrap.render(paragraph)       # word-wrap at the width
Charming::UI.style.width(20).truncate.render(long_line)   # cut with a trailing …
Charming::UI.style.width(20).truncate(ellipsis: "...").render(long_line)
```

All three preserve ANSI styling at the cut and never split a double-width glyph.
`Charming::UI::Truncate.tail(text, width)` exposes the ellipsis cut directly.

The render pipeline applies everything in order: wrap to width → align (horizontal
and vertical) → expand to height → padding → border → ANSI escapes → margin.

## Composing blocks

`Charming::UI` provides ANSI-aware helpers for assembling rendered strings (they
respect display width, so wide glyphs and escape codes stay aligned):

```ruby
Charming::UI.join_horizontal(left, right, gap: 2)   # side by side
Charming::UI.join_vertical(header, body, gap: 1)    # stacked
Charming::UI.center(block, width: 80, height: 24)   # center in a box
Charming::UI.place(block, width: 80, height: 24, top: :center, left: :center)
Charming::UI.overlay(base, modal, top: :center, left: :center)  # draw over a base
```

Joins take a cross-axis `align:` for blocks of unequal size — symbolic or a
fraction between `0.0` and `1.0`, matching Lip Gloss positions:

```ruby
Charming::UI.join_horizontal(tall, short, align: :center)   # :top | :center | :bottom
Charming::UI.join_vertical(wide, narrow, align: :right)     # :left | :center | :right
Charming::UI.join_horizontal(tall, short, align: 0.75)      # fractional position
```

`join_vertical` pads narrower blocks to the widest, so stacked output is always
rectangular. `place` and `overlay` accept fractional `top:`/`left:` positions too:
`place(block, width: 80, height: 24, top: 0.25, left: :center)`. The view helpers
`row(...)` and `column(...)` pass `align:` through.

For multi-step composition, `Charming::UI::Canvas` is a fixed-size character grid you
can build up and overlay onto:

```ruby
canvas = Charming::UI::Canvas.new(80, 24)        # blank grid
canvas = Charming::UI::Canvas.parse(rendered)    # …or from existing output
canvas.overlay(modal)                            # draw modal centered on top
canvas.to_s
```

## Gradients

`Charming::UI::Gradient` interpolates between two hex colors:

```ruby
Charming::UI::Gradient.blend("#ff0000", "#0000ff", 0.5)      # => "#800080"
Charming::UI::Gradient.steps("#ff0000", "#0000ff", 5)        # evenly spaced ramp
Charming::UI::Gradient.colorize("charming", from: "#ff0000", to: "#0000ff")
```

`colorize` sweeps the foreground color across the text one grapheme at a time
(multi-codepoint emoji stay intact). `Progressbar` and `ActivityIndicator` use the
same module for their gradient fills.

## A note on display width

All of the above measure **display width**, not character count, via
`Charming::UI::Width.measure` — it strips ANSI escapes and counts wide glyphs
(CJK and emoji) as two columns, so borders and alignment stay correct even over
emoji content. You rarely call it directly, but it is why a styled, emoji-laden row
still lines up.

For the compact method-by-method reference, see the
[API Reference]({{ '/docs/api/' | relative_url }}#ui).

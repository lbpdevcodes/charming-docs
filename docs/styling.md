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

`foreground` (alias `fg`) and `background` (alias `bg`) accept three forms:

- a **named color** symbol — `:black`, `:red`, `:green`, `:yellow`, `:blue`,
  `:magenta`, `:cyan`, `:white`, and their `:bright_*` variants;
- a **256-color index** integer, `0`–`255`;
- a **hex** string, `"#rrggbb"`.

```ruby
Charming::UI.style.foreground(:bright_green).render("ok")
Charming::UI.style.fg(81).bg(236).render(" indexed ")
Charming::UI.style.foreground("#ff8800").render("hex")
```

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

`border(style = :normal, sides: nil, foreground: nil)` draws a box. Four named
styles ship:

| Style | Looks like |
|-------|-----------|
| `:normal` | ASCII `+`, `-`, `|` |
| `:rounded` | `╭─╮ │ ╰─╯` |
| `:thick` | `┏━┓ ┃ ┗━┛` |
| `:double` | `╔═╗ ║ ╚═╝` |

```ruby
Charming::UI.style.border(:double).padding(0, 1).render("Boxed")
```

- `sides:` restricts the border to specific edges, e.g. `sides: [:top, :bottom]`.
- `foreground:` colors the border independently of the content:
  `border(:rounded, foreground: "#7dcfff")`.

In the [layout DSL]({{ '/docs/layouts/' | relative_url }}), panes take the same
border names: `pane(:content, border: :rounded)`.

## Box model

```ruby
Charming::UI.style.width(40).height(3).align(:center).padding(1, 2).render(text)
```

- `padding(*values)` follows CSS shorthand: **1** value → all sides, **2** →
  `[vertical, horizontal]`, **4** → `[top, right, bottom, left]`.
- `width(n)` / `height(n)` fix the rendered size in display columns / rows (content
  is wrapped/clipped or padded to fit).
- `align(:left | :center | :right)` aligns content within the width.

The render pipeline applies them in order: wrap to width → align → expand to height →
padding → border → ANSI escapes.

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

For multi-step composition, `Charming::UI::Canvas` is a fixed-size character grid you
can build up and overlay onto:

```ruby
canvas = Charming::UI::Canvas.new(80, 24)        # blank grid
canvas = Charming::UI::Canvas.parse(rendered)    # …or from existing output
canvas.overlay(modal)                            # draw modal centered on top
canvas.to_s
```

## A note on display width

All of the above measure **display width**, not character count, via
`Charming::UI::Width.measure` — it strips ANSI escapes and counts wide glyphs
(CJK and emoji) as two columns, so borders and alignment stay correct even over
emoji content. You rarely call it directly, but it is why a styled, emoji-laden row
still lines up.

For the compact method-by-method reference, see the
[API Reference]({{ '/docs/api/' | relative_url }}#ui).

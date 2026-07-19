---
title: StatusBar
layout: default
parent: Components
nav_order: 18
permalink: /docs/components/status-bar/
---
# StatusBar

StatusBar is the classic one-row TUI bottom bar: left, center, and right segments laid onto a single fixed-width row for mode indicators, key hints, and app state. It is a static renderable — no keyboard handling.

```text
 Entries         enter open  n new  q quit        8 entries
```

The whole row is painted in the bar style (reverse video by default), so it reads as a solid strip.

## Quick start

From the journal example's layout — a one-row pane at the bottom of the screen:

```ruby
screen_layout(background: theme.background) do
  split :vertical do
    split(:horizontal, gap: 1, grow: 1) { ... }

    pane(:status_bar, height: 1) do
      render_component status_bar
    end
  end
end
```

```ruby
def status_bar
  Charming::Components::StatusBar.new(
    width: screen.width,
    left: " #{controller.route&.title || "Journal"}",
    hints: controller.status_hints,
    right: "#{controller.entry_count} entries ",
    theme: theme
  )
end
```

where `status_hints` is an array of pairs like `[["ctrl+p", "commands"], ["?", "help"], ["q", "quit"]]`.

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `width:` | required | Total bar width — the row is padded (and clipped) to exactly this many columns. |
| `left:` | `""` | Left-anchored segment. |
| `center:` | `""` | Centered segment. |
| `right:` | `""` | Right-anchored segment. |
| `hints:` | `nil` | Array of `[key, description]` pairs rendered in the center as `key description  key description` — only used when `center:` is empty. |
| `style:` | `nil` | Overrides the bar style. Default is `theme.selected` (reverse video). |
| `theme:` | app default | Theme the default style comes from. |

## Working with it

- Segments that would collide are clipped rather than wrapped — the bar is always exactly one row of `width` columns.

## Tips

- Pass `width: screen.width` and put the bar in a `height: 1` pane; anything taller than one line in a segment will break the row.
- An explicit `center:` wins over `hints:` — the hints are silently ignored when both are given.
- Leading/trailing spaces in `left:` / `right:` (as in the example) keep text off the terminal edge, since the bar itself has no padding.

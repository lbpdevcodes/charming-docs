---
title: Sparkline
layout: default
parent: Components
nav_order: 28
permalink: /docs/components/sparkline/
---
# Sparkline

`Charming::Components::Sparkline` renders a series of numbers as a compact one-line bar graph using the eighth-block glyphs `▁▂▃▄▅▆▇█` — one cell per value, scaled between the series' min and max. It is a static renderable, pure text, so it works on every terminal. Reach for it when you want a trend at a glance inside a status bar, table row, or dashboard line; reach for [Chart]({{ '/docs/components/chart/' | relative_url }}) when you need more than one row of resolution.

```text
▁▂▄▆█▆▄▂▁▃▅▇
```

## Quick start

```ruby
render_component(Charming::Components::Sparkline.new(
  values: assigns.fetch(:series),
  style: theme.info
))
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `values:` | required | The numeric series — one output cell per value. |
| `style:` | `nil` | A `Charming::UI::Style` used to paint the glyphs (e.g. `theme.info`). |
| `theme:` | `nil` | Forwarded to the view layer. |

## Tips

- Values normalize to the eight bar heights: the minimum gets `▁`, the maximum gets `█`. There is no shared scale between two sparklines, and a flat series renders as all `▁`.
- Output width equals `values.length` — trim or bucket the series yourself to fit a layout; the component does no resampling.
- An empty series renders as an empty string.

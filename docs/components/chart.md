---
title: Chart
layout: default
parent: Components
nav_order: 27
permalink: /docs/components/chart/
---
# Chart

`Charming::Components::Chart` plots a numeric series into a `width` × `height` box of character cells. The default `kind: :line` draws a connected line on a `Charming::UI::BrailleCanvas`, which gives you 2×4 subpixels per cell; `kind: :bar` draws vertical bars from eighth-block glyphs. It is a static renderable — pure text with no keyboard handling — so it works on every terminal and composes with `row`/`column`/`box` like any other view output.

```text
Line (braille)                  Bar (blocks)
⠀⠀⠀⠀⢀⠔⠉⠑⢄⠀⠀⠀⠀⠀⠀⠀           ▂█
⠀⠀⡠⠊⠀⠀⠀⠀⠀⠑⢄⠀⠀⠀⠀⠀          ▅██▃
⡠⠊⠀⠀⠀⠀⠀⠀⠀⠀⠀⠑⢄⡀⠀⠀        ▁█████▆▂
```

## Quick start

```ruby
class ShowView < Charming::View
  def render
    render_component(Charming::Components::Chart.new(
      series: assigns.fetch(:series),
      width: 32,
      height: 4,
      style: theme.info
    ))
  end
end
```

Pass `kind: :bar` for bars:

```ruby
render_component(Charming::Components::Chart.new(
  series: series, width: 16, height: 4, kind: :bar, style: theme.warn
))
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `series:` | required | The numeric data to plot. |
| `width:` | required | Chart width in character cells. |
| `height:` | required | Chart height in character cells. |
| `kind:` | `:line` | `:line` for a braille-canvas line plot, `:bar` for eighth-block bars. |
| `style:` | `nil` | A `Charming::UI::Style` used to paint the whole chart (e.g. `theme.info`). |
| `theme:` | `nil` | Forwarded to the view layer. |

## How the two kinds scale

- **Line** spreads the series across `width * 2` × `height * 4` braille subpixels, scaled between the series' min and max, and connects consecutive points with line segments.
- **Bar** resamples the series to exactly `width` columns (nearest-neighbour), then scales bars from a baseline of `min(0, series minimum)` — so an all-positive series grows from zero, and a series with negatives grows from its minimum.

An empty series (or a non-positive box) renders as an empty string.

## Tips

- Values normalize to the box: the tallest point always touches the top and the range stretches to fill `height`. Two charts side by side are not on a shared scale unless you pad their series to the same min/max yourself.
- In bar mode, a series longer than `width` is resampled down — one column per cell — so fine detail between neighbouring values can disappear. Widen the chart or pre-aggregate if that matters.
- A flat series (all values equal) renders at the baseline rather than mid-box.

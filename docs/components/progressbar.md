---
title: Progressbar
layout: default
parent: Components
nav_order: 24
permalink: /docs/components/progressbar/
---
# Progressbar

`Charming::Components::Progressbar` renders a fixed-width ASCII bar that fills proportionally to a current value out of `total:`. It is a static renderable — advance it with `tick`/`update` from whatever reports progress (a [controller timer]({{ '/docs/controllers-and-views/' | relative_url }}) or a background task's progress events) and re-render.

```text
[==========          ] 10/20 exported
```

## Quick start

Adapted from the journal example app's export screen: a background task reports progress into [state]({{ '/docs/state/' | relative_url }}), and the view rebuilds the bar with `update` each render.

```ruby
class StatsController < ApplicationController
  on_task_progress :export, action: :export_progressed

  def export_progressed
    stats.export_current = event.current
    stats.export_total = event.total
    show
  end
end
```

```ruby
# In the view
bar = Charming::Components::Progressbar.new(
  total: [stats.export_total, 1].max,
  label: "exporting #{stats.export_current}/#{stats.export_total}"
)
bar.update(stats.export_current)
render_component(bar)
```

Timer-driven progress works the same way: increment a counter in the timer's action, `update` from it in the view.

## Options

| Option | Default | What it does |
|---|---|---|
| `total:` | required | Maximum unit count. Also the bar's width in characters. Negative values clamp to 0. |
| `complete:` | `"="` | Character for filled positions. |
| `incomplete:` | `" "` | Character for unfilled positions. |
| `bar_format:` | `:classic` | Reserved for future format variants. |
| `label:` | `nil` | Text appended after the closing bracket. |
| `gradient:` | `nil` | `["#rrggbb", "#rrggbb"]` pair; colors the filled region with a sweep across the full bar width. |

## Working with it

All mutators return `self`, so calls chain.

- `tick(count = 1)` — advances the current value by `count`, clamped to `[0, total]`.
- `update(value)` — sets the current value directly, clamped to `[0, total]`.
- `complete!` — jumps straight to 100%.
- `percent` — current completion as a rounded integer (0–100); returns 0 when `total` is zero.
- `total`, `current`, `label`, `complete`, `incomplete`, `bar_format` — read/write accessors.

## Tips

- The bar is exactly `total` characters wide (plus brackets). `total: 100` renders a 100-column bar — scale your units down (or pick a total that fits your layout) rather than expecting a separate width option.
- `render` shows only the bar and label; if you want a percentage, put `percent` in the label yourself: `label: "#{bar.percent}%"`.
- The gradient sweep spans the whole bar width (via `Charming::UI::Gradient`), so cell colors stay put as the bar fills instead of re-stretching each render — see [Styling]({{ '/docs/styling/' | relative_url }}).
- A bar built in a view lasts one dispatch. Persist the current value in state and `update` from it, as above.

---
title: ActivityIndicator
layout: default
parent: Components
nav_order: 23
permalink: /docs/components/activity-indicator/
---
# ActivityIndicator

`Charming::Components::ActivityIndicator` renders a fixed-width bar of gradient-colored characters plus an optional label with a cycling ellipsis — a busy indicator with more visual texture than a spinner. Characters are chosen by a stable hash of `seed`, frame, and position, so the pattern is deterministic per frame. It is a static renderable: call `tick` from a [controller timer]({{ '/docs/controllers-and-views/' | relative_url }}) and re-render.

```text
3f~A9e@1cB writing..
```

## Quick start

Adapted from the journal example app: the frame counter lives in [state]({{ '/docs/state/' | relative_url }}), a timer advances it, and the view rebuilds the indicator each render.

```ruby
class StatsState < ApplicationState
  attribute :activity_index, :integer, default: 0
end

class StatsController < ApplicationController
  timer :export_spinner, every: 0.1, action: :advance_spinner

  def advance_spinner
    return unless stats.exporting

    stats.activity_index += 1
    show
  end

  private

  def stats
    state(:stats, StatsState)
  end
end
```

```ruby
# In the view
render_component Charming::Components::ActivityIndicator.new(
  index: stats.activity_index, label: "writing"
)
```

## Options

| Option | Default | What it does |
|---|---|---|
| `width:` | `10` | Bar width in characters (minimum 1). |
| `label:` | `nil` | Text after the bar, followed by the cycling ellipsis. |
| `index:` | `0` | Frame counter; the bar pattern repeats every 10 ticks, the ellipsis advances every 4. |
| `seed:` | `0` | Hash seed — different seeds produce different character patterns. |
| `chars:` | hex digits + symbols | Character pool to draw from. Cannot be empty. |
| `gradient:` | `["#ff0000", "#6FD0E3"]` | Two hex colors interpolated across the bar width. |
| `label_style:` | `nil` | A `Style` for the label and ellipsis; falls back to gray (`#cccccc`). |
| `max_width:` | `nil` | Total width cap for bar + label + ellipsis; the bar shrinks to fit. |
| `fallback_label:` | `nil` | Shorter label used when the primary label can't fit within `max_width`. |

## Working with it

- `tick(count = 1)` — advances the frame counter by `count`. Returns `self`.
- Readers for every option: `width`, `label`, `index`, `seed`, `chars`, `gradient`, `label_style`, `max_width`, `fallback_label`.

## Tips

- Use `max_width:` and `fallback_label:` to keep labeled indicators stable in constrained layouts — the bar compresses first, then the label swaps for the fallback:

  ```ruby
  render_component Charming::Components::ActivityIndicator.new(
    width: 24,
    label: "Loading Top Stories",
    max_width: content_width,
    fallback_label: "Working"
  )
  ```

- The instance your view builds is discarded after the dispatch — persist the index in state (as above) rather than ticking a view-local instance.
- The default gradient's cyan endpoint matches the Phosphor theme palette; `gradient:` takes raw hex, so pass your own endpoints on other themes (see [Styling]({{ '/docs/styling/' | relative_url }})).

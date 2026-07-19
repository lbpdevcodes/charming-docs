---
title: Spinner
layout: default
parent: Components
nav_order: 22
permalink: /docs/components/spinner/
---
# Spinner

`Charming::Components::Spinner` is a rotating-frame indicator for "something is happening" moments with no measurable progress. It is a static renderable — it draws one frame per render, and you animate it by calling `tick` from a [controller timer]({{ '/docs/controllers-and-views/' | relative_url }}) and re-rendering.

```text
⠹ Fetching
```

## Quick start

Controllers are rebuilt on every dispatch, so keep the frame counter in [state]({{ '/docs/state/' | relative_url }}) and rebuild the spinner from it each render:

```ruby
class JobsState < ApplicationState
  attribute :spinner_index, :integer, default: 0
end

class JobsController < ApplicationController
  timer :spin, every: 0.1, action: :advance_spinner

  def show
    render :show, jobs: jobs
  end

  def advance_spinner
    jobs.spinner_index += 1
    show
  end

  private

  def jobs
    state(:jobs, JobsState)
  end
end
```

```ruby
# In the view
render_component Charming::Components::Spinner.new(
  style: :dots, index: jobs.spinner_index, label: "Fetching"
)
```

The frame is looked up with the index modulo the frame count, so a counter that grows forever is fine.

## Options

| Option | Default | What it does |
|---|---|---|
| `style:` | `:line` | Picks a named preset from `STYLES` (see below). Unknown styles raise `ArgumentError`. |
| `frames:` | `nil` | Any array of frame strings; overrides `style:`. An empty array raises `ArgumentError`. |
| `index:` | `0` | Starting frame index. |
| `label:` | `nil` | Text appended after the frame, separated by a space. |

## Styles

| Style | Frames |
|---|---|
| `:line` | <code>- \ &#124; /</code> |
| `:dots` | `⠋ ⠙ ⠹ ⠸ ⠼ ⠴ ⠦ ⠧ ⠇ ⠏` |
| `:mini_dot` | `⠁ ⠂ ⠄ ⡀ ⢀ ⠠ ⠐ ⠈` |
| `:jump` | `⢄ ⢂ ⢁ ⡁ ⡈ ⡐ ⡠` |
| `:pulse` | `█ ▓ ▒ ░` |
| `:points` | `∙∙∙ ●∙∙ ∙●∙ ∙∙●` |
| `:globe` | `🌍 🌎 🌏` |
| `:moon` | `🌑 🌒 🌓 🌔 🌕 🌖 🌗 🌘` |
| `:meter` | `▱▱▱ ▰▱▱ ▰▰▱ ▰▰▰ ▰▰▱ ▰▱▱` |
| `:hamburger` | `☱ ☲ ☴ ☲` |
| `:ellipsis` | `"   "` `".  "` `".. "` `"..."` |

## Working with it

- `tick` — advances to the next frame, wrapping at the end. Returns `self`.
- `frames` / `index` / `label` — readers for the current configuration.

## Tips

- A spinner instance built in a view lives for one dispatch only. Either persist the index in state (as above) or hold the spinner somewhere that survives dispatches — don't expect `tick` on a view-local instance to animate anything.
- Match the timer interval to the frame count: `:dots` has 10 frames, so `every: 0.1` completes a cycle per second; `:line` has 4.
- Emoji presets (`:globe`, `:moon`) are double-width in most terminals — account for that in tight layouts.

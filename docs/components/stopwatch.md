---
title: Stopwatch
layout: default
parent: Components
nav_order: 26
permalink: /docs/components/stopwatch/
---
# Stopwatch

`Charming::Components::Stopwatch` is a count-up clock. It renders elapsed time as `mm:ss` (or `h:mm:ss` past an hour) and only accumulates while running. It is a static renderable — call `tick` from a [controller timer]({{ '/docs/controllers-and-views/' | relative_url }}) firing once per second and re-render.

```text
01:05 elapsed
```

## Quick start

Controllers are rebuilt per dispatch, so keep the elapsed seconds and running flag in [state]({{ '/docs/state/' | relative_url }}) and rebuild the stopwatch from them:

```ruby
class WorkoutState < ApplicationState
  attribute :elapsed, :integer, default: 0
  attribute :running, :boolean, default: false
end

class WorkoutController < ApplicationController
  key "s", :toggle
  timer :clock, every: 1, action: :advance

  def show
    render :show, watch: watch
  end

  def toggle
    workout.running = !workout.running
    show
  end

  def advance
    workout.elapsed += 1 if workout.running
    show
  end

  private

  def watch
    Charming::Components::Stopwatch.new(label: "elapsed").start.tick(workout.elapsed)
  end

  def workout
    state(:workout, WorkoutState)
  end
end
```

```erb
<%= render_component watch %>
```

`start` is needed before `tick` because a stopped stopwatch ignores ticks — the view replays the accumulated seconds into a fresh instance each render.

## Options

| Option | Default | What it does |
|---|---|---|
| `label:` | `nil` | Text appended after the clock. |
| `theme:` | `nil` | Theme passed through to the component base class. |

## Working with it

All mutators return `self`, so calls chain.

- `start` — begins accumulating time.
- `stop` — pauses accumulation.
- `running?` — true while accumulating.
- `tick(seconds = 1)` — adds `seconds` while running; a no-op when stopped.
- `reset` — zeroes elapsed time and stops.
- `elapsed` / `label` — readers.

## Tips

- Fractional ticks accumulate while running: a controller timer firing at `every: 0.5` can call `tick(0.5)` and two ticks advance the clock one second. The display still shows whole seconds (`TimeDisplay.clock` floors).
- `tick` on a stopped stopwatch silently does nothing — a stopwatch that "never moves" usually means `start` was never called.
- The component measures nothing itself; elapsed time is exactly the sum of your ticks. A paused controller timer pauses the stopwatch.

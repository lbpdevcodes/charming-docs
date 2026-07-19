---
title: Timer
layout: default
parent: Components
nav_order: 25
permalink: /docs/components/timer/
---
# Timer

`Charming::Components::Timer` is a countdown clock. It renders the remaining time as `mm:ss` (or `h:mm:ss` once an hour is involved) and clamps at zero. It is a static renderable — call `tick` from a [controller timer]({{ '/docs/controllers-and-views/' | relative_url }}) firing once per second, check `expired?`, and re-render.

```text
04:32 remaining
```

## Quick start

Controllers are rebuilt per dispatch, so track elapsed seconds in [state]({{ '/docs/state/' | relative_url }}) and rebuild the countdown from it:

```ruby
class PomodoroState < ApplicationState
  attribute :elapsed, :integer, default: 0
end

class PomodoroController < ApplicationController
  DURATION = 25 * 60

  timer :countdown, every: 1, action: :advance

  def show
    render :show, countdown: countdown
  end

  def advance
    pomodoro.elapsed += 1
    return navigate_to("/break") if countdown.expired?

    show
  end

  private

  def countdown
    @countdown ||= Charming::Components::Timer.new(duration: DURATION, label: "remaining")
      .tick(pomodoro.elapsed)
  end

  def pomodoro
    state(:pomodoro, PomodoroState)
  end
end
```

```erb
<%= render_component countdown %>
```

## Options

| Option | Default | What it does |
|---|---|---|
| `duration:` | required | Countdown length in whole seconds. Negative values clamp to 0. |
| `label:` | `nil` | Text appended after the clock. |
| `theme:` | `nil` | Theme passed through to the component base class. |

## Working with it

- `tick(seconds = 1)` — counts down by `seconds`, clamping at zero. Returns `self`.
- `expired?` — true once `remaining` hits zero.
- `reset` — restores the full duration. Returns `self`.
- `remaining` / `duration` / `label` — readers.

## Tips

- Whole seconds only: `tick` converts its argument with `to_i`, so `tick(0.25)` truncates to zero and does nothing. Fire your controller timer at `every: 1` and use the default `tick`, or count sub-second ticks in state and pass whole seconds.
- `duration:` is also truncated to an integer — `duration: 90.5` becomes 90 seconds.
- The renderable holds no wall-clock reference; it only knows what you `tick` into it. If your timer pauses (for example while the terminal is unfocused), the countdown pauses too.

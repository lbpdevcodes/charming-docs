---
title: Animation
layout: default
nav_order: 11.5
permalink: /docs/animation/
---

# Animation

Charming animates with physics rather than easing curves: motion comes from a
damped spring (`Charming::Spring`) or projectile motion (`Charming::Projectile`),
stepped once per frame by an **animation timer** that runs only while something
is moving. Both primitives are ports of [charmbracelet/harmonica](https://github.com/charmbracelet/harmonica),
itself a port of [Ryan Juckett's damped springs](https://www.ryanjuckett.com/damped-springs/).

## The animation tick

Declare an animation on a controller with `animate` — a timer that starts
**stopped**, ticking an action at a frame rate once started:

```ruby
class BallController < ApplicationController
  animate :slide, fps: 60, action: :step_slide
end
```

Three instance helpers control it from actions:

- `start_timer(:slide)` schedules the tick (idempotent while running).
- `stop_timer(:slide)` unschedules it (idempotent when stopped).
- `timer_running?(:slide)` reports whether it is scheduled.

The pattern: a user action starts the timer, and the tick action **stops its own
timer** once the motion settles. While no animation runs, the event loop sits at
its normal idle poll — an app at rest costs nothing, no matter how many
animations it declares.

## Springs

A `Spring` precomputes coefficients for a fixed time step; each update is four
multiply-adds. It is immutable and stateless — your app owns the position and
velocity (as `:float` attributes on a state object) and feeds them back in every
frame:

```ruby
class BallController < ApplicationController
  SPRING = Charming::Spring.new(
    delta_time: Charming.fps(60),
    angular_frequency: 6.0,
    damping_ratio: 0.5
  )

  animate :slide, fps: 60, action: :step_slide
  key "b", :bounce, scope: :content

  def bounce
    ball.target_x = 40.0
    start_timer(:slide)
    render_ball
  end

  def step_slide
    ball.x, ball.velocity = SPRING.update(ball.x, ball.velocity, ball.target_x)
    settle if SPRING.settled?(ball.x, ball.velocity, ball.target_x, epsilon: 0.5)
    render_ball
  end

  private

  def ball
    state(:ball, BallState)
  end

  def settle
    ball.x = ball.target_x
    ball.velocity = 0.0
    stop_timer(:slide)
  end

  def render_ball
    render :show, ball: ball, palette: command_palette
  end
end
```

Tuning the two parameters:

| Parameter | Effect |
|---|---|
| `angular_frequency` | Speed — higher values reach the target faster (default `6.0`). |
| `damping_ratio` `< 1` | Under-damped: overshoots the target and oscillates before settling — bouncy (default `0.5`). |
| `damping_ratio` `= 1` | Critically damped: reaches the target as fast as possible with no overshoot. |
| `damping_ratio` `> 1` | Over-damped: no overshoot, slower approach — sluggish. |

`settled?(position, velocity, target, epsilon: 0.01)` is the stop guard: true
once both the distance to the target and the velocity are within `epsilon`.

{: .note }
> Terminal cells are coarse. On a character grid an `epsilon` around `0.5` (half
> a cell) stops the timer as soon as motion becomes invisible; the default
> `0.01` keeps ticking sub-cell motion you cannot see.

Match `delta_time` to the tick rate: a spring built with `Charming.fps(60)`
belongs on an `animate ..., fps: 60` timer. Animate several values by calling
`update` once per value with the same spring — the coefficients are shared.

## Projectiles

`Charming::Projectile` is simple game physics: a position advanced by a
velocity, and a velocity advanced by an acceleration (usually gravity), one
fixed step per frame:

```ruby
def launch
  session[:ball] = nil
  start_timer(:fall)
  render_game
end

def step_fall
  ball = session[:ball] ||= Charming::Projectile.new(
    delta_time: Charming.fps(60),
    position: Charming::Projectile::Point.new(x: 2.0, y: 1.0),
    velocity: Charming::Projectile::Vector.new(x: 18.0, y: 0.0),
    acceleration: Charming::Projectile::TERMINAL_GRAVITY
  )
  position = ball.update
  stop_timer(:fall) if position.y > screen.height
  render_game
end
```

Two gravity constants cover both coordinate conventions:

- `Charming::Projectile::TERMINAL_GRAVITY` — `(0, 9.81, 0)`, for terminal
  coordinates where the origin is top-left and y grows downward. This is the one
  you usually want.
- `Charming::Projectile::GRAVITY` — `(0, -9.81, 0)`, for a bottom-left origin.

Unlike `Spring`, a `Projectile` is stateful: it holds its own position and
velocity, and `update` mutates them (position moves by the current velocity
*before* the velocity accelerates). `Point` and `Vector` take `x:`/`y:` (and an
optional `z:`, defaulting to `0.0`).

## Where animation state lives

Controllers are ephemeral — a fresh instance is built per event — so spring
positions and velocities belong in an `ApplicationState` object retrieved
through `state(:name, StateClass)`:

```ruby
class BallState < ApplicationState
  attribute :x, :float, default: 2.0
  attribute :velocity, :float, default: 0.0
  attribute :target_x, :float, default: 2.0
end
```

{: .important }
> Session values must survive a JSON round-trip when the session persists.
> Floats are safe; live `Projectile` objects are not — keep them in
> non-persisted session slots (as above) or store their `x`/`y`/velocity floats
> in state and rebuild.

## Pausing when the terminal blurs

Pair animation timers with [terminal focus]({{ '/docs/controllers-and-views/' | relative_url }}#terminal-focus):
stop the timer in `focus_changed` when `event.focused?` is false, and restart it
on focus gain if the motion had not settled — cleaner than guarding every tick.

## Testing animations

Inject a scripted clock and assert on rendered frames — see
[Testing]({{ '/docs/testing/' | relative_url }}#timer-specs). To prove an
animation stops itself, keep feeding `nil` input events while the clock
advances after the motion settles: no new frames should appear.

```ruby
t = 0.0
clock = -> { t += 1.0 / 120 }
runtime = Charming::Runtime.new(app, backend: backend, clock: clock)
```

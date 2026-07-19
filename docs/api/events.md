---
title: Events, Tasks & Media
layout: default
parent: API Reference
nav_order: 7
permalink: /docs/api/events/
---
# Events, Tasks & Media

Everything that *happens* in a Charming app arrives as an event object — a key press, a resize, a timer tick, a finished task. This page lists the event classes and the machinery behind two event sources: the async **task executors** that run background work, plus the **animation** physics primitives and the **audio** player that typically drive timer- and task-based updates.

## Events

Runtime events include:

- `Charming::Events::KeyEvent` — `key`, `char`, `ctrl`, `alt`, `shift`
- `Charming::Events::ResizeEvent` — `width`, `height`
- `Charming::Events::MouseEvent` — `button`, `x`, `y`, modifiers (`ctrl`/`alt`/`shift`), `click?`/`scroll?`/`release?`/`drag?`/`motion?` (hover motion requires `mouse_motion :all` on the application)
- `Charming::Events::TimerEvent` — `name`, `now`
- `Charming::Events::TaskEvent` — `name`, `value`, `error`, `error?`
- `Charming::Events::TaskProgressEvent` — `name`, `current`, `total`, `message`, `fraction`
- `Charming::Events::PasteEvent` — `text` (bracketed paste)
- `Charming::Events::FocusEvent` — `focused?` (terminal focus reporting; dispatched to an optional `focus_changed` action)

Use `Charming.key_of(event)` when component code needs the normalized key symbol.

## Tasks

Async work submitted with `run_task` runs on an executor:

- `Charming::Tasks::ThreadedExecutor` — the default; one thread per task, name-based cancellation, graceful shutdown.
- `Charming::Tasks::InlineExecutor` — synchronous; for deterministic tests.
- `Charming::Tasks::Progress` — the reporter passed to task blocks: `progress.report(current, of: nil, message: nil)`.
- `Charming::Tasks::Cancelled` — raised inside a task by `cancel_task` or a `timeout:`; arrives as the `TaskEvent#error`.

Custom executors implement `submit(name, timeout: nil, &block)` (plain `submit(name, &block)` works until timeouts are used), plus optional `cancel(name)` and `shutdown(timeout:)`.

## Animation

Physics primitives for spring and projectile motion — see [Animation]({{ '/docs/animation/' | relative_url }}) for the full guide.

- `Charming.fps(n)` — the seconds-per-frame delta for a frame rate (`1.0 / n`).
- `Charming::Spring.new(delta_time:, angular_frequency: 6.0, damping_ratio: 0.5)` — immutable damped harmonic oscillator with precomputed coefficients.
- `Spring#update(position, velocity, target)` — returns `[new_position, new_velocity]` advanced one frame.
- `Spring#settled?(position, velocity, target, epsilon: 0.01)` — true once motion is effectively at rest (the `stop_timer` guard).
- `Charming::Projectile.new(delta_time:, position:, velocity:, acceleration:)` — mutable semi-implicit Euler motion; `#update` advances one frame and returns the new `Point`; readers `position`, `velocity`, `acceleration`.
- `Charming::Projectile::Point` / `Vector` — `x:`, `y:`, optional `z:` (default `0.0`).
- `Charming::Projectile::TERMINAL_GRAVITY` (top-left origin) and `GRAVITY` (bottom-left origin).

## Audio

`Charming::Audio::Player` plays a sound file by shelling out to a system audio binary. See [Audio]({{ '/docs/audio/' | relative_url }}) for the lifecycle and `run_task` pattern.

- `Player.new(system: Charming::Audio::System.new)` — the `system:` adapter is injectable so specs never shell out.
- `play(path)` — stops any current sound, spawns the resolved backend, returns the child PID; raises `Charming::Audio::Player::Unavailable` when no backend is installed.
- `stop` — terminates and reaps the current child (no-op when idle).
- `playing?` — true while the spawned sound is still running.
- `wait` — blocks until the current sound finishes, then clears it (drive from a background task).
- `available?` — true when a backend binary is installed for this platform.

Backends resolve in order: `ffplay` (any platform), then `afplay` (macOS), `paplay`/`mpg123`/`aplay` (Linux). `Charming::Audio::System` wraps `Process`/`ENV`/`RbConfig` and exposes `macos?`, `linux?`, `which?(command)`, `spawn(argv)`, `terminate(pid)`, `alive?(pid)`, and `wait(pid)`.

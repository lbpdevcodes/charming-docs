---
title: Audio
layout: default
nav_order: 13
parent: Docs
permalink: /docs/audio/
---
# Audio

Charming can play sound files by shelling out to a system audio binary — no extra
gem dependency. The playback engine lives in the `Charming::Audio` namespace; a
small `Charming::Components::Audio` view renders a one-line playback indicator.

## The Player

`Charming::Audio::Player` plays a single sound file by spawning a system audio
binary as a detached child process and tracking it:

```ruby
player = Charming::Audio::Player.new
player.play("assets/chime.wav")   # spawns the backend, returns the child PID
player.playing?                   # => true while the sound is still going
player.stop                       # terminates and reaps the child (safe if idle)
```

A backend binary is resolved on first use, in priority order:

| Backend | Platform | Notes |
|---------|----------|-------|
| `ffplay` | any | from ffmpeg; run with `-nodisp -autoexit -loglevel quiet` |
| `afplay` | macOS | |
| `paplay` | Linux | PulseAudio/PipeWire |
| `mpg123` | Linux | run with `-q` |
| `aplay`  | Linux | ALSA, run with `-q` |

If none are installed, `play` raises `Charming::Audio::Player::Unavailable` (a
`Charming::Error`) with a message listing the backends it searched. Probe first
with `available?` to degrade gracefully instead of rescuing:

```ruby
player.play(path) if player.available?   # skip the chime when no backend exists
```

### Player API

| Method | Behavior |
|--------|----------|
| `play(path)` | Stops any sound in progress, spawns the resolved backend, returns the child PID. Raises `Unavailable` when no backend is installed. |
| `stop` | Terminates and reaps the current child. No-op when nothing is playing. |
| `playing?` | True while the spawned sound is still running. |
| `wait` | Blocks until the current sound finishes, then clears it. Drive this from a background task, not the event loop. |
| `available?` | True when a backend binary is installed for this platform. |

## Playing without blocking the event loop

The player holds a **live child process**, so it is not a view object — keep it in
`session` rather than rebuilding it per render:

```ruby
session[:audio] ||= Charming::Audio::Player.new
```

`play` returns immediately, so a fire-and-forget chime can go straight in an
action. For a sound you want to track to completion — and tear down cleanly when
the app quits — drive it from a controller `run_task`: spawn, `wait`, and reap in
an `ensure`:

```ruby
class JukeboxController < Charming::Controller
  key "p", :play_track
  on_task :audio, action: :track_finished

  def play_track
    player = (session[:audio] ||= Charming::Audio::Player.new)
    run_task(:audio) do
      player.play(track_path)
      player.wait        # blocks the task thread until the sound ends
    ensure
      player.stop        # no-op on normal finish; reaps the child if shutdown interrupts the task
    end
    render :show
  end

  def track_finished
    render :show   # the TaskEvent arrives when the sound finishes
  end
end
```

If the task thread is killed mid-`wait` (for example on app shutdown), the player
keeps its PID so the `ensure player.stop` can still reap the child.

## The Audio component

`Charming::Components::Audio` is an optional one-line status indicator. It reads a
player's `playing?` state and renders `▶` while playing or `■` when stopped, with
an optional label:

```erb
<%= render_component Charming::Components::Audio.new(
  player: audio,
  label: "chime.wav",
  theme: theme
) %>
```

```ruby
# playing   => "▶ chime.wav"
# stopped   => "■ chime.wav"
# no label  => "▶"
```

The component only displays state — it never starts or stops playback. Pair it
with a controller `timer` (or re-render on the `on_task` completion) so the glyph
updates as the sound starts and finishes.

## Testing

`Player` talks to the OS through an injectable adapter, `Charming::Audio::System`,
which wraps Ruby's `Process`/`ENV`/`RbConfig`. Inject a fake so specs never shell
out, touch the real process table, or play sound:

```ruby
class FakeAudioSystem
  attr_reader :spawned

  def initialize(available: [], macos: false, linux: false)
    @available, @macos, @linux = available, macos, linux
    @spawned = []
  end

  def macos? = @macos
  def linux? = @linux
  def which?(command) = @available.include?(command)
  def spawn(argv) = @spawned.push(argv) && 1234
  def terminate(_pid) = nil
  def alive?(_pid) = false
  def wait(_pid) = nil
end

system = FakeAudioSystem.new(available: %w[ffplay], macos: true)
player = Charming::Audio::Player.new(system: system)

player.play("song.wav")
expect(system.spawned.last).to eq(["ffplay", "-nodisp", "-autoexit", "-loglevel", "quiet", "song.wav"])
```

For the component, stub the player's `playing?`:

```ruby
player = instance_double(Charming::Audio::Player, playing?: true)
expect(Charming::Components::Audio.new(player: player, label: "song.wav").render).to eq("▶ song.wav")
```

---
title: Audio
layout: default
parent: Components
nav_order: 31
permalink: /docs/components/audio/
---
# Audio

`Charming::Components::Audio` is a one-line playback-status indicator for a `Charming::Audio::Player`. It is a static renderable: it reads the player's `playing?` state and renders `▶` (playing) or `■` (stopped), followed by an optional label. This page covers just the indicator — the player itself (spawning sound, sessions, testing) lives in the [Audio]({{ '/docs/audio/' | relative_url }}) docs.

The component only displays state; note that `Charming::Components::Audio` (the view component) is distinct from the `Charming::Audio` namespace (the playback engine).

```text
▶ rain.wav
```

## Quick start

```ruby
class ShowView < Charming::View
  def render
    render_component(Charming::Components::Audio.new(
      player: assigns.fetch(:player),
      label: "rain.wav",
      theme: theme
    ))
  end
end
```

Keep the player in `session` (see [Audio]({{ '/docs/audio/' | relative_url }})) and pair the indicator with a controller timer or `on_task` re-render so the glyph flips when playback starts or stops.

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `player:` | required | The `Charming::Audio::Player` whose state is shown. |
| `label:` | `nil` | Optional suffix after the glyph (e.g. the track name). |
| `theme:` | `nil` | Forwarded to the view layer. |

## Tips

- The component reads `playing?` only at render time — nothing re-renders on its own. Without a timer or task-driven re-render, the glyph goes stale when the track ends.

---
title: Viewport
layout: default
parent: Components
nav_order: 9
permalink: /docs/components/viewport/
---
# Viewport

`Charming::Components::Viewport` is a scrollable window over multi-line content — long documents, logs, rendered Markdown. It clips lines with ANSI awareness, so styled text survives horizontal scrolling. It is interactive: put it in a focus ring and it handles scroll keys, page keys, and the mouse wheel.

```text
…                                    ← rows above the offset are hidden
Entry body line 4
Entry body line 5
Entry body line 6                    ← height: 3 shows this window
…                                    ← rows below are hidden
```

## Quick start

The journal example's reader scrolls an entry body this way — persist the offset in state and pass it back on construction (`reader_controller.rb`):

```ruby
class ReaderController < ApplicationController
  focus_ring :body

  def show
    entry_state.scroll_offset = body.offset
    render :show, body_pane: body
  end

  # The markdown body in a scrollable viewport (j/k/arrows/page keys via focus).
  def body
    @body ||= Charming::Components::Viewport.new(
      content: rendered_markdown,
      width: [screen.width - 32, 40].max,
      height: [screen.height - 12, 5].max,
      offset: entry_state.scroll_offset,
      wrap: true
    )
  end

  private

  def entry_state
    state(:"entry_#{params[:id]}", EntryState)
  end
end
```

```erb
<%= render_component body_pane %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `content:` | (required) | A string, an array of lines, or any object responding to `render`. |
| `width:` | `nil` | Visible column count; enables horizontal clipping (and wrapping with `wrap:`). |
| `height:` | `nil` | Visible row count; short content is padded to this height. Required for mouse support. |
| `offset:` | `0` | The top-visible row, clamped into range. |
| `column:` | `0` | The left-visible column, clamped into range. |
| `wrap:` | `false` | Soft-wrap long lines to `width:` instead of clipping (disables horizontal scroll). |
| `keymap:` | `:vim` | Keybinding style — `:vim` adds `h`/`j`/`k`/`l` aliases; pass `nil` to disable. |
| `follow:` | `false` | Pin the viewport to the bottom of the content regardless of `offset:` — for tailing logs. |

## Keyboard

| Key | Action |
|-----|--------|
| `up` / `k` | Scroll up one row. |
| `down` / `j` | Scroll down one row. |
| `left` / `h` | Scroll one column left. |
| `right` / `l` | Scroll one column right. |
| `page_up` / `page_down` | Scroll by one viewport page. |
| `home` | Jump to the top-left of the content. |
| `end` | Jump to the bottom-right. |

`h`/`j`/`k`/`l` come from the default `keymap: :vim`.

## Mouse

Requires `height:`. The scroll wheel moves the offset one row per notch; a click moves the top-visible row down by the clicked row's position within the window.

## What it returns

The viewport is navigation-only — it returns `:handled` for consumed scroll keys and mouse events, and `nil` otherwise. It never produces `[:selected, ...]`, so no selection hooks fire; your controller just re-renders with the new offset. See the shared [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `offset` / `column` — the current top-visible row and left-visible column; read these after dispatch to persist scroll position.
- `at_bottom?` — true when the last content row is visible; pair with `follow:` to decide whether to keep tailing.

## Tips

- **Components are rebuilt on every event** — persist `offset` (and `column`, if you scroll horizontally) into controller state during `show` and pass them back in on construction, exactly as the quick start does. Otherwise every keypress snaps back to the top.
- For log tails, construct with `follow: state.following` and after each dispatch store `state.following = viewport.at_bottom?` — the viewport sticks to new content until the user scrolls up, and resumes when they scroll back to the bottom.
- `wrap: true` needs a positive `width:` and turns off horizontal scrolling (`max column` becomes 0) — wrapped content has nothing to clip sideways.
- `content:` accepts a renderable (e.g. a `Markdown` component), which is rendered and then windowed — handy for scrollable docs and help screens.

---
title: Modal
layout: default
parent: Components
nav_order: 15
permalink: /docs/components/modal/
---
# Modal

Modal is a centered, boxed overlay with an optional title and help line around a body — the frame for confirmations, help screens, and pickers. The body can be a string, a View, or another Component. It is mostly a static frame, but it becomes lightly interactive when you give it `max_body_height:` and the body is taller: `handle_key` then scrolls the body and everything else falls through.

```text
╭──────────────────────────────────────────────────╮
│                                                  │
│               Delete entry                       │
│                                                  │
│  Delete "Morning pages"?                         │
│                                                  │
│  This cannot be undone.                          │
│                                                  │
│  y deletes · n or esc cancels                    │
│                                                  │
╰──────────────────────────────────────────────────╯
```

## Quick start

Modals are typically wrapped in a small component that owns the keyboard decisions, then composited as a layout overlay. From the journal example's delete confirmation:

```ruby
class DeleteConfirm < Charming::Component
  def initialize(entry_title:, theme: nil)
    super(theme: theme)
    @entry_title = entry_title
  end

  def handle_key(event)
    case Charming.key_of(event)
    when :y then [:submitted, true]
    when :n, :escape then :cancelled
    else :handled
    end
  end

  def render
    render_component(
      Charming::Components::Modal.new(
        content: "Delete \"#{@entry_title}\"?\n\nThis cannot be undone.",
        title: "Delete entry",
        help: "y deletes · n or esc cancels",
        width: 46,
        theme: theme
      )
    )
  end
end
```

The layout floats it above the panes with `overlay`:

```ruby
screen_layout(background: theme.background) do
  split(:vertical) { ... }

  overlay delete_modal, z_index: 5 if delete_modal
end
```

The controller routes keys to it by pushing a focus scope while it is open: `focus.push_scope([:delete_confirm], origin: :modal)`, and pops the scope when the `delete_confirm_submitted` / `delete_confirm_cancelled` hooks fire.

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `content:` | required | The body — a string, or anything responding to `render` (View or Component). |
| `title:` | `nil` | Centered heading at the top, in `theme.title`. |
| `help:` | `nil` | Muted footer line rendered below the body. |
| `width:` | `52` | Content width. The theme's border and padding add columns on top (6 with the default `theme.modal`). |
| `max_body_height:` | `nil` | Caps the visible body rows. A taller body is windowed through a Viewport and becomes scrollable. |
| `scroll_offset:` | `0` | Restores a previous scroll position. |
| `style:` | `nil` | Replaces the default `theme.modal` style (rounded border, padding) entirely. |
| `theme:` | app default | Theme used for the frame, title, and help styles. |

## Keyboard

Only active when `max_body_height:` is set and the body is taller than it:

| Key | Behavior |
|-----|----------|
| `Up` / `Down` | Scroll one line. |
| `PageUp` / `PageDown` | Scroll one window. |
| `Home` / `End` | Jump to top / bottom. |

When the body fits (or no `max_body_height:` was given), `handle_key` returns `nil` for every key, so your wrapping component or controller sees them.

## What it returns

`handle_key` returns `:handled` when a key moved the viewport, `nil` otherwise. See the [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `scroll_offset` — reader for the body's current scroll position (only meaningful with `max_body_height:`).

## Tips

- Components are rebuilt on every event, so a scroll position does not survive on its own. Read `modal.scroll_offset` after dispatching a key, stash it in `session`, and pass it back as `scroll_offset:` when you rebuild the modal.
- The `help:` line renders as a footer below the body — the stacking order is title, content, help. It stays put when a `max_body_height:` body scrolls.
- `style:` replaces `theme.modal` wholesale; set your own border, padding, and width on it if you use it.

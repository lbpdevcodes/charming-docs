---
title: HelpOverlay
layout: default
parent: Components
nav_order: 17
permalink: /docs/components/help-overlay/
---
# HelpOverlay

HelpOverlay is the classic `?` help screen: a controller's key bindings rendered as an aligned key/description cheat sheet inside a [Modal]({{ '/docs/components/modal/' | relative_url }}). It is interactive in the simplest possible way — any key dismisses it.

```text
╭────────────────────────────────────────────────╮
│                                                │
│           Keyboard Shortcuts                   │
│                                                │
│  ctrl+p  Command palette                       │
│  ?       Help                                  │
│  q       Quit                                  │
│                                                │
╰────────────────────────────────────────────────╯
```

## Quick start

Build it straight from a controller class with `for_controller`, which reads the class's `key_bindings` and humanizes the action names (`open_command_palette` → "Open command palette"). From the journal example:

```ruby
class ApplicationController < Charming::Controller
  key "?", :open_help, scope: :global

  def open_help
    session[:help_open] = true
    focus.push_scope([:help_overlay], origin: :modal)
    render_default_action
  end

  def help_overlay
    Charming::Components::HelpOverlay.for_controller(self.class, theme: theme)
  end

  def help_overlay_cancelled
    session.delete(:help_open)
    focus.pop_scope
    render_default_action
  end
end
```

The layout composites it as an overlay while it is open:

```ruby
overlay help_modal, z_index: 5 if help_modal
```

```ruby
def help_modal
  return unless controller.session[:help_open]

  render_component controller.help_overlay
end
```

Or list entries explicitly:

```ruby
Charming::Components::HelpOverlay.new(
  bindings: {"q" => "Quit", "ctrl+p" => "Command palette"}
)
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `bindings:` | required | Hash of key name (symbol or string) → description string. Empty hash renders "No key bindings". |
| `title:` | `"Keyboard Shortcuts"` | Modal title. |
| `width:` | `44` | Modal content width. |
| `theme:` | app default | Theme for the modal frame and key styling. |

`for_controller(controller_class, title:, theme:)` accepts the same `title:` and `theme:` and derives `bindings:` from the class's `key_bindings`.

## Keyboard

Any key dismisses the overlay. It also declares `captures_text?` as true, so while it is focused, printable characters (including `q` and `?`) reach it instead of triggering global bindings.

## What it returns

`handle_key` always returns `:cancelled`, which dispatches to your `<slot>_cancelled` controller hook. See the [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `HelpOverlay.for_controller(controller_class, ...)` — the usual constructor; keeps the cheat sheet in sync with `key` declarations for free.

## Tips

- Push a focus scope (`focus.push_scope([:help_overlay], origin: :modal)`) when opening it so every key routes to the overlay, and pop it in the `_cancelled` hook — otherwise the dismissing key also hits whatever was focused underneath.
- `for_controller` only shows bindings declared with `key` on that class (and its ancestors); descriptions are just humanized action names, so name actions readably.

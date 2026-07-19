---
title: Toast
layout: default
parent: Components
nav_order: 16
permalink: /docs/components/toast/
---
# Toast

Toast is a small notification box, usually composited as an overlay anchored near a screen corner: "Saved!", "Export failed", and friends. It is a static renderable — it just draws the styled box. Auto-dismissal is your controller's job, paired with a timer.

```text
╭───────────────────────╮
│ Saved "Morning pages" │
╰───────────────────────╯
```

The border color follows `kind:` — cyan for `:info`, green for `:success`, yellow for `:warn`, red for `:error`.

## Quick start

The journal example keeps toast state in `session` and lets the layout render it as an overlay. In the controller:

```ruby
class ApplicationController < Charming::Controller
  timer :toast_expiry, every: 0.5, action: :expire_toast

  def show_toast(message, kind: :success)
    session[:toast] = {message: message, kind: kind, expires_at: Time.now.to_f + 2.5}
  end

  def expire_toast
    toast = session[:toast]
    return unless toast
    return if Time.now.to_f < toast[:expires_at]

    session.delete(:toast)
    render_default_action
  end
end
```

In the layout, float it above everything else:

```ruby
screen_layout(background: theme.background) do
  split(:vertical) { ... }

  overlay toast, top: screen.height - 5, left: :center, z_index: 10 if toast
end
```

```ruby
def toast
  toast_state = controller.session[:toast]
  return unless toast_state

  render_component Charming::Components::Toast.new(
    message: toast_state[:message],
    kind: toast_state.fetch(:kind, :info),
    theme: theme
  )
end
```

Then any action can call `show_toast("Deleted \"#{entry.title}\"", kind: :warn)`.

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `message:` | required | The toast text. |
| `kind:` | `:info` | Accent color: `:info` (cyan), `:success` (green), `:warn` (yellow), `:error` (red). Unknown kinds fall back to `:info`. |
| `width:` | `nil` | Fixes the content width; border and padding add 4 more columns. Without it, the box hugs the message. |
| `theme:` | app default | Theme used for the base style. |

## Working with it

- `message` and `kind` — readers, handy in specs and layout logic.

## Tips

- The component never dismisses itself. The lifetime lives in your controller: store the toast with a deadline, run a timer, delete it when the deadline passes (the pattern above).
- Keep toast state in `session`, not instance variables — components and controllers are rebuilt on every event.

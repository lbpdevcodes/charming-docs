---
title: ErrorScreen
layout: default
parent: Components
nav_order: 21
permalink: /docs/components/error-screen/
---
# ErrorScreen

ErrorScreen renders an unhandled exception as a styled, centered panel instead of letting a raw backtrace crash into the terminal. The runtime builds it for you: when a dispatched action raises and no `rescue_from` handler claims the exception, this is the panel you see. It is a static renderable, and you rarely construct it by hand — but you can, for custom error handling.

```text
╭────────────────────────────────────────────────────────╮
│                                                        │
│  ArgumentError                                         │
│                                                        │
│  unknown entry id 42                                   │
│                                                        │
│  app/controllers/entries_controller.rb:12:in 'show'    │
│                                                        │
│  press any key to continue · q to quit                 │
│                                                        │
╰────────────────────────────────────────────────────────╯
```

The exception class renders in `theme.warn` bold, the message wraps to the panel width, and up to 6 backtrace lines follow with the app root stripped — all inside a red rounded border.

## Quick start

You normally never write this — the runtime does. To render one yourself, for example from a `rescue_from` handler that wants the standard panel with extra behavior around it:

```ruby
class ApplicationController < Charming::Controller
  rescue_from StandardError, with: :show_error

  def show_error(error)
    log_somewhere(error)
    render Charming::Components::ErrorScreen.new(
      error: error,
      width: 64,
      theme: theme
    ).render
  end
end
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `error:` | required | The rescued exception. Class name, message, and backtrace are read from it. |
| `width:` | `64` | The panel's total width; the message wraps and backtrace lines clip to fit inside it. |
| `root:` | `Dir.pwd` | App root stripped from the front of backtrace paths to keep them short. |
| `theme:` | app default | Theme for the warn/muted styles. |

## Working with it

- The panel shows at most 6 backtrace lines (the top of the trace); the full backtrace still goes to the log when the runtime renders it.
- A missing backtrace (e.g. an exception you built but never raised) simply omits that section.

## Tips

- If you handle an exception with `rescue_from`, the runtime does *not* render this panel — your handler owns the response entirely, including rendering something (or this component) itself.
- `root:` matters when your app doesn't run from its own directory — pass the app root so backtrace lines stay readable at the panel width.

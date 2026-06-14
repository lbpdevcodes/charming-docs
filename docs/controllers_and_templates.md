---
title: Controllers & Views
layout: default
nav_order: 5
permalink: /docs/controllers-and-views/
---
# Controllers & Views

Controllers dispatch actions, bind input, navigate between routes, run background tasks, and render responses.

## Controller Actions

Generated controllers inherit from the app's `ApplicationController`:

```ruby
module MyApp
  class HomeController < ApplicationController
    def show
      render :show, home: home, palette: command_palette
    end

    private

    def home
      state(:home, HomeState)
    end
  end
end
```

The controller's private `home` method returns the screen state object. `render :show, home: home` passes that object into the view as the `home` assign.

`render :show` resolves the conventional Ruby view class from the controller/action name, not from the state class:

```text
app/views/home/show_view.rb
```

## Rendering

Controller render forms:

| Form | Behavior |
|------|----------|
| `render :show, **assigns` | Renders `AppName::Home::ShowView`, falling back to `app/views/home/show.tui.erb` or `.txt.erb`. |
| `render_template "custom/page", **assigns` | Renders an explicit ERB template path under `app/views`. |
| `render "literal text"` | Renders a literal string. |
| `render view_object` | Renders a class-based view or component object. |

Assigns passed to `render` become methods in the view. In `home: home`, the left `home` is the assign name and the right `home` calls the controller method:

```ruby
render :show, home: home, palette: command_palette
```

```ruby
module MyApp
  module Home
    class ShowView < Charming::View
      def render
        text home.title, style: theme.title
      end
    end
  end
end
```

Because the controller passed `home: home`, the view can call `home.title`. That title comes from the `HomeState` object returned by the controller's `home` method.

Views also receive `screen`, `controller`, and `theme` assigns.

## ERB Template Fallback

ERB templates remain available for large text rendering or simple fallback views.

If no `Home::ShowView` class exists, `render :show` searches:

```text
app/views/home/show.tui.erb
app/views/home/show.txt.erb
```

`.tui.erb` is preferred before `.txt.erb`. `render_template` always renders an explicit template path.

## Template Helpers

Views and templates share the same helper set:

| Helper | Purpose |
|--------|---------|
| `text(value, style: nil)` | Render styled text. |
| `box(value, style: nil)` | Style or border a block. |
| `row(*items, gap: 0)` | Join blocks side by side. |
| `column(*items, gap: 0)` | Stack blocks vertically. |
| `screen_layout { ... }` | Build a full-screen declarative layout tree. |
| `render_component(component)` | Render a component or partial object. |
| `render_partial(partial)` | Alias for `render_component`. |
| `yield_content` | Render wrapped screen content in a layout. |
| `focused?(slot)` | Ask the controller whether a focus slot is active. |

## Action Hooks

Controllers support Rails-style lifecycle hooks. Hooks are inherited by subclasses
and accept `only:` / `except:` action filters:

```ruby
class EntryController < ApplicationController
  before_action :load_entry, only: [:show, :edit]
  after_action  :track_view
  around_action :measure

  rescue_from ActiveRecord::RecordNotFound, with: :entry_missing

  def show
    render :show, entry: @entry
  end

  private

  def load_entry
    @entry = Entry.find(params.fetch(:id))
  end

  def measure
    started = Time.now
    yield
    logger.info("rendered in #{Time.now - started}s")
  end

  def entry_missing(_error)
    render "Entry not found."
  end
end
```

`around_action` methods must `yield` to invoke the action. `rescue_from` picks the
handler whose rescued class is **most specific** for the raised exception (ties go to
the last registered handler) — deliberately less surprising than Rails' pure
declaration-order rule. Unmatched exceptions propagate to the runtime, which renders
the built-in error screen instead of crashing the terminal.

Hooks run around *actions* (`dispatch(:show)`, timers, task handlers). They do not run
on component key dispatch, so any method the focus system can call directly — focus
slot component builders, component result hooks — must load its own data instead of
relying on `before_action` instance variables.

## Key Bindings

Bind key events to controller actions:

```ruby
key "up", :increment
key "down", :decrement
key "q", :quit, scope: :global
```

Content-scoped bindings only run when content focus is active. Global bindings run from any focused pane.

### Dispatch priority

Key events flow through this chain, first match wins:

1. **Command palette** (when open) consumes everything.
2. **Text capture** — when the focused component accepts free-typed text (TextInput,
   TextArea, Form, Autocomplete…), printable characters go to it *before any binding*.
   Typing `q` or `?` into a field inserts the character; `ctrl`/`alt` shortcuts are
   unaffected.
3. **Global key bindings.**
4. **Overlay scopes** — a pushed modal focus scope captures all remaining keys.
5. **Sidebar keys** (when the sidebar is focused).
6. **Content key bindings.**
7. **The focused component** — text-capturing components also see `tab` here (so forms
   do field navigation); for all other components, `tab` cycles the controller's focus
   ring first.

A practical consequence: printable global keys like `q` only fire when no text field
is focused — your users can type the letter q.

## Help Overlay

`HelpOverlay.for_controller` turns a controller's key bindings into a `?`-style
cheat-sheet modal. Pair it with a pushed focus scope so the overlay captures the
dismissing keypress:

```ruby
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
```

The layout renders the overlay while `session[:help_open]` is set. Any key dismisses
it (the component returns `:cancelled`, which calls the `<slot>_cancelled` hook).

## Command Palette

Add commands with a method name or block:

```ruby
command "Refresh", :refresh

command "Home" do
  navigate_to "/"
end
```

Generated apps bind `ctrl+p` globally to `open_command_palette` and include commands for theme switching and quit. The palette filters with fuzzy matching — type any subsequence of a command label (`otp` finds "Open theme palette").

## Navigation And Quit

Use controller helpers to change app flow:

```ruby
navigate_to "/settings"
quit
```

These produce `Charming::Response` objects that the runtime interprets.

## Timers

Timers dispatch periodically while the current route is active:

```ruby
timer :clock, every: 0.5, action: :tick

def tick
  clock.now = Time.now
  show
end
```

`every:` must be positive — a zero or negative interval raises `ArgumentError` at
declaration time (it would busy-loop the runtime).

## Terminal Focus

When the terminal window gains or loses focus, the runtime dispatches an optional
`focus_changed` action (it enables focus reporting mode automatically). Define the
action to react — a common use is pausing a timer-driven clock or animation while the
window is in the background so it isn't burning CPU:

```ruby
class DashboardController < Charming::Controller
  timer :clock, every: 1, action: :tick

  def focus_changed
    dashboard.active = event.focused?   # event is a Charming::Events::FocusEvent
    show
  end

  def tick
    return show unless dashboard.active
    dashboard.now = Time.now
    show
  end
end
```

`event.focused?` is `true` on focus gain and `false` on blur. Controllers that don't
define `focus_changed` simply ignore the event.

## Background Tasks

Register a task handler and dispatch work with `run_task`:

```ruby
on_task :refresh_home, action: :refresh_loaded

def refresh
  run_task(:refresh_home) { "Loaded" }
  show
end

def refresh_loaded
  home.status = event.error? ? event.error.message : event.value
  show
end
```

Task results arrive as `Charming::Events::TaskEvent` with `value`, `error`, and `error?`.

### Progress streaming

Blocks that accept an argument receive a progress reporter. Each `report` dispatches
the matching `on_task_progress` handler with a `TaskProgressEvent` (`current`,
`total`, `message`, and a `fraction` helper):

```ruby
on_task :export, action: :export_finished
on_task_progress :export, action: :export_progressed

def start_export
  run_task(:export) do |progress|
    entries.each_with_index do |entry, index|
      write(entry)
      progress.report(index + 1, of: entries.length, message: entry.title)
    end
    entries.length
  end
  show
end

def export_progressed
  stats.export_current = event.current
  show
end
```

### Timeouts and cancellation

```ruby
run_task(:fetch, timeout: 10) { slow_api_call }
cancel_task(:fetch)
```

Both deliver a `TaskEvent` whose `error` is a `Charming::Tasks::Cancelled`, so the
`on_task` handler can distinguish cancellation from real failures with
`event.error.is_a?(Charming::Tasks::Cancelled)`.

Custom executors implement `submit(name, timeout: nil, &block)` (the keyword is only
passed when a timeout is requested, so the simple `submit(name, &block)` signature
keeps working), plus optional `cancel(name)` and `shutdown(timeout:)`.

## Class-Based Views

Generated apps use templates by default, but class-based views still work:

```ruby
class HomeView < Charming::View
  def render
    text title, style: theme.title
  end
end
```

```ruby
render HomeView.new(title: "Home", theme: theme)
```

---
title: Core Concepts
layout: default
nav_order: 3
parent: Docs
permalink: /docs/core-concepts/
---
# Core Concepts

Charming is a Rails-inspired framework for terminal apps. Generated apps use routes, controllers, state objects, templates, layouts, components, themes, and a runtime that talks to a terminal backend.

## Architecture

Generated apps follow this flow:

```text
Application -> Router -> Controller -> Template/Layout -> Component -> UI
                         Runtime -> Renderer -> Terminal Backend
```

At runtime, the flow for a screen is:

```text
Route -> Controller action -> Template -> Layout -> Renderer -> Terminal frame
```

## Applications

An application owns the route table, session, themes, and task executor. Generated apps define the application class in `lib/my_app/application.rb`:

```ruby
module MyApp
  class Application < Charming::Application
    root File.expand_path("../..", __dir__)

    Charming::UI::Theme.built_in_names.each do |theme_name|
      theme theme_name.to_sym, built_in: theme_name
    end

    default_theme :phosphor
  end
end
```

Routes are usually defined separately in `config/routes.rb`:

```ruby
MyApp::Application.routes do
  root "home#show"
end
```

## Controllers Are Ephemeral

Charming creates a fresh controller instance for each dispatch. A controller handles one action or event and returns a response.

Do not store durable state in controller instance variables. This is wrong for state that must survive multiple key presses:

```ruby
def increment
  @count ||= 0
  @count += 1
  render "Count: #{@count}"
end
```

Store durable state in an application state object instead:

```ruby
def increment
  counter.count += 1
  render "Count: #{counter.count}"
end

private

def counter
  state(:counter, CounterState)
end
```

`Controller#state` stores the state object in the application session and returns the same object on later dispatches.

## Views And Layouts

Generated controllers render views by symbol:

```ruby
def show
  render :show, home: home, palette: command_palette
end
```

For `HomeController`, `render :show` resolves:

```text
app/views/home/show_view.rb
```

Layouts wrap rendered views. Generated apps use a Ruby layout class:

```ruby
class ApplicationController < Charming::Controller
  layout Layouts::ApplicationLayout
end
```

That resolves:

```text
app/views/layouts/application_layout.rb
```

ERB templates remain available as a fallback. See [Controllers & Views]({{ '/docs/controllers-and-views/' | relative_url }}) and [Layouts]({{ '/docs/layouts/' | relative_url }}) for details.

## Runtime

Most apps start through:

```ruby
Charming.run(MyApp::Application.new)
```

The runtime:

1. Enters the terminal alternate screen and enables mouse tracking, bracketed paste, and focus reporting
2. Resolves the root route
3. Dispatches controller actions and events
4. Renders responses through a renderer
5. Reads key, mouse, paste, resize, timer, and task events
6. Restores terminal state on quit or error

### Key dispatch priority

Key events are matched in order: command palette (when open) → printable characters
to a focused text-capturing component → global key bindings → overlay focus scopes
(modals) → sidebar keys → content key bindings → the focused component / Tab
traversal. The practical upshot: typing into a field always types, shortcuts work
everywhere else. Details in
[Controllers & Views]({{ '/docs/controllers-and-views/' | relative_url }}).

### Errors

An unhandled exception from a controller action does not crash the terminal. The
runtime logs the full backtrace to the application logger and renders a centered
error panel showing the exception class, message, and top backtrace lines. Any key
dismisses it and re-renders the current route; `q` quits. Handle expected errors
yourself with `rescue_from` before they reach the runtime.

### Quitting

`q`-style bindings produce a quit `Response`. The runtime also traps SIGINT and
treats an *unbound* `ctrl+c` keypress as quit, so apps always have an escape hatch —
bind `"ctrl+c"` yourself to take it over. On the way out the runtime drains
background tasks (with a grace period), persists the session when configured, and
restores the terminal.

For tests, instantiate `Charming::Runtime` directly with `MemoryBackend`. See [Testing]({{ '/docs/testing/' | relative_url }}).

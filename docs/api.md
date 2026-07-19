---
title: API Reference
layout: default
nav_order: 15
permalink: /docs/api/
---
# API Reference

This is a compact reference for Charming's current public API. Prefer these APIs in app code. Classes under `Charming::Internal` are runtime internals and are documented mainly for testing.

For tutorial-style explanations, see [Getting Started]({{ '/docs/getting-started/' | relative_url }}). For topic guides, see the [docs index]({{ '/' | relative_url }}).

## Application

Inherit from `Charming::Application`:

```ruby
class MyApp::Application < Charming::Application
  root File.expand_path("../..", __dir__)
end
```

Generated apps define routes separately in `config/routes.rb`:

```ruby
MyApp::Application.routes do
  root "home#show"
end
```

Routes can also be defined inline on the application class:

```ruby
class MyApp::Application < Charming::Application
  root File.expand_path("../..", __dir__)

  routes do
    root "home#show"
  end
end
```

Class APIs:

- `routes { ... }` defines routes with the router DSL.
- `logger value` sets the application logger. The default logger writes to `File::NULL`.
- `root path` sets the application root path used for resolving relative files and templates.
- `theme name, built_in: "phosphor"` registers a built-in JSON theme (`phosphor`, `catppuccin-mocha`, `catppuccin-latte`, `gruvbox-dark`, `nord`, `tokyonight`).
- `theme name, from: "config/themes/custom.json"` registers an app-local theme file.
- `theme name, extends: :parent, overrides: {token => spec}` derives a theme from a registered one.
- `default_theme name` sets the default theme.
- `theme_for name` resolves a theme object.
- `persist_session to: "tmp/session.json"` opts into JSON session persistence across restarts (framework-internal keys and non-JSON-safe values are excluded).
- `coalesce_input true` collapses held-key auto-repeat bursts into one dispatch (off by default).
- `mouse_motion :all` enables buttonless hover reporting (`:drag`, the default, reports motion only while a button is held).
- `namespace` returns the application namespace used for controller and template binding lookup.

Instance APIs:

- `routes` returns the app router.
- `logger` returns the application logger.
- `logger=` overrides the logger for this application instance.
- `session` returns persistent app session state.
- `save_session` writes the session to the configured path (the runtime calls this on exit).
- `theme` returns the active theme.
- `use_theme name` switches the active theme.

Environment:

- `Charming.env` returns the current environment as a string inquirer (from `CHARMING_ENV`, default `"development"`): `Charming.env.test?`, `Charming.env.production?`.

Entrypoint:

```ruby
Charming.run(MyApp::Application.new)
```

## Router

Routes are usually defined in `config/routes.rb`:

```ruby
MyApp::Application.routes do
  root "home#show"
  screen "/users/:id", to: "users#show", title: "User"
end
```

DSL methods:

- `root "controller#action", title: "Home"` maps `/`.
- `screen "/path", to: "controller#action", title: nil` maps a screen path.
- Omitting `#action` in `to:` defaults the action to `#show`.

Resolution rules:

- Exact routes win over dynamic routes.
- Dynamic segments use `:name` and match one segment.
- Params are symbol-keyed, for example `params[:id]`.
- Params are URL-decoded.
- Missing routes raise `KeyError`.

Route objects expose:

- `path`
- `controller_class`
- `action`
- `title`
- `params`

## Controller

Inherit from `Charming::Controller` or your app's `ApplicationController`.

Class APIs:

- `key name, action, scope: :content` binds a content-pane key to an action.
- `key name, action, scope: :global` binds an app-level shortcut.
- `command label, action = nil, &block` adds a command palette item.
- `timer name, every:, action:` dispatches a periodic timer while the route is active (`every:` must be positive).
- `on_task name, action:` handles async task completion.
- `on_task_progress name, action:` handles `progress.report` calls from a running task.
- `before_action method, only: nil, except: nil` runs a hook before matching actions.
- `after_action method, only: nil, except: nil` runs a hook after matching actions.
- `around_action method, only: nil, except: nil` wraps matching actions (the hook must `yield`).
- `rescue_from ExceptionClass, with: :handler` handles action exceptions (most-specific class wins).
- `layout layout_class` wraps rendered output in a class-based layout view.
- `layout "layouts/application"` wraps rendered output in an ERB template layout fallback.
- `layout false` disables inherited layout wrapping.
- `focus_ring *slots` defines tab-traversable focus slots (the first slot starts focused).

Instance APIs:

- `dispatch(action)` calls an action (through its hooks) and returns a response.
- `dispatch_key`, `dispatch_timer`, `dispatch_task`, `dispatch_task_progress`, `dispatch_mouse`, and `dispatch_paste` dispatch event-specific handlers.
- `render(body = "", **assigns)` produces a render response.
- `render "literal"` renders a literal string.
- `render :show, **assigns` renders a conventional Ruby view class, falling back to `app/views/<controller>/show.tui.erb` or `.txt.erb`.
- `render view_object` renders a class-based view or component object.
- `render_template(name, **assigns)` renders an explicit template path under `app/views`.
- `navigate_to(path)` produces a navigation response.
- `quit` produces a quit response.
- `session` accesses the application session.
- `logger` returns the application logger.
- `state(name, state_class, **attributes)` stores or returns a session-backed state object.
- `component_state(name, **defaults)` stores or returns a JSON-safe primitive hash for widget state (cursor, selection, offsets) — seeded from `defaults` on first access; survives `persist_session`.
- `run_task(name, timeout: nil) { ... }` submits async work; blocks accepting an argument receive a `Tasks::Progress` reporter.
- `cancel_task(name)` cancels an in-flight task (raises `Tasks::Cancelled` inside it).
- `params` exposes current route params.
- `event` exposes the current key, timer, task, progress, resize, mouse, or paste event.
- `screen` exposes terminal dimensions.
- `theme` returns the current theme.
- `use_theme(name)` switches themes.
- `open_command_palette`, `close_command_palette`, and `command_palette` manage the command palette.
- `open_theme_palette` opens the theme picker.
- `command_palette_open?` returns whether a command or theme palette is open.
- `focus` returns the controller's focus object (`focus.push_scope`, `focus.pop_scope`, `focus.cycle`, `focus.focus(slot)`, `focus.current`, `focus.ring`, `focus.overlay?`).
- `focused?(slot)` asks whether a slot is currently focused.
- `focus_sidebar`, `focus_content`, `sidebar_focused?`, and `content_focused?` support generated layouts (`focus_content` targets `:content` or the first non-sidebar slot in the ring).
- `sidebar_routes` returns the routes listed in the sidebar — override to filter.

Controller instances are ephemeral. Store durable state in `ApplicationState` objects through `state(...)`. Focus-slot component methods are invoked on key-dispatch paths where `before_action` has not run — build components from `session`/`params`, not hook-set instance variables.

## ApplicationState

Inherit from `Charming::ApplicationState`:

```ruby
class CounterState < Charming::ApplicationState
  attribute :count, :integer, default: 0
end
```

It includes ActiveModel model and attributes support, so typed attributes and validations are available.

Common attribute types include `:string`, `:integer`, `:float`, `:boolean`, `:date`, `:datetime`, and `:time`.

## Views And Templates

Class-based views are the default. Inherit from `Charming::View` and implement `render`:

```ruby
module MyApp
  module Home
    class ShowView < Charming::View
      def render
        text title, style: theme.title
      end
    end
  end
end
```

For `render :show` in `HomeController`, Charming resolves `MyApp::Home::ShowView` first.

## Template Fallback

`Charming::Templates` resolves and renders ERB templates under `app/views` when no conventional view class exists or when `render_template` is used.

Template APIs:

- `Charming::Templates.register(extension, handler)` registers a template handler.
- `Charming::Templates.resolve(name, root:)` resolves a template from an app root.
- `Charming::Templates::MissingTemplateError` is raised when no candidate file exists.

Registered extensions:

- `.tui.erb`
- `.txt.erb`

For `Templates.resolve("home/show", root: app_root)`, Charming searches:

```text
app/views/home/show.tui.erb
app/views/home/show.txt.erb
```

`.tui.erb` is preferred before `.txt.erb`.

Template handlers implement:

```ruby
def self.render(path, view)
  # return rendered string
end
```

## TemplateView

`Charming::TemplateView` renders resolved templates with normal view helpers and assigns:

```ruby
template = Charming::Templates.resolve("home/show", root: app_root)
view = Charming::TemplateView.new(template: template, home: home, theme: theme)
view.render
```

Constructor:

- `template:` is a resolved template.
- `namespace:` optionally controls constant lookup during template binding.
- `**assigns` become reader methods available inside the template.

Instance APIs:

- `render` renders the template to a string.
- `template_binding` returns the binding used by ERB handlers.

Generated controllers usually do not instantiate `TemplateView` directly. Use Ruby views with `render :show`, or `render_template "path"` for ERB fallback content.

## View

Inherit from `Charming::View` and implement `render`:

```ruby
class HomeView < Charming::View
  def render
    text title, style: theme.title
  end
end
```

Assigns passed to `new` become reader methods:

```ruby
HomeView.new(title: "Home", theme: theme)
```

View and template helpers:

- `text(value, style: nil)` renders text through an optional style.
- `box(value, style: nil)` renders boxed or styled content.
- `box(style: style) { ... }` captures nested helper output into a styled block.
- `row(*items, gap: 0)` joins rendered items horizontally.
- `column(*items, gap: 0)` joins rendered items vertically.
- `screen_layout(background: nil) { ... }` renders a full-screen declarative layout tree with `split`, `pane`, and `overlay`. `pane` blocks may accept a `Charming::Layout::Rect` argument for the pane's inner content area.
- `style` returns a new `Charming::UI::Style`.
- `theme` returns the assigned theme or default theme.
- `render_component(component)` renders a component.
- `render_partial(view)` renders another view.
- `yield_content` returns layout content.
- `layout_assigns` returns assigns used when composing layouts.
- `focused?(slot)` delegates focus lookup to the controller assign.

## Component

Inherit from `Charming::Component` for reusable UI objects. Components inherit view helpers and assign readers.

```ruby
class BadgeComponent < Charming::Component
  def render
    text label, style: theme.title
  end
end
```

Interactive components can implement:

- `handle_key(event)`
- `handle_mouse(event)`
- `handle_paste(event)` for bracketed-paste text
- `captures_text?` — return true when the component accepts free-typed prose; the controller then routes printable characters (and Tab) to it before key bindings

Return conventions:

- `:handled` means the event was consumed.
- `[:selected, value]` means a value was selected.
- `[:submitted, value]` means a value was submitted.
- `:cancelled` means the interaction was cancelled.
- `nil` means the event was not handled.

Bundled components:

- `Charming::Components::TextInput` — options include `masked:` (password rendering) and `history:` (up/down recall)
- `Charming::Components::TextArea` — plain Enter inserts a newline by default (`enter_newline: false` to opt out)
- `Charming::Components::Form`
- `Charming::Components::List` — `filter:` / `filter=` narrow items by fuzzy match (best match first)
- `Charming::Components::MultiSelectList` — `selected_indices:`, `max_selections:`; Space toggles, Enter submits `[:submitted, items]`
- `Charming::Components::Table` — `height:` adds a scrolling window with page up/down; `sort_by!(column, direction:)` / `toggle_sort(column)` sort with a ▲/▼ header marker
- `Charming::Components::Tree` — collapsible node hashes (`{label:, children:, expanded:}`)
- `Charming::Components::Viewport`
- `Charming::Components::Modal` — `max_body_height:` makes the body scrollable; `scroll_offset` exposes the position
- `Charming::Components::Toast` — `kind:` of `:info`, `:success`, `:warn`, `:error`
- `Charming::Components::StatusBar` — `left:`, `center:`, `right:`, or `hints:` pairs
- `Charming::Components::TabBar`
- `Charming::Components::Breadcrumbs`
- `Charming::Components::Badge`
- `Charming::Components::Autocomplete`
- `Charming::Components::HelpOverlay` — `HelpOverlay.for_controller(controller_class)` builds one from key bindings
- `Charming::Components::EmptyState`
- `Charming::Components::CommandPalette`
- `Charming::Components::CommandPaletteModal`
- `Charming::Components::Markdown`
- `Charming::Components::Spinner` — `style:` picks a named preset (`:line`, `:dots`, `:mini_dot`, `:jump`, `:pulse`, `:points`, `:globe`, `:moon`, `:meter`, `:hamburger`, `:ellipsis`); `frames:` overrides
- `Charming::Components::Progressbar` — `gradient: ["#rrggbb", "#rrggbb"]` colors the fill; `percent` reports completion
- `Charming::Components::Paginator` — `total:`, `per_page:`, `format:` (`:dots`/`:arabic`); `page_items(items)`, `next_page`/`prev_page`
- `Charming::Components::Filepicker` — `root:`, `show_hidden:`, `height:`; Enter descends or returns `[:selected, path]`, Backspace ascends, `toggle_hidden`
- `Charming::Components::Timer` — `duration:`, `label:`; `tick`, `expired?`, `reset`
- `Charming::Components::Stopwatch` — `label:`; `start`/`stop`/`reset`, `tick` accumulates while running
- `Charming::Components::ActivityIndicator`
- `Charming::Components::Audio` — `▶`/`■` playback indicator for a `Charming::Audio::Player` (`player:`, `label:`)
- `Charming::Components::ErrorScreen` — rendered by the runtime for unhandled exceptions
- `Charming::Components::KeyboardHandler` (mixin)
- `Charming::Components::FuzzyMatcher` — `score(query, candidate)` and `filter(query, candidates) { |c| label }`

ActivityIndicator constructor options include `width:`, `label:`, `index:`, `seed:`, `chars:`, `gradient:`, `label_style:`, `max_width:`, and `fallback_label:`.

Form component constructor:

- `fields:` array of form field objects.
- `state:` mutable primitive state hash, usually from `session[:forms]`.
- `theme:` optional `Charming::UI::Theme`.

Form field classes:

- `Charming::Components::Form::Input`
- `Charming::Components::Form::Textarea`
- `Charming::Components::Form::Select`
- `Charming::Components::Form::Confirm`
- `Charming::Components::Form::Note`

Controller helper:

```ruby
form(:signup) do |f|
  f.input :name, required: true
  f.textarea :bio, height: 5
  f.select :plan, options: ["Free", "Pro"]
  f.confirm :terms, required: true
end
```

TextArea component constructor:

- `value:` current multiline string.
- `placeholder:` rendered when value is empty.
- `width:` optional visible columns per line.
- `height:` optional visible rows.
- `cursor:` absolute cursor offset into `value`.
- `offset:` first visible row.
- `preferred_column:` remembered column for up/down movement (in display columns — wide characters count as two).
- `enter_newline:` whether plain Enter inserts a newline (default `true`).

Textarea keys:

- Plain `Enter` inserts a newline (press twice for a blank line).
- `Shift+Enter`, `Ctrl+J`, and `Ctrl+N` also insert newlines (and keep working when `enter_newline: false`).
- In a form: `Tab` leaves the field, `Ctrl+S` submits from any field.

Focused component result hooks:

- `[:submitted, values]` calls `<focus_slot>_submitted(values)` when defined.
- `[:selected, value]` calls `<focus_slot>_selected(value)` when defined.
- `:cancelled` calls `<focus_slot>_cancelled` when defined.

Markdown component constructor:

- `content:` Markdown source string.
- `width:` optional terminal width used for paragraph wrapping.
- `theme:` optional `Charming::UI::Theme`.
- `syntax_highlighting:` controls Rouge-backed code block highlighting, defaulting to `true`.
- `style:` Markdown style name or style config, defaulting to `:dark`. Built-in styles are `:dark`, `:light`, and `:notty`.
- `base_url:` optional base URL used to resolve relative links and image targets.
- `hyperlinks:` wraps links in OSC 8 escapes for clickable terminals (default `false`; the ` <url>` suffix is dropped when enabled).

Markdown parsing uses Commonmarker with CommonMark/GFM support. Syntax highlighting uses Rouge. Charming maps parsed nodes and Rouge tokens to terminal text through a Glamour-inspired Markdown style config and ANSI theme styling. Supported GFM rendering includes tables, task lists, strikethrough, autolinks, links, terminal-friendly image labels, definition lists, and footnotes.

## UI

For a guided tour of the `Style` builder, colors, and borders, see [Styling]({{ '/docs/styling/' | relative_url }}).

`Charming::UI` provides ANSI-aware layout helpers:

- `Charming::UI.style`
- `Charming::UI.adaptive(light:, dark:)` — a color resolved against the terminal background at render time
- `Charming::UI.join_horizontal(*blocks, gap: 0, align: :top)` — `align:` is `:top`/`:center`/`:bottom` or a 0.0–1.0 fraction
- `Charming::UI.join_vertical(*blocks, gap: 0, align: :left)` — pads narrower blocks to the widest; `align:` is `:left`/`:center`/`:right` or a fraction
- `Charming::UI.center(block, width:, height:)`
- `Charming::UI.place(block, width:, height:, top: 0, left: 0, background: nil)` — positions take integers, `:center`, or fractions
- `Charming::UI.overlay(base, overlay, top: :center, left: :center)`
- `Charming::UI.visible_slice(line, start_column, width)`
- `Charming::UI::Width.measure(value)` / `pad_to(line, width)` / `widest(lines)`
- `Charming::UI::Width.strip_ansi(value)`
- `Charming::UI::Truncate.tail(text, width, ellipsis: "…")`
- `Charming::UI::TextWrapper.new(width:).wrap(value)`
- `Charming::UI::Gradient.blend(from, to, amount)` / `steps(from, to, count)` / `colorize(text, from:, to:)`

Color capability (`Charming::UI::ColorSupport`):

- `level` — the active capability: `:truecolor`, `:color256`, `:color16`, or `:none` (auto-detected from `NO_COLOR`, `COLORTERM`, `TERM`).
- `level = value` — force a level; `nil` re-enables detection.
- `at_least?(:color256)` — capability comparison.
- Hex colors in styles and themes downconvert automatically to the active level.

Terminal background (`Charming::UI::Background`):

- `dark?` — whether the background is dark (OSC 11 query at runtime startup, `COLORFGBG` fallback, dark default).
- `assume = :dark | :light | nil` — override or re-enable detection.

Styles are immutable builders:

```ruby
style.foreground(:cyan).bold.border(:rounded).padding(1, 2).width(40)
```

Common style methods:

- `foreground` / `fg`
- `background` / `bg`
- `bold`, `faint`, `italic`, `underline`, `reverse`, `strikethrough`
- `padding`, `margin` (CSS shorthand; per-side setters like `padding_left`, `margin_top`)
- `border(style, sides:, foreground:, background:)` — style name or a custom `UI::Border`; `foreground:` takes a color or per-side hash
- `width`, `height`, `max_width`, `max_height`
- `align`, `align_vertical`
- `wrap`, `truncate(ellipsis: "…")` — overflow fit modes at a fixed width (default is clip)
- `render(value)`

## Themes

Theme tokens return `Charming::UI::Style` objects:

- `text`
- `title`
- `muted`
- `border`
- `selected`
- `info`
- `warn`

Themes can be loaded with `theme name, built_in:` or `theme name, from:` on the application class.

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

- `Charming::Tasks::ThreadedExecutor` — the default; one thread per task, name-based cancellation, graceful shutdown.
- `Charming::Tasks::InlineExecutor` — synchronous; for deterministic tests.
- `Charming::Tasks::Progress` — the reporter passed to task blocks: `progress.report(current, of: nil, message: nil)`.
- `Charming::Tasks::Cancelled` — raised inside a task by `cancel_task` or a `timeout:`; arrives as the `TaskEvent#error`.

Custom executors implement `submit(name, timeout: nil, &block)` (plain `submit(name, &block)` works until timeouts are used), plus optional `cancel(name)` and `shutdown(timeout:)`.

## Audio

`Charming::Audio::Player` plays a sound file by shelling out to a system audio binary. See [Audio]({{ '/docs/audio/' | relative_url }}) for the lifecycle and `run_task` pattern.

- `Player.new(system: Charming::Audio::System.new)` — the `system:` adapter is injectable so specs never shell out.
- `play(path)` — stops any current sound, spawns the resolved backend, returns the child PID; raises `Charming::Audio::Player::Unavailable` when no backend is installed.
- `stop` — terminates and reaps the current child (no-op when idle).
- `playing?` — true while the spawned sound is still running.
- `wait` — blocks until the current sound finishes, then clears it (drive from a background task).
- `available?` — true when a backend binary is installed for this platform.

Backends resolve in order: `ffplay` (any platform), then `afplay` (macOS), `paplay`/`mpg123`/`aplay` (Linux). `Charming::Audio::System` wraps `Process`/`ENV`/`RbConfig` and exposes `macos?`, `linux?`, `which?(command)`, `spawn(argv)`, `terminate(pid)`, `alive?(pid)`, and `wait(pid)`.

## Responses

Controllers return response objects through helper methods:

- `render(...)` creates a render response.
- `navigate_to(path)` creates a navigation response.
- `quit` creates a quit response.

Response factories:

- `Charming::Response.render(body)`
- `Charming::Response.navigate(path)`
- `Charming::Response.quit`

Response predicates:

- `response.navigate?`
- `response.quit?`

Response attributes:

- `kind`
- `body`
- `path`

The runtime follows navigation responses, renders render responses, and exits on quit responses.

## Runtime And Testing Backends

Apps normally use `TTYBackend` through `Charming.run`. Tests should use `Charming::Internal::Terminal::MemoryBackend` to avoid real terminal I/O. The runtime stops its loop when a test backend reports `exhausted?` (all pre-seeded events consumed), renders unhandled action exceptions as an `ErrorScreen` panel, and quits cleanly on SIGINT or an unbound `ctrl+c`.

## TestHelper

`require "charming/test_helper"` and `include Charming::TestHelper`:

- `build_controller(klass, app:, screen:, route:, event:)` — controller with test defaults.
- `key_event("ctrl+p")` — `KeyEvent` from a human-readable string.
- `press(klass, "q", app:)` — dispatch one key press, returns the `Response`.
- `press_sequence(klass, %w[down enter], app:)` — multiple presses sharing the app session.
- `memory_backend("up", "q", width: 80, height: 24)` — pre-seeded `MemoryBackend`.

RSpec matchers (registered when RSpec is loaded): `render_text("...")` and
`render_match(/.../)` compare against the ANSI-stripped body; `navigate_to("/path")`
asserts navigation; `be_quit` / `be_navigate` are response predicates.

## CLI

- `charming new NAME [--database sqlite3] [--force]` — scaffold an app.
- `charming generate TYPE NAME [args]` (`g`) — `screen`, `controller`, `view`, `component`, `model`, `migration`.
- `charming console` (`c`) — IRB with the app loaded and `app` available.
- `charming db:COMMAND` — see [Database]({{ '/docs/database/' | relative_url }}) for the full command table.

For testing patterns, see [Testing]({{ '/docs/testing/' | relative_url }}).

---
title: Controllers & Responses
layout: default
parent: API Reference
nav_order: 3
permalink: /docs/api/controllers/
---
# Controllers & Responses

Controllers are where your app's behavior lives: they bind keys, run actions, kick off async tasks, and decide what happens next. Every action returns a **response object** — render, navigate, or quit — that the runtime then carries out. Durable data goes in **`ApplicationState`** objects, because controller instances themselves are thrown away between events. For a guided introduction, see [Controllers & Views]({{ '/docs/controllers-and-views/' | relative_url }}).

## Charming::Controller

Inherit from `Charming::Controller` or your app's `ApplicationController`.

### Class APIs

Key bindings and commands — how users trigger actions:

- `key name, action, scope: :content` binds a content-pane key to an action.
- `key name, action, scope: :global` binds an app-level shortcut.
- `command label, action = nil, &block` adds a command palette item.

Timers and animation — periodic dispatch while the route is active:

- `timer name, every:, action:, autostart: true` dispatches a periodic timer while the route is active (`every:` must be positive; `autostart: false` waits for `start_timer`).
- `animate name, fps:, action:` declares a stopped animation timer ticking at *fps* frames per second (start it with `start_timer`).

Async task callbacks:

- `on_task name, action:` handles async task completion.
- `on_task_progress name, action:` handles `progress.report` calls from a running task.

Action hooks and error handling:

- `before_action method, only: nil, except: nil` runs a hook before matching actions.
- `after_action method, only: nil, except: nil` runs a hook after matching actions.
- `around_action method, only: nil, except: nil` wraps matching actions (the hook must `yield`).
- `rescue_from ExceptionClass, with: :handler` handles action exceptions (most-specific class wins).

Layout and focus:

- `layout layout_class` wraps rendered output in a class-based layout view.
- `layout "layouts/application"` wraps rendered output in an ERB template layout fallback.
- `layout false` disables inherited layout wrapping.
- `focus_ring *slots` defines tab-traversable focus slots (the first slot starts focused).

### Instance APIs

Dispatch — how the runtime invokes actions (app code rarely calls these directly):

- `dispatch(action)` calls an action (through its hooks) and returns a response.
- `dispatch_key`, `dispatch_timer`, `dispatch_task`, `dispatch_task_progress`, `dispatch_mouse`, and `dispatch_paste` dispatch event-specific handlers.

Rendering and navigation — what actions return:

- `render(body = "", **assigns)` produces a render response.
- `render "literal"` renders a literal string.
- `render :show, **assigns` renders a conventional Ruby view class, falling back to `app/views/<controller>/show.tui.erb` or `.txt.erb`.
- `render view_object` renders a class-based view or component object.
- `render_template(name, **assigns)` renders an explicit template path under `app/views`.
- `navigate_to(path)` produces a navigation response.
- `quit` produces a quit response.

State and session — where data survives between events:

- `session` accesses the application session.
- `state(name, state_class, **attributes)` stores or returns a session-backed state object.
- `component_state(name, **defaults)` stores or returns a JSON-safe primitive hash for widget state (cursor, selection, offsets) — seeded from `defaults` on first access; survives `persist_session`.
- `logger` returns the application logger.

Tasks and timers at runtime:

- `run_task(name, timeout: nil) { ... }` submits async work; blocks accepting an argument receive a `Tasks::Progress` reporter.
- `cancel_task(name)` cancels an in-flight task (raises `Tasks::Cancelled` inside it).
- `start_timer(name)`, `stop_timer(name)`, and `timer_running?(name)` control declared timers at runtime (unknown names raise `ArgumentError`; both mutators are idempotent).

Request context — what the current event looks like:

- `params` exposes current route params.
- `event` exposes the current key, timer, task, progress, resize, mouse, or paste event.
- `screen` exposes terminal dimensions.
- `theme` returns the current theme.
- `use_theme(name)` switches themes.

Palettes:

- `open_command_palette`, `close_command_palette`, and `command_palette` manage the command palette.
- `open_theme_palette` opens the theme picker.
- `command_palette_open?` returns whether a command or theme palette is open.

Focus:

- `focus` returns the controller's focus object (`focus.push_scope`, `focus.pop_scope`, `focus.cycle`, `focus.focus(slot)`, `focus.current`, `focus.ring`, `focus.overlay?`).
- `focused?(slot)` asks whether a slot is currently focused.
- `focus_sidebar`, `focus_content`, `sidebar_focused?`, and `content_focused?` support generated layouts (`focus_content` targets `:content` or the first non-sidebar slot in the ring).
- `sidebar_routes` returns the routes listed in the sidebar — override to filter.

{: .important }
Controller instances are ephemeral. Store durable state in `ApplicationState` objects through `state(...)`. Focus-slot component methods are invoked on key-dispatch paths where `before_action` has not run — build components from `session`/`params`, not hook-set instance variables.

## Charming::ApplicationState

Typed, session-backed state objects — the durable home for data that outlives a single controller instance. Inherit from `Charming::ApplicationState`:

```ruby
class CounterState < Charming::ApplicationState
  attribute :count, :integer, default: 0
end
```

It includes ActiveModel model and attributes support, so typed attributes and validations are available.

Common attribute types include `:string`, `:integer`, `:float`, `:boolean`, `:date`, `:datetime`, and `:time`.

## Responses

Every controller action returns a response object telling the runtime what to do next. Controllers return response objects through helper methods:

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

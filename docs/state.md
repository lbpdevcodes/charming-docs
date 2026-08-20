---
title: State
layout: default
nav_order: 7
permalink: /docs/state/
---
# State

Controllers are persistent per screen, so screen-lifetime state belongs in controller ivars. Application state classes hold app-lifetime state: values that must survive navigation away from the screen.

## ApplicationState

State classes inherit from `Charming::ApplicationState`, which includes `ActiveModel::Model` and `ActiveModel::Attributes`:

```ruby
module MyApp
  class HomeState < ApplicationState
    attribute :title, :string, default: "Home"
    attribute :count, :integer, default: 0
    attribute :status, :string, default: "Ready"
  end
end
```

Common attribute types include:

- `:string`
- `:integer`
- `:float`
- `:boolean`
- `:date`
- `:datetime`
- `:time`

## Session-Backed State

Use `Controller#state` to lazily create and cache state objects in the application session:

```ruby
def home
  state(:home, HomeState)
end
```

Subsequent calls with the same name return the same state object.

```ruby
def increment
  home.count += 1
  show
end
```

## Initial Attributes

Pass initial attributes through `state`:

```ruby
def counter
  state(:counter, CounterState, count: 10)
end
```

Initial attributes are only used when the state object is first created.

## Validations

Use normal ActiveModel validations:

```ruby
class CounterState < Charming::ApplicationState
  attribute :count, :integer, default: 0

  validate :count_gte_zero

  def count_gte_zero
    errors.add(:count, "must be >= 0") if count < 0
  end
end
```

Controller actions can call `valid?` and inspect `errors`:

```ruby
def save
  if form.valid?
    navigate :root
  else
    render :edit, form: form
  end
end
```

## Component State

Interactive widgets need state that outlives one event — a cursor position, a
scroll offset, an expanded-node set. Persistent controllers keep components for
the screen's lifetime, so the idiom is a memoized ivar:

```ruby
def query
  @query ||= Charming::Components::TextInput.new
end
```

The component object itself holds its state (value, cursor) across key presses.
Do **not** store live component objects in the session — `save_session` drops
anything that can't survive a JSON round-trip. (The exception is runtime engine
handles like `session[:audio] ||= Charming::Audio::Player.new`, which wrap a live
process and are intentionally dropped by `save_session`.)

Do not memoize data-bound components. A memoized `List` of database rows serves
stale items after the data changes. Rebuild those each dispatch and keep their
interaction state, like the selected index, in a `state` object.

`component_state` (a session-backed widget-state hash) still works but is
deprecated — use ivars for screen-lifetime component state.

## Form State

Use `Controller#form` for terminal forms. Charming stores the form's primitive
state (values, field cursors, errors, focus) on the controller instance, so the
draft survives events on the screen and is discarded on navigation.

```ruby
def signup_form
  form(:signup) do |f|
    f.input :name, required: true
    f.textarea :bio, height: 5
    f.select :plan, options: ["Free", "Pro"]
    f.confirm :terms, required: true
  end
end
```

On submit, the focused form returns `[:submitted, values]` and dispatches to
the action declared for the focus slot:

```ruby
focus_ring :signup_form
on_submit :signup_form, :save_signup

def save_signup(values)
  profile.assign_attributes(values)
  profile.valid? ? navigate(:root) : show
end
```

The declarations are `on_submit`, `on_select`, and `on_cancel`. Submit and
select actions receive the value; cancel actions receive no arguments.

When one form serves both "new" and "edit" modes, clear the stale draft when
the mode (or the record) changes so the builder's defaults re-seed:

```ruby
before_action :prepare_form_state

def prepare_form_state
  mode = editing_entry ? "edit-#{editing_entry.id}" : "new"
  return if @compose_mode == mode

  @compose_mode = mode
  reset_form(:entry)
end
```

## Session Persistence

Sessions are in-memory by default and vanish on quit. Opt into persistence per app:

```ruby
class MyApp::Application < Charming::Application
  persist_session to: "tmp/session.json"
end
```

The runtime saves on exit and the application reloads on boot. Only JSON-safe values
survive: nil, booleans, numbers, strings, symbols, and arrays/hashes of those. Hash
keys come back as symbols; symbol *values* come back as strings. Everything else —
state objects, procs — is silently skipped, and the framework always excludes its
internal keys (`focus_state`, `command_palette`). A corrupt or
missing file falls back to an empty session.

Good candidates: the chosen theme (stored automatically by `use_theme`) and
scroll positions that must survive restarts. Form drafts live on the controller
instance and are not restart-persisted.

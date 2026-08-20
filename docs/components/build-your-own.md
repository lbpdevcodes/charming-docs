---
title: Build Your Own
layout: default
parent: Components
nav_order: 40
permalink: /docs/components/build-your-own/
---
# Build your own component

A component is a plain object subclassing `Charming::Component`, which itself
subclasses `Charming::View`. That gives it two things: assigns passed to `new`
become reader methods, and a `render` method you implement with the view DSL —
`text`, `box`, `row`, `column`, `render_component`.

One rule shapes everything on this page: **components have no lifecycle** — no
mount/unmount. Keep a component with interaction state in a memoized controller
ivar, where it lives for the screen's lifetime. Any state that must survive
navigation lives in
[controller state or session]({{ '/docs/state/' | relative_url }}) and gets
passed back in through the constructor on the next render.

## Generate one

```
charming generate component <name>
```

writes `app/components/<name>_component.rb`:

```ruby
# frozen_string_literal: true

module MyApp
  class BadgeComponent < Charming::Component
    def render
      text "Badge"
    end
  end
end
```

## A static renderable

The minimal component implements `render` and reads its assigns:

```ruby
class CounterComponent < Charming::Component
  def render
    text "Count: #{count}", style: theme.info
  end
end
```

Render it from a template or [view]({{ '/docs/controllers-and-views/' | relative_url }})
with `render_component`, passing the durable state back in:

```erb
<%= render_component CounterComponent.new(count: home.count, theme: theme) %>
```

## Handling keys

Interactive components implement `handle_key(event)`. Use `Charming.key_of(event)`
to get the normalized key symbol:

```ruby
def handle_key(event)
  case Charming.key_of(event)
  when :up then move_up
  when :down then move_down
  when :enter then return Result.selected(selected_item)
  when :escape then return Result.cancelled
  else return nil
  end
  Result.handled
end
```

The return value is a `Charming::Components::Result` — a protocol the
controller understands:

| Return value | Meaning | Controller action dispatched |
|--------------|---------|------------------------------|
| `Result.handled` | The component consumed the event. | none — the screen re-renders |
| `Result.selected(object)` | The user selected an item. | declared with `on_select :slot, :action` — the action receives the object |
| `Result.submitted(value)` | The user submitted a value. | declared with `on_submit :slot, :action` — the action receives the value |
| `Result.cancelled` | The user cancelled the interaction. | declared with `on_cancel :slot, :action` — the action takes no arguments |
| `nil` | The component did not handle the event. | none — dispatch continues |

`Result.changed(value)` is reserved for a future `on_change` DSL; nothing
consumes it yet. The legacy forms (`:handled`, `[:selected, object]`,
`[:submitted, value]`, `:cancelled`) still dispatch — the pipeline normalizes
them to `Result` — but new components return `Result`.

Dispatch works through focus: when no higher-priority handler consumes a key,
the controller builds the component for the focused slot (by calling the
controller method named after the slot) and sends it `handle_key`. The declared
action only fires when the component returns the matching result; with no
declaration, development and test raise `UnhandledComponentEvent` and
production falls back to the default render. Returning `nil` lets the event
fall through to other handlers.

## The KeyboardHandler mixin

For components that map keys straight to methods, skip the `case` statement and
include `Charming::Components::KeyboardHandler`. Define a `KEY_ACTIONS` constant
mapping key symbols to method names — here is `Viewport`'s, verbatim:

```ruby
class Viewport < Component
  include KeyboardHandler

  KEY_ACTIONS = {
    up: :scroll_up,
    down: :scroll_down,
    page_up: :page_up,
    page_down: :page_down,
    home: :scroll_home,
    end: :scroll_end,
    left: :scroll_left,
    right: :scroll_right
  }.freeze
end
```

`handle_key` resolves the key with `Charming.key_of`, looks it up in
`KEY_ACTIONS`, sends the method, and returns `:handled` — or `nil` when no
mapping exists, so unhandled keys fall through.

The mixin also supports keymap aliases via a `@keymap` instance variable
(built-ins expose it as a `keymap:` assign). `keymap: :vim` applies
`VIM_KEYMAP` — `{up: :k, down: :j, left: :h, right: :l}` — so `j` triggers the
`:down` action alongside the arrow key. Pass a custom hash in the same shape,
mapping `KEY_ACTIONS` keys to one alias key or an array of them:

```ruby
Charming::Components::Viewport.new(content: doc, keymap: {page_down: :space})
```

## Capturing text

Components users type prose into override `captures_text?` to return true
(`TextInput`, `TextArea`, `Form`, `Autocomplete`, `CommandPalette`, and
`HelpOverlay` do; the `Charming::Component` default is false). While such a
component is focused:

- printable characters route to it **before** global and content key bindings —
  typing `q` into a field inserts a q instead of quitting
- `tab` reaches it before focus-ring traversal, so forms handle field navigation
- ctrl/alt-modified shortcuts (`ctrl+p`, `ctrl+s`) keep working

Two rules for your own components:

1. Override `captures_text?` if users type prose into it.
2. Focus-slot methods on controllers are invoked on key-dispatch paths where
   `before_action` has **not** run — build the component from
   [`session`/`params`]({{ '/docs/state/' | relative_url }}), never from
   hook-populated instance variables.

## Handling paste

The runtime enables bracketed paste, so pasted text arrives as a single
`Charming::Events::PasteEvent` (a `Data` with one member, `text`) instead of a
storm of key events. Implement `handle_paste(event)` and the focused component
receives it via the controller's `dispatch_paste`:

```ruby
def handle_paste(event)
  insert(event.text)
  :handled
end
```

The built-ins show the two sensible policies: `TextInput` strips newlines and
control characters (it is single-line), while `TextArea` keeps newlines and
normalizes CRLF to LF. `Form` forwards paste to its focused field (text fields
insert it, other field types ignore it) and `Autocomplete` inserts into its
query and re-filters.

## Handling the mouse

Mouse-capable components implement `handle_mouse(event)` and use the same
return protocol as `handle_key`:

```ruby
def handle_mouse(event)
  return nil unless event.click?

  select_at(event.y)
  :handled
end
```

Mouse events expose button, coordinates, and modifier flags (`ctrl`, `alt`,
`shift`, decoded from the terminal's button-code bits), plus helpers:

| Helper | True for |
|--------|----------|
| `click?` | A left/middle/right button **press** (not a release, drag, or hover). |
| `release?` | The button coming back up. |
| `scroll?` | Either wheel direction (`button_name` is `:scroll_up`/`:scroll_down`). |
| `drag?` | Movement with a button held down. |
| `motion?` | Any movement report — drag or hover. |

The controller hit-tests each mouse event against the named panes of the last
layout, so by the time your component sees the event its `x`/`y` are
**pane-local** — `event.y` is the row within your component, not the screen. A
click on a focusable pane also moves focus to that slot.

Drag reporting is on by default. Buttonless **hover** motion is opt-in on the
application class, since it makes the terminal report every mouse movement:

```ruby
class Application < Charming::Application
  mouse_motion :all   # :drag is the default
end
```

## Fuzzy filtering

`Charming::Components::FuzzyMatcher` is the fzf-style subsequence scorer behind
`List` filtering and the `CommandPalette`. Use it in your own components:

```ruby
FuzzyMatcher.score("opl", "Open palette")   # => positive score
FuzzyMatcher.score("xyz", "Open palette")   # => nil (not a subsequence)

FuzzyMatcher.filter(query, candidates) { |i| i.label }
```

`score(query, candidate)` returns a relevance score when every character of the
query appears in order within the candidate (case-insensitive), `nil`
otherwise. Contiguous runs and word-start matches score higher, so `"pal"`
prefers the literal run in "Open palette" over scattered letters. `filter`
returns matching candidates ordered best-score first (original order breaks
ties); the block extracts the searchable label, defaulting to `to_s`.

## Theming

Components read `theme` through a fallback chain: the `theme:` assign, then the
controller's theme, then `UI::Theme.default`. A theme is a set of tokens —
`text`, `title`, `muted`, `border`, `selected`, `info`, `warn` — each returning
a `UI::Style` ready to render:

```ruby
def render
  items.each_with_index.map { |item, index|
    style = index == selected_index ? theme.selected : theme.text
    style.render(item.to_s)
  }.join("\n")
end
```

Use `theme.selected` for the highlighted row, `theme.muted` for hints and
placeholders, `theme.border` for box borders. Stick to the tokens rather than
hard-coded colors and your component works under every theme — see
[Themes]({{ '/docs/themes/' | relative_url }}).

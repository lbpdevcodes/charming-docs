---
title: Components API
layout: default
parent: API Reference
nav_order: 5
permalink: /docs/api/components/
---
# Components API

Components are reusable UI objects: they render like views, and interactive ones also receive key, mouse, and paste events and answer with a small set of conventional return values. Charming bundles a large set of ready-made components — inputs, lists, tables, modals, spinners, and more. This page covers the base class, the event/return protocol, and the bundled catalog; for usage guides with screenshots per component, see [Components]({{ '/docs/components/' | relative_url }}).

## Charming::Component

Inherit from `Charming::Component` for reusable UI objects. Components inherit view helpers and assign readers.

```ruby
class BadgeComponent < Charming::Component
  def render
    text label, style: theme.title
  end
end
```

### Event handling

Interactive components can implement:

- `handle_key(event)`
- `handle_mouse(event)`
- `handle_paste(event)` for bracketed-paste text
- `captures_text?` — return true when the component accepts free-typed prose; the controller then routes printable characters (and Tab) to it before key bindings

```ruby
class ConfirmComponent < Charming::Component
  def handle_key(event)
    case Charming.key_of(event)
    when :y then [:submitted, true]
    when :n, :escape then :cancelled
    end
  end
end
```

### Return conventions

- `:handled` means the event was consumed.
- `[:selected, value]` means a value was selected.
- `[:submitted, value]` means a value was submitted.
- `:cancelled` means the interaction was cancelled.
- `nil` means the event was not handled.

### Focused component result hooks

When a focus-ring slot's component returns one of the conventional results, the controller calls a matching hook if you define one:

- `[:submitted, values]` calls `<focus_slot>_submitted(values)` when defined.
- `[:selected, value]` calls `<focus_slot>_selected(value)` when defined.
- `:cancelled` calls `<focus_slot>_cancelled` when defined.

## Bundled components

Text input and forms:

- `Charming::Components::TextInput` — options include `masked:` (password rendering) and `history:` (up/down recall)
- `Charming::Components::TextArea` — plain Enter inserts a newline by default (`enter_newline: false` to opt out)
- `Charming::Components::Form`
- `Charming::Components::Autocomplete`

Lists, tables, and navigation:

- `Charming::Components::List` — `filter:` / `filter=` narrow items by fuzzy match (best match first)
- `Charming::Components::MultiSelectList` — `selected_indices:`, `max_selections:`; Space toggles, Enter submits `[:submitted, items]`
- `Charming::Components::Table` — `height:` adds a scrolling window with page up/down; `sort_by!(column, direction:)` / `toggle_sort(column)` sort with a ▲/▼ header marker
- `Charming::Components::Tree` — collapsible node hashes (`{label:, children:, expanded:}`)
- `Charming::Components::Filepicker` — `root:`, `show_hidden:`, `height:`; Enter descends or returns `[:selected, path]`, Backspace ascends, `toggle_hidden`
- `Charming::Components::Paginator` — `total:`, `per_page:`, `format:` (`:dots`/`:arabic`); `page_items(items)`, `next_page`/`prev_page`
- `Charming::Components::TabBar`
- `Charming::Components::Breadcrumbs`

Overlays and dialogs:

- `Charming::Components::Modal` — `max_body_height:` makes the body scrollable; `scroll_offset` exposes the position
- `Charming::Components::HelpOverlay` — `HelpOverlay.for_controller(controller_class)` builds one from key bindings
- `Charming::Components::CommandPalette`
- `Charming::Components::CommandPaletteModal`
- `Charming::Components::ErrorScreen` — rendered by the runtime for unhandled exceptions

Status and progress:

- `Charming::Components::Toast` — `kind:` of `:info`, `:success`, `:warn`, `:error`
- `Charming::Components::StatusBar` — `left:`, `center:`, `right:`, or `hints:` pairs
- `Charming::Components::Badge`
- `Charming::Components::EmptyState`
- `Charming::Components::Spinner` — `style:` picks a named preset (`:line`, `:dots`, `:mini_dot`, `:jump`, `:pulse`, `:points`, `:globe`, `:moon`, `:meter`, `:hamburger`, `:ellipsis`); `frames:` overrides
- `Charming::Components::Progressbar` — `gradient: ["#rrggbb", "#rrggbb"]` colors the fill; `percent` reports completion
- `Charming::Components::ActivityIndicator`
- `Charming::Components::Timer` — `duration:`, `label:`; `tick`, `expired?`, `reset`
- `Charming::Components::Stopwatch` — `label:`; `start`/`stop`/`reset`, `tick` accumulates while running

Content and media:

- `Charming::Components::Viewport`
- `Charming::Components::Markdown`
- `Charming::Components::Audio` — `▶`/`■` playback indicator for a `Charming::Audio::Player` (`player:`, `label:`)

Mixins and utilities:

- `Charming::Components::KeyboardHandler` (mixin)
- `Charming::Components::FuzzyMatcher` — `score(query, candidate)` and `filter(query, candidates) { |c| label }`

ActivityIndicator constructor options include `width:`, `label:`, `index:`, `seed:`, `chars:`, `gradient:`, `label_style:`, `max_width:`, and `fallback_label:`.

## Forms

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

## TextArea

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

## Markdown

Markdown component constructor:

- `content:` Markdown source string.
- `width:` optional terminal width used for paragraph wrapping.
- `theme:` optional `Charming::UI::Theme`.
- `syntax_highlighting:` controls Rouge-backed code block highlighting, defaulting to `true`.
- `style:` Markdown style name or style config, defaulting to `:dark`. Built-in styles are `:dark`, `:light`, and `:notty`.
- `base_url:` optional base URL used to resolve relative links and image targets.
- `hyperlinks:` wraps links in OSC 8 escapes for clickable terminals (default `false`; the ` <url>` suffix is dropped when enabled).

Markdown parsing uses Commonmarker with CommonMark/GFM support. Syntax highlighting uses Rouge. Charming maps parsed nodes and Rouge tokens to terminal text through a Glamour-inspired Markdown style config and ANSI theme styling. Supported GFM rendering includes tables, task lists, strikethrough, autolinks, links, terminal-friendly image labels, definition lists, and footnotes.

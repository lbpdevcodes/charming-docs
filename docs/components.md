---
title: Components
layout: default
nav_order: 9
permalink: /docs/components/
---
# Components

Components are reusable UI objects. They inherit from `Charming::Component`, which itself inherits from `Charming::View`, so they get assigns and rendering helpers.

## Rendering Components

Render components from templates or views with `render_component`:

```erb
<%= render_component Charming::Components::List.new(
  items: ["Alpha", "Beta", "Gamma"],
  selected_index: 0,
  theme: theme
) %>
```

## Custom Components

Define a component by implementing `render`:

```ruby
class CounterComponent < Charming::Component
  def render
    text "Count: #{count}", style: theme.info
  end
end
```

Assigns passed to `new` become reader methods:

```erb
<%= render_component CounterComponent.new(count: home.count, theme: theme) %>
```

## Built-In Components

| Component | Description |
|-----------|-------------|
| `TextInput` | Editable text field. `masked: true` for passwords, `history: [...]` for REPL-style up/down recall, paste support. |
| `TextArea` | Multiline editor. Plain Enter inserts a newline (`enter_newline: false` to opt out); paste support. |
| `Form` | Huh-inspired form with input, textarea, select, multiselect, confirm, and note fields. |
| `List` | Selectable list with keyboard navigation, mouse support, and fuzzy filtering (`filter:` / `list.filter = query`). |
| `MultiSelectList` | List with `[x]` checkboxes — Space toggles, Enter submits the checked set, `max_selections:` caps it. |
| `Table` | Unicode data table. `height:` adds a scrolling window with page up/down and window-relative clicks; `sort_by!`/`toggle_sort` sort by column with a ▲/▼ header marker. |
| `Paginator` | Page tracker rendering `○ ● ○` dots or `2/3`; `page_items(items)` slices the current page. |
| `Filepicker` | Directory browser — Enter descends or selects a file (`[:selected, path]`), Backspace ascends, `toggle_hidden` shows dotfiles. |
| `Tree` | Collapsible hierarchy — right/left expand/collapse, Enter selects leaves, mouse toggles branches. |
| `Viewport` | Scrollable container for tall content, with wrapping and horizontal scroll. |
| `Modal` | Overlay dialog with title, content, help text — `max_body_height:` makes the body scrollable. |
| `Toast` | Auto-dismissing notification box with `:info`/`:success`/`:warn`/`:error` accents. |
| `StatusBar` | One-row bar with left/center/right segments; pass `hints:` for key-hint pairs. |
| `TabBar` | Horizontal tabs — left/right move, Enter selects, mouse clicks select. |
| `Breadcrumbs` | `Home › Projects › Current` trail with the last item highlighted. |
| `Badge` | Inline styled pill for statuses, counts, and versions. |
| `Autocomplete` | Text input with a live-filtered suggestion dropdown. |
| `HelpOverlay` | Keyboard cheat-sheet modal built from a controller's key bindings. |
| `EmptyState` | "Nothing here yet" placeholder with loading and error variants. |
| `CommandPaletteModal` | Standard modal wrapper for command palette content. |
| `CommandPalette` | Fuzzy-search command picker used by the framework (and reusable). |
| `Markdown` | CommonMark/GFM renderer backed by Commonmarker with Rouge syntax highlighting. |
| `Spinner` | Animated frame-cycling indicator with named presets — `style: :dots`, `:moon`, `:meter`, … |
| `ActivityIndicator` | Gradient activity bar with label and ellipsis animation. |
| `Progressbar` | Text-based progress bar. `gradient: ["#a", "#b"]` colors the fill with a sweep; `percent` reports completion. |
| `Timer` | Countdown clock (`mm:ss`) — `tick` per timer interval, `expired?`, `reset`. |
| `Stopwatch` | Count-up clock — `start`/`stop`/`reset`, accumulates only while running. |
| `Audio` | One-line playback-status indicator (`▶`/`■` + label) for a `Charming::Audio::Player`. See [Audio]({{ '/docs/audio/' | relative_url }}). |
| `ErrorScreen` | The panel the runtime renders for unhandled exceptions (not usually built by hand). |
| `KeyboardHandler` | Key-mapping mixin for custom components. |
| `FuzzyMatcher` | fzf-style subsequence scorer used by the palette — `FuzzyMatcher.filter(query, items) { |i| i.label }`. |

A few quick examples:

```ruby
# Toast — usually composited as a layout overlay
Charming::Components::Toast.new(message: "Saved!", kind: :success, theme: theme)

# StatusBar — bottom row with hints
Charming::Components::StatusBar.new(
  width: screen.width,
  left: " Entries",
  hints: [["enter", "open"], ["n", "new"], ["q", "quit"]],
  right: "8 entries ",
  theme: theme
)

# Tree — node hashes, mutated in place to track expansion
Charming::Components::Tree.new(nodes: [
  {label: "src", expanded: true, children: [{label: "main.rb"}]},
  {label: "README.md"}
])

# Password input with shell-style history
Charming::Components::TextInput.new(masked: true)
Charming::Components::TextInput.new(history: ["last command", "older command"])

# Spinner presets (drive `tick` from a controller timer)
Charming::Components::Spinner.new(style: :dots, label: "Fetching")

# Gradient progress
Charming::Components::Progressbar.new(total: 30, gradient: ["#f43f5e", "#22d3ee"])

# Fuzzy-filtered list (wire the query from a TextInput or palette)
Charming::Components::List.new(items: commands, filter: "dep")

# File selection
Charming::Components::Filepicker.new(root: Dir.pwd, height: 12, theme: theme)
```

`ActivityIndicator` accepts `max_width:` and `fallback_label:` to keep labeled loading indicators stable in constrained layouts:

```ruby
render_component Charming::Components::ActivityIndicator.new(
  width: 24,
  label: "Loading Top Stories",
  max_width: content_width,
  fallback_label: "Working"
)
```

## Markdown

Render Markdown with `Charming::Components::Markdown`:

```erb
<%= render_component Charming::Components::Markdown.new(
  content: readme,
  width: 72,
  theme: theme,
  style: :dark
) %>
```

Markdown parsing is handled by Commonmarker with CommonMark/GFM support. Code block tokenization is handled by Rouge. Charming owns terminal rendering, wrapping, and Glamour-inspired Markdown styling.

The built-in Markdown styles are `:dark`, `:light`, `:notty`, and `:auto` — which
picks dark or light from the detected terminal background. GFM tables, task lists, strikethrough, autolinks, links, images, definition lists, and footnotes render as terminal-friendly text:

- **Definition lists** (`Term` / `: definition`) render bold terms with indented, wrapped descriptions.
- **Footnotes** render `[name]` references inline and labeled definition blocks with hanging indentation.
- **Task lists** detect the checked state from the list marker only — prose mentioning `[x]` doesn't check a box.
- **Raw HTML blocks** render as nothing by default (matching Glamour); a custom style config can set `html_block: {format: "..."}` to show a placeholder.

Make links clickable in modern terminals with `hyperlinks: true` — links are wrapped
in OSC 8 escapes and the ` <url>` suffix is dropped since the target is embedded.
Off by default so test captures and older terminals stay clean:

```erb
<%= render_component Charming::Components::Markdown.new(
  content: readme,
  hyperlinks: true
) %>
```

Relative links and image targets can be resolved with `base_url:`:

```erb
<%= render_component Charming::Components::Markdown.new(
  content: readme,
  width: 72,
  style: :dark,
  base_url: "https://example.com/docs/"
) %>
```

Use it with `Viewport` for scrollable documentation or help screens:

```erb
<%= render_component Charming::Components::Viewport.new(
  content: Charming::Components::Markdown.new(content: readme, width: 72, theme: theme, style: :dark),
  width: 72,
  height: 20
) %>
```

Disable code syntax highlighting when plain code blocks are preferred:

```erb
<%= render_component Charming::Components::Markdown.new(
  content: readme,
  syntax_highlighting: false
) %>
```

## Interaction

Interactive components should expose `handle_key(event)` and may expose `handle_mouse(event)`.

Return conventions:

| Return value | Meaning |
|--------------|---------|
| `:handled` | The component consumed the event. |
| `[:selected, object]` | The user selected an item. |
| `:cancelled` | The user cancelled the interaction. |
| `nil` | The component did not handle the event. |

Controllers dispatch events to focused components when no higher-priority handler consumes the event.

Component results map to controller hooks named after the focus slot: `:cancelled` →
`<slot>_cancelled`, `[:selected, value]` → `<slot>_selected(value)`,
`[:submitted, value]` → `<slot>_submitted(value)`.

## Text Capture

Components that accept free-typed text override `captures_text?` to return true
(`TextInput`, `TextArea`, `Form`, `Autocomplete`, `CommandPalette`, and `HelpOverlay`
do). While such a component is focused:

- printable characters route to it **before** global and content key bindings —
  typing `q` into a field inserts a q instead of quitting
- `tab` reaches it before focus-ring traversal, so forms handle field navigation
- ctrl/alt-modified shortcuts (`ctrl+p`, `ctrl+s`) keep working

Two rules for your own components:

1. Override `captures_text?` if users type prose into it.
2. Focus-slot methods on controllers are invoked on key-dispatch paths where
   `before_action` has **not** run — build the component from `session`/`params`,
   never from hook-populated instance variables.

## Paste

The runtime enables bracketed paste, so pasted text arrives as a single
`Charming::Events::PasteEvent` instead of a storm of key events. `TextInput` and
`TextArea` implement `handle_paste(event)` (inputs strip newlines; textareas keep
them and normalize CRLF). Custom components can implement it too — the focused
component receives the event via the controller's `dispatch_paste`.

## Forms

Build session-backed forms from controllers with `form(:name)`. Form state is stored as primitive values under `session[:forms]`, so input survives fresh controller instances.

```ruby
class SignupController < Charming::Controller
  focus_ring :signup_form

  def show
    render :show, form: signup_form
  end

  def signup_form_submitted(values)
    # values => {name: "Ada", plan: "Pro", terms: true}
    navigate_to "/"
  end

  def signup_form_cancelled
    navigate_to "/"
  end

  private

  def signup_form
    form(:signup) do |f|
      f.input :name, label: "Name", placeholder: "Ada Lovelace", required: true
      f.textarea :bio, label: "Bio", height: 5, placeholder: "Tell us about yourself"
      f.select :plan, label: "Plan", options: ["Free", "Pro", "Team"]
      f.multiselect :interests, label: "Interests", options: ["TUI", "CLI", "Web"]
      f.confirm :terms, label: "Accept terms?", required: true
      f.note "Enter submits from the last field. Escape cancels."
    end
  end
end
```

```erb
<%= render_component form %>
```

Form fields:

| Field | Behavior |
|-------|----------|
| `input` | Single-line text input. |
| `textarea` | Multiline text input — Enter inserts a newline (twice for a blank line). |
| `select` | Single-choice picker (`options:`, `option_label:` for display strings). |
| `multiselect` | Multiple-choice picker — Space toggles the highlighted option; the checked set (in option order) is the value. `max_selections:` caps it. |
| `confirm` | Boolean yes/no field. |
| `note` | Static, non-focusable text. |

Keyboard behavior:

| Key | Behavior |
|-----|----------|
| `Tab` / `Shift+Tab` | Move between focusable fields (including out of a textarea). |
| `Enter` | In a textarea: insert a newline. Elsewhere: next field, or submit from the last field. |
| `Ctrl+S` | Submit the form from any field. |
| `Escape` | Cancel the form. |
| `Up` / `Down` | Change select choices / move the multiselect highlight. |
| `Space` | Toggle the highlighted multiselect option; toggle confirm fields. |
| `y`, `n` | Set confirm fields. |

This matches charm.sh's huh: a textarea is a text editor, so it owns Enter — leave it
with Tab and submit with Ctrl+S. (Shift+Enter/Ctrl+J/Ctrl+N also insert newlines, for
muscle memory and for hosts that construct `TextArea` with `enter_newline: false`.)

Focused form results dispatch to controller hooks named after the focus slot: `signup_form_submitted(values)` and `signup_form_cancelled`.

## Key Events

Use `Charming.key_of(event)` when component code needs the normalized key symbol:

```ruby
def handle_key(event)
  case Charming.key_of(event)
  when :enter then [:selected, selected_item]
  when :escape then :cancelled
  else nil
  end
end
```

## Mouse Events

Mouse-capable components can implement `handle_mouse(event)`:

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

Drag reporting is on by default. Buttonless **hover** motion is opt-in on the
application class, since it makes the terminal report every mouse movement:

```ruby
class Application < Charming::Application
  mouse_motion :all   # :drag is the default
end
```

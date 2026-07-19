---
title: Autocomplete
layout: default
parent: Components
nav_order: 12
permalink: /docs/components/autocomplete/
---
# Autocomplete

`Charming::Components::Autocomplete` is a combobox: a TextInput with a suggestion list beneath it, filtered live against the typed value (case-insensitive substring match). It is interactive and captures text: while focused, printable characters route to it before any key binding — typing `q` inserts a q instead of quitting — `tab` reaches it before focus-ring traversal, and ctrl/alt shortcuts keep working.

```text
ra|
  rails
  rake
  rack
```

The first line is the inner text input (with `|` cursor marker); the highlighted suggestion renders in the theme's selected style, the rest muted.

## Quick start

Components are rebuilt on every event, so persist the typed value, cursor, and highlight in a `component_state` hash. Expose the component from a memoized method named after a `focus_ring` slot.

```ruby
class GemPickerController < Charming::Controller
  focus_ring :gem_picker

  def show
    gem_picker_state[:value] = gem_picker.value
    gem_picker_state[:cursor] = gem_picker.cursor
    gem_picker_state[:selected_index] = gem_picker.selected_index
    render :show, picker: gem_picker
  end

  def gem_picker_submitted(value)     # Enter: highlighted suggestion or free text
    session[:chosen_gem] = value
    navigate_to "/"
  end

  def gem_picker_cancelled            # Escape
    navigate_to "/"
  end

  private

  def gem_picker
    @gem_picker ||= Charming::Components::Autocomplete.new(
      suggestions: %w[rails rake rack rspec rubocop],
      value: gem_picker_state[:value],
      cursor: gem_picker_state[:cursor],
      selected_index: gem_picker_state[:selected_index],
      placeholder: "Pick a gem…",
      theme: theme
    )
  end

  def gem_picker_state
    component_state(:gem_picker, value: "", cursor: 0, selected_index: 0)
  end
end
```

```erb
<%= render_component picker %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `suggestions:` | required | Full candidate list (each entry is `to_s`'d). |
| `value:` | `""` | Initial typed text, seeding the inner TextInput. |
| `cursor:` | end of value | Cursor offset in the inner TextInput. |
| `placeholder:` | `""` | Shown while the value is empty. |
| `selected_index:` | `0` | Highlighted row in the filtered dropdown (clamped to the match count). |
| `max_suggestions:` | `6` | Caps the visible dropdown rows. |
| `theme:` | app theme | Styles the highlighted/muted suggestion rows. |

An empty value shows all suggestions (up to the cap); otherwise matches are the suggestions containing the value, case-insensitively.

## Keyboard

| Key | Action |
|-----|--------|
| printable characters | Edit the text and re-filter the dropdown. |
| `↑` / `↓` | Move the highlight (clamped, no wrap). |
| `Enter` | Submit — the highlighted suggestion, or the raw text when nothing matches. |
| `Escape` | Cancel. |
| `←` / `→`, `Home` / `End`, `Backspace`, `Delete` | Edit via the inner TextInput. |

## What it returns

`handle_key` returns `[:submitted, value]` on Enter (value is the highlighted suggestion, falling back to the typed text when no suggestion matches), `:cancelled` on Escape, `:handled` for consumed keys, and `nil` otherwise. On a focus slot these invoke `<slot>_submitted(value)` and `<slot>_cancelled` on the controller. See the [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `value` / `cursor` — the typed text and cursor offset (delegated to the inner TextInput).
- `selected_index` — the highlighted row in the filtered list.
- `filtered_suggestions` — the current matches, capped at `max_suggestions`; useful for rendering match counts or wiring custom behavior.

## Tips

- Persist `value`, `cursor`, and `selected_index` — the component is rebuilt from primitives on every event.
- The highlight is clamped after every edit, so it never points past the (possibly shrunken) match list — but it does not reset to 0 when the query changes; persist what the component reports.
- Pasting works: `handle_paste` inserts the pasted text into the query at the cursor and re-filters the suggestions, just like typing.

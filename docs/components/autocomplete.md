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

Controllers live for the screen's lifetime, so keep the component in a memoized ivar exposed from a method named after a `focus_ring` slot.

```ruby
class GemPickerController < Charming::Controller
  focus_ring :gem_picker

  on_submit :gem_picker, :pick_gem      # Enter: highlighted suggestion or free text
  on_cancel :gem_picker, :close_picker  # Escape

  def show
    render :show, picker: gem_picker
  end

  def pick_gem(value)
    session[:chosen_gem] = value
    navigate :root
  end

  def close_picker
    navigate :root
  end

  private

  def gem_picker
    @gem_picker ||= Charming::Components::Autocomplete.new(
      suggestions: %w[rails rake rack rspec rubocop],
      placeholder: "Pick a gem…",
      theme: theme
    )
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

`handle_key` returns `Result.submitted(value)` on Enter (value is the highlighted suggestion, falling back to the typed text when no suggestion matches), `Result.cancelled` on Escape, `Result.handled` for consumed keys, and `nil` otherwise. On a focus slot these dispatch the actions declared with `on_submit` and `on_cancel`. See the [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `value` / `cursor` — the typed text and cursor offset (delegated to the inner TextInput).
- `selected_index` — the highlighted row in the filtered list.
- `filtered_suggestions` — the current matches, capped at `max_suggestions`; useful for rendering match counts or wiring custom behavior.

## Tips

- The memoized ivar keeps `value`, `cursor`, and `selected_index` between events. Seed them from a `state` object when the picker must restore across navigation.
- The highlight is clamped after every edit, so it never points past the (possibly shrunken) match list — but it does not reset to 0 when the query changes.
- Pasting works: `handle_paste` inserts the pasted text into the query at the cursor and re-filters the suggestions, just like typing.

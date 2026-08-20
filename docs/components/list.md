---
title: List
layout: default
parent: Components
nav_order: 1
permalink: /docs/components/list/
---
# List

`Charming::Components::List` is a vertically-scrollable, selectable list. Reach for it whenever the user picks one thing from a set — menu entries, records, search results. It is interactive: it lives in a focus ring, handles keyboard navigation, and (when given a `height:`) responds to mouse clicks.

```text
  Alpha
> Beta
  Gamma
```

The selected row gets a `> ` prefix and the theme's selected style.

## Quick start

Build the list once in a memoized focus-slot method, put the slot in the focus ring, and refresh its items on render (adapted from the journal example's `entries_controller.rb`):

```ruby
class EntriesController < ApplicationController
  focus_ring :entries

  on_select :entries, :open_entry

  def show
    entries.items = Entry.recent_first.to_a
    entries_state.selected_index = entries.selected_index
    render :show, entries_list: entries
  end

  # Enter on the list opens the selected entry.
  def open_entry(entry)
    navigate :entry, id: entry.id
  end

  # The selectable entry list, memoized so j/k selection survives across events.
  # `show` refreshes the items each render — the component holds the interaction
  # state, the items always reflect the database.
  def entries
    @entries ||= Charming::Components::List.new(
      items: Entry.recent_first.to_a,
      selected_index: entries_state.selected_index,
      height: [screen.height - 10, 3].max,
      label: :list_label.to_proc,
      theme: theme
    )
  end

  private

  def entries_state
    state(:entries, EntriesState)
  end
end
```

```erb
<%= render_component entries_list %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `items:` | (required) | The array of selectable objects. |
| `selected_index:` | `0` | The initially selected index, clamped into range. |
| `height:` | `nil` | Fixed-height window over the items; auto-scrolls to keep the selection in view. Also required for mouse support. |
| `label:` | `nil` (`to_s`) | Callable that extracts an item's display string. |
| `theme:` | `nil` | Theme used for the selected-row style. |
| `keymap:` | `:vim` | Keybinding style — `:vim` adds `j`/`k` aliases; pass `nil` to disable. |
| `filter:` | `nil` | Fuzzy-filter query; narrows items via `FuzzyMatcher` (best match first). |

## Keyboard

| Key | Action |
|-----|--------|
| `up` / `k` | Move selection up. |
| `down` / `j` | Move selection down. |
| `home` | Jump to the first item. |
| `end` | Jump to the last item. |
| `enter` | Select the highlighted item. |

The `j`/`k` aliases come from the default `keymap: :vim`.

## Mouse

A click within the visible window selects the clicked row. Mouse handling requires `height:` to be set; clicks outside the window return `nil` (unhandled).

## What it returns

| Return value | Meaning |
|--------------|---------|
| `[:selected, item]` | Enter pressed with an item highlighted — dispatches the action declared with `on_select :slot, :action` (the action receives the item). |
| `:handled` | A navigation key or click was consumed. |
| `nil` | The event was not handled. |

See the shared [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `selected_item` — the highlighted item, or `nil` when the list is empty.
- `selected_index` — the highlighted index (within the filtered view).
- `items` — the visible items: the source list narrowed by the active filter, or the full list when no filter is set.
- `filter` / `filter = query` — read or replace the fuzzy query; assigning reclamps the selection to the narrowed view (`nil` clears it).
- `items = new_items` — replace the source items after the data changed; the selection reclamps. This is the data-refresh half of the memoized-list pattern.

## Tips

- **Memoize the list, refresh its items.** Keep the `List` in an ivar so j/k movement and scroll position survive across events, and assign `list.items = rows` on each render so the data stays fresh. Rebuilding the component per dispatch instead would discard the keypress that moved the selection before the state write-back could see it.
- `selected_index` is a position within the *filtered* view, not the source array. When you drive `filter=` from a text input, persist the index after filtering, and expect it to reclamp as the match set shrinks.
- Without `height:` the whole list renders and mouse clicks are ignored — set a height for long lists.

---
title: MultiSelectList
layout: default
parent: Components
nav_order: 2
permalink: /docs/components/multi-select-list/
---
# MultiSelectList

`Charming::Components::MultiSelectList` is a [List]({{ '/docs/components/list/' | relative_url }}) variant for picking several items at once: Space toggles a checkbox on the highlighted row, Enter submits the checked set. It is interactive — put it in a focus ring like any List.

```text
[x] TUI
[ ] CLI
[x] Web
```

The highlighted row renders in the theme's selected style; checked rows get an `[x]` prefix.

## Quick start

```ruby
class InterestsController < ApplicationController
  focus_ring :interests

  on_submit :interests, :submit_interests

  def show
    interests_state.selected_index = interests.selected_index
    interests_state.checked = interests.selected_indices
    render :show, interests_list: interests
  end

  # Enter submits the checked items, in list order.
  def submit_interests(items)
    save_interests(items)
    navigate :root
  end

  def interests
    @interests ||= Charming::Components::MultiSelectList.new(
      items: %w[TUI CLI Web],
      selected_index: interests_state.selected_index,
      selected_indices: interests_state.checked,
      max_selections: 2,
      theme: theme
    )
  end

  private

  def interests_state
    state(:interests, InterestsState)
  end
end
```

```erb
<%= render_component interests_list %>
```

## Options

All [List options]({{ '/docs/components/list/' | relative_url }}) (`items:`, `selected_index:`, `height:`, `label:`, `theme:`, `keymap:`, `filter:`), plus:

| Option | Default | What it does |
|--------|---------|--------------|
| `selected_indices:` | `[]` | The initially checked item indices (out-of-range values are dropped). |
| `max_selections:` | `nil` | Cap on how many items can be checked at once; `nil` means unlimited. |

## Keyboard

| Key | Action |
|-----|--------|
| `up` / `k` | Move the highlight up. |
| `down` / `j` | Move the highlight down. |
| `home` / `end` | Jump to first / last item. |
| `space` | Toggle the highlighted item's checkbox. |
| `enter` | Submit the checked items. |

`j`/`k` come from the default `keymap: :vim`.

## Mouse

Inherited from List: a click within the visible window moves the highlight to the clicked row (requires `height:`). Clicking does not toggle the checkbox — that stays on Space.

## What it returns

| Return value | Meaning |
|--------------|---------|
| `[:submitted, items]` | Enter pressed — dispatches the action declared with `on_submit` (the action receives the checked items in list order). |
| `:handled` | A toggle or navigation key was consumed (including a Space toggle refused by `max_selections:`). |
| `nil` | The event was not handled. |

Note the contrast with List: Enter here means *submit the set*, not *select one item*. See the shared [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `selected_indices` — the set of checked indices.
- `selected_items` — the checked items, sorted into list order.
- `selected_index` / `selected_item` — the highlight position (from List), independent of what's checked.

## Tips

- **Persist *both* pieces of state** — the highlight (`selected_index`) and the checked set (`selected_indices`) — into controller state and pass them back in on construction, as in the quick start above, or they reset when the screen is re-entered.
- `selected_indices` are positions in the visible items list. If you combine checkboxes with `filter:`, the indices point into the filtered view — keep the filter stable while collecting selections, or map items yourself.
- When `max_selections:` is reached, Space on an unchecked item is silently consumed (`:handled`) — nothing is toggled.

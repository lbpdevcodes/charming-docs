---
title: TabBar
layout: default
parent: Components
nav_order: 6
permalink: /docs/components/tab-bar/
---
# TabBar

`Charming::Components::TabBar` renders a horizontal row of tabs with one active tab. Left/right move the active tab, Enter confirms it, and mouse clicks select the clicked tab. It is interactive — put it in a focus ring when tabs should be keyboard-navigable.

```text
 Files   Search   Git
```

Each label renders padded with a space on both sides; the active tab uses the theme's selected style, the rest are muted.

## Quick start

```ruby
class WorkspaceController < ApplicationController
  focus_ring :tabs, :content

  on_select :tabs, :switch_tab

  def show
    workspace_state.tab_index = tabs.selected_index
    render :show, tab_bar: tabs
  end

  # Enter confirms the highlighted tab.
  def switch_tab(index)
    workspace_state.tab_index = index
    show
  end

  def tabs
    @tabs ||= Charming::Components::TabBar.new(
      tabs: ["Files", "Search", "Git"],
      selected_index: workspace_state.tab_index,
      theme: theme
    )
  end

  private

  def workspace_state
    state(:workspace, WorkspaceState)
  end
end
```

```erb
<%= render_component tab_bar %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `tabs:` | (required) | Array of tab labels (stringified). |
| `selected_index:` | `0` | The active tab, clamped into range. |
| `separator:` | `"  "` | String rendered between tabs. |
| `keymap:` | `:vim` | Keybinding style — `:vim` adds `h`/`l` aliases; pass `nil` to disable. |
| `theme:` | `nil` | Theme for the active (selected) and inactive (muted) styles. |

## Keyboard

| Key | Action |
|-----|--------|
| `left` / `h` | Move the active tab one position left. |
| `right` / `l` | Move the active tab one position right. |
| `home` / `end` | Jump to first / last tab. |
| `enter` | Select the active tab. |

`h`/`l` come from the default `keymap: :vim`. All keys return `nil` when there are no tabs.

## Mouse

A click hit-tests the rendered column spans (label widths plus separators, measured display-width-aware) and selects the tab under the pointer. Clicks landing on a separator return `nil`.

## What it returns

| Return value | Meaning |
|--------------|---------|
| `[:selected, index]` | Enter pressed — dispatches the action declared with `on_select` (the action receives the active tab's index). |
| `:handled` | A navigation key or tab click was consumed. |
| `nil` | The event was not handled. |

See the shared [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `tabs` — the stringified labels.
- `selected_index` — the active tab's index.

## Tips

- Persist `selected_index` in controller state and pass it back on construction, as in the quick start, or the bar snaps back to the first tab when the screen is re-entered.
- Left/right move the highlight but dispatch no action; only Enter (or a click plus your render) reports back. If you want tab switches to take effect immediately as the user arrows around, write `tabs.selected_index` into state during `show` and render the pane for that index — no Enter needed.

---
title: Table
layout: default
parent: Components
nav_order: 3
permalink: /docs/components/table/
---
# Table

`Charming::Components::Table` renders tabular data with unicode borders, a highlighted selected row, keyboard navigation, sorting, and mouse click selection. It is interactive — give it a focus slot when the user should navigate rows. Reach for it when data has columns; for a flat pick-one list use [List]({{ '/docs/components/list/' | relative_url }}).

```text
┌─────────┬───────┐
│Name     │Stars ▼│
│bubbletea│5400   │
│charm    │1200   │
│glow     │800    │
└─────────┴───────┘
```

The separator line under the header is trimmed for a compact layout, and the sorted column is marked `▲`/`▼`.

## Quick start

```ruby
class ReposController < ApplicationController
  focus_ring :repos

  on_select :repos, :open_repo

  def show
    repos.rows = Repo.all.map { |repo| [repo.name, repo.stars] }
    repos_state.selected_index = repos.selected_index
    render :show, repos_table: repos
  end

  # Enter opens the selected row.
  def open_repo(row)
    navigate :repo, id: row.first
  end

  # The table, memoized so j/k selection survives across events. `show`
  # refreshes the rows each render — the component holds the interaction
  # state, the rows always reflect the database.
  def repos
    @repos ||= Charming::Components::Table.new(
      header: ["Name", "Stars"],
      rows: Repo.all.map { |repo| [repo.name, repo.stars] },
      selected_index: repos_state.selected_index,
      height: [screen.height - 10, 3].max,
      theme: theme
    )
  end

  private

  def repos_state
    state(:repos, ReposState)
  end
end
```

```erb
<%= render_component repos_table %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `header:` | (required) | Array of column labels (scalars are wrapped and stringified). |
| `rows:` | `[]` | Body rows — each a String, an Array, or a Hash of column-value pairs. Excess cells merge into the last column. |
| `selected_index:` | `0` | The initially selected row, clamped into range. |
| `keymap:` | `:vim` | Keybinding style — `:vim` adds `j`/`k` aliases; pass `nil` to disable. |
| `theme:` | `nil` | Theme used for the selected-row style. |
| `height:` | `nil` | Limits the visible body rows; the window auto-scrolls to keep the selection in view, and page keys move by a full window. |

## Keyboard

| Key | Action |
|-----|--------|
| `up` / `k` | Move selection up one row. |
| `down` / `j` | Move selection down one row. |
| `home` / `end` | Jump to first / last row. |
| `page_up` / `page_down` | Move by one window of rows. |
| `enter` | Select the highlighted row. |

`j`/`k` come from the default `keymap: :vim`. All keys return `nil` when the table has no rows.

## Mouse

A click within the body area selects the clicked row. The handler subtracts `HEADER_HEIGHT` (2 rows: top border + header line) from the click's `y`, then maps it relative to the visible window when `height:` is set.

## What it returns

| Return value | Meaning |
|--------------|---------|
| `[:selected, row]` | Enter pressed — dispatches the action declared with `on_select` (the action receives the row as constructed: String, Array, or Hash). |
| `:handled` | A navigation key or click was consumed. |
| `nil` | The event was not handled. |

See the shared [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `selected_row` — the highlighted row, or `nil` for an empty table.
- `rows = new_rows` — replace the body rows after the data changed; the selection reclamps. This is the data-refresh half of the memoized-table pattern.
- `sort_by!(column, direction: :asc)` — sorts rows by a header label or 0-based index; numeric-looking cells compare numerically, everything else as strings. Returns `self`.
- `toggle_sort(column)` — sorts by `column`, flipping direction on repeated calls (ascending first). Returns `self`.
- `header` / `rows` / `selected_index` — readers for the constructor state.

## Tips

- **Memoize the table, refresh its rows.** Keep the `Table` in an ivar so j/k movement survives across events, and assign `table.rows = new_rows` on each render so the data stays fresh.
- Sorting mutates the *instance's* rows — a fresh `rows =` assignment discards the sort. Store the sort column and direction in state and reapply `sort_by!` after assigning: `table.sort_by!(state.sort_column, direction: state.sort_direction)`.
- `toggle_sort` only flips when called twice *on the same instance* for the same column; after a `rows =` refresh, track the direction yourself and use `sort_by!`.
- With header and rows both empty, `render` returns the placeholder string `(empty table)`.

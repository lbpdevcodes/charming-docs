---
title: Tree
layout: default
parent: Components
nav_order: 4
permalink: /docs/components/tree/
---
# Tree

`Charming::Components::Tree` renders a collapsible hierarchy — file explorers, nested categories, outline views. Nodes are plain hashes: `{label: "src", children: [...], expanded: true}`, where `children` and `expanded` are optional. It is interactive: the cursor moves through *visible* nodes, branches expand and collapse, and Enter selects leaves.

```text
▾ src
    main.rb
    cli.rb
▸ spec
  README.md
```

Expanded branches are marked `▾`, collapsed branches `▸`; each level indents by two spaces, and the cursor row uses the theme's selected style.

## Quick start

```ruby
class FilesController < ApplicationController
  focus_ring :files

  on_select :files, :open_node

  def show
    files_state.cursor_index = files.cursor_index
    render :show, files_tree: files
  end

  # Enter on a leaf returns its node hash.
  def open_node(node)
    open_file(node[:label])
  end

  def files
    @files ||= Charming::Components::Tree.new(
      nodes: files_state.nodes,
      cursor_index: files_state.cursor_index,
      theme: theme
    )
  end

  private

  # The node hashes live in durable state — the Tree mutates them in place.
  def files_state
    state(:files, FilesState, nodes: [
      {label: "src", expanded: true, children: [{label: "main.rb"}, {label: "cli.rb"}]},
      {label: "spec", children: [{label: "main_spec.rb"}]},
      {label: "README.md"}
    ])
  end
end
```

```erb
<%= render_component files_tree %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `nodes:` | (required) | Array of root node hashes (`label:`, optional `children:` and `expanded:`). Mutated in place to track expansion. |
| `cursor_index:` | `0` | Cursor position in the visible-node list, clamped into range. |
| `height:` | `nil` | Fixed-height window over the visible nodes; auto-scrolls to keep the cursor in view. |
| `keymap:` | `:vim` | Keybinding style — `:vim` adds `h`/`j`/`k`/`l` aliases; pass `nil` to disable. |
| `theme:` | `nil` | Theme used for the cursor-row style. |

## Keyboard

| Key | Action |
|-----|--------|
| `up` / `k` | Move the cursor up one visible node. |
| `down` / `j` | Move the cursor down one visible node. |
| `right` / `l` | Expand the branch under the cursor. |
| `left` / `h` | Collapse an expanded branch, or jump to the parent of a leaf or collapsed node. |
| `home` / `end` | Jump to first / last visible node. |
| `enter` | Select a leaf (`[:selected, node]`) or toggle a branch. |

`h`/`j`/`k`/`l` come from the default `keymap: :vim`.

## Mouse

A click moves the cursor to the clicked row; clicking a branch also toggles its expansion.

## What it returns

| Return value | Meaning |
|--------------|---------|
| `[:selected, node]` | Enter on a leaf — dispatches the action declared with `on_select` (the action receives the node hash). |
| `:handled` | Navigation, expand/collapse, branch toggle, or click was consumed. |
| `nil` | The event was not handled (or the tree is empty). |

See the shared [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `current_node` — the node hash under the cursor, or `nil` for an empty tree.
- `nodes` — the root node array you passed in.
- `cursor_index` — the cursor position in the flattened visible-node list.

## Tips

- **The node hashes are mutated in place** — expanding a branch sets `node[:expanded] = true` on *your* hash. Keep the nodes array in durable controller state (as in the quick start) so expansion survives navigation away and back. If you rebuild the array from scratch each visit, every branch snaps shut.
- Persist `cursor_index` in state too, and pass it back on construction — the memoized component is discarded when the screen is left.
- `cursor_index` indexes the *visible* rows, so collapsing a branch above the cursor shifts what the index points at. The constructor clamps it into range, but after structural changes verify the cursor still sits where you expect.
- Enter never selects a branch — it toggles it. Only leaves (nodes without non-empty `children`) produce `[:selected, node]`.

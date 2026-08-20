---
title: Filepicker
layout: default
parent: Components
nav_order: 5
permalink: /docs/components/filepicker/
---
# Filepicker

`Charming::Components::Filepicker` is a directory browser built on [List]({{ '/docs/components/list/' | relative_url }}). Enter descends into the highlighted directory or returns the absolute path of a file; Backspace goes up, never above the configured root. Reach for it when the user needs to pick a file from disk. It is interactive — give it a focus slot.

```text
> ../
  lib/
  spec/
  README.md
```

Below the root, a `../` entry leads back up; directories (with a `/` suffix) list before files, each group alphabetical.

## Quick start

The picker tracks its own `current_dir` as you browse, so keep the *instance* around in a memoized ivar instead of rebuilding it:

```ruby
class OpenFileController < ApplicationController
  focus_ring :picker

  on_select :picker, :open_file

  def show
    render :show, file_picker: picker
  end

  # Enter on a file returns its absolute path.
  def open_file(path)
    open_state.path = path
    navigate :editor
  end

  def picker
    @picker ||= Charming::Components::Filepicker.new(
      root: Dir.pwd,
      height: 12,
      theme: theme
    )
  end

  private

  def open_state
    state(:open_file, OpenFileState)
  end
end
```

```erb
<%= render_component file_picker %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `root:` | `Dir.pwd` | The starting directory and the upper boundary — Backspace never ascends above it. |
| `show_hidden:` | `false` | Include dotfiles from the start. |
| `height:` | `nil` | Windows the listing like List's `height:`, auto-scrolling to keep the highlight in view. |
| `theme:` | `nil` | Theme for the highlighted row. |
| `keymap:` | `:vim` | Passed through to the underlying List — `j`/`k` aliases; `nil` disables. |

## Keyboard

| Key | Action |
|-----|--------|
| `up` / `k` | Move the highlight up. |
| `down` / `j` | Move the highlight down. |
| `home` / `end` | Jump to first / last entry. |
| `enter` | Descend into a directory (or `../`), or select a file. |
| `backspace` | Go up one directory (no-op at the root). |

`j`/`k` come from the default `keymap: :vim`.

## What it returns

| Return value | Meaning |
|--------------|---------|
| `[:selected, absolute_path]` | Enter on a file — dispatches the action declared with `on_select` (the action receives the file's absolute path). |
| `:handled` | A descend, ascend, or navigation key was consumed. |
| `nil` | The event was not handled (e.g. Backspace at the root). |

See the shared [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `current_dir` — the directory currently being browsed.
- `entries` — the display entries for the current directory: optional `../`, then `name/` directories before files.
- `toggle_hidden` — shows or hides dotfiles and refreshes the listing (returns `self`). Wire it to a key binding, e.g. `key ".", :toggle_hidden`.

## Tips

- **Keep the instance, not just an index.** Unlike List, the picker's position — `current_dir` and the highlight within it — lives inside the component, and there is no constructor argument to restore `current_dir`. The memoized ivar (as in the quick start) keeps it for the screen's lifetime; stash the component in a state attribute (`open_state.picker ||= ...`) when browsing must survive navigation away and back.
- The `root:` boundary is a hard floor: Backspace and the `../` entry both refuse to leave it. Pass `/` (or a suitably high directory) if the user should roam.
- Directory entries end with `/` in `entries`; the selected *file* path handed to your action is absolute and has no suffix.

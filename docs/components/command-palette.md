---
title: CommandPalette
layout: default
parent: Components
nav_order: 14
permalink: /docs/components/command-palette/
---
# CommandPalette

`Charming::Components::CommandPalette` is a fuzzy-searchable command picker — a TextInput for the query stacked on a List of matching commands. The framework uses it for the `ctrl+p` palette and the theme picker, and you can reuse it as an ordinary component. It captures text: while it is focused, printable characters filter the list instead of firing key bindings (typing `q` narrows the query, it doesn't quit), and while the *built-in* palette is open it sits at the top of the key-dispatch ladder — it consumes every key before global and content `key` bindings.

```text
┌ Command palette ─────────────────┐
│ th|                              │
│ > Theme                          │
│   Stats                          │
│ Type to filter. Enter selects.   │
└──────────────────────────────────┘
```

That's the palette wrapped in `CommandPaletteModal` (see below); bare, it renders just the input row and the result list — or "No commands found" when nothing matches.

## Quick start

Most apps never build the component by hand — they use the opt-in shell integration. Include `Charming::Shell::Palette` in `ApplicationController` (generated apps get this from `charming generate layout`), declare commands with the `command` class DSL, and bind a key to `open_command_palette` (from the journal example):

```ruby
class ApplicationController < Charming::Controller
  include Charming::Shell::Palette

  key "ctrl+p", :open_command_palette, scope: :global

  command "Entries" do
    navigate :root
  end
  command "Compose" do
    navigate :compose
  end
  command "Theme", :open_theme_palette
  command "Quit app", :quit

  def show
    render :show, palette: command_palette   # nil when closed
  end
end
```

While open, the framework routes every key through the palette, persists its state under `session[:command_palette]`, runs the selected command (a block is `instance_exec`'d on the controller; a symbol is `send`'ed), and closes on Escape. Your only rendering job is compositing it as an overlay:

```ruby
# In a layout or view
def command_palette_modal
  return unless palette_component

  render_component Charming::Components::CommandPaletteModal.new(
    content: palette_component,
    theme: theme
  )
end
```

To use the component standalone, build it yourself and memoize it in an ivar; seed it from its persisted `state` hash when it must survive navigation:

```ruby
Charming::Components::CommandPalette.new(
  commands: [
    Charming::Components::CommandPalette::Command.new(label: "Deploy", value: :deploy),
    Charming::Components::CommandPalette::Command.new(label: "Rollback", value: :rollback)
  ],
  value: saved[:value], cursor: saved[:cursor], selected_index: saved[:selected_index],
  height: 6, theme: theme
)
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `commands:` | required | Array of `CommandPalette::Command` entries (`Data.define(:label, :value)` — a display label plus a callable or method symbol). |
| `placeholder:` | `"Search commands"` | Shown while the query is empty. |
| `height:` | `nil` | Visible rows in the result list (it scrolls). |
| `value:` | `""` | Initial query, seeding the inner TextInput. |
| `cursor:` | end of value | Query cursor offset. |
| `selected_index:` | `0` | Highlighted result row. |
| `theme:` | app theme | Styles the list rows. |

Filtering is `FuzzyMatcher.filter(query, commands)` — an fzf-style subsequence scorer over the labels, best matches first. An empty query shows all commands.

## Keyboard

| Key | Action |
|-----|--------|
| printable characters | Edit the query and re-filter the list. |
| `↑` / `↓` / `Home` / `End` | Move the result highlight (delegated to the List). |
| `Enter` | Select the highlighted command — returns `[:selected, command]`. |
| `Escape` | Cancel — returns `:cancelled`. |
| `←` / `→`, `Backspace`, `Delete` | Edit the query via the inner TextInput. |

## What it returns

`handle_key` returns `[:selected, command]` on Enter (the whole `Command` object — read `command.value` to act on it), `:cancelled` on Escape, `:handled` for consumed keys, and `nil` otherwise. Used as a focus slot, these dispatch the actions declared with `on_select` and `on_cancel`; the shell's `ctrl+p` integration instead performs the command and closes the palette for you. See the [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## The modal wrapper

`Charming::Components::CommandPaletteModal` wraps palette content in the framework's standard modal chrome (a `Modal` with title, body, and help line):

| Option | Default | What it does |
|--------|---------|--------------|
| `content:` | required | The rendered palette (component or string). |
| `title:` | `"Command palette"` | Modal title. |
| `help:` | `"Type to filter. Enter selects. Escape closes."` | Help line under the body. |
| `width:` | `52` | Modal width. |
| `style:` / `theme:` | `nil` / app theme | Modal styling overrides. |

It is purely presentational — key handling stays with the `CommandPalette` inside it.

## Working with it

- `state` — `{value:, cursor:, selected_index:}`, the exact hash to persist in the session and feed back to `new`.
- `selected_command` — the currently highlighted `Command`, or nil.
- `commands` / `input` — the full command list and the inner TextInput.
- Controller helpers from the palette module: `open_command_palette`, `close_command_palette`, `command_palette_open?`, and `command_palette` (builds the component from session state, nil when closed).

## Tips

- The shell integration persists `palette.state` after every dispatch for you. Standalone usage gets screen-lifetime state from a memoized ivar; persist the `state` hash by hand only to survive navigation.
- `command` bindings are inherited by subclasses, so declare app-wide commands once on `ApplicationController` and add screen-specific ones per controller.
- Remember the priority rule: an open palette consumes *everything*, including keys you bound with `key ... scope: :global`. Close it (Escape or `close_command_palette`) before expecting bindings to fire.

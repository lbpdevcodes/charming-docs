---
title: Layouts
layout: default
nav_order: 6
permalink: /docs/layouts/
---
# Layouts

Layouts wrap screen views with shared UI such as sidebars, headers, footers, command palettes, and modal overlays.

Generated apps use Ruby layout classes and the declarative layout DSL:

```ruby
class ApplicationController < Charming::Controller
  layout Layouts::ApplicationLayout
  focus_ring :sidebar, :content
end
```

That resolves:

```text
app/views/layouts/application_layout.rb
```

ERB layouts remain available as a fallback with `layout "layouts/application"`.

## Layout Class

Layout classes inherit from `Charming::View`:

```ruby
module MyApp
  module Layouts
    class ApplicationLayout < Charming::View
      def render
        screen_layout(background: theme.background) do
          split :horizontal, gap: 1 do
            pane(:sidebar, width: 24, border: :rounded, padding: [1, 2]) do
              navigation
            end

            pane(:content, grow: 1, border: :rounded, padding: [1, 2]) do
              yield_content
            end
          end
        end
      end

      private

      def navigation
        column("Home", "Settings", gap: 1)
      end
    end
  end
end
```

## Layout Assigns

Layouts receive standard assigns:

| Assign | Purpose |
|--------|---------|
| `content` | The already-rendered screen body. |
| `screen` | Current terminal dimensions. |
| `controller` | Current controller instance. |
| `theme` | Active theme. |

Any assigns passed to the screen view are also available to the layout.

Use `yield_content` to place the current screen inside the layout.

## DSL Primitives

`screen_layout` creates a full-screen layout tree for the current terminal size.

```ruby
screen_layout(background: theme.background) do
  split :horizontal, gap: 1 do
    pane(:sidebar, width: 24) { "Sidebar" }
    pane(:content, grow: 1) { |rect| "Content area: #{rect.width}x#{rect.height}" }
  end
end
```

Available primitives:

| Primitive | Purpose |
|-----------|---------|
| `screen_layout(background: nil) { ... }` | Render a full-screen layout using the current `screen`. |
| `split(:horizontal, gap: n) { ... }` | Divide space into left-to-right panes. |
| `split(:vertical, gap: n) { ... }` | Divide space into top-to-bottom panes. |
| `pane(:name, **options) { ... }` | Render content into an assigned rectangle. |
| `overlay(content, top: :center, left: :center)` | Draw content over the finished layout. |

Pane sizing options:

| Option | Behavior |
|--------|----------|
| `width: n` | Fixed outer width in a horizontal split. |
| `height: n` | Fixed outer height in a vertical split. |
| `grow: n` | Take remaining space, weighted by `n`. |
| `min_width: n` / `max_width: n` | Clamp the computed width (horizontal splits). |
| `min_height: n` / `max_height: n` | Clamp the computed height (vertical splits). |
| no size | Equivalent to `grow: 1` inside a split. |

Constraints apply after grow distribution: a clamped pane's surplus or deficit is
re-balanced onto the remaining flexible children, so `pane(:sidebar, width: 24,
min_width: 18)` keeps a usable sidebar on narrow terminals while `grow` panes absorb
the difference.

Pane styling options:

| Option | Behavior |
|--------|----------|
| `border: true` | Draw a normal border. |
| `border: :rounded` | Draw a named border style. |
| `padding: 1` | Add equal padding on all sides. |
| `padding: [1, 2]` | Add vertical and horizontal padding. |
| `style: theme.title` | Apply a base style. |
| `focus: true` | Include this pane in the layout's Tab focus ring. |
| `focused_style: theme.title` | Style the pane when it is focused. Defaults to `theme.title`. |
| `clip: true` | Clip content to the pane. This is the default. |
| `scroll: true` | Wrap the pane's content in a scrollable `Viewport` so tall content can scroll (the content must be a renderable component). |
| `wrap: true` | Wrap long lines inside the pane. |

Pane dimensions are outer dimensions. Borders and padding are included in the assigned width and height.

Pane blocks may accept a `Charming::Layout::Rect` argument. The yielded rect is the pane's inner content area after border and padding are applied, so components can render to their actual available width and height:

```ruby
pane(:content, grow: 1, border: :rounded, padding: [1, 2]) do |rect|
  render_component FeedList.new(width: rect.width, height: rect.height)
end
```

Focusable panes are opt-in. Named panes are not focusable unless `focus: true` is set:

```ruby
screen_layout do
  split :horizontal, gap: 1 do
    pane(:files, width: 32, border: :rounded, focus: true) { files_panel }
    pane(:diff, grow: 1, border: :rounded, focus: true) { diff_panel }
  end
end
```

The first focusable pane starts focused. `Tab` cycles forward and `Shift+Tab` cycles backward. Command palettes and other modal scopes still take priority while open.

## Screen Views

Screens are Ruby view classes by default:

```ruby
module MyApp
  module Home
    class ShowView < Charming::View
      def render
        column(
          text(home.title, style: theme.title),
          text("Press ctrl+p for commands, q to quit.", style: theme.muted),
          gap: 1
        )
      end
    end
  end
end
```

The controller can still render by action name:

```ruby
def show
  render :show, home: home, palette: command_palette
end
```

For `HomeController`, that resolves `MyApp::Home::ShowView` before falling back to ERB templates.

## Responsive Layouts

Use normal Ruby methods and `screen` dimensions:

```ruby
def render
  screen_layout do
    split(narrow? ? :vertical : :horizontal, gap: 1) do
      pane(:sidebar, **sidebar_options, border: :rounded, padding: [1, 2]) do
        navigation
      end

      pane(:content, grow: 1, border: :rounded, padding: [1, 2]) do
        yield_content
      end
    end
  end
end

private

def narrow?
  screen.narrow?(below: 72, min_height: 20)
end

def sidebar_options
  narrow? ? {height: [screen.height / 3, 5].max} : {width: 24}
end
```

## Mouse Interaction

Every render registers the layout's named panes as mouse targets (each pane's
rectangle plus its slot name). When a mouse event arrives, the runtime hit-tests it
against those rectangles — the topmost matching pane wins — so clicks map back to the
same slots used by Tab traversal:

- Clicking a **focusable** pane (`focus: true`) moves focus to that slot, so you get
  click-to-focus for free.
- If a method named after the slot returns a component that implements
  `handle_mouse(event)`, that component receives the event with pane-local
  coordinates, and its return value maps to the slot's hooks (`<slot>_selected`, etc.).

```ruby
screen_layout do
  split(:horizontal) do
    pane(:files, width: 32, focus: true) { files_list }   # click focuses this pane
    pane(:preview, grow: 1, focus: true) { preview }
  end
end
```

`MouseEvent` exposes `x`/`y`, modifier flags, and the predicates `click?`, `scroll?`,
and `release?`. For component-level mouse handling and the return conventions, see the
[Components]({{ '/docs/components/' | relative_url }}#mouse-events) guide.

## Modal Overlays

Use `overlay` inside `screen_layout` for command palettes, dialogs, tooltips, or toasts:

```ruby
def render
  screen_layout(background: theme.background) do
    split :horizontal, gap: 1 do
      pane(:sidebar, width: 24, border: :rounded) { navigation }
      pane(:content, grow: 1, border: :rounded) { yield_content }
    end

    overlay command_palette_modal if command_palette_modal
  end
end

private

def command_palette_modal
  return unless palette

  render_component Charming::Components::CommandPaletteModal.new(
    content: palette,
    theme: theme
  )
end
```

Overlays accept `z_index:` for stacking order — higher values composite on top, and
ties keep registration order. Useful when a toast must float above an open modal:

```ruby
overlay help_modal, z_index: 5 if help_modal
overlay toast, top: screen.height - 5, left: :center, z_index: 10 if toast
```

## Status Bar

The classic bottom bar is a one-row pane wrapping the whole layout in an outer
vertical split. (`status_hints` here is an app-defined controller method returning
`[key, description]` pairs — each screen can override it.)

```ruby
screen_layout(background: theme.background) do
  split :vertical do
    split :horizontal, gap: 1, grow: 1 do
      pane(:sidebar, width: 24, min_width: 18, border: :rounded) { navigation }
      pane(:content, grow: 1, border: :rounded) { yield_content }
    end

    pane(:status_bar, height: 1) do
      render_component Charming::Components::StatusBar.new(
        width: screen.width,
        left: " #{controller.route&.title}",
        hints: controller.status_hints,
        theme: theme
      )
    end
  end
end
```

## Lower-Level Helpers

The DSL sits above the lower-level string helpers. Use these inside panes and screen views:

| Helper | Purpose |
|--------|---------|
| `text(value, style: nil)` | Render styled text. |
| `box(value, style: nil)` | Style or border a block manually. |
| `row(*items, gap: 0)` | Join blocks side by side. |
| `column(*items, gap: 0)` | Stack blocks vertically. |
| `render_component(component)` | Render reusable components. |

Use `Charming::UI.place`, `center`, and `overlay` only when you need lower-level canvas control.

## Style Chaining

`Charming::UI::Style` objects are immutable and chainable:

```ruby
panel_style = Charming::UI.style
  .foreground(:bright_cyan)
  .background("#101820")
  .bold
  .border(:rounded, foreground: :bright_magenta)
  .padding(1, 2)
  .width(40)
  .align(:center)

box("System ready", style: panel_style)
```

Common style methods:

| Method | Purpose |
|--------|---------|
| `foreground(color)` / `fg(color)` | Set text color. |
| `background(color)` / `bg(color)` | Set background color. |
| `bold`, `faint`, `italic`, `underline`, `reverse`, `strikethrough` | Add text attributes. |
| `padding(1)`, `padding(1, 2)`, `padding(1, 2, 1, 2)` | Add CSS-like padding. |
| `border(:normal | :rounded | :thick | :double)` | Add a border. |
| `border(..., sides: [:top, :bottom])` | Draw only specific sides. |
| `width(value)` | Set content width. |
| `height(value)` | Set content height. |
| `align(:left | :center | :right)` | Align content within width. |

Colors can be named symbols (`:cyan`, `:bright_white`), 0-255 indexed colors, or truecolor hex strings (`"#ff00aa"`).

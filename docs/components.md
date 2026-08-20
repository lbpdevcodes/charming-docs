---
title: Components
layout: default
nav_order: 9
has_children: true
permalink: /docs/components/
---
# Components

Components are reusable UI objects — the widgets you compose into views. They
come in two kinds:

- **Interactive** components sit in a controller's focus ring, handle key and
  mouse events, and report back through return values — a `List` you arrow
  through, a `TextInput` you type into, a `Form` you submit.
- **Static renderables** just draw — a `Badge`, a `Sparkline`, a `StatusBar`.
  Some animate when you drive them from a timer (`Spinner`, `Progressbar`) —
  for physics-based motion, see [Animation]({{ '/docs/animation/' | relative_url }}).

Every component inherits from `Charming::Component`, so assigns passed to
`new` become reader methods and `render` can use the view DSL. Render any of
them from a template or view with `render_component`:

```erb
<%= render_component Charming::Components::List.new(
  items: ["Alpha", "Beta", "Gamma"],
  selected_index: 0,
  theme: theme
) %>
```

One rule underlies everything: **components have no lifecycle**. Keep a
component with interaction state in a memoized controller ivar, where it lives
for the screen's lifetime. Anything that must survive navigation (a selected
index, a filter query, text in a field) lives in controller
[state]({{ '/docs/state/' | relative_url }}) or `session` and gets passed back
in through the constructor. Each component page shows this idiom.

## Pickers & navigation

| Component | What it does |
|-----------|--------------|
| [List]({{ '/docs/components/list/' | relative_url }}) | Selectable list with keyboard navigation, mouse support, and fuzzy filtering. |
| [MultiSelectList]({{ '/docs/components/multi-select-list/' | relative_url }}) | List with `[x]` checkboxes — Space toggles, Enter submits the checked set. |
| [Table]({{ '/docs/components/table/' | relative_url }}) | Unicode data table with a scrolling window, sortable columns, and row selection. |
| [Tree]({{ '/docs/components/tree/' | relative_url }}) | Collapsible hierarchy — expand and collapse branches, select leaves. |
| [Filepicker]({{ '/docs/components/filepicker/' | relative_url }}) | Directory browser — descend into folders, pick a file. |
| [TabBar]({{ '/docs/components/tab-bar/' | relative_url }}) | Horizontal tabs — arrow between them, Enter or click selects. |
| [Breadcrumbs]({{ '/docs/components/breadcrumbs/' | relative_url }}) | `Home › Projects › Current` trail with the last item highlighted. |
| [Paginator]({{ '/docs/components/paginator/' | relative_url }}) | Page tracker rendering `○ ● ○` dots or `2/3`; slices the current page for you. |
| [Viewport]({{ '/docs/components/viewport/' | relative_url }}) | Scrollable window over tall content, with wrapping and horizontal scroll. |

## Text & input

| Component | What it does |
|-----------|--------------|
| [TextInput]({{ '/docs/components/text-input/' | relative_url }}) | Single-line text field — masking for passwords, shell-style history, paste support. |
| [TextArea]({{ '/docs/components/text-area/' | relative_url }}) | Multiline editor — Enter inserts a newline, wide-character aware. |
| [Autocomplete]({{ '/docs/components/autocomplete/' | relative_url }}) | Text input with a live-filtered suggestion dropdown. |
| [Form]({{ '/docs/components/form/' | relative_url }}) | Multi-field form with inputs, selects, confirms, validation, and submit/cancel. |
| [CommandPalette]({{ '/docs/components/command-palette/' | relative_url }}) | Fuzzy-search command picker, plus the modal chrome that frames it. |

## Overlays & messaging

| Component | What it does |
|-----------|--------------|
| [Modal]({{ '/docs/components/modal/' | relative_url }}) | Centered overlay dialog with title, help text, and an optionally scrollable body. |
| [Toast]({{ '/docs/components/toast/' | relative_url }}) | Auto-dismissing notification box with info/success/warn/error accents. |
| [HelpOverlay]({{ '/docs/components/help-overlay/' | relative_url }}) | Keyboard cheat-sheet modal, buildable straight from a controller's key bindings. |
| [StatusBar]({{ '/docs/components/status-bar/' | relative_url }}) | One-row bar with left/center/right segments and key-hint pairs. |
| [Badge]({{ '/docs/components/badge/' | relative_url }}) | Inline styled pill for statuses, counts, and versions. |
| [EmptyState]({{ '/docs/components/empty-state/' | relative_url }}) | "Nothing here yet" placeholder with loading and error variants. |
| [ErrorScreen]({{ '/docs/components/error-screen/' | relative_url }}) | The panel the runtime renders for unhandled exceptions. |

## Progress & time

| Component | What it does |
|-----------|--------------|
| [Spinner]({{ '/docs/components/spinner/' | relative_url }}) | Animated frame-cycling indicator with named presets — `:dots`, `:moon`, `:meter`, … |
| [ActivityIndicator]({{ '/docs/components/activity-indicator/' | relative_url }}) | Gradient activity bar with label and ellipsis animation. |
| [Progressbar]({{ '/docs/components/progressbar/' | relative_url }}) | Text progress bar with optional gradient fill and percent tracking. |
| [Timer]({{ '/docs/components/timer/' | relative_url }}) | Countdown clock (`mm:ss`) — tick it from a controller timer. |
| [Stopwatch]({{ '/docs/components/stopwatch/' | relative_url }}) | Count-up clock — start, stop, reset; accumulates only while running. |

## Data & media

| Component | What it does |
|-----------|--------------|
| [Chart]({{ '/docs/components/chart/' | relative_url }}) | Line charts (braille subpixels) and bar charts in a fixed-size box. |
| [Sparkline]({{ '/docs/components/sparkline/' | relative_url }}) | One-line `▁▂▄▇` bar graph, one cell per value. |
| [Markdown]({{ '/docs/components/markdown/' | relative_url }}) | CommonMark/GFM renderer with syntax highlighting and clickable links. |
| [Image]({{ '/docs/components/image/' | relative_url }}) | Inline terminal images on Kitty/Ghostty, with a text fallback everywhere else. |
| [Audio]({{ '/docs/components/audio/' | relative_url }}) | One-line playback-status indicator for an audio player. |

## How components talk to controllers

Interactive components return values from `handle_key` / `handle_mouse`, and
the runtime dispatches them to controller actions declared for the focus slot:

| Return value | Controller action |
|--------------|-------------------|
| `:handled` | — (event consumed) |
| `[:selected, object]` | declared with `on_select :slot, :action` — the action receives the object |
| `[:submitted, value]` | declared with `on_submit :slot, :action` — the action receives the value |
| `:cancelled` | declared with `on_cancel :slot, :action` — no arguments |
| `nil` | — (event falls through) |

The full protocol — key handling, text capture, paste, mouse events, theming —
lives in [Build Your Own]({{ '/docs/components/build-your-own/' | relative_url }}),
along with `charming generate component` for scaffolding your own.

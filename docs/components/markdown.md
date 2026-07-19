---
title: Markdown
layout: default
parent: Components
nav_order: 29
permalink: /docs/components/markdown/
---
# Markdown

`Charming::Components::Markdown` renders CommonMark/GFM source as ANSI-styled terminal text. Parsing is handled by Commonmarker, code block tokenization by Rouge, and Charming owns the terminal rendering, wrapping, and Glamour-inspired styling. It is a static renderable — no keyboard handling — so for scrollable documents you pair it with `Viewport`. Reach for it whenever you have READMEs, help text, or any prose authored in Markdown that should look good in the terminal.

```text
Getting started

  Install the gem, then run the generator:

  │ 1  gem install charming
  │ 2  charming new myapp

  • Bold, italic, and strikethrough render as ANSI styles
  • Links render as text <https://example.com>
```

## Quick start

```erb
<%= render_component Charming::Components::Markdown.new(
  content: readme,
  width: 72,
  theme: theme,
  style: :dark
) %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `content:` | required | The Markdown source string. |
| `width:` | `nil` | Wrap width in cells. `nil` leaves prose unwrapped. |
| `theme:` | `nil` | Forwarded to the view layer. |
| `syntax_highlighting:` | `true` | Rouge highlighting inside fenced code blocks. |
| `style:` | `:dark` | Markdown color style — `:dark`, `:light`, `:notty`, or `:auto`. |
| `base_url:` | `nil` | Base URL for resolving relative link and image targets. |
| `hyperlinks:` | `false` | Emit OSC 8 escapes so links are clickable in modern terminals. |

## Styles

The built-in styles are `:dark`, `:light`, `:notty`, and `:auto` — which picks dark or light from the detected terminal background. `:notty` renders without color, for plain-text output.

GFM tables, task lists, strikethrough, autolinks, links, images, definition lists, and footnotes render as terminal-friendly text:

- **Definition lists** (`Term` / `: definition`) render bold terms with indented, wrapped descriptions.
- **Footnotes** render `[name]` references inline and labeled definition blocks with hanging indentation.
- **Task lists** detect the checked state from the list marker only — prose mentioning `[x]` doesn't check a box.
- **Raw HTML blocks** render as nothing by default (matching Glamour); a custom style config can set `html_block: {format: "..."}` to show a placeholder.

## Hyperlinks

Make links clickable in modern terminals with `hyperlinks: true` — links are wrapped in OSC 8 escapes and the ` <url>` suffix is dropped since the target is embedded. Off by default so test captures and older terminals stay clean:

```erb
<%= render_component Charming::Components::Markdown.new(
  content: readme,
  hyperlinks: true
) %>
```

## Relative links with base_url

Relative link and image targets can be resolved with `base_url:`:

```erb
<%= render_component Charming::Components::Markdown.new(
  content: readme,
  width: 72,
  style: :dark,
  base_url: "https://example.com/docs/"
) %>
```

## Scrolling with Viewport

Use it with `Viewport` for scrollable documentation or help screens:

```erb
<%= render_component Charming::Components::Viewport.new(
  content: Charming::Components::Markdown.new(content: readme, width: 72, theme: theme, style: :dark),
  width: 72,
  height: 20
) %>
```

## Disabling syntax highlighting

Disable code syntax highlighting when plain code blocks are preferred:

```erb
<%= render_component Charming::Components::Markdown.new(
  content: readme,
  syntax_highlighting: false
) %>
```

## Tips

- Width matters: without `width:`, paragraphs render at their source line length and long lines will overflow or be clipped by the layout. Pass the width of the box the Markdown lives in.
- Horizontal rules and tables need room too — they size from `width:` (rules fall back to 40 cells when it's unset).
- The component is static; scrolling, and any keyboard behavior, comes from the wrapping `Viewport`.

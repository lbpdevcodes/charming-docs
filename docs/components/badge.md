---
title: Badge
layout: default
parent: Components
nav_order: 19
permalink: /docs/components/badge/
---
# Badge

Badge is a small inline pill for statuses, counts, and versions — a space-padded label with a style applied, meant to sit inside status bars, list rows, and headers. It is a static renderable.

```text
 v1.2 
```

rendered in `theme.selected` (reverse video) by default, so it reads as a filled pill.

## Quick start

Note the positional `label` argument — Badge takes the label first, not as a keyword:

```erb
<%= render_component Charming::Components::Badge.new("v1.2", theme: theme) %>
```

Style it for meaning:

```ruby
Charming::Components::Badge.new("3 errors", style: theme.warn)
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `label` (positional) | required | The badge text; anything responding to `to_s`. |
| `style:` | `nil` | Overrides the pill style. Default is `theme.selected`, which guarantees contrast. |
| `theme:` | app default | Theme the default style comes from. |

## Tips

- The pill is just `" label "` with a style — compose it inline with `row(...)` next to text; it adds no border or extra height.

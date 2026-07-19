---
title: EmptyState
layout: default
parent: Components
nav_order: 20
permalink: /docs/components/empty-state/
---
# EmptyState

EmptyState is the placeholder for screens with nothing to show yet. One component, three states: a default "nothing here" message, a loading message, or an error message with optional help text. It is a static renderable.

```text
Nothing to show.
```

```text
Could not load entries.
r retries
```

Default and loading states render muted; the error message renders in `theme.warn` with the help line muted below it.

## Quick start

From the journal example's entries screen — swap the list for an EmptyState when there are no items:

```ruby
def body
  return empty_state if entries_list.items.empty?

  render_component(entries_list)
end

def empty_state
  render_component(
    Charming::Components::EmptyState.new(
      message: "No entries yet — press n to write your first.",
      theme: theme
    )
  )
end
```

Loading and error variants:

```ruby
Charming::Components::EmptyState.new(loading: true, loading_message: "Fetching entries...")
Charming::Components::EmptyState.new(error: exception, help: "r retries")
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `message:` | `"Nothing to show."` | Text for the default state. |
| `loading:` | `false` | Switches to the loading state (wins over the error state). |
| `loading_message:` | `"Loading..."` | Text shown while loading. |
| `error:` | `nil` | An error (exception or string) — a non-blank value switches to the error state and renders via `to_s`. |
| `error_message:` | `nil` | Explicit error text; takes precedence over `error.to_s`. |
| `help:` | `nil` | Muted line under the error message (error state only). |
| `theme:` | app default | Theme for the muted/warn styles. |

## Tips

- State precedence is loading, then error, then the default message — `loading: true` hides an error you also passed.
- `help:` only shows in the error state; it is ignored for the default and loading messages.

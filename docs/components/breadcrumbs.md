---
title: Breadcrumbs
layout: default
parent: Components
nav_order: 7
permalink: /docs/components/breadcrumbs/
---
# Breadcrumbs

`Charming::Components::Breadcrumbs` renders a navigation trail — where the user is and how they got there. It is a static renderable: render-only, no focus slot, no key or mouse handling.

```text
Home › Projects › My App
```

Ancestors and the separators render in the theme's muted style; the final (current) item renders in the title style.

## Quick start

Build it inline where you render — there's no state to persist:

```ruby
class ProjectsController < ApplicationController
  def show
    render :show, project: project, trail: breadcrumbs
  end

  private

  def breadcrumbs
    Charming::Components::Breadcrumbs.new(
      items: ["Home", "Projects", project.name],
      theme: theme
    )
  end
end
```

```erb
<%= render_component trail %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `items:` | (required) | The trail, first-to-last — strings or anything responding to `to_s`. |
| `separator:` | `" › "` | String joining the items (rendered muted). |
| `theme:` | `nil` | Theme supplying the muted (ancestors, separator) and title (current item) styles. |

## Tips

- An empty `items:` array renders an empty string — no placeholder, no error.
- The last item is always the highlighted "current" one; there's no way to mark a different item, so build the trail in path order.
- It's a one-line, fire-and-forget renderable — rebuild it fresh each render from your route or navigation state.

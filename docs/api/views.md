---
title: Views & Templates
layout: default
parent: API Reference
nav_order: 4
permalink: /docs/api/views/
---
# Views & Templates

Views turn data into terminal output. The default path is a **class-based view** — a plain Ruby class with a `render` method and a toolbox of layout helpers. When no view class exists (or you ask for one explicitly), Charming falls back to **ERB templates** under `app/views`. This page covers the view base class and its helpers, then the template resolution machinery beneath the fallback.

## Class-based views

Class-based views are the default. Inherit from `Charming::View` and implement `render`:

```ruby
module MyApp
  module Home
    class ShowView < Charming::View
      def render
        text title, style: theme.title
      end
    end
  end
end
```

For `render :show` in `HomeController`, Charming resolves `MyApp::Home::ShowView` first.

## Charming::View

Assigns passed to `new` become reader methods:

```ruby
HomeView.new(title: "Home", theme: theme)
```

View and template helpers:

- `text(value, style: nil)` renders text through an optional style.
- `box(value, style: nil)` renders boxed or styled content.
- `box(style: style) { ... }` captures nested helper output into a styled block.
- `row(*items, gap: 0)` joins rendered items horizontally.
- `column(*items, gap: 0)` joins rendered items vertically.
- `screen_layout(background: nil) { ... }` renders a full-screen declarative layout tree with `split`, `pane`, and `overlay`. `pane` blocks may accept a `Charming::Layout::Rect` argument for the pane's inner content area.
- `style` returns a new `Charming::UI::Style`.
- `theme` returns the assigned theme or default theme.
- `render_component(component)` renders a component.
- `render_partial(view)` renders another view.
- `yield_content` returns layout content.
- `layout_assigns` returns assigns used when composing layouts.
- `focused?(slot)` delegates focus lookup to the controller assign.

## Template fallback

`Charming::Templates` resolves and renders ERB templates under `app/views` when no conventional view class exists or when `render_template` is used.

Template APIs:

- `Charming::Templates.register(extension, handler)` registers a template handler.
- `Charming::Templates.resolve(name, root:)` resolves a template from an app root.
- `Charming::Templates::MissingTemplateError` is raised when no candidate file exists.

Registered extensions:

- `.tui.erb`
- `.txt.erb`

For `Templates.resolve("home/show", root: app_root)`, Charming searches:

```text
app/views/home/show.tui.erb
app/views/home/show.txt.erb
```

`.tui.erb` is preferred before `.txt.erb`.

Template handlers implement:

```ruby
def self.render(path, view)
  # return rendered string
end
```

## Charming::TemplateView

`Charming::TemplateView` renders resolved templates with normal view helpers and assigns:

```ruby
template = Charming::Templates.resolve("home/show", root: app_root)
view = Charming::TemplateView.new(template: template, home: home, theme: theme)
view.render
```

Constructor:

- `template:` is a resolved template.
- `namespace:` optionally controls constant lookup during template binding.
- `**assigns` become reader methods available inside the template.

Instance APIs:

- `render` renders the template to a string.
- `template_binding` returns the binding used by ERB handlers.

{: .note }
Generated controllers usually do not instantiate `TemplateView` directly. Use Ruby views with `render :show`, or `render_template "path"` for ERB fallback content.

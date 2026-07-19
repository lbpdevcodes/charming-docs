---
title: Image
layout: default
parent: Components
nav_order: 30
permalink: /docs/components/image/
---
# Image

`Charming::Components::Image` displays a `Charming::Image::Source` inline in the terminal, sized to `rows` × `cols` character cells, using the Kitty graphics protocol (Ghostty and Kitty terminals). It is a static renderable: on a graphics-capable terminal it renders a block of Unicode placeholder cells and registers the image's one-time out-of-band transmission; on any other terminal it renders your `fallback:` text so layouts degrade gracefully.

The component is deliberately thin — the `Charming::Image::Source` owns the bytes, the stable image id, and the transmit state. Keep the source in `session` so it is built once; if it were rebuilt every render, the "already transmitted" flag would reset and the image would be re-sent each frame.

## Quick start

Build the source once in the controller, render the component in the view:

```ruby
class GalleryController < Charming::Controller
  def show
    session[:cover] ||= Charming::Image::Source.new(path: "assets/cover.png")
    render :show, image: session[:cover]
  end
end
```

```ruby
class ShowView < Charming::View
  def render
    render_component(Charming::Components::Image.new(
      source: assigns.fetch(:image),
      rows: 15,
      cols: 44,
      fallback: text("[ this terminal has no graphics protocol — try Ghostty or Kitty ]", style: theme.warn),
      theme: theme
    ))
  end
end
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `source:` | required | The `Charming::Image::Source` to display. |
| `rows:` | required | Image height in character cells. |
| `cols:` | required | Image width in character cells. |
| `fallback:` | `""` | Rendered instead of the image when the terminal lacks graphics support. |
| `theme:` | `nil` | Forwarded to the view layer. |

## Building a source

`Charming::Image::Source.new` takes PNG bytes from exactly one of `path:` or `data:`:

```ruby
Charming::Image::Source.new(path: "assets/cover.png")   # read from disk (binary)
Charming::Image::Source.new(data: png_bytes)            # bytes you already have
```

It also accepts `id:` to override the derived image id (normally a stable 24-bit id digested from the bytes) and `terminal:` to inject the protocol-detection seam in tests. Useful methods:

- `supports_graphics?` — true when the host terminal speaks a graphics protocol.
- `release` — builds the escape that frees the image from terminal memory and re-arms the source so a later render retransmits. Register it the same way the component registers a transmit when evicting an image.

## Tips

- Graphics-capable terminals only — Ghostty and Kitty today. Always provide `fallback:`; everywhere else (including CI and test captures) that string is all users see.
- Keep the source in `session`, not rebuilt per render — `transmitted?` gates the one-time transmission.
- The placeholder block is a normal width-`cols` string, so it composes with `row`/`column`/`box` like any other view output.

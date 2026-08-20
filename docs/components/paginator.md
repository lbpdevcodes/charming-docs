---
title: Paginator
layout: default
parent: Components
nav_order: 8
permalink: /docs/components/paginator/
---
# Paginator

`Charming::Components::Paginator` tracks a current page over a collection and renders a compact indicator — bubbles-style dots or an arabic `2/3`. It is a static renderable: it doesn't handle keys or mouse itself. You wire page turns to controller key bindings and pair it with a [List]({{ '/docs/components/list/' | relative_url }}) or [Table]({{ '/docs/components/table/' | relative_url }}) by slicing items through `page_items`.

```text
○ ● ○
```

With `format: :arabic`, the same state renders as `2/3`.

## Quick start

Persist the page in controller state; the paginator does the math and the slicing:

```ruby
class ArchiveController < ApplicationController
  key "right", :next_page
  key "left", :prev_page

  def show
    render :show, entries: paginator.page_items(all_entries), pager: paginator
  end

  def next_page
    archive_state.page = paginator.next_page.page
    show
  end

  def prev_page
    archive_state.page = paginator.prev_page.page
    show
  end

  private

  def paginator
    @paginator ||= Charming::Components::Paginator.new(
      total: all_entries.length,
      per_page: 10,
      page: archive_state.page,
      theme: theme
    )
  end

  def all_entries
    Entry.recent_first.to_a
  end

  def archive_state
    state(:archive, ArchiveState)
  end
end
```

```erb
<%= render_component pager %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `total:` | (required) | The collection size (negative values floor to 0). |
| `per_page:` | (required) | The page size (values below 1 floor to 1). |
| `page:` | `0` | The 0-based current page, clamped to the valid range. |
| `format:` | `:dots` | `:dots` renders `○ ● ○`; `:arabic` renders `2/3`. |
| `theme:` | `nil` | Standard component theme assign (the indicator itself is unstyled text). |

## Working with it

- `page_items(items)` — the slice of `items` belonging to the current page (empty array past the end).
- `next_page` / `prev_page` — step one page, clamping at the ends; both return `self` so you can chain `.page`.
- `page_count` — the number of pages, always at least 1 (an empty collection is a single page: `●`).
- `page`, `per_page`, `total` — readers for the constructor state.

## Tips

- `next_page` returns a new instance — assign it (`@paginator = @paginator.next_page`) or write the page back to controller state (`archive_state.page = paginator.next_page.page`) and pass it in as `page:` on the next build, or the pager resets to page 0.
- `page` is 0-based everywhere in the API; only the `:arabic` rendering is 1-based (`2/3` means `page == 1`).
- Dots render one glyph per page — beyond a dozen or so pages, switch to `format: :arabic`.

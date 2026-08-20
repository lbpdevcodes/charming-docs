---
title: TextInput
layout: default
parent: Components
nav_order: 10
permalink: /docs/components/text-input/
---
# TextInput

`Charming::Components::TextInput` is a single-line text editor: a search box, a title field, a REPL prompt, a password entry. It is interactive and it captures text (`captures_text?` returns true), which means that while it is focused, printable characters route to it before any key binding — typing `q` inserts a q instead of quitting — `tab` reaches it before focus-ring traversal, and ctrl/alt shortcuts like `ctrl+p` keep working.

```text
Search: charm|ing rb
```

The `|` is the rendered cursor marker. When the value is empty, the placeholder renders after it: `|Search…`.

## Quick start

Controllers live for the screen's lifetime, so keep the component in a memoized ivar exposed from a method named after a `focus_ring` slot; the framework dispatches keys to it and maps its results to actions declared with `on_submit`/`on_select`/`on_cancel`.

```ruby
class SearchController < Charming::Controller
  focus_ring :query

  on_submit :query, :run_search

  def show
    render :show, query: query
  end

  def run_search(value)                    # Enter pressed in the field
    session[:last_search] = value
    show
  end

  private

  def query
    @query ||= Charming::Components::TextInput.new(placeholder: "Search…")
  end
end
```

```erb
<%= render_component query %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `value:` | `""` | Initial text. |
| `placeholder:` | `""` | Shown (after the cursor marker) while the value is empty. |
| `width:` | `nil` | Pads the rendered output to this width. |
| `cursor:` | end of value | Cursor offset into the value, clamped to `0..value.length`. |
| `masked:` | `false` | Renders every character as `*` (password entry). The placeholder is not masked. |
| `history:` | `nil` | Array of prior values recalled with up/down, most recent last (like a shell). |

## Keyboard

| Key | Action |
|-----|--------|
| printable characters | Insert at the cursor. |
| `←` / `→` | Move the cursor. |
| `Home` / `End` | Jump to start / end. |
| `Backspace` | Delete before the cursor. |
| `Delete` | Delete at the cursor. |
| `Enter` | Submit — returns `[:submitted, value]`. |
| `↑` / `↓` | Cycle history (only when `history:` was given). |

## What it returns

`handle_key` returns `[:submitted, value]` on Enter, `:handled` for consumed edits, and `nil` for keys it doesn't understand (which then fall through to your key bindings). On a focus slot, `[:submitted, value]` dispatches the action declared with `on_submit :slot, :action` — the action receives the value. There is no cancel path — bind Escape yourself if you need one. See the [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Pasting

The runtime enables bracketed paste, so pasted text arrives as one `Charming::Events::PasteEvent` instead of a storm of key events. `TextInput#handle_paste` inserts the text at the cursor with newlines and all other control characters stripped — it is a single-line input. The focused component receives the event via the controller's `dispatch_paste`.

## Working with it

- `value` / `cursor` — readers for the current text and cursor offset; the memoized component keeps them current across events.
- `handle_key(event)` / `handle_paste(event)` — usually called for you by focus dispatch.
- `captures_text?` — returns true; this is what puts the field ahead of key bindings.

## Tips

- Don't store the component itself in the session — live component objects are dropped on persist. The memoized ivar covers the screen's lifetime; for text that must survive navigation, write `value`/`cursor` primitives into a `state` object and seed the constructor from them.
- `history:` gives REPL-style recall: `↑` steps back (saving your in-progress draft), `↓` steps forward, and stepping past the newest entry restores the draft. Recalled values move the cursor to the end.
- Enter always submits — there is no way to type a newline. Reach for [TextArea]({{ '/docs/components/text-area/' | relative_url }}) when you need multiline text.

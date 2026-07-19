---
title: TextArea
layout: default
parent: Components
nav_order: 11
permalink: /docs/components/text-area/
---
# TextArea

`Charming::Components::TextArea` is a multi-line text editor with cursor movement, scrolling, and a "preferred column" so vertical navigation feels stable across short lines. It is interactive and captures text: while focused, printable characters route to it before any key binding (typing `q` inserts a q instead of quitting), `tab` reaches it before focus-ring traversal, and ctrl/alt shortcuts keep working. Column math is display-width aware, so CJK characters and emoji (which occupy two terminal columns) don't break up/down movement.

```text
Dear diary,
today I wrote a TUI| framework
in Ruby.
```

The `|` is the rendered cursor marker. With `height:`, the buffer scrolls to keep the cursor row visible.

## Quick start

Components are rebuilt on every event, so persist the editor's state in a `component_state` hash and rebuild from it. Expose the component from a memoized method named after a `focus_ring` slot.

```ruby
class NotesController < Charming::Controller
  focus_ring :editor

  key "ctrl+s", :save_note, scope: :global

  def show
    %i[value cursor offset preferred_column].each do |key|
      editor_state[key] = editor.public_send(key)   # write changes back
    end
    render :show, editor: editor
  end

  def save_note
    Note.create!(body: editor.value)
    navigate_to "/"
  end

  private

  def editor
    @editor ||= Charming::Components::TextArea.new(
      value: editor_state[:value],
      cursor: editor_state[:cursor],
      offset: editor_state[:offset],
      preferred_column: editor_state[:preferred_column],
      height: 10,
      placeholder: "Write in Markdown…"
    )
  end

  def editor_state
    component_state(:editor, value: "", cursor: 0, offset: 0, preferred_column: nil)
  end
end
```

```erb
<%= render_component editor %>
```

## Options

| Option | Default | What it does |
|--------|---------|--------------|
| `value:` | `""` | Initial text. |
| `placeholder:` | `""` | Shown (after the cursor marker) while the value is empty. |
| `width:` | `nil` | Clips each visible line to this width and pads with spaces. |
| `height:` | `nil` | Visible row count; the buffer scrolls, and short buffers pad with empty lines. |
| `cursor:` | end of value | Cursor offset into the value. |
| `offset:` | `0` | Top-visible row of the scroll window. |
| `preferred_column:` | `nil` | Display column to resume at on vertical movement (set automatically as you move). |
| `enter_newline:` | `true` | Plain Enter inserts a newline. Set `false` when a host widget wants Enter for itself — the key then falls through unhandled. |

## Keyboard

| Key | Action |
|-----|--------|
| printable characters | Insert at the cursor. |
| `Enter` | Insert a newline (when `enter_newline: true`, the default). |
| `Shift+Enter`, `Ctrl+J`, `Ctrl+N` | Always insert a newline, even with `enter_newline: false`. |
| `←` / `→` / `↑` / `↓` | Move the cursor; vertical movement keeps the preferred column. |
| `Home` / `End` | Jump to start / end of the current line. |
| `Backspace` / `Delete` | Delete before / at the cursor. |
| `PageUp` / `PageDown` | Scroll the buffer by one viewport height. |

Many terminals can't distinguish Shift+Enter from plain Enter, so `Ctrl+N` is the TTY-safe newline when Enter is reserved for the host.

## What it returns

`handle_key` returns `:handled` for consumed keys and `nil` otherwise — a TextArea never submits or cancels on its own. It's a text editor, so it owns Enter; wire submission yourself (a `ctrl+s` binding as above), or use a [Form]({{ '/docs/components/form/' | relative_url }}) textarea field, which gets Tab navigation, Ctrl+S submit, and Escape cancel for free. See the [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Pasting

Bracketed paste delivers pasted text as one `Charming::Events::PasteEvent`. `TextArea#handle_paste` inserts it at the cursor with newlines preserved; carriage returns are stripped (CRLF pastes normalize to `\n`) along with other non-printable characters.

## Working with it

- `value` / `cursor` / `offset` / `preferred_column` — readers for everything you need to persist between events.
- `handle_key(event)` / `handle_paste(event)` — usually called for you by focus dispatch.
- `captures_text?` — returns true.

## Tips

- Persist all four state readers, not just `value` — dropping `offset` makes long buffers jump to the top on every keystroke, and dropping `preferred_column` makes up/down drift on short lines.
- `enter_newline: false` exists for hosts that need Enter (that's how a widget embedding a TextArea can advance/submit on Enter); users can still get newlines via Shift+Enter/Ctrl+J/Ctrl+N.
- Don't store the component in the session — live component objects are dropped on persist. Store primitives and rebuild each event.

---
title: Form
layout: default
parent: Components
nav_order: 13
permalink: /docs/components/form/
---
# Form

`Charming::Components::Form` is a huh-inspired multi-field form with built-in focus traversal, validation, and submit/cancel handling. You rarely construct it directly — the controller's `form(:name) { |f| ... }` helper builds one bound to a state hash kept on the controller instance, so typed input survives between events on the screen. Forms are interactive and capture text: while focused, printable characters route to the active field before any key binding (typing `q` inserts a q instead of quitting), `tab` does field navigation instead of focus-ring traversal, and ctrl/alt shortcuts like `ctrl+s` keep working.

```text
> Name: Ada|
  Bio:
    Programming pioneer.
    |
  Plan: Pro
  Interests: TUI, CLI
  [ ] Accept terms?
    must be accepted
  Enter submits from the last field. Escape cancels.
```

The `>` marks the active field (rendered in the selected style); help lines render muted and error lines in the warn style, each indented under their field.

## Quick start

Adapted from the journal example's `ComposeController`:

```ruby
class ComposeController < ApplicationController
  focus_ring :entry_form, :sidebar

  on_submit :entry_form, :save_entry
  on_cancel :entry_form, :cancel_edit

  def show
    render :show, form: entry_form
  end

  # Ctrl+S (or Enter on the last field) with valid values lands here.
  def save_entry(values)
    entry = Entry.create!(title: values[:title], mood: values[:mood], body: values[:body])
    reset_form(:entry)                       # clear the draft
    navigate :entry, id: entry.id
  end

  def cancel_edit
    reset_form(:entry)
    navigate :root
  end

  def entry_form
    form(:entry) do |f|
      f.input :title, required: true, placeholder: "What happened today?"
      f.select :mood, options: Entry::MOODS
      f.textarea :body, height: 10, placeholder: "Write in Markdown…"
      f.confirm :favorite, label: "Mark as favorite"
      f.note "enter for new line in body · tab next field · ctrl+s save · esc cancel"
    end
  end
end
```

```erb
<%= render_component form %>
```

`form(:entry)` scopes the form's mutable state to a hash kept on the controller instance — only primitive values are stored there, and the component is rebuilt from them on each dispatch. Zero-arity blocks are also supported and are `instance_eval`'d against the builder (`form(:entry) { input :title }`).

## Field types

All field declarations share these options (from the `Field` base class):

| Option | Default | What it does |
|--------|---------|--------------|
| `label:` | humanized name | Display label (`:full_name` → "Full name"). |
| `required:` | `false` | Adds an "is required" validation when the value is blank. |
| `validate:` | `nil` | A callable receiving the value — see Validation below. |
| `help:` | `nil` | Muted helper line rendered under the field. |
| `theme:` | form theme | Per-field theme override. |

### `input` — single-line text

Backed by [TextInput]({{ '/docs/components/text-input/' | relative_url }}); the cursor position is persisted per field so refocusing resumes mid-edit.

| Option | Default | What it does |
|--------|---------|--------------|
| `value:` | `""` | Initial text (only seeds the state on first bind). |
| `placeholder:` | `""` | Shown while the value is empty. |
| `width:` | `nil` | Constrains the rendered input width. |

### `textarea` — multi-line text

Backed by [TextArea]({{ '/docs/components/text-area/' | relative_url }}); cursor, scroll offset, and preferred column are persisted per field. Renders the label on its own line with the body indented beneath it.

| Option | Default | What it does |
|--------|---------|--------------|
| `value:` | `""` | Initial text. |
| `placeholder:` | `""` | Shown while the value is empty. |
| `width:` | `nil` | Constrains rendered line width. |
| `height:` | `nil` | Visible rows; the body scrolls. |

### `select` — single choice

Backed by a List; the chosen option (the object itself, not its label) becomes the value immediately as you move the highlight.

| Option | Default | What it does |
|--------|---------|--------------|
| `options:` | required | Array of selectable values. |
| `selected_index:` | `0` | Initial selection (clamped). |
| `option_label:` | `:to_s.to_proc` | Callable extracting the display string. |

```ruby
f.select :mood, options: Entry::MOODS,
  option_label: ->(m) { "#{Entry::MOOD_EMOJI.fetch(m)} #{m}" }
```

### `multiselect` — multiple choice

Backed by a MultiSelectList. Space toggles the highlighted option; the checked set, in option order, is the value.

| Option | Default | What it does |
|--------|---------|--------------|
| `options:` | required | Array of selectable values. |
| `selected_indices:` | `[]` | Pre-checked options. |
| `max_selections:` | `nil` | Caps how many can be checked (nil = unlimited). |
| `option_label:` | `:to_s.to_proc` | Callable extracting the display string. |

### `confirm` — boolean

Renders `[x] Label` / `[ ] Label`. With `required: true` the value must be `true` to pass validation (error: "must be accepted").

| Option | Default | What it does |
|--------|---------|--------------|
| `value:` | `false` | Initial state. |

### `note` — static text

`f.note "text"` renders a non-focusable line — headings, dividers, key hints. Notes take no part in Tab traversal, store no value, and never validate.

## Validation

Submitting (Ctrl+S, or Enter on the last field) validates every field. `required: true` adds "is required" for blank values (nil, whitespace-only strings, empty collections). `validate:` is any callable receiving the field value; its return value is interpreted as:

| Return | Meaning |
|--------|---------|
| `nil` or `true` | Valid. |
| `false` | Invalid — "is invalid". |
| `Array` | Invalid — each element is an error message. |
| anything else | Invalid — stringified into one message. |

```ruby
f.input :email, required: true,
  validate: ->(value) { value.include?("@") || "must contain an @" }
```

On failure the errors render under their fields, focus jumps to the first invalid field, and submission is blocked — the form returns `:handled`, not `[:submitted, ...]`.

## Keyboard

| Key | Action |
|-----|--------|
| `Tab` / `Shift+Tab` | Next / previous focusable field (including out of a textarea). |
| `Enter` | In a textarea: insert a newline. Elsewhere: next field, or submit from the last field. |
| `Ctrl+S` | Submit from any field. |
| `Escape` | Cancel. |
| `↑` / `↓` | Change the select choice / move the multiselect highlight. |
| `Space` | Toggle the highlighted multiselect option; toggle a confirm. |
| `y` / `→`, `n` / `←` | Set a confirm to yes / no. |

This matches charm.sh's huh: a textarea is a text editor, so it owns Enter — leave it with Tab and submit with Ctrl+S. (Shift+Enter, Ctrl+J, and Ctrl+N also insert newlines, for muscle memory and for hosts that construct `TextArea` with `enter_newline: false`.)

## Pasting

Bracketed paste routes to the focused field. Input fields insert the pasted text at the cursor (newlines stripped, single-line); textarea fields keep newlines and normalize CRLF to `\n`. Select, multiselect, confirm, and note fields ignore paste. The pasted value and cursor persist to form state like any other edit.

## What it returns

`handle_key` returns `[:submitted, values]` when validation passes (values is a `{field_name => value}` hash), `:cancelled` on Escape, and `:handled` while editing. On a focus slot these dispatch the actions declared with `on_submit` and `on_cancel` — the slot is the controller method name (`entry_form` above), not the form name. See the [component protocol]({{ '/docs/components/build-your-own/' | relative_url }}).

## Working with it

- `values` — the current `{field_name => value}` hash.
- `fields` / `state` — the field objects and the bound state hash (`:values`, `:fields`, `:errors`, `:focus_index`).
- `Form::Builder#build(state:, theme:)` — assembles fields into a Form when you're not going through the controller helper.

## Tips

- Field `value:` options only seed the state hash the first time the form is bound. To re-seed (say, switching from "new" to "edit"), clear the stale draft first: `reset_form(:entry)`. The journal example tracks a `@compose_mode` ivar in a `before_action` and resets the form whenever the mode changes.
- Focus-slot methods run on key-dispatch paths where `before_action` callbacks have **not** run — build the form from `session`/`params`, never from hook-populated instance variables.
- Select and multiselect values are the option objects themselves (symbols, records, whatever you passed in `options:`), with `option_label:` used only for display.
- Notes and other non-focusable fields are skipped by Tab and Enter traversal; Enter submits from the last *focusable* field even when a note renders after it.

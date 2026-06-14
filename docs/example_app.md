---
title: "Example App: Journal"
layout: default
nav_order: 17
parent: Docs
permalink: /docs/example-app/
---
# Example App: Journal

What the blog tutorial is to Rails, the Journal is to Charming: a complete,
runnable demo app that exercises every major framework feature. It lives in the
framework repository at
[`examples/journal`](https://github.com/pandorocks/charming/tree/main/examples/journal).

A daily-writing TUI: write entries in Markdown, tag them with a mood, browse and
reread them, track your writing streak, and export everything to a Markdown file.

## Run It

From a framework checkout:

```sh
cd examples/journal
bundle install
bundle exec ruby -I../../lib ../../exe/charming db:setup   # create + migrate + seed
bundle exec exe/journal
```

## Keys

| Key | Where | Does |
|-----|-------|------|
| `j` / `k`, arrows | lists & reading | move / scroll |
| `enter` | list | open entry |
| `n` | list | new entry |
| `e` | reading | edit entry |
| `f` | list & reading | toggle favorite (with toast) |
| `d` | list & reading | delete (confirm modal) |
| `x` | stats | export to `tmp/journal_export.md` |
| `enter` | compose body | new line (twice for a blank line) |
| `tab` | compose | next field |
| `ctrl+s` / `esc` | compose | save / cancel |
| `ctrl+p` | everywhere | command palette |
| `?` | everywhere | keyboard shortcut overlay |
| `q` | everywhere | quit |

## Feature Map

Use the journal as a reference implementation — each feature maps to a small,
readable file:

| Framework feature | Where to look |
|-------------------|---------------|
| Generated app + model + migrations | scaffolded with `charming new --database sqlite3`, `g model`, `g migration` |
| `before_action` + `rescue_from` | `app/controllers/reader_controller.rb` |
| Form (input/select/textarea/confirm) with edit-mode reseeding | `app/controllers/compose_controller.rb` |
| List + EmptyState + selection hooks | `app/controllers/entries_controller.rb` |
| Custom modal component capturing keys (delete confirm) | `app/components/delete_confirm.rb` |
| Markdown rendering in a scrollable viewport | `app/controllers/reader_controller.rb` |
| Async task with live progress + toast | `app/controllers/stats_controller.rb` |
| StatusBar, toasts, help overlay, overlay z-index | `app/views/layouts/application_layout.rb`, `app/controllers/application_controller.rb` |
| Themes (built-ins + `extends:`) and session persistence | `lib/journal/application.rb` |
| TestHelper specs + full-runtime journeys | `spec/journeys_spec.rb`, `spec/controllers/` |

## The Journeys Spec

`spec/journeys_spec.rb` is the pattern to copy for your own apps: it boots a real
`Charming::Runtime` against a `MemoryBackend` and types through complete flows —
composing an entry character by character, confirming a delete, opening the help
overlay, exporting with progress — then asserts on both the database and the
rendered frames. Real keystrokes, full stack, no TTY required.

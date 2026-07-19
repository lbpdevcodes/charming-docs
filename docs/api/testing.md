---
title: Testing & CLI
layout: default
parent: API Reference
nav_order: 8
permalink: /docs/api/testing/
---
# Testing & CLI

Charming apps are tested without a real terminal: a memory-backed runtime backend replays pre-seeded events, and `TestHelper` gives you one-liners for dispatching keys and asserting on responses. This page also lists the `charming` command-line interface. For testing patterns and worked examples, see [Testing]({{ '/docs/testing/' | relative_url }}).

## Runtime and testing backends

Apps normally use `TTYBackend` through `Charming.run`. Tests should use `Charming::Internal::Terminal::MemoryBackend` to avoid real terminal I/O. The runtime stops its loop when a test backend reports `exhausted?` (all pre-seeded events consumed), renders unhandled action exceptions as an `ErrorScreen` panel, and quits cleanly on SIGINT or an unbound `ctrl+c`.

## TestHelper

`require "charming/test_helper"` and `include Charming::TestHelper`:

- `build_controller(klass, app:, screen:, route:, event:)` — controller with test defaults.
- `key_event("ctrl+p")` — `KeyEvent` from a human-readable string.
- `press(klass, "q", app:)` — dispatch one key press, returns the `Response`.
- `press_sequence(klass, %w[down enter], app:)` — multiple presses sharing the app session.
- `memory_backend("up", "q", width: 80, height: 24)` — pre-seeded `MemoryBackend`.

RSpec matchers (registered when RSpec is loaded): `render_text("...")` and
`render_match(/.../)` compare against the ANSI-stripped body; `navigate_to("/path")`
asserts navigation; `be_quit` / `be_navigate` are response predicates.

## CLI

- `charming new NAME [--database sqlite3] [--force]` — scaffold an app.
- `charming generate TYPE NAME [args]` (`g`) — `screen`, `controller`, `view`, `component`, `model`, `migration`.
- `charming console` (`c`) — IRB with the app loaded and `app` available.
- `charming db:COMMAND` — see [Database]({{ '/docs/database/' | relative_url }}) for the full command table.

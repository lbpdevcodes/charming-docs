---
title: API Reference
layout: default
nav_order: 15
has_children: true
permalink: /docs/api/
---
# API Reference

This is the reference for Charming's current public API. Prefer these APIs in app code. Classes under `Charming::Internal` are runtime internals and are documented mainly for testing.

[API Stability]({{ '/docs/api/stability/' | relative_url }}) defines the 1.0 freeze contract: what semver covers, and what stays unversioned.

The reference is organized to follow the shape of an app — from the application object down to the pieces it renders:

- [Application & Configuration]({{ '/docs/api/application/' | relative_url }}) — the `Charming::Application` class, environment, and the `Charming.run` entrypoint.
- [Routing]({{ '/docs/api/routing/' | relative_url }}) — the routes DSL, resolution rules, and route objects.
- [Controllers & Responses]({{ '/docs/api/controllers/' | relative_url }}) — key bindings, actions, hooks, state, and the response objects actions return.
- [Views & Templates]({{ '/docs/api/views/' | relative_url }}) — class-based views, view helpers, and the ERB template fallback.
- [Components API]({{ '/docs/api/components/' | relative_url }}) — the component base class, event conventions, and the bundled component catalog.
- [UI & Themes]({{ '/docs/api/ui/' | relative_url }}) — the `Style` builder, layout helpers, color capability, and theme tokens.
- [Events, Tasks & Media]({{ '/docs/api/events/' | relative_url }}) — event classes, async task executors, animation physics, and audio playback.
- [Testing & CLI]({{ '/docs/api/testing/' | relative_url }}) — test backends, `TestHelper`, RSpec matchers, and `charming` commands.

## Where do I look for…?

| I want to… | Go to |
|---|---|
| Configure my app, register themes, persist the session | [Application & Configuration]({{ '/docs/api/application/' | relative_url }}) |
| Map a path like `/users/:id` to a controller | [Routing]({{ '/docs/api/routing/' | relative_url }}) |
| Handle a key press, run a background task, navigate, or quit | [Controllers & Responses]({{ '/docs/api/controllers/' | relative_url }}) |
| Keep data across events with typed attributes | [Controllers & Responses]({{ '/docs/api/controllers/' | relative_url }}) (`ApplicationState`) |
| Render output with `text`, `box`, `row`, or ERB templates | [Views & Templates]({{ '/docs/api/views/' | relative_url }}) |
| Use a bundled widget (list, table, form, modal, spinner…) | [Components API]({{ '/docs/api/components/' | relative_url }}) |
| Style text, draw borders, join blocks, use theme colors | [UI & Themes]({{ '/docs/api/ui/' | relative_url }}) |
| See what fields an event carries, animate, or play sound | [Events, Tasks & Media]({{ '/docs/api/events/' | relative_url }}) |
| Test a controller or use the CLI generators | [Testing & CLI]({{ '/docs/api/testing/' | relative_url }}) |

For tutorial-style explanations, see [Getting Started]({{ '/docs/getting-started/' | relative_url }}). For topic guides, see the [docs index]({{ '/' | relative_url }}).

---
title: API Stability
layout: default
parent: API Reference
permalink: /docs/api/stability/
---
# API Stability

Charming is pre-1.0, so breaking changes still happen — each one ships with a
CHANGELOG entry and an UPGRADING.md migration. This page is the contract for
what 1.0 will freeze.

## The rule

- Everything under `Charming::Internal` is unversioned. It can change in any
  release without notice. Do not build on it.
- Everything else follows semantic versioning from 1.0. Breaking changes after
  1.0 need a deprecation release first.
- Constants outside `Internal` whose docstring says "Internal" (dispatch
  collaborators, layout node internals) are public for structural reasons
  only. They follow the Internal rule.

## The public surface

From 1.0, semver covers:

- **Controller DSL** — `key`, `timer`, `animate`, `on_task`,
  `on_task_progress`, `on_submit`/`on_select`/`on_cancel`, `slot`,
  `focus_ring`, `layout`, `auto_render`, the action hooks, and `rescue_from`.
- **Controller actions** — `render`, `render_view`, `render_template`,
  `navigate`, `quit`, `session`, `state`, `form`/`reset_form`, `run_task`,
  `cancel_task`, timer controls, and the lifecycle hooks.
- **Slot, focus, and dispatch contracts** — `focus`, `focused?`,
  `component_for`, and the `Charming::Components::Result` return protocol for
  component `handle_key`/`handle_mouse`/`handle_paste`. Legacy return forms
  (`:handled`, `:cancelled`, `[:submitted, value]`, `[:selected, value]`)
  normalize to `Result`; components may return either.
- **View helpers** — assigns readers, `text`, `box`, `row`, `column`,
  `render_component`, `screen_layout`, `yield_content`.
- **Routing** — the `root`/`screen` DSL and named `navigate`.
- **Value objects** — `Charming::Response`, `Charming::Screen`, and the
  `Charming::Events::*` event classes.
- **Testing** — `Charming::TestHelper`.
- **Shell** — `Charming::Shell::Sidebar` and `Charming::Shell::Palette`
  (including the `command` DSL).
- **Tasks** — the executor contract, `Tasks::Context`, `Tasks::Cancelled`.
- **Components** — the built-in `Charming::Components::*` widgets and their
  documented constructor options.
- **Errors** — `Charming::Error`, `UnhandledComponentEvent`, `UnknownSlot`,
  `DoubleRenderError`, `CrossThreadAccess`.

## Drift control

`rake api:public` lists every public constant, and a spec locks the list. A PR
that adds or removes a public constant fails the suite until it updates the
list, so API drift is visible in review.

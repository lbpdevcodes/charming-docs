---
title: Testing
layout: default
nav_order: 14
parent: Docs
permalink: /docs/testing/
---
# Testing

Charming is designed to be tested without a real terminal. Use controller, template, view, and component specs for small units, and use `MemoryBackend` for runtime-level behavior.

For app structure and rendering concepts, see [Core Concepts]({{ '/docs/core-concepts/' | relative_url }}), [Controllers & Views]({{ '/docs/controllers-and-views/' | relative_url }}), and [Layouts]({{ '/docs/layouts/' | relative_url }}).

## Generated Specs

Generated apps include specs for the default state object, controller, template, and component. Run them from the generated app with:

```sh
bundle exec rspec
```

Framework development uses:

```sh
bin/rspec
bin/lint
bin/check
```

Apps generated with `--database sqlite3` get a spec_helper that pins
`CHARMING_ENV=test` before the app loads, prepares the test database from
`db/schema.rb`, and rolls back each example in a transaction — your specs run
against `db/test.sqlite3` in full isolation.

## Charming::TestHelper

`charming/test_helper` is the fastest way to write controller and journey specs.
It ships with the framework and registers RSpec matchers when RSpec is loaded:

```ruby
require "charming/test_helper"

RSpec.describe MyApp::EntriesController do
  include Charming::TestHelper

  let(:app) { MyApp::Application.new }

  it "renders the list" do
    response = build_controller(described_class, app: app).dispatch(:show)
    expect(response).to render_text("Entries")
  end

  it "navigates to compose on n" do
    build_controller(described_class, app: app).dispatch(:show)
    expect(press(described_class, "n", app: app)).to navigate_to("/compose")
  end

  it "deletes through the confirm modal" do
    build_controller(described_class, app: app).dispatch(:show)
    press_sequence(described_class, %w[d y], app: app)
    expect(Entry.count).to eq(0)
  end
end
```

Helpers:

| Helper | Purpose |
|--------|---------|
| `build_controller(klass, app:, screen:, route:, event:)` | Controller wired to an app (defaults: fresh `Application`, 80×24 screen). |
| `key_event("ctrl+p")` | Build a `KeyEvent` from a human-readable string (`"q"`, `"down"`, `"shift+tab"`). |
| `press(klass, "q", app:)` | Dispatch one key press; returns the `Response`. |
| `press_sequence(klass, %w[down down enter], app:)` | Dispatch several presses through fresh controllers sharing the app session (mirrors the runtime). |
| `memory_backend("up", "q", width: 80, height: 24)` | A `MemoryBackend` pre-seeded with parsed key events, ready for `Charming::Runtime`. |

Matchers:

| Matcher | Asserts |
|---------|---------|
| `render_text("Hello")` | The response body includes the text — compared **ANSI-stripped**, so styled output that interleaves escape codes mid-phrase still matches. |
| `render_match(/Count: \d+/)` | Regex variant, also ANSI-stripped. |
| `navigate_to("/path")` | The response is a navigation to that path. |
| `be_quit` / `be_navigate` | Predicate matchers on `Response`. |

## Journey Specs

Drive the whole app through a real `Runtime` — actual keystrokes, no TTY:

```ruby
it "creates an entry end-to-end" do
  keys = ["n", *"Demo day".chars, "enter", "down", "enter", *"It worked.".chars, "ctrl+s", "q"]
  backend = memory_backend(*keys, width: 100, height: 30)

  Charming::Runtime.new(MyApp::Application.new, backend: backend,
    task_executor: Charming::Tasks::InlineExecutor).run

  expect(Entry.find_by(title: "Demo day").body).to eq("It worked.")
  expect(Charming::UI::Width.strip_ansi(backend.frames.last)).to include("Demo day")
end
```

When a `MemoryBackend` runs out of events the runtime stops the loop
(`MemoryBackend#exhausted?`), so a forgotten trailing quit event ends the test
instead of hanging it.

## Controller Specs

Instantiate controllers with an application and dispatch actions directly:

```ruby
RSpec.describe MyApp::HomeController do
  let(:application) { MyApp::Application.new }

  it "renders the home screen" do
    response = described_class.new(application: application).dispatch(:show)

    expect(response.body).to include("Home")
  end
end
```

Pass events when testing key, timer, task, or mouse dispatch:

```ruby
event = Charming::Events::KeyEvent.new(key: :up)
response = described_class.new(application: application, event: event).dispatch_key
```

Route params can be passed directly for controller-level tests:

```ruby
controller = described_class.new(application: application, params: {id: "123"})
expect(controller.dispatch(:show).body).to include("123")
```

## Template Specs

Resolve and render templates directly when testing template output:

```ruby
template = Charming::Templates.resolve("home/show", root: app_root)
view = Charming::TemplateView.new(
  template: template,
  home: double(title: "Home"),
  theme: Charming::UI::Theme.default
)

expect(view.render).to include("Home")
```

Templates can use normal view helpers. Strip ANSI codes when assertions do not need styling:

```ruby
body = Charming::UI::Width.strip_ansi(view.render)
expect(body).to include("Home")
```

Controller tests can cover template rendering through `render :show`:

```ruby
response = MyApp::HomeController.new(application: application).dispatch(:show)

expect(response.body).to include("Home")
```

## Class-Based View Specs

Class-based views are plain objects. Pass assigns to `new` and call `render`:

```ruby
view = MyApp::HomeView.new(
  home: double(title: "Home"),
  theme: Charming::UI::Theme.default
)

expect(view.render).to include("Home")
```

## Component Specs

Render components directly:

```ruby
component = Charming::Components::List.new(items: %w[One Two])

expect(component.render).to include("One")
```

For interactive components, assert return values and state changes:

```ruby
input = Charming::Components::TextInput.new

expect(input.handle_key(Charming::Events::KeyEvent.new(key: :a, char: "a"))).to eq(:handled)
expect(input.value).to eq("a")
```

## Runtime Specs

Use `MemoryBackend` to script terminal events and capture rendered frames:

```ruby
backend = Charming::Internal::Terminal::MemoryBackend.new(
  events: [
    Charming::Events::KeyEvent.new(key: :up),
    Charming::Events::KeyEvent.new(key: :q)
  ],
  width: 80,
  height: 24
)

Charming::Runtime.new(MyApp::Application.new, backend: backend).run

expect(backend.frames).to eq(["Count: 0", "Count: 1"])
```

`MemoryBackend` accepts `events:`, `width:`, and `height:` keyword args. After running, inspect `backend.frames` to assert against rendered terminal frames.

## Timer Specs

Inject a deterministic clock into the runtime:

```ruby
times = [0.0, 0.0, 0.0, 0.1, 0.2]
clock = -> { times.shift || 0.2 }

runtime = Charming::Runtime.new(app, backend: backend, clock: clock)
runtime.run
```

This avoids sleeps and makes timer behavior deterministic.

## Task Specs

Use the inline task executor for deterministic async task tests — it runs blocks
synchronously and queues results (and progress events) immediately:

```ruby
runtime = Charming::Runtime.new(
  app,
  backend: backend,
  task_executor: Charming::Tasks::InlineExecutor
)
runtime.run
```

Controller-level task tests can stub the app task executor. The contract is
`submit(name, timeout: nil, &block)` — the keyword is only passed when the caller
sets a timeout, so the simple signature works until you test timeouts:

```ruby
executor = Class.new do
  attr_reader :name

  def submit(name, timeout: nil, &)
    @name = name
  end
end.new

application.task_executor = executor
controller.dispatch(:refresh)

expect(executor.name).to eq(:refresh_home)
```

Progress assertions: drain the queue and inspect `TaskProgressEvent`s
(`current`, `total`, `message`, `fraction`) ahead of the final `TaskEvent`.

## Renderer Specs

For renderer-level tests, pass a custom `renderer:` to `Charming::Runtime.new` or test renderer classes directly with `MemoryBackend`.

Backend and renderer classes under `Charming::Internal` are mostly test-facing implementation details. Prefer `MemoryBackend` for app and framework specs unless you specifically need TTY integration behavior.

## Snapshot-Style Assertions

For rendered terminal output, prefer small, stable assertions first. Use full-frame comparisons when the output is intentionally fixed.

Good:

```ruby
body = Charming::UI::Width.strip_ansi(response.body)
expect(body).to include("Status: Loaded")
```

Use exact frame assertions for runtime flow:

```ruby
expect(backend.frames).to eq(["Home", "Settings"])
```

Avoid tests that only assert a response exists unless the behavior under test is dispatch plumbing.

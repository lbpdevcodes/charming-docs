---
title: Build a TODO App
layout: default
parent: Tutorials
nav_order: 1
permalink: /docs/tutorials/todo/
---
# Build a TODO App

In this tutorial you'll build a complete todo app that runs in your terminal and
saves your tasks to SQLite — from `charming new` to a finished, tested app. You'll
add tasks by typing, toggle them done, delete them, and watch everything persist
across runs.

Along the way you'll touch nearly every tool Charming ships: the app generator,
the model generator, migrations, the `db:*` commands, the console, built-in
components (a list and a text input), key bindings, the focus ring, a status bar,
and a full-stack journey spec.

When you're done it looks like this:

```text
╭────────────────────╮ ╭───────────────────────────────────────────────────────╮
│                    │ │    New task                                             │
│        Todo        │ │  |Add a task and press enter…                           │
│                    │ │  ▸ Tasks  1 of 3 done                                   │
│    ● Home          │ │  > [x] Read the Charming tutorial                       │
│                    │ │    [ ] Build a todo app                                 │
│  tab focus         │ │    [ ] Add a task of your own                          │
│  ctrl+p commands   │ │                                                         │
│  q quit            │ │                                                         │
╰────────────────────╯ ╰───────────────────────────────────────────────────────╯
 Home   enter add  space toggle  d delete  tab focus  ctrl+p commands  q quit  3 tasks
```

The `▸` marks which part has focus — here the task list. `Tab` moves it between the
add-task input and the list.

This is a single-screen app, which keeps the focus on the essentials. The
[Example App: Journal]({{ '/docs/example-app/' | relative_url }}) shows where to go
next — multiple screens, forms, modals, and async work.

## 1. Install and scaffold

Install the CLI gem, then generate a database-backed app:

```sh
gem install charming
charming new todo --database sqlite3
```

The `--database sqlite3` flag wires in Active Record and SQLite from the start.
You'll see Charming write a full gem-style skeleton and initialize git:

```text
create Gemfile
create Rakefile
create .rspec
create README.md
create todo.gemspec
create exe/todo
create lib/todo.rb
create lib/todo/application.rb
create lib/todo/version.rb
create config/routes.rb
create app/state/application_state.rb
create app/state/home_state.rb
create app/controllers/application_controller.rb
create app/controllers/home_controller.rb
create app/views/layouts/application_layout.rb
create app/views/home/show_view.rb
create app/components/.keep
create spec/spec_helper.rb
create spec/state/home_state_spec.rb
create spec/controllers/home_controller_spec.rb
create spec/views/home/show_view_spec.rb
create config/database.rb
create app/models/application_record.rb
create db/migrate/.keep
create db/seeds.rb
init git
```

The database flag added the last four files: `config/database.rb`,
`app/models/application_record.rb`, `db/migrate/`, and `db/seeds.rb`. Install
dependencies and you're ready:

```sh
cd todo
bundle install
```

The generated `Gemfile` pulls in `charming` plus `rake`, `rspec`, and `irb` (so
`bundle exec rspec` and `charming console` work out of the box), and the gemspec
declares `activerecord` and `sqlite3` as app dependencies:

```ruby
# Gemfile
source "https://rubygems.org"

gemspec

group :development, :test do
  gem "irb"     # `charming console` (not a default gem on Ruby 4.0+)
  gem "rake", "~> 13.0"
  gem "rspec", "~> 3.13"
end
```

## 2. Tour the generated app

Charming apps follow Rails-like conventions. The pieces you'll edit live under
`app/`:

```text
app/controllers/home_controller.rb   # the screen's logic
app/state/home_state.rb              # durable in-memory state
app/views/home/show_view.rb          # what the screen renders
app/views/layouts/application_layout.rb
app/models/application_record.rb     # base class for models
config/routes.rb                     # url -> controller#action
config/database.rb                   # picks the db file from CHARMING_ENV
```

The root route already points at the home screen:

```ruby
# config/routes.rb
Todo::Application.routes do
  root "home#show"
end
```

Run it once to see the starter screen, then press `q` to quit:

```sh
bundle exec exe/todo
```

The flow for every screen is the same: **Route → Controller action → View →
Layout → Renderer → Terminal**. We'll fill in each piece for tasks. For the full
picture see [Core Concepts]({{ '/docs/core-concepts/' | relative_url }}).

## 3. The Task model

Generate a model to store tasks. We'll call the record a **Task** (the app module
is already `Todo`, so a model named `todo` would become the awkward `Todo::Todo`):

```sh
charming generate model task title:string done:boolean
```

```text
create app/models/task.rb
create db/migrate/20260614124436_create_tasks.rb
create spec/models/task_spec.rb
```

The generator pluralized the table to `tasks` and wrote a migration with your
columns plus timestamps:

```ruby
# db/migrate/20260614124436_create_tasks.rb
class CreateTasks < ActiveRecord::Migration[8.1]
  def change
    create_table :tasks do |t|
      t.string :title
      t.boolean :done
      t.timestamps
    end
  end
end
```

Create the database and run the migration. Charming refreshes `db/schema.rb`
afterward:

```sh
charming db:create
charming db:migrate
```

```text
create db/development.sqlite3
== 20260614124436 CreateTasks: migrating ======================================
-- create_table(:tasks)
== 20260614124436 CreateTasks: migrated =======================================
migrate db/migrate
```

Now flesh out the model. Add a validation, an ordering scope, and two small
helpers — a `toggle_done!` and a `list_label` that renders the checkbox we'll show
in the list:

```ruby
# app/models/task.rb
module Todo
  class Task < ApplicationRecord
    validates :title, presence: true

    scope :ordered, -> { order(created_at: :asc) }

    # Flips done and persists in one call.
    def toggle_done!
      update!(done: !done?)
    end

    # The row text shown in the list: a checkbox plus the title.
    def list_label
      "[#{done? ? "x" : " "}] #{title}"
    end
  end
end
```

Try it in the console, which boots your app with the database connected:

```sh
charming console
```

```text
Loading development environment (Charming 0.2.1)
irb> Todo::Task.create!(title: "Try Charming", done: false)
irb> Todo::Task.ordered.map(&:list_label)
=> ["[ ] Try Charming"]
irb> Todo::Task.count
=> 1
```

## 4. Listing tasks (the read path)

Controllers in Charming are **ephemeral** — a fresh instance per keypress — so
anything that must survive between keystrokes lives in a state object. Our screen
needs to remember two things: the text being typed into the add box, and which row
is highlighted. Replace the generated `HomeState`:

```ruby
# app/state/home_state.rb
module Todo
  class HomeState < ApplicationState
    # The in-progress text in the add-task input. Controllers are recreated every
    # dispatch, so the draft is parked here (session-backed) and re-seeded into the
    # TextInput on each render.
    attribute :draft, :string, default: ""

    # The highlighted row in the task list, preserved across dispatches.
    attribute :selected_index, :integer, default: 0
  end
end
```

Now the controller. Start with just the read path — build a
`Charming::Components::List` from the saved tasks and hand it to the view. The
`label:` tells the list how to render each row (our `list_label`):

```ruby
# app/controllers/home_controller.rb
module Todo
  class HomeController < ApplicationController
    def show
      home.selected_index = tasks.selected_index
      render :show, tasks: tasks, palette: command_palette
    end

    # The selectable task list, restored from session state each dispatch.
    def tasks
      @tasks ||= Charming::Components::List.new(
        items: Task.ordered.to_a,
        selected_index: home.selected_index,
        height: [screen.height - 12, 3].max,
        label: :list_label.to_proc,
        theme: theme
      )
    end

    private

    def home
      state(:home, HomeState)
    end
  end
end
```

And the view. Render the list, but show a friendly placeholder when there are no
tasks yet — Charming ships an `EmptyState` component for exactly this:

```ruby
# app/views/home/show_view.rb
module Todo
  module Home
    class ShowView < Charming::View
      def render
        column(heading, body, gap: 1)
      end

      private

      def heading
        done = tasks.items.count(&:done?)
        row(
          text("Tasks", style: theme.title),
          text("#{done} of #{tasks.items.length} done", style: theme.muted),
          gap: 2
        )
      end

      def body
        return empty_state if tasks.items.empty?

        render_component(tasks)
      end

      def empty_state
        render_component(
          Charming::Components::EmptyState.new(
            message: "No tasks yet — type above and press enter.",
            theme: theme
          )
        )
      end
    end
  end
end
```

Run the app and you'll see the task you created in the console. It's read-only for
now — we wire up focus and keys in the next two chapters.

## 5. Adding tasks (the write path)

To add tasks we need a text input the user can type into, and a way to submit it.
Charming's `TextInput` captures keystrokes; pressing **Enter** returns
`[:submitted, value]`, which the runtime dispatches to a `<slot>_submitted` hook on
the controller — the same pattern the `List` uses for selection.

Two new ideas make this work:

- **The focus ring.** `focus_ring :new_task, :tasks` lists the things `Tab` cycles
  through — just the two interactive parts of this screen, the input and the list.
  The first slot (`:new_task`) is focused on boot, so you can type immediately. We
  deliberately leave the sidebar *out* of the ring: on a single-screen app it's only
  showing "Home", so cycling through it would just be a confusing extra `Tab` stop.

  {: .important }
  > Each slot name **must match a controller method that returns the component** — the
  > runtime calls `new_task` for the `:new_task` slot and `tasks` for the `:tasks`
  > slot to route key events. This is why our list method is named `tasks` and not,
  > say, `tasks_list`: a slot with no matching method becomes a silent dead stop that
  > `Tab` lands on but no keys reach.
- **Re-seeding the input.** Because the controller is recreated every keystroke, we
  park the in-progress text in `home.draft` and rebuild the `TextInput` from it each
  render. On submit we clear the draft.

Update the controller to add the input, the focus ring, and the create hook:

```ruby
# app/controllers/home_controller.rb
module Todo
  class HomeController < ApplicationController
    focus_ring :new_task, :tasks

    def show
      home.draft = new_task.value
      home.selected_index = tasks.selected_index
      render :show, new_task: new_task, tasks: tasks, palette: command_palette
    end

    # Enter in the input creates a task. TextInput returns [:submitted, value],
    # which the runtime dispatches here as `<slot>_submitted(value)`.
    def new_task_submitted(value)
      title = value.to_s.strip
      return show if title.empty?

      Task.create!(title: title, done: false)
      clear_new_task
      show
    end

    # The add-task input, re-seeded from the session-backed draft each dispatch.
    def new_task
      @new_task ||= Charming::Components::TextInput.new(
        value: home.draft,
        placeholder: "Add a task and press enter…"
      )
    end

    def tasks
      @tasks ||= Charming::Components::List.new(
        items: Task.ordered.to_a,
        selected_index: home.selected_index,
        height: [screen.height - 12, 3].max,
        label: :list_label.to_proc,
        theme: theme
      )
    end

    private

    def home
      state(:home, HomeState)
    end

    # Empties the draft and drops the memoized input so `show` rebuilds it blank.
    def clear_new_task
      home.draft = ""
      @new_task = nil
    end
  end
end
```

Now update the view. We add the input at the top and give each section a heading
that brightens — with a `▸` marker — when it's focused, using the view's `focused?`
helper. Without that cue the input and list look the same whether or not they're
focused, and `Tab` feels like it does nothing:

```ruby
# app/views/home/show_view.rb
module Todo
  module Home
    class ShowView < Charming::View
      def render
        column(input_section, list_section, gap: 1)
      end

      private

      def input_section
        column(section_label("New task", :new_task), render_component(new_task))
      end

      def list_section
        done = tasks.items.count(&:done?)
        header = row(
          section_label("Tasks", :tasks),
          text("#{done} of #{tasks.items.length} done", style: theme.muted),
          gap: 2
        )
        column(header, body)
      end

      # A section heading that brightens (with a ▸ marker) when its slot is focused,
      # so it's obvious whether Tab has you typing or navigating the list.
      def section_label(name, slot)
        active = focused?(slot)
        text("#{active ? "▸" : " "} #{name}", style: active ? theme.title : theme.muted)
      end

      def body
        return empty_state if tasks.items.empty?

        render_component(tasks)
      end

      def empty_state
        render_component(
          Charming::Components::EmptyState.new(
            message: "No tasks yet — type above and press enter.",
            theme: theme
          )
        )
      end
    end
  end
end
```

`focused?(:new_task)` and `focused?(:tasks)` ask the controller which ring slot is
active — views get the controller passed in automatically, so this just works.

Run the app, type a task, and press Enter. It appears in the list and is saved to
the database. Press `Tab` and watch the `▸` marker jump between **New task** and
**Tasks** — that's focus moving between the input and the list.

{: .note }
> **A fix we made while writing this tutorial.** Building this app surfaced a gap in
> the framework: `TextInput` swallowed the Enter key, so a focused input could never
> trigger a `_submitted` hook the way `List`, `MultiSelectList`, and `Autocomplete`
> already did. We taught `TextInput` to return `[:submitted, value]` on Enter — which
> is exactly what makes the code above work. Dogfooding the generators is the fastest
> way to find rough edges.

## 6. Toggling and deleting

With the list focused, we want `space` to toggle the highlighted task done, `d` to
delete it, and `Enter` to also toggle (the list reports Enter as a selection). Key
bindings declared with `key` fire when a non-text component has focus; while the
input is focused those same keys are just typed characters.

Add the bindings and three handlers:

```ruby
# app/controllers/home_controller.rb — add near the top of the class
key "space", :toggle_selected
key "d", :delete_selected

# Enter on a list row toggles the highlighted task.
def tasks_selected(task)
  task.toggle_done!
  show
end

def toggle_selected
  task = tasks.selected_item
  return show unless task

  task.toggle_done!
  show
end

def delete_selected
  task = tasks.selected_item
  return show unless task

  task.destroy!
  home.selected_index = [home.selected_index - 1, 0].max
  @tasks = nil # rebuild from the database so the deleted row is gone
  show
end
```

{: .note }
> **A subtle gotcha worth understanding.** `tasks` is memoized for the duration
> of one dispatch so the view can reference it cheaply. In `delete_selected` we call
> `tasks.selected_item` *before* destroying the record — which builds and caches
> the list with that record still in its `items` array. After `destroy!`, the cached
> list is stale, so we clear `@tasks` to force `show` to query a fresh list.
> Toggling doesn't need this: it mutates the same object the list already holds.

Now you have the full loop: add, toggle, delete — all persisted.

## 7. Polish: a status bar

The journal example shows a status bar along the bottom with key hints and a live
count. Add one here. First expose the hints and count on the controllers. The base
`ApplicationController` gets sensible defaults; the home screen prepends its own:

```ruby
# app/controllers/application_controller.rb — add inside the class
# Key hints shown in the status bar; screens prepend their own via `*super`.
def status_hints
  [["ctrl+p", "commands"], ["q", "quit"]]
end

# Live count rendered on the right of the status bar.
def task_count
  Task.count
end
```

```ruby
# app/controllers/home_controller.rb — add to the class
def status_hints
  [["enter", "add"], ["space", "toggle"], ["d", "delete"], ["tab", "focus"], *super]
end
```

Then render a `StatusBar` from the layout. Wrap the existing sidebar/content split
in a vertical split and add a one-row pane beneath it:

```ruby
# app/views/layouts/application_layout.rb — update #render
def render
  screen_layout(background: theme.background) do
    split :vertical do
      split(narrow? ? :vertical : :horizontal, gap: 1, grow: 1) do
        pane(:sidebar, **sidebar_options, border: :rounded, padding: [1, 2], style: sidebar_style) do
          column(app_title, navigation, shortcuts, gap: 1)
        end

        pane(:content, grow: 1, border: :rounded, padding: [1, 2], style: content_style) do
          yield_content
        end
      end

      pane(:status_bar, height: 1) do
        render_component status_bar
      end
    end

    overlay command_palette_modal if command_palette_modal
  end
end

# add this private helper
def status_bar
  Charming::Components::StatusBar.new(
    width: screen.width,
    left: " #{controller.route&.title || "Todo"}",
    hints: controller.status_hints,
    right: "#{controller.task_count} tasks ",
    theme: theme
  )
end
```

Your app now matches the screenshot at the top. The built-in themes are available
from the command palette (`ctrl+p` → "Theme"), and `:phosphor` is the default — see
[Themes]({{ '/docs/themes/' | relative_url }}).

### Bonus: the migration generator

Say you want `done` to default to `false` at the database level. That's a
migration. Charming reads Rails-style names to decide the body:

```sh
charming generate migration set_done_default_on_tasks
```

The generator writes a timestamped migration with an empty `change` for you to
fill in:

```ruby
class SetDoneDefaultOnTasks < ActiveRecord::Migration[8.1]
  def change
    change_column_default :tasks, :done, from: nil, to: false
  end
end
```

```sh
charming db:migrate      # apply it
charming db:status       # see up/down per version
charming db:rollback     # undo the last migration
```

See [Database]({{ '/docs/database/' | relative_url }}) for the full command suite.

## 8. Tests

Database apps come wired for RSpec: the generated `spec/spec_helper.rb` pins
`CHARMING_ENV=test`, loads the schema into an isolated `db/test.sqlite3`, and rolls
back each example in a transaction. Run the suite any time with:

```sh
bundle exec rspec
```

Replace the placeholder model spec with real assertions:

```ruby
# spec/models/task_spec.rb
require "todo"

RSpec.describe Todo::Task do
  it "requires a title" do
    task = described_class.new(title: nil)

    expect(task).not_to be_valid
    expect(task.errors[:title]).to include("can't be blank")
  end

  it "orders tasks by creation time" do
    first = described_class.create!(title: "First", created_at: Time.now - 60)
    second = described_class.create!(title: "Second", created_at: Time.now)

    expect(described_class.ordered.to_a).to eq([first, second])
  end

  it "toggles done and persists" do
    task = described_class.create!(title: "Toggle me", done: false)

    task.toggle_done!

    expect(task.reload.done?).to be(true)
  end

  it "renders a checkbox label" do
    expect(described_class.new(title: "Walk dog", done: false).list_label).to eq("[ ] Walk dog")
    expect(described_class.new(title: "Walk dog", done: true).list_label).to eq("[x] Walk dog")
  end
end
```

The real prize is a **journey spec** — it boots a real `Charming::Runtime` against
an in-memory backend and types through the whole app, asserting on both the
database and the rendered frames. No terminal required. This is the pattern to copy
for any Charming app:

```ruby
# spec/journeys_spec.rb
require_relative "spec_helper"
require "charming/test_helper"

RSpec.describe "Todo journeys" do
  include Charming::TestHelper

  before do
    Todo::Task.create!(title: "Existing task", done: false)
  end

  def run_journey(*keys, width: 100, height: 30)
    backend = memory_backend(*keys, width: width, height: height)
    Charming::Runtime.new(Todo::Application.new, backend: backend,
      task_executor: Charming::Tasks::InlineExecutor).run
    backend
  end

  def plain(frame)
    Charming::UI::Width.strip_ansi(frame.to_s)
  end

  it "boots into the task list with the status bar" do
    backend = run_journey("tab", "q")
    frame = plain(backend.frames.first)

    expect(frame).to include("Tasks")
    expect(frame).to include("[ ] Existing task")
    expect(frame).to include("enter add")
    expect(frame).to include("1 tasks")
  end

  it "adds a task by typing and pressing enter" do
    backend = run_journey(*"Buy milk".chars, "enter", "tab", "q")

    expect(Todo::Task.find_by(title: "Buy milk")).not_to be_nil
    expect(plain(backend.frames.last)).to include("[ ] Buy milk")
  end

  it "toggles a task done from the list" do
    backend = run_journey("tab", "space", "q")

    expect(Todo::Task.find_by(title: "Existing task").done?).to be(true)
    expect(plain(backend.frames.last)).to include("[x] Existing task")
  end

  it "deletes a task and shows the empty state" do
    backend = run_journey("tab", "d", "q")

    expect(Todo::Task.count).to eq(0)
    expect(plain(backend.frames.last)).to include("No tasks yet")
  end

  it "types q into the input instead of quitting" do
    run_journey(*"q task".chars, "enter", "tab", "q")

    expect(Todo::Task.find_by(title: "q task")).not_to be_nil
  end
end
```

That last test is a nice illustration of focus: while the input is focused, `q` is
typed into the field rather than quitting the app — text-capturing components get
printable keys before global bindings. Run it all:

```sh
bundle exec rspec
```

```text
17 examples, 0 failures
```

## 9. Run it

Seed a few tasks so a fresh setup has something to show, then launch:

```ruby
# db/seeds.rb
[
  {title: "Read the Charming tutorial", done: true},
  {title: "Build a todo app", done: false},
  {title: "Add a task of your own", done: false}
].each do |attrs|
  Todo::Task.find_or_create_by!(title: attrs[:title]) { |task| task.done = attrs[:done] }
end
```

```sh
charming db:seed
bundle exec exe/todo
```

| Key | Where | Does |
|-----|-------|------|
| type + `enter` | input | add a task |
| `j` / `k`, arrows | list | move the highlight |
| `space` / `enter` | list | toggle done |
| `d` | list | delete |
| `tab` | anywhere | move focus between the input and the list |
| `ctrl+p` | anywhere | command palette |
| `q` | list | quit (while the input is focused, `q` is typed) |

## Where to go next

You've built a real, tested, persistent TUI with Charming's generators and tools.
To grow it:

- **A second screen.** `charming generate screen done` scaffolds a state /
  controller / view trio *and* wires up a route and a command-palette entry — the
  way to add a "completed tasks" view.
- **A custom component.** `charming generate component delete_confirm` to extract a
  reusable widget — for example a confirm-before-delete modal (the journal app has
  one).
- **Toasts, help overlays, async tasks.** All demonstrated in the
  [Example App: Journal]({{ '/docs/example-app/' | relative_url }}).

Reference docs for everything you used here:
[CLI & Generators]({{ '/docs/cli/' | relative_url }}),
[Database]({{ '/docs/database/' | relative_url }}),
[Controllers & Views]({{ '/docs/controllers-and-views/' | relative_url }}),
[Components]({{ '/docs/components/' | relative_url }}),
[State]({{ '/docs/state/' | relative_url }}),
[Testing]({{ '/docs/testing/' | relative_url }}).

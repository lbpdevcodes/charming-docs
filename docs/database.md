---
title: Database
layout: default
nav_order: 8
parent: Docs
permalink: /docs/database/
---
# Database

Charming can generate SQLite-backed apps with Active Record. Database support is opt-in; apps without it continue to use only session-backed state in `app/state`. The `activerecord` and `sqlite3` gems are dependencies of *your app* (added by the generator), not of the framework.

## New App

Generate an app with SQLite support:

```bash
charming new todos_tui --database sqlite3
```

This adds:

```text
app/models/application_record.rb
config/database.rb
db/migrate/.keep
db/seeds.rb
```

## Environments

`config/database.rb` selects the database file from `CHARMING_ENV` (default
`development`):

```ruby
environment = ENV["CHARMING_ENV"] || "development"
database_path = File.expand_path("../db/#{environment}.sqlite3", __dir__)
```

So `db/development.sqlite3`, `db/test.sqlite3`, and `db/production.sqlite3` stay
separate. In framework and app code, `Charming.env` returns a string inquirer:

```ruby
Charming.env              # => "development"
Charming.env.test?        # => false
Charming.env.production?  # => false
```

Generated specs pin `CHARMING_ENV=test` before the app loads (see
[Testing]({{ '/docs/testing/' | relative_url }})), so the test suite never touches
your development data.

## Existing App

Install SQLite support into an existing Charming app from the app root:

```bash
charming db:install sqlite3
```

This creates the same database files, updates the generated gemspec with Active Record dependencies, and updates `lib/my_app.rb` to load `config/database.rb` and `app/models`.

## State vs Models

Use state classes for in-memory TUI state:

```ruby
def home
  state(:home, HomeState)
end
```

Use Active Record models for persisted data:

```ruby
def show
  render :show,
    home: home,
    todos: Todo.order(:created_at)
end
```

## Generate A Model

Generate a persisted model and migration:

```bash
charming generate model todo title:string done:boolean
```

This creates:

```text
app/models/todo.rb
db/migrate/20260531000000_create_todos.rb
spec/models/todo_spec.rb
```

Generated models inherit from `ApplicationRecord`:

```ruby
module TodosTui
  class Todo < ApplicationRecord
  end
end
```

## Generate A Migration

Standalone migrations follow Rails naming conventions:

```bash
charming g migration create_posts title:string body:text
charming g migration add_email_to_users email:string
charming g migration remove_email_from_users email:string
charming g migration tune_indexes        # empty change method to fill in
```

`create_<table>` builds a `create_table` block (with `t.timestamps`),
`add_<x>_to_<table>` emits `add_column` lines, `remove_<x>_from_<table>` emits
`remove_column` lines. Migration versions are guaranteed unique even when two
generators run within the same second.

## Database Commands

Run database commands from the app root. The target database follows `CHARMING_ENV`:

| Command | Does |
|---------|------|
| `charming db:create` | Create the database file and connect. |
| `charming db:migrate` | Run pending migrations, then refresh `db/schema.rb`. |
| `charming db:rollback [STEP=n]` | Roll back the last n migrations (default 1), then refresh the schema. |
| `charming db:drop` | Delete the database file. |
| `charming db:seed` | Load `db/seeds.rb` (the app is loaded first, so seeds can use your models). |
| `charming db:setup` | Create + load schema (or migrate) + seed, in one command. |
| `charming db:reset` | Drop, then setup. |
| `charming db:prepare` | CI-friendly: setup when the database is missing, otherwise just migrate. |
| `charming db:status` | Print the migration status table (up/down per version). |
| `charming db:version` | Print the current schema version. |
| `charming db:schema:dump` | Write the current structure to `db/schema.rb`. |
| `charming db:schema:load` | Load `db/schema.rb` — fast fresh setup without replaying migrations. |
| `charming db:install sqlite3` | Retrofit database support into an existing app. |

```bash
CHARMING_ENV=test charming db:prepare   # prepare the test database
charming db:rollback STEP=3             # roll back three migrations
```

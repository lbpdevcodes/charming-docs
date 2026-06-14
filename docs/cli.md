---
title: CLI & Generators
layout: default
nav_order: 14
parent: Docs
permalink: /docs/cli/
---
# CLI & Generators

The `charming` executable scaffolds new apps and generates code inside them. Run it
from your app's root directory (where the `.gemspec` lives).

## Commands

| Command | Purpose |
|---------|---------|
| `charming new NAME [--database sqlite3] [--force]` | Scaffold a new app. |
| `charming generate TYPE NAME [args]` (alias `g`) | Run a generator (see below). |
| `charming console` (alias `c`) | Boot the app and open IRB with `app` available. |
| `charming db:COMMAND` | Run a database command — see [Database]({{ '/docs/database/' | relative_url }}). |

`--force` overwrites existing files instead of skipping them. Unknown commands print
usage and exit `1`; a generator error prints to stderr and exits `1`.

### new

```bash
charming new notes                    # plain app
charming new notes --database sqlite3 # app with Active Record + SQLite wired in
```

Only `sqlite3` is supported today; any other value is rejected. See
[Getting Started]({{ '/docs/getting-started/' | relative_url }}) for a tour of the
generated app.

### console

```bash
charming console
```

Loads `lib/<app>.rb` (which sets up Zeitwerk, and the database when configured),
prints the environment banner, and starts IRB. Inside, `app` returns a memoized
application instance. Must be run from an app root; `CHARMING_ENV` selects the
environment.

## Generators

`charming generate TYPE NAME [args]` dispatches to one of:

| Type | Command | Scaffolds |
|------|---------|-----------|
| `screen` | `g screen NAME` | A full vertical slice: state, controller, view, three specs, **plus** a route and a command-palette entry (inserted idempotently). |
| `controller` | `g controller NAME [ACTION...]` | `app/controllers/<name>_controller.rb` with one action per `ACTION` (default `show`). |
| `view` | `g view NAME [ACTION]` | `app/views/<name>/<action>_view.rb` (default `show`). |
| `component` | `g component NAME` | `app/components/<name>_component.rb` — a `Charming::Component` subclass. |
| `model` | `g model NAME field:type ...` | `app/models/<name>.rb`, a `create_<table>` migration, and a model spec. **Needs a database** (`--database sqlite3` or `db:install`). |
| `migration` | `g migration NAME field:type ...` | A timestamped migration in `db/migrate/`. **Needs a database.** |

### screen — the everyday generator

```bash
charming generate screen dashboard
```

Writes the state/controller/view trio and their specs, then inserts a `screen` route
into `config/routes.rb` and a `command "Dashboard"` block into `ApplicationController`
so the screen is reachable from the command palette. Re-running it is safe — existing
files are skipped and the route/command insertions no-op if already present. Takes no
extra arguments.

### controller and view

```bash
charming generate controller settings show edit   # show + edit actions
charming generate view settings edit              # just app/views/settings/edit_view.rb
```

Each generated controller action renders its conventional view with the command
palette passed as an assign.

### model and migration

These require database support — generate the app with `--database sqlite3`, or run
`charming db:install sqlite3` in an existing app first.

```bash
charming generate model article title:string body:text published:boolean
```

Fields are `name:type` pairs. Valid types: `string`, `text`, `integer`, `float`,
`decimal`, `boolean`, `date`, `datetime`, `time`. The model generator emits a
`create_<table>` migration (with `t.timestamps`) and a spec alongside the model.

Migration names follow Rails conventions, which decide the generated body:

| Name pattern | Generates |
|--------------|-----------|
| `create_<table>` | a `create_table` block with one column per field |
| `add_<x>_to_<table>` | `add_column` lines for each field |
| `remove_<x>_from_<table>` | `remove_column` lines for each field |
| anything else | an empty `change` with a placeholder comment |

```bash
charming generate migration add_author_to_articles author:string
```

## db:install

To add database support to an app created without it:

```bash
charming db:install sqlite3
```

This patches the app idempotently — it updates the gemspec (file glob + dependencies)
and the `lib/<app>.rb` loader, and creates `config/database.rb`,
`app/models/application_record.rb`, `db/migrate/.keep`, and `db/seeds.rb`. After it
runs, the `model`/`migration` generators and the `db:*` commands are available. See
[Database]({{ '/docs/database/' | relative_url }}) for the full command table.

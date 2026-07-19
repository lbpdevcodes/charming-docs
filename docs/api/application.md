---
title: Application & Configuration
layout: default
parent: API Reference
nav_order: 1
permalink: /docs/api/application/
---
# Application & Configuration

`Charming::Application` is the root object of every Charming app: it owns the routes, the logger, the registered themes, and the persistent session. You subclass it once, configure it with the class-level DSL below, and hand an instance to `Charming.run` to start the app. Everything else — controllers, views, components — hangs off this object at runtime.

## Charming::Application

Inherit from `Charming::Application`:

```ruby
class MyApp::Application < Charming::Application
  root File.expand_path("../..", __dir__)
end
```

Generated apps define routes separately in `config/routes.rb`:

```ruby
MyApp::Application.routes do
  root "home#show"
end
```

Routes can also be defined inline on the application class:

```ruby
class MyApp::Application < Charming::Application
  root File.expand_path("../..", __dir__)

  routes do
    root "home#show"
  end
end
```

### Class APIs

Routing and structure — where the app lives and how screens are reached:

- `routes { ... }` defines routes with the router DSL.
- `root path` sets the application root path used for resolving relative files and templates.
- `namespace` returns the application namespace used for controller and template binding lookup.

Logging:

- `logger value` sets the application logger. The default logger writes to `File::NULL`.

Themes — register and pick the app's look (tokens are covered in [UI & Themes]({{ '/docs/api/ui/' | relative_url }})):

- `theme name, built_in: "phosphor"` registers a built-in JSON theme (`phosphor`, `catppuccin-mocha`, `catppuccin-latte`, `gruvbox-dark`, `nord`, `tokyonight`).
- `theme name, from: "config/themes/custom.json"` registers an app-local theme file.
- `theme name, extends: :parent, overrides: {token => spec}` derives a theme from a registered one.
- `default_theme name` sets the default theme.
- `theme_for name` resolves a theme object.

Session and input behavior:

- `persist_session to: "tmp/session.json"` opts into JSON session persistence across restarts (framework-internal keys and non-JSON-safe values are excluded).
- `coalesce_input true` collapses held-key auto-repeat bursts into one dispatch (off by default).
- `mouse_motion :all` enables buttonless hover reporting (`:drag`, the default, reports motion only while a button is held).

### Instance APIs

- `routes` returns the app router.
- `logger` returns the application logger.
- `logger=` overrides the logger for this application instance.
- `session` returns persistent app session state.
- `save_session` writes the session to the configured path (the runtime calls this on exit).
- `theme` returns the active theme.
- `use_theme name` switches the active theme.

## Environment

- `Charming.env` returns the current environment as a string inquirer (from `CHARMING_ENV`, default `"development"`): `Charming.env.test?`, `Charming.env.production?`.

## Entrypoint

Start the runtime by handing it an application instance:

```ruby
Charming.run(MyApp::Application.new)
```

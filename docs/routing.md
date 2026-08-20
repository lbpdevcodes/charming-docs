---
title: Routing
layout: default
nav_order: 4
permalink: /docs/routing/
---
# Routing

Generated apps define routes in `config/routes.rb` by calling `routes` on the application class. Each screen registers under a Symbol name:

```ruby
MyApp::Application.routes do
  root "home#show"
  screen :cities, "cities#index", title: "Cities"
  screen :city, "cities#show", title: "City"
end
```

## Route DSL

| Method | Purpose |
|--------|---------|
| `root "home#show"` | Maps the reserved `:root` screen to `HomeController#show`. |
| `screen :name, "controller#action"` | Registers a screen name for a controller action. |
| `title:` | Sets a display title used by generated sidebar layouts. |

Controller names are resolved inside the application namespace. In a generated `MyApp` app, `"home#show"` resolves to `MyApp::HomeController`. When the `#action` part is omitted (`"home"`), the action defaults to `#show`.

## Navigation and Params

Controllers navigate by screen name and pass params directly:

```ruby
navigate :city, id: city.id
```

Controller actions access params through `params`:

```ruby
module MyApp
  class CitiesController < ApplicationController
    def show
      render "City #{params[:id]}"
    end
  end
end
```

Params pass through with their Ruby values intact. There is no URL decoding and no string conversion: `navigate :city, id: 5` yields `params[:id] == 5`. `navigate :root` goes home.

## Resolution Rules

- Unknown screen names raise `KeyError` listing the registered names.
- A string URL path passed to `screen` or `navigate` raises `ArgumentError` with a migration hint.
- `application.routes.all` returns routes in insertion order.

Generated layouts use `application.routes.all` to build sidebar navigation, and
sidebar rows respond to mouse clicks (a click on a route row navigates to it).
Controllers can override `sidebar_routes` to filter the list — for example, to hide
screens that only make sense with an id:

```ruby
def sidebar_routes
  application.routes.all.reject { |route| %i[city edit_city].include?(route.name) }
end
```

## Route Titles

When no `title:` is supplied, Charming derives one from the screen name:

```ruby
screen :project_settings, "settings#show"
# title: "Project Settings"
```

Use explicit titles for sidebar labels that should differ from the name.

## Generating Screens

Inside a generated app, create a screen with:

```sh
charming generate screen forecast
```

That creates a state object, controller, template, specs, inserts a route, and inserts a command palette entry.

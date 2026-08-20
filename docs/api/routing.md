---
title: Routing
layout: default
parent: API Reference
nav_order: 2
permalink: /docs/api/routing/
---
# Routing

The router maps named screens to a controller action. You declare routes once with a small DSL (usually in `config/routes.rb`), and the runtime resolves every navigation through them — there is no other way to reach a screen. For the broader picture of how routes, controllers, and views fit together, see [Routing]({{ '/docs/routing/' | relative_url }}).

## Defining routes

Routes are usually defined in `config/routes.rb`:

```ruby
MyApp::Application.routes do
  root "home#show"
  screen :user, "users#show", title: "User"
end
```

DSL methods:

- `root "controller#action", title: "Home"` registers the reserved `:root` screen.
- `screen :name, "controller#action", title: nil` registers a named screen. The name is a Symbol; the target is positional.
- Omitting `#action` in the target defaults the action to `#show`.

## Resolution rules

- Register one named screen per parametric page.
- Pass params at navigate time as Ruby values: `navigate :user, id: 42`.
- Params are symbol-keyed, for example `params[:id]`. There is no URL decoding.
- `Router#resolve(:name)` raises `KeyError` listing the registered names on a miss.
- String paths raise `ArgumentError` with a migration hint.

So `navigate :user, id: 42` with the routes above dispatches `UsersController#show` with `params[:id] == 42`.

## Route objects

Route objects expose:

- `name` (a Symbol)
- `controller_class`
- `action`
- `title`
- `params`

Filter routes by name, for example when overriding the sidebar list:

```ruby
application.routes.all.reject { |route| %i[city edit_city].include?(route.name) }
```

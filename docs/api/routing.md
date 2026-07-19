---
title: Routing
layout: default
parent: API Reference
nav_order: 2
permalink: /docs/api/routing/
---
# Routing

The router maps screen paths like `/users/42` to a controller action. You declare routes once with a small DSL (usually in `config/routes.rb`), and the runtime resolves every navigation through them — there is no other way to reach a screen. For the broader picture of how routes, controllers, and views fit together, see [Routing]({{ '/docs/routing/' | relative_url }}).

## Defining routes

Routes are usually defined in `config/routes.rb`:

```ruby
MyApp::Application.routes do
  root "home#show"
  screen "/users/:id", to: "users#show", title: "User"
end
```

DSL methods:

- `root "controller#action", title: "Home"` maps `/`.
- `screen "/path", to: "controller#action", title: nil` maps a screen path.
- Omitting `#action` in `to:` defaults the action to `#show`.

## Resolution rules

- Exact routes win over dynamic routes.
- Dynamic segments use `:name` and match one segment.
- Params are symbol-keyed, for example `params[:id]`.
- Params are URL-decoded.
- Missing routes raise `KeyError`.

So navigating to `/users/42` with the routes above dispatches `UsersController#show` with `params[:id] == "42"`.

## Route objects

Route objects expose:

- `path`
- `controller_class`
- `action`
- `title`
- `params`

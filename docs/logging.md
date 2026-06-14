---
title: Logging
layout: default
nav_order: 12
parent: Docs
permalink: /docs/logging/
---
# Logging

Charming applications have a logger available from the application and from controllers.

By default, the logger writes to `File::NULL`, which is the operating system's null device. Logging calls are safe to make, but no file is created and nothing is printed to the terminal.

```ruby
logger.info("Loaded todos")
logger.debug("Current route: #{route.path}")
```

## Configure A Logger

Configure a logger on your application class:

```ruby
require "fileutils"
require "logger"

class MyApp::Application < Charming::Application
  root File.expand_path("../..", __dir__)

  FileUtils.mkdir_p(File.join(root, "log"))
  logger Logger.new(File.join(root, "log", "app.log"))
end
```

Controllers access the configured application logger through `logger`:

```ruby
class MyApp::HomeController < MyApp::ApplicationController
  def show
    logger.info("Rendering home screen")
    render :show
  end
end
```

## Override At Runtime

Applications can also replace the logger before running:

```ruby
app = MyApp::Application.new
app.logger = Logger.new(options.fetch(:log_file))

Charming.run(app)
```

This is useful when your executable exposes its own `--log-file` option or reads a user config file.

## Avoid Terminal Streams

Avoid logging to `$stdout` or `$stderr` while the TUI is running. Charming renders into an alternate terminal screen, and writing ordinary log output to the terminal can corrupt the interface.

Prefer file logging:

```ruby
FileUtils.mkdir_p(File.join(root, "log"))
logger Logger.new(File.join(root, "log", "app.log"))
```

Or keep the default null logger when you do not need logs.

## Default Behavior

The default logger is equivalent to:

```ruby
Logger.new(File::NULL)
```

`File::NULL` maps to `/dev/null` on macOS and Linux, and `NUL` on Windows.

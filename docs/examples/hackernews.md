---
title: Hacker News
layout: default
parent: Examples
nav_order: 2
permalink: /docs/examples/hackernews/
---
# Hacker News

A terminal reader for Hacker News. Browse the
top, new, best, Ask HN, Show HN, and jobs feeds, open articles inline, and
navigate with a Vim-inspired keyboard layout — all themeable with the built-in
Charming themes. The source lives at
[pandorocks/hackernews](https://github.com/pandorocks/hackernews) and it
installs as the [`hackernews`](https://rubygems.org/gems/hackernews) gem.

<img src="{{ '/assets/images/hn-screenshot.webp' | relative_url }}"
     alt="A Charming TUI app rendering a Hacker News reader in the terminal">

## Run It

From RubyGems:

```sh
gem install hackernews
hackernews
```

Or from a source checkout:

```sh
git clone https://github.com/pandorocks/hackernews
cd hackernews
bundle install
bundle exec hackernews
```

Inline article reading extracts full article text with
[trafilatura](https://github.com/adverb-xyz/trafilatura), which must be
installed and on your PATH.

## Keys

| Key | Does |
|-----|------|
| `j` / `k`, arrows | move cursor |
| page up / page down | jump 10 items |
| `n` / right | next page of stories |
| left | previous page |
| `enter` | open selected article |
| `esc` | close article, back to feed |
| `r` | refresh current feed |

## What It Demonstrates

- **Async background loading** — feeds load in parallel threads without
  blocking the interface.
- **Talking to a real API** — the controller fetches from the Firebase-hosted
  Hacker News API with HTTParty and caches articles in state.
- **State without a database** — `HomeState` holds stories by page, the feed
  index, the article cache, and loading indicators, all in memory.
- **Reusable components** — widgets like `StoryListComponent` handle their own
  rendering.

---
title: Pac-Man
layout: default
parent: Examples
nav_order: 3
permalink: /docs/examples/pacman/
---
# Pac-Man

A Pac-Man arcade clone built with Charming, showing what the framework looks
like when the "app" is a real-time game: a render loop driven by timers, a
board that scales to your terminal, half-block sprite drawing, and SQLite-backed
high scores. The source lives at
[lbpdevcodes/pacman](https://github.com/lbpdevcodes/pacman) and it installs as the
[`pacmanrb`](https://rubygems.org/gems/pacmanrb) gem.

<img width="1256" height="1065" alt="Pac-Man running in a terminal" src="https://github.com/user-attachments/assets/72cc474b-f54d-40aa-b622-6e203251e0ca" />

## Run It

From RubyGems:

```sh
gem install pacmanrb
pacmanrb
```

Or from a source checkout:

```sh
git clone https://github.com/lbpdevcodes/pacman
cd pacman
bundle install
bundle exec pacmanrb
```

## How to Play

- **Enter** on the title screen starts a game.
- Steer with the **arrow keys** or **WASD** — turns are buffered and taken at
  the next opening, like the arcade.
- Eat every pellet to clear the level; levels get faster. Power pellets — drawn
  as cherries — frighten the ghosts so you can eat them for combo bonuses
  (200/400/800/1600); eaten ghosts race home as floating eyes.
- You have 3 lives. Game over shows the top-5 high score table, persisted in
  SQLite.
- **r**/**Enter** restarts from the game over screen; **q** quits anywhere;
  **ctrl+p** opens the command palette.

The four ghosts have distinct brains: one chases you, one ambushes four tiles
ahead, one wanders randomly, one patrols its corner. They alternate between
scatter and chase on a timer.

## What It Demonstrates

- **Game rules as plain Ruby** — the arcade logic under `lib/pacman/arcade/`
  has no TTY dependency, so the whole rule set is unit-tested headlessly.
- **The Charming app layer** — controllers, views, and components under `app/`
  render the board and screens.
- **Procedural sprites** — at larger terminal sizes the actors are drawn as
  half-block sprites: Pac-Man is a circle whose mouth points where he's headed,
  ghosts get their classic domed-and-scalloped shape.
- **Selective persistence** — live game state stays in memory per run; only
  high scores persist to the database.

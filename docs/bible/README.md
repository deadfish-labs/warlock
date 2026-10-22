# Warlock — Game Bible

Everything we know about **Warlock** and everything we decide about the remake lives here. If it isn't in the Bible,
it isn't decided.

## What Warlock is

Warlock is a player-versus-player arena game that started as a custom map for **Warcraft III**, made in the game's
World Editor. Each player controls one warlock on a platform surrounded by lava. Spells deal damage, but above all they
**push** other warlocks; the arena shrinks as the round goes on, and whoever is pushed into the lava burns. The last
warlock standing wins the round. Between rounds players spend gold on new spells, upgrades and items.

The remake rebuilds that game for today's platforms, with its own art, sound and code. Warcraft III assets are not
used.

## Source

| | |
|---|---|
| Original map | `Warlock.w3x` (in this repository), Warcraft III custom map |
| Version | **1.06n**, community edition (the map file in this repository; gameplay videos often show the older 0.99c) |
| Map authors | Demestotenes & Zymoran; 1.06n by Zymoran and Adynathos; community edition by Quetra |
| Special thanks (from the map's credits) | Chindril (supportive editing), Vexorian (WC3 map optimizer), and others listed in the credits chapter |

All numbers in this Bible come from the map itself (its script and object data) unless marked otherwise.

## Chapters

| # | Chapter |
|---|---|
| 00 | [Overview: a round from start to end, characters, credits](00-overview.md) |
| 01 | [Spells](01-spells.md) |
| 02 | [Items and the shop](02-items-shop.md) |
| 03 | [Arena and lava](03-arena-lava.md) |
| 04 | [Movement, physics and knockback](04-physics-knockback.md) |
| 05 | [Rounds, scoring and game modes](05-rounds-scoring-modes.md) |
| 06 | [Commands, camera and HUD](06-commands-ui.md) |
| 07 | [Remake requirements](07-remake-requirements.md) |
| 08 | [Multiplayer and the server (Colyseus)](08-multiplayer-server.md) |
| 09 | [Scope of the first version](09-scope-v1.md) |
| 10 | [Roadmap](10-roadmap.md) |
| 11 | [The lobby platform (vision)](11-lobby-platform.md) |
| 12 | [Client technology](12-client-technology.md) |
| 13 | Art and audio direction (to write) |

Also: [Decisions](decisions.md) · Sources: [the map's help screens](sources/in-game-help.md),
[what the map data covers](sources/data-coverage.md)

## Conventions

- One chapter per file, `NN-name.md`.
- Facts from the original map cite where they come from (script function, object id and field).
- Anything not yet confirmed is marked **uncertain**.
- Decisions for the remake are marked **Decision** with a date.

# 08 — Multiplayer and the server

Warlock is an online player-versus-player game: every match is several players in one arena, and the outcome of every
push decides who burns. The server design follows from that.

## Decision (owner, 2026-10-22): Colyseus

The server is built with **[Colyseus](https://colyseus.io)**.

What Colyseus is (from its site, version 0.18 at the time of writing):
- An open-source (MIT, free for commercial games) real-time multiplayer framework for **Node.js**, written in and for
  **TypeScript**.
- **Rooms:** each match is a room on the server, with code for joining, leaving and the game loop.
- **State sync:** the server owns the game state; Colyseus sends it to every client as compact binary patches.
- **Authoritative server:** clients send inputs, the server decides what happens.
- **Matchmaking:** creates rooms and places players, with filters and sorting.
- **Scaling:** rooms spread over several processes or machines.
- **Client SDKs:** TypeScript/JavaScript, Unity, Defold, Haxe, Construct 3, React; Godot, GameMaker and C in alpha.
- **Hosting:** self-hosted anywhere Node.js runs, or Colyseus Cloud (managed, from US$15/month).

## Why it fits Warlock

Warlock is decided by physics: where a fireball lands, how far a warlock slides, whether they stop at the lava's edge.
If each player's machine decided that for itself, two players would see different results and cheating would be easy.
An **authoritative server** that runs the one true simulation and tells everyone the result is the right model, and
that is exactly what Colyseus is built around. A match maps one-to-one onto a Colyseus room.

## Proposed architecture

```
 players' clients                         Colyseus server (Node.js, TypeScript)
 ───────────────                          ─────────────────────────────────────
 input: move target, cast spell + aim ──▶  Lobby / matchmaking room
                                           Match room (one per game):
                                             fixed-step simulation (e.g. 30 ticks/s)
                                               movement, knockback, collisions,
                                               spells and projectiles, lava damage,
                                               arena shrinking, rounds, gold, points
 render + interpolate state           ◀──    state sync (Colyseus schema)
```

- **Simulation on the server only.** All rules from chapters 01–05 run inside the match room at a fixed tick rate.
- **Clients send intentions, not results:** "move here", "cast fireball towards this point", "buy this item".
- **Smoothness on the client:** other players are drawn slightly in the past and interpolated between server states;
  the player's own movement can be predicted locally and corrected when the server answers. The game's feel depends on
  this, so it gets its own chapter once the client engine is chosen.
- **Shared rules code:** the simulation is plain TypeScript with no server-only dependencies, so the same code can run
  on the client for prediction and in tests that replay matches.
- **Phases as room state:** lobby → shop → round → round end → shop … → match end (chapter 05).

## Open questions

- **Client engine.** Colyseus does not render anything. The client could be a web game (TypeScript, e.g. Phaser or
  PixiJS, playable in a browser with no install) or an engine with a Colyseus SDK (Unity, Defold, Godot in alpha).
- **Players per match** and team modes: the original map's limits are documented in chapter 05.
- **Hosting:** self-hosted on our own servers or Colyseus Cloud; depends on where players are.
- **Accounts and persistence** (names, stats, cosmetics like the original map's contributor "Lich" look): not part of
  Colyseus itself; needs a small service and database alongside it.
- **Tick rate and bandwidth:** to measure with a prototype once we leave the planning phase.

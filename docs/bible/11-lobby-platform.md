# 11 — The lobby platform (vision)

**Roadmap: phase 3** (chapter 10). The game is built first; this chapter records the bigger picture so early technical
choices don't block it. Described by the owner on 2026-10-22.

## The idea

A lobby that sits above the games. Warlock is the first game in it; others can follow. From the lobby a player can:

| Feature | What it means |
|---|---|
| **Room list** | See every room that is open: which game, how many players are in it and how many places are left. |
| **Friends** | A friends list; see who is online. |
| **Invites** | Invite friends into a group. |
| **Groups (parties)** | Play together as a group. The group has a **leader**. |
| **Voice chat** | Members of a group talk to each other by voice while in the lobby. |
| **Leader picks the game** | The leader chooses a room (or a game); the whole group joins it together. |
| **Auto-match** | The group is placed automatically in a room that has enough free places for all its members. |

## How it fits the technology

- **Room list:** Colyseus can publish the list of open rooms with their player counts and custom metadata (game,
  mode, map) and update it live, so the lobby can show it without asking each server.
- **Group joins together:** the leader picks a room; the server reserves one seat per member and every member's
  client joins with its reservation, so nobody loses their place in a nearly full room.
- **Auto-match:** matchmaking filters rooms by game and by free seats ≥ group size, and opens a new room when none fits.
- **Friends, groups, invites, presence:** need player accounts (phase 2) and a small service with a database.
- **Voice chat:** separate from the game server. Usually real-time audio between browsers and phones over WebRTC, with
  a media server when groups get bigger than a few people. To choose later.
- **Several games:** each game is its own room type behind the same lobby, accounts and friends.

## What this means for phase 1

Nothing has to be built now, but phase 1 should not close these doors:
- Matches are Colyseus rooms with metadata (game, mode, players, free seats).
- Joining a room goes through one entry point that a lobby can call later.
- Player identity is a separate piece that accounts can replace later.

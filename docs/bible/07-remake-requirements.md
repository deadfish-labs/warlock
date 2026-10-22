# 07 - What a faithful remake must reproduce (MUST / SHOULD / COULD) + multiplayer needs

Basis: files 00-06 in this folder (all numbers come from the 1.06n map data; the video version 0.99c is probably a subset, see 00 §0). Legal/art premise: ORIGINAL art, models, sound, UI, names for UI text where they are trademarked; no Blizzard assets; do not reuse the map's custom models/sounds (WarlockModel, HereticModel, FireMageModel, OccultistModel, DruidOfTheTalon..., fbnosound.mdl, crown.mdl, LavaFire_*.mdl, Present.mdl, announcer sounds) without the authors' permission.

## 1. Product definition

Online PvP arena brawler, 2-12 players per match (slots 0-11), top-down fixed camera, match = a series of rounds (default 11) in which players cast projectile/utility spells to knock opponents into lava on a shrinking platform; between rounds a 30 s shop phase to buy spells, spell upgrades, masteries and items with gold.

## 2. MUST (without these it is not "Warlock")

1. **Fixed-step deterministic simulation at 33.33 Hz (0.03 s)** with the exact movement model of 04: walk velocity + knockback velocity, friction 0.96/tick, ice 0.93, sweep-tested sphere collisions, per-object `Range` as time, flags/priority/pierce-once mask, deferred destroy.
2. **HP / DP model**: max HP 100 (+items), DP accumulates damage, knockback `(100+DP) * dmg * 0.03 * multipliers * spellKnockbackFactor`; Heal reduces DP by half the amount; HP/DP reset every round; death at HP < 0.5; killer credit = last damager; assist = highest damage dealer.
3. **Lava platform**: lava tick 0.1 s, 0.9 HP per tick (9 HP/s), +0.45 DP per tick; safe radius `9+floor(sqrt(players))` tiles of 128; shrink 1 tile every `15*sqrt(alive)` s until 0; circle or square; donut optional; clock/timer UI for the next shrink; map-edge clamp reflect 0.5.
4. **Round flow**: first shop 40 s, shop 30 s; spawn ring (`768+64*sqrt(2n)`), 4 s spawn immortality (damage x0.1 and knockback x0.1), round ends when one team remains, dead players spectate/shop, revived next round; +10 gold per round; points; win after N rounds; tie-break rounds.
5. **Character roster** (at least Warlock) and **spell system**: fireball always; 6 spell columns D/R/T/C/E/Y + F character spell; each spell with its own cooldown, levels, toggled alternate form; all the spells listed in 01 (priority order is in §4). Spells are physics objects (missiles with speed, size, range-in-seconds, bounce, collision response) created through one common framework.
6. **Shop**: Merchant menu tree (Primary/Secondary spells, Items, Upgrades, Masteries, Ring, Sell), prices from 02, spell purchase penalty after 3rd-5th spell, sell refund rules, inventory slots 2/4/6, item stat effects (HP, regen, speed, mass, knockback reduction, missile range, lifesteal weakness, flat damage reduction).
7. **Pillars**: obstacles with HP `3*players+17`, collision bounce rules, velocity damage `0.7*sqrt(v-20)`, explosion `DealKnockbackAoE(320, 14)`, shards (offensive/defensive), layouts.
8. **Player-player collisions**: mass-weighted bounce and velocity-based collision damage.
9. **Buff/curse system** with duration multipliers (`TimeInc` buffs, `TimeDec` curses), DPS buffs (burn), silence, stun/freeze, slow, entangle; the `-stats`-style readout.
10. **Teams**: FFA default, team games (2v2 .. 6v6), observers, leaving mid-game, host settings before start.
11. **Online multiplayer** with low-latency input (see §5).
12. **Scoreboard** (name, kills, points, damage per round), floating damage numbers, kill feed with spree/first-blood messages.

## 3. SHOULD

- All five characters (Heretic, Inquisitor, Occultist, Enchanter) with their F spells.
- Modes: Deathmatch (point/time limit, auto-revive 10 s, ESC shop), Avatar, King; presets `league`, `competitive (-cm)`, `mgl` (5v5), duel (first to 6).
- Masteries, Stone of Jordan ring, Aegis, Lava Treads, all items.
- Host settings UI covering `-c1..-c17` constants and bans.
- Camera zoom setting, fireball colour setting, APM display, hints and 10-step tutorial text (rewrite in own words).
- Terrain themes (5) and random obstacle layouts 2-11; ice patches.
- Spectator view and auto-observer for idle players.
- Replays (deterministic sim makes this cheap: record inputs + seed).
- Bots (AI) for practice / empty slots (the original had only test-mode bots; a remake needs real ones for solo play).

## 4. COULD / later

- Custom terrain codes and a terrain/arena editor; hotkey remapper (the original community had external tools).
- Cosmetics/titles/tokens economy, league ranking site, tournaments (`WarlockLeague` export).
- Christmas (`-ymas`) event theme with present shards.
- KOTH (rules unclear in source).
- Hat, announcer packs, fatality (finisher) effects.

Spell implementation priority suggestion (based on frequency in tutorials/hints and being the core of play): Fireball, Scourge, Lightning, Homing, Boomerang, Bolt (D); Teleport, Thrust, Shield (R); Drain, Fire Spray, Dagger, Bouncer (T); Time Shift, Ethereal, Mirror, Vector, Dome, Rush (C); Wind Walk, Splitter, Burst, Repeater, Meteor (E); Link, Gravity, Disable, Control, Lock On (Y). Check 01-spells.md for the exact list actually present.

## 5. Multiplayer requirements

- **Match size**: 12 simultaneous humans max (slots 0-11), typical 2-8; 1v1 duels are a first-class mode (first to 6).
- **Authority & sync model**. WC3 ran deterministic lockstep: every client simulates the same fixed-step game from the same inputs (input delay ~100-250 ms). The JASS simulation is deterministic given the same inputs (only `GetRandomInt/Real` used, synced by WC3). Options for the remake:
  1. *Deterministic lockstep* (cheapest server, replays for free, but needs bit-exact determinism: use fixed-point or strictly controlled float math; 12 players on mobile/web with varied hardware makes float determinism risky) - or
  2. *Server-authoritative* simulation at 33 Hz with snapshots (~12 players + ~60 missiles + pillars are tiny; < 8 KB/s per client), client-side prediction for own movement (walk velocity is simple) and server reconciliation; knockback is applied only on the server (so everybody sees consistent hits). Recommended for web/mobile.
- **Latency budget**: knockback/avoidance gameplay is tight (projectile 1000 u/s; a 100 ms delay = 100 u). Target <= 100 ms RTT regions; interpolate remote players ~66-100 ms behind; rewind/lag-compensation for hit detection optional (original had none; it relied on lockstep delay).
- **Inputs**: move-to point/direction, cast spell (id, target x,y), stop, toggle spell form, buy/sell/upgrade (shop actions are plain commands), select character, chat commands, ready/vote. All inputs timestamped with tick numbers.
- **Server data**: RNG seeds per round (pillar layout, terrain theme, ring angle, shuffle), player slots, teams, ruleset object (all `C_*` constants plus mode flags), authoritative gold/inventory/spell levels.
- **Lobby**: create game with host settings (mode, rounds, gold, shop times, terrain, bans, teams), join/leave, ready, character select (15 s), team select, observers, reconnect (original: leave = death; `PlayerReturnToGame` exists for observers), AFK detection (APM in last minute == 0 -> observer).
- **Anti-cheat**: server authority solves most of what the JASS `RSA`/`AntiJassProtection` tried to do. Rate-limit inputs, validate cooldowns/costs on the server.
- **Matchmaking & stats** (later): rating from duel results / league stats (kills, deaths, assists, fireball hit %, APM, score) exported at game end.
- **Platforms**: not specified. Mobile touch is a design question: original is mouse (right-click move + hotkeys + shop UI).
- **Pause**: original supports pausing (`GamePaused`, `-unpause`); in online remake pause is a lobby/host feature.
- **Match duration**: 11 rounds x (shop 30 s + fight ~30-120 s) = 20-35 minutes typical.

## 6. Testing the remake against the original

- Port the formulas as unit tests: knockback impulse table (04 §4 example), lava damage per tick, shrink schedule, collision damage, pillar damage.
- Golden replays: record scripted inputs in the remake and compare displacement of a knocked-back player to the closed form (`v0/(1-0.96)`).
- Compare feel with the gameplay video the owner has (0.99c) frame-by-frame for projectile speeds and knockback distances; use it to decide where 1.06n numbers differ from 0.99c.

## 7. Open questions for the owner

1. **Which version is the target?** The extracted map is Warlock 1.06n (community edition) with 5 characters, 5 modes, cosmetics and league export, but the video is 0.99c, whose quest text still mentions `-splitter` and `-oath` commands that no longer exist in the code. Do we remake 1.06n's feature set (recommended) or approximate 0.99c from the video?
2. **Rights and naming**: do we have the permission of the authors (Demestotenes, Zymoran, Adynathos, Quetra for CE) to remake and sell/host it, and what must the name/credits be? Shall we use "Warlock" or a new original title/spell names? (Warcraft/Blizzard IP concerns: avoid "Warcraft" terms, Blizzard-looking assets.)
3. **Platform and controls**: web, desktop, mobile? Click-to-move + hotkeys (original) vs WASD / touch joystick + aim; keeps the 220 u/s feel and 1650 camera distance or a touch-friendlier design?
4. **Networking choice**: server-authoritative (recommended) vs deterministic lockstep; hosting budget; region count; max players per match (12 as original, or fewer?); do we need solo/bots from day one?
5. **v1 scope**: Round mode only with the Warlock character and the core ~25 spells, or all characters/modes/items? And is the cosmetics/tokens/titles/league economy in scope (monetisation, accounts)?

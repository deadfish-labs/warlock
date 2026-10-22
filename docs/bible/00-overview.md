# 00 - Overview: "Warlock" (WC3 custom map) - remake planning

Sources: `~/warlock-src/out/` (extracted MPQ), JASS `scripts\war3map.j` (21,352 lines once CR line-endings are normalised; copy at `notes/data/war3map.j.txt`), `war3map.wts` (3,073 strings), object data dumps in `notes/data/*.json`.
Citations below use "JASS Fn" (function name; line numbers refer to `notes/data/war3map.j.txt`) and "w3x id field" (object data).

## 0. IMPORTANT: version discrepancy

The brief said the version seen in the gameplay video is 0.99c. **The extracted map is not 0.99c.** `war3map.w3i` says:

- map name: `Warlock 1.06n`
- author field: `Zymoran and Adynathos`
- description: "Bug reports and feedback to: Quetra (Author of CE) ... Download the latest version at www.WarlockBrawl.com"

So the data we have is the later community edition ("CE") 1.06n. It contains features that a 0.99c build almost certainly does not have: five playable characters, five game modes, titles/cosmetics/tokens, Warlock League stat reporting, ban system, tutorial/hints, custom-terrain codes. A faithful remake of "the game from the video" needs the owner to decide which version is the target (see 07-remake-requirements.md, open questions). All numbers in these notes are from 1.06n.

## 1. What the game is

A top-down, last-one-standing arena brawler for 2-12 players (player slots 0-11; slot 12 = neutral/"no team"). Each player controls one caster ("warlock") on an island of safe ground surrounded by lava. Nobody has a normal HP-and-armour fight; instead:

- Every hit deals damage that lowers HP (max 100) **and** adds to a "damage points" (DP) meter (displayed in the unit's mana bar).
- Knockback strength scales with DP: `(100 + DP) * damage * ...` (see 04-physics-knockback.md). The more hit, the further you fly.
- The safe ground shrinks over time; standing on lava drains HP (default 9 HP/s).
- You die at HP < 0.5 (hit, lava, or both). Being knocked into lava is the main kill method.
- The last living team wins the round and scores points; between rounds everybody visits the shop to buy spells, upgrades and items with gold.

Players control movement with the standard WC3 right-click move, and spells with the hotkeys listed in 01-spells.md. The simulation (positions, velocities, collisions) is custom JASS running at a fixed tick of 0.03 s (JASS `Sys_IterRate`), not WC3 unit physics.

## 2. Credits (from map text, wts strings 0, 6, 9, and w3i)

- "Map made by: **Demestotenes** & **Zymoran**" (wts 0)
- Special thanks: **Chindril** (supportive editing), **Vexorian** (wc3 map optimizer), **General Frank** (standard warlock model). (wts 0)
- wts 0 also says players may see a **Lich** instead of a Warlock - dedicated players who contributed to the map; "Lich and Warlock have exactly the same stats".
- w3i: author "Zymoran and Adynathos"; bug reports to "Quetra (Author of CE)". (CE = community edition)
- Contact in wts 6: <e-mail removed>, <e-mail removed>; forum/host site `www.WarlockBrawl.com`. (wts 8/9: "Always make sure that you host the latest version from www.WarlockBrawl.com ... no cheats and all known bugs are fixed").
- JASS includes third-party libraries: ASCII, HCL (host command line), List, ObjectStruct (custom physics objects), TimerSys, Multiboard, Tips, WarlockLeague, CTConversion (custom terrain code), HEXConverter, RSA (anti-cheat), "Attachment System", "Pillars". These are implementation details, not gameplay.
- Original art/sound is Blizzard's; the remake must replace everything (the JASS references many Blizzard `.mdl` models and sounds). Custom models in the map (WarlockModel, HereticModel, FireMageModel, OccultistModel, DruidOfTheTalonBloodElfHeroV1.01, `fbnosound.mdl`, `LavaFire_Blue.mdl`, Present.mdl...) have their own authors (General Frank etc.) and must NOT be reused without clearance.

## 3. A round from start to end (Round mode, the default)

Defaults: `MaxRounds=11`, FFA on, Shuffle off, `C_StartingGold=20`, `C_RoundAddedGold=10`, `C_ShopTimeDuration=30` s, `C_ShopTimeDurationFirst=40` s, `C_ShrinkTimePerPlayer=15` s (JASS globals lines 514-534, 1139, 673).

1. **Lobby/host setup.** Host types setup commands (see 06). Then a character-select phase (15 s, 5 s in test mode, skipped when only one character is allowed) where each player picks Warlock / Heretic / Inquisitor / Occultist / Enchanter by clicking a unit (JASS `GameStart`, `CharacterSelectId`).
2. **First shop phase (40 s).** Each player stands in a row at y=-64 (`x = 128*(slot-6)-64`), with their own Merchant unit at y=64. Players have `C_StartingGold` (20) gold. They buy spells/upgrades/items (02-items-shop.md). A timer dialog "Shop time:" counts down. Arena is pre-shrunk to `8 + alive/2` tiles and lava timer is paused. (`InitMode_Round`)
3. **Round start** (`RoundStart_2`): screen fades; if Shuffle then teams reshuffled; all warlocks teleported to a ring around the centre (radius `768 + 64*sqrt(2*PlayerCount)` world units, angle spaced evenly with a random start; teams share an angle in team games); stats reset (HP 100, DP 0); `CurrentRound++`; `ShopTime=false`; `TerrainChange()` picks the lava/ground tileset and obstacle layout; lava starts; every player is `Immortalize`d for `C_ImmortTime = 4` s (damage dealt x0.1, knockback taken x0.1, cannot be killed).
4. **Fighting.** Safe radius starts at `9 + floor(sqrt(PlayerCount))` tiles (128 units each) and shrinks by 1 tile every `15 * sqrt(AliveCount)` seconds (03-arena-lava.md). Lava ticks every 0.1 s. Pillars/obstacles can be destroyed into shards (power-ups). Kills give a killing-spree announcer.
5. **Round end.** `Round_CheckForWinner` runs after every death: when all living players belong to one team, that team wins (or "Draw!" if nobody lives). Winner(s) get `C_WinPointReward` points (1 by default; the "default values" help text in wts 1 says 2, uncertain which is shipped - the code default is 1). Highest round-damage dealer is announced (and gets `C_DmgGoldReward` gold, 0 by default).
6. **Between rounds** (`RoundEnd`): everyone (including the dead) is revived, teleported to the shop row, the Merchant unit is recreated, everyone gets `+C_RoundAddedGold` (10) gold; "tokens" (lumber) are credited equal to points won; a 30 s shop timer starts; when it ends `RoundStart` runs again.
7. **Game end.** After `CurrentRound >= MaxRounds` (or a duel hits its early-win condition) `GameEndAction` computes the winner by highest points (draw -> an extra decider round/kill), shows Victory/Defeat/"Epic victory" screens with a rotating camera around the winner, and announces "Game will exit in 3 minutes."

Details of scoring, other modes and win conditions are in 05-rounds-scoring-modes.md.

## 4. Characters (5)

Selected at game start. All have 100 max HP, 200 max "mana" (shows DP), move speed 220, HP regen from player stat `P_Regeneration` (JASS init 0.05 per 0.1 s tick = 0.5 HP/s), `P_Mass` 1.0 (w3u h000, h003, h00J, h009, h010). The character differs by one unique ability and model (JASS `CharacterSelectId`):

| Character | unit | unique spell id | model (custom) | notes |
|---|---|---|---|---|
| Warlock | h000 | S001 | WarlockModel.mdl (size 0.92) | the default; gets Scourge as F spell |
| Heretic | h003 | S022 | HereticModel.mdl (0.96) | "Bane"/"bane orb", Link damage bonuses |
| Inquisitor | h00J | S023 | FireMageModel.mdl (0.95) | "Redemption" |
| Occultist | h009 | S051 | OccultistModel.mdl (0.98) | Ritual / Occultist missile |
| Enchanter | h010 | S100 | DruidOfTheTalonBloodElfHeroV1.01.mdl (0.92) | Enchant, damage via melee-style `4.8+0.6*LevelE` |

(Character details: 01-spells.md.) Mode flags `-vanilla` (Warlock only), `-ar` (random character), `-ap` (all pick) restrict the pool (JASS `ChatSettingsAction`).

## 5. Game modes (host chooses at startup)

`GAME_MODE_ROUND=1` (default, `-rd`), `DEATHMATCH=2` (`-dm`), `GHOST=3` = **Avatar** mode (`-av`), `KING=4` (`-kg`), `KOTH=5` (King of the Hill; set via the `-dm`-family; see 05 for what is uncertain). Details in 05-rounds-scoring-modes.md.

## 6. Scale of the content (for planning)

- 5 characters; roughly 50+ spell scripts (`S000`-`S128` ability ids on the unit; spell hotkey groups: D/R/T primary, C/E/Y secondary, F = character/Scourge-type spell, G = Fireball; see 01-spells.md)
- 29 item objects (w3t), 116 "research" objects (w3q: spell levels, masteries, item flags, tiers), 210 custom ability objects (w3a), 161 custom unit objects (w3u; mostly shop buttons and dummy missiles)
- 5 terrain tilesets, 10+ pillar layouts, 14 pillar models, ice patches, "donut" (centre-hole) option
- ~700 JASS functions; ~400 of them (lines 11131-16969) are spell logic

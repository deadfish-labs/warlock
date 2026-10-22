# 05 - Rounds, gold, points, game modes, win conditions, teams

Sources: JASS globals 514-591, 660-690, 1139-1157; `GameStart`, `GameStarting`, `ChatSettingsAction` (7916-8341), `LeagueSettings` (7591), `DuelTeams`, `TeamEdit`, `ShuffleTeams`, `RoundStart_2`, `RoundEnd`, `Round_CheckForWinner`, `Death`, `DeathmatchPeriodicAction`, `KOTHPeriodicAction`, `HiddenStart_2/HiddenEnd`, `KingStart_2/KingEnd/KingDeath/King_CheckForWinner`, `GameEndAction`, `InitMode_*`, `LeaveAction`; wts 1 ("default values" help text). Line refs are in `notes/data/war3map.j.txt`.

## 1. Players, slots and teams

- Up to **12 player slots** (ids 0-11). Team index 0-11 = real teams, **12 = no team/neutral** (used for the neutral "player 12", observers and empty slots). WC3 forces `P_Team[12]=12` for obstacles.
- Default: `FFA=true` (every player on his own team = team id equals slot id), `Shuffle=false`.
- Setup phase: host picks teams in a dialog (`InitTeamselectWindow`) or by chat `-teams <digits>` (`TeamEdit`: one digit/hex char per used slot, e.g. `-teams 0011` = slots 1-2 vs 3-4; chars map via `GetDecimalChar`, max team 11). The line `Selected teams: Team n: ...  Observers: ...` is printed to everyone. Alliances are applied in WC3 (`AdjustAlliance`) so allied units share vision/colour logic; **friendly fire is off** for spells (`P_Team[a] != P_Team[b]` checks in collision/damage), but allies still collide physically (`Obj_AllyColl`).
- **Shuffle mode** (`-shuffle` / `-sh`, `-sh max`, `-sh min`): teams are re-randomised at the start of *every* round (`ShuffleTeams`: up to 3 attempts to produce a different arrangement from the previous one; `max`/`min` = largest/smallest teams?, uncertain). In Shuffle mode points are tracked **per player** (`TeamPoints[slot]`), not per team, and the match ends as FFA.
- `-mgl`: forces 5 vs 5 (slots 0-4 vs 5-9, slots 10/11 must be empty), 10 rounds. `-cm`: "competitive mode" 11 rounds, shop 15 s/15 s, kill/assist points 0, win point 1, no suicide penalty. `-league`: "no gold rewards - ice off" preset: `KillPoint=1, KillGold=0, AssistGold=0, WinPoint=1, WinGold=0, DmgGold=0, obstacles random(0), RoundAddedGold=10, StartingGold=20, ShopTime 30, ShrinkTime 15, map=1 tileset fixed, all multipliers 1.00, regen 0.05` (JASS `LeagueSettings`).
- **Duel (ranked-style 1v1 / 2v2)**: `Duel = not DuelDisabled` in Round mode: with exactly 2 players (or 2v2 via `DuelTeams`) the match is a duel: **first to 6 points wins** (RoundEnd: `TeamPoints[i] >= 6` -> `DuelEnd`; in `Shuffle` duels the rule is "leader's lead is larger than rounds remaining"). `-ranked`/`-noranked` toggle `DuelDisabled`. Tick idle detection (`PlayerSetOBS`) is disabled in duel.
- Observers: team 12 players; `-unobs`, `-unobsid <n>` (host) bring an observer into the game between rounds (`UnobsQueue`); idle players (zero actions in a 30 s window, FFA, more than 2 players, shop time or Deathmatch/KOTH) are auto-moved to observer by `Tick` -> `PlayerSetOBS`. `PlayerReturnToGame` re-adds.
- **Leaving** (`LeaveAction`): if mid-round alive -> `Death`; objects removed; scoreboard row greyed with "-"; `PlayerCount--` (shrink formula follows). The game continues.

## 2. Game modes (`GameMode`, chosen by host chat command before start)

| const | cmd | summary |
|---|---|---|
| 1 ROUND | `-rd` (default) | classic last-team-standing rounds with shop phases; arena shrinks. `MaxRounds` default 11 |
| 2 DEATHMATCH | `-dm` | continuous respawning free-for-all, no shrink, point (kill) limit or time limit |
| 3 GHOST ("Avatar") | `-av` | one player per round is the powerful Avatar; everyone else fights him |
| 4 KING | `-kg [f]` | team mode, each team has a King with a crown; `f`/`fixed` = fixed kings |
| 5 KOTH | via `-dm` family (`GAME_MODE_KOTH` in 11 places) | King of the Hill. Code handles it like Deathmatch (respawns, no shrink, time limit allowed, periodic shuffle) - the actual hill/zone scoring rule was **not found** in the script. Uncertain: may have been disabled/unfinished. `ChatSettingsAction` still lists it |

Host start commands: `-<number>` = number of rounds (Round/Avatar/King) or point limit (DM/KOTH); `-<number>m` = time limit in minutes (DM/KOTH only) -> `TimeLimit=true`, game ends after `MaxRounds*60` s (`InitMode_Deathmatch`). `-test <n>` = test mode (1000 gold, pillar HP 100, shorter timers, no AFK kick); `-0` rounds = unlimited (1000).

### 2.1 Round mode (the main mode)
As described in 00-overview §3. Key numbers:
- shop first phase 40 s (`C_ShopTimeDurationFirst`), later phases 30 s (`C_ShopTimeDuration`); both host-configurable (`-c4`, `-c5`).
- Dead players spectate at the shop (a Merchant unit at (maxX, maxY)), cannot rejoin the fight until the next round. Message "You have died and will respawn when this round ends." (shown once).
- Round ends instantly (`Round_CheckForWinner` from `Death`) when <=1 team alive; the camera/screen fades, loser sees results.
- Winner: at `CurrentRound >= MaxRounds`: `GameEndAction`: highest team points wins (`TeamPoints[team]`); ties: "Draw! One more round to decide the battle" (`MaxRounds++`; for DM: "One more kill", `MaxRounds = highest_score+1`). Victory/Defeat screen and "EpicVictory"; stats sent (`SendStats` to `WarlockLeague`, see 06). "Game will exit in 3 minutes."
- Players who do not qualify for tie-break rounds get "You did not qualify for the final round!" and become spectators.

### 2.2 Deathmatch
- Everyone spawns once (ring, or custom revive spots), 20 s (first shop/2) pause (`PauseAll` after `C_ShopTimeDurationFirst/2`) while shopping, then free-for-all.
- On death (`Death`, branch DM/KOTH): victim `TeamPoints -= 1`(always, "suicide" or killed; code uses a literal 1, not the penalty constant), killer `+C_KillPointReward`, assist `+C_AssistPointReward`; gold `+C_KillGoldReward` to killer. **Auto-respawn after `C_ReviveTime = 10` s** at a random spot inside the current arena (`Deathmatch_RevivePlayer`: `rand(+-size*64)`), with HP reset and immortality for 4 s.
- Every `C_ShopTimeDuration*6` = 180 s (`DeathmatchPeriodicAction`): white screen flash, re-shuffle if `Shuffle`, new terrain/lava theme, `CurrentRound++`, tokens credited. Gold arrives through kills/assists/time (`C_RoundAddedGold`-style gold per period is added in the periodic action? uncertain; the visible code credits tokens only). Shop is reachable any time with **ESC** (toggles between shop and character: "Press ESC to toggle between the shop and your character").
- Win: first to the point limit (`TeamPoints[...] >= MaxRounds` in `Deathmatch_CheckForWinner`) or at the time limit.

### 2.3 Avatar (GHOST) mode
- `HiddenStart_2` chooses the Avatar: highest `AvatarPoints` from previous rounds; if none (round 0 or invalid) a random living player. Message "X is the Avatar".
- Avatar: own team (team 1), all others team 0; Avatar gets: size/model x1.45 (`Obj_Size +20`), mass +7, damage dealt x1.5, curse duration x0.7 (`LastLevelTimeTwist`), regeneration `+PlayerCount/2 * base`, knockback taken divided by `PlayerCount/1.2`, move speed +60, max HP via ability `A005` at level `PlayerCount+12`, his F-spell replaced by `S020`, his fireball replaced by Iceball (`FireballCast` checks `CasterID==HiddenPlayer` -> `IceCast`). Everyone shops as usual between rounds.
- Round ends when the Avatar dies or all others die (`AliveCount <= 1`). Kill message "- the Avatar has been killed!".
- Scoring (`HiddenEnd`): `AvatarPoints[i] += DmgToPlayer[Avatar][i]` (damage dealt *to the Avatar* by player i; if i is dead at end: half of it); `TeamPoints[i] += floor(AvatarPoints[i]/20)`; the Avatar gets `floor(total damage dealt to others /20)`; ranking printed for 20 s ("<points> : <name>"). Game ends at `MaxRounds`. Next Avatar = top `AvatarPoints` holder.

### 2.4 King mode
- `DefineKing` picks one King per team (random member; none for single-member Shuffle teams; `FixedKing` keeps the same). King wears `crown.mdl` + CommandAura; his F-spell becomes `S021` (King spell).
- When a King dies (`Death`, GAME_MODE_KING): `KingDeath`: every living member of the king's team loses **50 % of current HP** (and gains DP equal to it), the sum `TotalDmg` is converted to healing for the killer's team (`0.5 * TotalDmg / alive killers` each, via homing heal missiles); killer and top damage dealer receive `C_KillGoldReward`. Round continues until one team remains (`King_CheckForWinner`; same scoring as Round mode). Uncertain: whether the dead King's team continues playing or is eliminated - the code only removes the king and lets `King_CheckForWinner` run.

### 2.5 Other toggles
- `-ice` ice patches, `-circle`/`-square` (+ `donut`), `-ymas` (Christmas: theme 3, presents, snowman pillars, hats `-hat`), `-ymasmode` (shards are presents), `-dfree` (disable `M034`, a cosmetic/donation shop item), `-no reward`, `-gold <n>` start gold (6 digits max), `-c1..-c17` constants menu: 1 lava dmg/10, 2/3 per-player multiplier-style constants (JASS 8280-8320, uncertain), 4 shop time, 5 first shop time, 6 shrink time per player, 7 terrain id, 8 obstacle list 0-11, 9 (player stat), 10 kill points, 11 assist points, 12 kill gold, 13 assist gold, 14 win points, 15 win gold, 16 damage gold, 17 round added gold.
- Bans: `-ban`, `-banitem`, `-banduration`, `-banrange`, `-banlifesteal`, `-banjackofalltrades`, `-bancharacter` plus ban dialog windows (`Ban_*`): host can ban spells (by id), items, masteries (R00I Lifesteal?, R00Y, R00D, R01E "any mastery"), characters, hotkey columns. Banned items/spells are hidden/unavailable in the shop.

## 3. Economy and scoring numbers (defaults)

| constant | value | source |
|---|---|---|
| `C_StartingGold` | 20 (30 in a specific preset around JASS 9904, 1000 in test mode) | JASS 527, 9904, 8028 |
| `C_RoundAddedGold` | +10 every round end (everyone incl. dead) | JASS 526, `RoundEnd` |
| `C_KillGoldReward`, `C_AssistGoldReward`, `C_WinGoldReward`, `C_DmgGoldReward` | 0 each in code (the quest help text wts 1 *describes* defaults of win 2, damage-dealer 1, pure kill 2, kill 1, assist 1 gold, labelled "default values, some may be changed in startup"; the shipped code defaults are 0, so these numbers are probably for another config/league - uncertain) | JASS 517-523 vs wts 1 |
| `C_KillPointReward` / `C_AssistPointReward` | 1 / 1 (wts 1 says 1 point per kill) | JASS 517-518 |
| `C_WinPointReward` | 1 (wts 1 says 2 per victory - mismatch) | JASS 521 |
| `C_SuicidePenalty` | 1 (FFA only, -1 point on lava suicide) | `Death` |
| `C_ReviveTime` | 10 s (Deathmatch) | JASS 528 |
| `C_ImmortTime` | 4 s | JASS 529 |
| `MaxRounds` | 11 (10 in `-mgl`) | JASS 1139 |
| Kill assist | the player who dealt the highest damage to the victim (`DmgToPlayer`) if different from the killer; gets `+Assist` points, `+1` assist stat and `C_KillGold` (in some branches `C_AssistGold`) | `Death` |
| Killing spree | `MinKillingSpree=3`, `MaxKillingSpree=10`, `DoubleKillTime=9` s; first blood; "Ludicrous Kill" (kill someone who dealt 0 damage though active); multi-kill ladder (double, triple, ... mega, ultra, monster) and spree ladder (killing spree, dominating, ownage, rampage, unstoppable, whicked sick, godlike, holy shit...) with sounds and chat text | `Death`, globals 660-690 |
| Gold during game | shown as WC3 gold; **tokens** = lumber = permanent points mirrored for cosmetic purchases (`P_TokensEarned`) | `RoundEnd` |
| Points display | multiboard column "points" (green for current leader, white else) | `BoardMarkTop` |

Because of the host constants, a remake should model all of these as a `ruleset` object (see 07) and ship at least the "default", "league/competitive" (`-league`, `-cm`) and "mgl" presets.

## 4. Ending conditions summary

- Round mode: end of round when one team (or nobody = draw) is alive; game ends after `MaxRounds` rounds or earlier in duel (first to 6) / competitive early-win.
- Deathmatch/KOTH: point limit or time limit.
- Avatar/King: `MaxRounds`.
- Tie at the top: extra round / kill until a single leader.
- Game exit: 3-minute countdown after the end; stats are sent to WarlockLeague (a gamecache-based export for an external ranking site; replay-protection with `AckermannProtection`, `LoadHashes`, RSA - ignore in the remake, but design a stats export).

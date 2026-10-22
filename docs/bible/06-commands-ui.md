# 06 - Chat commands, HUD, controls, what players see

Sources: JASS `ChatCommand` (20553-20872), `ChatSettingsAction` (7916-8341), `InitChatSettings`, `Tips__Init` (3498-3565), `Multiboard*` (2606-2810), `Tick` (10092), `BoardItemSet`, `GameStart`/`GameStarting`, `Death` (messages), wts 0-9 and the quest texts, w3a `ahky` hotkeys. Line refs in `notes/data/war3map.j.txt`.

IMPORTANT: wts string 2 (the in-map "Commands" quest page) lists `-cam`, **`-splitter`**, **`-oath [numbers]`**, `-stats`, `-fbc [color]` (colours pink/blue/green/red). The script of this build (1.06n) has **no** `-splitter` or `-oath` handler (only the global `Oath_TempObject` remains) - Splitter is now a click-toggle on the spell button (`AbilityToggle`, "toggle" button under the spell, see Tips). So the quest text is stale and describes an *older* build (probably the 0.99c family seen in the video: Splitter mode and an "Oath" sensitivity setting were then chat commands). Behaviour of "oath" is unknown from this data (uncertain; probably a click/targeting sensitivity or Oath-of-something spell setting; ask the owner / check the video).

## 1. Chat commands available to all players (in game)

| command | effect (JASS) |
|---|---|
| `-cam <n>` / `-cam<n>` | Camera Z offset (`CameraHeight[player]`), standard 0; reapplied every second (`Tick`) along with distance 1650, angle 304. Prints "Camera zoom set to n" |
| `-stats` | prints for 10 s: Movement Speed, Regeneration (10x per-tick value /s), Knockback taken %, Damage taken %, Life Steal weakness %, Buff duration %, Curse duration %, Range modify, Mass (kg) |
| `-autoselect` | toggle: re-selects your character automatically when deselected |
| `-apm` | show actions-per-minute for all players (APM tracked by `APMAdd`/`APMDetect`) |
| `-fbc pink|blue|green|red`, `-fbc #RRGGBB`/`r g b`, `-fbc original` | fireball colour (stored in `P_FBColor`) |
| `-cf` / `-cfa` | combined camera+fireball-colour shortcut (`-cfa` also toggles autoselect) |
| `-sell <item>` | sell an item (alternative to the shop's Sell button) |
| `-tutorial` | enables the step-by-step beginner guide (10 steps, see §4); `-hint`/`-hint <n>` shows random/chosen hint |
| `-title` / `-untitle` | show/hide your earned title (coloured word before your name; awarded for tournaments/contributions; see Tips) |
| `-n` | clear the text log |
| `-hat` | Christmas hat (only when `-ymas`) |
| `-quit local` | leave the game on your side only |
| `-unpause` | resume after a pause (if one team left) |
| `-debug` | local filter reset |

Host-only in game (slot 0 / player 1): `-teamshuffle`, `-teams <digits>` (during shop phase or DM, not Avatar), `-unobs <name>` / `-unobsid <n>` (return an observer), `-terrain <n>` (test mode only), `-bot|-war|-her|-inq|-occ|-enc <slot>` (test mode: add a bot of a character on a slot; `-war` Warlock), `-quit player <n>` (test mode).

## 2. Host setup commands (before the game starts; processed by `ChatSettingsAction`)

Mode and structure: `-rd|-round`, `-dm|-deathmatch`, `-av|-avatar`, `-kg|-king [f|fixed]`, `<n>` rounds / point limit, `<n>m` minutes (DM/KOTH), `-shuffle|-sh [max|min]`, `-teams`, `-test <n>`, `-mgl`, `-cm`, `-league`, `-ranked`, `-noranked`, `-rse`, `-rsa` (random spell each / all? - `RandomSpellEach` / `RandomSpellAll`: random spells are given to each/all players, uncertain details), `-vanilla` (Warlock only), `-ar` (random character), `-ap` (all pick), `-ice`, `-circle`, `-square`, `-circle donut`, `-square donut`, `-ymas`, `-ymasmode`, `-dfree`, `-no reward`, `-gold <n>`, `-c<n> <value>` constants (see 05), `-ban*` family (items, spells, masteries, characters, hotkeys, duration/range/lifesteal/jack-of-all-trades masteries), `-#CTreset` and a long `-t...` custom-terrain code (> 100 chars; `CTConversion`). Dialogs mirror these as buttons (Auto-select window, Team-select window, Player-select window, Ban dialogs: character/mastery/item/spell/hotkey/category).

Startup text printed to all (`InitChatSettings`): lists `-tutorial`, `-cam ###`, `-autoselect`, `-stats`, `-apm`, `-fbc ### ### ###`.

## 3. Controls and in-world UI

- Camera: fixed top-down-ish (distance 1650, 304 deg angle, no rotation), fog off; selected unit = your warlock; ESC selects the shop unit (and back) (`SelectShop`; "Press ESC to select the shop in between rounds or while waiting to respawn").
- Move: WC3 right-click move; stop (`S`) cancels most spells (Scourge channel, fake casts).
- Spell hotkeys by column (w3a `ahky`, spell menu structure): **G** Fireball (always owned, `S000`), **F** character spell (`S001` Scourge for Warlock; `S022` Bane Heretic; `S023` Redemption Inquisitor; `S051` Ritual Occultist; `S100` Enchant Enchanter; `S021` Pious = King's; `S020` Cataclysm = Avatar's), **D** (Bolt, Arc, Boomerang, Homing, Lightning), **R** (Teleport/Thrust/Shield... `M024` "R Spells" menu - see 01), **T** (Fire Spray, Dagger, Bouncer, Drain), **C** (Ethereal Form, Dome, Vector, Mirror, Time Shift, Rush), **E** (Burst, Repeater, Wind Walk, Splitter, Meteor), **Y** (Lock On, Control, Link, Gravity, Disable/Entangle). Spells with a second form show a small toggle icon below them (`M0xx` toggle abilities, `AbilityToggle`): e.g. Link <-> Hook/Induction, Drain <-> Slow/Weakness, Fire Spray <-> Stream/Cluster, Bouncer <-> Bounce/Recharge, Arc direction CW/CCW, Teleport <-> Multiport, Thrust <-> Strike/Phoenix, Wind Walk <-> Charge/Invisibility, Splitter <-> Target, Time Shift (re-click), Rush.
- Several spells have a **second activation** while active (Link hook, Time Shift recall, Teleport/Multiport second click) (Tips).
- Spell targeting: `TargetPositionValid` clamps the cast point; missiles spawn at the caster and fly at the target direction; range often depends on masteries.
- Merchant (shop) units: `u000` "Merchant" with button menus: Primary Spells 1/2, Secondary Spells 1/2, Items, Upgrades, Masteries, Stone of Jordan Ring, Sell, Cosmetics (see 02). Hotkey Q opens Merchant, W Items etc. (`M032`, `M033`, `M023`).
- Floating combat text: damage (gold), heal (green), self damage (blue), "Whirlwind!" (orange), "Lock On" (red), stat messages at hits; killer-assist texts. Units show HP (life bar), DP (mana bar), buff and curse special effects attached to the unit.
- Death/kill feed in chat with colours: "X killed Y (assist) - First Blood - double kill... - Y committed suicide".
- Announcer sounds (Unreal Tournament-style): first blood, double/mega/ultra/monster kill, killing spree..., Hattrick (3 Scourge hits), Vampire (Mask of Death multi-hit), Whirlwind, Lock On, Pancake (squashed by pillar?), Burnout, Silencer, Denied, Last Second Save, Ludicrous Kill. In the remake: need original SFX, not these files.
- Result screens: Victory / Defeat / "Epic Victory" with a camera rotating around the winner (`RotateCameraAroundWinner`).

## 4. Scoreboard (multiboard) and timers

- Multiboard (top-right), 4 columns: **Player** (coloured name; dimmed once dead/left), **Kills**, **Points** (the leader's points are drawn green), **Dmg** (damage dealt this round, reset at round start). Title: `Round n/MaxRounds` (Round/Avatar/King), `First to 6` (duel), `Point limit = N` (DM/KOTH), `Time left` (timed DM), left/right paging `SwapMultiboard_Left/Right` for the second board (stats/more players).
- Timer dialog "Shop time:" (the countdown during the shop phase).
- The day/night clock at the top centre of the HUD doubles as "next arena shrink" clock (see 03).
- Resource bar: gold (spendable), lumber = tokens.
- Text messages: round start/end, "X has won the round!", "X has dealt the most damage this round (n)", "Team N has won the game!", "Game will exit in 3 minutes."

## 5. Tutorial and hints (text a remake can adapt)

10-step tutorial (`Tips__NewTutorialStep`): 1 Welcome (select Merchant, buy an R spell from Primary Spells), 2 second spell (D or T), 3 third spell, 4 fourth (C from Secondary), 5 save money, 6 fifth (E), 7 sixth: Y spell / items (boots if unsure) / upgrades / masteries, 8 toggles, 9 conclusion (standard game: 11 rounds, 20 starting gold, 10 per round), 10 questions (hotkey changer and terrain editor exist, www.WarlockBrawl.com). Plus ~45 random hints (mechanics: stop-cancel, mass, combos like Lightning hitting your Fireball for AoE, "pre-Scourge", Whirlwind/Hattrick/Vampire/Burnout, Shield does not block AoE, Time Shift stutter, pillars reflect Splitter, saving gold, tokens/titles).

## 6. Titles, cosmetics, tokens, league (meta-game; low priority)

- **Titles** (`Title_*`, wts): coloured word before a player's name; encoded per account name check (hash/RSA protected) - awarded for tournaments/contributions.
- **Tokens** (lumber): earned = points; used to buy **cosmetics** (head, body, back, wrists, staff, fatality finishers; "Free" and "Paid" sets, `P_COSMETICS_*`; `-dfree` disables a free-tier ability) and shown as attachments (`Attachment System`, `HideAttachments`). Fatalities = custom death effects (`P_COSMETICS_FATALITY`, default `j000`).
- **WarlockLeague** library: collects fireball cast/hit counts, APM, score, kills/deaths/assists per player and writes them to a game cache for an external site (`SendStats`). Anti-cheat: `AntiJassProtection`, `AckermannProtection`, RSA hash of settings (`Desync`). A remake should have its own accounts/matchmaking/stat service instead.

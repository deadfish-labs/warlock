# 02 - Items, Shop, Spell Purchasing (map data is Warlock 1.06n, not 0.99c)

Sources: `w3t` (items I000-I00S), `w3u` (shop units h0xx/u0xx, cosmetic units i/is/j), `w3q` (researches), `w3a` (abilities) and the JASS in `war3map.j.txt`
(function names and line numbers below refer to that file). Everything is plain data; nothing was executed.
"uncertain" marks anything I could not prove from the data.

## 0. Quick mental model (read this first)

* The **item "price" shown in the shop is NOT `igol` of the w3t item.** The shop buttons are *units* `h001..h00H` (trained at shop building `u003`
  "Items"); their `ugol` is the price. The real inventory items `I000..I00S` are created by script (`UnitAddItemById`) in `BuyItem`.
  The w3t `igol` values (6, 20, 10, 2, 3, 0...) are leftovers and are not used by the logic (uncertain what they did originally; ignore them).
* **One purchase = one click on a shop button; the price is the same for every level** (an "upgrade" is the same button clicked again and costs the
  full unit price again). Example: Boots button costs 5 gold; Boots level 1 costs 5, level 2 costs another 5.
* Purchase flow (JASS `InitTrig_Items` ~18165): `EVENT_PLAYER_UNIT_TRAIN_START` on any player 0..11 -> `BuyItemAction` -> issues order `0xD0008` (cancel) on the shop
  unit and calls `BuyItem(GetTrainedUnitType(), playerId)`. The engine has already deducted the unit's gold when training starts. `BuyItem` gives
  the money back via `RefundIf` when the purchase is invalid (see 3.3). Remake: simply "deduct price, and undo the deduction when a rule blocks it".
  How the cancel order interacts with the engine refund is uncertain, but the intended net effect is clear: price deducted only on success.
* Inventory: 6 slots max, **but only 2/4/6 are usable**: ability `S128` (Inventory) level L allows `2*L` items (`no_inventory = 2*L <= itemcount`, BuyItem ~17857).
  Level 1 at start (uncertain, not set in script; the w3a `inv1` table gives 4 at lvl 2 and 6 at lvl 3). Research `R000` raises it (see 5.2).
* Selling is only allowed during shop time (`SellItem`: `if not ShopTime ... return`, also disabled in Deathmatch and KOTH game modes).
* **Items are bought during "shop time" between rounds**, but `BuyItem` itself has no `ShopTime` check, so a dead player (who gets a shop unit
  placed at the map corner and "will respawn when this round ends", `Death` ~4529) can buy during the round (but not sell).

## 1. Shop flow between rounds

| What | Value | Source |
|---|---|---|
| Starting gold (default) | **20** | `C_StartingGold=20` (line 527, also `InitSettings` ~7602); a few game modes set 30 (lines 9904-9911) or 1000 (8028); host command `-gold #####` |
| Gold per round | **+10** at the end of every round (all slots with `P_SlotUsed`) | `C_RoundAddedGold=10`, `RoundEnd` ~19503 and ~19590; Deathmatch/KOTH: +10 on each periodic tick (`DeathmatchPeriodicAction` ~19791) |
| Kill / assist / win / damage gold | all **0** by default (`C_KillGoldReward`, `C_AssistGoldReward`, `C_WinGoldReward`, `C_DmgGoldReward`) - exist as host settings only | lines 520-525, 4131, 4160, 4573 |
| Tokens (lumber) | at round end each player's tokens (PLAYER_STATE_RESOURCE_LUMBER) rise by the points they have gained (`TeamPoints - P_TokensEarned`); tokens only buy paid cosmetics | `RoundEnd` ~19495 |
| First shop time (before round 1) | **40 s** (`C_ShopTimeDurationFirst=40`) | line 533, `InitMode_Round` ~19557 |
| Shop time between rounds | **30 s** (`C_ShopTimeDuration=30`); competitive mode `-cm` sets both to 15 s | 532, 7614, 8102 |
| Countdown UI | a `TimerDialog` titled "Shop time:" on `ShopTimer`; when it expires `RoundStart` -> `RoundStart_2` runs (fade out 3 s, teleport to arena, `ShopTime=false`, immortal period) | 19259-19400, 19516 |
| Hint text | "Press ESC to select the shop inbetween rounds or while waiting to respawn" | 19557 / 7806 |

Round end (`RoundEnd` ~19402): `ShopTime=true`, lava off, every living player is moved back to their shop pad (x = 128*(i-6)-64, shop at y=64, warlock at y=-64, facing 270),
a fresh `u000` Merchant is created for each player (old shop removed), the warlock is selected-in-shop, gold +10, the 30 s `ShopTimer` starts.
Dead players: shop unit `u000` is created at the map corner (maxX,maxY), unpathable, vertex alpha 1.

Shop selection: ESC (trigger `SelectShop`, `ShopSelectAction` ~7202) toggles the **local player's** selection between their shop unit and their warlock
(if the shop is selected and the player is alive it selects the warlock, otherwise the shop). Disabled for 1.4 s at round start.

## 2. Shop UI structure (what the player sees)

Each player has a private shop unit `P_Shop[id]` (an Acolyte model, invulnerable `Avul`). It is a "menu by unit swap": the ability buttons on
`u000` select/enter the sub-shops. After most purchases the script destroys the shop and creates a fresh `u000` (main menu) - so the UI snaps
back to the root menu after buying a **spell** or **upgrade**; item purchases do not reset it (only the Jordan ring creates `u004`).

Main Merchant `u000` ("Merchant"): buttons (w3a M0xx): `M022` D Spells (hotkey D), `M021` T Spells, `M024` R Spells, `M019` E Spells, `M020` C Spells,
`M018` Y Spells, `M023` Upgrades (Q), `M033` Items (W), `M034` Sell Items (Y), `M036` Stone of Jordan Ring (G, only visible after buying the ring),
`M016/M015` Free/Paid Cosmetics, toggles `M037-M042` (see 8). `M032` "Merchant" (hotkey Q) is the Back button present on every sub-shop.

Sub-shop units (button grids are `ubpx/ubpy` of the research/item definitions):

| Unit | Name | Sells |
|---|---|---|
| u003 | Items | trains the 14 valuables `h004,h00H,h006,h00E,h00C,h008,h007,h00B,h002,h00A,h005,h00G,h00F,h001` + all cosmetic units; research `R000` (inventory); abilities `M046` Page Previous (hotkey T) and `M045` Page Next (hotkey Y) |
| u005 | Sell | trains `s000..s005` = "Sell 1..6" (sell item from inventory slot N; same as chat `-sell N`; tooltip "Sell item from slot N") |
| u006 | Upgrades | researches: all spell **upgrade** researches, `R002` Fireball, masteries `R00Y R00D R00I R01E`, (R017 display) |
| u009 / u001 / u010 / u012 / u011 / u013 | D / R / T / E / C / Y Spells | the spell "learn" researches of that hotkey (+ the "Random X Spell" research) |
| u007 / u008 / u002 | Primary Spells 2 / Secondary Spells 2 / Secondary Spells 1 | extra spellbooks (changelog 1.00: "2 new spellbooks added to shop ... yet to be decided whether on by default or only by setup command") (uncertain which are enabled by default) |
| u004 | Stone of Jordan Ring | the `T000..T064` pseudo-researches (+2 upgrades for one chosen spell), see 6 |
| u014 / u015 | Free / Paid Cosmetics | cosmetic category browsers (Head, Body, Back, Staff, Wrists, Fatalities) = units u00A..u00J |

Inventory page system (`GotoItemPage` ~10468): the valuables are held in `AllItemUIDs[]` (set up in `Init_SetItemResearchUnits` ~17549); 8 buttons per page, via
`SetPlayerTechMaxAllowed(player, unit, 1/0)` (only buttons with tech max 1 are visible). Valuables order = index 0..13:
page 1: `h00E` Pocket Watch, `h00C` Lava Treads, `h007` Blood Sword, `h004` Stone of Jordan Ring, `h00B` Boots, `h002` Cape, `h00A` Pendant, `h005` Helmet;
page 2: `h006` Staff of Fireball, `h008` Aegis, `h00G` Warden, `h00F` Cursed Pendant, `h001` Mask of Death, `h00H` Scroll of Fireball (indices 14-15 are empty).
Cosmetics are indices 16.. (see 7). `M045`/`M046` ability levels are set so the tooltips read "Next Page (n)" / "Previous Page (n)" / "Last Page"; wrapping
past the last page goes to page 0 and "previous" on page 0 jumps to the last page. `P_ItemShopPage[id]` remembers the page (and `id+12` the page of the other shop type).
Tech-max for all cosmetic units is set to 0 at init (`Init_SetItemResearchUnits` loops i2 from 8 to size) so only valuables page 0 is visible at first.
Items are "banned" via host command `-banitem <index...>` (`ItemBan` ~7286 / `BanThisItem` 7260): sets TechMaxAllowed of that button to 0 for all 12 players; chat reply "<item> banned/unbanned" (index<8 unbans to 1 so visible page-1 items return). Host-only; for the remake this is a lobby setting.

## 3. Items (valuables): the 14 shop buttons

### 3.1 Price / sell table (price = `ugol` of the `h` unit; sell value = JASS `SellItem`)

| Button | Item name | Price per click | Item ids by level | Sell value (JASS) | Allitems index |
|---|---|---|---|---|---|
| h00B | Boots of Speed ("Boots") | 5 | I000 (lvl1, +10 ms) -> I008 (lvl2, +20 ms total) | 4 / 8 | 0 / 1 |
| h002 | Cape ("Warlock's amulet") | 4 | I005 -> I009 -> I003 | 2 / 4 / 6 | 9 / 10 / 11 |
| h00A | Pendant | 2 | I00B -> I002 -> I00C | 1 / 2 / 3 | 3 / 4 / 5 |
| h005 | Helmet | 6 | I001 -> I006 -> I00A | 5 / 10 / 15 | 6 / 7 / 8 |
| h006 | Staff of Fireball | 3 | I00D | 1 | 13 |
| h00H | Scroll of Fireball | 2 | I00Q | 1 | 27 |
| h001 | Mask of Death | 4 | I004 (or combined I00O) | 3 | 12 |
| h007 | Blood Sword | 7 | I00F (lvl1) -> I00G (lvl2); with Mask: I00O | 6 / 12; I00O cannot be sold | 15 / 16 |
| h008 | Aegis | 9 | I00H | 8 | 17 |
| h004 | Stone of Jordan Ring | 1 | I00E | cannot be sold | 14 (+18 = "unused ring") |
| h00C | Lava Treads | 6 | I00J (the only one ever created by BuyItem; I00K, I00L are unused tiers) | 5 | 19 |
| h00E | Pocket Watch | 4 | I00M | 3 | 22 |
| h00F | Cursed Pendant | 6 | I00I | 2 | 26 |
| h00G | Warden | 2 | I00N -> I00P -> I00R -> I00S | 1 / 2 / 3 / 4 | 24 / 25 / 28 / 29 |

(Sell prices are refunded as gold by `SellItem` = about 50-80 % of what was paid; the sell text in the w3t tooltips agrees: "Sell value: N".)
Unit tooltips (`h0xx utub`) list per-level effects in one block; the per-item w3t `utub` list the effect of that specific level. A "level" can
only be bought in order (each click upgrades the current copy). Selling a level frees that level only; e.g. selling a lvl 2 Helmet gives 10 gold
and removes the helmet entirely (it is not downgraded).

### 3.2 Item details (exact numbers)

Units: `P_Mass` base **1.00 = 100 kg** (so 0.05 = 5 kg); `P_Regeneration` base **0.05 per 0.1 s tick = 0.5 hp/s**, shown per second in tooltips (item value x10, e.g. 0.015 -> 0.15 hp/s);
`P_KnockbackMultiplier` base 1.0 (lower = less knockback); `P_LifeStealMultiplier` base 1.0 (the *victim's* weakness: life steal gained by an attacker is multiplied by the victim's value;
higher = victim is more exposed); `P_TimeDec` = duration multiplier for curses/debuffs received; `P_TimeInc` = duration multiplier for buffs/curses cast.

**Boots (h00B; I000, I008; I007 exists but is not buyable)**
- I000: +10 movement speed, -5 kg mass (mass is only reduced once, at the first purchase), upgradeable once. I008: +20 ms total (upgrade adds +10 ms), still -5 kg. (w3t I000/I008 utub; JASS h00B branch sets `SetObjectMS +10` and `P_Mass -0.05` only when no boots yet.)
- I007 "Boots of Speed" (+40 ms, -5 kg, sell 12) is only referenced in `SellItem`; BuyItem never creates it (uncertain: cut content from a 3-level chain).
- Does not stack with Lava Treads (both buttons refuse: "Item does not stack with Boots/Lava Treads").
- Sell: refunds 4 / 8, ms -10 / -20, mass +0.05.
- Real world: tooltip says "can be upgraded once" for I000 and nothing for I008.

**Cape (h002; I005 -> I009 -> I003)**: life regeneration +0.15 / +0.25 / +0.35 hp/s (total, by level; JASS adds 0.015 then +0.010 then +0.010 per tick). Max level 3 ("Item is already at max level" otherwise). Sell 2 / 4 / 6.

**Pendant (h00A; I00B -> I002 -> I00C)**: max HP +5 / +10 / +15 (carried by hidden item abilities A007 = +5, A004 = +10, A00H = +15 via Ilif) and the wearer's life steal weakness +5 % / +10 % / +15 %
(`P_LifeStealMultiplier += 0.05` per level); level 3 also adds +0.10 hp/s life regen (`P_Regeneration += 0.01`). The unit tooltip says "Item is stackable": the code first upgrades an existing I00B, then I002 (checks the lower level first), and only if neither exists creates a new I00B, so a player can hold a full I00C plus a fresh lower-level chain (inventory allowing). The per-level boolean array `Allitems` records only one copy of each level, so holding two copies of the same level is not tracked properly (uncertain/bug, don't replicate). Sell 1 / 2 / 3.

**Helmet (h005; I001 -> I006 -> I00A)**: knockback taken x0.84 / x0.76 / x0.68 (i.e. -16 % / -24 % / -32 %), mass +10 / +20 / +30 kg, **missile range -12 % / -24 % / -36 %** (implemented as `LevelRanged -= 1/2/3`; range multiplier = `1 + 0.12 * LevelRanged`, `RangePerLvl=0.12`, so helmet levels cancel Range Mastery). "Helmet does not stack" (w3t text) = there is only one helmet that is upgraded in place; max level 3. The helmet also lowers the cap for Range Mastery by its level (max mastery 6 - helmet level) and counts as virtual mastery levels in the knockback bonus (see 5.3). Sell 5 / 10 / 15 (reverses mass, LevelRanged and KB).

**Staff of Fireball (h006; I00D)**: adds ability A009 and raises the Fireball (G spell `S000`) by +12 ability levels (i.e. switches to a burning variant, levels 13-24 of S000). Fireball becomes burn-over-time: base damage **6.2 + 0.4 x upgrades**, burning damage **1 + 0.5 x consecutive hit**, burning is strengthened by consecutive hits (`Consecutive[id]` set to 1 on purchase, counts successive fireball hits, used also by the freeze duration `3.0 + 0.5*Consecutive` at ~11305). Buying again: "Item is already at max level". Sell 1: removes A009, `Consecutive=0`, `S000` level -12. (Fireball level cap = 11, +2 by Jordan ring; see 5.1.)

**Scroll of Fireball (h00H; I00Q)** (named "Warlock's pendant" in w3t): fireballs bounce off pillars; each bounce increases velocity and damage by 1. Only a flag (`Allitems[27]`), no ability is added by BuyItem (effect lives in the fireball code ~11223 `Fukki_Gold_Rate3` / `ScrollFBBonus`, constant 1 at line 534; uncertain). Sell 1.

**Mask of Death (h001; I004)**: Scourge / Bane (and orb) / Redemption (the character's F spell) gain **+20 % life steal and +10 % damage point recovery**
(`P_ScourgeLSBonus += 2`; in the F spell code `LevelMelee` is temporarily raised by `P_ScourgeLSBonus`: each melee level = +10 % life steal and +5 % DP recovery) and the wearer's weakness to life steal is **reduced by 25 %** (`P_LifeStealMultiplier -= 0.25`). Sell 3.

**Blood Sword (h007; I00F -> I00G)**: raises the character's F spell (Scourge, ability index `SpellsAcquired[+7]`) level by 1 per sword level (`LevelF`), plus the attachment ability A010 (lvl1) / A011 or A013 (lvl2) for the melee model. Scourge damage = 10 + `LevelF` (line 16100), melee hit 5 + `LevelF`.
Tooltip numbers by class: Warlock & Inquisitor (I00F also lists Enchanter): lvl1 **11 dmg, 2 DP recovery**; lvl2 **12 dmg, 3 DP recovery**. Heretic: lvl1 **4.5**, lvl2 **6.0 bane orb damage**. Occultist: lvl1 **7.7 dmg, 3.0 hp** (w3t I00F says 8.7; the shop unit text and I00G say 7.7: use 7.7, uncertain), lvl2 **8.4 dmg, 4.0 hp** (I00F text says 9.4). 
Lvl2 attachment is `A013` for models `h003` (Heretic skin) and `h009` (Occultist), `A011` otherwise. **Makes Aegis unavailable** (Aegis refuses if a sword exists and vice versa). Buying the sword while owning lvl2 or a Mask+Sword fails.
Sell: 6 gold for lvl1, 12 for lvl2; resets `LevelF=0` and Scourge level to 1.

**Combined "Mask + Sword" (I00O)**: Not a shop button; created when the sword (h007) is bought while the Mask (I004) is held, or the Mask (h001) is bought while a lvl1 sword is held. Result: a single item I00O that is **sword level 1 stats** (11 dmg + 2 DP etc.) AND the mask upgraded to **50 % life steal, 25 % damage point recovery** for Scourge/Bane(+orb)/Redemption (`P_ScourgeLSBonus` = 5) with **no weakness change** (the Mask's -25 % weakness is cancelled in the combine path of h007; in the h001 path it was never applied). It occupies one slot, **cannot be sold**, and blocks any further purchase of Mask, Sword (incl. lvl2) or Aegis ("Item does not stack with Mask and Sword"). Cost paid: whichever button was clicked last (mask 4 or sword 7); i.e. total sword 7 + mask 4 = 11 gold.

**Aegis (h008; I00H)**: Scourge (F spell) grants a defensive stance for **5 seconds** x `P_TimeInc`: damage taken x0.75 (`/ (4/3)`), knockback x0.5. It must be charged first: each time a fireball hits an opponent `AegisReady` becomes true and the item shows 1 charge (`ChangeItemCharges(2,1)` at ~11292/11366); casting Scourge consumes it (`ActivateAegis` ~10239, called from the Scourge spells at 16105 etc.), buff id `aegs`, small ability A01C as visual flag. Mass +20 kg, **HP -10** (hidden ability A000 "Item Life Penalty" = -10 life). Attachment ability A012. Sell 8 (mass back, ability removed). Not purchasable with any sword item. Buying the Aegis itself calls `AegisRemove(id)` (~2080), which clears `AegisReady` and the charge indicator.

**Stone of Jordan Ring (h004; I00E)**: see section 6. Price 1; cannot be sold; once bought you can raise one spell 2 levels above its max; the gold price is 1 plus nothing else.

**Lava Treads (h00C; I00J, with unused I00K/I00L)**: a boots item that does not stack with Boots (and refuses if any boots/Lava Treads owned). Ability trigger: when the wearer casts the Scourge/another listed spell while standing on lava terrain (`LavaTreads[id]` true and `GetTerrainType == TerrainTypeLava[TerrainOption]`, checked in about 9 spell casts: lines 14977, 15011, 15043, 15176, 15243, 15285, 16112, 16253, 16543...) `ActivateLavaTreads` (~10260) runs: lava damage `C_LavaDmgPlayer /= 8` (**87.5 % resistance**), **+35 movement speed** during the effect, HP topped up to **10.5** if lower, duration **3 s x P_TimeInc** (JASS), reload **28 s** (`AddAction_Real LavaTreadingReload 28`). The tooltips differ by tier: I00J "Level 1: +10 ms, 5 seconds"; I00K "+24 ms, 4 s"; I00L "+35 ms, 5 s"; all say "Recovers up to 10 HP when below that limit", cooldown 28 s, life steal weakness +5 % for K/L. The remake must pick one: the script only implements the level-3 numbers (87.5 %, +35 ms, 3 s) while the only item ever granted is I00J, whose tooltip says 10 ms / 5 s (uncertain / inconsistent; BuyItem adds no passive ms, yet SellItem subtracts 10 ms for I00J, and 24 / 35 for K / L). Charge display: item charge 1 = ready, 0 = cooling. Sell 5 (I00J), 10 (K), 15 (L).

**Pocket Watch (h00E; I00M)**: curses received last **35 % shorter** (`P_TimeDec *= 0.65`), buffs received and curses cast last **5 % shorter** (`P_TimeInc -= 0.05`). Sell 3.

**Cursed Pendant (h00F; I00I)**: max HP **+10** (ability A004 on I00I), knockback taken **-15 %** (`x0.85`), curses on yourself last **+25 %** longer (`P_TimeDec *= 5/4`). Sell 2.

**Warden (h00G; I00N -> I00P -> I00R -> I00S)**: flat damage reduction for knockback-damaging sources, applied *before* other damage modifiers (code subtracts `P_DamageFlatReduce` of attacker and of the target from `value`, lines ~4941-5043, 11541): **0.4 / 0.8 / 1.2 / 1.6** per level (also reduces the damage the wearer *deals*: "taken and dealt"). Max level 4. Sell 1 / 2 / 3 / 4.

### 3.3 Rules and error messages (`RefundIf`)

`RefundIf(id, gold, lumber, cond1, msg1, cond2, msg2, cond3, msg3)`: if any condition is true, shows (red `|cffee1005`) the message of the *first* true condition, adds back `gold` and `lumber`, returns true (purchase cancelled). Message shortcuts: `$0` "No inventory available", `$1` "Item is already at max level", `$2 X` "Item does not stack with X".
* Boots: refuses when I008 owned ($1), when any Lava Treads owned ("$2 Lava Treads"), or no inventory ($0); otherwise lvl1 / upgrade.
* Cape / Pendant / Helmet / Warden: upgrade in place when a lower level is held; refuse at max level; otherwise need a free slot.
* Staff, Scroll, Pocket Watch, Cursed Pendant, Aegis, Ring, Lava Treads: single copy, refuse with $1 if held.
* Mask: refuses if Mask+Sword combined or mask+lvl2 sword; with lvl1 sword -> combine.
* Sword (h007): with a Mask held (and no sword) -> combine into I00O; with a lvl1 sword (and no lvl2/Aegis) -> upgrade to lvl2; refuses with Mask+Sword combined, lvl2 sword ($1), or Aegis ("$2 Aegis").
* Lava Treads vs Boots refusal: "$2 Boots".
* Aegis vs Sword refusal: "$2 Blood Sword" (and vice versa "$2 Aegis").
* Messages after success: "Item purchased" (inventory count grew) or "Item upgraded" (count equal). `SetUnitState(warlock, LIFE, P_HP)` re-applies the current HP after any buy/sell (max-HP changes) and the sell-related inventory ability `M028` is shown/hidden depending on whether any item is held.
* Selling messages: "Sold <item>.", "<item> cannot be sold." (Ring, I00O), "No item found." Sell uses the *inventory slot* (1..6 buttons Sell 1-6, chat `-sell N`) via `UnitItemInSlot` and `InvIndex(id, itemId)` (finds the first slot containing a type; returns 0 if none).
* Max HP is applied through item life-bonus abilities, so `P_HP` (the game's own HP array) is reset to the current life after each transaction.

## 4. Stat effect summary (what each stat means in the remake)

| Stat | Code variable | Base | Items that change it |
|---|---|---|---|
| Max HP | item abilities (Ilif) | 100 (P_HP init 100) | Pendant +5/+10/+15, Cursed Pendant +10, Ring -10, Aegis -10 |
| Mass | `P_Mass` | 1.00 (100 kg) | Helmet +0.1/0.2/0.3, Ring +0.2, Aegis +0.2, Boots -0.05 (once). Used in collision/knockback: `r = 2*m_self/(m_self+m_other)` etc. (line 6203), and hero acceleration `/(mass+1)` (14700) |
| Life regen | `P_Regeneration` x `P_RegenMultiplier` | 0.05 per 0.1 s tick (0.5 hp/s); multiplier 1.0 + 0.05 per mastery level | Cape +0.15/0.25/0.35 hp/s, Pendant lvl3 +0.10 hp/s |
| Knockback taken | `P_KnockbackMultiplier` | 1.0 | Helmet 0.84/0.76/0.68, Cursed Pendant x0.85, each mastery level -2.5 %, Aegis buff x0.5, Ring/Scourge "50 % knockback reduction" |
| Missile range | `LevelRanged` (range = 1 + 0.12 x level) | 0 | Range Mastery +1/level, Helmet -1/-2/-3 |
| Move speed | `SetObjectMS` | per-class | Boots +10/+20, Lava Treads +35 temp |
| Damage point (DP) recovery | F spell `LevelMelee` + `P_ScourgeLSBonus` | 0 | Mask +10 %, combined +25 %, Life steal mastery +5 %/level, Sword (+2/+3 DP recovery flat) |
| Life steal weakness | `P_LifeStealMultiplier` | 1.0 | Pendant +5/10/15 %, Lava Treads K/L +5 %, Mask -25 % |
| Curse duration taken | `P_TimeDec` | 1.0 | Pocket Watch x0.65, Cursed Pendant x1.25 |
| Own buff/curse duration | `P_TimeInc` | 1.0 | Duration mastery +8 %/level, Pocket Watch -5 % |
| Flat damage reduction | `P_DamageFlatReduce` | 0 | Warden 0.4/0.8/1.2/1.6 |
| Damage taken multiplier | `P_DamageMultiplierTake` | 1.0 | Aegis buff x0.75 (Ring: "25 % damage reduction") |

## 5. Spells, upgrades and masteries in the shop (Research___Actions)

Purchases of spells use WC3 **researches**. Trigger: `EVENT_PLAYER_UNIT_RESEARCH_START` on any player -> `Research___Actions` (~10642), which also issues the cancel order `0xD0008`. Cost is the research gold cost (`gglb` base, `gglm` per-level increment, here always +1 per level) deducted by the engine. After a learn-spell research the shop is rebuilt as `u000` (root menu).

### 5.1 Categories (hotkeys / SpellsAcquired slots)

`SpellsAcquired[8*id + slot]` slot 0 = `S000` Fireball (G, always owned), 1 = D spell, 2 = E spell, 3 = R spell, 4 = T spell, 5 = Y spell, 6 = C spell, 7 = the character's F spell (Scourge/Bane/Redemption). The player may own **at most one spell per letter D/E/R/T/Y/C** (learning one disables all other learn-researches of that letter, incl. its Random research), so max 6 bought spells + Fireball + F spell.

How the script tells a first-learn research from an upgrade: first-learn researches have `SetPlayerTechMaxAllowed == 10` (`glvl` 10) and upgrades have a smaller cap (5, 6 for Teleport/Thrust, 4 for Gravity); this is why the learn branch sets the matching upgrade research max (`SetPlayerTechMaxAllowed(R00P,5)` etc.).
Upgrade branch (`GetPlayerTechMaxAllowed > 1`): increments the Level{D,E,R,T,Y,H} counter and `SetUnitAbilityLevel(spell, +1)`, message "Spell upgraded".
Learn branch message: "Spell purchased"; `SpellsPurchased[id]++`.

Costs (from w3q): learn = `gglb`; upgrade = `gglb` of the upgrade research (+1 per upgrade level beyond the first, `gglm=1`).
The numbers below are `gglb` at level 1; level n of that research costs `gglb + (n-1)`.
(Assumption, standard WC3 upgrade cost = base + mod*(level-1); uncertain because the data was parsed without engine semantics.)

### D spells (SpellsAcquired slot; shop units u009 (R021,R01H,R01P,R00Z,R005,R009))

| Spell | Learn research | Learn cost (base, +1/level) | Upgrade research | Upgrade cost lvl1 (+1 per level) | Max upgrades (T-allowed) | Ability id |
|---|---|---|---|---|---|---|
| Lightning | R009 | 11 | R00P | 7 | 5 | S002 |
| Homing | R005 | 11 | R00O | 7 | 5 | S003 |
| Boomerang | R00Z | 11 | R010 | 7 | 5 | S004 |
| Unused | R01F | 11 | R01G | 6 | 5 | S053 |
| Arc | R01H | 11 | R01I | 7 | 5 | S054 |
| Poison Bolt | R01P | 11 | R01Q | 7 | 5 | S058 |
| Shock | R01X | 11 | R01Y | 7 | 5 | S062 |

### R spells (SpellsAcquired slot; shop units u001 (R023,R01R,R013,R007,R004) + u007 Primary 2 (R01R,R01P,R01H))

| Spell | Learn research | Learn cost (base, +1/level) | Upgrade research | Upgrade cost lvl1 (+1 per level) | Max upgrades (T-allowed) | Ability id |
|---|---|---|---|---|---|---|
| Teleport | R004 | 11 | R00V | 6 | 6 | S011 |
| Thrust | R007 | 11 | R00W | 6 | 6 | S012 |
| Swap | R013 | 11 | R014 | 6 | 5 | S013 |
| Return | R01R | 11 | R01S | 6 | 5 | S059 |

### T spells (SpellsAcquired slot; shop units u010 (R022,R01V,R003,R00E,R001))

| Spell | Learn research | Learn cost (base, +1/level) | Upgrade research | Upgrade cost lvl1 (+1 per level) | Max upgrades (T-allowed) | Ability id |
|---|---|---|---|---|---|---|
| Drain | R00E | 14 | R00L | 6 | 5 | S014 |
| Fire Spray | R001 | 14 | R00J | 6 | 5 | S015 |
| Bouncer | R003 | 14 | R00K | 6 | 5 | S016 |
| Dagger | R01V | 14 | R01W | 6 | 5 | S061 |

### E spells (SpellsAcquired slot; shop units u012 (R025,R018,R00F,R00A,R00H,R01J) + u008 Secondary 2)

| Spell | Learn research | Learn cost (base, +1/level) | Upgrade research | Upgrade cost lvl1 (+1 per level) | Max upgrades (T-allowed) | Ability id |
|---|---|---|---|---|---|---|
| Meteor | R00A | 14 | R00R | 6 | 5 | S008 |
| Wind Walk | R00F | 14 | R00X | 6 | 5 | S010 |
| Splitter | R00H | 14 | R00U | 7 | 5 | S009 |
| Repeater | R018 | 14 | R01D | 6 | 5 | S030 |
| Burst | R01J | 14 | R01K | 7 | 5 | S055 |

### C spells (SpellsAcquired slot; shop units u011 (R024,R011,R00C,R006,R01L,R01B,R01N,R01T) + u002/u008)

| Spell | Learn research | Learn cost (base, +1/level) | Upgrade research | Upgrade cost lvl1 (+1 per level) | Max upgrades (T-allowed) | Ability id |
|---|---|---|---|---|---|---|
| Rush | R006 | 12 | R00S | 6 | 5 | S007 |
| Shield | R00C | 12 | R00T | 6 | 5 | S005 |
| Time Shift | R011 | 12 | R012 | 6 | 5 | S006 |
| Mirror | R01B | 12 | R01C | 6 | 5 | S048 |
| Vector | R01L | 12 | R01M | 6 | 5 | S056 |
| Dome | R01N | 12 | R01O | 5 | 5 | S057 |
| Ethereal Form | R01T | 12 | R01U | 6 | 5 | S060 |

### Y spells (SpellsAcquired slot; shop units u013 (R026,R008,R00G,R00B,R019))

| Spell | Learn research | Learn cost (base, +1/level) | Upgrade research | Upgrade cost lvl1 (+1 per level) | Max upgrades (T-allowed) | Ability id |
|---|---|---|---|---|---|---|
| Link | R00B | 11 | R00Q | 4 | 5 | S019 |
| Disable | R008 | 11 | R00M | 6 | 5 | S017 |
| Gravity | R00G | 11 | R00N | 7 | 4 | S018 |
| Control | R019 | 11 | R01A | 3 | 5 | S050 |
| Lock On | R01Z | 11 | R020 | 6 | 5 | S064 |


Random buttons (first-learn research that picks a random spell of the category and replaces `researchedTech` with it, then disables itself; categories are the arrays in `Research___InitTrig`):

- R021 Random D Spell: cost 11
- R023 Random R Spell: cost 11
- R022 Random T Spell: cost 14
- R025 Random E Spell: cost 14
- R024 Random C Spell: cost 12
- R026 Random Y Spell: cost 11

(Random spell pools: D = Lightning, Homing, Boomerang, Arc, Poison Bolt; R = Teleport, Thrust, Swap, Return; T = Drain, Fire Spray, Bouncer, Dagger; E = Meteor, Wind Walk, Splitter, Repeater (and Burst E in the E list of the shop, not in the random pool: `SpellResearchIdArrayMax[3]=3`); C = Rush, Shield, Time Shift, Mirror, Vector, Dome, Ethereal Form; Y = Disable, Gravity, Link, Control (4 of 5; Lock On not in the random pool). Spells R01F (Burst D), R01X (Shock) and R01Z (Lock On) exist in the Research code but R01F/R01X are not on any shop unit's `ures` list (disabled/hidden, likely from "spellbook" extras; uncertain).)

Fireball (G): `R002`, cost 3 (+1 per level, 11 levels: Fireball 2..12). Levels 11 and 12 are only reachable with the Stone of Jordan Ring (the w3q text says "Applied by the Stone of Jordan Ring"). Each level +0.50 damage. `LevelG` counter (also SpellPenalty is not applied to it).

### 5.2 Inventory research

`R000` "Inventory" (3 levels, 2 gold each, no increment): +2 item slots per level (`IncUnitAbilityLevel(S128)`, w3q gub1 "Grants 2 additional item slots"). Hence 2 slots at start -> 4 -> 6 for a total 4 gold.

### 5.3 Masteries (shop unit u006)

All masteries are 6 levels; each mastery level also gives **-2.5 % knockback taken and +5 % HP regeneration** to the owner (`P_RegenMultiplier += 0.05`, KB multiplier recomputed from the total `treal` of masteries plus helmet levels: `x = (1-0.025*treal)/(1-0.025*(treal-1))` for single, ... for Jack of All Trades: `treal-3`).

| Research | Name | Base cost (+1 per level) | Effect per level |
|---|---|---|---|
| R00D | Life steal Mastery | 6 | +10 % life steal and +5 % damage point recovery (F spell melee; `LifestealPerLvl=0.10`, `LevelMelee`) |
| R00I | Duration Mastery | 4 | +8 % duration of all spells (buffs cast, buffs received, curses cast and duration spells); `DurationPerLvl=0.08`, `P_TimeInc` |
| R00Y | Range Mastery | 5 | +12 % missile range (`RangePerLvl=0.12`, `LevelRanged`); cap shrinks by helmet level |
| R01E | Jack of all Trades | 13 | raises all three masteries by one level at a 2 gold discount vs buying separately (15 total); 18 levels (6 x 3); disables itself when any one mastery reaches 6 |
| R017 | "N any mastery" | 0 | display-only counter of total mastery levels (`SetPlayerTechResearched R017 = treal`) |

Gold needed for a full mastery = sum of base+n.

## 6. Stone of Jordan Ring

* Button `h004` (price **1**), item `I00E`. Description: "When purchased, you can upgrade a spell of choice 2 times beyond its maximum level. This bonus is applied to the first spell you upgrade after purchasing this item. Cannot be sold." ("Learn an ultimate" in the M036 tooltip.)
* `BuyItem` (h004): refuses if already owned; adds `I00E`; sets `Allitems[14]` (owned) and `Allitems[18]` (pending, "Jordan" flag); makes ability `M036` available; `JordanMake(id,x,y)`: removes the player's shop, creates `u004` (Stone of Jordan Ring shop), calls `JordanRemove` (TechMax 0 for all T researches) then, for slots 0..6 of the player's acquired spells, enables TechMax 1 of `T00x = spell id + ('T000'-'S000')`; selects the new shop for that player.
* The `T0xx` entries are zero-cost pseudo-researches named "+2 upgrades for this spell" (Tooltip "Apply Stone of Jordan Ring"; special names for `T064` Lock On, `T062` shock). They are the Jordan menu: one button per owned spell.
* On researching one (`Research___Actions`: `researchedTech` in `T000..T061` -> `applyjordan`, ~10507): if `Allitems[18]` is true: adds +2 to the maximum of that spell's **upgrade** research (`R002` for Fireball = `T000`; or the first upgrade research with tech max > 0 of that category: D = T002/3/4/53/54/58/62 -> R010,R00O,R00P,R01G,R01I,R01Q,R01Y; R = T011/12/13/59 -> R00V,R00W,R014,R01S; T = T014/15/16/61 -> R00L,R00K,R00J,R01W; E = T008/9/10/30/55 -> R00R,R00U,R00X,R01D,R01K; C = T005/6/7/48/56/57/60 -> R00T,R012,R00S,R01C,R01M,R01O,R01U; Y = T017/18/19/50/64 -> R00M,R00N,R00Q,R01A,R020), then `JordanRemove`, `Allitems[18]=false`, `M036` hidden, message "Stone of Jordan Ring has been applied", the shop is rebuilt as `u000`. If the flag is already false: "Stone of Jordan Ring can only be applied once". So **the bonus is one-time per purchase of the ring**; the ring item stays in the inventory (cannot be sold, `Allitems[14]` stays true, so it cannot be re-bought): net one permanent +2 for one spell, with the ring item occupying one slot (uncertain whether the ring item is deleted somewhere else).
* Ring costs 1 gold; the HP/mass penalty in the other item named "Stone of Jordan Ring" (**I00H**) belongs to the **Aegis** item (its w3t name is a leftover): -10 HP, +20 kg, 25 % damage reduction, 50 % knockback reduction on Scourge (see Aegis).
* The Ring buttons `T000..T064` are hidden unless the ring was bought and the player has the matching spell (SetPlayerTechMaxAllowed 1).

## 7. Cosmetics (summary)

`Cosmetics` (~18209), trigger `EVENT_PLAYER_UNIT_SELL` (cosmetic units are *sold* from the shop: `u014` Free Cosmetics / `u015` Paid Cosmetics browsers). Purely visual: the sold unit's type maps to an *attachment ability* (A00S etc. for the model on the warlock) or, for the `j0xx` units, a **death effect** (death sfx + blood splat model). One equipped item per slot: head / body / back / staff / wrists (`P_COSMETICS_HEAD/BODY/BACK/STAFF/WRISTS` with their token cost) and one Fatality (`P_COSMETICS_FATALITY`, default `j000` Flames). Buying a new item of the same slot refunds the previous item's tokens ("Refunding previous Cosmetic") and re-clicking the same cosmetic unequips it and refunds. Cost = `GetFoodMade(unit)` = the unit's `ufma` in tokens (lumber): 0 = free. Tokens are earned by points in-game (see 1). Some cosmetics depend on character (Patriot `i034` picks A01F / A01E / A01G / A01D by model type h003, h00J, h009, else). The sold unit is removed. VIP-style cosmetics are in `AllItemUIDs[size..]` pages (index offset 16 and 72 for fatalities) and paged by the same `GotoItemPage`; "VIP" flag chooses between paid and free page sets. Remake: low priority; cosmetics are optional.

Cosmetic units (token cost from w3u):

| id | Name | Slot | Token cost (ufma) | Note |
|---|---|---|---|---|
| i003 | Horned | Head | free |  |
| i005 | Vampiric Wings | Back | 60 |  |
| i007 | Demon Wings | Back | 50 |  |
| i008 | Belt | Body | free |  |
| i009 | Bone Armor | Body | free |  |
| i010 | Dragon's Head | Head | 20 |  |
| i011 | Marauder Helm | Head | free |  |
| i012 | Left Hand Claws | Wrists | free |  |
| i013 | Right Hand Claws | Wrists | free |  |
| i014 | Warrior's Crown | Head | free |  |
| i015 | Dual Claws | Wrists | free |  |
| i016 | Crypt Helm | Head | free |  |
| i017 | Demon Mask | Head | free |  |
| i019 | Golden Claws | Wrists | free |  |
| i020 | Horned Helm | Head | free |  |
| i021 | Frozen | Body | 30 |  |
| i022 | Magician's Hat | Head | 60 | ufma=60, ulum=None (uncertain which is charged; JASS reads ufma) |
| i024 | Pumpkinhead | Head | 20 |  |
| i025 | Skullface | Head | free |  |
| i026 | Voodoo | Head | free |  |
| i027 | Spikehead | Head | free |  |
| i028 | Tribal | Head | free |  |
| i029 | Spikes | Body | 20 |  |
| i030 | Inquisitor Hat | Head | 30 |  |
| i031 | Mask Shop | Head | free |  |
| i032 | Wizard's Hat | Head | 30 |  |
| i033 | Crystal | Back | free |  |
| i034 | Patriot | Back | 20 |  |
| i036 | Angel Wings | Back | 50 |  |
| i037 | Spartan | Head | free |  |
| i038 | Traveler | Back | free |  |
| i039 | Lich | Back | 70 |  |
| i040 | Emperor | Back | free |  |
| i041 | Insignia | Back | 40 |  |
| is04 | Staff of Crystals | Staff | 20 |  |
| is05 | Brilliant Staff | Staff | 40 |  |
| is06 | Demon Staff | Staff | free |  |
| is07 | Dragon Tamer | Staff | 30 |  |
| is08 | Druid Staff | Staff | free |  |
| is09 | Horus | Staff | free |  |
| is10 | Rod of Necromancy | Staff | free |  |
| is11 | Scepter of Tides | Staff | 30 |  |
| is12 | Staff of Decay | Staff | free |  |
| is13 | Staff of Negation | Staff | 20 |  |
| is14 | Staff of Silence | Staff | free | ufma=None, ulum=40 (uncertain which is charged; JASS reads ufma) |
| is15 | Shaman Staff | Staff | free |  |
| is16 | Stick | Staff | free |  |
| is17 | Trident | Staff | free |  |
| is18 | Magic Wand | Staff | 50 |  |
| is19 | Angel Staff | Staff | 20 |  |
| is20 | Noble Staff | Staff | 40 |  |
| is21 | Emperor Scepter | Staff | free |  |
| j000 | Flames | Fatality (death effect) | free |  |
| j001 | Underworld | Fatality (death effect) | free |  |
| j002 | Afterworld | Fatality (death effect) | 30 |  |
| j003 | Cursed | Fatality (death effect) | free |  |
| j004 | Illusion | Fatality (death effect) | free |  |
| j005 | Bloody Mess | Fatality (death effect) | free |  |
| j006 | Cold Blooded | Fatality (death effect) | free |  |
| j007 | Kaboom! | Fatality (death effect) | 40 |  |
| j008 | Cold Hearted | Fatality (death effect) | 20 |  |
| j009 | Made of Money | Fatality (death effect) | free |  |
| j010 | Bolt of Lightning | Fatality (death effect) | 30 |  |
| j011 | Vapor | Fatality (death effect) | free |  |
| j012 | Green Goo | Fatality (death effect) | 20 |  |
| j013 | Watery Grave | Fatality (death effect) | free |  |
| j014 | Haunted | Fatality (death effect) | free |  |

## 8. Other shop-related toggles

`M037` (E), `M039` (R), `M040` (Y), `M041` (T), `M042` (D) "Toggle hotkey spell" switch a spell between two variants (e.g. Meteor: Meteorite / Magma; Splitter; Teleport: Blink / Multiport; Thrust: Phoenix...; Gravity: Dark Matter / Force Field; Link; Fire Spray: Stream / Cluster; Drain; Arc direction) via `AbilityToggle` (~10345; shifts the ability level by +/-10). They are shown on the main merchant only when a spell with variants has been learned. Variant data is the other agent's topic (spell file).

## 9. Spell purchase penalty (`SpellPenalty` ~10602)

After the **3rd, 4th and 5th** spell purchase (`SpellsPurchased` after increment: `==6` -> nothing; `>2` -> penalty) the player sees "Purchase cost of spells has increased" and the script calls `AddPlayerTechResearched(+1)` on **38 first-learn researches** (R001..R026 learn buttons incl. the six Random buttons). Because each is a leveled research with `gglm=1`, that raises their cost by **+1 gold each time**: the 1st-3rd spells cost base, the 4th base+1, the 5th base+2, the 6th base+3 (uncertain by one: depends on whether the cost uses the researched count; derived from `AddPlayerTechResearched` in the penalty and from the "Purchase cost ... has increased" message). Upgrades and masteries are not penalised.

## 10. Full research (w3q R-series) list

R-series are the real shop researches. T-series (`T000..T064`) are the Jordan pseudo-researches (zero cost, listed in section 6).
`Q000` "Disabled" (silence shown on a disabled spell) and `Q001` "Requires Stone of Jordan Ring" are ability-requirement placeholders. `Rhpm` / `Rhme` are map-info items ("Player commands", "Will be available soon").
Columns: id, name, base gold (`gglb`), per-level increment (`gglm`), level count (`glvl`). Item flags: **none of the R-researches is used as an item flag**; the item tier flags are the `Allitems[]` booleans (JASS) not researches. R000 is the inventory-slot research and R017 is the mastery counter; everything else is a spell level (learn or upgrade), mastery, or random-spell button.

| id | Name | base gold | +/level | levels |
|---|---|---|---|---|
| R000 | Inventory | 2 | 0 | 3 |
| R001 | Fire Spray | 14 | 1 | 10 |
| R002 | Fireball | 3 | 1 | 11 |
| R003 | Bouncer | 14 | 1 | 10 |
| R004 | Teleport | 11 | 1 | 10 |
| R005 | Homing | 11 | 1 | 10 |
| R006 | Rush | 12 | 1 | 10 |
| R007 | Thrust | 11 | 1 | 10 |
| R008 | Disable | 11 | 1 | 10 |
| R009 | Lightning | 11 | 1 | 10 |
| R00A | Meteor | 14 | 1 | 10 |
| R00B | Link | 11 | 1 | 10 |
| R00C | Shield | 12 | 1 | 10 |
| R00D | Life steal Mastery | 6 | 1 | 6 |
| R00E | Drain | 14 | 1 | 10 |
| R00F | Wind Walk | 14 | 1 | 10 |
| R00G | Gravity | 11 | 1 | 10 |
| R00H | Splitter | 14 | 1 | 10 |
| R00I | Duration Mastery | 4 | 1 | 6 |
| R00J | Fire Spray | 6 | 1 | 10 |
| R00K | Bouncer | 6 | 1 | 10 |
| R00L | Drain | 6 | 1 | 10 |
| R00M | Disable | 6 | 1 | 10 |
| R00N | Gravity | 7 | 1 | 10 |
| R00O | Homing | 7 | 1 | 10 |
| R00P | Lightning | 7 | 1 | 10 |
| R00Q | Link | 4 | 1 | 10 |
| R00R | Meteor | 6 | 1 | 10 |
| R00S | Rush | 6 | 1 | 10 |
| R00T | Shield | 6 | 1 | 10 |
| R00U | Splitter | 7 | 1 | 10 |
| R00V | Teleport | 6 | 1 | 10 |
| R00W | Thrust | 6 | 1 | 10 |
| R00X | Wind Walk | 6 | 1 | 10 |
| R00Y | Range Mastery | 5 | 1 | 6 |
| R00Z | Boomerang | 11 | 1 | 10 |
| R010 | Boomerang | 7 | 1 | 10 |
| R011 | Time Shift | 12 | 1 | 10 |
| R012 | Time Shift | 6 | 1 | 10 |
| R013 | Swap | 11 | 1 | 10 |
| R014 | Swap | 6 | 1 | 10 |
| R015 | unnamed | 15 | 1 | 5 |
| R016 | unnamed | 8 | 1 | 10 |
| R017 | 1 any mastery | 0 | 0 | 20 |
| R018 | Repeater | 14 | 1 | 10 |
| R019 | Control | 11 | 1 | 10 |
| R01A | Control | 3 | 1 | 10 |
| R01B | Mirror | 12 | 1 | 10 |
| R01C | Mirror | 6 | 1 | 10 |
| R01D | Repeater | 6 | 1 | 10 |
| R01E | Jack of all Trades | 13 | 1 | 18 |
| R01F | Unused | 11 | 1 | 10 |
| R01G | Unused | 6 | 1 | 10 |
| R01H | Arc | 11 | 1 | 10 |
| R01I | Arc | 7 | 1 | 10 |
| R01J | Burst | 14 | 1 | 10 |
| R01K | Burst | 7 | 1 | 10 |
| R01L | Vector | 12 | 1 | 10 |
| R01M | Vector | 6 | 1 | 10 |
| R01N | Dome | 12 | 1 | 10 |
| R01O | Dome | 5 | 1 | 10 |
| R01P | Poison Bolt | 11 | 1 | 10 |
| R01Q | Poison Bolt | 7 | 1 | 10 |
| R01R | Return | 11 | 1 | 10 |
| R01S | Return | 6 | 1 | 10 |
| R01T | Ethereal Form | 12 | 1 | 10 |
| R01U | Ethereal Form | 6 | 1 | 10 |
| R01V | Dagger | 14 | 1 | 10 |
| R01W | Dagger | 6 | 1 | 10 |
| R01X | Shock | 11 | 1 | 10 |
| R01Y | Shock | 7 | 1 | 10 |
| R01Z | Lock On | 11 | 1 | 10 |
| R020 | Lock On | 6 | 1 | 10 |
| R021 | Random D Spell | 11 | 1 | 10 |
| R022 | Random T Spell | 14 | 1 | 10 |
| R023 | Random R Spell | 11 | 1 | 10 |
| R024 | Random C Spell | 12 | 1 | 10 |
| R025 | Random E Spell | 14 | 1 | 10 |
| R026 | Random Y Spell | 11 | 1 | 10 |
| Rhme | Will be available soon. | None | 0 | None |
| Rhpm | None | None | 0 | None |

## 11. Gaps / uncertain points (summary)

1. Exact engine interaction between "train start deducts gold", the immediate cancel order and `RefundIf` (use the simple "pay unless refused" model).
2. Upgrade research cost increment: assumed +1 per level from `gglm`.
3. Lava Treads tier consistency (script implements only one set of numbers; item created is the lowest tier).
4. Whether the Ring item persists after applying (data says yes: cannot be sold).
5. Blood Sword Occultist numbers differ between w3t I00F (8.7 / 9.4) and unit text / I00G (7.7 / 8.4).
6. Staff of Fireball and Scroll of Fireball actual in-spell implementation is in fireball code (not read here); numbers come from tooltips.
7. Which extra spellbooks (u007, u008, u002) and spells (Shock, Burst D, Lock On) are active by default.
8. Death-time shopping (buy yes, sell no) is inferred from the code; there is no explicit comment.

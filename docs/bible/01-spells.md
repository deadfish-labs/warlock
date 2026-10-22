# Warlock - Spell and Character-Ability Reference (map data is v1.06n, not 0.99c)

Sources: `~/warlock-src/notes/data/war3map.j.txt` (JASS, cited as `JASS <Function>` plus line), `a.w3a.json` (custom abilities, cited as `w3a S0xx <field>`), `w3q.json` (research/upgrades), `w3u.json` (units/items), `wts.json`.
Everything below was read from the script; anything marked **uncertain** was inferred or contradicts another source. Where tooltip and code disagree the **code is treated as truth** and the disagreement is noted.

Conventions: `L` = the spell's category level (see section 2). `r` = Range Mastery level (`LevelRanged`). `TI`/`TD` = `P_TimeInc[caster]` / `P_TimeDec[victim]` (duration multipliers, section 1.6). "units" = WC3 world units. `kb` = the `KnockbackA` multiplier handed to `DamageTarget*` (section 1.4).

---------------------------------------------------------------------------------------------------

## 1. Shared spell framework (mirror this in the remake)

### 1.1 The simulation tick and "Object" model (JASS lines ~2869-3125, 5845-6300)

* **Everything physical is an `Object` (an integer slot id)** stored in parallel global arrays (`Obj_PosX/Y/Z`, `Obj_VelX/Y/Z`, `Obj_AccX/Y/Z`, `Obj_Size`, `Obj_Range`, `Obj_Bounce`, `Obj_Friction`, `Obj_Int`, `Obj_Real/Real2/Real3`, `Obj_Type`, `Obj_OwnerId`, `Obj_SpellType`, `Obj_SpellLevel`, ...). Slots come from a free list (`Object_Allocate`, `Object_Remove`); objects live in one linked list (players are put at the head of the list, `PlayerListStart`).
* **Object types** (`Obj_Type`): `1` = warlock (player), `2` = missile/effect, `3` = pillar/obstacle (and the `bPillar`/`bShard` flags mark destructible pillars and pillar shards; `IsPillar(obj)` = `bPillar or bShard`).
* **Fixed timestep** `Sys_IterRate = 0.03 s` (~33.3 Hz). **All velocities are stored in units/tick**: `speed_per_tick = speed_per_second * 0.03`. Code writes speeds as `1000*Sys_IterRate`. Warlock walk speed `Stats_MS = 220*0.03 = 6.6` units/tick (220 u/s), see `SetObjectMS` (val is in u/s and multiplied by `Sys_IterRate`).
* `Obj_Range` for missiles is a **lifetime in SECONDS** (decremented by `Sys_IterRate` per tick). A missile that should fly `D` units at `v` u/s sets `Obj_Range = D / v`. When it reaches 0 the engine moves it one more step and calls `Obj_OnEndRng` (or removes it if none).
* Helper periodic flags: `Period3` is true every 3rd tick (0.09 s); `Period5` flips every Period3 so it is true every **6th tick (0.18 s)** (name is historical). Buffs run on a separate 0.25 s timer (`BUFFS_ITER_RATE`).
* **Per-tick order** (`SystemIteration`): (1) for every object in list order: player objects apply friction/walk/ice/thrust/lock-on/link logic; missiles apply `Vel = (Vel + Acc) * Friction`, then run `OnIteration`, then count down `Obj_Range` and fire `OnEndRng`; `OnGround` if z hits ground. (2) **Pairwise collision pass** over all object pairs. (3) Run queued `OnDestroy` callbacks (`Sys_AddToDestroy`). (4) Integrate positions: world bounds (`minX..maxX`, bounce 0.5 off the arena edge), `TrSpecial` terrain is a wall (players/bouncy missiles reflect by `Obj_Bounce`, non-bouncy missiles get destroyed), then `SetUnitX/Y`.
* **Player movement model**: velocity = knockback velocity + walk velocity. Each tick: `Vel = (Vel - Walk) * Stats_Friction(0.96)`, then `Walk = Obj_Real (the move speed per tick) * direction to where the WC3 unit wants to go`, `Vel += Walk`. Ice terrain uses `Stats_IceFriction 0.93` and a slow acceleration `Stats_IceAcc = MS/10`. Stunned/entangled players use ice friction and zero walk. `P_Thrust[obj]` (thrust/ghost) uses `Vel = (Vel+Acc)*0.96` with `Obj_Range` as the remaining dash time.
* **Collision test** (`SystemIteration` second loop): two objects collide only if `Obj_Collision[3*a + type(b)] and Obj_Collision[3*b + type(a)]` (each object has three flags: collides-with-players, collides-with-missiles, collides-with-pillars), they are on different owners (unless both have `Obj_OwnerColl`) and on different teams (unless both have `Obj_AllyColl`), and neither is blocked by a shield (`S_ShieldObj`). Collision radius is `Obj_Size[a]+Obj_Size[b]` (+30 if one is a player flagged with `Player_Coll`). It is a swept-sphere test (solves for the time of impact within the tick, rewinds both objects to that point, then fires callbacks). `Obj_IgnPriority` decides which side's `OnColl` fires (callback fires only if its priority >= the other's). Player-vs-player and player-vs-pillar use soft push-out and an elastic bounce (`BounceColl`, factor 0.5, impulse weighted by `P_Mass`); impact speed > 30 gives **"pancake/velocity damage"** `0.7*sqrt(v-20)` (`WarlockVelocityDamage`) to the opponent that was last damaged (credited to `P_DamagingPlayer`).
* **Callbacks are `boolexpr` fields** set per object: `Obj_OnColl` (TObj = this, TCollObj = other), `Obj_OnEndRng` (lifetime/range ended), `Obj_OnDestroy` (queued removal), `Obj_OnIteration` (every tick), `Obj_OnGround`. Globals `TObj`, `TCollObj`, `TCasterObj`, `CasterID`, `CX/CY` (caster pos), `TX/TY` (target point) are the implicit arguments; the remake should pass them explicitly.
* **Cast pipeline** (`AbilityActions`, JASS 16969): a single trigger on `SPELL_EFFECT` for players 0-11. It sets `TCasterObj`, `CasterID`, `TX/TY` (target point), `CX/CY`, then handles **Control** (if `P_Control>0` the target point is replaced by the controlling missile's position; if `<0` the cast happens from the remote marker `Obj_PosX[4*id+514]`), cancels Mirror/WindWalk states, adds pending `CastingDamage` to DP, then dispatches by ability raw id (table in section 3). Casts in the dispatch use `TargetPositionValid()` (if target == caster, aim along facing).
* **Toggles** (`AbilityToggle`, M037/M039/M040/M041/M042): five bool arrays `toggleE/R/T/Y/D`. Every "dual" spell has two modes selected by a toggle; toggling adds/subtracts **10 ability levels** on the same ability id (so levels 11-20 in w3a hold the "mode B" tooltip/cooldown). Mode A = toggle false, mode B = toggle true. The host can ban either mode with `toggleData` (0 = free choice, 1 = only mode A, 2 = only mode B).
* **Scheduled actions** (`TimerSys`, `AddAction_Real(func, delay, int1, int2, int3, real1, real2, real3)`): a sorted event list used for every duration/cooldown effect. `SearchExec/SearchFind/SearchDel/SearchExecReal(match1,int1,match2,int2,match3,int3)` find actions by their three int keys and either **end them early (run their function now)**, find, delete or read real1. Typical key convention: `(obj, abilityKey, EndType)` where `EndType` is `A_EndBuff=139`, `B_EndBuff=140` or `C_EndBuff=141` (A = ends on cast/dispel by owner, B = ends on "dispel buffs", C = ends only by time). Many "Dispels link/curses" effects are `SearchExec(true,obj,false,0,true,B_EndBuff)` (run every B-type ending for obj).

### 1.2 Damage pipeline (JASS 4779-5215)

* `DamageTargetNoKnockback(damager_obj, target_obj, value, textvx, textvy)` = final HP loss:
  `value *= P_DamageMultiplierInflict[damager] * P_DamageMultiplierTake[target]` (after Rush absorption `Rushgain` and Mirror-break), then `HP -= value`, `DP += value` (DP = damage points, the knockback accumulator, see 1.4), lifesteal `Heal(damager, value*0.10*LevelMelee*P_LifeStealMultiplier[target])`, floating text. Time Shift intercepts death (`PObj_TS_Action`). Immortal/shop time: no HP change. Entangled players: damage accumulates in `EntangleDmgCount`; **>= 20 damage breaks the entangle**.
* Pillars/shards go through `Pillars_TakeDamage` (pillar HP in `Obj_Real`, shown as floating tag).
* `DamageTargetNormal(d, t, value, kb)` = knockback along (target - damager) + damage; `DamageTargetLine(d, t, value, kb)` = knockback along the damager's *velocity* direction; `DamageTargetPoint(x, y, d, t, value, kb)` = knockback away from a point, **returns the lifesteal heal amount** used by ray spells; `KnockbackOnly`. `DealAOEDamage(d, value, kb, x, y, radius, bonus)` (linear falloff `value*(bonus+(1-bonus)*(1-dist/radius))`), `DealCustomAoE` (same, centred on the damager object, bonus 0.5), `DealKnockbackAoE` (pillar explosions, 1.6x velocity).
* **Flat reduction** (item Warden): `value -= P_DamageFlatReduce[target] + P_DamageFlatReduce[damager]`, hit ignored if <= 0 (for pillars only the damager's term applies).
* Buff damage (`Buffs_DPS_Add`) ticks every 0.25 s through `DamageTargetNoKnockback`.

### 1.3 Buff system (`Buff_*`, JASS 5217-5365, 3757-3800)

* Arrays `Buff_Obj/Type/Owner/Time/TimeElapsed/DPS/Slow/Effect` + linked list. Types (`BUFF_TYPE_*`): MAGMA 1246, QUAKE 659, POISON 2023, FBALL 1353, FMAGE 1723, BOLT 1828, ARC 1923, DRAIN 2123, DAGGER 2223 (`BUFF_FLAME 50` unused).
* `Buffs_DPS_Add(type, damager_obj, target_obj, dmgTotal, time)`: stores `DPS = dmgTotal / time * 0.25` per 0.25 s tick (i.e. total damage `dmgTotal` over `time` seconds; **DPS in seconds = dmgTotal/time**), `Time = time`. `Buff_Find(type, target, ownerId)` finds a same-type buff from the same caster; repeat hits normally just do `Buff_SetTime(bf, newTime)` (refresh, no stacking), `Buffs_DPS_Prolong` extends time and damage.
* `Buff_SetSlow(bf, value, obj, owner, only_positive)`: slows by subtracting `value` (units/tick) from the target's move speed `Obj_Real` (removes slows from other buffs first, never below 0), restored in `Buff_Destroy`. Standard slow strength: `0.50 * P_KnockbackMultiplier[target] * Stats_MS` (= 50% of base speed, scaled by KB multiplier).
* Bolt buff has a delayed slow (after `TimeElapsed >= 2` slow 50%, removed at `>= 3`). Buffs are removed with `Buffs_ClearObjBuffs(obj)` (Mirror, Time Shift).
* Non-DPS buffs (move speed, damage multipliers, silence, stun, curses) are done with `AddAction_Real(endFunc, duration, obj, key, EndType, real...)`: apply the change immediately, and the scheduled `endFunc` undoes it; `ActEff` carries the attached effect to destroy.

### 1.4 Knockback model

* Each player has `P_DP` (damage points, shown as mana bar). On damage: `P_DP += value`. Knockback velocity impulse per hit:
  `impulse(units/tick) = (100 + DP_target) * value * InflictMult * TakeMult * 0.03 * P_KnockbackMultiplier[target] * kb`, applied along the chosen direction. (So a hit of damage `v` on a target with DP `d` gives velocity `(100+d)*v*0.03*kb` units/tick; velocity then decays 4%/tick.)
* Shards use `100*value*kb*0.03` (x0.4 defensive, x0.8 offensive shard factor).
* `P_KnockbackMultiplier` is reduced by items/masteries (see 4.2); lowered Mass/higher Mass changes collision impulse only.

### 1.5 Spell categories, levels and slots

Each spell category has a hotkey, a `SpellsAcquired[8*id + index]` slot and a level array. A warlock can own **at most one spell per category** (buying one zeroes the other techs of that category).

| Hotkey | SpellsAcquired idx | Level array | Category (my label) | Spells |
|---|---|---|---|---|
| G | 0 | `LevelG` (starts 1) | Fireball (always owned) | Fireball `S000` |
| D | 1 | `LevelD` | "D" ranged attacks | Lightning `S002`, Boomerang `S004`, Homing `S003`, Arc/Phantasm `S054`, Poison Bolt `S058`, Shock `S062` (`S053` exists in the tech table but is unused/unnamed) |
| E | 2 | `LevelE` | "E" area/utility attacks | Meteor/Magma `S008`, Wind Walk `S010`, Splitter `S009`, Repeater `S030`, Burst `S055` |
| R | 3 | `LevelR` | "R" movement | Teleport/Multiport `S011`, Thrust/Ghost `S012`, Swap/Relocate `S013`, Return `S059` |
| T | 4 | `LevelT` | "T" projectile specials | Drain `S014`, Fire Spray `S015`, Bouncer `S016`, Dagger `S061` |
| Y | 5 | `LevelH` (**note: the array is H**) | curses/control/links | Link `S019`, Disable `S017`, Gravity `S018`, Control `S050`, Lock On `S064` |
| C | 6 | `LevelY` (**note: array is Y**) | defence | Rush `S007`, Shield `S005`, Time Shift `S006`, Mirror `S048`, Vector `S056`, Dome `S057`, Ethereal `S060` |
| F | 7 | `LevelF` (0 at start, raised by items) | character spell (see section 8) | Scourge `S001`, Bane `S022`, Redemption `S023`, Ritual `S051`, Enchant `S100`, (Cataclysm `S020`, Pious `S021` = game-mode only) |

(`SPELL_ID_*` constants in the globals use a different 6-per-row grid: row 0 D, 1 R, 2 T, 3 E, 4 C(Y), 5 Y(H); Fireball = -2, Scourge = -1.)

* `SpellLevel` handed to a missile (`Obj_SpellLevel`) = the relevant Level array **at cast time** (e.g. `LevelD` for lightning). Ability level shown to the player = the same number (tooltip values below are tabulated by ability level, level 1 = freshly bought).
* **Buying** (`Research___Actions`, JASS 10642): researching a spell tech (`R0xx`, "first" tech) increments the category Level by 1, stores the new ability id in `SpellsAcquired`, makes it available and sets the max for the upgrade tech (`SetPlayerTechMaxAllowed`). Random-spell techs `R021-R026` pick a random tech of D, R, T, E, C(Y), Y(H) categories (`SpellResearchIdArrayMax`: D 4, R 3, T 3, E 3, Y 6, H 3; D random set excludes `S062`/`R01X`).
* **Upgrading**: research of the "upgrade" tech (`R00P`, `R00O`, ... ) increments the Level and does `SetUnitAbilityLevel(+1)`. Upgrade limits (`s__SpellResearchLevels` / `SetPlayerTechMaxAllowed`): D spells 5 upgrades, R spells 6 (Teleport, Thrust) / 5 (Swap, Return), T 5, E 5, Y(C) 5, H(Y) 5 (Gravity 4). **Stone of Jordan Ring** (item `h004`, `T0xx` techs) adds +2 max upgrades to the first spell upgraded after purchase (`applyjordan`). Fireball (`R002`) upgrades are capped by tech max 10 (+2 Jordan); w3a has Fireball tooltips to level 12. **uncertain**: exact max level of Fireball (10 vs 11 upgrades) - `w3q R002 glvl=11`.
* **Mastery gates** (`w3q greq/grqc`): some upgrade techs require mastery points. `R017` = "any mastery" total (set to `LevelMelee+LevelRanged+LevelDuration (+item bonus)`); Fireball upgrade n needs `R017 >= 1,1,2,2,3,3,4,4,5,5,6`; Lightning/Boomerang upgrades need Range Mastery (`R00Y`) `1,1,2,2,2,2,2,2` / `1,1,1...`; Bouncer `R00K` needs `R00Y` `1,2,2,2,2,2,2`; Fire Spray `R00J` needs `R00Y`; Homing/Arc/Burst/Shock-upgrade need `R017` `1,1,2,2,2,2,2,2`; Meteor `R00R` and Repeater `R01D` need `R017 >= 1`.
* **Costs** (`w3q gglb/gglm`, gold = base + 1 per researched level): buying a spell costs 11-14 gold (D 11, R 11, T 14 (Drain/Fire Spray/Dagger 14, Bouncer 14), E 14, C(Y) 11-12, Y(H) 11); upgrades cost 3-7 gold (Fireball 3, Link 4, Disable/Gravity 6-7, ...). After the 3rd purchased spell (`SpellsPurchased > 2`, `SpellPenalty`) every spell tech gets +1 level cost ("Purchase cost of spells has increased"). Masteries: Life steal `R00D` 6g (6 lv), Duration `R00I` 4g (6 lv), Range `R00Y` 5g (6 lv, minus helmet levels), Jack-of-all-trades `R01E` 13g (3 lv each in one go, 18 max).

### 1.6 Global per-player multipliers used by all spells

`P_DamageMultiplierInflict` / `P_DamageMultiplierTake` (default 1), `P_KnockbackMultiplier` (1), `P_LifeStealMultiplier` (1; "weakness to life steal" - higher means the *victim* gives more life steal), `P_Mass` (1), `P_DamageFlatReduce` (0), `P_TimeInc` (1, duration of own buffs/curses cast; +0.08 per Duration Mastery level, -0.40 while Controlled), `P_TimeDec` (1, duration of buffs received; Pocket Watch -35% etc.), `P_Regeneration` (HP/s; 0.05), `P_RegenMultiplier`. Every duration below is written `base * TI * TD` when both are applied by the code (curses use both; self-buffs only `TI`). Range factor `RangePerLvl = 0.12` per Range-Mastery level (`(1 + 0.12*r)`, r can be negative with helmets); `LifestealPerLvl = 0.10` per Life steal level; `DurationPerLvl = 0.08`.

---------------------------------------------------------------------------------------------------

## 2. Standard values every missile spell shares

* `Dummy` missile unit type `'e000'` (`DummyType`), created with `Object_Create(2, ownerId, 'e000', x, y, facing, modelPath)`; owner colour via `SetUnitColor`. Distance = `Obj_Range * speed`.
* `Obj_Bounce` > 0 makes it bounce off terrain wall (`TrSpecial`) with that restitution, `0`/`-1` means it dies on the wall.
* Missile `Obj_Size` is the collision radius in units (fireball 25, bolt 38, lock-on 39, silence 23, ...).
* Standard hit handling: `Sys_AddToDestroy(TObj)` + clearing `Obj_Collision` so it cannot hit twice, then `OnDestroy` plays the effect and removes the unit.

---------------------------------------------------------------------------------------------------

## 3. Ability raw id -> cast function map (`AbilityActions`, JASS 16969-17300)

| Raw id | Display name | Dispatch (mode A / mode B) | Research id (first / upgrade) |
|---|---|---|---|
| S000 | Fireball | `FireballCast` (Ghost-mode HiddenPlayer -> `IceCast`) | -, `R002` |
| S001 | Scourge (Warlock) | `ScourgeCast` (becomes `EnchantCast` while `LockScourge`) | - |
| S022 | Bane (Heretic) | `BaneCast` | - |
| S023 | Redemption (Inquisitor) | `RedemptionCast` | - |
| S051 | Ritual (Occultist) | `RitualCast` | - |
| S100 | Enchant (Enchanter) | `EnchantCast` | - |
| S020 | Cataclysm | `CataclysmCast` (Ghost game mode) | - |
| S021 | Pious | `PiousCast` (King game mode) | - |
| S002 | Lightning | `LightningCast` | `R009` / `R00P` |
| S003 | Homing | `HomingCast` | `R005` / `R00O` |
| S004 | Boomerang | `BoomerangCast` | `R00Z` / `R010` |
| S054 | Arc (internal `PHANTASM`) | `ArcCast` | `R01H` / `R01I` |
| S058 | Poison Bolt | `BoltCast` | `R01P` / `R01Q` |
| S062 (+`A02G` action) | Shock | `ShockCast` | `R01X` / `R01Y` |
| S053 | (unused, no name) | none | `R01F` / `R01G` |
| S011 | Teleport | `toggleR`: false `TeleportCast`, true `MultiportCast` (+`S029` `TeleSecondCast`) | `R004` / `R00V` |
| S012 | Thrust | false `ThrustCast`, true `GhostCast` | `R007` / `R00W` |
| S013 | Swap | false `SwapCast`, true `RelocateCast` | `R013` / `R014` |
| S059 | Return | `ReturnCast` | `R01R` / `R01S` |
| S014 | Drain | `DrainCast` (`toggleT` picks slow vs weakness mode) | `R00E` / `R00L` |
| S015 (alt ids `S15B`, `S15C`) | Fire Spray | `FlameClusterCast` (`toggleT`: false stream, true cluster) | `R001` / `R00J` |
| S016 | Bouncer | `toggleT` false `BouncerCast`, true `RechargeCast` | `R003` / `R00K` |
| S061 | Dagger | `DaggerCast` | `R01V` / `R01W` |
| S008 | Meteor | `toggleE` false `MeteorCast`, true `MagmaCast` | `R00A` / `R00R` |
| S010 | Wind Walk | false `WWCast` (Charge), true `WWCast2` (Invisibility) | `R00F` / `R00X` |
| S009 | Splitter | false `SplitterCast` (target), true area mode inside the same function | `R00H` / `R00U` |
| S030 | Repeater | `RepeaterCast` (no toggle branch in code) | `R018` / `R01D` |
| S055 (+`A02F` action) | Burst | `BurstInitiate` / `BurstCast` | `R01J` / `R01K` |
| S007 | Rush | `RushCast` | `R006` / `R00S` |
| S005 | Shield | `ShieldCast` | `R00C` / `R00T` |
| S006 (+`S047` action) | Time Shift | `TimeShiftCast` / `TS_EndEarly` | `R011` / `R012` |
| S048 | Mirror | `MirrorCast` (arms; clones on next move order) | `R01B` / `R01C` |
| S056 (+`A02J` action) | Vector | `VectorCast` | `R01L` / `R01M` |
| S057 | Dome | `DomeCast` | `R01N` / `R01O` |
| S060 (+`A02H` action) | Ethereal Form | `EtherealCast` | `R01T` / `R01U` |
| S019 (+`S031` action) | Link | `toggleY` false `LinkCast` (Hook), true `AdhesionCast` (Induction) | `R00B` / `R00Q` |
| S017 | Disable | false `EntangleCast`, true `SilenceCast` | `R008` / `R00M` |
| S018 | Gravity | false `GravityCast` (Dark Matter), true `ForcefieldCast` (Force Field) | `R00G` / `R00N` |
| S050 (+`S052` action) | Control | false `ControlCast` (Enemy cast), true `ControlToggleCast` (Self cast) | `R019` / `R01A` |
| S064 | Lock On | `LockOnCast` | `R01Z` / `R020` |

Toggle display names (JASS `AbilityToggle`): Splitter Target/Area, Meteor Meteorite/Magma, Wind Walk Charge/Invisibility, Repeater Target/Area (not wired, see Repeater), Teleport Blink/Multiport, Thrust Strike/Ghost, Swap Displacement/Relocate, Drain Slow/Weakness, Fire Spray Stream/Cluster, Bouncer Bounce/Recharge, Gravity Dark Matter/Force Field, Link Hook/Induction, Disable Entangle/Silence, Control Enemy cast/Self cast; `M042` "Arc direction" toggles `toggleD` but **no cast code reads `toggleD`** (the toggle is a no-op in 0.99c, uncertain).

---------------------------------------------------------------------------------------------------

## 4. Attack spells

Notation: damage tables list ability level 1..n (n shown in w3a). Cooldowns are the ability cooldown in seconds (`acdn`). Mana cost = 0 for every spell (`amcs`); no cast range limits (`aran` 99999) except where noted; cast animation `acas` 0 for all (except F-spells 0.7).

### 4.1 Fireball (G) - `S000` - `FireballCast`, `FBColl`, `FBEndRng`, `FBDestr` (JASS 11442, 11358)

| Item | Value |
|---|---|
| Missile | speed **1000 u/s**, lifetime `1 + 0.12*r` s (= 1000 u at r=0), `Obj_Size` 25, collides with players/missiles/pillars, model `fbnosound.mdl` (`fb2nosound.mdl` tinted when custom colour `-fbc`) |
| Damage on warlock | `6.5 + 0.5*LevelG` (w3a: 7.0 at L1 ... 12.5 at L12). `Obj_Real` = 6.5 base (`ScrollFBBonus` raises it) |
| Knockback | `kb = 1` via `DamageTargetNormal` |
| Cooldown | **4.8 s** all levels (`w3a S000 acdn`) |
| Level | starts at 1; 10 upgrades (see 1.5); `R002` gold 3 |
| Hits pillar/shard | same damage kb 1; fireball vs missile: both destroyed (fireball only dies, damage to shards) |

Special / item behaviour:
* **Staff of Fireball** (item `h006`, `Allitems[13]`): `Consecutive[id]` starts at 1 and the ability gains +12 levels (levels 13-24 in w3a): base damage becomes `dmg - 0.7 + 0.4*LevelG` (6.2 + 0.4*(L-1)) and a burn buff (`BUFF_TYPE_FBALL`) with total `(0.9+0.1*LevelG) * D` over `D = TI*TD*(1+0.5*Consecutive)` seconds; `Consecutive` counts consecutive hits (increments on hit, resets to 0/1 on a miss i.e. `FBEndRng`, and shows a purple counter / "miss" text).
* **Scroll of Fireball** (item `h00H`, `Allitems[27]`): `Obj_Bounce` 1.2 - fireball bounces off pillars; each bounce adds `ScrollFBBonus` (1) to `Obj_Real` damage (and speed boost described in the item text).
* **Aegis** (`Allitems[17]`): a warlock hit by your fireball charges the owner's Aegis (`AegisReady`).
* **Mirror clones** also fire: each clone fires an identical weak fireball with damage `0.5 + 0.5*LevelY` and `kb 2/(0.5+0.5*LevelY)` (`FBMColl`, speed 1000, size 25).
* **Ghost game mode hidden player**: uses `IceCast`: frost-wyrm model, size 30, damage `1.2*(6.5+0.5*lvl)`, freezes the target in place for `TI*TD*(3.0+0.5*Consecutive)` seconds (target kb multiplier x0.5 and lava damage x0.5 while frozen).
* Fireball stats counters `FB_stats` (casts/hits) and kill-streak text only matter for the league/stat screen.

### 4.2 Lightning (D) - `S002` - `LightningCast`, `LightningMake` (JASS 13336, 13488)

* **Instant ray** (no projectile): from the caster toward the target, half-width `w=50`, length `545*(1+0.12*r)`.
* Damage `6.4 + 0.6*LevelD` (w3a 7.0 ... 11.8 at L9), cooldown 16, 15, 14, 13, 12, 11, 10, 9, 8 (L1-9). Level cap 9 (5 upgrades + Jordan 2 + 1 start).
* Hits the **first** thing along the ray: warlock (`DamageTargetPoint` kb from the ray origin with `GetAngleFactor` = 0.95 unless the target is moving toward the caster, then `95/(100-0.6*v*cos)` (a bigger push)), pillar (kb 0.92), or an enemy missile (the missile is destroyed through its `OnDestroy`; Magma and Repeater are not destroyable). The damage dealt returns as a heal to the caster scaled by life steal.
* **Shield interaction**: the ray reflects off a shield sphere (radius 120) and continues from the contact point with `damage-1` while `damage > 2.1` and the remaining length > 100 (`LightningMake` recursion).
* Own-team bonus: ray striking **your own fireball** destroys it and detonates `LGH_FB`: AoE `2*(damage + 6.3 + 0.7*LevelG)/3` radius 225 bonus 0.6 kb `3*damage/(damage+6.3+0.7*LevelG)/2`. Ray striking an allied **Magma** boulder charges it (becomes a bigger exploding bolt, lifetime 1.2 s, size `sqrt(size²+1000)`, +LevelD spell level).
* Lightning type "CLSB". Cast hotkey D.

### 4.3 Homing (D) - `S003` - `HomingCast`, `HomingColl/Coll2/Seek/Accelerate` (JASS 13875)

* Normal mode (not thrusting): slow homing missile `HomingMissile.mdx`, scale 1.85, size 34, ally+owner collision on, speed `800 + max(0, forward component of your velocity along the aim)` u/s, lifetime `4.5*(1+0.12*r)` s. Spawns 4+size units in front of the caster. Targets the **closest enemy to the aim point** (`TX,TY`), and each tick steers: `v = 0.98*v + 0.02*(1000 u/s toward target)`; if there is no valid target it uses `HomingSeek` (retargets nearest enemy). Tracks `Obj_Real` = flight time.
* On enemy warlock: damage `HomingDamage = 6 + 0.6*LevelD + min(flightTime/2, 1)` (so up to +1), kb 0.95; destroys. On pillar: same damage and an extra line damage `0.75*(6+LevelD)` at kb 0.95 in a 110-wide, 200-long line behind (`HomingPillarCollision`).
* **Hitting your own homing missile** (owner collision): AoE damage around it (`DealCustomAoE` radius `200*1.46`, enemies only) with `HomingDamage`, kb 0.9 and grants the caster **+50 move speed for 4*TI s**; if the caster is mid-thrust: radius `200*1.54`, **+120 move speed**, "Burn out" text when it hit someone.
* **Thrust mode**: if `P_Thrust[caster]` the cast instead fires a straight fast bolt (`ZigguratMissile`, speed 1300, lifetime `1+0.12r`, size 34): damage `6 + 1*LevelD` (`DamageTargetLine` kb 0.95) and applies `ReduceRangeInitiate` (victim range-mastery -3 for 7.5*TI*TD s).
* Cooldown 14, 13, ..., 6 (L1-9), damage tooltip "7 + range" (7.0 + 0.6/level + flight bonus).
* Linking: `P_TransferLinkObj` = this missile when a Link is attached so the caster's Link can transfer onto the missile (also Boomerang, Arc, Phoenix).

### 4.4 Boomerang (D) - `S004` - `BoomerangCast`, `BoomerangColl/Move` (JASS 14050)

* Model `ArcaneGlaive.mdl`, size 38, collides with players and pillars (not missiles), bounce 1. Launch speed 1500 u/s forward + 300 u/s sideways (curved path). Travels distance `d = clamp(dist to target, min 300, max 800*(1+0.12*r))` with constant deceleration `a = -v²/(2d)` (so it stops at the target distance after `t = -v/a` s), then accelerates back (`Walk` acceleration = reverse of the outgoing, `BoomerangEndRng1` sets `Acc = Walk`), finally `BoomerangMove`: homes to the caster at 1000 u/s with steering 30*0.03/tick, destroyed within 75 units of the caster (or after 1.5 s if the owner dies).
* Spawns at the nearest pillar's edge if the caster stands within `38+35(PillarSize)` of a pillar.
* Damage `(6.4 + 0.6*LevelD) * (1 + 0.1*hitsSoFar)` and kb `0.95/(1+0.1*hitsSoFar)` (`Obj_Int` counts hits); per-target hit cooldown 0.3 s (`boomhitcd`); a hit bounces it back (velocity `-500/100*dir`) and it switches to the return phase. Blocked (reflected) by shields (not reflected: stays owned).
* Cooldown 15, 14, 13, 12, 11, 10, 9, 8, 7 (tooltip level 8 shows 6 but the ability table has 8 -> uncertain), damage 7.0 ... 11.8.  Shield/Dome do not change a Boomerang's owner; they force it into its return phase (`Obj_OnIteration = Boom_Iter`).

### 4.5 Arc (D, internal Phantasm) - `S054` - `ArcCast`, `PhantasmColl` (JASS 13587)

* Fires **two curved missiles** (`PhantasmMissile.mdl`, scale 1.85, size 46, lifetime 1.1 s) from 50 units in front of the caster, one curving clockwise and one counter-clockwise; aimed so the two arcs meet around the target point (range capped at `800*(1+0.12r)`). Curve via `Phantasm_Acc1/2` (rotates the velocity vector each tick by a precomputed angle `Obj_Real = 1 - 2q²`, speed x1.02/tick).
* Damage `3.9 + 0.3*LevelD` (4.2 at L1 ... 6.3 at L8) to the first warlock/pillar hit; knockback factor `Obj_Real2 = 0.5` if the target point is < 250 away else `1.1`.
* If **both missiles hit the same warlock within 1 s** (`ArcHit/ArcHitTarget` with 1 s timer): the second hit applies the Arc slow buff (50% move speed, duration `2*TI*TD` s) (`BUFF_TYPE_ARC`).
* Cooldown 14, 13, ..., 6. Tooltip says level 9 "missing tooltip". Linkable.

### 4.6 Poison Bolt (D) - `S058` - `BoltCast`, `BoltColl` (JASS 13164)

* Model `ChimaeraAcidMissile`, scale 1.1, speed 1000 u/s, lifetime `1.1*(1+0.12r)`, size 38.
* Hit on warlock: damage `6 + 0.6*LevelD`, kb 1; plus the Bolt DoT: total `(0.4+0.1*LevelD) * 3*TI*TD` over `3*TI*TD` s (= `0.4+0.1L` per second; w3a "DPS 1.2 ... 3.6" is **uncertain/stale**: 3x(0.4+0.1*(L-1)) matches it, code uses `L`). **Slow 50% from 2 s to 3 s after impact** (done in `Buffs_Iteration`).
* Missile vs missile: both die; vs pillar/shard: damage.
* Cooldown 14, 12.5, 11.5, 11, 10.5, 9.5, 9, 8.5, 8.

### 4.7 Shock (D) - `S062`, action `A02G` - `ShockCast`, `ShockEndRng`, `ShockCharges` (JASS 13021)

* Tooltip (wts): "Cast a shock bolt that explodes at the target point, dealing AoE damage. This spell has two charges. Targets hit are shocked for 3 seconds. Shock deals additional damage to shocked targets. Damage 4-5, Charges 2, Cooldown 7 per charge, hotkey D". The "shocked for 3 s / extra damage" part is **not implemented in 0.99c code** (only declared `ShockedEffect`), uncertain.
* Orb `OrbOfLightning.mdx` scale 0.5, speed 800 u/s, size 38, flies to the target point (lifetime `min(dist, 800*(1+0.12r)) / 800`), collides only with missiles and pillars (destroys itself, damages pillars/shards `5 + 1*LevelD`). At the end of its path it explodes: `DealAOEDamage(4 + 1*LevelD, kb 1.2, radius 120, bonus 0.8)`. **Pressing the spell again (`A02G`, temporary ability) detonates it early**, kb 1.1, same radius.
* Two charges: `Shock_Charges` starts 2; casting uses 1 and starts a 0.03 s timer; +1 charge after **8 s** of accumulated timer when 0 left, second charge after **16 s** total (so effectively 8 s per charge; tooltip says 7). Ability cooldown resets (`ResetSpellCooldown`) when a charge is available and no orb is in flight.

### 4.8 Meteor / Magma (E) - `S008`

**Meteor (toggleE false)** - `MeteorCast/Initiate/EndRng` (JASS 15347):
* Meteor object (unit `'e001'`, no collision, fly height 700) is created 400 units short of the target along the cast direction, visible 0.55 s then a 0.6 s fall to the target (`MeteorInitiate` scales velocity x100 to hit the target point; a `DominationAura` marker shows the landing spot).
* On landing: for every enemy warlock and every pillar/shard: `dmg = maxdmg - (dist/85)²` with `maxdmg = 10 + LevelE` (pillars `/1.4`); applied if `dmg > 3.5 + 0.5*LevelE + flatReduce`; `DamageTargetPoint(kb 0.66)`; the caster is healed with the lifesteal of total damage dealt. Radius (where dmg exceeds the minimum) = `85*sqrt(6.5+0.5E)` = 225 at E=1 ... 318 at E=8 (w3a text). Impact scale/explosion `ExplosionBIG`. Damage range 6-11 ... 6-18 (tooltip).
* Cooldown 20, 19, ..., 13 (L1-8).

**Magma (toggleE true)** - `MagmaCast`, `MagmaColl`, `MagmaDestr` (JASS 15426-15577):
* Boulder `BallsOfFireMissile`, size `72 + 2*LevelE` (scaled model), speed **400 u/s**, lifetime `1.43*(1+0.12r)` s (~572 u), bounce 0.75, collides with everything. `ID_Magma` spell type; **absorbs enemy missiles** (grows `size = sqrt(size²+1000)` per absorbed missile, destroyed on touching another Magma/Repeater).
* Touching a warlock: DoT buff `BUFF_TYPE_MAGMA` total `0.5+0.3*LevelE` over 0.25 s (halved if that warlock is linked to you) refreshed each tick, plus 50% slow, plus a push along the boulder's walk direction toward `425 u/s` speed (`Obj_WalkX/Y`). **Pancake**: if the touched warlock is stunned (entangled/dagger-stunned/etc.) the first time: `P_DamageMultiplierInflict *= 0.5` (and `P_DamageMultiplierTake *= 1.2` in non-Magma variant) for `2.8*TI*TD` s, text "Pancake", then follow-up DoT `0.3+0.3*LevelE` over 0.2 s while pancaked.
* Hitting a pillar: explodes (`MagmaNoBolt_Destr`) with direct `6 + 2*LevelE` damage kb 0.8 to the pillar.
* On death (range end or pillar): explosion AoE `DealCustomAoE(3 + 1.5*LevelE, kb 0.8, radius = 200*AoE)` where `AoE = sqrt(128+2*size)/14.142` (so grows with absorbed missiles); lightning-charged variant (`MagmaBoltDestr`) chains a lightning ("CHIM") to the closest warlock within `400*(1+0.12r)*1.25`.
* Cooldown (mode B) 20, ..., 13; tooltip "DPS factor 0.6 ... 2.7, Max AoE damage 4.5 ... 15".

### 4.9 Splitter (E) - `S009` - `SplitterCast` (JASS 15992)

* **Target mode (toggleE false, "Target")**: orb `SpiritOfVengeanceMissile` scale 2.5, size 50, speed 700 u/s, flies to the target point (max `700*(1+0.12r)`). On hit/end it splits into **two children flying at +/- angles** (`SplitterExtract`): a mini-orb of lifetime 0.48 s (`Obj_Range = 0.12*4`, speed 250 u/s) whose `SplitterIterB` fires two bolts every 0.12 s (lifetime `1.2*(1+0.12r)`, speed 600 u/s, size 21, pairs at angles `+/- Obj_Range` so they fan out). Each bolt hits for `2.5 + 0.5*LevelE` (3.0 ... 6.5) kb 0.65. Main orb touching a warlock deals 3 damage kb 1.4.
* **Area mode (toggleE true)**: a slow orb (speed 280 u/s, lifetime `700*(1+0.12r)/280` s) that every `Period5` (0.18 s) emits 2 bolts in opposite directions rotating 30 degrees each time (speed 750 u/s, lifetime 1 s, size 20, same damage) - a spinning spray. If the target is < 128 units away it stays put with 1.1x lifetime.
* Cooldown mode A 25, 23, ... 11 (L1-8); mode B 29, 26, 24, 23, 22, 21, 20, 19. Splitter bolts are `ID_Splitter` (immune to shield/dome reflection).

### 4.10 Wind Walk (E) - `S010` - `WWCast`, `WWCast2`, `PlayerColl` (JASS 16028, 5591)

* **Charge (toggleE false)**: `S_WW`: **+200 move speed for `2.6*TI` s**, ghost walk (`Agho` ability), collision size +0; on colliding with an enemy warlock the effect ends with a **backstab**: damage `4.6 + 0.8*LevelE` (5.4 at L1; kb 1) and `RoarCaster` effect. Cooldown 30, 27, 24.5, 22, 20.5, 19.5, 18.5, 17.5.
* **Invisibility (toggleE true)**: `S_WWW`: +66.7 move speed (`WW_MS/3`) for `3.6*TI` s, invisible; contact with an enemy (`WWW_hit`, 0.2 s per-hit cooldown): knockback hit (kb with `75/(100+DP)`), life steal hit `2.6 + 0.2*LevelE` (counted with +5 life steal levels: tooltip 2.6 ... 4.0), and after the first hit grows to +200 speed for the rest of the duration; own knockback receipt `2.5` kb. If thrusting it instead deals `5 + 0.4*LevelR`. Cooldown 26 all levels.
* Both modes are cancelled by casting any other spell (`AbilityActions` start).

### 4.11 Repeater (E) - `S030` - `RepeaterCast` (JASS 15789)

* Places a turret `RepeaterMissile.mdx` at the target point but at most **100 units** from the caster (clamped), lifetime `2.8*TI` s, size 1. Every `Period5` (0.18 s) it fires a child missile from its own position **away from the caster** (direction caster->turret), child speed 1000 u/s, lifetime `0.9*(1+0.12r)` s, size 25, damage `max(2.4, 2.7 + 0.3*LevelE - RepeaterDmgReduction)` (+0.2 reduction accumulates per warlock hit) kb 0.60, destroyed on first collision.
* Cooldown 26, 23, 21, 20, 18, 16, 14, 12. Tooltip damage 3.0 ... 5.1. The level 11-18 (mode B) tooltips are copy-pasted Magma text and the toggle is not read by `AbilityActions` -> **uncertain**.
* Destroys itself when touching Magma/Repeater.

### 4.12 Burst (E) - `S055`, action `A02F` - `BurstInitiate`, `BurstCast`, `BurstIter` (JASS 15608-15714)

* First press: starts a **channel**: the caster stops, `Burst_Active`, `Burst_Channel`, a text tag shows `50*chargeTime`, charge time grows +0.03/tick up to **2 s**; window of **5 s** to release (second press `A02F`). The original ability is replaced by `A02F` ("Burst Action") during the window.
* Release: a missile `FireWave.mdx` (scale 0.3, size 25) of speed `800 + 100*chargeTime` u/s (800-1000), lifetime `1.1*(1+0.12r)` s, damage `5.5 + chargeTime + 0.5*LevelE` (6.0-8.0 at L1 ... 9.5-11.5 at L8) kb 0.7 (kb 1 vs shards). Dies on any collision.
* A **move command during channel cancels and flings the caster**: `Vel += 16 * direction` (a 16-unit/tick dash, `Trig_Untitled_Trigger_100_Actions`) while channeling is ended.
* Cooldown 27, 25, ..., 13.

### 4.13 Drain (T) - `S014` - `DrainCast`, `DrainColl`, `DrainColl2`, `HealMissileIter` (JASS 11663)

* **Slow mode (toggleT false)**: skull missile `Drain.mdx`, size 24.5, speed 675 u/s, travels to the target point (max `675*(1+0.12r)` u), bounce 1, ally collision on. Hit enemy: damage `5 + LevelT` kb 0.2, Drain buff slow (18% of base move speed `0.18*KBmult*MS`, duration `(3.2+0.4*LevelT)*TI`) and **steals life**: spawns a heal orb `FaerieDragonMissile` that flies to the caster and heals `(dmg-flat)*inflict*take*(1+0.10*LevelMelee)`. Hit ally (including self): heals `5+LevelT` and gives them **+50 move speed** for the same duration.
* **Weakness mode (toggleT true)**: bigger translucent skull (size 35, speed 850 u/s, lifetime `1+0.12r`, owner+ally collision, `Drain2_Coll`): enemy: damage `4 + LevelT` kb 0.6 and **-50% damage inflicted** on the victim for `5*TI*TD` s (`P_DamageMultiplierInflict*=0.5`), lifesteal heal scaled by the victim's weakness. If it hits an ally/self (not self-heal): heals `4+LevelT` and gives that ally `+10% damage inflicted` for `5*TI*TI2` s, then **re-aims at the nearest enemy** (speed 550*0.85 u/s, lifetime `(1+0.12r)*7/6`).
* If it misses (range end) it re-targets the nearest object (`DrainEndRng`) at speed 600 u/s for `(1+0.12r)*7/6` s (`ID_Drain` second life). Cooldown 22, 21, ..., 15 (both modes). Damage tooltip A: 6 ... 13, duration 3.6 ... 6.4; B: 5 ... 11, duration 5.

### 4.14 Fire Spray (T) - `S015` (also ids `S15B`, `S15C`) - `FlameClusterCast`, `FlameCastPeriodic` (JASS 11811, 11750)

* **Stream mode (toggleT false)**: schedules `5 + LevelT` missiles, one every 0.08 s (a sweep: angle offset `0.40 rad * (n-2)/N`, n counting down, so the stream sweeps across a cone from about +0.4 rad down to a small negative angle). Each missile: speed **1200 u/s**, size 22, lifetime `(1+0.12r)/1.2` s (~833 u), scale 0.73, damage `2.3 + 0.3*LevelT` on warlock (1.2 + 0.3L on pillars), kb 0.6 (`DamageTargetLine`). Tooltip L1 2.6 dmg, 6 missiles ... L8 4.7 dmg, 13 missiles.
* **Cluster mode (toggleT true)**: 4 missiles at once, angles `6.9 deg * (i - 1.5)` around the aim, speed **800 u/s**, size 22, lifetime `760*(1+0.12r)/900` s, damage `3.3 + 0.3*LevelT` (pillar 2.0+0.3L) kb 0.65.
* Cooldown mode A 15.5, 14.75, ... 10.25; mode B 16.5, 15.5, ... 9.5. Ends the Control remote cast on the last missile when cast from a remote marker.

### 4.15 Bouncer (T) - `S016` - `BouncerCast`, `BouncerColl`, `Bouncertarget` (JASS 12039, 11964)

* **Bounce mode (toggleT false)**: missile `IllidanMissile` size 38, speed 900 u/s, first lifetime `1+0.12r` s, bounce 1, `ID_Bouncer`. When it hits a warlock it deals `Obj_Real * (4.6 + 0.8*LevelT)` damage (`Obj_Real` starts at 1, x0.7 per hit; **30% reduction per hit** in code while tooltip says 20%) kb 1.2, then **retargets the next nearest other warlock** (velocity solved with a lead calculation `Pow(Real,0.25)*900 u/s`, lifetime `(750+150*LevelT)*(1+0.12r)/900` s, can hit allies/owner after the first hit). When `Real <= 0.2` and lifetime > 0.75 s it only knocks back (kb 0.8). Stops when damage < 0.1. Dies on hitting another missile of an enemy team / pillar.
* **Recharge mode (toggleT true, `RechargeCast`)**: orb `BouncerRecharge.mdl` scale 0.1, size 35, speed 900 u/s, lifetime `1+0.12r`, bounce 1: on enemy damage `(5.3 + 0.7*LevelT)*(1+0.05*Consecutive[id+12])` kb 1.15 and the hit refunds: an orb (`DevourMagic` birth model) spawns on the victim, flies back to the caster at 12 units/tick (0.03 s) and on arrival **resets the cooldown of the caster's T-slot spell** (`BouncerRecharge` removes/re-adds `SpellsAcquired[8*id+4]`, i.e. the Bouncer itself). Consecutive-hit counter `Consecutive[id+12]` (+5% damage each) is reset on miss/missile hit.
* Cooldown mode A 20, 19, ... 13; mode B 22.5, 21, 20, 19.5, 19, 18.5, 18, 17.5. Damage tooltip A 5.4 ... 11.0; B 6.0 ... 10.9.

### 4.16 Dagger (T) - `S061` - `DaggerCast`, `DaggerColl`, `DaggerColl2` (JASS 12209)

* `SpinningKnife.mdx` scale 2, size 38, speed **1000 u/s**, flies to the target point (max `800*(1+0.12r)`), collides with players and pillars (not missiles). On first contact: bleed buff `BUFF_TYPE_DAGGER` total `(0.4+0.1*LevelT)*3*TI*TD` over `3*TI*TD` s (= `0.4+0.1L` /s; tooltip L1 1.5 = 3 s x 0.5) and then **turns around** (if a different enemy is within 100 units at end-of-range it extends; otherwise return). Return flight: speed 1000 u/s toward the caster, size 57, owner+ally collisions: hits for `6 + 0.6*LevelT` (6.6 ... 10.8) kb 1 and **stuns for 0.8 s** (move speed 0, `P_Stunned`, effect), plus the bleed again; destroyed on reaching the caster / any pillar.
* Cooldown mode A 20, 19, ..., 13; w3a level 11-18 reuse mode-B numbers (unused; no toggle for this spell).

### 4.17 Phoenix (legacy) (`Phoenix*` functions at JASS 14815-14915)

`PhoenixColl`, `PhoenixComboColl`, `PhoenixEnd`, `PhoenixfireEnd`, `CancelPhoenix` exist (fire missile `4.5+0.5*lvl` damage kb 1.1 with a 6*TI*TD s "phoenix fire" DP curse and heal orb) but **nothing in `AbilityActions` or `InitTrig_Spells` references them** -> dead code in 0.99c. Ignore for the remake (uncertain: a hidden callers may exist in a removed trigger).

---------------------------------------------------------------------------------------------------

## 5. Movement spells (R)

All use `LevelR` and the `R` hotkey. If the caster escapes lava while at lethal HP (`HP <= 6*lavaDmg+0.5`) a "Last Second Save" text/sound plays (cosmetic). They call `ActivateLavaTreads` if the item is owned and the landing spot is lava. Common: cast clears walk (`IssueImmediateOrder "move"`), kills velocity partly.

### 5.1 Teleport (R) - `S011` - `TeleportCast`, `MultiportCast` (JASS 15247, 14991)

* **Blink (toggleR false)**: instantly moves to the target up to **`700 + 70*LevelR`** units (770 at L1 ... 1330 at L9), keeps 80% of knockback velocity (min velocity 12/tick if velocity is smaller, tooltip "reduces knockback by 20%"), leaves a fading ghost model and effects. Cooldown 16, 13.5, 11.5, 10, 9, 8, 7, 6, 5.5 (w3a `acdn` ends 5, tooltip 5.5).
* **Multiport (toggleR true)**: first jump up to `300 + 50*LevelR` (350 ... 750), then for **3.3 s** a secondary ability `S029` (range 300, hotkey R, `TeleSecondCast`) allows a second jump of up to 300 units (`TeleTo(..., 300)`); the primary ability is hidden for those 3.3 s. Cooldown 18, 15, 12.5, 10.5, 9, 8, 7, 6, 5.

### 5.2 Thrust (R) - `S012` - `ThrustCast`, `PlayerColl`, `ThrustEnd_1/2/3` (JASS 16056, 5448)

* **Strike (toggleR false)**: caster becomes `P_Thrust`: initial velocity `0.77*ThrustVel` (ThrustVel = 1300 u/s = 39 u/tick) accelerating by `ThrustAcc = ThrustVel*(1-0.96)` per tick toward the target direction; lifetime `Obj_Range = dist/ThrustVel*0.03 + 0.09 s`, distance capped at `900*(1+0.12r)` (tooltip range 700). Size +30 for the dash, collision as Player_Coll. Hits the first enemy warlock: damage `(5.0 + 0.4*LevelR) * min(1, v²/(39²)/0.8)` (full damage if speed ratio > 0.8, scaled down at low velocity), kb 1.0; both dash ends (`ThrustEnd_2`: velocity zero, "ThunderClap" effect). If both players thrust into each other: both take `5+0.4*LevelR` and stop (`ThrustEnd_3`). Ends with velocity x0.2 at range end. Blocked by enemy shield: no hit (`return false` when target has a shield).
* **Thrust + Wind Walk** combos and **Lock On** combos are in `PlayerColl` (Wind Walk charge during a thrust deals `4.8+0.6*LevelE + ...`, see 4.10 / 7.5). If the victim is stunned (entangled), thrust hit performs "Lock On": `SearchExec(... 'en74' ...)` ends the stun and `ThrustRecharge` resets the thrust cooldown.
* **Ghost (toggleR true)**: `GhostCast`: ethereal thrust: no collisions for the duration (`Obj_Collision` all false, alpha 56), initial `0.9*0.77*ThrustVel`, `Acc = 0.55*ThrustAcc`, `Obj_Range = 999999999` - lasts until the caster issues a point/target order (`CancelPhoenix` trigger via `RegisterMove`; `PhoenixEnd` finishes it). Passes through enemies and obstacles.
* Cooldown mode A 16.5, 14.5, 13, 12, 11, 10, 9, 8, 7; mode B 16, 14, 12.25, 10.75, 9.5, 8.5, 7.5, 6.5, 5.5. Damage tooltip 5.4 ... 8.6.

### 5.3 Swap (R) - `S013` - `SwapCast`, `SwapColl`, `RelocateCast` (JASS 15075, 15108)

* **Displacement (toggleR false)**: missile `ShadowStrikeMissile` speed **1700 u/s**, size 40, ally-collision on, lifetime `min(dist, 800*(1+0.12r))/1700`, hits players and pillars. On the first object hit (ally, enemy, or pillar; blocked by enemy shield; an `Obj_Terrain==2` platform caster cannot swap into non-platform targets) the **caster and the target exchange positions**; both are stopped; enemy victims get `P_DamagingPlayer = caster` (so a lava death credits you). If it ends in the air (`SwapEndRng`) the caster teleports to where it ended.
* **Relocate (toggleR true)**: finds the closest non-player, non-`Grav` object (own or any missile/pillar) within **290 units** of the target point (`r < 84100`, `84101` initial) and within 1500 units of the caster. Caster teleports to the target point and the object is moved by the same offset/swapped with caster (missiles are moved to the caster's position `+dx,dy`; pillars (type 3) are not moved; the caster lands next to the pillar). Grants **+50 move speed**: 3.5*TI s if the object was allied, 4.5*TI s if enemy. If it was your own boomerang/homing/arc the link transfer target changes. If no object is in range the cooldown is immediately reset (`ResetSwapCD`).
* Cooldown mode A 15.8, 13.3, 12.3, 9.8, 8.8, 7.8, 6.8, 5.8; mode B 13, 10.5, 8.5, 7, 6, 5, 4, 3.

### 5.4 Return (R) - `S059` - `ReturnCast` (JASS 15201)

* Teleport up to `550 + 50*LevelR` (550 ... 1000 at L9) to the target point, **unless the caster was standing on lava** (then it is a normal teleport with no return). Otherwise: for **1.5*TI s** the caster gets **+10% damage inflicted** (`dmgmodifier 1.1`), then (`ReturnEnd`) is **teleported back to the original position** and becomes immortal for 0.5 s (`Immortalize2`). Effect `ManaFlare` while active.
* Cooldown 16, 13.5, 11.5, 10, 9, 8, 7, 6, 5.

---------------------------------------------------------------------------------------------------

## 6. Defence spells (C hotkey, `LevelY`)

### 6.1 Rush - `S007` - `RushCast`, `Rushgain`, `RushEnd` (JASS 3954)
* For `(5.5 + 0.5*LevelY)*TI` s (6.0 ... 9.5): **+35 move speed immediately**, and incoming damage is partly absorbed: reservoir `rushdeflect = 3 + 2*LevelY` (5 ... 19). Each hit of value `v`: if `2*deflect > v` then half the damage is absorbed (`v/2` reduces reservoir) else absorb the whole reservoir; **every absorbed damage point gives +15 move speed** (accumulated in `rushbonus`). On end the bonus speed is removed and a red text shows the accumulated bonus. Dispels links. Cooldown 21, 20, 19, 18, 17, 16, 15, 14 (L1-8; w3a defines 10 levels, ending 13, 12). While Rush is active pending `CastingDamage` is not added to DP (`not rush[TCasterObj]` check in `AbilityActions`).

### 6.2 Shield - `S005` - `ShieldCast`, `ShieldColl` (JASS 12813)
* Creates a type-3 object glued to the caster (`ShieldIter` copies its position and velocity), radius `90 + Obj_Size[caster]`, collides with missiles and players? (flags: 3+3 false, 3+2 true, 3+1 true -> missiles and players), **IgnPriority 2**. Duration `2*TI` s (fixed 2). Incoming missiles whose velocity points toward the shield are **reflected** (`vel -= 2*(v·n)*n`), moved out by 5 units, lifetime x0.9, and **their ownership changes to the shield owner** (except Boomerang, Link, Silence): `SetUnitOwner`, colour change, and `SpellLevel -= 1`; Magma/Burst/Repeater/Dagger are instead destroyed; Bouncer/Homing lose their lock (`Obj_Int=0`). Immune: Splitter, Gravity-type objects, pillars; an enemy Swap missile passes through. Opposing thrusters bounce too. Lightning reflects. Dispels link.
* Cooldown 25, 22.5, 20.5, 19, 18, 17, 16, 15, 14.

### 6.3 Time Shift - `S006` (+ action `S047`) - `TimeShiftCast`, `TS_Effect`, `TS_EndEarly` (JASS 12706)
* On cast: a ghost clone unit (`copy_object = id+500`) records position, velocity, acc, walk, thrust state, HP and DP, facing. Ability swaps to `S047` ("Time Shift Action": instantly ends it, only allowed after 0.2 s). After `3.6*TI` s (tooltip says 3.5) `TS_Effect` rewinds: position/velocity/range and thrust restored, **HP set back to the saved value if it is higher**, **DP restored partially**: `DP = DPrec + (0.8 - 0.1*LevelY)*(DPnow - DPrec)` (you still take 70% of the DP gained at L1, 0% at L8), buffs cleared (`Buffs_ClearObjBuffs`, `ClearLinksOnObj`), stuns ended. A death that happens during the window triggers `TS_Effect` instead of dying (`PObj_TS_Action`) - the rewind is the "save".
* Cooldown 23, 20.5, 18.5, 17, 16, 15, 14, 13. If a previous Time Shift is pending the cast is aborted with a message.

### 6.4 Mirror - `S048` - `MirrorCast`, `MirrorMoveCast`, `MirrorMove`, `MirrorEnd` (JASS 12425-12634)
* Two-step: casting only **arms** it for `3.0*TI` s (effect `HowlTarget`, curses cleared, links cleared); the **next move (point) order** triggers `MirrorMoveCast`: two clones (unit type `'k000'` offset by model type) appear at `167` units, 120 degrees apart from the move direction on both sides, the real warlock jumps 167 units along the move direction, all three are indistinguishable (the real one is made invisible `Aloc`/hidden locust and the owner still controls it, clones copy every order with offset `MirrorMove`). Duration `3.6*TI` s, +25 move speed, damage taken: if any of the three is hit the mirror breaks (`Obj_Mirror` -> `SearchExec 7337`). Fireballs and other spells fire from all three positions (see 4.1) - fireball clones deal `0.5 + 0.5*LevelY`. Mirror also silences the scourge for 1.5 s (`SilenceEntry('W007',7,...,1.5)`) at creation unless king/hidden. Dispels links and curses.
* Cooldown 17, 15, 13, 12, 11, 10, 9, 8 (L1-8; w3a has 10 levels, last 7, 6).

### 6.5 Vector - `S056` (+ action `A02J`) - `VectorCast`, `VectorIter` (JASS 12303)
* Marks a circular zone at the caster for 6 s (`Obj_Range 6.0`, no TI scaling; effect `UnholyAura`, scale 3, radius threshold **225 units**). Ability swaps to `A02J` ("Vector Action": end early). When the warlock *leaves* the zone (distance > 225) `VectorIter` triggers once: his velocity is reversed to point back toward the marker with speed `max(current speed, 16 u/tick)` (reverses knockback), **heals `0.016*maxHP*sqrt(speed-16)` (min 2)**, makes the warlock ethereal for `0.5*TI` s (no missile collisions, alpha 56) and clears links, then the zone ends. Dispels link. Cooldown 20, 19, 17, 16, 15, 14, 13, 12, 11, 10, 9 (L11) and a 22.5 variant at level 12 (w3a); tooltips differ: level 1-10 includes the ethereal + heal text, level 12 "Reverts knockback upon leaving" (uncertain which is shipped).

### 6.6 Dome - `S057` - `DomeCast`, `DomeColl` (JASS 12398)
* Creates a static (it keeps the caster's walk velocity at creation) barrier object `MassTeleportTo` scale 1.5, radius **150**, lifetime `3.6*TI` s, collides with everything, `IgnPriority 2`, `ID_Grav` type. Objects whose velocity points into it are slowed (`v = 0.5*(v - 2(v·n)n)`, moved out 5 units, lifetime x0.9), **ownership changes to the dome owner** (not Boomerang, Link, Silence) with `SpellLevel = 0`; Boomerang -> return mode; Magma/Burst/Repeater destroyed; Bouncer/Homing lose lock. Enemy players are pushed only if thrusting; a "Farseer" effect pulses (Period5). Immune: Splitter, Gravity, an enemy Swap missile, pillars. Dispels link. Cooldown 23, 21, 19, 18, 17, 16, 15, 14, 13, 12 (10 levels; "Tooltip missing" at the last).

### 6.7 Ethereal Form - `S060` (+ action `A02H`) - `EtherealCast`, `EtherealColl/Iter/End` (JASS 12886)
* Caster becomes ethereal for `2.4*TI` s: all `Obj_Collision` off for the warlock (alpha 56, passes through players, missiles, pillars), **+200 move speed for 1.2 s**; an invisible helper missile (type 2) tracks the warlock and **adds +1.2 s of the speed bonus every time it passes through an enemy warlock** (`EtherealColl`; speed can be re-applied once at a time). Ability swaps to `A02H` ("Ethereal Action": end early). Casting any F spell (Scourge, Bane, ...) cancels it (`InitTrig_Ethereal_Cancel_Scourge` on channel of `S023,S051,S021,S020,S022,S001,S100`). Dispels link. Cooldown 23, 21, 19, 17, 16, 15, 14, 13, 12, 11.

---------------------------------------------------------------------------------------------------

## 7. Curse / control spells (Y hotkey, `LevelH`)

### 7.1 Link (Y) - `S019` (+ action `S031` "Link Action") - `LinkCast`, `AdhesionCast`, `LinkColl`, `LinkInductionColl`, `InductionMakeLine`, `ExtendInduction`, `LinkEnd` (JASS 14639, 14663, 3694)
* Link is a missile that, on contact, ties the caster to the target with a lightning line; while linked the ability is replaced by `S031` whose level selects what the second press does (level 1 = pull back/end quickly, 2 = make induction line, 3 = extend induction, 4 = cancel link).
* **Hook (toggleY false)** `LinkCast`: `FarseerMissile`, size 35, speed **950 u/s**, lifetime `860*(1+0.12r)/1000` s... precisely `Obj_Range = 860*(1+0.12r)/1000` s at 950 u/s (~817 u), collides with players and pillars, ally collision on. If it hits a warlock (enemy or ally) or pillar the link attaches (`P_TransferLinkObj`). Link maintenance in `SystemIteration`: each tick the line is stretched `MoveLightningEx`; if the target is further than `60 + sizes` (player) / `30 + sizes` (self-link to non-player) the target is **pulled toward the caster with 1.4 u/tick acceleration** (and the caster toward it for pillars) and the target takes damage every `Period5` (0.18 s): `(5 - (LinkDur-2)/5)/5 + 0.1 + 0.1*LevelH` first 25 ticks, afterwards `0.1+0.1*LevelH` (tooltip damage per tick 0.2 ... 0.9) - plus extra initial damage; line damage also hurts enemies in between through `DamageLine` (width 75; applies Magma-type slow). The link ends when the target comes within range, on Shield/Dome/Rush/Time Shift/Mirror/Vector/Ethereal casts, on any pillar death, by pressing `S031` or on the missile range end (the hook returns at 20 u/tick; fast return 40 u/tick).
* **Induction (toggleY true)** `AdhesionCast`: `InductionOrb.mdx` scale 1.3 (same speed/range/size). If it hits an ally: both gain +100 move speed for `2.25*TI*TI2` s and the lightning stays 4.5*TI s; if it hits a pillar/wall: caster gets +100 speed for `4.5*TI` s and link lasts 4.5*TI s; `ExtendInduction` (second press) costs **5% of max HP** to add `2.75*TI` (pillar) or `1.75*TI*TI2` (ally; also heals ally `2*HP/(HP+HP2)`) seconds; second-press `InductionMakeLine` leaves a persistent lightning line between orb and the previous target that deals line damage `4.5 + 0.5*LevelH` initially then `0.25*(0.9+0.1*LevelH)` per Period5 tick to enemies on the line (width 75) for `4*TI` s.
* Cooldown mode A 17, 15, 13, 11, 9, 7, 6, 5 (w3a also 1 at L9); mode B 16, 15, 14, 13, 12, 11, 10, 9, 8, 7. Lightning damage (mode B) 1.0 ... 1.7.

### 7.2 Disable (Y) - `S017` - `EntangleCast` / `SilenceCast` (JASS 14436, 14225)
* **Entangle (toggleY false)**: `Entangle.mdl` scale 1.1, size 39, speed **1000 u/s**, lifetime `1.1*(1+0.12r)` s, collides with players and pillars. On a non-immortal warlock (`AoEBind`): duration `(3.6 + 0.25*abilityLevel)*TI*TD` (tooltip 3.6 ... 5.0; **uncertain +0.2 vs 0.25 per level**), stops its current order, **move speed set to 0**, `P_KnockbackMultiplier /= 3` (takes **3x knockback**), `P_Stunned` (ice friction, no walking), a `CastingDamage` curse of `5 - flatReduce` DP is added (it accrues to DP when the victim next casts), and **damage >= 20 total breaks the entangle** (`EntangleDmgCount`). Ends at duration: speed/KB multipliers restored. Cooldown 24, 20, 17, 15, 13, 12, 10.5, 9 (L1-8).
* **Silence (toggleY true)**: two `FaerieDragonMissile` (scale 1.1, size 23, speed 900 u/s) leave 210 units *behind* the caster, flying along the aim rotated by +/- 0.30 rad and curving to meet at the target point (lifetime `((min(dist,900*(1+0.12r))+210)/900)*1.0468`); on reaching their end they draw a lightning (`HWPB`, 0.4 s) and silence **every non-immortal warlock within 100 units of the segment between the two missiles** (allies hit are cleansed: `SearchExec(...)`). `Silence(victim, caster)`: for each spell slot (G, D, E, R, T, H(Y), Y(C); Scourge/F is *unaffected*) where `dX = LevelH(caster) - Level_X(victim) >= -2` and the victim owns that slot: that slot is disabled for `(4 + dX)*TI*TD` s (R slot only `0.5*(4+dR)`), minimum visual 0.5; the slot ability is replaced by a disabled `W00x` dummy. Text shows the total silence seconds; 3+ victims = "Silencer". Silence missiles collide only with other missiles. Cooldown 17, 16.5, 16, 15.5, 15, 14.5, 14, 13.5. Tooltip: duration 3.6 + difference.

### 7.3 Gravity (Y) - `S018` - `GravityCast`, `GravityIter`, `ForcefieldCast`, `QuakeIter` (JASS 14726, 14799)
* **Dark Matter (toggleY false)**: dark-summon missile, speed **400 u/s**, lifetime `900*(1+0.12r)/400` s (~900 u), bounce 1, collides with pillars only, `ID_Splitter` type (immune to deflection), each tick pulls: enemy warlocks within **450 units** (`TDst < 202500`) get velocity `(1.35 + 0.09*LevelH) * (1 - d²/450²) / (Mass+1)` toward it (mass 1 -> /2); non-meteor/gravity/repeater missiles within **600 units** are pulled with `1.5*(1 - d²/600²)`. Enemy warlocks within **250 units** take `0.1 + 0.2*LevelH` damage per Period5 (0.18 s) tick (tooltip 0.3 ... 1.7), the caster is healed the total damage. Cooldown 21, 20.5, ..., 17.5 (L1-8) (L9 17).
* **Force Field (toggleY true)**: `BansheeMissile` scale 2, speed **850 u/s**, lifetime `min(dist, 850*(1+0.12r))/850`, no collision, bounce 1; where it ends it drops a ground field (`GreenPentagramVampiric.mdl`) radius `250*0.8 = 200`, lifetime `3.6*TI` s. Each Period5 tick: allied warlocks inside heal `maxHP*(0.0012+0.0003*LevelH)`; enemy warlocks inside get a DoT (total `0.15+0.25*LevelH` over 0.2 s, i.e. DPS 2.0 ... 7.75 as tooltip) with a **slow** `0.45*KB²*MS` (field slow). Fades alpha at the end. Cooldown 26.

### 7.4 Control (Y) - `S050` (+ action `S052`) - `ControlCast`, `ControlColl`, `ControlToggleCast`, `ControlEnd` (JASS 14322, 14355)
* **Enemy cast (toggleY false)**: `ControlProjectile.mdx` scale 1.6, size 60, speed **850 u/s**, lifetime `3.5*(1+0.12r)` s (~2975 u), collides with players only. On the first free enemy (`P_Control==0`, remaining lifetime > 0.15 s): the missile slows to **400 u/s** and keeps flying; while it lives (remaining `Obj_Range - 0.15` s) the victim is *controlled*: every spell they cast is aimed at the **control missile's position** (`P_Control>0` target substitution in `AbilityActions`), and the victim gets `P_LifeStealMultiplier += 0.40` and `P_TimeInc -= 0.40` ("40% life steal weakness, 40% decreased buff duration"). Cooldown 14.5, 13.25, 12, 10.75, 9.5, 8.25, 7, 5.75.
* **Self cast (toggleY true)**: first press marks the current position as a remote casting point for up to **120 s** (action `S052`/"Cancel Control" allows cancelling). While marked, the **next spell cast originates from the marked point** (`P_Control<0`; `CX,CY = Obj_Pos[4*id+514]`, gate list of allowed spells such as teleports/shield/etc. is cast from the real position instead) and then control ends, granting a 3 s bonus (`MasteryBonus`): `+5%*LevelH` damage inflicted, `-10%*LevelH` life steal weakness (via `P_LifeStealMultiplier`/`TimeDec`). Cooldown 22, 20, 18, 16, 14, 12, 10, 8 (L11-18), 6 at L20.

### 7.5 Lock On (Y) - `S064` - `LockOnCast`, `LockOnColl`, `LockOnEnd` (JASS 16213)
* Missile `Daemonic Arrow` scale 1.1, size 39, speed **1000 u/s**, lifetime `1.1*(1+0.12r)` s, hits a warlock or pillar. On hit the target is **marked for 2.6 s** (`lkon`): the caster is pulled toward it by the lock-on velocity (`LockOnVel = 900 u/s`, velocity forced to `0.77*LockOnVel` toward the target, acceleration `LockOnAcc`), size +30, collides as Player_Coll, ends any thrust. On the dash collision: damage `2 + 0.5*LevelH` (+ `4.8 + 0.6*LevelE` if Wind Walk charge is also active, with an additional thrust-scaled part) kb 0.7; ends. While marked, channeling the F spell (Scourge) with `LockScourge` true (`0.75 s` after the lock) makes it an `EnchantCast`: targets take 20% extra damage and deal 50% for 2.8*TI s. Cooldown 24, 20, 17, 15, 13, 12, 10.5, 9 (L1-8) / 17 ... 13.5 (11-18). **uncertain**: tooltip text says "Damage: 2, Mark duration 2.6, Scourge buff duration 2.6 (3, 3.4 ...)" scaling by level.

---------------------------------------------------------------------------------------------------

## 8. Character (F-hotkey) abilities

Characters (`CharacterSelectId`, JASS 7662): index -> model/unit/F spell: 0 **Warlock** `h000` Scourge `S001`; 1 **Heretic** `h003` Bane `S022`; 2 **Inquisitor** `h00J` Redemption `S023`; 3 **Occultist** `h009` Ritual `S051`; 4 **Enchanter** `h010` Enchant `S100`. 5 = random. Stats are identical across characters (the lich model note in the readme); only the F spell and model differ. All F spells: cooldown **3.0 s**, cast time **0.7 s** channel (`acas 0.7`, 'cripple'-based `ANcl`), no mana, hotkey F, cancel casting with hotkey S. Cast effect: `DarkPortalTarget` + `Nova_yellow` at the caster, radius **250 units** (`dst² <= 62500`), self damage, and `LevelF` is the F level (0 at start; +1 per item purchase: Mask of Death (`h001`)/Sword (`h007`)/Aegis `h008` chain and the Blood Sword; each also raises the ability level `S001` etc.). Mask of Death grants `P_ScourgeLSBonus` (+3 life-steal levels).

Common behaviour in every F spell (`ScourgeCast` etc.):
* Self damage via `SelfDamage` (never kills: HP floor 0.5) unless standing on lava with Lava Treads (activates the boots instead); Aegis (`Allitems[17]` + `AegisReady`) activates the defensive stance (`ActivateAegis`: damage taken x3/4, knockback x0.5, 5*TI s, mass up).
* For each enemy warlock in radius: `DamageTargetPoint(cx, cy, ..., damage, kb = factor - dist/1000)`, remembers `P_ScourgeOwner` for 0.5 s (`ScourgedRecently` re-scourge window) - **the recently scourged victim can be scourged again at the stronger rate**, `n` counts victims. Caster heals `H` = lifesteal of damage dealt if `LevelMelee != 0`. 3+ victims: "Hattrick" (or "Vampire" with Mask of Death) text and `ArcaneExplosion`.
* "Denied": any enemy currently *linked to the caster* standing on lava with link not in Y-toggle mode gets the link cut and takes `5 + LevelF` damage.

| F spell | id | Self dmg | Damage to enemies | kb factor | Extra |
|---|---|---|---|---|---|
| Scourge | `S001` | 10 | `10 + LevelF` (10 / 11 / 12) | `1.0 - d/1000` | if `LevelF > 0`: DP recovery `(LevelF+1)*n` (3 per hit at lvl 3) |
| Enchant | `S100` | 10 | `10 + LevelF` | `1.0 - d/1000` for the previous scourger, else `0.6 - d/1000` | victims (not the previous scourge owner) take **x1.2 damage and deal x0.5 damage for `2.8*TI` s** (`LockOnShockEnd`) |
| Bane | `S022` | 8 | 8 | `1.1*(1 - d/1000)` | after 0.8 s spawns a **bane orb** at each victim (radius 250 zone, 3.3 s, ally collision): when the heretic or an ally reaches it, a "FORK" lightning (range 1200) hits that victim for `3.0 + 1.5*LevelF` (+flat reductions) and heals the caster; orb tooltip "3 -> 6 damage" |
| Redemption | `S023` | 10 | `10 + LevelF` | `0.9*(1 - d/1000)` | unless the victim is the scourge owner or linked: victim is **teleported to the mirrored position across the caster** (`ReplaceWarlocks`: `pos = 2*caster - pos`) and knocked back so as to arrive with the computed speed; DP recovery `(LevelF+1)*n` |
| Ritual | `S051` | 8 | `8 + 0.7*LevelF` | `1.3*(1 - d/1000)` | spawns a **mist** (4 s, radius 300) if `n>0` that slows enemy missiles x0.90 per tick inside it (except Splitter/Meteor/Gravity) and heals the occultist `2 + 1*LevelF` when it expires if within 300 |
| Cataclysm | `S020` | 15 | stack 0: `11`, radius 300 (`damage - d/60`); stack 1: `12`, radius 300; stack >= 2: `12 + sqrt(stack+2)`, radius 400 (`damage - d/40`); `+ LevelF` if LevelF > 0 | 1 | Ghost-mode hidden player's F spell. Each cast re-arms a 6.5 s window (key 5812) whose `level` (0,1,2,...) is the stack; kb is 1 (not distance-scaled); an enemy still linked to the caster takes `10 + 2*LevelF` and the link is cut; "Hattrick" needs 5 victims |
| Pious | `S021` | 10 | 10 | `1.0 - d/1000` | King-mode F spell. Allies within 500 units get **+60 move speed for `4*TI*TI_ally` s**, heal `5 + 1.5*(LevelF-1)` HP and lose the same amount of DP (tooltip 5 -> 8); the spell cancels with S |

Tooltips (w3a): Scourge "Damage nearby warlocks by 10 including yourself; level 3: 12, DP recovery 3 per hit"; Pious "Heals by 5 / Reduces damage points by 5 / +60 ms for 4 s" (level 3: 8); Bane "Damage nearby warlocks by 8 and yourself by 8; bane orb 3 -> 6 damage"; Redemption "teleported and knocked back in the opposite of their current position"; Ritual "8 and yourself 8 / 9.4; mist heals 2 -> 4"; Enchant "gain 20% damage bonus and apply 50% damage reduction"; Cataclysm text as above.

---------------------------------------------------------------------------------------------------

## 9. Masteries and item interactions that change spell behaviour

* **Range Mastery** `R00Y` (+1 `LevelRanged`, max 6, helmet items subtract 1/2/3 from the cap): all missile ranges `x (1 + 0.12*LevelRanged)`; `P_RegenMultiplier += 0.05`.
* **Life steal mastery** `R00D` (+1 `LevelMelee`, max 6): `LifestealPerLvl = 0.10` of damage healed on every hit (`DamageTargetNoKnockback` lifesteal) scaled by victim weakness; Mask of Death +3 levels for the F spells; Wind Walk invisibility strikes count +5.
* **Duration mastery** `R00I` (+1 `LevelDuration`, max 6): `P_TimeInc += 0.08` (buff/curse durations and cooldown-like `AddAction` times that use `TI`).
* **Jack of all trades** `R01E`: +1 to all three at once.
* Each mastery point (and each helmet level) multiplies the player's knockback taken by `(1 - 0.025 * totalMasteryPoints)`; `R017` = total points (gate for upgrades).
* **Items touching spells**: Staff of Fireball (burn, 4.1), Scroll of Fireball (bounce, 4.1), Aegis (F spells), Lava Treads (F spells / teleports on lava), Blood Sword & Mask (F level, life steal), Stone of Jordan Ring (+2 upgrades), Helmet (range -12%/level, knockback -8%/level, mass +10), Warden (flat damage reduction 0.4/level, applies before multipliers), Pocket Watch (`P_TimeDec` buffs -5%, curses -35%), Cursed Pendant (+25% curse duration on self, knockback -15%).

---------------------------------------------------------------------------------------------------

## 10. Complete checklist of spells documented (39 spell abilities + helper actions)

Attack: Fireball (incl. Ice variant), Lightning, Homing, Boomerang, Arc, Poison Bolt, Shock, Meteor, Magma, Splitter (2 modes), Wind Walk (2 modes), Repeater, Burst, Drain (2 modes), Fire Spray (2 modes), Bouncer (2 modes: bounce / recharge), Dagger.
Movement: Teleport, Multiport, Thrust, Ghost, Swap, Relocate, Return.
Defence: Rush, Shield, Time Shift, Mirror, Vector, Dome, Ethereal Form.
Curse/control: Link (Hook/Induction), Disable (Entangle/Silence), Gravity (Dark Matter/Force Field), Control (Enemy/Self cast), Lock On.
Character: Scourge, Bane, Redemption, Ritual, Enchant, Cataclysm, Pious.
Helper abilities: `S029` secondary teleport, `S031` link action, `S047` time-shift action, `S052` control-cancel, `A02F` burst action, `A02G` shock action, `A02H` ethereal action, `A02J` vector action, `S128` inventory, `A02I/A02K` lava treads dummies.

## 11. Known gaps / uncertain items (summary)

1. `S053` (tech `R01F`/`R01G`, "Unused") and the Phoenix functions are dead code.
2. Arc direction toggle (`toggleD`, M042) is never read by code; Repeater toggle labels exist but `AbilityActions` has no branch -> both single-mode.
3. Several tooltips (Bolt DPS, Bouncer 20% reduction, Time Shift 3.5 s, Disable +0.2/level, Shock cooldown 7 s and "shocked targets") disagree with the code; the code values are used above.
4. Exact Fireball level cap (10 vs 11 upgrades), meaning of Link damage tooltip units, Bane orb homing behaviour (`HereticColl/Iter`), Bouncer-Recharge target slot (`SpellsAcquired[4]`) and Control remote-cast whitelist need an in-game check.
5. Level 11+ (mode B) rows exist in w3a for spells that have no toggle (Dagger, Repeater); treat as unused.
6. Terrain/pillar/shard behaviour, the game modes, items, shop and multiboard are outside this file (see the other notes files).

# 04 - Physics and knockback (the core feel)

Everything here is from the custom physics engine in the JASS ("ObjectStruct"/"PhysicsSystem" libraries): `SystemIteration` (JASS 5845-6305), `Object_*` (2815-3125), `DamageTarget*`/`Deal*AoE` (4779-5216), `PlayerColl` (5591), `PillarBounce` (6309), constants (lines 498-534). All values are per **tick** unless stated; a tick is `Sys_IterRate = 0.03 s` (33.3 Hz). Per-second = per-tick / 0.03.

## 1. Constants (JASS lines 498-512, 722-724)

| name | value | meaning |
|---|---|---|
| Sys_IterRate | 0.03 s | physics tick, driven by one repeating timer (`StartSystem`) |
| Stats_Friction | 0.96 | per-tick velocity multiplier for players on normal ground |
| Stats_IceFriction | 0.93 | per-tick multiplier on ice (and when stunned/frozen: also used) |
| Stats_MS | 220 * 0.03 = 6.6 | base walking velocity (units/tick) = 220 u/s (matches unit move speed 220, w3u h000 umvs) |
| Stats_IceAcc | Stats_MS/10 = 0.66 | ice steering acceleration |
| Stats_Bounce | 0.5 | player-vs-player bounce factor (`Obj_BounceColl_Factor`) |
| Stats_PillarBounce | 0.5 | player bounce off pillars (`Obj_Bounce`) |
| Stats_Size | 30 | player collision radius |
| Stats_PillarSize | 35 | pillar radius |
| Z_Ground | -128 | ground height (only for shards, flying objects and `Obj_EnableZ`) |
| player max HP | 100 | w3u h000 `uhpm`; DP is shown as "mana" (max 200 bar, value can exceed) |
| LifestealPerLvl 0.10, RangePerLvl 0.12, DurationPerLvl 0.08 | | mastery steps |

The world unit is the WC3 unit (1 cell = 128). Treat "units" as game metres in the remake; 128 u = 1 tile.

## 2. State per object

Each simulated thing is an **object** with type 1 = player, 2 = missile/projectile/shard, 3 = pillar/obstacle. Fields: position X/Y/Z, velocity X/Y/Z, acceleration X/Y/Z, `Walk` X/Y (players), `Size` (radius), `Friction` (missile default 1.0 = none), `Bounce` (-1 = no bounce, destroyed on solid contact), `Range` (remaining lifetime in *seconds*, decremented by 0.03/tick), `Real/Real2/Real3/Int` scratch, `Collision[type]` flags (which of types 1/2/3 it collides with), `IgnPriority`, `OwnerColl/AllyColl/BounceColl` flags, callbacks `OnColl/OnEndRng/OnGround/OnIteration/OnDestroy`, and for players per-player stats in arrays `P_*`.

Note `Range` is a time: e.g. fireball `Obj_Range = 1 + 0.12*LevelRanged` seconds at speed `1000*0.03 = 30` u/tick -> ~1000 units reach at base (JASS `FireballCast`).

## 3. Per-tick order (`SystemIteration`)

1. Update the 3/5 sub-period counters (`Period3` every 3rd tick, `Period5` alternates on every 3rd tick) used for periodic effects (links, thrust trail).
2. **For every object in list order** (players are kept first):
   - **Player (type 1)**:
     - *Thrusting* (`P_Thrust`, i.e. R-spell Thrust/Phoenix state): `v += Acc; v *= Stats_Friction`; range decrements; ends when range <= 0.
     - *Stunned* (frozen/entangled): `v *= Stats_IceFriction`; walk zeroed; cannot steer.
     - *Normal ground*: `v = (v - Walk_prev) * 0.96`, then derive the new walk direction from where WC3 moved the unit (`GetUnitX/Y - Obj_Pos`, normalised) and set `Walk = dir * MS_tick` (`Obj_Real`); `v += Walk`. **So walking is a separate velocity that is subtracted and re-added each tick, and knockback velocity (the remainder) decays with 0.96 per tick** -> half-life ln(0.5)/ln(0.96) = 17 ticks = 0.51 s; total displacement from an impulse `v0` is `v0/(1-0.96) = 25*v0` (<= ~24*v0 because friction is applied before the move).
     - *Ice* (`PObj_Terrain==3`): let `dir` = steering unit vector; if the current velocity along `dir` is below the walk speed: `v = (v + IceAcc*dir*(walk/MS)) * 0.93` else just `v *= 0.93`. No instant stopping; half-life 9.5 ticks; displacement factor 14.3.
     - Mirror/clone illusions (`Obj_Mirror`) run `MirrorIter`; "lock-on" (Enchanter's spell) overrides velocity to `0.77*LockOnVel` towards its target and adds `LockOnAcc`.
     - Link/hook systems (`Link_Dur`, `P_TransferLinkObj`): a tether pulls a target by adding `1.4*unit_vector` per tick to velocity and deals damage on `Period5` ticks (see 01-spells.md Link).
   - **Missile (type 2)**: `v = (v + Acc) * Friction` (missile Friction default 1); `Z` handled if `EnableZ`; runs `OnIteration`; if `Range>0` decrement, at <=0 call `OnEndRng` (or remove).
   - If `EnableZ`: apply `OnGround` when it would go below ground (shards bounce/stop).
3. **Collision pass**: O(n^2) pair loop over the list. Pair is tested if both flag-matrices allow (`Collision[3*a+type(b)] and Collision[3*b+type(a)]`), different owners (or both `OwnerColl`), and either different teams, or both `AllyColl`; shield objects only collide with matching shield state. The test is a **continuous sphere sweep**: relative position `p`, relative velocity `v`, `r2 = size_a + size_b` (+30 if either has the player-collision handler, so player-vs-player "touch" for thrust/whirlwind uses radius 60). Uses quick AABB reject then solves `|p + v t| = r2` for t in (0,1] and moves both objects to the contact point. If already overlapping (`dist² < r2²`) callbacks fire immediately. `OnColl` of A is called if `IgnPriority(A) >= IgnPriority(B)`, and vice-versa (higher number "wins" attention). Callback receives `TObj`, `TCollObj`.
   - **Player vs player bounce** (only when both have `BounceColl`): impulse `s = (p·v)/(dist² + 0.01) * (bf_a + bf_b + 2)/4` (= 0.75 for two players), with `p = pos_a - pos_b`: `r = 2*m_a/(m_a + m_b)`; `v_b += s*p*r`; then `r = r*m_b/m_a`; `v_a -= s*p*r` (momentum-conserving, mass-weighted; JASS `SystemIteration`, bl1 branch). Overlap resolution pushes apart by `p/s` with `s` min 30. If the impact `|s*p| > 30` and neither is thrusting: **collision damage** `WarlockVelocityDamage(dmg = |s p|)`: `dmg' = 0.7*sqrt(dmg - 20)`, credited to the player who last damaged the faster-moving victim; both enemies of the credited player take it; a splat effect plays. That is the "colliding at high velocity with an enemy or pillar causes both damage" rule.
4. **Deferred destroy stack**: objects marked via `Sys_AddToDestroy` get their `OnDestroy` run after the pass (so a missile removed in a collision still finishes).
5. **Integrate**: for each non-pillar: `pos += vel` after checking the *special wall* tile (`TrSpecial`): per-axis, players reflect velocity with their bounce factor (`-v*Obj_Bounce`), missiles with `Bounce>0` reflect, others stop and are destroyed. Then clamp to map rect with velocity reflected `*0.5`. Then `SetUnitX/Y`, `SetUnitFlyHeight`.

## 4. Damage and knockback formulas

Entry points (all in `DamageTarget*`):

```
DamageTargetNormal(damager_obj, target_obj, value, KnockbackA)      // direction = target - damager (radial)
DamageTargetPoint(x, y, ...)                                        // direction = target - explicit point
DamageTargetLine(...)                                               // direction = damager's velocity direction
KnockbackOnly(...)                                                  // same as Normal but no damage
DamageTargetNoKnockback(damager_obj, target_obj, value, ...)        // HP/DP/lifesteal/assist bookkeeping only
```

For a player target (type 1), after the flat reduction `value = value - P_DamageFlatReduce[target] - P_DamageFlatReduce[damager]` (return if <= 0):

```
speed = (100 + DP_target) * value
        * DamageMultiplierInflict[damager] * DamageMultiplierTake[target]
        * 0.03                      // Sys_IterRate
        * KnockbackMultiplier[target] * KnockbackA
v_target += speed * unit_direction         // instantaneous velocity impulse, units/tick
```

then `DamageTargetNoKnockback`:

```
value *= Inflict[damager] * Take[target]            // multipliers are applied once to damage here and once (inside 'speed') to knockback
if shop time or target immortal: show cyan number, nothing else
else:
  P_Damage[damager]      += value                   // round damage board
  DmgToPlayer[target][damager] += value             // assist/kill credit
  P_DamagingPlayer[target] = damager
  if HP - value < 0.5 -> Death(target)  (or the team-specific TS action)
  else HP -= value; DP += value
  if damager has melee mastery: Heal(damager, value * 0.10 * LevelMelee * LifeStealMultiplier[target])
```

- **HP** `P_HP` (max 100, items raise it), **DP** `P_DP` (damage points = how much knockback you take). `DP += damage` on every hit and `DP += lava/2` on lava; `Heal()` gives HP and **reduces DP by heal/2**. DP resets to 0 each round (`StatsRenew`).
- Knockback grows linearly with `100 + DP`: DP 0 -> x1, DP 100 -> x2, DP 200 -> x3.
- **Worked example** (Fireball base damage 6.5, KnockbackA 1, all multipliers 1): impulse = `(100+DP) * 6.5 * 0.03` = 19.5 u/tick at DP 0 (585 u/s), 29.3 at DP 50, 39.0 at DP 100, 58.5 at DP 200; travel ~25x that: 490 / 730 / 975 / 1460 units (about 3.8 / 5.7 / 7.6 / 11.4 tiles). Compare safe-area radius 9-12 tiles.
- Pillars/shards: pillar target takes damage only (`value - flatReduce`); shard target: `speed = 100*value*KnockbackA*0.03*(0.8 offensive | 0.4 defensive)` (no DP term).
- AoE helpers: `DealAOEDamage(damager, value, knockback, x, y, radius, bonus)` -> per enemy within radius: `value * (bonus + (1-bonus)*(1 - d/radius))`; `DealCustomAoE` uses bonus 0.5 (centre 100 %, edge 50 %). `Obj_CollData` bitmask (`Power2[id]`) excludes players already hit by the projectile ("pierce once").
- `DealKnockbackAoE` (pillar/shard explosion): impulse `dx*dst'*1.6` with `dst' = (1 - d/radius/2)*damage` (not DP-scaled, and teammates are pushed too).
- Self damage: `SelfDamage` reduces own HP but never below 0.5 (can't suicide by it), blue floating number.
- Floating text: damage numbers (`0.007*sqrt(value+3.5)` size, 1.2 s, gold; cyan in shop/immortal), heals green, self-damage blue; knockback sets the text velocity from the hit direction (`0.05*dir`, line hits use `0.002*vel`).

## 5. Collision/contact rules summary

| pair | what happens |
|---|---|
| missile vs enemy player | missile `OnColl` (spell-specific): damage + knockback, usually then destroyed |
| missile vs pillar | `PillarBounce`: reflect if `Bounce>0` else destroy; damages pillar HP |
| player vs pillar | slide at walk speed; if normal speed > 30: pillar damage `0.7*sqrt(d-20)` and bounce `(1+0.5)*d`; a hard hit can destroy the pillar and the player pays `rdmg+5` |
| player vs player | mass-weighted bounce, collision damage above impulse 30 (`0.7*sqrt(x-20)` to enemies of the credited player) |
| player vs player (Whirlwind/Thrust) | extra effects at radius 60 (`PlayerColl`): both winding = "Whirlwind" cancel (`ReduceRangeInitiate 7.5`), Lock On, Thrust damage `5.0+0.4*LevelR` scaled by speed/(39*39), etc. |
| anything vs map edge | clamp, reflect `*0.5` |
| anything vs `TrSpecial` tile | wall |

## 6. Speed/mass/other stat hooks that change feel

- Move speed `GetPlayerMS = Obj_Real/0.03`, changed by `SetPlayerMS`: Boots +10/+20/+40 (items), Lava Treads +35 etc., Rush spell +35 and deflect pool, Drain curse reductions.
- Mass `P_Mass` default 1.0; helmet/cloak items change in kg (`-stats` shows `100*mass` "kg"); Avatar (Ghost mode) gets +7 mass and x1.5 damage. Mass only matters in player-player collisions.
- `P_KnockbackMultiplier` (items "Decreases knockback taken by X %", mastery 2.5 %/level, Immortalize x0.1), `P_DamageMultiplierTake/Inflict`, `P_DamageFlatReduce` (Warden: -0.4..-1.6 per hit before multipliers), `P_LifeStealMultiplier`, `P_TimeInc` (buff duration) and `P_TimeDec` (curse duration), `P_RegenMultiplier` - all exposed by `-stats` (06).
- Immortalize (round start 4 s, shop time): damage dealt x0.1, knockback taken x0.1.
- Spell cast dummy "missiles" are created at the caster; speeds in spells are written as `N * Sys_IterRate` where N is units/second (fireball 1000, etc.).

## 7. Remake notes (determinism and tick rate)

- Reproduce the simulation at a fixed 33.33 Hz (0.03 s) tick with the above formulas on 2D floats. Do not tie it to render frame rate. Positions are floats in the JASS (WC3 `real`, 32-bit). For online play use a deterministic fixed-step sim (see 07).
- Walking: replace the WC3 "move unit then read delta" hack by directly setting the walk direction from input (click-to-move target or WASD), `Walk = dir * MS_tick`. WC3 stops moving a unit when it reaches its click target; reproduce: walk velocity is zero when within about one tick of the target.
- The pair loop is O(n^2) but n is small (<= 12 players + ~100 missiles + ~16 pillars).

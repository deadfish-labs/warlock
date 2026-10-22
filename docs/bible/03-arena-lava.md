# 03 - Arena, lava, shrinking, obstacles

Sources: JASS `ActivateLava`, `SetSafeTerrainSize`, `ShrinkAction`, `LavaPeriodic`, `TerrainChange`, `InitTrig_Terrains`, `Obstacles___InitObstacles`, `ObstacleDefine`, `CreateObstacle`, `PillarShard_*`, `PillarBounce`, `PillarVelocityDamage`, `Func_PresentShard_Destroy`, `SystemIteration` (end part), `GameStarting`; `war3map.w3e`; JASS globals 1-1163. Line refs are in `notes/data/war3map.j.txt`.

## 1. Geometry

- Terrain file: `war3map.w3e`, tileset `O` (Outland), 65 x 65 vertices = 64 x 64 cells of 128 world units; origin corner (-4096, -4096) -> the world is 8192 x 8192 units, centre (0,0). JASS: `ArenaSize=65`, `LeftBorder=DownBorder=-4096`.
- Camera bounds set in `GameStarting`: `SetCameraBounds(3584, 3840, 3584, -3840, -3584, -3840, -3584, 3840)` (approx +-3584 x +-3840). Objects are clamped to the playable rect (`minX..maxX`, `minY..maxY`, from `bj_mapInitialPlayableArea`, `SystemIteration`); hitting the rect edge reflects velocity with factor 0.5.
- Camera (`Tick`, every 1.0 s): target distance 1650, angle of attack 304 (deg, i.e. looking down about 56 deg), Z offset = per-player `-cam` value. Fog and fog-mask disabled. Main time-of-day is used as a visible "arena shrinks soon" clock (see below).
- The map's ground is painted in two regions: **safe tile** (platform) and **lava tile**. Gameplay checks the tile type under a player each 0.1 s.

### Terrain tilesets (JASS `InitTrig_Terrains`; chosen at random per round in `TerrainChange` unless the host fixed it)

| option | safe tile | lava tile | lava fire effect | pick weight |
|---|---|---|---|---|
| 1 | Jwmb | Dlav (Outland lava) | ElfSmallBuildingFire2 | 30 % (roll 0-1000: <=300) |
| 2 | Odtr | Dlvc | ElfSmallBuildingFire2 | 20 % (<=500) |
| 3 | Ngrs | Glvc (blue "lava") | LavaFire_Blue.mdl (custom) | 20 % (<=700) |
| 4 | Zgrs | Cpos (green poison) | LavaFire_Green.mdl (custom) | 20 % (<=900) |
| 5 | Zsan | Oaby (abyss/white) | LavaFire_White.mdl (custom) | 10 % |

Host can fix with `-c7 <n>` (`C_Map`, 0 = random, 3 = Christmas theme used by `-ymas`). Tile types only differ visually; damage is identical. Remake: 5 cosmetic themes are enough, any art.

### Shape

`CurrentShape` is 0 (circle) or 1 (square) - JASS line 9919 sets it randomly per game; the host can force with `-circle` / `-square` (and `-circle donut`, `-square donut`). It is the 6th argument of WC3 `SetTerrainType(x, y, type, variation, area, shape)`, i.e. the safe area is a **circle or a square** of "radius" N cells (WC3 area semantic: N cells around the centre cell, 128 units per cell).
- `Donut=true` adds a lava hole in the centre: `SetTerrainType(0,0,lava,0,4-CurrentShape,CurrentShape)` (radius ~4 cells) after each resize (`SetSafeTerrainSize`).

## 2. Safe-area size and shrinking (Round / Avatar / King modes)

- Initial radius when the round begins (`ActivateLava(true)`): `CurrentTerrainSize = 9 + R2I(SquareRoot(PlayerCount))` cells. (12 players -> 12 cells = ~1536 units; 2 players -> 10.) During the shop phase the radius is `8 + AliveCount/2` (`InitMode_Round`) / `9 + PlayerCount/2` (`ActivateLava(false)`).
- Spawn ring radius: `768 + 64*sqrt(2*PlayerCount)` units (`RoundStart_2`) - inside the safe area.
- The whole map is repainted with lava first (`SetTerrainType(0,0,lava,-1,42,1)` = 42-cell square, i.e. everything), then the safe disc/square is painted (`SetSafeTerrainSize`).
- **Shrink**: `ShrinkTimer` is started at round start with period `C_ShrinkTimePerPlayer * sqrt(AliveCount)` seconds (default `C_ShrinkTimePerPlayer=15`; `-c6 <sec>` changes). Each time it fires (`ShrinkAction`): `CurrentTerrainSize -= 1`, repaint, and re-arm with `15*sqrt(AliveCount)` using the *current* number of living players, until the radius reaches 0 (then the timer is paused; everyone eventually dies on lava).
  - Example: 12 alive -> 52 s per cell; 4 alive -> 30 s; 1 alive -> 15 s. As players die the interval shortens.
- Visible clock: `SetTimeOfDayScale(240 / (sqrt(AliveCount)*C_ShrinkTimePerPlayer))` with time-of-day starting at 6 - the WC3 day/night clock at the top of the HUD advances so that it reaches the next shrink moment on the hour. (Tips hint: "The arena gets smaller over time in round mode. Look at the game clock (top middle of the HUD)".) In the remake: show a "next shrink in" timer.
- **No shrink** in Deathmatch and King of the Hill modes: radius is fixed at `9 + floor(sqrt(PlayerCount))` for the whole match (`ActivateLava`, branch `GAME_MODE_DEATHMATCH or KOTH`).
- Custom terrain (`CustomTerrain=true`, a pasted code from a terrain editor, see `CTConversion_*`, `-t...` command): no automatic shrink, ready-made revive positions (`RevivePos`, `RevivePosCount`), and obstacles included. The special tile type `TrSpecial` (6) acts as a **solid wall** (`SystemIteration`: velocity along that axis is reflected with the object's bounce factor, projectiles are destroyed). Tile classes used by custom terrain: `TrLava 0, TrLavaPillar 1, TrPlatform 2, TrPlatformPillar 3, TrRevive 4, TrIce 5, TrSpecial 6`. Remake: optional (could), but design the terrain as a data grid so it could be added later.

## 3. Lava damage

`LavaPeriodic` runs every **0.1 s** (`LavaTimer`, started in `ActivateLava(true)`), for each living player object:

- Tile under the player is read from `GetTerrainType(x,y)`:
  - `TerrainTypeLava[option]` (and player not `P_Immortal`) -> `PObj_Terrain=2`; `P_HP -= C_LavaDmgPlayer[id]`; `DP += C_LavaDmgPlayer/2`; a lava-fire effect plays. Default `C_LavaDmgPlayer = 0.900` per tick -> **9 HP/s**, and +4.5 DP/s. With 100 max HP a player dies after ~11 s on lava (less with already-reduced HP).
  - `TERRAIN_ICE` -> `PObj_Terrain=3` (ice physics; regen applies).
  - otherwise (safe) -> `PObj_Terrain=1`; `HP += P_Regeneration * P_RegenMultiplier` per tick (default 0.05 -> 0.5 HP/s; `-stats` displays "10 * regen /s").
- If HP < 0.5 the player dies (`Death`). If several die the same tick they are killed lowest-HP first (the loop at the end of the function).
- The player who last damaged you gets the kill (`P_DamagingPlayer[id]`; 12 = nobody -> counted as a suicide). Lava ticks also add half the damage to `DmgToPlayer[victim*12 + lastDamager]` so assists count lava damage.
- Modifiers on `C_LavaDmgPlayer` (per player, all multiplicative and reverted when the effect ends): **Lava Treads** item (`ActivateLavaTreads`, JASS ~10260): lava damage /8 (87.5 % resist) and +35 move speed for a few seconds (the code's base duration is 3 s, scaled by buff-duration `P_TimeInc`; item tooltips say 4-5 s by tier, so the tier extends it elsewhere - uncertain), instant heal up to 10.5 HP if HP is below that, cooldown 28 s (`LavaTreadingReload`); **frozen by Iceball** (`IceCast`/`IceEnd`): target is paused ("stop" order + `PauseUnit`), takes x0.5 knockback and x0.5 lava damage for `3.0 + 0.5*consecutive` seconds (x curse/buff duration multipliers); **Present shard** collected while standing on lava: x1/8; host constant `-c1 <n>` sets `C_LavaDmgPlayer = n/10`.
- Immortal state (first 4 s of each round, or shop time) ignores lava.

## 4. Obstacles (pillars)

- Obstacles are WC3 doodad-like units (`e000`) with no collision of their own; the JASS physics registers each as an object of **type 3** (`Obj_Allocate(3)`), `Obj_Size = Stats_PillarSize = 35`, not moving (they can be pushed only when they become shards - see below). 14 visual styles `utype` 0-13 (ruins columns, rocks, trees, ruined pillars, icecrown pillars, obelisks, snowman for `-ymas`). Style is picked per layout: `i = GetRandomInt(0,4)`, `i = 5` with 0.3 % chance (rare style), then each obstacle uses `i*2` or `i*2+1` randomly (JASS `Obstacles___GetRandomPillar`).
- **Layout lists**: `Obstacles_SelectList(n)` hides the old list and activates list n. `TerrainChange` calls it with `GetRandomInt(2,11)` unless `C_Obstacleson` (`-c8 <0..11>`) fixes a list; list 0 = none (used in the shop phase), list 1 = default set (initial). Lists 2-11 are hard-coded coordinate sets (`Obstacles___InitObstacles`, JASS 6688-6823); the first generated set (list 2) is up to six pillars on a hexagon of radius 700 (1-2 random ones omitted), the next set is 4 random points inside radius `768 + 64*alive`, the remaining sets are further hard-coded patterns. The exact coordinate tables are in JASS lines 6700-6823 and were not transcribed (uncertain: which pattern is which list index). `PlayerCount`-dependent: pillar HP = `PlayerCount*3 + 17` (`PillarShard_Init`; 100 in test mode).
- **Collision with pillars** (`PillarBounce`, called from `PillarColl`):
  - pillar pushes the object out so they don't overlap (`Obj_Size sum + 0.02`).
  - For a **player**: let `d` = velocity component into the pillar. If `d > 30` and not thrusting -> `PillarVelocityDamage`: damage to pillar `0.7*sqrt(d-20)`; if pillar HP is not exhausted the player bounces: `v -= d*(1+bounce)*normal`, where `Obj_Bounce` for players = `Stats_PillarBounce = 0.5` (so reflects with 1.5x of the normal component removal); if the hit would destroy the pillar ("GoThrough") the player keeps `0.2 - 0.3*...` of his velocity and takes `rdmg+5` self damage "dust". Walking speed contact (d <= 1.1*Stats_MS = 7.26) just removes the normal velocity (slide). Thrusting players ignore pillars for the bounce.
  - For a **missile** (type 2): normal component reflected `v -= d*(1+Obj_Bounce)*n`; most missiles have `Obj_Bounce = -1` -> destroyed instead (the "pillar bounce" only applies to missiles with `Bounce > 0`, e.g. fireball with Warlock's pendant: bounce 1.2).
- **Pillar HP and destruction**: `Pillars_TakeDamage` (JASS 3665) lowers `Obj_Real`; at <=0 the pillar `Func_PillarShard_Destroy`s: `DealKnockbackAoE(radius 320, damage 14)` explosion, then `PillarShard_Explode` creates `Shard_Number = 3` shards (config `Shard_Number`, `ShardLife=10`, height step 50, launched upward with `VelZ 750-900 * 0.03` and random horizontal `+-150*0.03`), pillar disappears and may **respawn later** (`PillarShard_Respawn` re-creates the pillar object from its stored position; `A_RespawnPillar=752` is the timed-action id; the delay was not captured, uncertain).
- **Shards**: size 48, bounce 0.75, friction 0.97 once on the ground, live `Obj_Range = 15` (offence) time; types (`PillarShard_SpawnMode` default MIX: each shard 1/3 offensive, 2/3 defensive):
  - *Offensive* shard: damage to anyone it hits when destroyed: `DealKnockbackAoE(radius 275, damage 9)`; knockback factor 0.8 (`Shard_KB_Factor_Offensive`); `Obj_Real=1` HP.
  - *Defensive* shard: HP `ShardLife=10`, knockback factor 0.4; used as a movable shield.
  - Shards can be hit by spells (`DamageTargetNormal ... Obj_bShard` branch: knockback `100*value*KnockbackA*0.03*factor`, no DP term), pushing them around.
  - *Present* shards (`-ymas`, Christmas mode; `SHARD_MODE_PRESENTS`, 1 shard, life 5): collect = power-up chosen by state: if HP<=50 -> regen +0.35 HP/tick-unit, else x1.15 damage; if DP>100 -> damage taken /(4/3) and knockback /2, else +40 move speed; if standing on lava, lava damage x1/8. (JASS `Func_PresentShard_Destroy`; the helper `Present*_End` actions revert effects after a duration.)
- Explosion knockback to players: `DealKnockbackAoE`: for living players within radius: `dst' = (1 - d/radius/2)*damage`; velocity += unit_vector * dst' * 1.6; same-team players get `KnockbackPointsAdd` instead of damage, enemies take `dst'/inflict-multiplier` damage. `Obj_CollData` bit mask prevents hitting a player twice by the same explosion.
- Pillars are **solid for missiles and players** but not for AoE.

## 5. Ice

`IceEnabled` (host `-ice` toggles, off in league mode). If on, each round has a 50 % chance (`GetRandomInt(0,1)`) of an ice patch: centre uniform in [-750,750]^2, shape random circle/square, size 2-3 cells (`TerrainChange`). It shrinks with the lava (`SetSafeTerrainSize`). On ice: friction 0.93 instead of 0.96, no instant stop, acceleration only `Stats_IceAcc = Stats_MS/10`, and HP regen still applies (see 04-physics-knockback.md).

## 6. Hidden-layer / misc

- Player spawn when the round starts: ring (see overview). In team games, the team shares one random angle; in FFA each player gets a slot on the ring, shuffled by `InitRandomCircle` (JASS 18514).
- Dead players (round mode, `Death`) spend the rest of the round at the shop: their Merchant is recreated at (maxX, maxY) and they get "You have died and will respawn when this round ends." (`Death`).
- Observers (`-unobs`/obs mode) see the arena; see 06.

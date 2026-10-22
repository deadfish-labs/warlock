# Source: what the map data tells us, and what it doesn't

Can the remake be built from the map file alone, or do we need Warcraft III itself? This page answers that for each
kind of information.

## Why most of the game is in the map file

Warlock 1.06n does not rely on Warcraft III's own spell and movement systems for its core. It ships its **own physics
engine in its script** (the `ObjectStruct` library, timer-driven at 0.03 s): every warlock, missile, pillar and shard is
an object the script moves, collides and pushes itself. So the script holds the real rules:

- **Projectiles:** speed, lifetime (and so range), size, what they collide with, what happens on impact.
  Example: Fireball flies at **1000 units/s**, lives `1 + 0.12 × level` s, size 25, deals `6.5 + 0.5 × level` damage
  with knockback factor 1, cooldown 4.8 s (chapter 01 §4.1).
- **Area spells:** radius and how damage and knockback fall off inside it.
- **Damage and knockback:** the formula `(100 + DP) × damage × 0.03 × multipliers` and the collision and friction model
  (chapter 04).
- **Durations, channel times, toggles, buffs and curses.**
- **Lava, shrinking, rounds, gold, points and every mode's rules** (chapters 03 and 05).

The object data files (abilities, units, items, upgrades) add cooldowns, costs, levels and tooltips.

## What the map file does not contain

| Missing | Why | How we get it |
|---|---|---|
| Warcraft III's **default values** for fields the map didn't change (e.g. some unit turn rates, cast-point timings, collision sizes) | The map stores only its changes; the defaults live in the game's own data tables. | The base data tables (`UnitData`, `AbilityData`, ...) from a Warcraft III install, or the community's published dumps of them. Only a handful of fields matter because the custom physics overrides most of them. |
| **How spells look and sound** | Models, particles and sounds are Warcraft III assets. | Not needed: the remake has original art (decisions). Gameplay videos show timing and readability. |
| **Feel** (how sliding, aiming and dodging feel at speed) | Numbers describe it but don't show it. | Watch recorded games; ideally play the map. |
| The **exact meaning of some code paths** | A few behaviours disagree with their tooltips; chapter 01 marks 12 of them *uncertain*. | Read the code path again, or test in the game. |

## Do we need to install Warcraft III?

**Not for the numbers.** The script and object data hold almost every rule. Installing the game would help with three
things: reading the base data tables, settling the *uncertain* points by testing, and getting the feel by playing.
Warcraft III: Reforged is a paid Windows/macOS game on Battle.net and can't run on our Linux servers, so this would be
done on a player's own PC. Recorded games are a good second source for feel and timing.

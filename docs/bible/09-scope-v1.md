# 09 — Scope of the first version

What the first playable version of the remake contains. Decided by the owner on 2026-10-22 (see `decisions.md`).

## In

| Area | First version |
|---|---|
| Characters | All five: **Warlock**, **Heretic**, **Inquisitor**, **Occultist**, **Enchanter** (chapter 00 §4, chapter 01). Same base stats; each has its own F spell. Host options: Warlock only (vanilla), random, all pick. |
| Spells | Every spell of 1.06n, with levels, upgrades and alternate forms (chapter 01). |
| Items and shop | Every item, the shop phases and research (chapter 02). |
| Arena | Lava, shrinking, the terrain themes, pillars, ice and the arena shape options (chapter 03). |
| Physics | The original movement and knockback model at its original tick (chapter 04). |
| Modes | **Round** (default) with **duel** (two players or two teams: first to 6) and **teams**; **Deathmatch**; **Avatar**; **King** (chapter 05). |
| Players | 2 to 12 per match. |
| Platforms | Web (keyboard + mouse) and mobile (virtual joystick + touch spell buttons). |

## Out

- **King of the Hill.** The mode exists in the 1.06n code, but its hill rule was never finished: in practice it plays
  like Deathmatch. Not remade.
- **Bots.** Not needed at the start.

## Not decided yet

- **Cosmetics and tokens:** players earn permanent tokens every round and spend them on looks only (the "Lich" look,
  titles, hats, fireball colours). Needs player accounts and storage.
- **League:** the original exported match statistics to an external ranking site (Warlock League) and had
  league/competitive rule presets (`-league`, `-cm`, `-mgl`). The presets are cheap to keep as rule sets; the ranking
  needs accounts, a database and anti-cheat on the server (the authoritative Colyseus server already prevents most
  cheating, chapter 08).

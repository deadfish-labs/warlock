# 13 — Art and audio direction

## Decision (owner, 2026-10-22): two stages

1. **Test skin (internal only, never public):** the first playable versions look like the original — Warcraft III's
   look, or assets very close to it — so we can judge the game's feel against the map we know, without new art
   getting in the way.
2. **Release skin:** replaced by a **more realistic** look with a **different theme** of our own before anything is
   public.

## Rules for the test skin

The Warcraft III models, textures and sounds belong to Blizzard. To keep the project clean:

- **Never in this repository** (it is public) and never in any public build or deploy. Test-skin files live in private
  storage and are loaded only by internal test builds.
- Preferred source, in order:
  1. **Free look-alike assets** with a licence that allows any use (for example low-poly fantasy character and effect
     packs released as CC0): same top-down, chunky, readable fantasy style as Warcraft III, and nothing to remove later.
  2. **Original Warcraft III assets** exported from an owner's own Warcraft III install, only for private testing.
- The community-made models referenced by the map (Warlock, Heretic, Fire Mage/Inquisitor, Occultist) are credited if
  used as reference.

## Skins are swappable from day one

So the test skin can be replaced without touching the game:
- The game never refers to a file directly; it asks a **skin manifest** for "warlock model", "fireball missile",
  "lava tile", "fireball cast sound", etc.
- A skin is a folder of models (glTF), textures (KTX2), effects and sounds plus that manifest.
- Gameplay sizes come from the simulation (chapter 04), not from the models, so a new skin can't change the game.
- Test skin and release skin can both exist; the build picks one.

## Release skin (to define later)

- Realistic, with a new theme (not Warcraft's).
- Has to stay readable from the fixed top-down camera: who is who, which spell is coming, where the lava edge is.
- Has to run on weak phones (chapter 12): quality levels decide detail, not the art style itself.

## Audio

Same two stages. The release audio is original: music, spell sounds, voice lines. The test skin may use placeholders.

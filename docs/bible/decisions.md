# Decisions

Every decision about the remake, newest last. Each one is dated and says who made it. Chapters link here when a
decision changes what they describe.

| Date | Decision | By |
|---|---|---|
| 2026-10-22 | **Source version:** the remake follows the map in this repository, Warlock **1.06n** (community edition). The 0.99c seen in gameplay videos is not the target. | Owner |
| 2026-10-22 | **Name and rights:** we have permission to use the name **Warlock** and to remake the game. | Owner |
| 2026-10-22 | **Art and sound:** all original; nothing from Warcraft III is used. | Owner |
| 2026-10-22 | **Server:** Colyseus, authoritative (chapter 08). | Owner |
| 2026-10-22 | **Platforms:** web (browser) and mobile. | Owner |
| 2026-10-22 | **Controls on the web:** keyboard and mouse. | Owner |
| 2026-10-22 | **Controls on mobile:** a virtual joystick to walk and touch buttons to cast spells. | Owner |
| 2026-10-22 | **Match size:** up to 12 players, as in the original. | Owner |
| 2026-10-22 | **Bots:** not needed at the start. | Owner |
| 2026-10-22 | **Game modes:** Round (with duel and teams), Deathmatch, Avatar and King. **King of the Hill is not remade** (its rule was never finished in 1.06n). | Owner |
| 2026-10-22 | **First version scope:** all five characters, every spell and item, the modes above (chapter 09). | Owner |
| 2026-10-22 | **Cosmetics/tokens and league:** in the first release, built after the core game (roadmap phase 2, chapter 10). | Owner |
| 2026-10-22 | **Lobby platform** above the games (room list, friends, groups with a leader, voice, joining together, auto-match, several games): roadmap phase 3 (chapter 11). The game comes first. | Owner |
| 2026-10-22 | **Client rendering:** Three.js with the WebGPU renderer (WebGL2 fallback), in TypeScript; the simulation is a separate core that may move to Rust/WebAssembly only if profiling asks for it (chapter 12). | Owner |
| 2026-10-22 | **Camera:** the same as the original map: 3D world seen from a fixed top-down camera. | Owner |
| 2026-10-22 | **Performance target:** must run well on **weak phones** without giving up visual quality where the hardware allows it (quality levels, chapter 12). Optimisation is a first-class requirement. | Owner |
| 2026-10-22 | **In-game voice chat** (chapter 14): push to talk (default) or open mic, mute yourself, mute or turn down any player. Self-hosted LiveKit voice server, separate from Colyseus. Team modes: teammates only during the fight, everyone in the shop; the dead hear the living but not the other way round. Built at the end of phase 1. | Owner |

## Open

- The weakest phone we test on (a concrete device or benchmark score for the "weak phone" target).
- How the original's click-to-move and aimed spells map onto a joystick and touch buttons (aiming on touch).

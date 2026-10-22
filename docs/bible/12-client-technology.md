# 12 — Client technology (proposal)

How the game is drawn in the browser and on phones, and where performance really comes from. **Status: proposal,
waiting for the owner's decision** (2026-10-22). The owner's goal: use the player's graphics card through the browser
and be fast enough for more players, more effects and more animation later.

## Where the time goes in a game like Warlock

| Work | Size in Warlock | Where it runs |
|---|---|---|
| Simulation: movement, knockback, collisions, spells | 12 warlocks + a few dozen missiles at 33 ticks/s: small | Server (authoritative); a copy on the client for prediction |
| Drawing: arena, characters, missiles, particles, lava, lights | The big cost, and it grows with "more effects" | The GPU |
| Network: state from the server | Small per player; grows with players | Colyseus (chapter 08) |

So "more players and more effects" is mostly a **GPU** problem. The language the page is written in matters much less
than how the drawing is organised (instancing, GPU particles, few draw calls, small textures).

## The options

| | **A. TypeScript + Three.js (WebGPU renderer)** | **B. TypeScript + Babylon.js** | **C. Rust → WebAssembly (e.g. Bevy on wgpu)** |
|---|---|---|---|
| GPU through the browser | WebGPU, with WebGL2 fallback | WebGPU, with WebGL2 fallback | WebGPU, with WebGL2 fallback |
| Effects | Node-based shaders (TSL), compute shaders for GPU particles | Built-in GPU particles, node material editor | Full control, more to write |
| Load time on phones | Small (hundreds of KB to a few MB) | Medium | Large (WASM binaries of many MB), slower first start |
| Colyseus client | Official TypeScript SDK | Official TypeScript SDK | No Rust SDK: needs a JavaScript bridge |
| Shares code with the server | Yes: server and client are both TypeScript | Yes | Only if the server also runs the Rust core as WASM |
| Mobile browsers | Mature | Mature | Works, but less tested; threads and memory are more limited |
| Building with AI | Very large body of examples | Large | Smaller, more breakage between versions |

## Recommendation

**A hybrid: draw with Three.js (WebGPU renderer, WebGL2 fallback) in TypeScript, and keep the simulation as a separate
"core" that can move to Rust/WebAssembly later if measurements ask for it.**

1. **Rendering: Three.js with WebGPU.** It reaches the graphics card directly (WebGPU is the modern browser graphics
   API, with compute shaders), falls back to WebGL2 where WebGPU is missing, and keeps the download small for phones.
2. **Simulation core: plain TypeScript first,** shared by the Colyseus server and the client (prediction). It has no
   rendering or network code inside, so it can be tested on its own.
3. **Rust → WebAssembly only where it pays:** if profiling shows the simulation is the bottleneck (for example, far more
   players or physics objects than Warlock has), the core is rewritten in Rust and compiled to WebAssembly, and the same
   module runs inside the Node.js server and in the browser. Rendering stays in Three.js.

Why not all-Rust from the start: the cost of Warlock is drawing, which the GPU does either way; an all-Rust client
loads slower on phones, has no Colyseus SDK, and moves us away from the most-used tools for no measured gain.

## Performance rules from day one

- **Instancing:** draw all warlocks, missiles and pillars of a kind in one call.
- **GPU particles:** fire, smoke, sparks and lava bubbles simulated in compute shaders, not one object each.
- **No garbage in the loop:** reuse objects and arrays every frame (pools), so the browser never pauses to clean up.
- **Fixed simulation step, free render rate:** the simulation ticks at the original 0.03 s; drawing runs at the
  screen's rate and interpolates between ticks.
- **Small assets:** glTF models, KTX2/Basis compressed textures, texture atlases.
- **Quality levels:** effects, shadows and resolution scale down automatically on weaker phones.
- **Off the main thread where possible:** rendering in a worker with OffscreenCanvas, so input and UI stay smooth.
- **Measure before optimising:** a built-in frame-time and network overlay from the first prototype.

## Open questions

- 3D with a fixed top-down camera (like the original) or 2D? The proposal assumes **3D**.
- Minimum phones to support (sets the quality levels).
- Art style (chapter 13), which also decides how heavy the scenes are.

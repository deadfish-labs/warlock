# 14 — In-game voice chat

Status: **proposal** (2026-10-22). Players in a match talk to each other by voice in real time. Each player can mute
other players, mute themselves, and choose **push to talk** or **open mic**.

The original map has no voice: Warcraft III only had text chat (chapter 06). This is new for the remake.

## What the player gets

| Feature | How it works |
|---|---|
| **Voice mode** | One setting with three choices: **Off**, **Push to talk** (default) and **Open mic**. |
| **Push to talk** | Desktop: hold a key (default `V`, can be changed). Mobile: hold a mic button on the touch HUD, with an option to tap once to turn it on and again to turn it off. The mic stays live for about 200 ms after release so the last word isn't cut off. |
| **Open mic** | The mic sends whenever the player is speaking. A sensitivity slider with a live level meter sets when it opens, so breathing and keyboard noise stay out. |
| **Mute myself** | One key (default `M`) or a HUD button. It works in both modes. Other players see a crossed-out mic next to the name. |
| **Mute a player** | From the scoreboard (`Tab`) or the player list: a mute button and a volume slider per player. Only the person who mutes stops hearing that player; nobody else is affected. The muted player is not told. |
| **Who is talking** | A speaker icon lights up next to the name on the scoreboard and above the warlock in the arena. |
| **Deafen** | One button silences all incoming voice. |
| **Settings** | Microphone and speaker choice, mic test with a level meter, master voice volume, and noise suppression on or off. |

Voice is opt-in. The game asks for the microphone only when the player first turns voice on. A player without a mic
can still listen.

## Who hears whom

- **Free-for-all modes** (Round, Deathmatch, King): everyone in the match.
- **Team modes** (Round with teams, Avatar): **proposal:** teammates only during the fight, and everyone between
  rounds in the shop phase. **Open question** for the owner.
- **Spectators and dead players:** **proposal:** dead players keep hearing the living, but the living don't hear the
  dead until the round ends, so nobody spies for a living teammate. **Open question.**

## Technology

### Why not peer-to-peer

With up to 12 players, a fully peer-to-peer setup (each player sends their voice directly to everyone else) means
each phone uploads 11 copies of its voice. That is too much for weak phones and mobile data, and those phones are a
requirement (chapter 07).

### Proposed: a voice server (SFU) with LiveKit

Each player sends their voice once to a voice server, and the server forwards it to the others. This kind of
server is called an SFU (selective forwarding unit).

- **LiveKit** is a good fit. It is open source (Apache 2.0) and self-hostable, uses WebRTC in every modern browser,
  and has client libraries for the web and for React Native / native mobile. Alternatives are mediasoup (lower level,
  more work) and a hosted service (Agora, Daily, hosted LiveKit), which costs money per minute.
- Voice runs **separately from the game server**. The Colyseus room stays the authority for gameplay. Voice never
  passes through it, so voice traffic can't slow down the simulation.
- **Joining:** when a player joins a Colyseus match, the game server gives them a short-lived voice token for that
  match's voice room. The token carries their player id and which team channels they may hear. Nobody can join a
  match's voice without being in the match.
- **Teams:** the voice server decides who receives which voice from the token permissions (per-track subscription
  permissions in LiveKit). The client is not trusted with this.
- **Muting a player** = the client stops receiving that player's audio, which also saves bandwidth. **Muting myself**
  and push to talk mute the microphone track instead of disconnecting it, so talking starts again instantly.
- **Audio:** Opus codec at about 24–32 kbit/s per speaker. The browser's echo cancellation, noise suppression and auto
  gain are on. Only people who are actually talking use bandwidth, so a full 12-player match with 2–3 people
  talking costs a phone well under 100 kbit/s down.
- **Networks:** the voice server includes a TURN relay for players behind strict NAT or firewalls (some mobile
  carriers, offices).
- **Latency target:** under 300 ms mouth to ear.

### Platform details

- **Browsers** only play audio after a click or tap. The "Join match" click already counts.
- **iOS Safari** stops the microphone when the tab goes to the background. Voice reconnects when the player comes back.
- **Bluetooth headsets** on phones can switch to a low-quality call mode when the mic opens. Push to talk keeps
  that to the moments a player speaks. This needs testing on real phones.

## Safety

- Per-player mute is always available and instant.
- **Report a player** from the same menu (it needs accounts, phase 2). Muted players are remembered per account
  from phase 2. Before that, they are remembered for the match only.
- Voice is not recorded or stored by us.
- A host or moderator can't listen in without being in the match.

## Where it sits on the roadmap

**Proposal:** build it at the end of phase 1, after the core game works online, because it only needs the match
(no accounts). Phase 3 lobby voice (chapter 11) reuses the same voice server, with rooms for groups instead of
matches. **Owner to confirm.**

## Testing

- 12 clients in one match (a script with fake audio) to check server load and latency.
- Real devices: a weak Android phone on mobile data, an iPhone with Safari, desktop Chrome and Firefox.
- Push to talk timing, open-mic sensitivity, mute and unmute, team-only hearing, reconnecting after a network drop.

## Open questions

1. Team modes: teammates only during the fight, or everyone? (proposal above)
2. Can dead players talk to the living? (proposal above)
3. Default mode: push to talk (proposed), open mic, or off?
4. Hosting: self-hosted LiveKit next to the game servers (proposed), or a paid hosted service?
5. Phase: end of phase 1 (proposed), or later?

# Car Racing Game — Design Spec

**Date:** 2026-04-07
**Status:** Approved
**Project name:** TBD (working title: "Turbo Clash")

## Overview

A multiplayer car racing game with combat and stunts, inspired by Mini Militia's multiplayer model. Supports up to 10 players over LAN/hotspot or internet. Features four progressively unlocked camera views (top-down 2D → side-scroll 2D → isometric 2.5D → 3D third-person) that reward player progression.

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Game client | Godot 4 + GDScript |
| Backend server | Node.js |
| LAN networking | ENet (Godot built-in) |
| Internet networking | WebSocket |
| Target platforms | Android, iOS, Web (HTML5), Desktop (Windows/Linux/Mac) |
| Monetization | Deferred — architecture supports plugging in later |

## Architecture

### Two main components

1. **Godot Client** — the game itself. Contains all game logic, rendering, physics, UI, and networking code. Single codebase exports to all platforms.
2. **Node.js Backend** — lightweight server handling matchmaking, room code management, and WebSocket relay for internet play. Not required for LAN/hotspot mode.

### Network model: Host-authoritative

- One player (or the server in matchmaking fallback) runs the authoritative game simulation.
- Peers send input to the host; the host validates and broadcasts game state.
- Client-side prediction: each player's car moves instantly on their screen. If the host disagrees, position is corrected via smooth interpolation.
- This prevents cheating and keeps all 10 players in sync.

### Shared track format

Tracks are defined as data (waypoints, boundaries, obstacles, ramp positions, power-up spawn points). Each view (top-down, side-scroll, isometric, 3D) renders the same track data differently. This means tracks are authored once and work across all four views.

## Connection Modes

### 1. LAN / Hotspot

- Host clicks "Create LAN Game" → Godot starts ENet server on local IP.
- Other players click "Join LAN" → auto-discover via UDP broadcast, or enter host IP manually.
- No internet required. Works on airplane, in classroom, anywhere.
- Latency: ~1-5ms.

### 2. Internet Room (friend play)

- Host clicks "Create Room" → backend generates a room code (e.g., `RACE-7X4K`).
- Friends enter the room code → backend relays connections via WebSocket.
- NAT traversal handled by the relay server (no port forwarding needed).
- Host still runs game logic; server only relays packets.
- Latency: ~30-100ms typical.

### 3. Matchmaking (random opponents)

- Player clicks "Find Match" → joins matchmaking queue with their ELO rating.
- Server groups 2-10 players by skill (ELO ± 200 range).
- If not enough players after 30s, range expands progressively.
- Lowest-ping player auto-assigned as host.
- Fallback: server-hosted if no suitable host found.

## Gameplay Systems

### Racing core

- 3 laps per race.
- Race flow: Lobby (countdown) → 3-2-1 start → Race → Results & XP → Back to lobby.
- Configurable lap count in private rooms.

### Combat system

**Weapons (picked up on track):**

| Weapon | Behavior |
|--------|----------|
| Missile | Fires forward, slight homing |
| Oil Slick | Drops behind, causes spin-out |
| EMP Blast | Short range, disables nearby cars briefly |
| Shield | Blocks one incoming hit |
| Boost Pad | Instant speed burst, can ram opponents |

**Combat rules:**

- Cars have 3 HP per race.
- Getting hit = spin-out + slow-down for 1.5 seconds.
- 0 HP = wreck → respawn after 3 seconds at last checkpoint.
- Weapon pickups respawn every 10 seconds.
- Players hold only 1 weapon at a time.
- Rubber-banding: trailing players receive stronger weapon pickups.

### Stunt system

**Tricks:**

- Barrel Roll — spin in air off ramps.
- Flip — front/back flip off jumps.
- Drift — sustained drift around corners.
- Near Miss — pass close to obstacles or other cars.
- Hang Time — stay airborne for extended duration.

**Boost meter:**

- Tricks fill a boost meter (0-100%).
- Player can activate boost anytime for a speed burst.
- Chaining tricks = combo multiplier (1.5x at 2 tricks, 2x at 3+).
- Crashing during a trick = lose all meter progress.
- Core risk/reward: bigger tricks fill more meter but are harder to land.

## Progression System

### Player leveling

| Level | Unlock |
|-------|--------|
| 1 (start) | Top-Down 2D view, 1 car, 2 tracks, basic weapons (missile, shield) |
| 5 | Side-Scroll 2D view, 3 cars, 4 tracks, stunts enabled |
| 15 | Isometric 2.5D view, 6 cars, 6 tracks, all weapons |
| 30 | 3D Third-Person view, all cars & tracks, ranked mode |

### XP sources

| Action | XP |
|--------|-----|
| Finishing a race | 50-200 (by position) |
| Combat knockout | 30 per knockout |
| Stunt combo | 10-50 (by combo size) |
| Daily first win bonus | 100 |
| Clean lap (no hits taken) | 25 |

## Multiplayer Sync

### Data sync strategy

- **60 times/sec:** Car positions, rotations, velocity (UDP-style unreliable channel).
- **On event (reliable):** Weapon fired, hit detected, HP change, stunt started/landed, boost activated, lap completed, race finished, player disconnected, power-up spawned/picked up.

### Disconnect handling

- **Player disconnects:** Car goes AI-autopilot for 15 seconds. If no reconnect, car removed from race.
- **Host disconnects:** Host migration — next player in the player list becomes new host. Game pauses for 3 seconds during transfer.

## Content scope (MVP)

For the initial playable version:

- **1 view:** Top-down 2D only.
- **2 tracks:** One simple oval, one with ramps and obstacles.
- **1 car model** with 3 color variations.
- **3 weapons:** Missile, oil slick, shield.
- **LAN multiplayer** only (no backend server needed for MVP).
- **Basic stunt system:** Drift and barrel roll only.
- **No progression system** in MVP — unlocks come later.

This gets a playable game in hands quickly. Views, weapons, tracks, and progression are layered on incrementally.

## Out of scope (for now)

- Monetization / in-app purchases.
- Account system / login.
- Leaderboards.
- Replay system.
- Track editor.
- AI opponents (single-player).
- Chat / voice comms.

These can be added in future iterations without architectural changes.

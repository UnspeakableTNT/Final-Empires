# Final Empires

Real-time multiplayer strategy: you start as a single nation, try to swallow the whole map, and then survive the **Final Empire** — an end-game boss nation that awakens late and comes for everyone.

**▶ Play it: https://finalempires.com**

This repository is **not** the game's source. The code stays private; this is where I write about how the game is built: the engineering, the systems, and the occasional bug that nearly breaks everything.

## About the game

- Browser-based, nothing to install, runs on Cloudflare.
- Real-time multiplayer on a host-authoritative simulation (one host runs the world, clients stay in sync).
- 195 real-world nations, plus procedurally generated filler nations for large maps (400+).
- Bots with difficulty tiers from Easy up to Impossible.
- A full war economy (cities, factories, troop camps, ports) and a high-tech arsenal: jet strikes, warships, submarines, and nuclear / MIRV options.
- An infamy-and-sanctions system, so conquering the world has a cost.
- The **Final Empire** end-game boss that turns the late game into a survival fight.

## Technical write-ups

Devlogs and post-mortems on building and debugging the game:

- **[How three characters almost killed my game's final boss](reports/final-empire-bug-case-study.md)** — a debugging post-mortem on an identity-collision bug that quietly crippled the end-game boss.

*(more on the way)*

## Screenshots

<!-- Drop images into a /screenshots folder and link them here, e.g. ![Map](screenshots/map.png) -->

_Coming soon._

## About / contact

Built by Martin Garas. Find me at https://mut-studios.itch.io/ https://www.youtube.com/@MUT-Studios.

---

*Want to follow along? Watch or star the repo — I post a new write-up whenever something interesting breaks.*

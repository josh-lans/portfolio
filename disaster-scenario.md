# Disaster Scenario: The Game Show

*A comedic board-deckbuilder party game for Steam and mobile. An independent project — built solo, end to end.*

[← Back to portfolio](README.md) · 🎮 [Full showcase & gallery →](https://github.com/josh-lans/disaster-scenario-game)

![Disaster Scenario gameplay](screenshots/disaster-scenario/gameplay.png)

## What it is

Picture *Wipeout* crossed with *Slay the Spire* and *Jackbox*. Contestants race down a game-show obstacle course where every square is an absurd disaster. You clear each one by playing **tool cards** and **pitching a ridiculous plan**, the table **bets karma** on whether you'll survive, and the dice decide. The app keeps score; the humans do the judging — so multiplayer needs zero AI and has zero latency.

## Why it's here

The first two projects are enterprise platforms. This one is a *game*, in a different engine, with a different toolchain — and it was built with the same method. That's the point: the approach travels.

## How it's built

- **Godot 4 (~39,000 lines of GDScript)** with a pure, deterministic, unit-tested rules core (14 CI test gates) cleanly separated from rendering and networking.
- **Online multiplayer over ENet**, plus a solo roguelite against bots. Cross-platform from one codebase (Steam + mobile), with controller and touch support.
- The **3D art was produced with a multi-agent AI pipeline** — one model writes the asset brief, another generates it, a third renders it into the game through Meshy and Scenario. ([How that orchestration works →](https://github.com/josh-lans/agentic-dev-kit))

➡️ **[See the full gallery and pitch in the showcase repo](https://github.com/josh-lans/disaster-scenario-game)**

---

*Joshua Lans · [github.com/josh-lans](https://github.com/josh-lans) · [LinkedIn](https://www.linkedin.com/in/joshualans/)*

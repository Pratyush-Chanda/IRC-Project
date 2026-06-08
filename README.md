# CJP Decentralised Communications Project

> *Free speech. Resilient infrastructure. Community-owned networks.*

---

## What Is This?

This project builds a layered, decentralised communication network for communities organising around civil rights, protest, and accountability. It is designed to survive network disruptions, jammer deployments, platform takedowns, and corporate or state interference.

It is not a replacement for Discord or Telegram overnight. It is a parallel infrastructure that grows alongside existing tools — and is ready to take over when those tools fail.

---

## Why This Exists

Centralised platforms — Discord, Telegram, WhatsApp — are single points of failure. They can be:

* **Surveilled** by state or corporate actors
* **Subpoenaed** and forced to hand over data
* **Deplatformed** with no warning or appeal
* **Jammed or blocked** at the network level during critical moments

This project addresses that. The goal is infrastructure that no single entity controls and no single failure can bring down.

---

## Architecture Overview

The network is built in three layers, each sitting on top of the next as a fallback:

```
┌─────────────────────────────────────────┐
│         LAYER 3 — Matrix Server         │  ← Primary (internet required)
│   Rich messaging, federation, history   │
├─────────────────────────────────────────┤
│         LAYER 2 — IRC Network           │  ← Fallback (internet required)
│   Lightweight, relay-based, WebIRC UI   │
│   Mobile phones + Raspberry Pi nodes    │
├─────────────────────────────────────────┤
│     LAYER 1 — Mesh / LoRa / BT P2P      │  ← Last resort (no internet)
│   Meshtastic, Bluetooth, local only     │
└─────────────────────────────────────────┘
```

Each layer is independent. If the top layer goes down, the one below keeps working.

See [`TECHNICALS.md`]() for a full breakdown of how each layer works and how they interact.

---

## Phases

### ✅ Phase 1 — IRC Foundation *(active)*

Set up a basic IRC network using mobile phones and Raspberry Pi nodes as relays, with a WebIRC browser frontend so anyone can connect without installing anything.

→ [`PHASE-1.md`]() — Concept and design

→ [`PHASE-1-TECHNICALS.md`]() — Code, setup, and repo structure

### 🔜 Phase 2 — Matrix Deployment

Deploy a federated Matrix homeserver (via VPS + volunteer-hosted redundant nodes) as the primary communication layer, with IRC as fallback.

### 🔜 Phase 3 — Mesh Integration

Integrate Meshtastic/LoRa hardware and Bluetooth P2P for fully offline, last-resort communication during full network shutdowns.

---

## Who Is This For?

* Community members who want secure, private communication channels
* Volunteers willing to run relay nodes on their Android phones or a Raspberry Pi
* Technically experienced contributors who can help with setup, documentation, and troubleshooting
* Journalists, whistleblowers, and organisers who need channels that can't be easily monitored or shut down

---

## How To Contribute

1. Read [`TECHNICALS.md`]() to understand the architecture
2. Pick the phase you want to contribute to
3. Open an issue or pull request
4. Join the discussion on the CJP Discord `#tech-and-networking` channel *(pending approval)*

We especially need:

* Android users willing to run relay nodes
* Anyone with a spare Raspberry Pi
* People who can write clear setup guides for non-technical members

---

## Project Status

| Phase             | Status         | Lead           |
| ----------------- | -------------- | -------------- |
| Phase 1 — IRC    | 🟡 In Planning | @hsxtheemperor |
| Phase 2 — Matrix | 🔴 Not Started | Open           |
| Phase 3 — Mesh   | 🔴 Not Started | Open           |

---

## Contributors

* [@hsxtheemperor](https://github.com/hsxtheemperor) — Project Lead
* [@Bhavin](https://github.com/) — Infrastructure / IT (10 yrs exp)

*Want to be added? Open a PR or reach out on Discord.*

---

## License

This project is open source. All code, configs, and documentation are free to use, modify, and redistribute. Attribution appreciated.

# TECHNICALS — Architecture Reference

> This document explains how each communication layer works, what hardware it needs, what scale it operates at, and how the layers sit on top of each other.

---

## The Layered Architecture

The network is designed as a  **resilience stack** . Each layer operates independently, but they are designed to complement and fall back to each other gracefully.

```
Normal conditions     →  Matrix (Layer 3)
Disruption/outage     →  IRC   (Layer 2)
Full network shutdown →  Mesh  (Layer 1)
```

---

## Layer 3 — Matrix

### What It Is

Matrix is a modern, open, federated messaging protocol. Think of it as a decentralised version of Discord — it supports text, voice, video, file sharing, and persistent message history. The most common client is **Element** (available on Android, iOS, and browser).

### How It Works

* Anyone can run a **homeserver** — a self-hosted Matrix instance
* Homeservers talk to each other via **federation** — a user on `matrix.cjp.org` can message a user on `matrix.volunteer.net`
* If one homeserver goes down, others are unaffected
* Messages are stored on the homeserver, so history is preserved even when clients go offline

### What Sits On Top

Nothing — this is the primary layer. Matrix is the main interface for day-to-day communication when internet is available.

### What Sits Below

IRC (Layer 2) acts as fallback. A Matrix-IRC bridge can be configured so both networks share the same channels during transitions.

### Hardware Requirements

| Setup               | Hardware                   | Notes                                       |
| ------------------- | -------------------------- | ------------------------------------------- |
| Minimal (Conduit)   | Raspberry Pi 4, 2GB RAM    | Lightweight Rust-based homeserver           |
| Standard (Dendrite) | Pi 4, 4GB RAM or small VPS | Go-based, more features                     |
| Full (Synapse)      | VPS with 2 vCPU, 2GB+ RAM  | Python-based, resource-heavy, most features |

### Scale

* **Per node** : Handles tens to hundreds of users comfortably on modest hardware
* **Federation** : Effectively unlimited — more homeservers = more redundancy
* **Geographic** : Internet-wide

---

## Layer 2 — IRC

### What It Is

IRC (Internet Relay Chat) is one of the oldest real-time messaging protocols, dating to 1988. It is extremely lightweight, battle-tested, and runs on almost anything. Communication happens in **channels** (chat rooms prefixed with `#`), and messages are relayed between servers in real time.

### How It Works

* Users connect to an **IRC server** (daemon) using a client app or via browser through **WebIRC**
* Servers can **link** to each other to form a network — a message sent on one server propagates to all linked servers
* There is no central authority — anyone can run a server and link it in
* Mobile phones running an IRC daemon (via Termux on Android) can act as  **relay nodes** , extending the network without dedicated hardware

### What Sits On Top

Matrix. In normal conditions, Matrix is primary and IRC runs quietly as a parallel fallback. A bridge keeps them in sync.

### What Sits Below

Mesh/LoRa (Layer 1) as last resort when internet is fully cut.

### Hardware Requirements

| Setup         | Hardware                | Notes                                |
| ------------- | ----------------------- | ------------------------------------ |
| Minimal relay | Android phone + Termux  | Runs `ngircd`in background         |
| Standard node | Raspberry Pi Zero W     | Incredibly low power draw            |
| Full server   | Raspberry Pi 3/4 or VPS | Can serve 100s of simultaneous users |

### Scale

* **Per node** : A single Pi can handle hundreds of concurrent IRC connections
* **Network** : A chain of linked servers can scale to thousands of users
* **Geographic** : Internet-wide, but each relay only extends the network if it has uplink

### WebIRC

WebIRC is a browser-based IRC frontend. Users visit a URL and connect to the IRC network without installing any app. Options include:

* **The Lounge** — self-hosted, persistent, polished UI
* **KiwiIRC** — lightweight, embeddable
* **Gamja** — minimal, single-page, easy to self-host

---

## Layer 1 — Mesh / LoRa / Bluetooth P2P

### What It Is

This layer operates entirely without internet. It uses radio frequencies (LoRa) and Bluetooth to create a local mesh network where devices communicate directly with each other. The most accessible implementation is  **Meshtastic** .

### How It Works

* **LoRa (Long Range Radio)** : Low-power radio signals that can travel 1–10km per hop depending on terrain. Each device acts as a relay — messages hop from node to node until they reach their destination.
* **Bluetooth** : Short-range (10–30m) P2P connection between phones. Useful in dense crowds where people are physically close.
* **Meshtastic** : An open-source firmware for cheap LoRa radios (~₹1500–3000 for hardware). Devices form a self-healing mesh automatically.

### What Sits On Top

IRC (Layer 2). In situations where internet is partially available, LoRa mesh nodes can act as bridges — routing local mesh traffic out through any node that has uplink.

### What Sits Below

Nothing — this is the floor. If all else fails, Meshtastic keeps local comms alive.

### Hardware Requirements

| Setup          | Hardware                        | Notes                       |
| -------------- | ------------------------------- | --------------------------- |
| Minimal        | Android phone + Meshtastic app  | Bluetooth only, short range |
| LoRa node      | TTGO T-Beam or Heltec LoRa32    | ~₹1500–3000, 1–5km range |
| Extended range | Directional antenna + LoRa node | 10km+ in open terrain       |

### Scale

* **Per node** : Each LoRa node covers 1–5km radius
* **Mesh** : Each additional node extends the network. 10 nodes in a city can cover a significant area
* **Geographic** : Purely local — useful for protest grounds, neighbourhoods, or areas under blackout

### Limitations

* **Low bandwidth** : Text only, no large files or voice
* **Range** : Limited — not suited for city-wide or inter-city comms alone
* **Latency** : Messages can take seconds to route through multiple hops

---

## How the Layers Stack

```
                    INTERNET AVAILABLE
                           │
              ┌────────────▼────────────┐
              │      Matrix (L3)        │
              │  - Primary channel      │
              │  - Full featured        │
              │  - VPS + Pi homeservers │
              └────────────┬────────────┘
                    Bridge │ (Matrix-IRC bridge)
              ┌────────────▼────────────┐
              │       IRC (L2)          │
              │  - Fallback channel     │
              │  - Mobile relay nodes   │
              │  - Pi servers           │
              │  - WebIRC frontend      │
              └────────────┬────────────┘
                  Uplink   │ (any node with internet routes mesh traffic)
              ┌────────────▼────────────┐
              │    Mesh / LoRa (L1)     │
              │  - Last resort          │
              │  - No internet needed   │
              │  - Text only            │
              │  - Local range only     │
              └─────────────────────────┘
                    NO INTERNET NEEDED
```

### Rule of Thumb

* Run **all three layers simultaneously** when possible
* When internet degrades → fall down the stack automatically
* When internet returns → resume from the top

---

## Raspberry Pi Role at Each Layer

| Layer       | Pi Role                                                                  |
| ----------- | ------------------------------------------------------------------------ |
| Matrix (L3) | Homeserver (Conduit or Dendrite)                                         |
| IRC (L2)    | IRC daemon (ngircd), WebIRC host (The Lounge)                            |
| Mesh (L1)   | Gateway node bridging LoRa mesh to IRC/Matrix when internet is available |

A single Pi 4 can realistically run both an IRC server and a Matrix homeserver simultaneously on a modest network.

---

## Android Phone Role at Each Layer

| Layer       | Phone Role                                                      |
| ----------- | --------------------------------------------------------------- |
| Matrix (L3) | Client (Element app)                                            |
| IRC (L2)    | Relay node via Termux + ngircd (background process)             |
| Mesh (L1)   | Meshtastic node via BT-connected LoRa hardware, or BT-only mesh |

The goal is that any volunteer's Android phone can silently run a relay node in the background, extending the network without requiring dedicated hardware.

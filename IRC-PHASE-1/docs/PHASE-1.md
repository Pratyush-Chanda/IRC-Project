# Phase 1 — IRC Network Foundation

> *Goal: Get a working IRC network up with at least one permanent node, WebIRC access for non-technical members, and a clear path for volunteers to run relay nodes from their phones.*

---

## What We're Building

A small, functional IRC network that:

* Any community member can **join from a browser** (no app install required) via WebIRC
* Volunteers can **extend by running a relay node** on their Android phone or a Raspberry Pi
* Is **independent of any corporation** — no accounts, no data harvesting, no terms of service
* Can be **spun up quickly** during disruptions when primary channels go down

This is not meant to be polished or feature-rich immediately. Phase 1 is about getting the bones in place.

---

## Why IRC First?

Matrix is the better long-term platform, but it's heavier to set up and harder to explain to new volunteers. IRC has almost no dependencies — a server can run on a Pi Zero, a phone, or a ₹500/month VPS. It requires almost no maintenance once running, and the protocol is so simple it can be reimplemented from scratch if needed.

IRC also has decades of proven reliability. It survived the early internet, and it will survive a network disruption.

---

## Network Design

### Topology

```
                    [ WebIRC Frontend ]
                    (browser interface)
                           │
                    [ Primary IRC Node ]
                  (Pi or VPS — always on)
                    /       │        \
                   /        │         \
          [Relay A]    [Relay B]    [Relay C]
         (Android)    (Android)    (Pi Zero)
              │
         [Relay D]
         (Android)
```

* The **Primary Node** is the backbone — it should be as stable as possible (Pi on a reliable home connection, or a VPS)
* **Relay Nodes** are volunteer-run phones or Pis that extend the network and add redundancy
* The **WebIRC Frontend** sits on top of the Primary Node and gives anyone a browser-based entry point

### Channels (Initial)

* `#general` — Main community chat
* `#announcements` — One-way, moderated broadcast channel
* `#tech` — Network maintenance and volunteer coordination
* `#secure` — OpSec discussion (invite/registered users only)

---

## Mobile Phones as Relay Nodes

This is the key idea of Phase 1. Android phones — which most volunteers already have — can run a lightweight IRC relay daemon silently in the background using  **Termux** , a terminal emulator app for Android.

### How It Works

1. Volunteer installs **Termux** from F-Droid (not Play Store — more reliable)
2. Termux runs `ngircd`, a tiny IRC daemon written in C
3. The phone connects to the Primary Node and registers itself as a linked server
4. Other users in physical proximity (or on the same local Wi-Fi) can connect to the phone's relay instead of the primary node
5. The phone relays their traffic through to the main network

### Why This Matters

* During a protest or disruption, even if the primary node is unreachable from some locations, a volunteer's phone nearby can act as a local bridge
* No dedicated hardware needed — just willingness and a charged phone
* The more volunteers run relays, the more resilient the network becomes

### Limitations to Be Honest About

* Phone battery drain is real — relay nodes should be run on charging when possible
* Mobile data is still required for the relay to connect upstream — this is not offline P2P (that's Phase 3/Mesh)
* Android may kill background processes — Termux requires a wake-lock and notification to stay alive

---

## WebIRC — The Public Entry Point

WebIRC means a volunteer can share a single URL — e.g. `http://chat.cjp-net.in` — and anyone can connect to the IRC network from their browser without installing anything.

We'll use  **The Lounge** , a self-hosted WebIRC client that:

* Keeps your session alive even when you close the browser (like a persistent IRC bouncer)
* Has a clean, mobile-friendly UI
* Requires no account creation by default
* Can be password-protected to limit access

The Lounge runs on the same Pi or VPS as the Primary Node, and points at the local IRC daemon.

---

## Phase 1 Milestone Checklist

* [ ] Primary IRC node running (`ngircd`) on Pi or VPS
* [ ] WebIRC frontend live (`The Lounge`) and accessible via browser
* [ ] At least 2 volunteer relay nodes tested and running on Android (Termux)
* [ ] Basic channel structure set up (`#general`, `#announcements`, `#tech`, `#secure`)
* [ ] Setup guide written for non-technical volunteers
* [ ] Connection tested during a mock disruption scenario (Wi-Fi off, mobile data only)

---

## What Phase 1 Does NOT Cover

* Offline/no-internet communication (→ Phase 3, Mesh)
* Matrix federation (→ Phase 2)
* End-to-end encryption at the channel level (IRC is not E2EE by default — use OMEMO-capable clients or OTR for sensitive comms in the meantime)
* Domain name / SSL certificate setup (documented separately in Phase 1 Technicals)

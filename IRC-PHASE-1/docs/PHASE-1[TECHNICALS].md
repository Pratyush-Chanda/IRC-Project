# Phase 1 — Technical Reference

> Code, configuration, packaging, and repository structure for the IRC network foundation.

---

## Repo Structure

```
IRC-Project/
├── README.md
├── TECHNICALS.md
│
├── IRC-PHASE-1[CONCEPT]/
│   ├── server/
│   │   ├── ngircd.conf.example       # IRC daemon config template
│   │   ├── install.sh                # One-command server setup (Pi/Debian)
│   │   └── start.sh                  # Start/restart the IRC daemon
│   │
│   ├── relay/
│   │   ├── termux-setup.sh           # Android relay setup via Termux
│   │   ├── relay.py                  # Lightweight Python relay node
│   │   └── relay-ngircd.conf.example # ngircd config for relay mode
│   │
│   ├── webirc/
│   │   ├── docker-compose.yml        # The Lounge via Docker (VPS/Pi 4)
│   │   ├── lounge-config.js.example  # The Lounge config template
│   │   └── install-lounge.sh         # Manual install (no Docker)
│   │
│   └── docs/
|	├── PHASE-1.md
|	└── PHASE-1-TECHNICALS.md
│
├── phase-2/                          # Matrix (future)
│   └── .gitkeep
│
└── mesh/                             # Meshtastic / LoRa (future)
    └── .gitkeep
```

---

## IRC Daemon — `ngircd`

We use **ngircd** as the IRC daemon. It is written in C, has minimal dependencies, runs on Raspberry Pi Zero W, and is available via `apt`. A relay node running ngircd uses less than 5MB of RAM.

### Install (Raspberry Pi / Debian / Ubuntu)

```bash
# install.sh
#!/bin/bash
set -e

echo "[CJP] Installing ngircd..."
sudo apt update && sudo apt install -y ngircd

echo "[CJP] Copying config..."
sudo cp ngircd.conf.example /etc/ngircd/ngircd.conf

echo "[CJP] Starting service..."
sudo systemctl enable ngircd
sudo systemctl start ngircd

echo "[CJP] Done. IRC server running on port 6667."
```

### Primary Node Config (`ngircd.conf.example`)

```ini
[Global]
    Name = cjp.irc.node                 ; Server name — change per node
    Info = CJP Decentralised Network
    AdminInfo1 = CJP Tech Team
    AdminEMail = admin@example.com
    Ports = 6667, 6697                  ; 6667 plain, 6697 TLS
    Listen = 0.0.0.0
    MotdFile = /etc/ngircd/motd.txt
    Password =                          ; Leave blank for open, or set a network password

[Options]
    DNS = no                            ; Disable reverse DNS for speed
    Ident = no

[SSL]
    CertFile = /etc/ssl/cjp/server.crt  ; Add SSL cert here (Let's Encrypt recommended)
    KeyFile  = /etc/ssl/cjp/server.key
    Ports = 6697

[Channel]
    Name = #general
    Topic = CJP General — Welcome
    Modes = nt                          ; No external messages, topic protected

[Channel]
    Name = #announcements
    Topic = Official CJP Announcements
    Modes = ntm                         ; Moderated — only ops can speak

[Channel]
    Name = #tech
    Topic = Network Maintenance and Volunteer Coordination

[Channel]
    Name = #secure
    Topic = OpSec Discussion
    Modes = nti                         ; Invite only
```

### Relay Node Config (`relay-ngircd.conf.example`)

A relay node links to the Primary Node as a peer server. This is what volunteers run on phones or Pi Zeros.

```ini
[Global]
    Name = relay-1.cjp.irc              ; Unique name per relay — change this
    Ports = 6667

[Server]
    Name = cjp.irc.node                 ; Must match Primary Node's [Global] Name
    Host = <PRIMARY_NODE_IP>            ; IP or domain of primary node
    Port = 6667
    MyPassword = relay_secret           ; Must match what primary node expects
    PeerPassword = relay_secret
    Passive = no
```

---

## Python Relay Node (`relay.py`)

For volunteers who can't run ngircd, this lightweight Python script acts as a TCP relay — it accepts connections locally and forwards them upstream to the primary IRC node. No dependencies beyond Python 3 (which Termux ships with).

```python
#!/usr/bin/env python3
"""
CJP IRC Relay Node
Minimal Python relay — connects to upstream IRC server and relays local clients.
Run via: python3 relay.py <upstream_ip>
"""

import socket
import threading
import sys
import logging

logging.basicConfig(
    level=logging.INFO,
    format='[%(asctime)s] %(levelname)s %(message)s'
)
logger = logging.getLogger('cjp-relay')

UPSTREAM_HOST = sys.argv[1] if len(sys.argv) > 1 else '127.0.0.1'
UPSTREAM_PORT = int(sys.argv[2]) if len(sys.argv) > 2 else 6667
LOCAL_PORT    = int(sys.argv[3]) if len(sys.argv) > 3 else 6668


def pipe(src: socket.socket, dst: socket.socket):
    """Forward bytes from src to dst until connection closes."""
    try:
        while True:
            data = src.recv(4096)
            if not data:
                break
            dst.sendall(data)
    except (ConnectionResetError, BrokenPipeError, OSError):
        pass
    finally:
        src.close()
        dst.close()


def handle_client(client_sock: socket.socket, addr):
    logger.info(f'Client connected: {addr}')
    try:
        upstream_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        upstream_sock.connect((UPSTREAM_HOST, UPSTREAM_PORT))
    except ConnectionRefusedError:
        logger.error(f'Cannot reach upstream {UPSTREAM_HOST}:{UPSTREAM_PORT}')
        client_sock.close()
        return

    # Two threads — one for each direction
    t1 = threading.Thread(target=pipe, args=(client_sock, upstream_sock), daemon=True)
    t2 = threading.Thread(target=pipe, args=(upstream_sock, client_sock), daemon=True)
    t1.start()
    t2.start()


def main():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('0.0.0.0', LOCAL_PORT))
    server.listen(20)
    logger.info(f'CJP Relay listening on :{LOCAL_PORT} → {UPSTREAM_HOST}:{UPSTREAM_PORT}')

    try:
        while True:
            client_sock, addr = server.accept()
            t = threading.Thread(
                target=handle_client,
                args=(client_sock, addr),
                daemon=True
            )
            t.start()
    except KeyboardInterrupt:
        logger.info('Relay shutting down.')
        server.close()


if __name__ == '__main__':
    main()
```

**Usage:**

```bash
python3 relay.py 192.168.1.100        # upstream IP, local port defaults to 6668
python3 relay.py 192.168.1.100 6667 6668
```

---

## Android Setup via Termux (`termux-setup.sh`)

```bash
#!/data/data/com.termux/files/usr/bin/bash
# Run this inside Termux on Android
# Install Termux from F-Droid, not Play Store

echo "[CJP] Updating packages..."
pkg update && pkg upgrade -y

echo "[CJP] Installing dependencies..."
pkg install -y ngircd python

echo "[CJP] Fetching relay config..."
# Replace with actual raw GitHub URL once repo is live
curl -o ~/ngircd.conf https://raw.githubusercontent.com/hsxtheemperor/IRC-Project/main/phase-1/relay/relay-ngircd.conf.example

echo "[CJP] Edit ~/ngircd.conf to set your relay name and primary node IP."
echo "[CJP] Then run: ngircd -f ~/ngircd.conf"
echo ""
echo "[CJP] To keep relay running in background, use:"
echo "  termux-wake-lock"
echo "  nohup ngircd -f ~/ngircd.conf &"
```

---

## WebIRC — The Lounge

### Via Docker (recommended for VPS or Pi 4)

```yaml
# docker-compose.yml
version: '3.8'

services:
  thelounge:
    image: ghcr.io/thelounge/thelounge:latest
    container_name: cjp-webirc
    restart: unless-stopped
    ports:
      - "9000:9000"     # WebIRC accessible at http://<your-ip>:9000
    volumes:
      - ./lounge-data:/var/opt/thelounge
    environment:
      - THELOUNGE_HOME=/var/opt/thelounge
```

```bash
docker compose up -d
# Then create the first user:
docker exec -it cjp-webirc thelounge add <username>
```

### Manual Install (Pi 3 or if Docker isn't available)

```bash
# install-lounge.sh
#!/bin/bash
set -e

echo "[CJP] Installing Node.js..."
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

echo "[CJP] Installing The Lounge..."
sudo npm install -g thelounge

echo "[CJP] Creating first user (follow prompts)..."
thelounge add admin

echo "[CJP] Starting The Lounge..."
thelounge start &

echo "[CJP] WebIRC available at http://localhost:9000"
```

### The Lounge Config (`lounge-config.js.example`)

```javascript
// lounge-config.js
module.exports = {
  public: false,          // Set true for open access (no login required)
  port: 9000,
  bind: "0.0.0.0",
  reverseProxy: true,     // Set true if behind nginx

  defaults: {
    name: "CJP Network",
    host: "127.0.0.1",   // Points at local ngircd
    port: 6667,
    tls: false,
    rejectUnauthorized: false,
    nick: "cjp_%",       // % replaced with random chars
    username: "cjp",
    realname: "CJP Member",
    join: "#general,#announcements",
  },
};
```

---

## Quick Start — Minimum Viable Setup

If you just want to get something running fast:

```bash
# 1. On a Pi or Debian VPS:
sudo apt install -y ngircd
sudo nano /etc/ngircd/ngircd.conf    # Set [Global] Name
sudo systemctl start ngircd

# 2. Install The Lounge:
sudo npm install -g thelounge
thelounge add admin
thelounge start

# 3. Share the URL with community members:
#    http://<your-ip>:9000
```

That's a working IRC network with a browser frontend in under 10 minutes.

---

## Security Notes

* Enable TLS (port 6697) before making the network public — use Let's Encrypt (`certbot`)
* `#secure` channel should require registration: set channel mode `+i` (invite only) and use NickServ if needed
* The Python relay (`relay.py`) does **not** encrypt traffic — use TLS on the upstream connection for sensitive relays
* For whistleblowing use cases, advise users to connect over a VPN or Tor before joining

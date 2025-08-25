# RustDesk Docker Stack

A production‑ready Docker Compose stack to self‑host the **RustDesk** server components — **HBBS** (ID/Signal server) and **HBBR** (Relay server).

RustDesk is an open‑source, cross‑platform remote desktop solution and a private alternative to TeamViewer/AnyDesk. In a self‑hosted setup, clients register and discover peers via **HBBS** and fall back to **HBBR** for traffic relaying when direct P2P is not possible. Communication is end‑to‑end encrypted with an **Ed25519** keypair. The **public key** is shared with clients to pin your server identity.

> This repository provides a simple, reproducible way to run HBBS/HBBR with Docker. First boot generates the server keypair in your mounted data directory; you’ll then distribute the **public key** to clients.

---

## Features

* 🧰 One‑command bring‑up with Docker Compose
* 🔐 Automatic server keypair generation and persistence
* 🌐 Works on LAN or public Internet (NAT traversal supported)
* 📈 Optional container logging snippet (e.g., Splunk)
* 🚦 Clear firewall/port guidance

---

## Stack Overview

* **hbbs** – ID/Signal server (registration, heartbeat, NAT test, TCP hole punching)
* **hbbr** – Relay server (forwards traffic when direct connection fails)

> Optional: Reverse proxy (e.g., Nginx) if you plan to expose web clients; otherwise direct ports are fine.

---

## Prerequisites

* Docker ≥ 20.x and Docker Compose v2
* A hostname (e.g., `rustdesk.example.com`) pointing to your server
* Ability to open/firewall the required ports (see below)

---

## Quick Start

### 1) Clone and enter

```bash
git clone https://github.com/amiirsadeghi/rustdesk-docker-stack.git
cd rustdesk-docker-stack
```

### 2) Adjust configuration (if needed)

* Confirm the data volume in `docker-compose.yml` (by default `./data:/root`).
* (Optional) Set environment variables (e.g., `ALWAYS_USE_RELAY=Y`).
* Ensure firewall/NAT rules for the ports below.

### 3) Bring the stack up

```bash
docker compose up -d
```

This starts **hbbs** and **hbbr**. On **first run**, RustDesk generates an Ed25519 keypair in your mounted **data** directory.

### 4) Locate the server keys

The keypair is persisted to your host under the mapped data path. With the default mapping:

```
./data/id_ed25519       # private key (keep safe!)
./data/id_ed25519.pub   # public key (share with clients)
```

> If you changed the volume mapping, check that folder for `id_ed25519` and `id_ed25519.pub`.

### 5) Grab the public key

```bash
# Print the public key so you can copy/paste it into clients
cat ./data/id_ed25519.pub
```

> Do **not** share `id_ed25519` (private key). Keep it backed up; this identity is what clients pin to.

### 6) Point clients at your server

On each RustDesk client:

1. Open **Settings → Network** and **Unlock Network Settings**.
2. Fill **ID Server** with your host (default port is `21116`):

   * Example: `rustdesk.example.com` or `rustdesk.example.com:21116`
3. (Recommended) Fill **Relay Server** (default port `21117`):

   * Example: `rustdesk.example.com` or `rustdesk.example.com:21117`
4. Paste the content of your **public key** (`id_ed25519.pub`) into the **Key** field.
5. Save/Apply. (If running as a service, you may need to restart the RustDesk service.)

You’re done. Clients will now use your self‑hosted infrastructure.

---

## Ports & Firewall

Open these on your server and forward from your router if hosting publicly:

| Component | Port(s) | Protocol | Purpose                                       |
| --------- | ------- | -------- | --------------------------------------------- |
| **hbbs**  | 21115   | TCP      | NAT type test                                 |
|           | 21116   | TCP/UDP  | ID registration, heartbeat, TCP hole punching |
|           | 21118   | TCP      | Web client support (optional)                 |
| **hbbr**  | 21117   | TCP      | Relay services                                |
|           | 21119   | TCP      | Web client support (optional)                 |

> If you don’t need **web clients**, you can omit `21118`/`21119`. Make sure **UDP 21116** is allowed; it’s required for registration/heartbeats.

---

## Directory Layout

```
rustdesk-docker-stack/
├─ docker-compose.yml        # Main stack
├─ data/                     # Persisted data (keys, etc.)
│  ├─ id_ed25519             # Private key (keep safe; back it up)
│  └─ id_ed25519.pub         # Public key (distribute to clients)
└─ README.md
```

---

## Environment Hints

You can pass environment variables to **hbbs**/**hbbr** in your compose file. Common examples:

```yaml
services:
  hbbs:
    image: rustdesk/rustdesk-server:latest
    environment:
      - ALWAYS_USE_RELAY=Y   # force all traffic through hbbr
    command: hbbs
    volumes:
      - ./data:/root
    network_mode: host
    depends_on: [hbbr]
    restart: unless-stopped

  hbbr:
    image: rustdesk/rustdesk-server:latest
    command: hbbr
    volumes:
      - ./data:/root
    network_mode: host
    restart: unless-stopped
```

> Keep `./data` persistent so your server identity (keys) remains stable across restarts/upgrades.

---

## Optional: Container Logging (e.g., Splunk)

Add a logging block per‑service if you forward logs to a collector:

```yaml
logging:
  driver: splunk
  options:
    splunk-token: "YOUR_TOKEN"
    splunk-url: "https://splunk.example.com:8088"
    splunk-index: "docker"
    splunk-sourcetype: "rustdesk"
```

---

## Operations

### View status & logs

```bash
docker compose ps
docker compose logs -f hbbs
docker compose logs -f hbbr
```

### Backup keys (recommended)

```bash
# Stop containers first if you like, then back up the private key
cp ./data/id_ed25519 ./backups/id_ed25519.$(date +%F)
```

> Moving to a new host? Copy both `id_ed25519` and `id_ed25519.pub` into the new server’s data directory before starting containers to preserve identity.

---

## Troubleshooting

* **Clients connect without asking for a Key**: ensure clients actually set the **Key** from your `id_ed25519.pub`. (Key pinning protects against MITM.)
* **Registration fails**: verify **UDP 21116** is open and forwarded correctly; confirm firewall isn’t blocking it.
* **Relay not used / connection drops**: make sure **TCP 21117** is reachable from clients; consider `ALWAYS_USE_RELAY=Y` for strict routing via relay.
* **Key mismatch**: distribute the latest `id_ed25519.pub` to clients. If you rotated keys, all clients must update the **Key** field.
* **Fresh keys**: delete `./data/id_ed25519*` and restart the stack (clients will need the new public key).

---

## FAQ

**Where do I find the public key?**
In your mapped data directory (default `./data/id_ed25519.pub`). Copy its entire content into the **Key** field on clients.

**Will my key change on updates?**
No — as long as the **data volume is persistent**. If you delete the `id_ed25519` files, a new keypair is generated.

**Do I need Nginx/SSL?**
Not for native clients. If you plan to support **web clients**, also open 21118/21119 and consider a reverse proxy/TLS in front.

---

## Contributing

Issues and PRs are welcome. Please open an issue for bugs or feature requests.

## License

MIT

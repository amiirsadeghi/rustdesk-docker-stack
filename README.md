# RustDesk Docker Stack

A production‑ready Docker Compose stack to self‑host the **RustDesk** server components — **HBBS** (ID/Signal server) and **HBBR** (Relay server).

RustDesk is an open‑source, cross‑platform remote desktop solution and a private alternative to TeamViewer/AnyDesk. In a self‑hosted setup, clients register and discover peers via **HBBS** and fall back to **HBBR** for traffic relaying when direct P2P is not possible. Communication is end‑to‑end encrypted with an **Ed25519** keypair. The **public key** is shared with clients to pin your server identity.

> This repository provides a simple, reproducible way to run HBBS/HBBR with Docker. First boot generates the server keypair in your mounted data directory; you’ll then distribute the **public key** to clients.

---

## Features

* 🧰 One‑command bring‑up with Docker Compose
* 🔐 Automatic server keypair generation and persistence
* 🌐 Works on LAN or public Internet (NAT traversal supported)
* 📄 Built‑in Nginx landing page for end‑user instructions
* 📈 Optional container logging snippet (e.g., Splunk)

---

## Stack Overview

* **hbbs** – ID/Signal server (registration, heartbeat, NAT test, TCP hole punching)
* **hbbr** – Relay server (forwards traffic when direct connection fails)
* **nginx** – Serves a landing page with configuration instructions (edit `nginx/html/index.html`)

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

### 2) Adjust configuration

* Confirm the data volume in `docker-compose.yml` (by default `./data:/root`).
* Edit **`nginx/html/index.html`** to replace placeholders (`your-company`, `your-domain.url`) with your real company name, domain, and paste your **public key**.
* (Optional) Add your company logo to `nginx/html/` and reference it in `index.html`.

### 3) Bring the stack up

```bash
docker compose up -d
```

This starts **hbbs**, **hbbr**, and **nginx**. On **first run**, RustDesk generates an Ed25519 keypair in your mounted **data** directory.

### 4) Locate the server keys

```
./data/id_ed25519       # private key (keep safe!)
./data/id_ed25519.pub   # public key (share with clients)
```

### 5) Grab the public key

```bash
cat ./data/id_ed25519.pub
```

Copy this content and paste it into:

* Your landing page (`index.html`)
* Each client’s **Key** field

### 6) Point clients at your server

On each RustDesk client:

1. Open **Settings → Network** and **Unlock Network Settings**.
2. Fill **ID Server** with your host (default port is `21116`):

   * Example: `rustdesk.example.com` or `rustdesk.example.com:21116`
3. (Recommended) Fill **Relay Server** (default port `21117`):

   * Example: `rustdesk.example.com` or `rustdesk.example.com:21117`
4. Paste the content of your **public key** into the **Key** field.
5. Save/Apply.

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

---

## Nginx Landing Page

This stack includes an **Nginx container** serving a landing page on your domain (e.g., `https://your-domain.url`).

* The page is at: `./nginx/html/index.html`
* Edit the file to set:

  * **Company name**
  * **Domain name**
  * **Public key** (`id_ed25519.pub`)
* Employees can visit this page, copy the public key, and follow the instructions.

Example `index.html` snippet:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Your Company Remote Access</title>
</head>
<body>
  <h1>Welcome to YourCompany RustDesk</h1>
  <p>ID Server: rustdesk.your-domain.url:21116</p>
  <p>Relay Server: rustdesk.your-domain.url:21117</p>
  <p>Public Key:</p>
  <pre>PASTE_YOUR_PUBLIC_KEY_HERE</pre>
  <p>Follow these steps in RustDesk client settings to connect.</p>
</body>
</html>
```

---

## Directory Layout

```
rustdesk-docker-stack/
├─ docker-compose.yml        # Main stack
├─ data/                     # Persisted data (keys)
│  ├─ id_ed25519             # Private key (keep safe)
│  └─ id_ed25519.pub         # Public key (distribute)
├─ nginx/
│  ├─ html/index.html        # Landing page (edit placeholders)
│  ├─ ssl/                   # Place your SSL certs here
│  └─ nginx.conf             # Reverse proxy config
└─ README.md
```

---

## Troubleshooting

* **Clients connect without asking for a Key**: ensure clients set the **Key** from your `id_ed25519.pub`.
* **Registration fails**: verify **UDP 21116** is open.
* **Relay not used / connection drops**: ensure **TCP 21117** is reachable.
* **Key mismatch**: update clients with the latest `id_ed25519.pub`.

---

## Contributing

Issues and PRs are welcome. Please open an issue for bugs or feature requests.

## License

MIT

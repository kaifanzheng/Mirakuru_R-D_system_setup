# 🌍 Cross‑Border Secure File Sharing (China ↔ Canada)

This document is a **personal technical log + reproducible guide** describing how I set up a **secure, reliable, no‑port‑forwarding file sharing system** between my Ubuntu machine in Canada and a friend in China.

The final system:

* ✅ Works behind NAT / firewalls
* ✅ No port forwarding
* ✅ China‑friendly
* ✅ Survives reboot
* ✅ Uses proper Linux service management (systemd)

---

## 🎯 Goal

I wanted my friend (in China) to:

* Access my files remotely
* Upload / download binaries (embedded dev artifacts)
* Avoid Git for large binaries
* Avoid unstable VPNs
* Avoid exposing my home IP

And I wanted:

* Auto‑start on reboot
* Clean, debuggable services
* Minimal attack surface

---

## 🧠 High‑Level Architecture

```
Friend (China)
   │
   │ HTTPS / SSH (Cloudflare Zero Trust)
   ▼
Cloudflare Tunnel (outbound only)
   │
   ▼
Ubuntu Host (Canada)
   ├─ cloudflared (systemd service)
   └─ filebrowser (systemd service, localhost only)
```

Key idea:

> **My server initiates all connections outbound.**

---

## 🧩 Components Used

| Component         | Purpose                                       |
| ----------------- | --------------------------------------------- |
| Cloudflare Tunnel | Secure inbound access without port forwarding |
| File Browser      | Simple web‑based file manager                 |
| systemd           | Service management + auto‑start               |
| SSH               | Secure transport + debugging                  |

---

## 🚦 Step‑by‑Step Journey

### 1️⃣ Domain & DNS (Cloudflare)

* Added domain to Cloudflare
* Switched nameservers from GoDaddy → Cloudflare
* Verified with:

```bash
dig NS mydomain.ca
```

Once Cloudflare nameservers appeared, DNS control was confirmed.

---

### 2️⃣ Cloudflare Tunnel Setup

Authenticated locally:

```bash
cloudflared tunnel login
```

Created tunnel:

```bash
cloudflared tunnel create miarkuru-tunnel
```

Configured tunnel ingress:

```yaml
ingress:
  - hostname: files.mydomain.ca
    service: http://127.0.0.1:8081
  - service: http_status:404
```

Mapped DNS:

```bash
cloudflared tunnel route dns miarkuru-tunnel files.mydomain.ca
```

---

### 3️⃣ Running cloudflared as a systemd service

Problem encountered:

* cloudflared failed on reboot
* error: missing `cert.pem`

Cause:

* systemd runs as root
* Cloudflare cert existed only in user home

Fix:

* Run cloudflared **as my user**

```ini
[Service]
User=mirakuru-canada
Group=mirakuru-canada
ExecStart=/usr/local/bin/cloudflared tunnel run miarkuru-tunnel
Restart=always
```

After this, cloudflared became stable and reboot‑safe.

---

### 4️⃣ File Browser Setup

Initial symptom:

* Login worked once
* After reboot → "Wrong credentials"

Root cause:

* File Browser was creating a **new database each start**

---

### 5️⃣ Fixing File Browser Persistence

Created persistent state directory:

```bash
sudo mkdir -p /var/lib/filebrowser
sudo chown mirakuru-canada:mirakuru-canada /var/lib/filebrowser
```

Initialized database:

```bash
filebrowser config init --database /var/lib/filebrowser/filebrowser.db
```

Created admin user:

```bash
filebrowser users add admin <password> --perm.admin \
  --database /var/lib/filebrowser/filebrowser.db
```

Pinned DB + config in systemd:

```ini
ExecStart=/usr/local/bin/filebrowser \
  --root /srv/shared \
  --address 127.0.0.1 \
  --port 8081 \
  --database /var/lib/filebrowser/filebrowser.db \
  --config /var/lib/filebrowser/settings.json
```

Result:

* Credentials persist across reboots
* State is predictable and inspectable

---

### 6️⃣ systemd Service Management

Enabled auto‑start:

```bash
sudo systemctl enable filebrowser
sudo systemctl enable cloudflared-tunnel
```

Verified:

```bash
systemctl is-enabled filebrowser
systemctl is-enabled cloudflared-tunnel
```

Checked runtime state:

```bash
systemctl status filebrowser
systemctl status cloudflared-tunnel
```

---

## 🧪 Debugging Tools Learned

| Tool                      | Purpose            |
| ------------------------- | ------------------ |
| systemctl status          | Service health     |
| journalctl -u             | Logs               |
| systemctl list-units      | Running services   |
| systemctl list-unit-files | Installed services |
| pgrep / ps                | Map to processes   |

---

## 🔐 Security Decisions

* File Browser bound to `127.0.0.1` only
* No inbound ports opened
* Cloudflare handles auth + TLS
* Services run as non‑root user
* Clear separation of data vs binaries

---

## ✅ Final Result

✔ Reliable access from China
✔ No VPN required
✔ No port forwarding
✔ Survives reboot
✔ Easy to debug
✔ Clean Linux‑native setup

---

## 🧭 Key Lessons

* systemd context ≠ shell context
* Always pin databases explicitly
* Outbound tunnels > inbound exposure
* Logs tell the truth, not dashboards
* DNS is always the real gatekeeper

---

**This setup now feels boring — which means it’s correct.**

---
machine_name: homeserver
hardware: Lenovo G50-30 Laptop
os: Debian 12 (Bookworm) minimal headless # Verify with: cat /etc/os-release
cpu: Intel Bay Trail Celeron/Pentium SoC (x86_64, 7.5W TDP)
ram: 4GB DDR3L (Non-upgradable)
storage: 291GB SATA HDD
idle_power: ~4W
idle_temp: 41°C - 44°C
network:
  lan_ip_static: 192.168.1.11
  gateway: 192.168.1.1
  mdns_hostname: homeserver.local
  tailscale_ip: 100.124.195.110
  tailscale_domain: homeserver.tail274b3b.ts.net
  host_user: user
  isp: Univision (Router admin locked; no port forwarding)
active_services:
  recasaos: { port: 80, role: "Web OS Dashboard" }
  navidrome: { port: 4533, role: "Subsonic Music Streaming Server" }
  adguard_home: { port_web: 3001, port_dns: 53, role: "DNS Sinkhole & Ad Blocker" }
  vaultwarden: { port: 10380, url_https: "https://homeserver.tail274b3b.ts.net", role: "Bitwarden Password Vault" }
  opengym: { port_local: "127.0.0.1:3840", url_https: "https://homeserver.tail274b3b.ts.net:8443", role: "Self-Hosted Gym & Workout Tracker (Passkey/PWA)" }
  upsnap: { port: 8090, network: "host", role: "Wake-on-LAN & Device Status Dashboard" }
  slskd: { port: 5030, status: "on-demand", role: "Soulseek P2P Downloader (Stopped when idle)" }
  samba: { port: 445, role: "Native SMB Network Drive" }
key_constraints:
  - "Do NOT suggest ideapad_laptop kernel module (Causes ACPI rfkill bug that hard-blocks Wi-Fi)"
  - "Do NOT suggest video transcoding or heavy VMs (Weak 7.5W CPU)"
  - "Do NOT run containrrr/watchtower (Archived; crash-loops on Docker 29+ due to API v1.25)"
  - "Vaultwarden strictly requires HTTPS (Tailscale Serve port 443 -> 127.0.0.1:10380)"
  - "openGym strictly requires HTTPS + matching RP_ID/ORIGIN for WebAuthn passkeys (Tailscale Serve port 8443 -> 127.0.0.1:3840; never change RP_ID after creating profiles)"
  - "Storage directories: /DATA/Media (Music, Movies), /DATA/AppData (Container data), and ~/openGym (Git + Compose stack)"
  - "AdGuard Home rate limit MUST be 0 (disabled) to avoid whole-subnet DNS blackouts"
  - "AdGuard Home upstream mode MUST be parallel with IPv4 upstreams (1.1.1.1, 8.8.8.8); avoid DoH/IPv6 timeouts on Univision ISP"
  - "Tailscale client devices MUST keep Exit Node set to None for split-tunnel operation"
  - "Windows SMB requires connecting as 192.168.1.11\\user when AllowInsecureGuestAuth=0"
---

# 🛸 Lenovo G50-30 Home Server (`homeserver`)

> **Note for AI Assistants:** Refer to the YAML frontmatter above and the **[AI Assistant Context & Rules](#-ai-assistant-context--rules)** section below for exact hardware limits, network topology, and known quirks before suggesting changes.

---

## 🤖 AI Assistant Context & Rules

When assisting with this server in future sessions, follow these operational directives:

1. **Hardware Constraints:**
   * Soldered Bay Trail CPU + 4GB RAM + mechanical HDD.
   * **Never recommend:** 4K video transcoding, Proxmox/heavy hypervisors, resource-heavy Java/PHP stacks (e.g. Nextcloud), or local LLMs.
   * **Prefer:** Lightweight Go, Rust, Node/C binaries and containers with low idle CPU/RAM footprints.
2. **Networking & Router Constraints:**
   * ISP router (Univision) is completely locked down. No port-forwarding or router DHCP changes are possible.
   * Remote access and secure HTTPS are handled entirely through **Tailscale** (`100.124.195.110` / `homeserver.tail274b3b.ts.net`).
   * **Tailscale Serve Port Allocation:** Port `443` is reserved for **Vaultwarden** (`127.0.0.1:10380`). Additional HTTPS services use custom Tailscale Serve ports (e.g., `:8443` for **openGym** -> `127.0.0.1:3840`) so existing services are not overwritten.
   * Local access is pinned to static IP `192.168.1.11` via NetworkManager (`wlp2s0`).
3. **Critical Driver Blacklist:**
   * **`ideapad_laptop` MUST remain blacklisted.** Loading this module on the Lenovo G50 triggers an ACPI bug that hard-blocks the Wi-Fi card.
4. **Data Paths & Media Architecture:**
   * Persistent container data (CasaOS/Docker): `/DATA/AppData/<app_name>`
   * Git-managed Compose stacks: `~/openGym` (stores DB/VAPID/audit data in `~/openGym/data`)
   * User media: `/DATA/Media/Music` (Casual streaming library for Navidrome).
   * **DJ Techno Library:** Must remain on a dedicated external/local DJ drive (not on the server's 291GB HDD) to eliminate Wi-Fi latency and save server capacity.
5. **openGym WebAuthn / Passkey Rules:**
   * `RP_ID` (`homeserver.tail274b3b.ts.net`) and `ORIGIN` (`https://homeserver.tail274b3b.ts.net:8443`) MUST match the exact Tailscale Serve URL. Changing `RP_ID` later invalidates existing passkeys.
   * Bound locally to `127.0.0.1:3840` (`WEB_PORT`) so it is reachable only via Tailscale Serve, avoiding LAN/dev port `8080` conflicts.
6. **AdGuard Home Operational Directives:**
   * **Rate limiting MUST remain disabled (`ratelimit: 0`).** Setting a default limit (e.g. 20) chokes the entire `/24` subnet when a single device loads heavy web pages.
   * **Upstream Mode:** Must be set to `parallel` using reliable IPv4 resolvers (`1.1.1.1`, `8.8.8.8`). Avoid DNS-over-HTTPS (DoH) and IPv6 bootstrap servers that fail or time out on Univision ISP.
7. **Samba & Windows SMB Directives:**
   * In `/etc/samba/smb.conf`, always maintain a trailing newline before share headers (especially following `include = /etc/samba/smb.casa.conf`) to prevent syntax corruptions.
   * Windows 10/11 clients with `AllowInsecureGuestAuth = 0` must authenticate with `192.168.1.11\user` rather than attempting anonymous/guest connections.
8. **Tailscale Client Split-Tunnel Directives:**
   * The homeserver advertises an exit node (`--advertise-exit-node`).
   * Client devices (iPhones, Macs, Windows PCs) should keep **Exit Node set to "None"** unless intentionally routing full web traffic through the laptop. This prevents cellular internet drops and relay timeouts.
9. **Container Updates:**
   * Do **not** use `containrrr/watchtower` (archived and incompatible with Docker 29+). Prefer controlled `docker compose pull && docker compose up -d` updates or modern alternatives (`getwud/wud` / `nickfedor/watchtower`).

---

## 📋 System Profile

| Specification | Value | Notes |
| :--- | :--- | :--- |
| **Model** | Lenovo G50-30 | Entry-level budget laptop |
| **OS** | Debian 12 (Bookworm) | Headless (No desktop environment) |
| **CPU** | Intel Bay Trail (Celeron/Pentium) | 7.5W TDP, passive/ultra-low power |
| **RAM** | 4GB DDR3L | ~2.0GB–2.3GB available under normal load |
| **Storage** | ~291GB SATA HDD | Mounted at `/` (System & `/DATA`) |
| **Power Draw** | ~4 Watts (Idle) | Screen blanked, runs on AC wall adapter |
| **Thermals** | 41°C – 44°C | Stable under continuous operation |
| **LAN IP** | `192.168.1.11` | Static (`/24`, Gateway: `192.168.1.1`) |
| **Local mDNS** | `homeserver.local` | Accessible via Avahi |
| **Tailscale IP** | `100.124.195.110` | Global mesh network access |
| **Tailscale Host** | `homeserver.tail274b3b.ts.net` | Valid Let's Encrypt HTTPS via `tailscale serve` |
| **Host User** | `user` | Has passwordless Docker group access |

---

## 🚀 Services

| Service | Purpose | Local Access URL | Tailscale (Remote) URL | Host Port | Container Port | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **ReCasaOS** | Web OS / Dashboard | `http://192.168.1.11` | `http://100.124.195.110` | `80` | Native | Always-on |
| **Navidrome** | Music Streamer | `http://192.168.1.11:4533` | `http://100.124.195.110:4533` | `4533` | `4533` | Always-on |
| **AdGuard Home** | DNS & Ad Blocker | `http://192.168.1.11:3001` | `http://100.124.195.110:3001` | `3001` (Web) / `53` (DNS) | `3000` / `53` | Always-on |
| **Vaultwarden** | Bitwarden Password Vault | `http://192.168.1.11:10380` | `https://homeserver.tail274b3b.ts.net` *(HTTPS)* | `10380` | `80` | Always-on (Tailscale Serve `:443`) |
| **openGym** | Gym & Workout Tracker | `—` *(localhost only)* | `https://homeserver.tail274b3b.ts.net:8443` *(HTTPS)* | `127.0.0.1:3840` | `80` | Pause/resume via Compose (Tailscale Serve `:8443`) |
| **UpSnap** | Wake-on-LAN Dashboard | `http://192.168.1.11:8090` | `http://100.124.195.110:8090` | `8090` | `host` | Always-on |
| **slskd** | Soulseek P2P Downloader | `http://192.168.1.11:5030` | `http://100.124.195.110:5030` | `5030` | `5030` | On-demand (stopped when idle) |
| **Samba (SMB)** | Native Network Drive | `smb://192.168.1.11` | `smb://100.124.195.110` | `445` | Native | Always-on |

---

## 📂 Storage & Directory Architecture

```text
/DATA/
├── AppData/
│   ├── navidrome/           # Navidrome database & cache
│   ├── adguardhome/         # AdGuard filter lists & query logs
│   ├── vaultwarden/data/    # Encrypted password vault database
│   ├── upsnap/              # UpSnap PocketBase data
│   └── slskd/               # Soulseek config & transfer cache
└── Media/
    ├── Music/               # Shared casual music (Fed by slskd, streamed by Navidrome)
    ├── Movies/              # Reserved for video media
    └── Documents/           # General network drive storage

~/openGym/                   # openGym git repo, docker-compose.yml, .env
└── data/                    # openGym db.json, audit.log, vapid.json, secret
```

---

## 🎵 Music Architecture & Format Guide

| Collection Tier | Recommended Format | Location | Primary Client |
| :--- | :--- | :--- | :--- |
| **Casual Listening** | **AAC 256kbps (`.m4a`)** or **MP3 320kbps** | Server: `/DATA/Media/Music` | iPhone (Amperfy), Web Player |
| **DJ Techno Tracks** | **16-bit / 44.1kHz FLAC** or **AIFF** | Dedicated External / Local DJ Drive | Rekordbox, Traktor, Serato |

### Client Streaming Best Practices:
* **iPhone & Bluetooth:** iOS transmits AAC natively over Bluetooth to AirPods and car systems. Using **AAC 256kbps (`.m4a`)** eliminates transcoding generation loss.
* **Amperfy App (iOS):**
  * **Wi-Fi:** Set Streaming Quality to **Original / Direct Play** (0% server CPU).
  * **Cellular Data:** Transcodes to AAC 256kbps on the fly to save data (~6MB per track).
  * **Offline Cache:** Set storage cache to 5GB–10GB to ensure repeat plays use 0 bytes of mobile data.

---

## 📱 Client Setup Guide

### 🎵 Music Streaming (Amperfy / Substreamer)
* **iOS Apps:** [Amperfy Music](https://apps.apple.com/app/amperfy-music/id1552309193) or [Substreamer](https://apps.apple.com/app/substreamer/id1012991665)
* **Android Apps:** Tempo (F-Droid) or Symfonium
* **Server Address (Home):** `http://192.168.1.11:4533`
* **Server Address (Remote):** `http://100.124.195.110:4533`
* **Protocol:** Subsonic API

### 🔐 Password Manager (Bitwarden)
* **Client:** Official Bitwarden mobile app (iOS/Android) or Chrome/Safari extensions.
* **Server URL:** `https://homeserver.tail274b3b.ts.net`
* *(Note: Requires Tailscale active on the client device for trusted HTTPS certificates).*

### 🏋️ Workout Tracker (openGym)
* **Access:** Safari PWA (iPhone Home Screen) or openGym mobile app.
* **Server URL:** `https://homeserver.tail274b3b.ts.net:8443`
* *(Note: Requires Tailscale active on the client device. Sign-in uses device Passkey bound to `homeserver.tail274b3b.ts.net`.)*

### 📁 Network Storage (Mac Finder & Windows Explorer)
* **macOS:** Finder ➡️ `Cmd + K` ➡️ `smb://192.168.1.11` *(Connect as Guest or `user`)*.
* **Windows Explorer (Map Network Drive):**
  1. Open File Explorer ➡️ **This PC** ➡️ **Map network drive**.
  2. Folder: `\\192.168.1.11\DATA` (or `\\192.168.1.11\Media`).
  3. Check **"Connect using different credentials"**.
  4. Username: `192.168.1.11\user` (or click *More choices ➡️ Use a different account*).
  5. Password: *(Samba password configured on host)*.
  * *Quick CLI Mount:*
    ```cmd
    net use Z: \\192.168.1.11\DATA /user:192.168.1.11\user
    ```

---

## 🛠️ Remote Access & Useful Commands

### SSH Access (via Tailscale from anywhere)
```bash
ssh user@100.124.195.110
# or via MagicDNS
ssh user@homeserver.tail274b3b.ts.net
# or local home Wi-Fi
ssh user@192.168.1.11
```

### Live System Dashboard
A custom dashboard script is installed at `/usr/local/bin/stats`:
```bash
stats
```
Outputs temperature, battery level, AC status, storage, available RAM, and live Docker stats.

### Docker App Management
```bash
# Manage Soulseek on demand
docker stop slskd
docker start slskd

# Pause / Resume / Update openGym (keeps ~/openGym/data intact)
cd ~/openGym
docker compose stop          # Pause (stays off across reboots)
docker compose up -d         # Resume
git pull && docker compose pull && docker compose up -d && docker image prune -f

# Verify Tailscale Serve routes
tailscale serve status

# Inspect live container logs
docker logs --tail 30 navidrome
docker logs --tail 30 vaultwarden
docker logs --tail 30 adguard-home
docker logs --tail 30 upsnap
```

---

## ⚙️ Host System Configurations Applied

1. **Lid Sleep Ignored:**
   * `/etc/systemd/logind.conf.d/lid.conf` sets `HandleLidSwitch=ignore` and `HandleLidSwitchExternalPower=ignore`.
2. **Display Sleep (60 Seconds):**
   * `/etc/default/grub` includes `consoleblank=60` in `GRUB_CMDLINE_LINUX_DEFAULT` to power down the screen backlight after 1 minute of inactivity.
3. **Wi-Fi Driver Blacklist:**
   * `/etc/modprobe.d/blacklist-ideapad.conf` contains `blacklist ideapad_laptop` to prevent phantom ACPI rfkill triggers that hard-block the Wi-Fi interface.
4. **Static IP Configuration:**
   * Wi-Fi interface (`wlp2s0`) locked to `192.168.1.11/24` with Gateway `192.168.1.1` and DNS `1.1.1.1 8.8.8.8` via NetworkManager.
5. **Tailscale HTTPS Reverse Proxies (`tailscale serve`):**
   * `https://homeserver.tail274b3b.ts.net` (`443`) ➡️ `http://127.0.0.1:10380` (**Vaultwarden**)
   * `https://homeserver.tail274b3b.ts.net:8443` (`8443`) ➡️ `http://127.0.0.1:3840` (**openGym**)
6. **Samba File Sharing Configuration:**
   * Installed natively via `apt install samba` and exposed on port 445. Reference configuration stored at `samba/smb.conf`. Includes CasaOS shares and dedicated `[Media]` share.
7. **AdGuard Home Tuning:**
   * Rate limiting set to `0` to prevent whole-network throttling.
   * Upstreams configured to parallel IPv4 queries (`1.1.1.1`, `8.8.8.8`) to avoid ISP DoH timeouts.
8. **Wi-Fi Power Management & Sleep Disabled:**
   * `/etc/NetworkManager/conf.d/disable-wifi-powersave.conf` sets `wifi.powersave = 2` (disabled) and `iw dev wlp2s0 set power_save off`.
   * `/etc/modprobe.d/rtl8723be.conf` configures `ips=0 fwlps=0 swlps=0 msi=1` to prevent Realtek Wi-Fi radio from entering periodic DTIM sleep states that cause 5,000+ ms latency spikes.

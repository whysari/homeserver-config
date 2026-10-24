# 🛸 Lenovo G50-30 Home Server (`homeserver`)

> A silent, ultra-efficient (~4W) personal cloud server built from a 2014 Lenovo G50-30 laptop running Debian 12 (Bookworm) headless.

![Status](https://img.shields.io/badge/status-active-success?style=flat-square)
![OS](https://img.shields.io/badge/OS-Debian%2012%20(Bookworm)-red?style=flat-square)
![Power](https://img.shields.io/badge/power-~4W%20idle-brightgreen?style=flat-square)
![Architecture](https://img.shields.io/badge/arch-x86__64-blue?style=flat-square)

---

## 📋 System Profile

| Specification | Value |
| :--- | :--- |
| **Hardware** | Lenovo G50-30 Laptop |
| **CPU** | Intel Bay Trail Celeron/Pentium SoC (7.5W TDP) |
| **RAM** | 4GB DDR3L |
| **Storage** | ~291GB SATA HDD |
| **Idle Power** | ~4 Watts (Display backlight off after 60s) |
| **Operating Temp** | ~41°C – 44°C |
| **Local Static IP** | `192.168.1.11` |
| **Local mDNS** | `homeserver.local` |
| **Tailscale IP** | `100.124.195.110` |
| **Tailscale Domain** | `homeserver.tail274b3b.ts.net` |
| **Host User** | `user` |

---

## 🚀 Running Services

| Service | Purpose | Local URL | Tailscale (Remote) URL | Port |
| :--- | :--- | :--- | :--- | :--- |
| **ReCasaOS** | Web OS & Dashboard | `http://192.168.1.11` | `http://100.124.195.110` | `80` |
| **Navidrome** | Music Streamer (Spotify alternative) | `http://192.168.1.11:4533` | `http://100.124.195.110:4533` | `4533` |
| **AdGuard Home** | DNS & Ad Blocker | `http://192.168.1.11:3001` | `http://100.124.195.110:3001` | `3001` (Web) / `53` (DNS) |
| **Vaultwarden** | Bitwarden Password Vault | `http://192.168.1.11:10380` | `https://homeserver.tail274b3b.ts.net` *(HTTPS)* | `10380` |
| **slskd** | Soulseek Music Downloader | `http://192.168.1.11:5030` | `http://100.124.195.110:5030` | `5030` |
| **Samba (SMB)** | Native Network Drive | `smb://192.168.1.11` | `smb://100.124.195.110` | `445` |

---

## 📱 Client Setup Guide

### 🎵 Music Streaming (Amperfy / Substreamer)
* **iOS:** [Amperfy Music](https://apps.apple.com/app/amperfy-music/id1552309193) or [Substreamer](https://apps.apple.com/app/substreamer/id1012991665)
* **Android:** Tempo or Symfonium
* **Server Address (Home):** `http://192.168.1.11:4533`
* **Server Address (Remote via Tailscale):** `http://100.124.195.110:4533`

### 🔐 Password Manager (Bitwarden)
* **Client:** Official free Bitwarden iOS/Android app or Chrome/Safari extension.
* On the login screen, tap the **Gear / Settings icon**:
  * **Server URL:** `https://homeserver.tail274b3b.ts.net`
  * *(Requires Tailscale enabled on the client device for trusted HTTPS).*

### 📁 Network Storage (Mac Finder & Windows Explorer)
* **macOS:** Finder ➡️ `Cmd + K` ➡️ `smb://192.168.1.11`
* **Windows:** File Explorer ➡️ `Win + R` ➡️ `\\192.168.1.11`

---

## 🛠️ Remote Access & Useful Commands

### SSH Access (via Tailscale from anywhere)
```bash
ssh user@100.124.195.110
# or
ssh user@homeserver.tail274b3b.ts.net
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

# Inspect live container logs
docker logs --tail 30 navidrome
docker logs --tail 30 vaultwarden
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
5. **Tailscale HTTPS Reverse Proxy:**
   * Built-in Tailscale Serve proxies `https://homeserver.tail274b3b.ts.net` to local port `10380` with automatic Let's Encrypt certificates.

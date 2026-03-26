# VPN + Proxy in Russia

This guide explains how to set up a self-hosted VPN using the **VLESS + XTLS-Reality** protocol — the most censorship-resistant tunneling protocol available today. Unlike WireGuard or OpenVPN, it disguises your traffic as ordinary HTTPS and cannot be reliably detected or blocked by deep packet inspection (DPI).

The guide covers the full picture: from renting and configuring a VPS abroad, to connecting your personal devices or routing an entire Linux server through the VPN. It also describes a two-server setup suited for users in countries with internet restrictions (e.g., Russia), where a local proxy server forwards traffic to a VPN server abroad.

**Who this guide is for:**
- You want to bypass internet censorship or geo-restrictions
- You want full control over your VPN — no third-party services, no shared infrastructure
- You need a personal device (phone, PC) or an entire server to go through the VPN
- You are in Russia or a similarly restricted region and need a reliable, hard-to-block solution

**What you will need:**
- A VPS in an unrestricted country (Europe, USA, etc.) for the VPN server — see [VPS Server Requirements](#vps-server-requirements) for provider recommendations
- Optionally, a second VPS close to your location (e.g., Moscow) if you want a local proxy entry point

The guide consists of **two independent parts**:

- **PART A — VPN Server:** Set up a VLESS + XTLS-Reality server using the 3X-UI web panel on a remote VPS.
- **PART B — Connecting to the VPN:** Connect to your VPN server. Two approaches are covered:
  - **Direct connection** using GUI client apps (Windows, macOS, iOS, Android) — simplest for personal devices
  - **System-wide proxy on a Linux server** using Xray core — routes all traffic from a server through the VPN

You can follow Part A alone if you just need a VPN server, or skip to Part B if someone else already set up the server for you.

## Table of Contents

**PART A — VPN Server**

1. [How It Works](#how-it-works)
2. [VPS Server Requirements](#vps-server-requirements)
3. [Install 3X-UI Panel](#install-3x-ui-panel)
   - [Connect to Your Server via SSH](#connect-to-your-server-via-ssh)
   - [Update the System](#update-the-system)
   - [Install Curl](#install-curl)
   - [Install the 3X-UI Panel](#install-the-3x-ui-panel)
   - [Initial Credentials and Panel URL](#initial-credentials-and-panel-url)
4. [Server Security](#server-security)
   - [Install fail2ban](#install-fail2ban)
   - [Create a New User with Sudo Privileges](#create-a-new-user-with-sudo-privileges)
   - [Disable Root SSH Login](#disable-root-ssh-login)
5. [Create VLESS + XTLS-Reality Protocol in 3X-UI](#create-vless--xtls-reality-protocol-in-3x-ui)
   - [Access the Web Panel](#access-the-web-panel)
   - [Add a New Connection](#add-a-new-connection)
   - [Configure the Inbound](#configure-the-inbound)
   - [Save](#save)
   - [Get Connection Details](#get-connection-details)

**PART B — Connecting to the VPN**

6. [Option 1: Direct Connection via Client Apps](#option-1-direct-connection-via-client-apps)
   - [Windows — Invisible Man XRay](#windows--invisible-man-xray)
   - [macOS — FoXray](#macos--foxray)
   - [iOS — FoXray](#ios--foxray)
   - [Android — NekoBox](#android--nekobox)
7. [Option 2: System-Wide Proxy on a Linux Server](#option-2-system-wide-proxy-on-a-linux-server)
   - [Install Xray Core](#install-xray-core)
   - [Create the Xray Config](#create-the-xray-config)
   - [Enable and Start Xray](#enable-and-start-xray)
   - [Configure System-Wide Proxy](#configure-system-wide-proxy)
   - [Persistence Summary](#persistence-summary)
8. [Expose Proxy to Remote Clients](#expose-proxy-to-remote-clients)
   - [Change Listen Address](#change-listen-address)
   - [Configure Firewall](#configure-firewall)
   - [Connect from Your Home Machine](#connect-from-your-home-machine)
9. [Troubleshooting](#troubleshooting)
   - [Xray won't start](#xray-wont-start)
   - [Proxy works but some sites don't load](#proxy-works-but-some-sites-dont-load)
   - [Check your exit IP](#check-your-exit-ip)
   - [Xray is running but connection fails](#xray-is-running-but-connection-fails)
   - [3X-UI panel is not accessible](#3x-ui-panel-is-not-accessible)
   - [Disable the proxy temporarily](#disable-the-proxy-temporarily)
   - [Full uninstall script (client side)](#full-uninstall-script-client-side)
10. [Quick Reference: Complete Client Setup Script](#quick-reference-complete-client-setup-script)

---

# PART A — VPN Server

Everything in this part happens on your **VPN server** — the machine that will act as the tunnel endpoint.

---

## How It Works

VLESS with XTLS-Reality uses a unique identification method during the TLS handshake to distinguish authorized clients from outsiders. When the server recognizes the client as "its own," it functions as a proxy. When unrecognized traffic arrives, the connection is redirected to a real website (e.g., google.com) and mimics its behavior.

Your proxy traffic is disguised as regular browser HTTPS traffic using a real TLS certificate. This makes proxy detection virtually impossible for deep packet inspection (DPI) systems.

**Why VLESS + XTLS-Reality over WireGuard or Shadowsocks?**

- WireGuard is already being blocked in some regions
- Shadowsocks is an older protocol with known detection signatures
- VLESS + XTLS-Reality is the most modern obfuscation and encryption protocol available

---

## VPS Server Requirements

> Skip this section if you already have a VPS server.

You need a VPS server in a country of your choice (Europe, US, etc.). Requirements:

- **OS:** Ubuntu 20.04 / 22.04 / 24.04
- **Minimum specs:** 1 CPU core, 512 MB RAM, 10 GB disk
- **Unlimited traffic** (important for VPN usage)
- **Dedicated IPv4 address**

**Recommended providers:**

| Use case | Provider | Notes |
|----------|----------|-------|
| VPN server (Europe / USA) | [HostVDS](https://hostvds.com) | Good selection of European and US locations, affordable plans, KVM virtualization, Ubuntu supported |
| Proxy server (Moscow / Russia) | [FirstVDS](https://firstvds.ru) | Russian data centers, low latency from Moscow, budget-friendly, Ubuntu supported |

> For a VPN server the location matters: choose a country where internet is not restricted. For a local proxy server (Part B, Option 2) you want a server close to your users — a Moscow-based VDS from FirstVDS works well if you and your users are in Russia.

**Possible setups — choose what fits your situation:**

---

**Setup 1 — VPN server only, direct connection from a personal device**

Use this when you just want to bypass restrictions on your phone, laptop, or PC.

```
Your device (phone / PC)
       │
       │  VLESS + XTLS-Reality tunnel, encrypted
       ▼
HostVDS server (Europe / USA)
  └─ runs 3X-UI + VLESS inbound on port 443
       │
       ▼
   Internet
```

→ Follow [Part A](#vps-server-requirements) to set up the VPN server, then [Part B → Option 1](#option-1-direct-connection-via-client-apps) to connect from your device using a GUI client app.

---

**Setup 2 — VPN server only, system-wide proxy on a Linux server**

Use this when you want an entire Linux server (e.g., a cloud VM) to route all its traffic through the VPN — no separate proxy VDS needed.

```
Linux server (any location)
  └─ runs Xray HTTP proxy on 127.0.0.1:10801
       │
       │  VLESS + XTLS-Reality tunnel, encrypted
       ▼
HostVDS server (Europe / USA)
  └─ runs 3X-UI + VLESS inbound on port 443
       │
       ▼
   Internet
```

→ Follow [Part A](#vps-server-requirements) to set up the VPN server, then [Part B → Option 2](#option-2-system-wide-proxy-on-a-linux-server) to configure system-wide proxy on the Linux server.

---

**Setup 3 — Two-server setup with a Russian proxy (recommended for Russia)**

Use this when you are based in Russia and want all your devices to go through the VPN via a single local entry point.

```
Your devices (Russia)
       │
       │  direct connection, low latency
       ▼
FirstVDS server (Moscow, Russia)
  └─ runs Xray HTTP proxy on port 10801
       │
       │  VLESS + XTLS-Reality tunnel, encrypted
       ▼
HostVDS server (Europe / USA)
  └─ runs 3X-UI + VLESS inbound on port 443
       │
       ▼
   Internet
```

- Your devices connect to the **FirstVDS proxy** using a short local hop (low latency, no censorship friction)
- The FirstVDS proxy forwards all traffic through an encrypted **VLESS + XTLS-Reality tunnel** to the HostVDS VPN server abroad
- From the outside, the tunnel looks like normal HTTPS traffic to google.com — DPI cannot distinguish it

→ Follow [Part A](#vps-server-requirements) to set up the HostVDS server, then [Part B → Option 2](#option-2-system-wide-proxy-on-a-linux-server) to configure the FirstVDS server as the proxy, and [Expose Proxy to Remote Clients](#expose-proxy-to-remote-clients) to allow your devices to connect to it.

---

Any other VPS provider will also work (Hetzner, DigitalOcean, Vultr, OVH, etc.).

After purchasing, you'll receive:
- Server IP address
- Root password (or SSH key)

---

## Install 3X-UI Panel

### Connect to Your Server via SSH

```bash
ssh root@YOUR_SERVER_IP
```

When asked about the fingerprint, type `yes` and press Enter. Then enter your password.

### Update the System

```bash
apt update && apt upgrade -y
```

### Install Curl

```bash
apt install curl -y
```

### Install the 3X-UI Panel

Run the official installation script from GitHub:

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

> This same script can be used to update the panel to the latest version at any time.

### Initial Credentials and Panel URL

After installation completes, it will ask if you want to make modifications — answer **Y** (yes).

- **Username:** Choose any username you like
- **Password:** Choose a strong password (mix of uppercase, lowercase, and numbers)
- **Port:** Choose a port for the web panel (e.g., `5580`; use a high port up to 65535)

> **Important:** If you skip the modification step (answer **N**), 3X-UI generates **random** credentials (login, password) and a **random port** with a **random URL path**. In that case, the default `http://YOUR_SERVER_IP:PORT/panel/` will **NOT** work — the panel will be at a randomized path like `http://YOUR_SERVER_IP:RANDOM_PORT/RANDOM_PATH/`. The installer prints these values to the terminal — copy them immediately. You can always change the credentials, port, and URL path later from within the panel settings, or by running `x-ui` in the terminal to access the admin menu.

The panel is now installed. You can manage it with these commands:

| Command | Action |
|---------|--------|
| `x-ui` | Enter admin menu |
| `x-ui start` | Start the panel |
| `x-ui stop` | Stop the panel |
| `x-ui restart` | Restart the panel |
| `x-ui status` | Show panel status |
| `x-ui enable` | Enable auto-start on boot |
| `x-ui disable` | Disable auto-start on boot |
| `x-ui log` | View logs |
| `x-ui update` | Update the panel |

---

## Server Security

### Install fail2ban

This blocks IP addresses after too many failed login attempts:

```bash
apt install fail2ban -y
systemctl start fail2ban
systemctl enable fail2ban
```

### Create a New User with Sudo Privileges

```bash
adduser YOUR_USERNAME
```

Enter a strong password and optionally fill in user details (or leave them blank).

Add the new account to the sudo group:

```bash
usermod -aG sudo YOUR_USERNAME
```

Switch to the new account:

```bash
su - YOUR_USERNAME
```

### Disable Root SSH Login

Edit the SSH config:

```bash
sudo nano /etc/ssh/sshd_config
```

Find the line `PermitRootLogin` and set it to:

```
PermitRootLogin no
```

Save with `Ctrl+X`, then `Y`, then `Enter`.

Restart SSH:

```bash
sudo service sshd restart
```

From now on, connect using: `ssh YOUR_USERNAME@YOUR_SERVER_IP`

---

## Create VLESS + XTLS-Reality Protocol in 3X-UI

### Access the Web Panel

Open in your browser:

```
http://YOUR_SERVER_IP:YOUR_PORT/YOUR_PATH/
```

> Note: use `http://`, not `https://`. Replace `YOUR_PORT` and `YOUR_PATH` with the values shown during installation. If you set custom values during the modification step (e.g., port `5580`), then the URL would be `http://YOUR_SERVER_IP:5580/panel/`.

Log in with the username and password you created (or the random ones generated during installation).

### Add a New Connection

1. Go to **Inbounds**
2. Click **Add Inbound**

### Configure the Inbound

**General settings:**

| Setting | Value |
|---------|-------|
| Remark | Any name (e.g., `VLESS XTLS-Reality`) |
| Protocol | `vless` |
| Listen IP | Leave empty |
| Port | `443` |

**Client settings:**

| Setting | Value |
|---------|-------|
| Email | Any client name (e.g., `MyClient`) |
| ID | Auto-generated UUID |
| Flow | `xtls-rprx-vision` (appears after enabling Reality) |

> The Flow field only appears after you enable Reality in the transport section below.

**Transport settings:**

| Setting | Value |
|---------|-------|
| Transmission Protocol | `TCP` |
| AcceptProxyProtocol | OFF |
| HTTP Masking | OFF |
| Transparent Proxy | OFF |
| TLS | OFF |
| **Reality** | **ON** |
| XTLS | OFF (this refers to the legacy XTLS protocol, not Reality) |
| xVer | `0` |
| uTLS | `chrome` |
| Domain | Leave empty (auto-fills server IP) |
| Dest | `google.com:443` |
| Server Names | `google.com, www.google.com` |
| ShortIds | Auto-generated |
| SpiderX | `/` |
| Private Key / Public Key | Click **Get New Key** |
| Sniffing | ON (HTTP, TLS, QUIC, FAKEDNS all checked) |

> Note: The XTLS and Reality toggles in 3X-UI are mutually exclusive. "XTLS" here means the older protocol versions. Reality is what you want.

### Save

Click **Create**. Your VLESS + XTLS-Reality inbound is ready!

### Get Connection Details

Click the **Info** icon next to your client to see the connection URL. It will look like:

```
vless://UUID@SERVER_IP:443/?type=tcp&security=reality&fp=chrome&pbk=PUBLIC_KEY&sni=google.com&flow=xtls-rprx-vision&sid=SHORT_ID&spx=%2F#NAME
```

Save this URL — you'll need it for client configuration. You can also click the QR code icon to get a scannable code for mobile apps.

---

# PART B — Connecting to the VPN

Everything below happens on the **client side** — the device or server that wants to use the VPN. You need the connection URL (or the individual values: server IP, UUID, public key, short ID) from Part A.

There are two fundamentally different approaches:

- **Option 1: Direct connection** — Use a GUI app (Windows, macOS, iOS, Android) that connects directly to the VPN. This is the simplest method. The app handles everything: it creates a local proxy or TUN interface, routes your traffic through the VPN, and you're done.
- **Option 2: System-wide proxy on a Linux server** — Install Xray core on another Ubuntu server, configure it as an HTTP proxy pointing to your VPN, and route all system traffic through it. This is useful when you want an entire server to go through the VPN (e.g., for web scraping, CI/CD, or bypassing geo-restrictions on a cloud server).

---

## Option 1: Direct Connection via Client Apps

These apps connect directly to your VLESS + XTLS-Reality server. No separate proxy setup is needed — just import the connection URL and click connect.

### Windows — Invisible Man XRay

**GitHub:** [https://github.com/InvisibleManVPN/InvisibleMan-XRayClient](https://github.com/InvisibleManVPN/InvisibleMan-XRayClient)

1. Download the latest release from [GitHub Releases](https://github.com/InvisibleManVPN/InvisibleMan-XRayClient/releases) and extract the archive
2. Launch the app and click **Manage server configuration**
3. Click **+** (plus button in the bottom-right corner)
4. Select **Import from link** and paste the `vless://...` URL from 3X-UI
5. Close the configuration manager and click **RUN**
6. The status should change from "Stopped" to "Connected"

### macOS — FoXray

**App Store:** [FoXray](https://apps.apple.com/app/foxray/id6448898396)

1. Download FoXray from the App Store
2. In 3X-UI, click the Info icon next to your client and copy the connection URL
3. In FoXray, click the **clipboard/paste** icon to import the URL
4. Press **Play** and allow the VPN configuration when prompted

### iOS — FoXray

**App Store:** [FoXray](https://apps.apple.com/app/foxray/id6448898396) (requires iOS 16+)

1. Download FoXray from the App Store
2. In 3X-UI, click the QR code icon next to your client
3. In FoXray, tap the **QR scan** icon (top-left corner) and scan the code
4. Tap **Play**, allow the VPN configuration, and enter your device passcode

### Android — NekoBox

**GitHub:** [https://github.com/MatsuriDayo/NekoBoxForAndroid](https://github.com/MatsuriDayo/NekoBoxForAndroid)

1. Download the latest APK from [GitHub Releases](https://github.com/MatsuriDayo/NekoBoxForAndroid/releases) (choose `arm64-v8a` for most modern devices)
2. Install the APK and launch NekoBox
3. Tap **+** (top-right corner) → **Scan QR code** and scan the QR code from 3X-UI
4. The new profile should appear in the list
5. Tap the **connect** button on the main screen and confirm the VPN connection request

---

## Option 2: System-Wide Proxy on a Linux Server

This approach installs Xray core on another Ubuntu server. Xray creates a local HTTP proxy (`127.0.0.1:10801`) that tunnels all traffic through your VLESS VPN server. Then you configure the OS to route everything through that proxy.

### Install Xray Core

```bash
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

### Create the Xray Config

Replace the placeholder values below with your actual server details from the 3X-UI connection URL.

```bash
sudo tee /usr/local/etc/xray/config.json > /dev/null << 'EOF'
{
  "log": {
    "access": "",
    "error": "",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 10801,
      "listen": "127.0.0.1",
      "protocol": "http",
      "settings": {
        "udp": true,
        "allowTransparent": false
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "YOUR_VPN_SERVER_IP",
            "port": 443,
            "users": [
              {
                "id": "YOUR_UUID",
                "alterId": 0,
                "email": "t@t.tt",
                "encryption": "none",
                "flow": "xtls-rprx-vision"
              }
            ]
          }
        ],
        "userLevel": 0
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "fingerprint": "chrome",
          "serverName": "www.google.com",
          "publicKey": "YOUR_PUBLIC_KEY",
          "shortId": "YOUR_SHORT_ID",
          "spiderX": "/"
        }
      }
    }
  ]
}
EOF
```

**Values to replace:**

| Placeholder | Where to find it |
|-------------|-----------------|
| `YOUR_VPN_SERVER_IP` | Your VPN server's IP address |
| `YOUR_UUID` | The client ID from 3X-UI (e.g., `876e7cdd-44e3-4e71-86e1-66b8959ca6ea`) |
| `YOUR_PUBLIC_KEY` | The Public Key generated in 3X-UI |
| `YOUR_SHORT_ID` | The ShortId from 3X-UI |

### Enable and Start Xray

```bash
sudo systemctl daemon-reload
sudo systemctl enable xray
sudo systemctl start xray
```

Verify it's running:

```bash
sudo systemctl status xray --no-pager
```

Test the connection:

```bash
curl -x http://127.0.0.1:10801 -s -o /dev/null -w "HTTP status: %{http_code}\n" https://www.google.com
```

You should see `HTTP status: 200`.

Check your exit IP:

```bash
curl -x http://127.0.0.1:10801 https://ifconfig.me
```

This should return your VPN server's IP, not the client server's IP.

### Configure System-Wide Proxy

#### Proxy for All Login Shells

Create `/etc/profile.d/proxy.sh`:

```bash
sudo tee /etc/profile.d/proxy.sh > /dev/null << 'EOF'
export http_proxy="http://127.0.0.1:10801"
export https_proxy="http://127.0.0.1:10801"
export HTTP_PROXY="http://127.0.0.1:10801"
export HTTPS_PROXY="http://127.0.0.1:10801"
export no_proxy="localhost,127.0.0.1,::1"
export NO_PROXY="localhost,127.0.0.1,::1"
EOF
```

Load it into the current session:

```bash
source /etc/profile.d/proxy.sh
```

> Both lowercase and uppercase variants are needed because different tools read different ones — curl and wget use lowercase, some Java/Python libraries use uppercase.

#### Proxy for APT (Optional)

```bash
sudo tee /etc/apt/apt.conf.d/95proxy > /dev/null << 'EOF'
Acquire::http::Proxy "http://127.0.0.1:10801";
Acquire::https::Proxy "http://127.0.0.1:10801";
EOF
```

#### Proxy for All Systemd Services (Optional)

This ensures background services (Docker, cron jobs, nginx, etc.) also use the proxy:

```bash
sudo mkdir -p /etc/systemd/system.conf.d

sudo tee /etc/systemd/system.conf.d/proxy.conf > /dev/null << 'EOF'
[Manager]
DefaultEnvironment="http_proxy=http://127.0.0.1:10801" "https_proxy=http://127.0.0.1:10801" "HTTP_PROXY=http://127.0.0.1:10801" "HTTPS_PROXY=http://127.0.0.1:10801" "no_proxy=localhost,127.0.0.1,::1"
EOF

sudo systemctl daemon-reexec
```

### Persistence Summary

Everything above persists across reboots automatically:

| Component | File | Persists? |
|-----------|------|-----------|
| Xray service | `systemctl enable xray` | Yes |
| Shell proxy | `/etc/profile.d/proxy.sh` | Yes (every login shell) |
| APT proxy | `/etc/apt/apt.conf.d/95proxy` | Yes |
| Systemd proxy | `/etc/systemd/system.conf.d/proxy.conf` | Yes |

---

## Expose Proxy to Remote Clients

By default, Xray listens on `127.0.0.1:10801` (local only). If you want other machines (e.g., your home PC) to connect through this server's proxy instead of running their own Xray instance:

### Change Listen Address

Edit `/usr/local/etc/xray/config.json` and change:

```json
"listen": "0.0.0.0"
```

Restart Xray:

```bash
sudo systemctl restart xray
```

### Configure Firewall

**Option A — Open to everyone (no IP restriction)**

Use this if you want any machine to be able to connect to your proxy (e.g., a small trusted network, or you'll handle access control at the application level):

```bash
sudo ufw allow 10801/tcp
```

> **Warning:** An open proxy with no authentication is a security risk. Anyone on the internet who discovers port 10801 can route their traffic through your server and VPN. Only use this on isolated or trusted networks.

**Option B — Restrict to a specific IP (recommended)**

Use this to allow only your home/office IP and block everyone else:

```bash
# Allow only your IP to access the proxy
sudo ufw allow from YOUR_HOME_IP to any port 10801 proto tcp

# Deny everyone else
sudo ufw deny 10801/tcp
```

> Replace `YOUR_HOME_IP` with your actual public IP address (find it with `curl ifconfig.me` from your home machine).

In both cases, make sure ufw is enabled:

```bash
sudo ufw enable
sudo ufw status
```

### Connect from Your Home Machine

Set your system proxy to:

```
http://YOUR_CLIENT_SERVER_IP:10801
```

Test with curl:

```bash
curl -x http://YOUR_CLIENT_SERVER_IP:10801 https://ifconfig.me
```

This should return the VPN server's IP.

---

## Troubleshooting

### Xray won't start

```bash
# Check the logs
sudo journalctl -u xray -n 50 --no-pager

# Validate the config
sudo /usr/local/bin/xray run -test -config /usr/local/etc/xray/config.json
```

Common issues:
- Invalid JSON (trailing commas, missing brackets)
- Wrong UUID format
- Mismatched public key / short ID

### Proxy works but some sites don't load

The HTTP proxy only handles HTTP/HTTPS traffic. For full system tunneling (including DNS and raw TCP), you'd need a SOCKS5 inbound or transparent proxy with iptables rules.

Add a SOCKS5 inbound alongside the HTTP one in the `inbounds` array:

```json
{
  "port": 10802,
  "listen": "127.0.0.1",
  "protocol": "socks",
  "settings": {
    "udp": true
  }
}
```

### Check your exit IP

```bash
# Through the proxy
curl -x http://127.0.0.1:10801 https://ifconfig.me

# Direct (without proxy)
curl https://ifconfig.me
```

### Xray is running but connection fails

1. Verify the VPN server is reachable: `ping YOUR_VPN_SERVER_IP`
2. Verify port 443 is open on the VPN server: `nc -zv YOUR_VPN_SERVER_IP 443`
3. Check that the UUID, public key, and short ID match exactly between 3X-UI and the client config
4. Ensure Reality is enabled (not TLS or XTLS) in 3X-UI

### 3X-UI panel is not accessible

If `http://YOUR_SERVER_IP:PORT/panel/` returns nothing, the panel is likely using a **randomized URL path**. Run `x-ui` in SSH to access the admin menu, where you can view or reset the panel URL, port, and credentials.

### Disable the proxy temporarily

```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY no_proxy NO_PROXY
```

### Full uninstall script (client side)

```bash
sudo systemctl stop xray
sudo systemctl disable xray
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ remove
sudo rm -f /etc/profile.d/proxy.sh
sudo rm -f /etc/apt/apt.conf.d/95proxy
sudo rm -f /etc/systemd/system.conf.d/proxy.conf
sudo systemctl daemon-reexec
```

---

## Quick Reference: Complete Client Setup Script

Copy and customize this script for fast deployment on a fresh Ubuntu 24 server:

```bash
#!/usr/bin/env bash
set -euo pipefail

# ═══════════════════════════════════════════════════════
# CONFIGURE THESE VALUES
# ═══════════════════════════════════════════════════════
VPN_SERVER_IP="YOUR_VPN_SERVER_IP"
UUID="YOUR_UUID"
PUBLIC_KEY="YOUR_PUBLIC_KEY"
SHORT_ID="YOUR_SHORT_ID"
# ═══════════════════════════════════════════════════════

# 1. Install Xray
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install

# 2. Write config
cat > /usr/local/etc/xray/config.json << EOF
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "port": 10801,
      "listen": "127.0.0.1",
      "protocol": "http",
      "settings": { "udp": true, "allowTransparent": false }
    }
  ],
  "outbounds": [
    {
      "protocol": "vless",
      "settings": {
        "vnext": [{
          "address": "${VPN_SERVER_IP}",
          "port": 443,
          "users": [{
            "id": "${UUID}",
            "alterId": 0,
            "encryption": "none",
            "flow": "xtls-rprx-vision"
          }]
        }],
        "userLevel": 0
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "fingerprint": "chrome",
          "serverName": "www.google.com",
          "publicKey": "${PUBLIC_KEY}",
          "shortId": "${SHORT_ID}",
          "spiderX": "/"
        }
      }
    }
  ]
}
EOF

# 3. Start Xray
systemctl daemon-reload
systemctl enable xray
systemctl start xray
sleep 2
systemctl status xray --no-pager

# 4. System-wide proxy (login shells)
cat > /etc/profile.d/proxy.sh << 'PROXYEOF'
export http_proxy="http://127.0.0.1:10801"
export https_proxy="http://127.0.0.1:10801"
export HTTP_PROXY="http://127.0.0.1:10801"
export HTTPS_PROXY="http://127.0.0.1:10801"
export no_proxy="localhost,127.0.0.1,::1"
export NO_PROXY="localhost,127.0.0.1,::1"
PROXYEOF

# 5. APT proxy
cat > /etc/apt/apt.conf.d/95proxy << 'APTEOF'
Acquire::http::Proxy "http://127.0.0.1:10801";
Acquire::https::Proxy "http://127.0.0.1:10801";
APTEOF

# 6. Systemd services proxy
mkdir -p /etc/systemd/system.conf.d
cat > /etc/systemd/system.conf.d/proxy.conf << 'SYSEOF'
[Manager]
DefaultEnvironment="http_proxy=http://127.0.0.1:10801" "https_proxy=http://127.0.0.1:10801" "HTTP_PROXY=http://127.0.0.1:10801" "HTTPS_PROXY=http://127.0.0.1:10801" "no_proxy=localhost,127.0.0.1,::1"
SYSEOF
systemctl daemon-reexec

# 7. Load proxy in current session
source /etc/profile.d/proxy.sh

# 8. Test
echo "---"
echo "Exit IP (should be VPN server):"
curl -x http://127.0.0.1:10801 -s https://ifconfig.me
echo ""
echo "---"
echo "Setup complete! Log out and back in for proxy to take effect in new shells."
```

Save as `setup-xray-vpn.sh`, edit the 4 variables at the top, and run with:

```bash
chmod +x setup-xray-vpn.sh
sudo ./setup-xray-vpn.sh
```

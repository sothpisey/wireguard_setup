# WireGuard VPN Setup Guide

A complete guide to setting up a WireGuard VPN server with a mobile client, based on the configuration files `wg0.conf` (server) and `mobile.conf` (client).

---

## Architecture Overview

```
Mobile Client (10.8.0.2)
        |
        | Internet (UDP 51820)
        |
VPN Server (10.8.0.1) — Public IP: 45.32.109.14
        |
  enp1s0 (LAN/WAN)
```

| Role          | VPN IP      | Public IP      |
|---------------|-------------|----------------|
| Server        | 10.8.0.1/24 | 45.32.109.14   |
| Mobile Client | 10.8.0.2/24 | (dynamic)      |

---

## Part 1: Server Setup

### 1.1 Install WireGuard

```bash
sudo apt update && sudo apt install wireguard -y
```

### 1.2 Generate Server Key Pair

```bash
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
chmod 600 /etc/wireguard/privatekey
```

View the keys:

```bash
cat /etc/wireguard/privatekey   # Keep this SECRET
cat /etc/wireguard/publickey    # Share this with clients
```

### 1.3 Find Your Network Interface

The server config uses `enp1s0` for NAT. Confirm your actual interface name:

```bash
ip -o -4 route show to default | awk '{print $5}'
```

> ⚠️ If the output is **not** `enp1s0`, update the `PostUp` and `PostDown` lines in `wg0.conf` accordingly.

### 1.4 Create the Server Config

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>

# Replace 'enp1s0' with your actual primary network interface name!
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o enp1s0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o enp1s0 -j MASQUERADE

[Peer]
# Mobile Client
PublicKey = 35ESuZTsdznVFpxLL6spf1tQnDVhRfy2la/BfzEriFU=
AllowedIPs = 10.8.0.2/32
```

> The `PostUp` / `PostDown` rules enable NAT (masquerade), allowing VPN clients to route their traffic through the server's internet connection.

Set correct permissions:

```bash
chmod 600 /etc/wireguard/wg0.conf
```

### 1.5 Enable IP Forwarding

Check current state:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Enable immediately (temporary):

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Make it **permanent** across reboots:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 1.6 Start and Enable the WireGuard Service

```bash
# Start the interface
sudo wg-quick up wg0

# Enable on boot
sudo systemctl enable wg-quick@wg0

# Check status
sudo systemctl status wg-quick@wg0
```

### 1.7 Open Firewall Port

Allow UDP port 51820 through the firewall:

```bash
sudo ufw allow 51820/udp
sudo ufw reload
```

---

## Part 2: Mobile Client Setup

### 2.1 Generate Client Key Pair

On the **server** (or any machine with WireGuard tools):

```bash
wg genkey | tee mobile_privatekey | wg pubkey > mobile_publickey
```

### 2.2 Mobile Client Config (`mobile.conf`)

```ini
[Interface]
# The phone's internal VPN IP
Address = 10.8.0.2/24
DNS = 8.8.8.8, 1.1.1.1
PrivateKey = <MOBILE_PRIVATE_KEY>

[Peer]
# The Server's Public Key
PublicKey = WO8NY0H+rZgm0yUBsLbfOO+wMW3wcFAI4G7/F6ZUXz8=

# Your Server's Public IP and Port
Endpoint = 45.32.109.14:51820

# Route ALL mobile traffic through the VPN
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 20
```

**Key fields explained:**

| Field | Value | Purpose |
|-------|-------|---------|
| `Address` | `10.8.0.2/24` | Client's IP inside the VPN tunnel |
| `DNS` | `8.8.8.8, 1.1.1.1` | DNS servers used when VPN is active |
| `Endpoint` | `45.32.109.14:51820` | Server's public IP and UDP port |
| `AllowedIPs` | `0.0.0.0/0` | Routes **all** traffic through the VPN |
| `PersistentKeepalive` | `20` | Sends a keepalive packet every 20s to maintain NAT mappings |

### 2.3 Install on Mobile

**Android / iOS — WireGuard App:**

1. Install the [WireGuard app](https://www.wireguard.com/install/) from the App Store or Play Store.
2. Follow the steps below to generate a QR code and scan it in the app.
3. Tap **Add Tunnel → Scan QR Code** and point your camera at the QR code.

---

### 2.4 Generating a QR Code with `qrencode`

`qrencode` converts the `mobile.conf` file into a scannable QR code so you never need to manually type keys or IPs on your phone.

#### Install

```bash
sudo apt install qrencode -y
```

#### Display QR Code in the Terminal

```bash
qrencode -t ansiutf8 < /etc/wireguard/mobile.conf
```

This prints the QR code directly in the terminal using Unicode block characters. Use this when you have SSH access to the server and want the fastest workflow.

> 💡 **Tip:** If the QR code looks broken, try reducing your terminal font size or zooming out — the QR code needs enough pixels to be scannable.

#### Save QR Code as a PNG Image

```bash
qrencode -t png -o /tmp/mobile-qr.png < /etc/wireguard/mobile.conf
```

Then transfer it to your local machine for viewing:

```bash
scp user@45.32.109.14:/tmp/mobile-qr.png ~/Desktop/mobile-qr.png
```

#### Output Format Options

| Flag | Format | Use Case |
|------|--------|----------|
| `-t ansiutf8` | Terminal (Unicode) | Quick display over SSH |
| `-t png -o file.png` | PNG image file | Save and view on another device |
| `-t svg -o file.svg` | SVG vector image | Print or embed in documentation |
| `-t utf8` | Terminal (fallback) | Use if `ansiutf8` looks wrong |

#### Security: Delete the QR Image After Use

The QR code encodes the **private key** of the client. Delete it from the server once you've scanned it:

```bash
rm /tmp/mobile-qr.png
```

> ⚠️ **Never leave QR images or private key files in publicly accessible directories.**

---

## Part 3: Adding the Client's Public Key to the Server

After generating the client keys, register the mobile client on the server:

**Option A — Edit `wg0.conf` directly** (requires restart):

```ini
[Peer]
# Mobile Client
PublicKey = <MOBILE_PUBLIC_KEY>
AllowedIPs = 10.8.0.2/32
```

Then restart the interface:

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

**Option B — Add peer live** (no restart needed):

```bash
sudo wg set wg0 peer <MOBILE_PUBLIC_KEY> allowed-ips 10.8.0.2/32
```

---

## Part 4: Managing the VPN

### Service Control

```bash
# Start
sudo wg-quick up wg0

# Stop
sudo wg-quick down wg0

# Restart (apply config changes)
sudo systemctl restart wg-quick@wg0

# Status
sudo systemctl status wg-quick@wg0
```

### Live Monitoring

```bash
# Show connected peers and traffic stats
sudo wg show

# Watch traffic in real time
sudo watch -n 1 wg show
```

### View Active Config

```bash
sudo wg showconf wg0
```

---

## Part 5: Verification & Troubleshooting

### ✅ Verify the Connection

From the mobile client, after connecting:

```bash
# Ping the server's VPN IP
ping 10.8.0.1

# Check your public IP (should show the server's IP)
curl ifconfig.me
```

On the server, check that the peer appears and traffic is flowing:

```bash
sudo wg show
```

Expected output:
```
interface: wg0
  public key: WO8NY0H+...
  private key: (hidden)
  listening port: 51820

peer: 35ESuZTs...
  endpoint: <client-ip>:<port>
  allowed ips: 10.8.0.2/32
  latest handshake: X seconds ago
  transfer: X MiB received, X MiB sent
```

### 🔧 Common Issues

| Problem | Likely Cause | Fix |
|--------|-------------|-----|
| No handshake | Firewall blocking UDP 51820 | `sudo ufw allow 51820/udp` |
| Can't reach internet through VPN | IP forwarding disabled | `sudo sysctl -w net.ipv4.ip_forward=1` |
| Traffic not routing | Wrong interface in `PostUp` | Replace `enp1s0` with correct interface |
| Peer drops connection | No keepalive | Set `PersistentKeepalive = 20` on client |
| Config changes not applied | Interface not restarted | `sudo wg-quick down wg0 && sudo wg-quick up wg0` |

---

## Part 6: Security Recommendations

- **Never share private keys.** Each device needs its own key pair.
- **Restrict file permissions:**
  ```bash
  chmod 600 /etc/wireguard/privatekey
  chmod 600 /etc/wireguard/wg0.conf
  ```
- **Use unique IPs per client.** Assign each peer its own `/32` IP (e.g., `10.8.0.2/32`, `10.8.0.3/32`).
- **Keep WireGuard updated:**
  ```bash
  sudo apt update && sudo apt upgrade wireguard
  ```
- **Revoke a client** by removing its `[Peer]` block from `wg0.conf` and restarting the service.

---

## Quick Reference

```bash
# Key generation
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey

# Start/stop interface
sudo wg-quick up wg0
sudo wg-quick down wg0

# Restart service
sudo systemctl restart wg-quick@wg0

# Check status
sudo systemctl status wg-quick@wg0
sudo wg show

# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# QR code for mobile
qrencode -t ansiutf8 < /etc/wireguard/mobile.conf
```

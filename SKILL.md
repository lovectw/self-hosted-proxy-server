---
name: self-hosted-proxy-server
description: "Set up self-hosted proxy servers on Linux for mobile/remote device access (Mihomo + Shadowsocks chaining, WireGuard VPN, Xray VLESS)."
tags: [proxy, shadowsocks, mihomo, clash, vpn, network, socks5, http-proxy, iptables, aws-security-group, wireguard, xray, vless]
platforms: [linux, ubuntu, aws]
---

# Self-Hosted Proxy Server

## When to use
- User wants a proxy server on a Linux VPS/cloud instance for mobile or remote devices to use
- User needs to access foreign websites via a cloud server
- Chaining a SOCKS5/HTTP frontend proxy to an upstream SS/Shadowsocks server

## Architecture — Two Options

### Option A: HTTP/SOCKS5 Proxy (Mihomo + Shadowsocks)
**Best for:** Android w/ GMS, devices with traditional proxy support
**Not for:** 纯血鸿蒙 NEXT (HarmonyOS without Android compat layer)

```
Mobile/Android (GMS)
        │
        ▼
┌───────────────────┐
│  Mihomo (:7890)   │  ← HTTP/SOCKS5, mobile-facing
│  allow-lan: true  │
└────────┬──────────┘
         │ ss protocol
         ▼
┌───────────────────────┐
│  Shadowsocks (:8388)  │  ← SS protocol, upstream
│  aes-256-gcm          │
└────────┬──────────────┘
         │ direct connection
         ▼
    Foreign Internet
```

### Option B: WireGuard VPN
**Best for:** 纯血鸿蒙 NEXT, iOS, devices that only support VPN-mode tunnels
**Limitation:** WireGuard client must be available on the device (AppGallery上架情况不稳定)
**Why:** WireGuard is kernel-level; HarmonyOS NEXT routes ALL traffic through it (no app-level proxy bypass)

### Option C: Xray + VLESS (Recommended for 纯血鸿蒙 NEXT)
**Best for:** 纯血鸿蒙 NEXT when no WireGuard app is installable
**Why:** VLESS + real TLS looks like normal HTTPS traffic; clients like V2RayNG (APK install) work on鸿蒙
**Tradeoff:** Requires APK install (no AppGallery); needs port 80 open for Let's Encrypt certificate issuance

```
Mobile/HarmonyOS NEXT
        │
        ▼
┌──────────────────────────┐
│  WireGuard (:51820/UDP)  │  ← VPN tunnel, full system routing
│  10.0.0.1/24             │
└────────┬─────────────────┘
         │ tunneled
         ▼
    Foreign Internet
```

**Decision rule:**
1. Ask user what device/OS they have
2. If 纯血鸿蒙 NEXT → try WireGuard first; if WireGuard app unavailable → use Xray/VLESS (Option C)
3. If Android with GMS → use HTTP proxy (Option A)

## WireGuard VPN Setup (Option B — for 纯血鸿蒙 NEXT)

### 1. Install
```bash
sudo apt-get update && sudo apt-get install -y wireguard wireguard-tools
```

### 2. Generate Keys (server + client)
```bash
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key
chmod 600 /etc/wireguard/*.key
```

### 3. Server Config — /etc/wireguard/wg0.conf
```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>

# NAT / forwarding — eth0 is the primary NIC on AWS EC2
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = 10.0.0.2/32
```

### 4. Client Config — give to user (for QR code or manual import)
```ini
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <SERVER_PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

### 5. Enable IP Forwarding
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

### 6. Open UDP 51820 in Cloud Firewall

**AWS EC2 (from within instance with IAM role):**
```bash
SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=launch-wizard-*" \
  --query 'SecurityGroups[0].GroupId' --output text)
aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID --protocol udp \
  --port 51820 --cidr 0.0.0.0/0
```

**Other providers:** open UDP 51820 from 0.0.0.0/0 in their firewall/dashboard.

### 7. Start
```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0

# Verify
sudo wg show wg0
```

### 8. Generate QR Code for Phone
On the server (requires qrencode):
```bash
sudo apt-get install -y qrencode
cat /path/to/client.conf | qrencode -t ansiutf8
```
User scans it with the WireGuard app on their phone.

### 9. Mobile Client Setup
- **纯血鸿蒙 NEXT:** AppGallery → search "WireGuard" → 新建隧道 → 导入配置
- **iOS:** WireGuard app from App Store → scan QR code
- Set AllowedIPs = `0.0.0.0/0` (full tunnel — routes all traffic through VPN)

---

## Cloud Provider Prerequisites

### AWS EC2
1. Open ports in **Security Group** (EC2 → Security Groups → Edit inbound rules):
   - `7890/TCP` — Mihomo HTTP/SOCKS5 (Option A)
   - `8388/TCP` — Shadowsocks (Option A)
   - `51820/UDP` — WireGuard (Option B)
2. If using AWS CLI from within the instance, the instance **needs an IAM role** with EC2 permissions — IMDS metadata access alone is not sufficient for `aws ec2 *` commands

### AWS Security Group Checklist (Critical — Do First)
After any EC2 stop/start or new instance, verify security group BEFORE troubleshooting proxy issues:
- `7890/TCP` — Mihomo HTTP/SOCKS5 — 0.0.0.0/0
- `8388/TCP` — Shadowsocks — 0.0.0.0/0
- `51820/UDP` — WireGuard — 0.0.0.0/0 (if using Option B)
- After creating a new instance, these rules are often missing — add them with the `authorize-security-group-ingress` command above in the WireGuard section

### Other providers
- DigitalOcean: Firewall → Add rules for 7890, 8388
- Vultr/linode: Cloud Firewall → same ports
- GCP: Firewall rules → allow TCP 7890, 8388 from 0.0.0.0/0

## Service Installation

### Shadowsocks (ss-server)

**Option 1: snap package (Ubuntu)** — recommended for easy updates
```bash
sudo snap install shadowsocks-libev
```

The snap binary requires `snap run`, NOT direct execution:
```
ExecStart=/usr/bin/snap run shadowsocks-libev.ss-server -s 0.0.0.0 -p 8388 -k PASSWORD -m aes-256-gcm
```

**Do NOT use** the raw path `/snap/shadowsocks-libev/<rev>/usr/bin/ss-server` in systemd — it will fail with exit 127 (missing shared libraries).

**Option 2: v2.8.2 via pip** — for older servers (Python < 3.14)
```bash
pip install shadowsocks==2.8.2 --user
ss-server -s 0.0.0.0 -p 8388 -k PASSWORD -m aes-256-gcm
```
Note: Python 3.14 breaks all `shadowsocks` < 2.9 (collections.MutableMapping removed).

### Mihomo (Clash Meta)

Download from GitHub (use mirror if blocked):
```bash
cd /tmp
curl -LO https://github.com/MetaCubeX/mihomo/releases/download/v1.18.10/mihomo-linux-amd64-v1.18.10.gz
gunzip mihomo-linux-amd64-v1.18.10.gz
sudo mv mihomo-linux-amd64-v1.18.10 /usr/local/bin/mihomo
sudo chmod +x /usr/local/bin/mihomo
```

## Configuration

### Shadowsocks
Minimal config via command line (systemd service or manual):
```
ss-server -s 0.0.0.0 -p 8388 -k PASSWORD -m aes-256-gcm
```

### Mihomo config.yaml

Key settings for mobile-facing proxy:
```yaml
mixed-port: 7890          # HTTP + SOCKS5 on same port
allow-lan: true           # MUST be true for remote mobile access
bind-address: "*"
mode: rule
log-level: info
ipv6: false
external-controller: 127.0.0.1:9090  # REST API (optional)
dns:
  enable: true
  listen: 0.0.0.0:53       # Built-in DNS (optional, needs port 53)
  enhanced-mode: fake-ip
  nameserver:
    - 223.5.5.5           # Domestic DNS
    - 119.29.29.29

proxies:
  # Option A: Chain to local Shadowsocks
  - name: "Local-Shadowsocks"
    type: ss
    server: 127.0.0.1
    port: 8388
    cipher: aes-256-gcm
    password: "PASSWORD"

  # Option B: Upstream remote proxy provider
  - name: "Remote-Provider"
    type: ss
    server: your.proxy.server
    port: 8443
    cipher: aes-256-gcm
    password: "remote-password"

proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - "Local-Shadowsocks"   # or "Remote-Provider"

rules:
  - GEOIP,CN,DIRECT
  - MATCH,PROXY
```

## systemd Service Files

### /etc/systemd/system/shadowsocks.service
```ini
[Unit]
Description=Shadowsocks Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/snap run shadowsocks-libev.ss-server -s 0.0.0.0 -p 8388 -k PASSWORD -m aes-256-gcm
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### /etc/systemd/system/mihomo.service
```ini
[Unit]
Description=Mihomo Proxy
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mihomo -d /home/ubuntu/mihomo
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mihomo shadowsocks
```

## Verification

```bash
# Check processes
ps aux | grep -E 'mihomo|ss-server' | grep -v grep

# Check listening ports
ss -tlnp | grep -E '7890|8388'

# Get current public IP (important: EC2 IPs change on stop/start)
curl -s ifconfig.me

# Test from local
curl -s --proxy http://127.0.0.1:7890 https://www.google.com -o /dev/null -w "本地测试: %{http_code}\n"

# Test from external IP (after security group is confirmed open)
curl -s --proxy http://<PUBLIC_IP>:7890 https://www.google.com -o /dev/null -w "外部测试: %{http_code}\n"
```

## Mobile Client Setup (Hamidroid/鸿蒙)

| Field | Value |
|-------|-------|
| Proxy type | HTTP or SOCKS5 |
| Server | <SERVER_PUBLIC_IP> |
| Port | 7890 |
| (If SOCKS5) | Protocol: SOCKS5 |

### Adding Password Protection (Optional)

Mihomo supports per-user authentication on the mixed port:

```yaml
mixed-port: 7890
allow-lan: true
authentication:
  - "user1:pass1234"
```

Restart Mihomo after adding/removing users:
```bash
sudo systemctl restart mihomo
```

Test locally:
```bash
# With auth
curl --proxy http://user1:pass1234@<SERVER_IP>:7890 https://www.google.com

# Without auth (should fail)
curl --proxy http://<SERVER_IP>:7890 https://www.google.com
```

## Xray + VLESS Setup (Option C — for 纯血鸿蒙 NEXT without WireGuard App)

### Prerequisites
- Ports 80 (TCP) and 443 (TCP) open in cloud firewall
- A domain name resolving to the server (use sslip.io free subdomain: `<anything>-<IP>.sslip.io`)

### 1. Install Xray
```bash
wget -O ~/xray-install.sh "https://github.com/XTLS/Xray-install/raw/refs/heads/main/install-release.sh"
sudo bash ~/xray-install.sh
# Installs: /usr/local/bin/xray, /usr/local/etc/xray/config.json, systemd service
```

### 2. Get TLS Certificate (Let's Encrypt)
```bash
sudo apt-get install -y certbot
DOMAIN="vless-<SERVER_IP>.sslip.io"  # e.g. vless-52-74-246-53.sslip.io
sudo certbot certonly --standalone --agree-tos --email your@email.com --non-interactive -d $DOMAIN
# Certificates saved to: /etc/letsencrypt/live/$DOMAIN/
```

### 3. Copy Certificates to xray-readable location
```bash
sudo mkdir -p /etc/xray/certs
sudo cp /etc/letsencrypt/live/$DOMAIN/fullchain.pem /etc/xray/certs/
sudo cp /etc/letsencrypt/live/$DOMAIN/privkey.pem /etc/xray/certs/
sudo chmod 644 /etc/xray/certs/*
```
**Why:** Xray runs as `nobody` user; default `/etc/letsencrypt/` paths are not world-readable.

### 4. Generate VLESS UUID
```bash
cat /proc/sys/kernel/random/uuid
# Or via xray: /usr/local/bin/xray x25519
```

### 5. Xray Server Config — /usr/local/etc/xray/config.json
```json
{
  "log": {"loglevel": "warning"},
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [{"id": "<UUID>"}],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "tls",
        "tlsSettings": {
          "certificates": [
            {
              "certificateFile": "/etc/xray/certs/fullchain.pem",
              "keyFile": "/etc/xray/certs/privkey.pem"
            }
          ],
          "alpn": ["h2", "http/1.1"]
        }
      },
      "sniffing": {"enabled": true, "destOverride": ["http", "tls"]}
    }
  ],
  "outbounds": [
    {"protocol": "freedom", "tag": "direct"},
    {"protocol": "blackhole", "tag": "block"}
  ]
}
```

### 6. Start Xray
```bash
sudo systemctl enable --now xray
# Verify
ss -tlnp | grep ':443'
```

### 7. Client URI (for V2RayNG / Shadowrocket / etc.)
```
vless://<UUID>@<DOMAIN>:443?encryption=none&security=tls&alpn=h2,http/1.1&type=tcp&headerType=none&sni=<DOMAIN>#<REMARK>
```

**Note:** Do NOT include `flow=xtls-rprx-vision` in the URI — it causes "client flow is empty" rejection on servers without corresponding inbound flow. Use plain TLS TCP.

### 8. Mobile Client Setup
1. Install V2RayNG APK on the phone (download from GitHub 2dust/v2rayNG releases, transfer via USB or cloud)
2. Enable "Allow unknown apps" in phone security settings
3. Open V2RayNG → + → Import from clipboard (paste the URI) or Manual entry
4. Protocol: VLESS, fill in fields from the URI above
5. Connection test → browser open youtube.com

### Key Differences from HTTP Proxy
| | Option A (HTTP) | Option C (VLESS) |
|---|---|---|
| Protocol | HTTP/SOCKS5 | VLESS + TLS |
| App behavior | App-level, bypassable | VPN-mode (full tunnel) |
| 纯血鸿蒙 | ❌ App traffic bypasses proxy | ✅ Works |
| Certificate | None | Real Let's Encrypt TLS |
| Client APK needed | System proxy | V2RayNG APK |
| Port used | 7890/TCP | 443/TCP |

---

## Mobile-Side Troubleshooting

### NET_ERROR(ERR_EMPTY_RESPONSE, -324) on 鸿蒙/Chrome

**Symptom:** Server-side `curl` tests work perfectly (200 OK), but phone browser/app shows `ERR_EMPTY_RESPONSE`.

**Root cause:** The phone may be using QUIC/HTTP3 for Google/YouTube, which bypasses system HTTP proxy settings entirely.

**Debugging steps:**
1. On phone browser (not app), try opening `https://www.google.com` and `https://youtube.com`
   - If browser works → App QUIC issue (see below)
   - If browser also fails → DNS issue
2. Check DNS: phone's DNS may be polluted even though proxy tunnel works
   - Configure Wi-Fi static DNS: `8.8.8.8` / `8.8.4.4` (Google DNS)
   - Path: Wi-Fi → long-press network → modify network → advanced → IP settings = static
3. For YouTube App specifically: QUIC transport is hardcoded — the app may bypass the system proxy
   - Try using browser for YouTube instead
   - Or disable QUIC in Chrome flags (`chrome://flags/#quic`)

### DNS Configuration on Phone

When the proxy is working but DNS is broken (sites unreachable by name):
1. Wi-Fi → connected network → modify → advanced options
2. IP settings: **Static**
3. Add DNS servers:
   - DNS 1: `8.8.8.8`
   - DNS 2: `8.8.4.4`
4. Save and restart browser

## Service Management

### Start / Restart
When the proxy is unreachable from mobile devices, always check service status first:

```bash
# Check if services are running
ps aux | grep -E 'mihomo|ss-server' | grep -v grep
systemctl status mihomo shadowsocks

# If not running, start them:
sudo systemctl daemon-reload
sudo systemctl enable --now shadowsocks mihomo
```

### Verify External Connectivity
Before telling users the proxy is working, verify from the server that external access is possible:

```bash
# Get current public IP
curl -s ifconfig.me

# Test from external perspective (if you have a remote test endpoint)
# Or verify security group rules are correct (see below)
```

---

## Pitfalls

- **`allow-lan: false`** — default in many configs; blocks remote mobile access, only local apps work
- **Snap binary path in systemd** — must use `/usr/bin/snap run <pkg>.<binary>`, never `/snap/bin/...` symlink directly
- **AWS security group** — ports 7890/8388 (TCP) for Mihomo, 51820 (UDP) for WireGuard; must be open to 0.0.0.0/0
- **Mihomo DNS port 53** — if another DNS server (e.g. systemd-resolved) is using port 53, Mihomo DNS will fail to start; either stop the competing service or disable Mihomo's built-in DNS
- **External-controller port 9090** — binds to 127.0.0.1 only by default; if you want external access change to `0.0.0.0:9090` (and restrict via firewall)
- **Shadowsocks protocol** — ss-server is NOT a SOCKS5 proxy; curl directly to :8388 will time out; Mihomo (and only Mihomo) understands the SS protocol and can chain to it
- **WireGuard keygen directory** — always redirect key output to `/etc/wireguard/` explicitly; agent's cwd may not be writable or persistent
- **纯血鸿蒙 NEXT + HTTP proxy** — HarmonyOS NEXT does not route app traffic through system HTTP proxy settings; apps (YouTube, Chrome) bypass the proxy entirely; use WireGuard VPN instead
- **YouTube App QUIC/HTTP3** — YouTube app hardcodes QUIC transport, bypassing system proxy even on Android; use browser for testing, or switch to WireGuard VPN
- **GitHub curl download fails (exit 23)** — `curl -L` with process substitution or large files to /tmp fail in constrained envs; use `wget -O ~/` instead
- **/tmp is tmpfs with limited space** — `wget` to `/tmp` fails with "Disk quota exceeded"; always download to `$HOME` first
- **Certificate permission denied (xray exit 23)** — xray runs as `nobody`; can't read `/etc/letsencrypt/` archive files or private key symlinks; copy certs to `/etc/xray/certs/` with 644 permissions before starting xray
- **Port 443 not binding after xray restart** — usually caused by cert permission issue; check `journalctl -u xray` for "permission denied" on cert files
- **`xtls-rprx-vision` incompatibility on 鸿蒙/v2rayNG** — some v2rayNG builds on HarmonyOS reject VLESS with `xtls-rprx-vision` flow (connection hangs or immediate close). Workaround: remove `flow=xtls-rprx-vision` from server config entirely (set client UUID with no flow field), then use plain TLS TCP on the client. When in doubt, generate SS URI first as primary recommendation.
- **`client flow is empty" rejection** — xray server log shows `account <UUID> is rejected since the client flow is empty` and `proxy/vless/inbound: connection ends`. This means the server config has `flow` set (forcing a specific flow type) but the client is not sending one. Fix: remove `"flow"` from the server config's `clients` array entirely, restart xray. The server must NOT mandate a flow if clients are using plain TLS TCP.
- **SNI/域名 mismatch with Let's Encrypt certificates** — When using `sslip.io` or similar free DNS for cert issuance, the certificate CN/SAN is the domain (e.g. `vless-<IP>.nip.io`), not the raw IP. If the client connects to the IP directly without setting the SNI/域名 field to match the certificate's CN, TLS verification fails silently. Always set the SNI field in the VLESS client config to the certificate's domain, even when connecting via IP.
- **VLESS URI alpn encoding** — v2rayNG clipboard import parses ALPN correctly when written as `h2,http/1.1` (no URL encoding needed); URL-encoding commas (`%2c`) can cause parse failures on some builds.

## AWS EC2 Instance Metadata (IMDS)

To query instance metadata from within an EC2 instance:
```bash
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token \
  -H 'X-aws-ec2-metadata-token-ttl-seconds: 300')
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

IMDS (metadata service) and AWS API access (IAM roles) are independent systems — IMDS works without IAM, but `aws ec2 describe-*` commands need IAM credentials.

- **Server public IP changes on stop/start** — After stopping and starting an EC2 instance, AWS assigns a new public IP. Users must update their mobile/client proxy configuration with the new IP. There is no way to recover the old IP. Always check `curl -s ifconfig.me` to confirm the current IP before telling users their configuration is wrong.

## Reference Configurations

> ⚠️ **IMPORTANT:** The `references/` directory in this repo contains sanitized example configs only. All credential values (passwords, private keys, UUIDs, tokens) are replaced with `<PLACEHOLDER>` or similar. Never commit live keys or passwords to any public repository.

Sanitized references:
- `references/aws-ubuntu26-mihomo-ss-config.md` — Example config for AWS Ubuntu 26.04 with Mihomo + Shadowsocks chaining (Option A).
- `references/aws-wireguard-deploy.md` — WireGuard VPN deployment on AWS Ubuntu 26.04, including server/client key pairs (example keys, not real).
- `references/xray-vless-aws-deploy.md` — Xray VLESS deployment on AWS Ubuntu with Let's Encrypt TLS.

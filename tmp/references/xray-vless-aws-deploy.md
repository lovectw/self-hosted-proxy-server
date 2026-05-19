# Xray VLESS + Shadowsocks on AWS Ubuntu (Example)

> ⚠️ This is a **sanitized example** — replace all `<PLACEHOLDER>` values with your actual credentials.

## Environment
- OS: Ubuntu 26.04 (AWS EC2)
- Server public IP: `<YOUR_SERVER_IP>`
- xray: VLESS+TLS on :443
- shadowsocks-libev (snap): aes-256-gcm on :8388

## VLESS Server Setup

### 1. Install Xray
```bash
wget -O ~/xray-install.sh "https://github.com/XTLS/Xray-install/raw/refs/heads/main/install-release.sh"
sudo bash ~/xray-install.sh
```

### 2. Get TLS Certificate (Let's Encrypt)
```bash
sudo apt-get install -y certbot
DOMAIN="vless-<YOUR_SERVER_IP>.sslip.io"
sudo certbot certonly --standalone --agree-tos --email your@email.com --non-interactive -d $DOMAIN
```

### 3. Copy Certificates to xray-readable location
```bash
sudo mkdir -p /etc/xray/certs
sudo cp /etc/letsencrypt/live/$DOMAIN/fullchain.pem /etc/xray/certs/
sudo cp /etc/letsencrypt/live/$DOMAIN/privkey.pem /etc/xray/certs/
sudo chmod 644 /etc/xray/certs/*
```

### 4. Generate VLESS UUID
```bash
cat /proc/sys/kernel/random/uuid
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
        "clients": [{"id": "<YOUR_UUID>"}],
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
```

## Verified Working Client URIs

### VLESS URI
```
vless://<YOUR_UUID>@<YOUR_DOMAIN>:443?encryption=none&security=tls&alpn=h2,http/1.1&type=tcp&headerType=none&sni=<YOUR_DOMAIN>#home-proxy
```

### Shadowsocks URI
```
ss://YWVzLTI1Ni1nY206PHlvdXJfcGFzc3dvcmQ+@<YOUR_SERVER_IP>:8388#home-ss
```
Encode your password with: `echo -n "your_password" | base64`

### v2rayNG Import Steps (鸿蒙)
1. `+` → **从剪贴板导入** → paste SS or VLESS URI
2. Or **手动输入[Shadowsocks]** → server: `<YOUR_SERVER_IP>`, port: `8388`, cipher: `aes-256-gcm`, password: `<YOUR_SS_PASSWORD>`
3. Save → select server → 启动

## Issues Encountered (Lessons Learned)

1. **`xtls-rprx-vision` hangs on 鸿蒙 v2rayNG** — remove `flow=xtls-rprx-vision` from URI; use plain TLS TCP instead.
2. **QR code not visible in CLI** — save QR codes as PNG files for transfer instead of displaying in terminal.
3. **ALPN comma encoding** — `h2,http/1.1` works fine in URI; don't URL-encode commas.
4. **`client flow is empty" rejection** — server config had `flow` field forcing a type; fix by removing `"flow"` from server config's `clients` array entirely.
5. **SNI mismatch** — set SNI to the certificate's domain in v2rayNG even when connecting via IP.
6. **Certificate permission denied (xray exit 23)** — xray runs as `nobody`; copy certs to `/etc/xray/certs/` with 644 permissions.

## Service Verification
```bash
curl -s ifconfig.me  # confirm server IP
ss -tlnp | grep -E ':443|:8388'  # both LISTEN
timeout 3 bash -c "echo > /dev/tcp/<YOUR_SERVER_IP>/443" && echo "OPEN"  # TCP reachability
```

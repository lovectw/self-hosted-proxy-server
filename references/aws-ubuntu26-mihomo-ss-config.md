# AWS Ubuntu 26.04 — Mihomo + Shadowsocks Example Config (Sanitized)

> ⚠️ This is a **sanitized example** — replace all `<PLACEHOLDER>` values with your actual credentials.

## Environment
- OS: Ubuntu 26.04 LTS (AWS EC2)
- Python: 3.14+
- Server public IP: `<YOUR_SERVER_IP>`

## Packages Installed
- `shadowsocks-libev` via snap (canonical)
- `mihomo` v1.18.10 via GitHub (MetaCubeX release)
- `awscli` via apt (for security group queries — requires IAM role on instance)

## Shadowsocks
- Installed via: `sudo snap install shadowsocks-libev`
- Service: `snap.shadowsocks-libev.ss-server`
- Actual binary: `/snap/shadowsocks-libev/<rev>/usr/bin/ss-server`
- Working systemd ExecStart: `/usr/bin/snap run shadowsocks-libev.ss-server`
- Port: 8388, protocol: aes-256-gcm
- Password: `<YOUR_SS_PASSWORD>`

## Mihomo
- Config dir: `/home/ubuntu/mihomo/`
- Binary: `/usr/local/bin/mihomo`
- Port: 7890 (mixed HTTP+SOCKS5)
- DNS: built-in fake-ip on port 53
- Controller: 127.0.0.1:9090 (local only)
- Chaining to local Shadowsocks as upstream proxy

## Working config.yaml proxies section
```yaml
proxies:
  - name: "Local-Shadowsocks"
    type: ss
    server: 127.0.0.1
    port: 8388
    cipher: aes-256-gcm
    password: "<YOUR_SS_PASSWORD>"
```

## systemd service files
- `/etc/systemd/system/shadowsocks.service`
- `/etc/systemd/system/mihomo.service`

## AWS Security Group
- Required inbound rules:
  - TCP 7890 from 0.0.0.0/0
  - TCP 8388 from 0.0.0.0/0

## Verification commands
```bash
# Check ports
ss -tlnp | grep -E '7890|8388'

# Test local
curl --proxy http://127.0.0.1:7890 https://www.google.com -o /dev/null -w "%{http_code}\n"

# Test external
curl --proxy http://<YOUR_SERVER_IP>:7890 https://www.google.com -o /dev/null -w "%{http_code}\n"
```

## Issues Encountered (Lessons Learned)
1. Python 3.14 — all pip shadowsocks versions fail (collections.MutableMapping removed in Python 3.14)
2. Snap ss-server binary — requires `snap run`, direct exec fails with exit 127 (missing libev.so.4)
3. systemd ExecStart with `/snap/bin/shadowsocks-libev.ss-server` → exit 203/EXEC
4. systemd ExecStart with `/usr/bin/snap run shadowsocks-libev.ss-server` → WORKS
5. AWS `aws ec2 describe-*` fails with "Unable to locate credentials" — instance has no IAM role
6. AWS IMDSv2 requires token even from within instance: `curl -X PUT .../latest/api/token`
7. Port 9090 (mihomo controller) already in use → mihomo still works on 7890, just skip 9090
8. DNS port 53 conflict with systemd-resolved on Ubuntu → mihomo DNS startup fails but proxy still works
9. Phone `NET_ERROR(ERR_EMPTY_RESPONSE, -324)` — server curl tests work (200), phone fails → QUIC/HTTP3 bypasses system proxy on mobile apps; DNS misconfiguration is also a common cause. Fix: set static DNS on Wi-Fi (8.8.8.8), use browser instead of app for testing.

# AWS Ubuntu 26.04 — WireGuard VPN Deployment (Example)

> ⚠️ This is a **sanitized example** — replace all key values and IPs with your actual ones.

## Environment
- OS: Ubuntu 26.04 LTS (AWS EC2)
- Server public IP: `<YOUR_SERVER_IP>`
- WireGuard version: 1.0.x (apt package)

## Deployment Overview
- WireGuard installed via `apt install wireguard wireguard-tools`
- Server config: `/etc/wireguard/wg0.conf`
- Client config: `/home/ubuntu/client.conf`
- Service running: `wg-quick@wg0` (enabled + active)
- AWS security group: UDP 51820 opened

## Key Pairs (EXAMPLE — generate your own)
```
Server:
  PrivateKey: <SERVER_PRIVATE_KEY>  # wg genkey on YOUR server
  PublicKey:  <SERVER_PUBLIC_KEY>   # wg pubkey from server_private.key

Client (for 鸿蒙 NEXT / iOS):
  PrivateKey: <CLIENT_PRIVATE_KEY>  # wg genkey on YOUR server
  PublicKey:  <CLIENT_PUBLIC_KEY>   # wg pubkey from client_private.key
```

## Cloud Security Group Rules
- Protocol: UDP, Port: 51820, Source: 0.0.0.0/0

## Server Config — /etc/wireguard/wg0.conf
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

## Client Config (for WireGuard App)
```ini
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <YOUR_SERVER_IP>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

## Key Generation Commands
```bash
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key
chmod 600 /etc/wireguard/*.key
```

## Enable IP Forwarding
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

## QR Code for Client Import
To generate QR code on server (requires qrencode):
```bash
sudo apt-get install -y qrencode
cat /etc/wireguard/client.conf | qrencode -t ansiutf8
```
User scans with WireGuard app on their phone.

## Verification Commands
```bash
# Check WireGuard status
sudo wg show wg0

# Check UDP port is listening
ss -ulnp | grep 51820

# Check IP forwarding is enabled
cat /proc/sys/net/ipv4/ip_forward
```

## Issues Encountered (Lessons Learned)

### Key pair generation in wrong directory
- Always `cd /etc/wireguard` before keygen, or redirect output explicitly:
  ```bash
  wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
  ```

### AWS CLI IAM Role requirement
- `aws ec2 describe-security-groups` requires IAM role on the EC2 instance
- Without IAM role: command returns "Unable to locate credentials"

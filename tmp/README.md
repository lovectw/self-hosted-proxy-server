# Self-Hosted Proxy Server

在 Linux 云服务器上部署代理服务，支持手机/移动设备远程访问。

## 三种方案

| 方案 | 协议 | 适用场景 | 鸿蒙 NEXT 兼容 |
|------|------|----------|--------------|
| **A** | Mihomo + Shadowsocks | Android (GMS) | ❌ |
| **B** | WireGuard VPN | 纯血鸿蒙 NEXT、iOS | ✅ |
| **C** | Xray + VLESS | 纯血鸿蒙 NEXT（无 WireGuard App） | ✅ |

## 快速选择

1. **Android 有 GMS** → 方案 A（HTTP/SOCKS5 代理）
2. **纯血鸿蒙 NEXT** → 先试方案 B（WireGuard），如果 AppGallery 找不到 WireGuard App → 方案 C（VLESS）
3. **iOS** → 方案 B（WireGuard）

## 主要内容

- `SKILL.md` — 完整部署文档
- `references/` — 各方案的参考配置（已脱敏）

## 安全提醒

- `references/` 目录下的配置文件均为**脱敏示例**，所有凭据值已替换为 `<PLACEHOLDER>`
- 切勿将真实密码、私钥、UUID 等提交到任何公开仓库

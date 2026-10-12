---
name: proxy-server-setup
description: >-
  在海外 VPS 上一键部署 Hysteria2 + VLESS-REALITY 双协议代理服务器，并生成兼容 Clash Verge / mihomo 的客户端配置文件。
  Use when 用户要求搭建梯子、部署代理、配置 VPN、搭建 Hysteria2、部署 VLESS、生成 Clash 配置、
  或提到海外服务器/VPS 代理部署、翻墙、科学上网等场景。
  涵盖完整流程：服务器环境探测 → 系统优化(BBR) → Hysteria2 部署 → Xray VLESS+REALITY 部署 → Clash 客户端配置生成。
  Do not use for 纯本地网络调试、与代理无关的 SSH 运维、或非 Clash 系客户端(如 Surge/Quantumult)的配置生成。
---

# 海外代理服务器部署

在海外 Linux VPS 上部署 **Hysteria2 (UDP/QUIC)** + **VLESS+REALITY (TCP/TLS)** + **SOCKS5 / HTTP / HTTPS** 多协议代理，
生成一份同时兼容桌面和手机的 Clash Verge / mihomo 配置文件。

## 技术选型理由

| 协议 | 传输层 | 优势 | 角色 |
|---|---|---|---|
| Hysteria2 | UDP (QUIC) | 内置 Brutal 拥塞控制，高延迟下吞吐远超 TCP；自签证书即可，无需域名 | **主力** |
| VLESS+REALITY | TCP (TLS 1.3) | 部分网络封锁 UDP，TCP 兜底；REALITY 无需域名/证书，伪装为正常 HTTPS 流量 | **备用** |
| SOCKS5 | TCP | 通用性最强，几乎所有客户端/工具都支持；支持 UDP relay | **通用兜底** |
| HTTP | TCP | 浏览器/curl 等原生支持，无需额外客户端 | **简易接入** |
| HTTPS | TCP (TLS) | HTTP 代理 + TLS 加密，抗明文嗅探 | **加密兜底** |

所有协议共用一个 Xray 进程（SOCKS5/HTTP/HTTPS 作为 Xray 额外 inbound），总内存约 40-70 MB。

## 前置条件

- 一台海外 Linux VPS（Ubuntu 20.04+ / Debian 11+ 推荐），有 root 权限
- 至少 5 个可用端口（1 UDP + 4 TCP），记录为 `PORT_HY2`、`PORT_VLESS`、`PORT_SOCKS5`、`PORT_HTTP`、`PORT_HTTPS`
- 本地可通过 SSH 连接到服务器

## 执行流程

按以下 6 步顺序执行。

### Step 1: 服务器环境探测

SSH 连接后收集基础信息，确认部署可行性：

```bash
uname -a && nproc && free -h && df -h /
ss -tulnp                         # 确认目标端口空闲
curl -sI https://www.google.com | head -1  # 确认出站可达
sysctl net.ipv4.tcp_congestion_control     # 查看当前拥塞控制
```

同时在本地测延迟：`ping -c 20 <SERVER_IP>`。延迟 >200ms 属跨洲链路，Hysteria2 优势尤其明显。

### Step 2: 开启 BBR

```bash
cat >> /etc/sysctl.conf << 'EOF'

# BBR congestion control
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
EOF
sysctl -p
```

验证：`sysctl net.ipv4.tcp_congestion_control` 输出 `bbr`，`lsmod | grep bbr` 有 `tcp_bbr`。

### Step 3: 部署 Hysteria2

详细配置和 systemd 单元文件 → [references/hysteria2.md](references/hysteria2.md)

操作摘要：

1. 从 GitHub 下载最新二进制（⚠️ 仓库已迁移至 `HyNetworks/hysteria`，见避坑 #4）
2. 用 `openssl` 生成自签 EC 证书
3. 生成随机密码 `openssl rand -hex 16`
4. 写入配置 → `/etc/hysteria/config.yaml`
5. 创建 systemd 服务并启动
6. `ss -ulnp | grep PORT_HY2` 验证监听

### Step 4: 部署 Xray（VLESS + SOCKS5 + HTTP + HTTPS）

详细配置和 systemd 单元文件 → [references/vless-reality.md](references/vless-reality.md)

Xray 通过多 inbound 同时提供 4 种协议，共用一个进程。操作摘要：

1. 从 GitHub 下载最新 Xray-core
2. `xray x25519` 生成 REALITY 密钥对（⚠️ 注意新版输出格式变更，见避坑 #5）
3. `xray uuid` 生成 UUID，`openssl rand -hex 8` 生成 Short ID
4. 生成 SOCKS5/HTTP/HTTPS 共用的认证账号：`openssl rand -hex 8` 作为密码
5. 写入多 inbound 配置 → `/etc/xray/config.json`（含 VLESS:PORT_VLESS + SOCKS5:PORT_SOCKS5 + HTTP:PORT_HTTP + HTTPS:PORT_HTTPS）
6. HTTPS inbound 复用 Hysteria2 的自签证书（`/etc/hysteria/server.crt`）
7. 创建 systemd 服务并启动
8. `ss -tlnp | grep -E "PORT_VLESS|PORT_SOCKS5|PORT_HTTP|PORT_HTTPS"` 验证全部监听

### Step 5: 生成 Clash Verge 配置

使用 → [assets/clash-template.yaml](assets/clash-template.yaml) 作为模板。

替换占位符：

| 占位符 | 来源 |
|---|---|
| `__SERVER_IP__` | 服务器公网 IP |
| `__HY2_PORT__` | Step 3 的 UDP 端口 |
| `__HY2_PASSWORD__` | Step 3 生成的密码 |
| `__VLESS_PORT__` | Step 4 的 VLESS TCP 端口 |
| `__VLESS_UUID__` | Step 4 生成的 UUID |
| `__REALITY_PUBLIC_KEY__` | Step 4 生成的公钥 |
| `__REALITY_SHORT_ID__` | Step 4 生成的 Short ID |
| `__SOCKS5_PORT__` | Step 4 的 SOCKS5 端口 |
| `__HTTP_PORT__` | Step 4 的 HTTP 端口 |
| `__HTTPS_PORT__` | Step 4 的 HTTPS 端口 |
| `__PROXY_USER__` | Step 4 生成的认证用户名 |
| `__PROXY_PASS__` | Step 4 生成的认证密码 |

如用户提供了额外的精确域名规则（如特定网站走代理），直接追加到 `rules:` 的代理区块内。
无需为每类域名创建独立代理组——只有一台服务器时，所有代理流量走同一个选择组即可。

### Step 6: 验证部署

```bash
systemctl is-active hysteria && systemctl is-active xray
ss -tulnp | grep -E "PORT_HY2|PORT_VLESS"
curl -sI https://www.google.com | head -1
curl -sI https://www.youtube.com | head -1
free -m && ps aux --sort=-rss | head -5
systemctl is-enabled hysteria && systemctl is-enabled xray
```

## ⚠️ 关键避坑指南

### 1. DNS 是所有平台能否正常使用的核心问题

**现象**：国内网站打不开，日志中出现 `dns resolve failed: couldn't find ip`。手机 TUN 模式和 Windows 桌面均可能触发。

**根因**：
- 配置中混入海外 DNS → TUN 模式下 DNS 请求自身需要代理 → 死循环
- `nameserver` / `direct-nameserver` 仅配 UDP 明文 DNS → 被 ISP 劫持或丢包 → 解析失败
- 使用 `fallback` + `fallback-filter` 的旧方案 → 双查询开销大，geoip 判定有误差

**正确做法**（对齐 mihomo 社区主流，已在模板中实现）：

```
架构：nameserver-policy 前置分流，不使用 fallback
```

- `nameserver`：仅国内 DoH（`doh.pub`、`dns.alidns.com`），不含任何海外 DNS，不含 UDP 明文
- `nameserver-policy`：`"geosite:cn,private"` → 国内 DoH，让 CN 域名跳过默认流程直接查国内 DNS
- `proxy-server-nameserver`：国内 DoH，解析代理服务器自身域名，避免鸡生蛋问题
- `direct-nameserver`：国内 DoH，为 DIRECT 规则命中后的域名回查真实 IP
- `default-nameserver`：纯 IP（`223.5.5.5`、`119.29.29.29`），仅用于 bootstrap 解析 DoH 域名
- **不配 `fallback` 和 `fallback-filter`**：现代方案用 `nameserver-policy` 前置分流取代 fallback 后置过滤
- **全链路 DoH**：`nameserver` / `direct-nameserver` / `proxy-server-nameserver` 全部使用 HTTPS，抗 UDP 劫持

原理：fake-ip 模式下代理域名由远端服务器解析，本地 DNS 结果无意义。本地 DNS 唯一职责是为直连域名返回正确 IP → 全用国内 DoH 即可。

### 2. fake-ip-filter 必须覆盖系统探测域名

**现象**：手机状态栏显示无网络，部分 App 拒绝请求。

**根因**：系统网络探测域名返回 fake-ip 后判定无网。

**必须包含**（已在模板中完整列出）：
- Android: `+.connectivitycheck.gstatic.com`, `+.connectivitycheck.android.com`
- iOS: `+.captive.apple.com`, `+.gsp-ssl.ls.apple.com`
- Windows: `+.msftconnecttest.com`, `+.msftncsi.com`
- 国内音乐/游戏/运营商/推送 SDK 域名

### 3. fake-ip-range 使用 28.0.0.1/8

默认 `198.18.0.1/16` 仅 ~65K 地址，手机 App 多连接场景容易耗尽。
`28.0.0.1/8` 提供 ~16M 地址池（DoD 未路由段，不会与真实网络冲突）。

### 4. Hysteria2 GitHub 仓库已迁移

从 `apernet/hysteria` → `HyNetworks/hysteria`。旧地址 301 跳转，脚本解析版本号时需 follow redirect 或直接使用新地址：

```
https://github.com/HyNetworks/hysteria/releases/latest
```

### 5. Xray x25519 输出格式变更

v26+ 输出：

```
PrivateKey: xxxx
Password (PublicKey): xxxx
Hash32: xxxx
```

旧版为 `Private key: / Public key:`。解析时匹配 `PrivateKey:` 和 `PublicKey`。

### 6. SSH 非交互密码登录

macOS 上 `expect` 无法可靠捕获 SSH 密码提示符（提示走 TTY 而非 pipe）。用 `SSH_ASKPASS` 方式：

```bash
export SSH_ASKPASS_REQUIRE=force
export SSH_ASKPASS=$(mktemp)
chmod +x "$SSH_ASKPASS"
echo '#!/bin/bash' > "$SSH_ASKPASS"
echo 'echo "PASSWORD"' >> "$SSH_ASKPASS"
ssh -p PORT -o StrictHostKeyChecking=no \
    -o PreferredAuthentications=keyboard-interactive,password \
    -o NumberOfPasswordPrompts=1 user@host 'COMMAND'
```

### 7. VLESS 非 443 端口警告

Xray 启动时警告 `Listening on non-443 ports may get your IP blocked by the GFW`，功能不受影响，可忽略。

### 8. 部分网站 IP 封禁

ChatGPT 等服务封禁 VPS IP 段（返回 403），属 IP 信誉问题与协议无关。可套 Cloudflare WARP 出口或换 IP。

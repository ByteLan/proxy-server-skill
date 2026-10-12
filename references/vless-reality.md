# Xray 多协议部署详细步骤

Xray 通过多 inbound 同时提供 VLESS+REALITY、SOCKS5、HTTP、HTTPS 四种协议，共用一个进程。

## 1. 下载 Xray-core

```bash
# 获取最新版本
XRAY_VER=$(curl -sIL "https://github.com/XTLS/Xray-core/releases/latest" \
  | grep -i "^location:" | tail -1 | sed "s|.*/tag/||" | tr -d "\r\n ")

echo "版本: $XRAY_VER"  # 格式如 v26.3.27

# 下载并解压
cd /tmp
curl -L -o xray.zip \
  "https://github.com/XTLS/Xray-core/releases/download/${XRAY_VER}/Xray-linux-64.zip"
apt-get install -y unzip > /dev/null 2>&1 || true
mkdir -p /usr/local/xray
unzip -o xray.zip -d /usr/local/xray
chmod +x /usr/local/xray/xray
rm -f xray.zip

# 验证
/usr/local/xray/xray version
```

> ARM64 服务器使用 `Xray-linux-arm64-v8a.zip`。

## 2. 生成 REALITY 密钥对

```bash
/usr/local/xray/xray x25519
```

⚠️ **v26+ 输出格式**：

```
PrivateKey: mMQly3rLNBNf93se93ZQjBRWkDMRHh81S4ikFU_H23o
Password (PublicKey): oqymHs0PvrOi1jEU1bB1vXxe7tcpdYpOGoqy9rj2yTI
Hash32: TbD1FXEWJ6AxOG3FlVDbRKJC6CQLALiKWdrQzp84UuI
```

提取方式：

```bash
KEYS=$(/usr/local/xray/xray x25519)
PRIVATE_KEY=$(echo "$KEYS" | head -1 | awk '{print $NF}')
PUBLIC_KEY=$(echo "$KEYS" | sed -n '2p' | awk '{print $NF}')
```

- `PRIVATE_KEY` → 写入服务端配置
- `PUBLIC_KEY` → 写入客户端配置

## 3. 生成 UUID、Short ID、代理认证

```bash
UUID=$(/usr/local/xray/xray uuid)
SHORT_ID=$(openssl rand -hex 8)
PROXY_USER="proxy"
PROXY_PASS=$(openssl rand -hex 8)
echo "UUID: $UUID"
echo "Short ID: $SHORT_ID"
echo "Proxy Auth: $PROXY_USER / $PROXY_PASS"
```

## 4. 写入多 inbound 配置

SOCKS5/HTTP/HTTPS 共用同一套用户名密码。HTTPS inbound 复用 Hysteria2 的自签证书。

```bash
mkdir -p /etc/xray

cat > /etc/xray/config.json << EOF
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "tag": "vless-reality",
      "listen": "0.0.0.0",
      "port": __VLESS_PORT__,
      "protocol": "vless",
      "settings": {
        "clients": [{ "id": "${UUID}", "flow": "xtls-rprx-vision" }],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.microsoft.com:443",
          "xver": 0,
          "serverNames": ["www.microsoft.com", "microsoft.com"],
          "privateKey": "${PRIVATE_KEY}",
          "shortIds": ["${SHORT_ID}", ""]
        }
      },
      "sniffing": { "enabled": true, "destOverride": ["http","tls","quic"] }
    },
    {
      "tag": "socks5",
      "listen": "0.0.0.0",
      "port": __SOCKS5_PORT__,
      "protocol": "socks",
      "settings": {
        "auth": "password",
        "accounts": [{ "user": "${PROXY_USER}", "pass": "${PROXY_PASS}" }],
        "udp": true
      },
      "sniffing": { "enabled": true, "destOverride": ["http","tls","quic"] }
    },
    {
      "tag": "http-proxy",
      "listen": "0.0.0.0",
      "port": __HTTP_PORT__,
      "protocol": "http",
      "settings": {
        "accounts": [{ "user": "${PROXY_USER}", "pass": "${PROXY_PASS}" }]
      },
      "sniffing": { "enabled": true, "destOverride": ["http","tls","quic"] }
    },
    {
      "tag": "https-proxy",
      "listen": "0.0.0.0",
      "port": __HTTPS_PORT__,
      "protocol": "http",
      "settings": {
        "accounts": [{ "user": "${PROXY_USER}", "pass": "${PROXY_PASS}" }]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "tls",
        "tlsSettings": {
          "certificates": [{
            "certificateFile": "/etc/hysteria/server.crt",
            "keyFile": "/etc/hysteria/server.key"
          }]
        }
      },
      "sniffing": { "enabled": true, "destOverride": ["http","tls","quic"] }
    }
  ],
  "outbounds": [
    { "protocol": "freedom", "tag": "direct" },
    { "protocol": "blackhole", "tag": "block" }
  ]
}
EOF
```

**各 inbound 说明：**

| inbound | 协议 | 端口 | 认证 | 特点 |
|---|---|---|---|---|
| vless-reality | VLESS | `__VLESS_PORT__` | UUID + REALITY 密钥 | 抗 DPI，伪装正常 HTTPS |
| socks5 | SOCKS5 | `__SOCKS5_PORT__` | 用户名/密码 | 通用性强，支持 UDP relay |
| http-proxy | HTTP | `__HTTP_PORT__` | 用户名/密码 | 浏览器/curl 原生支持 |
| https-proxy | HTTP over TLS | `__HTTPS_PORT__` | 用户名/密码 | HTTP 代理 + TLS 加密层 |

**dest 推荐站点**（TLS 1.3 + H2，全球可达）：
- `www.microsoft.com:443`（推荐）
- `www.apple.com:443`
- `www.amazon.com:443`

## 5. 创建 systemd 服务

```bash
cat > /etc/systemd/system/xray.service << 'EOF'
[Unit]
Description=Xray Multi-Protocol Proxy Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/xray/xray run -c /etc/xray/config.json
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable xray
systemctl start xray
```

## 6. 验证

```bash
systemctl status xray --no-pager

# 确认所有端口监听
ss -tlnp | grep -E "__VLESS_PORT__|__SOCKS5_PORT__|__HTTP_PORT__|__HTTPS_PORT__"
```

## 运维命令

```bash
systemctl restart xray
systemctl stop xray
journalctl -u xray -f
journalctl -u xray --since "1 hour ago"
```

## 保存凭据

```bash
cat > /etc/xray/.credentials << EOF
UUID=${UUID}
PUBLIC_KEY=${PUBLIC_KEY}
PRIVATE_KEY=${PRIVATE_KEY}
SHORT_ID=${SHORT_ID}
SOCKS5_PORT=__SOCKS5_PORT__
HTTP_PORT=__HTTP_PORT__
HTTPS_PORT=__HTTPS_PORT__
PROXY_USER=${PROXY_USER}
PROXY_PASS=${PROXY_PASS}
EOF
chmod 600 /etc/xray/.credentials
```

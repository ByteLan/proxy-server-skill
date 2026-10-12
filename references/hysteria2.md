# Hysteria2 部署详细步骤

## 1. 下载二进制

```bash
# 获取最新版本号（⚠️ 仓库已迁移至 HyNetworks）
HY_VER=$(curl -sIL "https://github.com/HyNetworks/hysteria/releases/latest" \
  | grep -i "^location:" | tail -1 | sed "s|.*/tag/||" | tr -d "\r\n ")

echo "版本: $HY_VER"  # 格式如 app/v2.13.0

# 下载（根据架构选择 amd64 或 arm64）
ARCH=$(uname -m)
[[ "$ARCH" == "x86_64" ]] && HY_ARCH="amd64" || HY_ARCH="arm64"

curl -L -o /usr/local/bin/hysteria \
  "https://github.com/HyNetworks/hysteria/releases/download/${HY_VER}/hysteria-linux-${HY_ARCH}"
chmod +x /usr/local/bin/hysteria

# 验证
/usr/local/bin/hysteria version
```

## 2. 生成自签证书

```bash
mkdir -p /etc/hysteria

openssl ecparam -genkey -name prime256v1 -out /etc/hysteria/server.key
openssl req -new -x509 -days 3650 -key /etc/hysteria/server.key \
  -out /etc/hysteria/server.crt -subj "/CN=bing.com"
```

> CN 设为 `bing.com` 是为了伪装，客户端配置中需对应设置 `sni: bing.com` + `skip-cert-verify: true`。

## 3. 生成认证密码

```bash
HY_PASS=$(openssl rand -hex 16)
echo "$HY_PASS" > /etc/hysteria/.auth_pass
echo "密码: $HY_PASS"
```

## 4. 写入配置

```bash
cat > /etc/hysteria/config.yaml << EOF
listen: :__HY2_PORT__

tls:
  cert: /etc/hysteria/server.crt
  key: /etc/hysteria/server.key

auth:
  type: password
  password: ${HY_PASS}

masquerade:
  type: proxy
  proxy:
    url: https://bing.com
    rewriteHost: true

quic:
  initStreamReceiveWindow: 8388608
  maxStreamReceiveWindow: 8388608
  initConnReceiveWindow: 20971520
  maxConnReceiveWindow: 20971520

bandwidth:
  up: 50 mbps
  down: 50 mbps
EOF
```

**配置说明：**

| 字段 | 说明 |
|---|---|
| `listen` | UDP 监听端口，替换 `__HY2_PORT__` |
| `masquerade` | 非认证客户端访问时反代到 bing.com，增强伪装 |
| `quic.*Window` | QUIC 窗口大小，适合高延迟链路 |
| `bandwidth` | 告知客户端服务器带宽上限，Brutal 拥塞控制据此调整发送速率；按服务器实际带宽填写 |

## 5. 创建 systemd 服务

```bash
cat > /etc/systemd/system/hysteria.service << 'EOF'
[Unit]
Description=Hysteria2 Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/hysteria server -c /etc/hysteria/config.yaml
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable hysteria
systemctl start hysteria
```

## 6. 验证

```bash
# 服务状态
systemctl status hysteria --no-pager

# 确认 UDP 端口监听
ss -ulnp | grep __HY2_PORT__
```

日志应包含 `server up and running {"listen": ":__HY2_PORT__"}`。

## 运维命令

```bash
systemctl restart hysteria      # 重启
systemctl stop hysteria         # 停止
journalctl -u hysteria -f       # 实时日志
journalctl -u hysteria --since "1 hour ago"  # 最近 1 小时日志
```

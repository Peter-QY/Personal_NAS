目标：公网可访问。

---

## 域名托管

Cloudflare → Add a site → 填域名 → Free 计划。

获得两条 NS，形如 `dina.ns.cloudflare.com`。

到域名注册商修改 NS。阿里云：域名 → DNS 修改。腾讯云：DNS 解析 → 修改 DNS 服务器。

```bash
dig +short NS 你的域名
```

返回两条 `*.ns.cloudflare.com` 即完成。

生效时间十分钟到数小时。新注册域名需等待实名审核同步。

## 安装 cloudflared

```bash
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared
cloudflared --version
```

已安装则跳过。apt 方式安装可随 `apt upgrade` 更新。

## 创建隧道

Zero Trust → Networks → Tunnels → Create a tunnel。

首次进入需开通 Zero Trust，选 Free 计划，不需要付费。

类型选 Cloudflared，命名 `raspberrypi-nas`。

创建后页面显示安装命令，其中长字符串为 **Tunnel Token**，点击 Copy to clipboard 复制。

## 写入 Token

```bash
sudo install -d -m 700 -o root -g root /etc/cloudflared
read -r -s -p "粘贴 Tunnel Token: " TOK; echo
printf '%s' "$TOK" | sudo tee /etc/cloudflared/token >/dev/null
sudo chmod 600 /etc/cloudflared/token
unset TOK
```

不使用命令行参数传递 Token，避免出现在 `ps` 输出和 shell 历史。

## 开机自启

```bash
sudo tee /etc/systemd/system/cloudflared.service >/dev/null <<'EOF'
[Unit]
Description=Cloudflare Tunnel
After=network-online.target
Wants=network-online.target
ConditionPathExists=/etc/cloudflared/token

[Service]
Type=notify
TimeoutStartSec=0
ExecStart=/usr/bin/cloudflared --no-autoupdate --metrics 127.0.0.1:20241 tunnel run --token-file /etc/cloudflared/token
Restart=on-failure
RestartSec=5s
NoNewPrivileges=true
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now cloudflared
```

`ConditionPathExists`：Token 未配置时跳过启动，不产生重启循环。

`--token-file`：Token 从文件读取，不出现在进程列表。

## 验证连接

```bash
systemctl is-active cloudflared
curl -s http://127.0.0.1:20241/ready
journalctl -u cloudflared -n 30 -o cat | grep -i registered
```

`ready` 返回 `{"status":200,"readyConnections":4,...}`。4 条连接为正常值。

日志显示落地机房：

```
Registered tunnel connection ... location=hkg13 protocol=quic
```

## 添加公网域名

Tunnel 详情页 → Public Hostname → Add：

| | |
|---|---|
| Subdomain | `nas` |
| Domain | 你的域名 |
| Type | **HTTP** |
| URL | `127.0.0.1:8080` |

Type 必须选 HTTP。TLS 在 Cloudflare 边缘终止，边缘到源站走 Tunnel 已加密。选 HTTPS 会导致边缘用 TLS 连接明文端口，返回 502。

## 加密模式

域名 → SSL/TLS → Overview → **Full**。

不要选 Flexible，HTTP 回源会与 Nextcloud 重定向逻辑冲突。

## 证书签发

域名刚完成委派时，Edge Certificates 显示 `Pending Validation (TXT)`。

验证期间访问报 TLS 握手失败。签发完成后自动恢复，通常 5–30 分钟。

## 追加域名到 Nextcloud

```bash
OCC="docker exec -u www-data nextcloud-app-1 php occ"
$OCC config:system:set trusted_domains 5 --value="nas.你的域名"
$OCC config:system:set overwrite.cli.url --value="https://nas.你的域名"
```

序号接在已有条目之后。

## 检查点

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://nas.你的域名/status.php
curl -s -o /dev/null -w '%{http_code}\n' -L https://nas.你的域名/login
curl -I https://nas.你的域名/ | grep -i location
echo | openssl s_client -connect nas.你的域名:443 -servername nas.你的域名 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

前三项分别返回 200 / 200 / `location: https://...`。

证书签发者为 Let's Encrypt，主体为你的域名，有效期 90 天。

手机关闭 WiFi 用移动数据打开 `https://nas.你的域名` 验证。

---


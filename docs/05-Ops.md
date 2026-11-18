目标：三条入口可用，重启后自动恢复。

---

## 三条入口

| | 地址 | 场景 |
|---|---|---|
| 公网 | `https://nas.你的域名` | 任意网络 |
| 局域网 | `http://192.168.1.105:8081` | 同一 WiFi，速度最快 |
| Tailscale | `https://<host>.<tailnet>.ts.net` | Cloudflare 故障时备用 |

日常使用公网地址。传大文件时切局域网。

## Tailscale（可选）

作为 Cloudflare 之外的备用通道。

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
sudo tailscale serve --bg 8080
tailscale serve status
```

生成的域名需加入 `trusted_domains`。

## 自动守护

每 2 分钟检查挂载、服务、容器状态。

```bash
sudo tee /usr/local/sbin/nas-guard >/dev/null <<'EOF'
#!/usr/bin/env bash
set -u
log() { logger -t nas-guard "$*"; }

UUID=$(awk '$2=="/srv/nas"{print $1}' /etc/fstab | sed 's/UUID=//')

if ! mountpoint -q /srv/nas; then
  log "CRITICAL /srv/nas 未挂载"
  systemctl is-active --quiet docker && { log "停 docker 防止写根分区"; systemctl stop docker; }
  if [ -n "$UUID" ] && [ -e /dev/disk/by-uuid/"$UUID" ]; then
    log "设备就绪，尝试挂载"
    mount /srv/nas && systemctl start docker
  fi
  exit 0
fi

[ -w /srv/nas ] || log "WARN /srv/nas 不可写，可能转只读"

systemctl is-active --quiet cloudflared || { log "重启 cloudflared"; systemctl restart cloudflared; }
systemctl is-active --quiet docker      || systemctl start docker

if docker info >/dev/null 2>&1 && ! docker ps --format '{{.Names}}' | grep -q '^nextcloud-app-1$'; then
  log "容器缺失，按 compose 拉起"
  ( cd /srv/nas/nextcloud && docker compose up -d >/dev/null 2>&1 )
fi
EOF

sudo chmod 755 /usr/local/sbin/nas-guard

sudo tee /etc/systemd/system/nas-guard.service >/dev/null <<'EOF'
[Unit]
Description=NAS guard
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/nas-guard
TimeoutStartSec=120
EOF

sudo tee /etc/systemd/system/nas-guard.timer >/dev/null <<'EOF'
[Unit]
Description=Run nas-guard every 2 minutes

[Timer]
OnBootSec=90
OnUnitActiveSec=120
Unit=nas-guard.service

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now nas-guard.timer
```

日志：

```bash
journalctl -t nas-guard -f
```

正常运行时无输出。

设备识别使用 UUID，不使用 `/dev/sda`，避免掉盘重连后名称变化导致判断失效。

## 重启测试

```bash
sudo reboot
```

等待 2 分钟后执行：

```bash
df -hT /srv/nas
docker ps
curl -s http://127.0.0.1:20241/ready
curl -s -o /dev/null -w '%{http_code}\n' https://nas.你的域名/status.php
systemctl is-active nas-guard.timer
```

五项全部正常即完成部署。

---

## 日常操作

**状态检查**

```bash
docker ps
systemctl is-active docker cloudflared
curl -s -o /dev/null -w '%{http_code}\n' https://nas.你的域名/status.php
df -hT /srv/nas
```

**重启**

```bash
cd /srv/nas/nextcloud && docker compose restart
cd /srv/nas/nextcloud && docker compose restart app
sudo systemctl restart cloudflared
```

**备份配置**

```bash
TS=$(date +%Y%m%d-%H%M%S); B=/var/backups/yiqiu-nas/$TS
sudo install -d -m 700 "$B"
sudo cp -a /srv/nas/nextcloud/app/config/config.php "$B/"
sudo cp -a /srv/nas/nextcloud/compose.yaml "$B/"
sudo cp -a /srv/nas/nextcloud/.env "$B/"
sudo ln -sfn "$B" /var/backups/yiqiu-nas/latest
```

回滚：拷回 `config.php`，`chown www-data`，重启容器。

仅备份配置，用户数据不包含在内，需另行规划。

**日志**

```bash
docker logs -f --tail 100 nextcloud-app-1
sudo journalctl -u cloudflared -f
sudo journalctl -t nas-guard -f
docker exec nextcloud-app-1 tail -100 /var/www/html/data/nextcloud.log
```

**升级**

```bash
cd /srv/nas/nextcloud && docker compose pull && docker compose up -d
docker exec -u www-data nextcloud-app-1 php occ upgrade
sudo apt update && sudo apt install --only-upgrade cloudflared
```

**体检**

```bash
docker exec -u www-data nextcloud-app-1 php occ setupchecks
```

---

## 故障处理

**硬盘掉线**

USB 转 NVMe 在持续写入下会断链路。三种表现：

- 设备消失，`dmesg` 出现 `device not accepting address, error -71`
- 文件系统只读，`/proc/mounts` 出现 `emergency_ro`
- 设备重枚举为 `sdb`，容器全部 Exited

```bash
mountpoint /srv/nas && echo 已挂载 || echo 未挂载
grep " /srv/nas " /proc/mounts
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT

sudo reboot
cd /srv/nas/nextcloud && docker compose up -d
```

前两种执行重启，开机 fsck 修复。第三种盘仍挂载，只需拉起容器。

根本原因是硬件。硬盘盒独立供电最有效，其次更换 JMS583 / ASM2362 芯片方案。

**公网 502**

Public Hostname 的 Type 为 HTTPS。改为 HTTP。

**公网 400**

域名未加入 `trusted_domains`。

```bash
docker exec -u www-data nextcloud-app-1 php occ config:system:get trusted_domains
```

**上传 413**

Cloudflare 单请求 100MB 限制。网页上传无法绕过，大文件使用客户端分块上传。

**手机传视频 400**

客户端分块 100MB 触及上限被截断。确认 `files.chunked_upload.max_size` 为 52428800，然后彻底关闭 App 重启（从最近任务划掉）。

**Mac 客户端修改地址后回退**

客户端运行时用内存值覆写配置文件：

```bash
pkill -TERM -x Nextcloud; sleep 4
pgrep -x Nextcloud && pkill -9 -x Nextcloud
# 编辑配置
open -a Nextcloud
```

配置路径：`~/Library/Containers/com.nextcloud.desktopclient/Data/Library/Preferences/Nextcloud/nextcloud.cfg`

**cloudflared 不启动**

`systemctl status cloudflared` 显示 `ConditionPathExists=/etc/cloudflared/token was not met`，Token 未写入，重新执行第 04 步。

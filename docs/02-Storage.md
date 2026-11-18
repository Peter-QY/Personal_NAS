目标：数据盘挂载成功，Docker 可用且受挂载保护。

---

## 识别设备

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT
```

数据盘通常为 `/dev/sda`。`mmcblk0` 是系统盘，不要操作。

## 格式化

```bash
sudo mkfs.ext4 -L NAS /dev/sda2
```

使用 ext4。exFAT 缺少 Unix 权限和日志，Docker 与 Nextcloud 会产生权限错误，断电后易损坏。

格式化会清空数据。

## 挂载

```bash
sudo mkdir -p /srv/nas
sudo blkid /dev/sda2
```

记录 UUID，写入 fstab：

```bash
echo "UUID=你的UUID /srv/nas ext4 defaults,noatime,nofail,x-systemd.device-timeout=10 0 2" | sudo tee -a /etc/fstab
sudo mount -a
df -hT /srv/nas
```

参数说明：

- UUID 而非 `/dev/sda2`：USB 盘掉线重连后设备名会变
- `nofail`：盘未挂载时系统仍可启动，否则进入 emergency shell，SSH 无法连接
- `noatime`：减少写入

## 安装 Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```

## 挂载保护

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/10-nas.conf >/dev/null <<'EOF'
[Unit]
RequiresMountsFor=/srv/nas
EOF
sudo systemctl daemon-reload
```

不添加此配置的后果：数据盘未挂载时 Docker 会在根分区创建空数据目录，Nextcloud 视为全新安装并初始化空数据库，管理界面显示为空网盘。

添加后，`/srv/nas` 未挂载时 Docker 拒绝启动。

## 数据目录迁移

```bash
sudo mkdir -p /srv/nas/docker
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "data-root": "/srv/nas/docker"
}
EOF
sudo systemctl restart docker
docker info | grep -i "root dir"
```

## 检查点

```bash
df -hT /srv/nas
docker info | grep -i "root dir"
```

验证保护生效：

```bash
sudo systemctl stop docker
sudo umount /srv/nas
sudo systemctl start docker && echo "保护未生效" || echo "拒绝启动，符合预期"
sudo mount -a && sudo systemctl start docker
```

---



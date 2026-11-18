目标：树莓派可 SSH 登录。

---

## 烧录系统

Raspberry Pi Imager 选 **Raspberry Pi OS Lite (64-bit)**。

Imager 的齿轮设置里配置：

- 主机名
- SSH 公钥认证
- 用户名密码
- WiFi SSID 与密码
- 时区 Asia/Shanghai

公钥用 `~/.ssh/id_ed25519.pub` 的内容，没有则先 `ssh-keygen -t ed25519` 生成。

插卡上电，等待 1–2 分钟。

## 登录

```bash
ssh pi@raspberrypi.local
```

mDNS 不通时，从路由器后台查 IP，直接 `ssh pi@192.168.1.x`。

修改默认密码：

```bash
passwd
```

## 更新

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

内核更新需重启生效。

## 常用工具

```bash
sudo apt install -y curl vim htop git rsync
```

## 网络

板载 WiFi 在 2.4G 频段较慢，优先连 5G。

USB 3.0 与 2.4G WiFi 存在干扰，硬盘盒与无线网卡同时工作会降速。

有线连接更稳定，千兆口局域网可达 110 MB/s。

## 检查点

```bash
ssh pi@raspberrypi.local 'hostname; uname -m; cat /etc/debian_version'
```

输出包含主机名、`aarch64`、`13.x`。

---

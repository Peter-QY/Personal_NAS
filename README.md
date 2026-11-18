# 树莓派私人 NAS

树莓派接一块 2TB NVMe，通过 Cloudflare Tunnel 对外提供 Nextcloud 服务。

不需要公网 IP，不需要端口映射，不需要 VPS。

---

## 硬件

|     | 型号                              | 价格      |
| --- | ------------------------------- | ------- |
| 主机  | 树莓派 4B 4GB                      | 400     |
| 电源  | 官方 5.1V 3A                      | 60      |
| 系统盘 | 32GB microSD (A2)               | 30      |
| 数据盘 | 2TB NVMe                        | 600     |
| 硬盘盒 | USB3 转 NVMe，JMS583 / ASM2362 芯片 | 150–300 |
| 散热  | 金属外壳带风扇（暂未购入）                   | 50      |


---

## 架构

```
手机 / 电脑 / 浏览器
        │ HTTPS
        ▼
   Cloudflare 边缘        TLS 在此终止
        │ Tunnel
        ▼
   树莓派 cloudflared
        │ http://127.0.0.1:8080
        ▼
   Nextcloud（Docker）
        │
        ▼
   /srv/nas  2TB NVMe
```

三条入口：

|           |                                      |
| --------- | ------------------------------------ |
| 公网        | `https://nas.example.com`            |
| 局域网       | `http://192.168.1.105:8081`          |
| Tailscale | `https://<host>.<tailnet>.ts.net`，备选 |

---

## 功能范围

**可用**

- 手机照片自动上传（WiFi / 移动数据均可）
- 电脑文件夹双向同步
- 浏览器访问，上传下载
- 重启后全自动恢复

**边界**

- 网页上传单文件上限 100MB，Cloudflare 免费版限制。大文件用客户端分块传
- 公网下载 1.5–1.8 MB/s，受宽带上行限制
- 国内访问首字节 0.3–2.5 秒，边缘节点在境外
- 单盘无冗余，盘坏数据丢失

**实测性能**

| | |
|---|---|
| 磁盘顺序读写 | 204 / 285 MB/s |
| 局域网 | 71–78 Mbps（WiFi） |
| 公网下载 | 1.5–1.8 MB/s |
| CPU 加密吞吐 | 322 MB/s |

---

## 模块

按顺序执行，每步有检查点。

| | 内容 | 完成后 |
|---|---|---|
| [01-hardware.md](docs/01-hardware.md) | 烧系统、开 SSH | 能远程登录 |
| [02-storage.md](docs/02-storage.md) | 挂盘、Docker、挂载保护 | 盘挂上，Docker 可用 |
| [03-nextcloud.md](docs/03-nextcloud.md) | 部署 Nextcloud、改配置 | 局域网可登录 |
| [04-tunnel.md](docs/04-tunnel.md) | 建隧道、接域名 | 公网可访问 |
| [05-ops.md](docs/05-ops.md) | 三条入口、自动守护 | 重启自恢复 |

---

## 前置条件

- 已注册域名（阿里云），可修改 DNS
- Cloudflare 账号（免费）
- 树莓派可联网

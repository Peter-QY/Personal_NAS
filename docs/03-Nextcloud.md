目标：局域网可登录，配置一次改到位。

这一步的配置必须在接公网前完成。

---

## 创建 compose 文件

```bash
sudo mkdir -p /srv/nas/nextcloud && cd /srv/nas/nextcloud
```

写入 `compose.yaml`：

```yaml
services:
  db:
    image: mariadb:11.8
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED
    environment:
      MARIADB_ROOT_PASSWORD: ${MARIADB_ROOT_PASSWORD}
      MARIADB_DATABASE: nextcloud
      MARIADB_USER: nextcloud
      MARIADB_PASSWORD: ${NEXTCLOUD_DB_PASSWORD}
    volumes:
      - /srv/nas/nextcloud/db:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--save", "", "--appendonly", "no"]

  app:
    image: nextcloud:34.0.6-apache
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:80"
      - "8081:80"
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: ${NEXTCLOUD_DB_PASSWORD}
      REDIS_HOST: redis
      NEXTCLOUD_ADMIN_USER: peter
      NEXTCLOUD_ADMIN_PASSWORD: ${NEXTCLOUD_ADMIN_PASSWORD}
    volumes:
      - /srv/nas/nextcloud/app:/var/www/html
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  cron:
    image: nextcloud:34.0.6-apache
    restart: unless-stopped
    entrypoint: /cron.sh
    volumes:
      - /srv/nas/nextcloud/app:/var/www/html
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
```

`cron` 容器不可省略。预览图生成、版本清理、外部存储扫描依赖后台任务。

端口说明：`8080` 仅绑 loopback，供 cloudflared 回源；`8081` 供局域网访问。

## 密码

```bash
cat > .env <<'EOF'
MARIADB_ROOT_PASSWORD=随机长串
NEXTCLOUD_DB_PASSWORD=随机长串
NEXTCLOUD_ADMIN_PASSWORD=登录密码
EOF
chmod 600 .env
```

生成随机串：`openssl rand -base64 24`

## 启动

```bash
docker compose up -d
docker ps
```

首次启动初始化数据库约需 30 秒。`db` 健康检查通过后 `app` 才启动。

## 配置

使用 `occ` 修改，不要手改 `config.php`（Nextcloud 会重写该文件，注释丢失）。

```bash
OCC="docker exec -u www-data nextcloud-app-1 php occ"
```

### overwriteprotocol

```bash
$OCC config:system:delete overwriteprotocol
```

设置该项后，Nextcloud 对所有请求强制生成 https 链接，包括局域网明文 HTTP 访问，导致跳转到无 TLS 监听的端口，页面无法打开。

删除后协议由 `X-Forwarded-Proto` 头决定：Cloudflare 请求携带该头生成 https 链接，局域网直连不携带则保持 http。

### trusted_proxies

```bash
$OCC config:system:set trusted_proxies 0 --value="172.18.0.1"
```

`172.18.0.1` 是 Docker 网桥网关，容器收到的所有请求源地址均为它。

不要添加公网 IP。该项表示信任该来源转发的协议头。

### trusted_domains

```bash
i=0
for d in localhost 127.0.0.1 192.168.1.105 raspberrypi.local; do
  $OCC config:system:set trusted_domains $i --value="$d"
  i=$((i+1))
done
```

未列出的域名返回 400。接入域名后需追加。

### 分块大小

```bash
$OCC config:system:set files.chunked_upload.max_size --value=52428800 --type=integer
$OCC config:system:set files.chunked_upload.max_parallel_count --value=2 --type=integer
```

设为 50MB。Cloudflare 单请求上限 100MB，Nextcloud 客户端默认分块 100MB 会贴线失败。

### 后台任务

```bash
$OCC config:system:set maintenance_window_start --type=integer --value=3
$OCC maintenance:repair --include-expensive
```

## 检查点

```bash
curl -s http://127.0.0.1:8080/status.php
$OCC config:system:get trusted_proxies
$OCC config:system:get files.chunked_upload.max_size
curl -s -o /dev/null -w '%{http_code}\n' http://192.168.1.105:8081/login
```

`status.php` 返回 `"installed":true`，最后一项返回 200。

浏览器打开 `http://<树莓派IP>:8081` 登录，上传一个几十 MB 文件确认读写正常。

---
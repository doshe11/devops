针对 `RELEASE.2025-04-22T22-12-26Z` 这个版本，有两种主流的 Linux 部署方式。考虑到这个版本是官方开源控制台功能完整的最后一个版本，使用二进制文件配合 Systemd 是最稳定、可控的选择。

### 二进制部署（推荐）
这种方式无需 Docker，直接运行在宿主机上，性能损耗最小，也便于通过 Systemd 管理。

**1. 准备用户与目录**
出于安全考虑，不建议用 root 直接运行，需创建专用用户。

```bash
# 创建不可登录的 minio-user 用户
sudo useradd -r minio-user -s /sbin/nologin

# 创建数据目录并授权（假设数据盘挂载在 /data/minio）
sudo mkdir -p /data/minio
sudo chown -R minio-user:minio-user /data/minio
```

**2. 放置二进制文件**
将你下载好的特定版本二进制文件放入系统路径。

```bash
# 移动到 /usr/local/bin 并重命名
sudo mv minio.RELEASE.2025-04-22T22-12-26Z /usr/local/bin/minio

# 赋予执行权限
sudo chmod +x /usr/local/bin/minio
```

**3. 配置环境变量**
集中管理账号密码和数据路径。

```bash
sudo vim /etc/default/minio
```

写入以下内容（**务必修改密码**）：
```ini
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="请设置一个复杂的密码"
MINIO_VOLUMES="/data/minio"
MINIO_OPTS="--address :9000 --console-address :9001"
```

**4. 创建 Systemd 服务**
实现开机自启和进程守护。

```bash
sudo vim /etc/systemd/system/minio.service
```

```ini
[Unit]
Description=MinIO Object Storage
After=network.target

[Service]
User=minio-user
Group=minio-user
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**5. 启动与验证**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now minio
sudo systemctl status minio
```
看到 `active (running)` 即表示启动成功。控制台访问地址为 `http://服务器IP:9001`。

### Docker 部署（快速验证）
如果你不想在宿主机安装，可以用 Docker 快速拉起。**注意：必须指定具体的版本号**，否则拉取 `latest` 会因官方策略变更导致控制台功能缺失。

```bash
docker run -d \
  --name minio \
  -p 9000:9000 \
  -p 9001:9001 \
  -v /your/data/path:/data \
  -e "MINIO_ROOT_USER=admin" \
  -e "MINIO_ROOT_PASSWORD=admin123456" \
  minio/minio:RELEASE.2025-04-22T22-12-26Z \
  server /data --console-address ":9001"
```
运行后同样通过 `http://服务器IP:9001` 访问控制台。

### 关键提醒
由于你指定的是 `RELEASE.2025-04-22T22-12-26Z`，这是 MinIO 开源版控制台功能完整的分水岭。部署时请务必：
*   **锁定版本**：无论二进制还是 Docker，都严格使用该版本，不要更新到最新版。
*   **安全加固**：该版本已停止官方维护，如果暴露在公网，建议在前端加一层 Nginx 配置 SSL 和访问控制。
*   **端口放行**：如果开启了防火墙，记得放行 `9000`（API）和 `9001`（控制台）端口。

MinIO 的 Nginx 代理配置，核心在于 **API（9000 端口）和控制台（9001 端口）分开处理**，并针对流式上传和 WebSocket 做特殊设置。

结合你正在使用的 `RELEASE.2025-04-22T22-12-26Z` 版本，以下是一份可直接使用的配置方案。

### 使用 Nginx代理
#### Nginx 配置文件

在 `/etc/nginx/conf.d/` 下新建 `minio.conf`（或直接编辑主配置文件）：

```nginx
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }
    upstream minio_s3 {
        server 127.0.0.1:9000; 
        keepalive 32;
        keepalive_timeout 60s;
        keepalive_requests 1000;
    }

    upstream minio_console {
        server 127.0.0.1:9001;    
        keepalive 32;
        keepalive_timeout 60s;
        keepalive_requests 1000;
    }

    server {
        listen 25515 ssl http2;
	    ...
	    
    ####### minio
    # ========== S3 API 路径 ==========
        location / {
            proxy_pass http://minio_s3;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            chunked_transfer_encoding off;

            proxy_buffering off;
            proxy_request_buffering off;
            proxy_connect_timeout 300s;
            proxy_send_timeout    300s;
            proxy_read_timeout    300s;


         }

 # ========== 控制台 UI 路径 ==========
        location /minio/ui/ {
            proxy_pass http://minio_console/;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            chunked_transfer_encoding off;

            proxy_connect_timeout 300s;
            proxy_send_timeout    300s;
            proxy_read_timeout    300s;


        }
```

#### MinIO 环境变量调整

Nginx 代理后，MinIO 需要知道外部访问的地址，否则控制台可能无法正常跳转或报 WebSocket 403 错误。

编辑 `/etc/default/minio`，在原有变量基础上**增加**
**MINIO_SERVER_URL 和 MINIO_BROWSER_REDIRECT_UR**：

```ini
MINIO_ROOT_USER="xxx"
MINIO_ROOT_PASSWORD="xxx"
MINIO_VOLUMES="/xxx/minio"
MINIO_OPTS="--address :9000 --console-address :9001"
MINIO_SERVER_URL="https://xxxx:443"
MINIO_BROWSER_REDIRECT_URL="https://doshell.cn:25515/minio/ui"
```

改完后重启 MinIO：
```bash
sudo systemctl restart minio
```

#### 验证代理是否生效

```bash
# 检查 API 健康端点（通过 Nginx）
curl http://your-domain.com/minio/health/live
# 应返回 OK

# 检查 Nginx 配置语法
sudo nginx -t

# 重载 Nginx
sudo systemctl reload nginx
```

#### 几个容易踩的坑

1. **路径必须区分**：S3 API 走 `/`，控制台走 `/minio/ui/`。如果混在一起，控制台会加载异常。
2. **`proxy_buffering off` 不能省**：MinIO 的流式传输依赖这个设置，否则大文件上传/下载会出问题。
3. **WebSocket 的 `Connection` 头**：S3 API 用 `Connection ""`（长连接），控制台用 `Connection "upgrade"`（WebSocket），两者不能搞反。
4. **`MINIO_BROWSER_REDIRECT_URL` 必须设置**：这是 2023 年后版本控制台正常工作的前提，否则会出现 WebSocket 403 或登录后跳转异常。
5. **配置obsidian同步时强制使用 Path-Style URL**：Remotely Save的S3设置中有一个“**S3 URL style**”选项。对于自建MinIO，应选择“**Path-Style**”。这会让插件使用 `https://your-domain/bucket-name/...` 的格式，而非 `https://bucket-name.your-domain/...`，能避免大量因域名解析和虚拟主机风格引起的兼容性问
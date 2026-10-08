

`hacdias/webdav` 是一个用 Go 编写的轻量级、单文件 WebDAV 服务器，以极低的资源占用和便捷的部署著称。以下是基于最新 v5 版本的详细部署教程。

---

| 方式 | 适用场景 | 难度 |
|---|---|---|
| 预编译二进制 | 快速体验、个人使用 | ⭐ |
| Docker / Docker Compose | 容器化环境、服务器部署 | ⭐⭐ |
| Go 工具链安装 | 有 Go 开发环境 | ⭐⭐ |
| 源码编译 | 需要自定义构建 | ⭐⭐⭐ |

---

## 二进制文件部署（推荐）

### 1. 下载二进制

访问 [GitHub Releases](https://github.com/hacdias/webdav/releases) 页面，根据系统架构下载对应文件。以 Linux amd64 为例：

```bash
mkdir -p /opt/webdav
cd /opt/webdav
wget https://github.com/hacdias/webdav/releases/download/v5.16.1/linux-amd64-webdav.tar.gz
tar xf linux-amd64-webdav.tar.gz
chmod +x webdav
```

### 2. 创建配置文件

在 `/opt/webdav/` 下创建 `config.yml`：

```yaml
address: 127.0.0.1            # 仅监听本地，由 Nginx 对外暴露
port: 6060
directory: /data/webdav/data  # 文件存储根目录
behindProxy: true             # 关键：启用反向代理支持
prefix: /dav/                 # nginx 代理 location 
# 日志（可选）
log:
  format: console
  outputs:
    - stderr
    - /data/webdav/webdav.log

# TLS 配置（可选，生产环境强烈建议启用）
#tls: false
#cert: cert.pem
#key: key.pem

# 用户配置
users:
  - username: sundongsheng
    password: "Sds@289970862"  # 建议使用 bcrypt 加密，见下方说明
    permissions: CRUD          # 增删改查权限
```

**关键配置说明**：

| 配置项                  | 作用                                                                                                                    |
| -------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `address: 127.0.0.1` | 仅监听回环地址，外部无法直连，必须经过 Nginx                                                                                             |
| `behindProxy: true`  | 使 WebDAV 从 `X-Forwarded-For` 头中获取真实客户端 IP[](https://deepwiki.com/hacdias/webdav/5.3-integration-with-other-systems#1) |
| `permissions: CRUD`  | C=创建，R=读取，U=更新，D=删除                                                                                                   |

**密码安全建议**：生产环境建议使用 bcrypt 哈希替代明文密码，格式为 `"{bcrypt}$2y$10$..."`[](https://socket.dev/go/package/github.com/hacdias/webdav/v3?section=overview&version=v3.2.0#1)。可用 `htpasswd -bnBC 10 "" yourpassword | tr -d ':\n'` 生成。


### 3. 设置文件权限

```bash
# 创建专用用户
sudo useradd -r -s /sbin/nologin webdav
sudo chown -R webdav:webdav /data/webdav/data
sudo chmod 750 /data/webdav/data
```

### 4. 配置 systemd 服务

创建 `/etc/systemd/system/webdav.service`[](https://deepwiki.com/hacdias/webdav/5.3-integration-with-other-systems#1)：

```bash
sudo tee /etc/systemd/system/webdav.service > /dev/null <<'EOF'
[Unit]
Description=WebDAV Server (hacdias)
After=network.target
[Service]
Type=simple
User=webdav
Group=webdav
ExecStart=/usr/local/bin/webdav --config /etc/webdav/config.yml
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

启用并启动：
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now webdav
sudo systemctl status webdav
```

验证本地服务：
```bash
curl -u doshell:YourStrongPassword http://127.0.0.1:6060/
```
应返回目录列表或 207 Multi-Status 响应。

---

## Docker 部署

### 1. 拉取镜像

```bash
# GitHub Container Registry
docker pull ghcr.io/hacdias/webdav:latest

# 或 Docker Hub
docker pull hacdias/webdav:latest
```

镜像基于 `scratch` 构建，体积极小，仅包含二进制文件和 CA 证书。

### 2. 运行容器

```bash
docker run -d \
  --name webdav \
  -p 6065:6065 \
  -v $(pwd)/config.yml:/config.yml:ro \
  -v $(pwd)/data:/data \
  --restart unless-stopped \
  ghcr.io/hacdias/webdav:latest \
  -c /config.yml
```

关键参数说明：
- `-p 6065:6065`：映射端口，容器默认暴露 6065
- `-v ./config.yml:/config.yml:ro`：挂载配置文件（只读）
- `-v ./data:/data`：挂载数据目录，**配置文件中 `directory` 需与此路径一致**
- `-c /config.yml`：指定容器内配置文件路径


---



## 反向代理（Nginx）

通过 Nginx 提供 HTTPS 终结和域名访问：

```nginx
server {
    listen 80;
    server_name doshell.cn;

    # 后续 Certbot 会自动添加 HTTPS 跳转，此处先保留 HTTP 用于证书验证
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name doshell.cn;

    # ── SSL 证书（Certbot 会自动填充，见第 6 步）──
    # ssl_certificate     /etc/letsencrypt/live/doshell.cn/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/doshell.cn/privkey.pem;

    # ── WebDAV 反向代理核心配置 ──
    location /dav/ {
        proxy_pass http://127.0.0.1:6060/dav/;

        # 透传真实客户端信息
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebDAV 关键头：Destination 用于 MOVE/COPY，必须透传
        proxy_set_header Destination       $http_destination;

        # 确保 PROPFIND 等方法被转发
        proxy_pass_request_headers on;

        # 大文件上传支持
        client_max_body_size 0;
        proxy_request_buffering off;

        # 超时设置（WebDAV 操作可能较慢）
        proxy_connect_timeout 300s;
        proxy_send_timeout    300s;
        proxy_read_timeout    300s;
    }
}
```

**关键点**：
- WebDAV 协议依赖 `Destination` 头进行 MOVE/COPY 操作，反向代理时必须正确转发该头部。
- 需要注意webdav目录权限要授予nginx（www-data）用户

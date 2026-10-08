
**Gossa** 是一款基于 Go 语言（Golang）开发的极简、快速的轻量级文件服务器

---

## 第一步：安装 Gossa

Gossa 是单个二进制文件，无需运行时依赖。从 GitHub Releases 下载对应架构的版本：

```bash
# 下载最新版 Linux amd64 二进制
curl -LO https://github.com/pldubouilh/gossa/releases/latest/download/gossa-linux-amd64

# 移动到系统路径并赋予执行权限
sudo mv gossa-linux-amd64 /usr/local/bin/gossa
sudo chmod +x /usr/local/bin/gossa

# 验证
gossa --help
```

如果你更喜欢 Docker，也可以用官方镜像：

```bash
mkdir -p ~/LocalDirToShare
sudo docker run -v ~/LocalDirToShare:/shared -p 8001:8001 pldubouilh/gossa
```

---

## 第二步：创建专用用户与数据目录

不要以 root 运行 Gossa。

```bash
# 创建不可登录的系统用户
sudo useradd -r -s /sbin/nologin -m -d /var/lib/gossa gossa

# 创建要共享的文件目录
sudo mkdir -p /srv/gossa/files
sudo chown -R gossa:gossa /srv/gossa
```

---

## 第三步：配置 systemd 服务（用户级）

Gossa 官方推荐使用**用户级 systemd 服务**来管理启动。以 `gossa` 用户身份创建服务文件：

```bash
# 切换到 gossa 用户
sudo -u gossa -s /bin/bash

# 创建用户级 systemd 目录
mkdir -p ~/.config/systemd/user/

# 创建服务文件
cat > ~/.config/systemd/user/gossa.service << 'EOF'
[Unit]
Description=Gossa file server
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/bin/gossa -h 127.0.0.1 -p 8001 /srv/gossa/files
Restart=always
RestartSec=30

[Install]
WantedBy=default.target
EOF

# 退出回普通用户
exit
```

**关键参数说明**：

- `-h 127.0.0.1`：**只监听本地回环地址**，确保 Gossa 不直接暴露在公网上，所有外部访问必须经过 Nginx。
- `-p 8001`：Gossa 默认端口。
- `/srv/gossa/files`：要共享的目录路径。

启用并启动用户级服务（注意命令中**没有 `sudo`**）：

```bash
# 以 gossa 用户身份启用 linger，使其在未登录时也能保持服务运行
sudo loginctl enable-linger gossa

# 以 gossa 用户身份启动服务
sudo -u gossa systemctl --user daemon-reload
sudo -u gossa systemctl --user enable --now gossa

# 检查状态
sudo -u gossa systemctl --user status gossa
```

---

## 第四步：配置 Nginx 认证代理（必须）

Gossa **没有内置认证机制**，官方明确将 HTTPS 和认证交由中间件和代理处理。你之前在 Copyparty 部署中已经配置过 Nginx，这里沿用同样的思路，用 **HTTP Basic Auth** 为 Gossa 添加一层认证。

### 1. 创建密码文件

```bash
# 安装 htpasswd 工具（如尚未安装）
sudo apt install -y apache2-utils

# 创建密码文件（会提示输入密码）
sudo htpasswd -c /etc/nginx/.htpasswd_gossa 你的用户名
```

### 2. 创建 Nginx 站点配置

```bash
sudo nano /etc/nginx/sites-available/gossa
```

填入以下内容（假设使用子域名 `files.yourdomain.com`）：

```nginx
server {
    listen 80;
    server_name files.yourdomain.com;

    # 认证
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd_gossa;

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Gossa 的上传和流媒体需要关闭缓冲
        proxy_buffering off;
        proxy_request_buffering off;

        # 支持大文件上传
        client_max_body_size 0;
    }
}
```

如果你暂时没有域名，只想通过公网 IP 访问，可以将 `server_name` 改为你的公网 IP，但**强烈建议后续配置域名并启用 HTTPS**。

### 3. 启用站点并配置 HTTPS

```bash
sudo ln -s /etc/nginx/sites-available/gossa /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# 如果有域名，用 certbot 自动配置 HTTPS
sudo certbot --nginx -d files.yourdomain.com
```

---

## 第五步：开放安全组端口

在阿里云控制台的安全组中，只开放 **80** 和 **443**。**不要**开放 8001 端口，因为 Gossa 已经通过 `-h 127.0.0.1` 限制为仅本地监听，外部无法直接访问。

---

## 验证部署

1. 浏览器访问 `https://files.yourdomain.com`（或 `http://你的公网IP`）
2. 输入刚才用 `htpasswd` 设置的用户名和密码
3. 进入 Gossa 界面，尝试拖拽上传文件

Gossa 支持拖放上传、视频流播放、图片浏览、PDF 查看和内置笔记编辑器，界面非常简洁。

---

## 补充说明：为什么 Gossa 必须配合代理使用

Gossa 的设计哲学是“只做一件事，并做好它”。它把认证和加密完全交给上游代理，自身代码因此能保持在 250 行以内，极易于审计和安全审查。这也意味着：

- **不能**直接把 Gossa 的 8001 端口映射到公网
- **不能**依赖 Gossa 自身做任何访问控制
- 必须通过 Nginx（Basic Auth）或 Caddy（多用户路由）等工具来保护

官方在 `support/` 目录中提供了 Caddy 的多用户配置示例，如果你的需求更复杂（比如区分只读用户和管理员），可以参考官方示例进一步调整。
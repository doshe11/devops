**Copyparty** 是一款开源的轻量级、便携式自托管文件服务器工具，其核心特点是**所有的功能和代码都集成在一个 Python 文件或简单的容器中**，无需安装复杂的外部数据库或繁琐的依赖。

## 第一步：环境准备与快速启动

连接到你的 VPS，下载最新版 Copyparty 脚本：

```bash
# 下载自解压脚本
curl -LO https://github.com/9001/copyparty/releases/latest/download/copyparty-sfx.py

# 赋予执行权限
chmod +x copyparty-sfx.py

# 快速测试运行（默认在当前目录共享，端口 3923）
python3 copyparty-sfx.py
```

此时访问 `http://你的公网IP:3923` 应该能看到 Web 界面。默认情况下，**任何人都能读写当前目录**，这只适合临时测试，绝不能长期暴露在公网。

按 `Ctrl+C` 停止，接下来进行正式配置。

---

## 第二步：创建专用用户与目录

为安全起见，不要用 root 运行 Copyparty。

```bash
# 创建专用系统用户（不可登录）
useradd -r -s /sbin/nologin -M  copyparty

# 创建配置目录和数据目录
mkdir -p /data/copyparty
mkdir -p /data/copyparty/files

# 将数据目录所有权交给 copyparty 用户
chown -R copyparty:copyparty /data/copyparty
```



---

## 第三步：编写配置文件

Copyparty 支持配置文件，比命令行参数更清晰，也便于后续维护。创建 `/etc/copyparty/copyparty.conf`：

```ini
[global]
  # 监听所有网卡的 3923 端口
  p: 3923
  # 使用nginx路径代理时，必须通过 `--rp-loc` 参数告诉它实际的挂载路径
  rp-loc: /copyparty
  # 启用文件索引（搜索、去重功能的基础）
  e2dsa
  # 启用多媒体索引（音频标签、缩略图，需要 ffmpeg）
  e2ts
  # 日志输出到 systemd journal
  lo: -
  # 启用用户名模式（必须提供用户名才能登录）
  usernames
[accounts]
  # 格式：用户名: 密码
  # 请务必替换为强密码
  admin: 你的强密码
[/]
  # 共享根目录
  /srv/copyparty/files
  # 权限配置：
  # admin 用户拥有全部权限（rwmda）
  accs:
    rwmda: admin
```

### 权限速查

Copyparty 的权限采用简短字母表示：

| 权限 | 含义 |
|---|---|
| `r` | 读取（浏览目录、下载文件） |
| `w` | 写入（上传文件） |
| `m` | 移动文件 |
| `d` | 删除文件 |
| `a` | 管理员（查看上传 IP、重载配置） |

`rwmda` 等于全部权限。如果希望其他用户只能读不能写，可以这样配置：

```ini
[/]
  /srv/copyparty/files
  accs:
    r: 其他用户名
    rwmda: admin
```

---

## 第四步：配置 systemd 服务（开机自启）

创建 `/etc/systemd/system/copyparty.service`：

```ini
[Unit]
Description=copyparty file server
After=network.target

[Service]
Type=notify
User=copyparty
Group=copyparty
WorkingDirectory=/var/lib/copyparty
Environment=PYTHONUNBUFFERED=x

# 安全加固
MemoryMax=50%
MemorySwapMax=50%
ProtectClock=true
ProtectControlGroups=true
ProtectHostname=true
ProtectKernelLogs=true
ProtectKernelModules=true
ProtectKernelTunables=true
ProtectProc=invisible
RemoveIPC=true
RestrictNamespaces=true
RestrictRealtime=true
RestrictSUIDSGID=true

# 启动命令
ExecStart=/usr/bin/python3 /usr/local/bin/copyparty-sfx.py -c /etc/copyparty/copyparty.conf

[Install]
WantedBy=multi-user.target
```



将脚本复制到系统路径，然后启用服务：

```bash
# 复制脚本到系统 bin 目录
cp copyparty-sfx.py /usr/local/bin/
chmod +x /usr/local/bin/copyparty-sfx.py

# 重载 systemd 并启动
systemctl daemon-reload
systemctl enable --now copyparty

# 查看状态
systemctl status copyparty
```

---

## 第五步：配置反向代理与 HTTPS（强烈推荐）

直接暴露 3923 端口在公网上不够安全。**最佳实践是让 Copyparty 只监听 localhost，由 Nginx 处理 HTTPS 和外部访问**。

### 1. 修改 Copyparty 配置，仅监听本地

在 `/etc/copyparty/copyparty.conf` 的 `[global]` 中修改：

```ini
[global]
  i: 127.0.0.1
  p: 3923
  xff-hdr: x-forwarded-for
  # ... 其余保持不变
```

### 2. 安装 Nginx 并配置反向代理

```bash
sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx
```

创建 `/etc/nginx/sites-available/copyparty`：

```nginx
server {
    listen 80;
    server_name your-domain.com;  # 替换为你的域名

    client_max_body_size 0;  # 不限制上传大小

    location /copyparty {
        proxy_pass http://127.0.0.1:3923;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 支持（Copyparty 的部分实时功能需要）
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```


### 3. 重启 Copyparty 使配置生效

```bash
systemctl restart copyparty
```

---

## 第六步：开放防火墙与阿里云安全组

Copyparty 默认使用 3923 端口，但经过 Nginx 代理后，**只需开放 80 和 443 端口**。

**服务器防火墙**（如果使用 `firewalld`）：

```bash
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload
```



**阿里云安全组**（在阿里云控制台操作）：添加两条入方向规则，分别允许 **TCP 80** 和 **TCP 443**，来源设为 `0.0.0.0/0`。**不要**开放 3923 端口。

---

## 第七步：安装可选功能（推荐）

以下功能能显著提升使用体验：

```bash
sudo apt install --no-install-recommends python3-pil ffmpeg
```

- **python3-pil**：生成图片缩略图
- **ffmpeg**：音频/视频元数据索引与在线转码

安装后重启服务：

```bash
systemctl restart copyparty
```

---

## 验证部署

1. 浏览器访问 `https://your-domain.com`
2. 使用配置文件中 `admin` 账号的**用户名和密码**登录
3. 尝试上传文件、创建文件夹，确认权限正常

---

## 补充：VPS 上的额外安全建议

由于你的服务器有公网 IP，除了上述基础配置外，建议额外注意：

- **启用密码哈希**：在 `[accounts]` 中使用哈希密码而非明文。可以通过 `echo "admin:你的密码" | python3 copyparty-sfx.py --ah-alg argon2 --ah-cli` 生成哈希值
- **限制上传大小**：在配置中添加 `--max-size 500M` 等参数，防止磁盘被塞满
- **定期检查日志**：使用 `journalctl -u copyparty -f` 查看实时访问日志，关注异常 IP

部署完成后，通过 `https://你的域名` 即可访问，上传速度应该会比之前的 OpenVPN+SSH 方案有明显提升。如果遇到配置问题，可以先用 `journalctl -u copyparty -n 50` 查看最近的日志来排查。
在 Nginx 上启用 WebDAV 功能，需要配置两个模块协同工作：Nginx 内置的 `ngx_http_dav_module` 负责处理基础的 `PUT`、`DELETE`、`MKCOL`、`COPY`、`MOVE` 方法；而第三方模块 `nginx-dav-ext-module` 则负责补全 `PROPFIND`、`OPTIONS`、`LOCK`、`UNLOCK` 等关键方法，否则 Windows 或 macOS 客户端可能无法正常挂载或频繁报错。

### 环境准备与模块确认

首先需要确认 Nginx 的编译参数中是否已包含内置的 DAV 模块。

```bash
nginx -V 2>&1 | grep -o with-http_dav_module
```

如果终端有 `with-http_dav_module` 输出，说明内置模块已就绪。若没有输出，则需要重新编译 Nginx 并加入 `--with-http_dav_module` 参数。

> **注意**：对于宝塔面板等集成环境，默认通常未编译此模块，可能需要卸载后选择“编译安装”并添加相应参数。

### 安装 nginx-dav-ext-module 扩展模块

该模块是完整 WebDAV 功能的核心，安装方式取决于你的系统。

>`with-http_dav_module`和`nginx-dav-ext-module` 这两个东西经常一起出现，但角色完全不同：**一个是 Nginx 官方内置的 WebDAV 基础模块，另一个是第三方扩展模块，用来补全官方没实现的方法。** 两者是互补关系，不是二选一。


**方式一：通过包管理器安装（推荐）**

如果使用 Fedora、CentOS 或 Ubuntu，可以尝试通过 GetPageSpeed 等第三方仓库安装。

```bash
# Fedora 示例
sudo dnf -y install https://extras.getpagespeed.com/release-latest.rpm
sudo dnf -y install nginx-module-dav-ext
```

安装后，需要在 `nginx.conf` 中加载模块：

```nginx
load_module modules/ngx_http_dav_ext_module.so;
```

**方式二：从源码编译**

如果包管理器不可用，则需要下载源码编译。编译 Nginx 时，将模块路径通过 `--add-dynamic-module` 或 `--add-module` 加入：

```bash
# 编译动态模块（推荐）
./configure --with-http_dav_module --add-dynamic-module=/path/to/nginx-dav-ext-module
make && make install
```

> **兼容性提示**：`nginx-dav-ext-module` 可能与 Nginx 1.25.0+ 版本不兼容，建议锁定在 1.22.x 或 1.24.x 版本进行编译。

### 核心配置示例

以下是一个兼顾了安全性、客户端兼容性和功能完整性的配置示例。

```nginx
server {
    listen 443 ssl http2;
    server_name your.domain.com;

    ssl_certificate     /etc/ssl/certs/your.crt;
    ssl_certificate_key /etc/ssl/private/your.key;

    # 认证配置
    auth_basic "WebDAV Access";
    auth_basic_user_file /etc/nginx/.webdav_users;

    location /webdav/ {
        # 绑定到真实的物理存储目录
        root /data;
        # 确保 Nginx 运行用户对该目录有读写权限

        # 启用内置的 WebDAV 方法
        dav_methods PUT DELETE MKCOL COPY MOVE;

        # 启用扩展模块提供的方法（关键）
        dav_ext_methods PROPFIND OPTIONS LOCK UNLOCK;

        # 允许自动创建多级子目录
        create_full_put_path on;

        # 设置新创建文件和目录的权限
        dav_access user:rw group:r all:r;

        # 锁定区域配置（用于支持 LOCK/UNLOCK）
        # 需要在 http 块中预先定义：dav_ext_lock_zone zone=webdav_lock:10m;
        dav_ext_lock zone=webdav_lock;

        # 大文件上传支持
        client_max_body_size 10G;
        client_body_temp_path /tmp/nginx_webdav_temp;

        # 解决 macOS 客户端创建目录时缺少尾部斜杠的问题
        set $x $uri$request_method;
        if ($x ~ [^/]MKCOL$) {
            rewrite ^(.*)$ $1/;
        }

        # 可选：启用目录索引的 XML 格式，方便部分客户端解析
        autoindex on;
        autoindex_format xml;
    }
}
```

**配置指令说明**：

*   **`dav_methods`**：放行 PUT、DELETE、MKCOL、COPY、MOVE 方法。
*   **`dav_ext_methods`**：**必须包含 PROPFIND、OPTIONS、LOCK、UNLOCK**，否则 Windows 映射驱动器会失败或反复弹出认证框。
*   **`create_full_put_path on`**：允许客户端上传时自动创建缺失的父目录。
*   **`dav_access`**：控制新创建文件和目录的权限，建议避免使用 `all:rw` 以降低安全风险。
*   **`dav_ext_lock_zone`**：在 `http` 块中定义一个共享内存区域，用于存储 WebDAV 的锁信息。
*   **`client_max_body_size`**：根据需求设置，否则上传大文件时可能被拒绝。

### 设置目录权限与认证

**创建存储目录并赋予权限**：

```bash
sudo mkdir -p /data/webdav
# 将目录所有者改为 Nginx 的运行用户（通常为 www-data 或 nginx）
sudo chown -R www-data:www-data /data/webdav
```

**生成基础认证密码文件**：

```bash
# 首次创建需要使用 -c 参数
sudo htpasswd -c /etc/nginx/.webdav_users your_username
# 后续添加用户时，去掉 -c 参数
sudo htpasswd /etc/nginx/.webdav_users another_user
```

务必确保密码文件权限安全，建议设置为 `640`，仅允许 Nginx 主进程用户读取。

### 强制启用 HTTPS

WebDAV 的基础认证凭据在 HTTP 下是明文传输的，存在安全风险。此外，现代 Windows 系统默认禁止对非加密站点进行 WebDAV 挂载。

配置 HTTPS 有两种常见方式：

1.  **使用 Let‘s Encrypt 等免费证书**：按照 Certbot 等工具的指引获取并自动配置证书。
2.  **直接配置 SSL 证书**：在 `server` 块中指定 `ssl_certificate` 和 `ssl_certificate_key` 的路径。

配置完成后，务必使用 `https://` 协议进行访问和挂载。

### 验证与客户端挂载

**服务端测试**：可以使用 `cadaver` 命令行工具进行快速测试。

```bash
cadaver https://your.domain.com/webdav/
```

**Windows 客户端挂载**：

*   首先，以管理员身份启动 `WebClient` 服务：`net start webclient`。
*   然后，在“此电脑”中右键选择“映射网络驱动器”，输入 `https://your.domain.com/webdav/` 作为文件夹路径。

> **注意**：Windows 原生客户端可能存在 50MB 文件大小限制。如果需要处理大文件，建议使用 RaiDrive、Cyberduck 或 ExpanDrive 等第三方客户端，它们通常更稳定且无此限制。

### 🛠️ 常见问题排查

*   **Windows 挂载失败或反复弹窗**：最常见的原因是 `dav_ext_methods` 中缺少 `PROPFIND` 或 `OPTIONS` 方法。
*   **上传大文件时连接重置**：检查 `client_max_body_size` 是否设置得足够大，并确认 `client_body_temp_path` 所在的磁盘分区有足够空间。
*   **权限错误（403 Forbidden）**：确认 Nginx 运行用户（如 `www-data`）对 `root` 指令指向的物理目录拥有读写权限。
*   **客户端兼容性**：对于 macOS 的 Finder，上述配置中的 `rewrite` 规则有助于解决创建目录时的问题。

完成配置后，别忘了使用 `nginx -t` 测试语法，然后 `systemctl reload nginx` 重载服务使配置生效。
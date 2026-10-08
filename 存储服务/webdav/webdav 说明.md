
 **WebDAV**全称是 **Web-based Distributed Authoring and Versioning**，中文常译为“基于 Web 的分布式创作和版本控制”。

简单说：**WebDAV 是一种基于 HTTP/HTTPS 的协议，让你能像操作本地文件一样，远程管理服务器上的文件。**

---

## 一、WebDAV 是什么

它是对 HTTP/1.1 的扩展协议，由 IETF 制定，目前标准是 **RFC 4918**。普通 HTTP 服务器主要用来“看网页”，而 WebDAV 让服务器支持：

- 上传、下载文件
- 创建、删除目录
- 重命名、移动、复制文件
- 读取和修改文件属性
- 对文件加锁，防止多人同时修改冲突

因此，WebDAV 常被用作**个人网盘、远程文件访问、NAS 文件共享、同步盘**的底层协议。

---

## 二、核心特点

| 特点 | 说明 |
|---|---|
| 基于 HTTP/HTTPS | 通常走 80/443 端口，容易穿透防火墙和代理 |
| 支持加密 | 配合 HTTPS 可安全传输 |
| 支持认证 | 常用 Basic、Digest 认证 |
| 支持文件锁 | 防止并发编辑冲突 |
| 支持属性 | 可读写文件的元数据 |
| 跨平台 | Windows、macOS、Linux、手机都能连接 |

> 注意：基础 WebDAV 不包含完整版本控制，版本控制由 DeltaV 等扩展定义。

---

## 三、常用 HTTP 方法

WebDAV 在普通 HTTP 方法之外，增加了很多专用方法：

| 方法 | 作用 |
|---|---|
| GET / PUT | 下载 / 上传文件 |
| DELETE | 删除文件或目录 |
| MKCOL | 创建目录 |
| COPY / MOVE | 复制 / 移动 |
| PROPFIND | 读取文件属性、列目录 |
| PROPPATCH | 修改文件属性 |
| LOCK / UNLOCK | 加锁 / 解锁 |
| OPTIONS | 查询服务器支持的方法 |

其中 **PROPFIND** 非常关键，很多 WebDAV 客户端挂载时全靠它列目录。Nginx 原生模块不支持它，所以之前搭建 Nginx WebDAV 时需要额外编译 `nginx-dav-ext-module`。

---

## 四、典型用途

- **远程网盘**：把服务器目录挂载到本地电脑
- **NAS 共享**：群晖、威联通等常支持 WebDAV
- **办公协作**：多人编辑同一目录下的文件
- **同步工具**：rclone、RaiDrive、Cyberduck 等通过 WebDAV 同步
- **手机文件管理**：ES 文件管理器、Solid Explorer 等可添加 WebDAV 账号

---

## 五、和 FTP / SMB 的区别

| 协议 | 端口 | 安全性 | 适用场景 |
|---|---|---|---|
| WebDAV | 80/443 | 可 HTTPS 加密 | 公网、跨平台、易穿透 |
| FTP | 21 + 数据端口 | 明文，需 FTPS | 传统文件传输，配置较麻烦 |
| SMB | 445 | 局域网内常用 | Windows 局域网共享，公网风险高 |

WebDAV 的优势是**走 HTTP/HTTPS，部署简单、兼容性好**；缺点是性能通常不如 SMB，大量小文件时可能较慢。

---

## 六、常见实现和客户端

**服务端：**
- Apache `mod_dav`
- Nginx + `nginx-dav-ext-module`
- `hacdias/webdav`（你之前问的 Go 轻量实现）
- SabreDAV、Nextcloud、ownCloud
- `rclone serve webdav`

**客户端：**
- Windows：映射网络驱动器
- macOS：Finder → 连接服务器
- Linux：`davfs2`、GNOME Files
- 第三方：RaiDrive、Cyberduck、WinSCP、ES 文件管理器

---

## 七、安全提醒

WebDAV 面向公网时务必注意：

1. **强制 HTTPS**，不要用明文 HTTP。
2. **启用强密码认证**，不要匿名写入。
3. **最小权限**，只给必要目录读写权限。
4. **限制请求方法**，不需要 COPY/MOVE 就关掉。
5. **及时更新服务端软件**，避免已知漏洞。

---


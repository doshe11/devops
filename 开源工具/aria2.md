**aria2** 是一个轻量级的多协议、多来源命令行下载工具。它支持 **HTTP/HTTPS、FTP、SFTP、BitTorrent 以及 Metalink** 协议，能够从多个来源同时下载同一个文件，最大限度地利用带宽。其命令格式为 `aria2c [选项] [URI|磁力链接|种子文件|Metalink文件]`。

---

### 安装

- **Debian/Ubuntu**: `sudo apt install aria2`
- **Arch Linux**: `sudo pacman -S aria2`
- **macOS**: `brew install aria2`
- **Windows**: 可通过 scoop 安装，`scoop install aria2`

---

### 基本下载

#### 下载单个文件

```bash
aria2c https://example.com/file.zip

nohup aria2c https://example.com/file.zip > aria2.log 2>&1 &
```

#### 指定保存目录

使用 `-d` 或 `--dir` 指定下载文件的存放目录：

```bash
aria2c -d /home/user/downloads https://example.com/file.zip
```

#### 自定义文件名

使用 `-o` 或 `--out` 指定输出文件名：

```bash
aria2c -o myfile.zip https://example.com/file.zip
```

#### 多线程下载（加速）

`-x` 用于设置**单服务器最大连接数**（即线程数），`-s` 用于设置**分片数**。最大线程数为 16：

```bash
aria2c -x 16 -s 16 https://example.com/largefile.iso
```

#### 断点续传

使用 `-c` 或 `--continue` 可以继续未完成的下载：

```bash
aria2c -c -s 2 https://example.com/partiallydownloadedfile.zip
```

#### 限速下载

使用 `--max-download-limit` 限制单个任务的下载速度，`--max-overall-download-limit` 限制全局下载速度：

```bash
# 限制单任务速度为 1MB/s
aria2c --max-download-limit=1M https://example.com/file.zip

# 限制全局下载速度为 500KB/s
aria2c --max-overall-download-limit=500K https://example.com/file.zip
```

---

### 批量下载（输入文件）

将多个下载链接写入一个文本文件，每行一个 URI，然后使用 `-i` 读取：

```bash
# urls.txt 内容示例：
# https://example.com/file1.zip
# https://example.com/file2.zip
# https://example.com/file3.zip

aria2c -i urls.txt

nohup aria2c -i urls.txt > aria2.log 2>&1 &
```
```

输入文件支持注释（以 `#` 开头）和 gzip 压缩。若一行中用 **Tab 分隔**多个 URI，表示这些 URI 是同一文件的镜像源。

使用 `-j` 控制**同时下载的任务数**（默认 5），`-x` 控制每个任务的连接数：

```bash
aria2c -j 5 -x 16 -i urls.txt
```

---

### BitTorrent 下载

#### 下载种子文件

```bash
aria2c file.torrent
```

#### 下载磁力链接

磁力链接包含 `&` 符号，建议用引号包裹以避免 Shell 解析错误：

```bash
aria2c "magnet:?xt=urn:btih:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

#### 查看种子内容并选择下载

先用 `-S` 列出种子内的文件索引：

```bash
aria2c -S file.torrent
```

输出示例：

```
idx|path/length
===+======================
1|dist/base.iso        |99.9MiB
2|dist/driver.iso      |169.0MiB
```

然后用 `--select-file` 选择要下载的文件（支持范围 `1-4` 和逗号分隔 `1,3,5`）：

```bash
aria2c --select-file=1,3 file.torrent
```

#### 同时通过 Torrent 和 HTTP/FTP 下载

对于同一文件，可以同时提供种子和 HTTP/FTP 源，加速下载：

```bash
aria2c -T file.torrent "http://server1/file" "ftp://server2/file"
```

#### 做种控制

设置做种时间和分享率，达到任一条件即停止做种：

```bash
# 做种 120 分钟 或 分享率达到 1.0 后停止
aria2c --seed-time=120 --seed-ratio=1.0 file.torrent
```

#### 启用 DHT

```bash
aria2c --enable-dht --dht-listen-port=6881 file.torrent
```

注意 DHT 使用 **UDP 端口**，需自行配置防火墙放行。

---

### RPC 模式（远程控制）

启用 RPC 后，可以通过 Web UI（如 AriaNg）或 API 远程管理下载任务。

```bash
aria2c --enable-rpc --rpc-listen-all=true --rpc-allow-origin-all \
       --rpc-secret=your_secret_token --rpc-listen-port=6800
```

| 选项 | 说明 |
|---|---|
| `--enable-rpc` | 启用 JSON-RPC/XML-RPC 服务 |
| `--rpc-listen-all` | 监听所有网络接口（默认仅本地回环） |
| `--rpc-allow-origin-all` | 允许跨域请求 |
| `--rpc-secret` | 设置 RPC 授权令牌（**强烈建议设置**） |
| `--rpc-listen-port` | RPC 监听端口，默认 6800 |

启动后，在 AriaNg 等 Web UI 中填入相同的 `rpc-secret` 即可连接。

---

### 配置文件

aria2 默认读取 `$XDG_CONFIG_HOME/aria2/aria2.conf`（通常为 `~/.aria2/aria2.conf`）作为全局配置。

一个典型的 `aria2.conf` 示例：

```ini
# 继续未完成的下载
continue=true

# 下载目录
dir=/home/user/downloads

# 不预分配磁盘空间
file-allocation=none

# 从输入文件读取任务
input-file=/home/user/.aria2/input.conf

# 日志级别
log-level=warn

# 单服务器最大连接数
max-connection-per-server=4

# 最小分片大小
min-split-size=5M

# 完成后退出
on-download-complete=exit
```

等效于命令行：

```bash
aria2c --dir=/home/user/downloads --file-allocation=none \
       --input-file=/home/user/.aria2/input.conf \
       --on-download-complete=exit --log-level=warn
```

使用 `--conf-path` 指定自定义配置文件路径。

---

### 代理设置

```bash
# 为所有协议设置代理
aria2c --all-proxy="http://127.0.0.1:7890" https://example.com/file.zip

# 仅为 HTTP 设置代理
aria2c --http-proxy="http://127.0.0.1:7890" https://example.com/file.zip

# 带认证的代理
aria2c --all-proxy="http://user:pass@proxy:8080" https://example.com/file.zip
```

可分别使用 `--http-proxy`、`--https-proxy`、`--ftp-proxy` 覆盖特定协议的代理。

---

### 完整性校验

使用 `-V` 或 `--check-integrity` 在下载时校验分片哈希或整个文件的哈希值。该选项对 BitTorrent、Metalink 以及带有 `--checksum` 选项的 HTTP/FTP 下载有效。

```bash
aria2c -V https://example.com/file.iso
```

如果配合 `--hash-check-only=true`，校验完成后无论结果如何都会中止下载。

---

### 常用选项速查

| 选项                                | 说明               |
| --------------------------------- | ---------------- |
| `-d, --dir`                       | 指定下载目录           |
| `-o, --out`                       | 指定输出文件名          |
| `-x, --max-connection-per-server` | 单服务器最大连接数（最多 16） |
| `-s, --split`                     | 分片数（默认 5）        |
| `-j, --max-concurrent-downloads`  | 同时下载的任务数（默认 5）   |
| `-c, --continue`                  | 断点续传             |
| `-i, --input-file`                | 从文件批量读取下载链接      |
| `-V, --check-integrity`           | 校验文件完整性          |
| `--max-download-limit`            | 单任务下载限速          |
| `--max-overall-download-limit`    | 全局下载限速           |
| `--seed-time`                     | 做种时间             |
| `--seed-ratio`                    | 分享率上限            |
| `--enable-rpc`                    | 启用 RPC 远程控制      |
| `--conf-path`                     | 指定配置文件路径         |


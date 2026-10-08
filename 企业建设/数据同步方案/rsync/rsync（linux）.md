下面是一份从入门到进阶的 Linux `rsync` 详细教程。`rsync` 是 Linux 下最常用的文件同步、备份、镜像工具，支持本地/远程、增量传输、断点续传、排除规则、删除同步、守护进程模式等。

> 危险操作如 `--delete`，建议先加 `-n` 或 `--dry-run` 测试。

---

## 1. 安装 rsync

```bash
# Debian / Ubuntu
sudo apt update
sudo apt install rsync

# RHEL / CentOS / Rocky / AlmaLinux / Fedora
sudo dnf install rsync

# Arch Linux
sudo pacman -S rsync

# openSUSE
sudo zypper install rsync

# 查看版本
rsync --version
```

---

## 2. 基本语法

```bash
rsync [选项] 源 目标
```

本地同步：

```bash
rsync -av /data/ /backup/data/
```

远程同步，默认通过 SSH：

```bash
# 推送到远程
rsync -av /data/ user@host:/backup/data/

# 从远程拉取
rsync -av user@host:/data/ /backup/data/
```

通过 rsync daemon：

```bash
# 双冒号表示 rsync 协议
rsync -av user@host::module /dest/

# 或 rsync:// 形式
rsync -av rsync://user@host:873/module /dest/
```

注意：

- 单冒号 `host:/path`：走 SSH。
- 双冒号 `host::module`：走 rsync daemon，默认端口 `873`。
- `rsync://`：也是 rsync daemon。

---

## 3. 尾部斜杠非常重要

这是 rsync 最容易出错的地方。

```bash
# 同步 /src 目录里的“内容”到 /dest/
rsync -av /src/ /dest/

# 在 /dest/ 下创建 src 目录，再同步 /src 的内容
rsync -av /src /dest/
```

举例：

```bash
rsync -av /data/ /backup/
# 结果：/backup/file1、/backup/dir1 ...

rsync -av /data /backup/
# 结果：/backup/data/file1、/backup/data/dir1 ...
```

远程路径同理：

```bash
rsync -av /data/ user@host:/backup/
rsync -av /data user@host:/backup/
```

---

## 4. 最常用选项

| 选项                          | 说明                                   |
| --------------------------- | ------------------------------------ |
| `-a`                        | 归档模式，等价于 `-rlptgoD`，最常用              |
| `-v`                        | 显示详细信息                               |
| `-h`                        | 人类可读单位                               |
| `-z`                        | 传输时压缩，适合慢网络                          |
| `-P`                        | 等价于 `--partial --progress`，断点续传并显示进度 |
| `--delete`                  | 删除目标端多余文件，使目标与源一致                    |
| `-n` / `--dry-run`          | 只模拟，不实际执行                            |
| `--exclude`                 | 排除文件/目录，源目录的相对路径                     |
| `--include`                 | 包含文件/目录                              |
| `--exclude-from`            | 从文件读取排除规则                            |
| `--bwlimit`                 | 限速，单位约 KiB/s                         |
| `--checksum`                | 用校验和判断是否传输，较慢但准确                     |
| `--size-only`               | 只比较大小                                |
| `--update`                  | 跳过目标端较新的文件                           |
| `--existing`                | 不创建目标端不存在的文件                         |
| `--ignore-existing`         | 跳过目标端已存在的文件                          |
| `--remove-source-files`     | 传输成功后删除源文件                           |
| `--backup`                  | 覆盖/删除前备份                             |
| `--backup-dir`              | 指定备份目录                               |
| `--link-dest`               | 硬链接增量快照                              |
| `--log-file`                | 记录日志                                 |
| `--stats`                   | 显示统计信息                               |
| `-i` / `--itemize-changes`  | 显示变化项                                |
| `--info=progress2`          | 显示整体进度                               |
| `-e`                        | 指定远程 shell，如 SSH                     |
| `--rsync-path`              | 指定远程 rsync 路径，如 `sudo rsync`         |
| `-H`                        | 保留硬链接                                |
| `-A`                        | 保留 ACL                               |
| `-X`                        | 保留扩展属性                               |
| `-S`                        | 处理稀疏文件                               |
| `-L`                        | 跟随符号链接                               |
| `-l`                        | 保留符号链接，`-a` 已包含                      |
| `--numeric-ids`             | 不映射用户/组名                             |
| `--chmod`                   | 修改权限                                 |
| `--chown`                   | 修改属主属组                               |
| `--max-size` / `--min-size` | 限制文件大小                               |
| `--timeout`                 | 超时时间，秒                               |
| `--files-from`              | 从文件读取文件列表                            |

---

## 5. 本地同步示例

### 5.1 普通同步

```bash
rsync -avh /data/ /backup/data/
```

### 5.2 镜像同步，目标端与源完全一致

```bash
rsync -avh --delete /data/ /backup/data/
```

先测试：

```bash
rsync -avhn --delete /data/ /backup/data/
```

### 5.3 保留权限、属主、时间、硬链接、ACL、xattr

```bash
rsync -aHAX --delete /data/ /backup/data/
```

说明：

- `-a`：保留权限、时间、属主、属组、符号链接等。
- `-H`：保留硬链接。
- `-A`：保留 ACL。
- `-X`：保留扩展属性。
- 非 root 用户可能无法保留属主/属组，可加 `--no-o --no-g`。

---

## 6. 远程同步：通过 SSH

### 6.1 推送到远程

```bash
rsync -avz /data/ user@192.168.1.10:/backup/data/
```

### 6.2 从远程拉取

```bash
rsync -avz user@192.168.1.10:/data/ /backup/data/
```

### 6.3 指定 SSH 端口

```bash
rsync -avz -e 'ssh -p 2222' /data/ user@host:/backup/data/
```

### 6.4 使用 SSH 密钥免密

```bash
ssh-keygen -t ed25519
ssh-copy-id user@host
rsync -avz /data/ user@host:/backup/data/
```

### 6.5 远程使用 sudo

远程用户需要能免密 sudo：

```bash
rsync -avz --rsync-path="sudo rsync" /data/ user@host:/backup/data/
```

### 6.6 常用组合

```bash
rsync -avzP --delete -e 'ssh -p 2222' /data/ user@host:/backup/data/
```

---

## 7. 排除和包含规则

### 7.1 排除单个文件或目录

```bash
rsync -avh --exclude='*.log' --exclude='.git/' --exclude='node_modules/' /src/ /dest/
```

### 7.2 从文件读取排除规则

```bash
cat > /etc/rsync-exclude.txt <<'EOF'
.git/
node_modules/
*.log
*.tmp
cache/
EOF

rsync -avh --exclude-from=/etc/rsync-exclude.txt /src/ /dest/
```

### 7.3 只包含某些文件

rsync 过滤规则按顺序匹配，先匹配先生效。

```bash
# 只同步 .jpg 和 .png，但保留目录结构
rsync -avh \
  --include='*/' \
  --include='*.jpg' \
  --include='*.png' \
  --exclude='*' \
  /photos/ /dest/
```

### 7.4 排除目录但保留目录本身

```bash
rsync -avh --exclude='cache/*' /src/ /dest/
```

### 7.5 过滤规则常用通配符

- `*`：匹配任意非 `/` 字符。
- `**`：匹配任意路径，包括 `/`。
- `?`：匹配单个字符。
- `[0-9]`：匹配字符范围。
- `/` 开头：锚定到传输根目录。
- `/` 结尾：只匹配目录。

---

## 8. 删除同步：--delete

让目标端和源端完全一致：

```bash
rsync -avh --delete /src/ /dest/
```

先模拟：

```bash
rsync -avhn --delete /src/ /dest/
```

相关选项：

```bash
--delete-before     # 传输前删除
--delete-during     # 传输中删除，默认
--delete-after      # 传输后删除
--delete-excluded   # 同时删除目标端被排除的文件，危险
--max-delete=N      # 最多删除 N 个文件
--prune-empty-dirs  # 删除空目录
```

注意：

- `--delete` 只影响目标端。
- 如果源路径写成 `/src` 而不是 `/src/`，删除行为可能不符合预期。
- 使用 `--delete-excluded` 要非常小心。

---

## 9. 断点续传、限速、进度

### 9.1 断点续传并显示进度

```bash
rsync -avhP /src/ /dest/
```

`-P` 等价于：

```bash
--partial --progress
```

推荐使用独立部分目录：

```bash
rsync -avh --partial-dir=.rsync-partial --progress /src/ /dest/
```

### 9.2 显示整体进度

```bash
rsync -avh --info=progress2 /src/ /dest/
```

### 9.3 限速

限制为约 5 MB/s：

```bash
rsync -avh --bwlimit=5000 /src/ user@host:/dest/
```

### 9.4 超时

```bash
rsync -avh --timeout=600 /src/ user@host:/dest/
```

---

## 10. 备份与增量快照

### 10.1 备份被覆盖或删除的文件

```bash
rsync -avh --delete \
  --backup \
  --backup-dir=/backup/old \
  --suffix=.bak \
  /src/ /dest/
```

说明：

- 覆盖或删除目标端文件前，先备份到 `/backup/old`。
- `--suffix=.bak` 指定备份后缀。

### 10.2 使用 --link-dest 做硬链接增量快照

第一次全量：

```bash
rsync -aH --delete /data/ /backup/2026-09-29/
```

创建 current 链接：

```bash
ln -sfn /backup/2026-09-29 /backup/current
```

下一次增量快照：

```bash
rsync -aH --delete --link-dest=/backup/current /data/ /backup/2026-09-30/
ln -sfn /backup/2026-09-30 /backup/current
```

未变化的文件会硬链接到上一份快照，节省空间。

---

## 11. 日志与统计

```bash
rsync -avh --log-file=/var/log/rsync.log /src/ /dest/
```

显示统计：

```bash
rsync -avh --stats /src/ /dest/
```

显示变化项：

```bash
rsync -avhi /src/ /dest/
```

`-i` 输出示例：

```text
>f+++++++++ file.txt
.d..t...... dir/
```

常见含义：

- `>f`：普通文件被传输。
- `+++++++++`：新文件。
- `.d..t......`：目录时间变化。
- `*deleting`：正在删除目标端文件。

---

## 12. rsync daemon 模式

rsync daemon 不依赖 SSH，使用 rsync 协议，默认端口 `873`。适合内网备份服务器。

### 12.1 服务端配置

编辑 `/etc/rsyncd.conf`：

```ini
uid = nobody
gid = nobody
use chroot = yes
max connections = 10
pid file = /var/run/rsyncd.pid
log file = /var/log/rsyncd.log
timeout = 600

[backup]
path = /srv/backup
comment = Backup Area
read only = no
list = yes
auth users = backupuser
secrets file = /etc/rsyncd.secrets
hosts allow = 192.168.1.0/24
```

创建密码文件：

```bash
echo 'backupuser:你的密码' | sudo tee /etc/rsyncd.secrets
sudo chmod 600 /etc/rsyncd.secrets
sudo chown root:root /etc/rsyncd.secrets
```

创建备份目录：

```bash
sudo mkdir -p /srv/backup
sudo chown nobody:nogroup /srv/backup
```

启动服务：

```bash
sudo systemctl enable --now rsync
sudo systemctl status rsync
```

或手动启动：

```bash
sudo rsync --daemon --config=/etc/rsyncd.conf
```

查看端口：

```bash
ss -tlnp | grep 873
```

防火墙：

```bash
# ufw
sudo ufw allow 873/tcp

# firewalld
sudo firewall-cmd --add-service=rsync --permanent
sudo firewall-cmd --reload
```

### 12.2 客户端使用 daemon

拉取：

```bash
rsync -avz backupuser@host::backup /local/backup/
```

推送：

```bash
rsync -avz /local/data/ backupuser@host::backup/data/
```

使用密码文件：

```bash
echo '你的密码' > /etc/rsync.pass
chmod 600 /etc/rsync.pass

rsync -avz --password-file=/etc/rsync.pass backupuser@host::backup /local/backup/
```

注意：

- daemon 密码文件在客户端只写密码，不写用户名。
- 服务端 `secrets file` 写 `用户名:密码`。
- 文件权限必须是 `600`。
- daemon 模式建议只在内网或 VPN 使用，或配合严格 `hosts allow`。

---

## 13. 文件列表同步

```bash
cat > list.txt <<'EOF'
file1.txt
dir1/file2.txt
dir2/
EOF

rsync -avh --files-from=list.txt /src/ /dest/
```

如果列表使用 NUL 分隔：

```bash
rsync -avh --from0 --files-from=list.txt /src/ /dest/
```

保留相对路径结构：

```bash
rsync -avhR /src/./dir1/file2.txt /dest/
# 结果：/dest/dir1/file2.txt
```

---

## 14. 权限、属主、ACL、xattr

```bash
# 保留权限、时间、属主、属组、设备、符号链接
rsync -a /src/ /dest/

# 保留硬链接
rsync -aH /src/ /dest/

# 保留 ACL 和 xattr
rsync -aAX /src/ /dest/

# 不保留属主/属组
rsync -a --no-o --no-g /src/ /dest/

# 强制设置权限
rsync -a --chmod=Du=rwx,Dgo=rx,Fu=rw,Fgo=r /src/ /dest/

# 强制设置属主属组
rsync -a --chown=www-data:www-data /src/ /dest/

# 使用数字 ID，不映射用户名
rsync -a --numeric-ids /src/ /dest/
```

---

## 15. 常见场景命令

### 15.1 本地目录镜像

```bash
rsync -avh --delete /data/ /backup/data/
```

### 15.2 远程增量备份

```bash
rsync -avzP --delete -e 'ssh -p 2222' /data/ user@host:/backup/data/
```

### 15.3 从远程拉取网站文件

```bash
rsync -avzP --delete user@host:/var/www/html/ /backup/www/
```

### 15.4 排除缓存和日志

```bash
rsync -avh --delete \
  --exclude='cache/' \
  --exclude='*.log' \
  --exclude='.git/' \
  /var/www/html/ /backup/www/
```

### 15.5 只同步新文件

```bash
rsync -avh --ignore-existing /src/ /dest/
```

### 15.6 只更新已存在文件

```bash
rsync -avh --existing /src/ /dest/
```

### 15.7 只同步比目标端更新的文件

```bash
rsync -avh --update /src/ /dest/
```

### 15.8 传输后删除源文件，类似移动

```bash
rsync -avh --remove-source-files /src/ /dest/
```

注意：只删除文件，不删除空目录。

### 15.9 限制最大文件

```bash
rsync -avh --max-size=100M /src/ /dest/
```

### 15.10 只同步小文件

```bash
rsync -avh --max-size=10M /src/ /dest/
```

---

## 16. 常见问题与排查

### 16.1 为什么目标多了一层目录？

因为源路径尾部没有 `/`。

```bash
rsync -av /src /dest/   # /dest/src/...
rsync -av /src/ /dest/  # /dest/...
```

### 16.2 为什么每次都在重新传？

rsync 默认比较文件大小和修改时间。可能原因：

- 源和目标 mtime 不一致。
- 文件系统不支持精确时间，如 FAT。
- 时区问题。
- 文件内容相同但大小不同。

解决：

```bash
rsync -avh --checksum /src/ /dest/
```

或只比较大小：

```bash
rsync -avh --size-only /src/ /dest/
```

### 16.3 --delete 没有删除文件？

检查：

- 源路径是否带 `/`。
- 是否被 `--exclude` 排除。
- 是否需要 `--delete-excluded`，但该选项危险。
- 是否只用了 `-n`。

### 16.4 Permission denied

解决方向：

- 使用 `sudo`。
- 远程使用 `--rsync-path="sudo rsync"`。
- 加 `--no-o --no-g` 跳过属主属组。
- 检查 SELinux、ACL、目录权限。

### 16.5 SSH 连接慢

可以尝试：

```bash
rsync -avz -e 'ssh -o GSSAPIAuthentication=no' /src/ user@host:/dest/
```

### 16.6 中文文件名乱码

可以尝试：

```bash
rsync -avh --iconv=UTF-8,GBK /src/ /dest/
```

### 16.7 保留硬链接

```bash
rsync -aH /src/ /dest/
```

### 16.8 保留 ACL 和 xattr

```bash
rsync -aAX /src/ /dest/
```

### 16.9 查看 rsync 退出码

常用退出码：

- `0`：成功。
- `23`：部分文件传输失败。
- `24`：源文件在传输中消失。
- `30`：超时。
- `35`：连接超时。

脚本中可判断：

```bash
rsync -avh /src/ /dest/
if [ $? -eq 0 ]; then
  echo "同步成功"
else
  echo "同步失败，退出码：$?"
fi
```

---

## 17. 最佳实践

1. 危险操作先 `-n` 测试。
2. 同步目录内容时，源路径加 `/`。
3. 备份脚本中明确使用 `--delete` 或明确不使用。
4. 远程同步优先使用 SSH 密钥。
5. daemon 模式只在内网或 VPN 使用，并设置 `hosts allow`。
6. 脚本中使用日志：

   ```bash
   rsync -avh --log-file=/var/log/rsync.log /src/ /dest/
   ```

7. 大目录备份考虑 `--link-dest` 做增量快照。
8. 定期验证备份可恢复性。
9. 保留 ACL/xattr 时使用 `-aAX`。
10. 保留硬链接时使用 `-aH`。

---

## 18. 速查表

```bash
# 本地镜像
rsync -avh --delete /src/ /dest/

# 模拟执行
rsync -avhn --delete /src/ /dest/

# SSH 推送
rsync -avzP --delete /src/ user@host:/dest/

# SSH 拉取
rsync -avzP --delete user@host:/src/ /dest/

# 指定 SSH 端口
rsync -avz -e 'ssh -p 2222' /src/ user@host:/dest/

# 排除
rsync -avh --exclude='*.log' --exclude='.git/' /src/ /dest/

# 限速约 5MB/s
rsync -avh --bwlimit=5000 /src/ /dest/

# 保留 ACL/xattr/硬链接
rsync -aHAX /src/ /dest/

# 增量快照
rsync -aH --delete --link-dest=/backup/current /data/ /backup/2026-09-30/

# daemon 拉取
rsync -avz user@host::module /dest/

# daemon 推送
rsync -avz /src/ user@host::module
```

掌握以上内容后，日常备份、服务器迁移、网站同步、远程镜像基本都能覆盖。最关键的三点：**源路径尾部斜杠、`--delete` 先 dry-run、远程优先用 SSH**。
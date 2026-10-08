下面是一份 Linux `sshd` 详细教程。`sshd` 是 OpenSSH 的服务端守护进程，负责接受 SSH 登录、SFTP、SCP、端口转发等连接。本文以 OpenSSH 为主，覆盖安装、配置、安全加固、密钥认证、SFTP chroot、Match 块、日志排查和最佳实践。

> 修改 `sshd_config` 前，务必保留一个已登录的 SSH 会话，并先执行 `sshd -t` 测试配置，避免把自己锁在服务器外。

---

## 1. 安装 OpenSSH Server

大多数 Linux 发行版默认已安装，若没有：

```bash
# Debian / Ubuntu
sudo apt update
sudo apt install openssh-server

# RHEL / CentOS / Rocky / AlmaLinux / Fedora
sudo dnf install openssh-server

# Arch Linux
sudo pacman -S openssh

# openSUSE
sudo zypper install openssh
```

查看版本：

```bash
ssh -V
sshd -V 2>/dev/null || true
```

---

## 2. 服务管理

不同发行版服务名可能不同：

- Debian / Ubuntu：`ssh.service`
- RHEL / CentOS / Rocky / AlmaLinux / Fedora：`sshd.service`

常用命令：

```bash
# 启动并设置开机自启
sudo systemctl enable --now sshd
# 或 Debian/Ubuntu
sudo systemctl enable --now ssh

# 停止
sudo systemctl stop sshd

# 重启
sudo systemctl restart sshd

# 重载配置，不断开现有连接
sudo systemctl reload sshd

# 查看状态
sudo systemctl status sshd

# 查看是否开机自启
sudo systemctl is-enabled sshd
```

如果不确定服务名：

```bash
systemctl list-units --type=service | grep -i ssh
```

查看监听端口：

```bash
ss -tlnp | grep sshd
# 或
ss -tlnp | grep :22
```

---

## 3. 配置文件位置

主配置文件：

```bash
/etc/ssh/sshd_config
```

新版 OpenSSH 支持 drop-in 目录：

```bash
/etc/ssh/sshd_config.d/*.conf
```

很多发行版在 `sshd_config` 顶部有：

```text
Include /etc/ssh/sshd_config.d/*.conf
```

注意：OpenSSH 对多数配置项采用“首次匹配生效”。如果 `Include` 在顶部，drop-in 文件里的配置可能优先于主文件后面的配置。建议用 `sshd -T` 查看最终生效值。

客户端配置：

```bash
/etc/ssh/ssh_config
~/.ssh/config
```

主机密钥：

```bash
/etc/ssh/ssh_host_rsa_key
/etc/ssh/ssh_host_ecdsa_key
/etc/ssh/ssh_host_ed25519_key
```

查看主机密钥指纹：

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

重新生成所有主机密钥：

```bash
sudo ssh-keygen -A
```

---

## 4. 测试配置

修改后必须先测试：

```bash
sudo sshd -t
```

如果没有输出，表示语法正确。查看最终生效配置：

```bash
sudo sshd -T | less
```

查看关键项：

```bash
sudo sshd -T | grep -E 'port|passwordauthentication|permitrootlogin|pubkeyauthentication|allowusers|allowgroups'
```

测试 Match 块最终结果：

```bash
sudo sshd -T -C user=alice,host=localhost,addr=127.0.0.1
```

调试模式前台运行，另开终端连接测试：

```bash
sudo /usr/sbin/sshd -d -p 2222
```

---

## 5. 常用 sshd_config 配置项

`sshd_config` 每行一个指令，`#` 开头为注释。

### 5.1 监听与端口

```conf
Port 22
# 可以监听多个端口
Port 2222

AddressFamily any
# 或 inet / inet6

ListenAddress 0.0.0.0
ListenAddress ::
# 只监听指定地址
# ListenAddress 192.168.1.10
```

### 5.2 主机密钥

```conf
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key
```

### 5.3 认证相关

```conf
PubkeyAuthentication yes
PasswordAuthentication yes
PermitEmptyPasswords no
PermitRootLogin prohibit-password
MaxAuthTries 6
MaxSessions 10
LoginGraceTime 2m

# 新版替代 ChallengeResponseAuthentication
KbdInteractiveAuthentication no
UsePAM yes
```

`PermitRootLogin` 可选值：

- `yes`：允许 root 密码登录，不推荐。
- `no`：完全禁止 root 登录。
- `prohibit-password`：允许 root 使用密钥登录，禁止密码。
- `forced-commands-only`：只允许 root 使用密钥且必须指定命令。

### 5.4 用户与组限制

```conf
AllowUsers alice bob@192.168.1.10
AllowGroups sshusers
DenyUsers root test
DenyGroups nogroup
```

如果设置了 `AllowUsers` 或 `AllowGroups`，只有匹配的用户/组可以登录。

### 5.5 会话与保活

```conf
ClientAliveInterval 300
ClientAliveCountMax 3
TCPKeepAlive yes
```

表示每 300 秒发送一次保活，连续 3 次无响应则断开。

### 5.6 转发与 X11

```conf
X11Forwarding no
AllowTcpForwarding yes
AllowAgentForwarding no
GatewayPorts no
PermitTunnel no
```

安全要求高时可禁用转发：

```conf
AllowTcpForwarding no
X11Forwarding no
AllowAgentForwarding no
PermitTunnel no
```

### 5.7 日志与信息

```conf
SyslogFacility AUTH
LogLevel INFO
PrintMotd yes
PrintLastLog yes
Banner /etc/issue.net
```

调试时可临时设为：

```conf
LogLevel VERBOSE
```

### 5.8 SFTP 子系统

```conf
Subsystem sftp internal-sftp
```

或使用外部 sftp-server：

```conf
# Debian/Ubuntu
Subsystem sftp /usr/lib/openssh/sftp-server

# RHEL/CentOS
Subsystem sftp /usr/libexec/openssh/sftp-server
```

做 chroot 时推荐 `internal-sftp`。

### 5.9 其他安全项

```conf
StrictModes yes
IgnoreRhosts yes
HostbasedAuthentication no
UseDNS no
GSSAPIAuthentication no
PermitUserEnvironment no
```

---

## 6. 密钥认证

### 6.1 客户端生成密钥

```bash
ssh-keygen -t ed25519 -a 100 -C "alice@example"
```

或 RSA：

```bash
ssh-keygen -t rsa -b 4096 -C "alice@example"
```

默认生成：

```bash
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

### 6.2 复制公钥到服务器

```bash
ssh-copy-id user@host
```

指定端口：

```bash
ssh-copy-id -p 2222 user@host
```

手动方式：

```bash
cat ~/.ssh/id_ed25519.pub
# 在服务器上执行
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo '公钥内容' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 6.3 服务端配置

```conf
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
StrictModes yes
```

### 6.4 权限要求

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

家目录不能组或其他用户可写：

```bash
chmod g-w,o-w /home/user
```

### 6.5 调试密钥登录

```bash
ssh -v user@host
```

查看服务端日志：

```bash
sudo journalctl -u sshd -f
# 或
sudo tail -f /var/log/auth.log
sudo tail -f /var/log/secure
```

---

## 7. 安全加固建议

### 7.1 基础加固配置示例

```conf
Port 22
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 4
LoginGraceTime 30
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no
GatewayPorts no
UseDNS no
GSSAPIAuthentication no
```

修改后：

```bash
sudo sshd -t
sudo systemctl reload sshd
```

### 7.2 修改默认端口

```conf
Port 2222
```

防火墙放行：

```bash
# ufw
sudo ufw allow 2222/tcp

# firewalld
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload
```

SELinux 需要放行非标准端口：

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
```

如果没有 `semanage`：

```bash
sudo dnf install policycoreutils-python-utils
```

### 7.3 限制用户

```conf
AllowUsers alice bob
AllowGroups sshusers
DenyUsers root
```

### 7.4 限制来源 IP

使用防火墙：

```bash
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
```

或在 `sshd_config` 中使用 Match：

```conf
Match Address 192.168.1.0/24
    PasswordAuthentication yes
```

### 7.5 Fail2ban

安装：

```bash
sudo apt install fail2ban
# 或
sudo dnf install fail2ban
```

配置 `/etc/fail2ban/jail.local`：

```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime = 1h
findtime = 10m
```

RHEL 系日志路径可能为：

```ini
logpath = /var/log/secure
```

启动：

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

### 7.6 双因素认证 2FA

可安装 Google Authenticator：

```bash
sudo apt install libpam-google-authenticator
```

配置 PAM 和 `sshd_config`：

```conf
UsePAM yes
KbdInteractiveAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
```

然后配置 `/etc/pam.d/sshd`。生产环境修改前务必测试。

---

## 8. Match 块

`Match` 可以根据用户、组、地址、主机等条件覆盖部分配置。全局配置必须放在 Match 之前。

示例：限制 backup 用户只能 SFTP：

```conf
Match User backup
    PasswordAuthentication no
    ForceCommand internal-sftp
    ChrootDirectory /backup/%u
    AllowTcpForwarding no
    X11Forwarding no
```

示例：限制某个组：

```conf
Match Group sftpusers
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

测试：

```bash
sudo sshd -t
sudo sshd -T -C user=backup,host=localhost,addr=127.0.0.1
```

---

## 9. SFTP chroot 配置

### 9.1 创建用户和目录

```bash
sudo groupadd sftpusers
sudo useradd -m -G sftpusers -s /usr/sbin/nologin alice
sudo passwd alice
```

创建 chroot 目录：

```bash
sudo mkdir -p /sftp/alice/uploads
sudo chown root:root /sftp /sftp/alice
sudo chmod 755 /sftp /sftp/alice
sudo chown alice:sftpusers /sftp/alice/uploads
sudo chmod 755 /sftp/alice/uploads
```

关键：`ChrootDirectory` 及其上级目录必须由 root 拥有，且组和其他用户不可写。

### 9.2 配置 sshd

```conf
Subsystem sftp internal-sftp

Match Group sftpusers
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
    PasswordAuthentication yes
```

重载：

```bash
sudo sshd -t
sudo systemctl reload sshd
```

测试：

```bash
sftp alice@host
```

---

## 10. 端口转发与 SSH 隧道

服务端相关配置：

```conf
AllowTcpForwarding yes
GatewayPorts no
PermitOpen any
PermitListen any
```

限制目标：

```conf
Match User alice
    PermitOpen db.internal:3306
    PermitListen 127.0.0.1:8080
```

客户端示例：

```bash
# 本地转发：访问本地 8080 等于访问远程 80
ssh -L 8080:localhost:80 user@host

# 远程转发：远程 8080 转发到本地 80
ssh -R 8080:localhost:80 user@host

# 动态 SOCKS 代理
ssh -D 1080 user@host
```

安全要求高时：

```conf
AllowTcpForwarding no
X11Forwarding no
AllowAgentForwarding no
PermitTunnel no
```

---

## 11. 日志与审计

日志位置：

```bash
# systemd 系统
sudo journalctl -u sshd
sudo journalctl -u ssh

# Debian/Ubuntu
/var/log/auth.log

# RHEL/CentOS
/var/log/secure
```

实时查看：

```bash
sudo journalctl -u sshd -f
sudo tail -f /var/log/auth.log
sudo tail -f /var/log/secure
```

查看失败登录：

```bash
sudo grep 'Failed password' /var/log/auth.log
sudo grep 'Failed password' /var/log/secure
```

查看成功登录：

```bash
last
lastlog
who
w
```

提高日志级别：

```conf
LogLevel VERBOSE
```

可记录密钥指纹，便于审计。

---

## 12. 故障排查

### 12.1 配置语法错误

```bash
sudo sshd -t
```

### 12.2 服务启动失败

```bash
sudo systemctl status sshd
sudo journalctl -xe
sudo journalctl -u sshd --no-pager
```

### 12.3 端口未监听

```bash
ss -tlnp | grep sshd
sudo ss -tlnp | grep :22
```

### 12.4 防火墙拦截

```bash
# ufw
sudo ufw status
sudo ufw allow OpenSSH

# firewalld
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

### 12.5 SELinux

```bash
getenforce
sudo ausearch -m avc -ts recent
```

非标准端口：

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
```

### 12.6 认证失败

检查：

- 密码是否正确。
- 密钥权限是否正确。
- `AllowUsers` / `AllowGroups` 是否限制。
- `PasswordAuthentication` 是否被禁用。
- `PermitRootLogin` 是否禁止。
- PAM 配置是否正常。
- 日志中的具体原因。

### 12.7 连接超时

可能原因：

- 网络不通。
- 防火墙丢包。
- `ListenAddress` 绑定错误。
- `UseDNS yes` 导致反向解析慢。
- 路由或安全组问题。

建议：

```conf
UseDNS no
```

### 12.8 主机密钥变更警告

客户端删除旧记录：

```bash
ssh-keygen -R host
```

### 12.9 无法使用 SFTP

检查：

```bash
sudo sshd -T | grep subsystem
sudo sshd -T | grep -i sftp
```

chroot 目录权限是否正确。

---

## 13. 客户端配置简化

`~/.ssh/config`：

```conf
Host myserver
    HostName 192.168.1.10
    User alice
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
```

连接：

```bash
ssh myserver
sftp myserver
```

---

## 14. 最佳实践

1. 修改前备份：

   ```bash
   sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
   ```

2. 保持一个已登录会话，再修改配置。
3. 每次修改后执行：

   ```bash
   sudo sshd -t
   sudo systemctl reload sshd
   ```

4. 使用密钥认证，禁用密码登录。
5. 禁止 root 直接登录：

   ```conf
   PermitRootLogin no
   ```

6. 限制用户和来源 IP。
7. 关闭不必要的转发和 X11。
8. 使用 Fail2ban 防暴力破解。
9. 定期更新 OpenSSH。
10. 监控 `/var/log/auth.log` 或 `/var/log/secure`。
11. 非标准端口可减少扫描，但不能替代安全加固。
12. 生产环境考虑跳板机、VPN、SSH 证书。

---

## 15. 速查表

```bash
# 安装
sudo apt install openssh-server
sudo dnf install openssh-server

# 启动并自启
sudo systemctl enable --now sshd
sudo systemctl enable --now ssh

# 测试配置
sudo sshd -t

# 查看最终配置
sudo sshd -T

# 重载
sudo systemctl reload sshd

# 查看状态
sudo systemctl status sshd

# 查看端口
sudo ss -tlnp | grep sshd

# 生成密钥
ssh-keygen -t ed25519 -a 100

# 复制公钥
ssh-copy-id user@host

# 调试连接
ssh -v user@host

# 查看日志
sudo journalctl -u sshd -f
sudo tail -f /var/log/auth.log
sudo tail -f /var/log/secure

# 防火墙放行
sudo ufw allow OpenSSH
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload

# SELinux 非标准端口
sudo semanage port -a -t ssh_port_t -p tcp 2222
```

掌握以上内容后，基本可以完成 `sshd` 的安装、配置、加固、SFTP chroot、端口转发、日志审计和故障排查。最关键的三点：**修改前先测试 `sshd -t`、保留一个已登录会话、生产环境优先密钥认证并禁用密码和 root 直接登录。**
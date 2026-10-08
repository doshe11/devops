
在 Debian 12 上部署 Postfix + Dovecot 邮件服务器

## 1. 系统准备和更新

### 1.1 更新系统

```bash
apt update && sudo apt upgrade -y
apt install curl wget vim net-tools -y
```

### 1.2 设置主机名

```bash
# 设置主机名
hostnamectl set-hostname mail.example.com
vim /etc/hostname
# 内容: mail.example.com

# 编辑hosts文件
vim /etc/hosts
# 添加以下内容:
127.0.0.1 localhost
127.0.1.1 mail.example.com mail

# 重新应用主机名
hostname -F /etc/hostname
```

## 2. 安装 Postfix

### 2.1 安装 Postfix 和相关包

```bash
apt install postfix postfix-pcre postfix-mysql -y
```

### 2.2 配置 Postfix

在安装过程中会出现配置界面，选择以下选项：

- **邮件配置类型**: Internet Site
- **系统邮件名称**: [example.com](https://example.com/) (替换为你的域名)

### 2.3 基本 Postfix 配置

```bash
vim /etc/postfix/main.cf
```

修改或添加以下内容：
```bash
# 基本设置
myhostname = mail.example.com
mydomain = example.com
myorigin = $mydomain
inet_interfaces = all
inet_protocols = all

# 网络设置
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128

# 限制设置
smtpd_recipient_restrictions = 
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination

# TLS设置
smtpd_tls_cert_file = /etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file = /etc/ssl/private/ssl-cert-snakeoil.key
smtpd_use_tls = yes
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

# SASL认证
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname

# 邮箱设置
home_mailbox = Maildir/
mailbox_command =
```

## 3. 安装和配置 Dovecot

### 3.1 安装 Dovecot

```bash
apt install dovecot-core dovecot-imapd dovecot-pop3d dovecot-lmtpd dovecot-sqlite -y
```
### 3.2 配置 Dovecot
`
```bash
vim /etc/dovecot/dovecot.conf
```
确保包含以下内容：
```bash
# 启用协议
protocols = imap pop3 lmtp

# 监听地址
listen = *, ::

# 禁用SSLv2和SSLv3
ssl_protocols = !SSLv2 !SSLv3
```

### 3.3 配置认证

```bash
vim /etc/dovecot/conf.d/10-auth.conf
```

修改以下设置：
```bash
# 禁用明文认证（除非在SSL/TLS上）
disable_plaintext_auth = yes

# 认证机制
auth_mechanisms = plain login

# 用户数据库
!include auth-system.conf.ext
```

### 3.4 配置邮件存储

```bash
vim /etc/dovecot/conf.d/10-mail.conf
```

修改以下设置：
```bash
# 邮件存储格式
mail_location = maildir:~/Maildir

# 命名空间
namespace inbox {
  inbox = yes
}
```

### 3.5 配置SSL/TLS

```bash
vim /etc/dovecot/conf.d/10-ssl.conf
```

修改以下设置：
```bash
# 启用SSL
ssl = required

# 证书路径（暂时使用自签名证书，后面会配置Let's Encrypt）
ssl_cert = </etc/ssl/certs/dovecot.pem
ssl_key = </etc/ssl/private/dovecot.key
```

### 3.6 创建自签名证书（临时）

```bash
openssl req -new -x509 -days 3650 -nodes \
  -out /etc/ssl/certs/dovecot.pem \
  -keyout /etc/ssl/private/dovecot.key \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=mail.example.com"
chmod 600 /etc/ssl/private/dovecot.key
```

## 4. 集成 Postfix 和 Dovecot

### 4.1 配置 Postfix 使用 Dovecot 进行认证

```bash
vim /etc/postfix/main.cf
```

添加以下内容：
```bash
# 使用Dovecot进行SASL认证
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_authenticated_header = yes
```

### 4.2 配置 Dovecot 提供认证服务

```bash
vim /etc/dovecot/conf.d/10-master.conf
```

找到 `service auth` 部分并修改：
```bash
service auth {
  # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}
```

## 5. 创建邮件用户

### 5.1 创建系统用户

```bash
# 创建邮件用户组
groupadd -g 5000 vmail
useradd -g vmail -u 5000 vmail -d /var/vmail -m -s /usr/sbin/nologin

# 创建用户邮箱目录
mkdir -p /var/vmail/example.com
chown -R vmail:vmail /var/vmail
chmod -R 770 /var/vmail
```

### 5.2 配置 Dovecot 使用系统用户

```bash
vim /etc/dovecot/conf.d/10-auth.conf
```

确保以下设置：
```bash
auth_mechanisms = plain login
!include auth-system.conf.ext
```

## 6. 防火墙配置

### 6.1 配置防火墙规则

```bash
# 安装UFW防火墙
apt install ufw -y

# 配置规则
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 25/tcp    # SMTP
ufw allow 587/tcp   # Submission
ufw allow 465/tcp   # SMTPS
ufw allow 110/tcp   # POP3
ufw allow 995/tcp   # POP3S
ufw allow 143/tcp   # IMAP
ufw allow 993/tcp   # IMAPS
ufw enable
```

## 7. 配置 Let's Encrypt SSL 证书

### 7.1 安装 Certbot

```bash
apt install certbot -y
```

### 7.2 获取证书

```bash
# 确保80端口开放用于验证
ufw allow 80/tcp

# 获取证书
certbot certonly --standalone -d mail.example.com --non-interactive --agree-tos --email admin@example.com

# 更新防火墙规则
ufw delete allow 80/tcp
```

### 7.3 配置 Postfix 使用新证书

```bash
vim /etc/postfix/main.cf
```

更新证书路径：
```bash
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.example.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.example.com/privkey.pem
```

### 7.4 配置 Dovecot 使用新证书

```bash
vim /etc/dovecot/conf.d/10-ssl.conf
```

更新证书路径：
```bash
ssl_cert = </etc/letsencrypt/live/mail.example.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.example.com/privkey.pem
```

## 8. 测试配置

### 8.1 重启服务

```bash
systemctl restart postfix
systemctl restart dovecot
systemctl enable postfix
systemctl enable dovecot
```

### 8.2 检查服务状态

```bash
systemctl status postfix
systemctl status dovecot
netstat -tulpn | grep -E ':25|:110|:143|:587|:993|:995'
```

### 8.3 测试SMTP连接

```bash
# 测试本地SMTP
telnet localhost 25
# 输入: EHLO localhost
# 应该看到250-STARTTLS

# 测试认证
openssl s_client -connect localhost:587 -starttls smtp
# 输入: EHLO localhost
```

### 8.4 测试IMAP/POP3连接

```bash
# 测试IMAP
openssl s_client -connect localhost:993 -crlf
# 输入: a login username password

# 测试POP3
openssl s_client -connect localhost:995 -crlf
# 输入: USER username
# 输入: PASS password
```

## 9. 客户端配置示例

### 9.1 邮件客户端设置

```bash
收件服务器 (IMAP):
- 服务器: mail.example.com
- 端口: 993
- 加密: SSL/TLS
- 认证: 普通密码

发件服务器 (SMTP):
- 服务器: mail.example.com
- 端口: 587
- 加密: STARTTLS
- 认证: 普通密码
```


端口映射/端口转发在 Linux 里通常分几类：**SSH 隧道、内核 NAT 转发、用户态转发工具、容器/K8s 端口发布**。下面按场景给命令。你前面在看 `sshd` 和 `rsync`，所以先讲 SSH 端口映射。

---

## 1. SSH 端口映射/隧道

SSH 端口转发不依赖额外软件，加密传输，适合临时访问内网服务。

### 1.1 本地转发：`-L`

把**本地端口**转发到**远程服务器能访问的目标**。

```bash
ssh -L [本地绑定地址:]本地端口:目标主机:目标端口 用户@SSH服务器
```

例：通过跳板机访问内网 Web：

```bash
ssh -N -L 8080:192.168.1.100:80 user@jump-server
```

访问本机：

```bash
curl http://127.0.0.1:8080
```

表示：本地 `127.0.0.1:8080` → 跳板机视角的 `192.168.1.100:80`。

常用参数：

```bash
-N    # 不执行远程命令，只做转发
-f    # 后台运行
-C    # 压缩
-g    # 允许其他主机连接本地转发端口
```

例：允许其他机器访问本地 8080：

```bash
ssh -N -f -g -L 0.0.0.0:8080:192.168.1.100:80 user@jump-server
```

### 1.2 远程转发：`-R`

把**远程服务器端口**转发到**本地或本地能访问的目标**。

```bash
ssh -R [远程绑定地址:]远程端口:目标主机:目标端口 用户@SSH服务器
```

例：把本地 80 暴露到远程服务器的 8080：

```bash
ssh -N -R 8080:localhost:80 user@public-server
```

在 public-server 上访问：

```bash
curl http://127.0.0.1:8080
```

默认只绑定远程 `127.0.0.1`。如果要让外部访问，需要服务端 `sshd_config`：

```conf
GatewayPorts yes
# 或
GatewayPorts clientspecified
```

然后客户端指定绑定地址：

```bash
ssh -N -R 0.0.0.0:8080:localhost:80 user@public-server
```

### 1.3 动态转发：`-D`

创建 SOCKS5 代理：

```bash
ssh -N -f -D 1080 user@server
```

浏览器或程序设置 SOCKS5 代理 `127.0.0.1:1080`。

### 1.4 sshd 服务端相关配置

`/etc/ssh/sshd_config`：

```conf
AllowTcpForwarding yes
# 可限制：local / remote / no
GatewayPorts no
# yes / no / clientspecified
PermitOpen any
# 或 PermitOpen db.internal:3306
PermitListen any
```

限制某用户只能转发到指定目标：

```conf
Match User alice
    PermitOpen db.internal:3306
    PermitListen 127.0.0.1:8080
```

修改后：

```bash
sudo sshd -t
sudo systemctl reload sshd
```

### 1.5 自动重连

```bash
autossh -M 0 -f -N -o ServerAliveInterval=60 -L 8080:192.168.1.100:80 user@jump
```

---



## 2. iptables NAT 端口映射

适合把公网端口映射到内网服务器。

假设：

- 网关公网 IP：`203.0.113.10`
- 内网目标：`192.168.1.100:80`
- 映射公网端口：`8080`

2.1 开启 IP 转发

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

永久：

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-ipforward.conf
sudo sysctl --system
```

### 2.2 添加 DNAT

```bash
sudo iptables -t nat -A PREROUTING \
  -p tcp -d 203.0.113.10 --dport 8080 \
  -j DNAT --to-destination 192.168.1.100:80
```

### 2.3 添加 SNAT/MASQUERADE

```bash
sudo iptables -t nat -A POSTROUTING \
  -p tcp -d 192.168.1.100 --dport 80 \
  -j MASQUERADE
```

### 2.4 放行 FORWARD

```bash
sudo iptables -A FORWARD -p tcp -d 192.168.1.100 --dport 80 -j ACCEPT
```

### 2.5 保存规则

Debian/Ubuntu：

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

RHEL/CentOS 旧版：

```bash
sudo service iptables save
```

查看：

```bash
sudo iptables -t nat -L -n -v
```

---

## 3. nftables 端口映射

现代系统推荐 nftables。

```bash
sudo nft add table ip nat
sudo nft add chain ip nat prerouting '{ type nat hook prerouting priority dstnat; policy accept; }'
sudo nft add chain ip nat postrouting '{ type nat hook postrouting priority srcnat; policy accept; }'

sudo nft add rule ip nat prerouting tcp dport 8080 dnat to 192.168.1.100:80
sudo nft add rule ip nat postrouting ip daddr 192.168.1.100 tcp dport 80 masquerade
```

永久保存到 `/etc/nftables.conf`：

```bash
sudo nft list ruleset | sudo tee /etc/nftables.conf
sudo systemctl enable --now nftables
```

---

## 4. firewalld 端口映射

```bash
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --permanent \
  --add-forward-port=port=8080:proto=tcp:toport=80:toaddr=192.168.1.100
sudo firewall-cmd --reload
```

查看：

```bash
sudo firewall-cmd --list-all
```

---

## 5. socat 用户态端口转发

临时转发最简单：

```bash
sudo socat TCP-LISTEN:8080,fork,reuseaddr TCP:192.168.1.100:80
```

UDP：

```bash
sudo socat UDP-LISTEN:5353,fork UDP:192.168.1.100:53
```

后台 systemd 示例 `/etc/systemd/system/socat-web.service`：

```ini
[Unit]
Description=socat forward 8080 to 192.168.1.100:80
After=network.target

[Service]
ExecStart=/usr/bin/socat TCP-LISTEN:8080,fork,reuseaddr TCP:192.168.1.100:80
Restart=always

[Install]
WantedBy=multi-user.target
```

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now socat-web
```

---


## 6. Docker 端口映射

```bash
docker run -d --name web -p 8080:80 nginx
```

只监听本机：

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

UDP：

```bash
docker run -d -p 8080:80/udp nginx
```

查看：

```bash
docker port web
```

注意：Docker 会修改 iptables，可能绕过 ufw 规则。生产环境要限制绑定地址和防火墙。

---

## 7. Kubernetes 端口转发

```bash
kubectl port-forward pod/mypod 8080:80
kubectl port-forward svc/mysvc 8080:80
kubectl port-forward svc/mysvc 8080:80 --address 0.0.0.0
```

仅用于调试，不适合长期暴露。

---

## 8. 测试与排错

查看监听：

```bash
ss -tlnp | grep :8080
```

测试端口：

```bash
nc -vz 127.0.0.1 8080
curl -v http://127.0.0.1:8080
```

抓包：

```bash
sudo tcpdump -i any port 8080
```

检查转发：

```bash
sysctl net.ipv4.ip_forward
sudo iptables -t nat -L -n -v
sudo nft list ruleset
sudo firewall-cmd --list-all
```

SELinux：

```bash
getenforce
sudo ausearch -m avc -ts recent
```

非标准端口可能需要：

```bash
sudo semanage port -a -t http_port_t -p tcp 8080
```

---

## 9. 安全建议

1. 能绑 `127.0.0.1` 就不要绑 `0.0.0.0`。
2. 公网映射必须限制来源 IP。
3. 不要把数据库、Redis、Docker API 直接映射到公网。
4. SSH 隧道优先于裸暴露端口。
5. Docker `-p` 会改 iptables，注意 ufw 可能不生效。
6. 定期检查：

```bash
ss -tlnp
```


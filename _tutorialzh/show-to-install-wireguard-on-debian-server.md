---
title: "How to setup Wireguard on Debian server"
layout: default 
---

### **安装 WireGuard**

```bash
# Debian 上安装
sudo apt update
sudo apt install wireguard
```

### **生成密钥**

```bash
# 生成服务器密钥对
wg genkey | tee server_private.key | wg pubkey > server_public.key

# 生成客户端密钥对
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

### **服务器配置**

创建 `/etc/wireguard/wg0.conf`：

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 443
PrivateKey = <server_private.key 的内容>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public.key 的内容>
AllowedIPs = 10.0.0.2/32
```

**注意**：将 `eth0` 改成你的实际网络接口名（用 `ip a` 查看）

### **启用 IP 转发**

```bash
# 临时启用
sudo sysctl -w net.ipv4.ip_forward=1

# 永久启用
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### **启动服务**

```bash
# 启动 WireGuard
sudo wg-quick up wg0

# 设置开机自启
sudo systemctl enable wg-quick@wg0
```

### **客户端配置**

创建客户端配置文件：

```ini
[Interface]
Address = 10.0.0.2/24
PrivateKey = <client_private.key 的内容>
DNS = 8.8.8.8

[Peer]
PublicKey = <server_public.key 的内容>
Endpoint = YOUR_SERVER_IP:443
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

### **防火墙设置**

```bash
# 允许 UDP 443 端口
sudo ufw allow 443/udp
# 或使用 iptables
sudo iptables -A INPUT -p udp --dport 443 -j ACCEPT
```

### **验证运行状态**

```bash
sudo wg show
```

整个过程 10-15 分钟就能完成。客户端可以用 WireGuard 官方应用导入配置文件即可连接。

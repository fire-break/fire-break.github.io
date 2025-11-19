---
title: "How to setup Wireguard on Debian server"
layout: default 
---

# WireGuard VPN Setup Guide

A step-by-step guide to setting up WireGuard VPN on a Debian server.

## Prerequisites

Before you begin, ensure you have:
- A VPS running Debian 10 or later (Ubuntu also works)
- Root or sudo access to the server
- Basic familiarity with SSH and command line
- A client device (Windows, macOS, Linux, iOS, or Android)

## Step 1: Install WireGuard

SSH into your server and install WireGuard:

```bash
# Update package list
sudo apt update

# Install WireGuard
sudo apt install wireguard
```

## Step 2: Generate Key Pairs

Generate cryptographic keys for both server and client:

```bash
# Generate server keys
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

# Generate client keys
wg genkey | sudo tee /etc/wireguard/client_private.key | wg pubkey | sudo tee /etc/wireguard/client_public.key

# Secure the private keys
sudo chmod 600 /etc/wireguard/server_private.key
sudo chmod 600 /etc/wireguard/client_private.key
```

## Step 3: Configure the Server

Create the server configuration file:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Paste the following configuration:

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <paste server_private.key content here>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <paste client_public.key content here>
AllowedIPs = 10.0.0.2/32
```

**Important notes:**
- Replace `<paste server_private.key content here>` with the content of `/etc/wireguard/server_private.key`
- Replace `<paste client_public.key content here>` with the content of `/etc/wireguard/client_public.key`
- Replace `eth0` with your actual network interface name (check with `ip a`)

**To view key contents:**
```bash
sudo cat /etc/wireguard/server_private.key
sudo cat /etc/wireguard/client_public.key
```

### Port Selection and Censorship Considerations

The default WireGuard port is **51820**. However, if you need to evade detection by network censorship systems (such as the Great Firewall), you can configure WireGuard to use **port 443** instead:

**To use port 443:**
Change the `ListenPort` line to:
```ini
ListenPort = 443
```

**Why port 443?**
- Port 443 is the standard HTTPS port
- Blocking port 443 would disrupt all HTTPS traffic (websites, apps, etc.)
- It's less likely to be targeted by simple port-based filtering
- However, note that deep packet inspection (DPI) can still potentially identify WireGuard traffic patterns

**Important:** If you use port 443, make sure no other service (like a web server) is already using that port.

## Step 4: Enable IP Forwarding

Allow the server to forward packets:

```bash
# Enable IP forwarding temporarily
sudo sysctl -w net.ipv4.ip_forward=1

# Make it permanent
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Step 5: Configure Firewall

Allow WireGuard traffic through the firewall:

```bash
# For default port (51820)
sudo ufw allow 51820/udp

# Or for port 443 (if using that)
sudo ufw allow 443/udp

# Alternative: using iptables
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
```

## Step 6: Start WireGuard Service

Start WireGuard and enable it to run on boot:

```bash
# Start WireGuard
sudo systemctl start wg-quick@wg0

# Enable automatic start on boot
sudo systemctl enable wg-quick@wg0

# Check status
sudo systemctl status wg-quick@wg0
```

**Verify it's running:**
```bash
sudo wg show
```

You should see output showing your interface, public key, listening port, and peer information.

## Step 7: Configure the Client

Create a client configuration file on your local machine. The location depends on your operating system:

- **Windows**: `C:\Program Files\WireGuard\Data\Configurations\wg0.conf`
- **macOS/Linux**: `~/wg0.conf` or `/etc/wireguard/wg0.conf`
- **Mobile**: You can create the file anywhere and import it via the app

**Client configuration (`wg0.conf`):**

```ini
[Interface]
Address = 10.0.0.2/24
PrivateKey = <paste client_private.key content here>
DNS = 8.8.8.8

[Peer]
PublicKey = <paste server_public.key content here>
Endpoint = YOUR_SERVER_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

**Important notes:**
- Replace `<paste client_private.key content here>` with the content from the server's `/etc/wireguard/client_private.key`
- Replace `<paste server_public.key content here>` with the content from the server's `/etc/wireguard/server_public.key`
- Replace `YOUR_SERVER_IP` with your actual server's IP address
- If you changed the server to port 443, update the `Endpoint` to `YOUR_SERVER_IP:443`

**To retrieve keys from the server:**
```bash
sudo cat /etc/wireguard/client_private.key
sudo cat /etc/wireguard/server_public.key
```

## Step 8: Install WireGuard Client

Download and install the WireGuard client for your platform:

- **Windows**: Download from [wireguard.com](https://www.wireguard.com/install/)
- **macOS**: Download from the App Store or [wireguard.com](https://www.wireguard.com/install/)
- **Linux**: `sudo apt install wireguard` (Debian/Ubuntu)
- **iOS**: Download from the App Store
- **Android**: Download from Google Play Store

## Step 9: Connect to VPN

### Desktop (Windows/macOS/Linux)
1. Open the WireGuard application
2. Click "Add Tunnel" or "Import tunnel(s) from file"
3. Select your `wg0.conf` file
4. Click "Activate"

### Mobile (iOS/Android)
1. Open the WireGuard app
2. Tap the "+" button
3. Select "Create from file or archive" or scan a QR code
4. Import your configuration file
5. Toggle the connection on

### Linux Command Line
```bash
sudo wg-quick up wg0

# To disconnect
sudo wg-quick down wg0
```

## Step 10: Verify Connection

Once connected, verify your VPN is working:

**Check your IP address:**
- Visit [https://whatismyipaddress.com](https://whatismyipaddress.com)
- You should see your server's IP address, not your local IP

**Test DNS:**
```bash
nslookup google.com
```

**Check connection status (on server):**
```bash
sudo wg show
```

You should see your client listed under peers with data transfer statistics.

## Troubleshooting

### Cannot connect to server

**Check server status:**
```bash
sudo systemctl status wg-quick@wg0
sudo wg show
```

**Check if port is listening:**
```bash
sudo netstat -tulpn | grep 51820
# Or if using port 443:
sudo netstat -tulpn | grep 443
```

**Check firewall:**
```bash
sudo ufw status
# Make sure your WireGuard port is allowed
```

### Connection established but no internet

**Verify IP forwarding:**
```bash
sysctl net.ipv4.ip_forward
# Should return: net.ipv4.ip_forward = 1
```

**Check iptables rules:**
```bash
sudo iptables -L -v -n
sudo iptables -t nat -L -v -n
```

**Verify network interface name in config:**
```bash
ip a
# Make sure the interface name in PostUp/PostDown matches your actual interface
```

### Time synchronization errors

WireGuard requires accurate time synchronization. If you see authentication errors:

```bash
# Check server time
date

# Install NTP client
sudo apt install systemd-timesyncd

# Enable time sync
sudo timedatectl set-ntp true

# Verify
timedatectl status
```

### Server logs

View WireGuard logs for debugging:

```bash
# View system logs
sudo journalctl -u wg-quick@wg0 -f

# Or check kernel messages
sudo dmesg | grep wireguard
```

## Managing WireGuard Service

**Stop WireGuard:**
```bash
sudo wg-quick down wg0
# Or
sudo systemctl stop wg-quick@wg0
```

**Start WireGuard:**
```bash
sudo wg-quick up wg0
# Or
sudo systemctl start wg-quick@wg0
```

**Restart WireGuard:**
```bash
sudo systemctl restart wg-quick@wg0
```

**Disable autostart:**
```bash
sudo systemctl disable wg-quick@wg0
```

## Adding Additional Clients

To add more clients, repeat the key generation and add another `[Peer]` section to the server config:

```bash
# Generate new client keys
wg genkey | sudo tee /etc/wireguard/client2_private.key | wg pubkey | sudo tee /etc/wireguard/client2_public.key
```

**Add to server config:**
```ini
[Peer]
PublicKey = <client2_public.key content>
AllowedIPs = 10.0.0.3/32
```

**Restart WireGuard:**
```bash
sudo systemctl restart wg-quick@wg0
```

## Security Best Practices

1. **Keep private keys secure**: Never share private keys. Only public keys should be exchanged.
2. **Use strong firewall rules**: Only allow necessary ports.
3. **Regular updates**: Keep WireGuard and your system updated:
   ```bash
   sudo apt update && sudo apt upgrade
   ```
4. **Monitor connections**: Regularly check connected peers:
   ```bash
   sudo wg show
   ```
5. **Limit peer access**: Only add trusted devices as peers.

## Performance Tips

- WireGuard is designed to be fast and efficient
- For maximum performance, ensure your server has adequate CPU and bandwidth
- Monitor resource usage:
  ```bash
  htop
  vnstat
  ```

## Advanced: Using Port 443 with Other Services

If you need to run a web server alongside WireGuard on port 443:

**Option 1:** Use WireGuard on a different port and set up port forwarding
**Option 2:** Use WireGuard over WebSocket (more complex, requires additional setup)
**Option 3:** Run web server on a different port and use WireGuard on 443

For most use cases focused on evading censorship, dedicating port 443 to WireGuard is the simplest and most effective approach.

## Conclusion

You now have a working WireGuard VPN. Remember:
- Standard port: 51820 (good for general use)
- Censorship-resistant port: 443 (reduces detection likelihood)
- Keep your configuration files and private keys secure
- Regularly update your system and WireGuard

For more information, visit the [official WireGuard documentation](https://www.wireguard.com/).

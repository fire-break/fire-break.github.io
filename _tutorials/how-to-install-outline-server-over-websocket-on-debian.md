---
title: "How to Install Outline Server with WebSocket on Debian"
layout: default 
---

### **How to Install Outline Server with WebSocket on Debian**

This tutorial will guide you through setting up the `outline-ss-server` to listen for WebSocket connections on a private port and using Nginx to proxy public traffic from port 443 to the service. This setup enhances security and allows you to run other web services on the same server.

#### **Prerequisites**

1.  **A Debian Server**: A fresh installation of Debian 11 or 12 is recommended. (Ubuntu 22.04, 24.04, or later versions also work.)
2.  **A Domain Name**: A domain (e.g., `your.domain.com`) with its A/AAAA record pointing to your server's public IP address.
3.  **Sudo User Access**: You must be able to log in and execute commands as a user with `sudo` privileges.
4.  **Required Packages**: Install Nginx and Certbot.
    ```bash
    sudo apt update
    sudo apt install nginx python3-certbot-nginx -y
    ```

---

### **Step 1: Download and Install `outline-ss-server`**

We will fetch the latest pre-compiled binary from the official GitHub releases, extract it, and place it in a system-wide executable path.

1.  **Find the Latest Release URL**
    Go to the [outline-ss-server GitHub Releases page](https://github.com/Jigsaw-Code/outline-ss-server/releases). Find the latest version and right-click on the `...linux_x86_64.tar.gz` file to copy its link address.

2.  **Download and Extract the Archive**
    Use `wget` to download the file. **Replace the URL below with the link you just copied.**

    ```bash
    # Example URL. Make sure to use the latest version!
    wget https://github.com/Jigsaw-Code/outline-ss-server/releases/download/v1.9.2/outline-ss-server_1.9.2_linux_x86_64.tar.gz
    ```

    Now, extract the executable from the archive:
    ```bash
    # Replace the filename with the one you downloaded
    tar -xzf outline-ss-server_1.9.2_linux_x86_64.tar.gz
    ```
    This will create a file named `outline-ss-server` in your current directory.

3.  **Install the Binary**
    Move the extracted executable to `/usr/local/bin` and make it executable.
    ```bash
    sudo mv outline-ss-server /usr/local/bin/
    sudo chmod +x /usr/local/bin/outline-ss-server
    ```

4.  **Clean Up (Optional)**
    You can now remove the downloaded archive.
    ```bash
    # Replace the filename with the one you downloaded
    rm outline-ss-server_1.9.2_linux_x86_64.tar.gz
    ```

---

### **Step 2: Create the Outline Server Configuration**

We need a `config.yaml` file to tell the server how to listen and what keys to use.

1.  **Create a Configuration Directory**:
    ```bash
    sudo mkdir -p /etc/outline-ss-server
    ```

2.  **Generate a Secure Secret**:
    Use `openssl` to generate a strong, random secret for your service.
    ```bash
    openssl rand -base64 16
    ```
    This will output a random string like `xYzc2vR+aB1eF5gHjK9LqQ==`. **Copy your unique generated string** to use in the next step.

3.  **Create the `config.yaml` file**:
    Use a text editor like `nano` to create the configuration file.
    ```bash
    sudo nano /etc/outline-ss-server/config.yaml
    ```
    Paste the following content into the file. **Remember to replace the `secret` value with the one you generated.**

    ```yaml
    # /etc/outline-ss-server/config.yaml

    web:
      servers:
        # Defines an internal web server, not exposed to the internet.
        - id: server1
          listen:
            # Listens on a local-only, non-privileged port for security.
            - "127.0.0.1:10000"

    services:
      - listeners:
          # Defines the listener for stream-based traffic (TCP).
          - type: websocket-stream
            web_server: server1
            path: "/wss"
          # Defines the listener for packet-based traffic (UDP).
          - type: websocket-packet
            web_server: server1
            path: "/wsp"
        keys:
          - id: 1
            cipher: chacha20-ietf-poly1305
            # !! IMPORTANT: Replace this with the secret you generated !!
            secret: xYzc2vR+aB1eF5gHjK9LqQ==
    ```
    Press `Ctrl+X`, then `Y`, then `Enter` to save and exit.

---

### **Step 3: Create a Systemd Service**

To ensure the server runs reliably in the background and starts automatically on boot, we'll create a `systemd` service for it.

1.  **Create the Service File**:
    ```bash
    sudo nano /etc/systemd/system/outline-ss-server.service
    ```

2.  **Paste the Service Definition**:
    This definition runs the service as a non-privileged user (`nobody`) for better security.

    ```ini
    [Unit]
    Description=Outline SS Server (WebSocket)
    Documentation=https://developers.google.com/outline/docs/guides/service-providers/websockets
    After=network.target

    [Service]
    Type=simple
    User=nobody
    Group=nogroup
    CapabilityBoundingSet=CAP_NET_BIND_SERVICE
    AmbientCapabilities=CAP_NET_BIND_SERVICE
    NoNewPrivileges=true
    ExecStart=/usr/local/bin/outline-ss-server -config /etc/outline-ss-server/config.yaml
    Restart=on-failure
    RestartSec=10
    LimitNPROC=100
    LimitNOFILE=1048576

    [Install]
    WantedBy=multi-user.target
    ```

3.  **Enable and Start the Service**:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable outline-ss-server
    sudo systemctl start outline-ss-server
    ```

4.  **Check the Service Status** (Recommended):
    ```bash
    sudo systemctl status outline-ss-server
    ```
    You should see a green `active (running)` status. Press `Q` to exit.

---

### **Step 4: Configure Nginx and SSL Certificate**

Now, we'll set up Nginx to handle public HTTPS traffic on port 443 and proxy the WebSocket connections to our backend service.

1.  **Obtain an SSL Certificate**:
    Use Certbot to automatically obtain a certificate for your domain and configure Nginx for SSL. Replace `your.domain.com` with your actual domain.
    ```bash
    sudo certbot --nginx -d your.domain.com
    ```
    Follow the on-screen prompts. Certbot will modify your Nginx configuration file.

2.  **Add the WebSocket Proxy Configuration**:
    Edit the Nginx configuration file that Certbot just modified.
    ```bash
    # The file is usually at /etc/nginx/sites-available/default or your domain's name
    sudo nano /etc/nginx/sites-available/default
    ```
    Inside the `server { ... }` block (look for the one with `listen 443 ssl`), add the following `location` block.

    ```nginx
    # Add this location block inside your server { ... } block

    location ~ ^/(wss|wsp)$ {
        # Proxy to the internal address and port defined in config.yaml
        proxy_pass http://127.0.0.1:10000;

        # Required headers for WebSocket proxying
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;

        # Forward the real client IP address
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    ```

3.  **Test and Reload Nginx**:
    ```bash
    sudo nginx -t
    # If the test is successful, reload the configuration
    sudo systemctl reload nginx
    ```

---

### **Step 5: Create the Client Configuration File (YAML)**

This file acts as a "profile" for your server. The Outline client simply downloads this file and automatically configures all the necessary settings—your domain, WebSocket paths, and secret keys. For this process to work, the YAML file must be placed at a location that is publicly accessible via the internet. The recommended way to add this server to the Outline client is by hosting a special YAML configuration file.

If you prefer not to host this file on your Outline server, you can use any service that provides a direct, raw link to a text file. Good alternatives include AWS S3 and Cloudflare R2.

1.  **Create the Client YAML File**:
    Create a new file in your website's root directory.
    ```bash
    sudo nano /var/www/html/outline-config.yaml
    ```

2. **Paste the Client Configuration**

    This format is specifically for the Outline client. You will need to replace the placeholder values with your own.

    **To make your connection more resilient against sophisticated network filtering**, we will also add a `prefix`. This small piece of data is added to the very beginning of your connection, making it look like standard, legitimate internet traffic (like a regular HTTPS connection). This helps it bypass automated systems that block unfamiliar-looking data, significantly increasing its reliability in harsh network environments.

    According to research by [GFW Report](http://gfw.report/publications/usenixsecurity23/en/), a 3-byte sequence like `\x16\x03\x03` is sufficient to mimic a standard TLS handshake. This triggers an "exemption rule" in systems like the GFW, causing them to stop inspecting the traffic further. Using a minimal prefix is recommended because it's both effective and avoids creating a unique fingerprint that could be blocked in the future.

   Here are some effective examples based on real-world research. You can replace the one in the example with any of these.

    | Prefix Example (in YAML) | Description |
    | :--- | :--- |
    | `"\x16\x03\x03"` | Mimics a **TLS 1.2/1.3 Handshake**. This makes your connection look like the start of a modern HTTPS session. It's one of the most effective and common disguises. |
    | `"POST "` | Mimics an **HTTP POST Request**. This makes your connection look like data being sent to a web server. It is simple and widely recognized as legitimate traffic. Note the space at the end. |
    | `"\x16\x03\x01"` | Mimics a **TLS 1.0/1.1 Handshake**. This is slightly older but still very common across the internet, making it a reliable choice. |

    <details>
    <summary><b>Click to view: Even more "prefix" options</b></summary>

    This Advanced TLS Prefix Reference Table provides the raw hexadecimal values, the format required for your YAML file (`YAML-encoded`), and the format required if you were adding it directly to an `ss://` link (`URL-encoded`).

    ### TLS Handshake Record Type (`0x16`)

    This is the most natural choice for a prefix, as a `ClientHello` handshake is the very first packet in a new TLS connection.

    | Description | Raw Hex | YAML-encoded | URL-encoded |
    | :--- | :--- | :--- | :--- |
    | SSL 3.0 Handshake | `16 03 00` | `"\x16\x03\x00"` | `%16%03%00` |
    | TLS 1.0 Handshake | `16 03 01` | `"\x16\x03\x01"` | `%16%03%01` |
    | TLS 1.1 Handshake | `16 03 02` | `"\x16\x03\x02"` | `%16%03%02` |
    | **TLS 1.2 Handshake (Recommended)** | `16 03 03` | `"\x16\x03\x03"` | `%16%03%03` |
    | Permitted Future/Undefined Version 1 | `16 03 04` | `"\x16\x03\x04"` | `%16%03%04` |
    | Permitted Future/Undefined Version 2 | `16 03 05` | `"\x16\x03\x05"` | `%16%03%05` |
    | Permitted Future/Undefined Version 3 | `16 03 06` | `"\x16\x03\x06"` | `%16%03%06` |
    | Permitted Future/Undefined Version 4 | `16 03 07` | `"\x16\x03\x07"` | `%16%03%07` |
    | Permitted Future/Undefined Version 5 | `16 03 08` | `"\x16\x03\x08"` | `%16%03%08` |
    | Permitted Future/Undefined Version 6 | `16 03 09` | `"\x16\x03\x09"` | `%16%03%09` |

    ### TLS Application Data Record Type (`0x17`)

    Using this type is also a valid strategy, though less common for the very first data packet of a connection.

    | Description | Raw Hex | YAML-encoded | URL-encoded |
    | :--- | :--- | :--- | :--- |
    | SSL 3.0 Application Data | `17 03 00` | `"\x17\x03\x00"` | `%17%03%00` |
    | TLS 1.0 Application Data | `17 03 01` | `"\x17\x03\x01"` | `%17%03%01` |
    | TLS 1.1 Application Data | `17 03 02` | `"\x17\x03\x02"` | `%17%03%02` |
    | TLS 1.2 Application Data | `17 03 03` | `"\x17\x03\x03"` | `%17%03%03` |
    | Permitted Future/Undefined Version 1 | `17 03 04` | `"\x17\x03\x04"` | `%17%03%04` |
    | Permitted Future/Undefined Version 2 | `17 03 05` | `"\x17\x03\x05"` | `%17%03%05` |
    | Permitted Future/Undefined Version 3 | `17 03 06` | `"\x17\x03\x06"` | `%17%03%06` |
    | Permitted Future/Undefined Version 4 | `17 03 07` | `"\x17\x03\x07"` | `%17%03%07` |
    | Permitted Future/Undefined Version 5 | `17 03 08` | `"\x17\x03\x08"` | `%17%03%08` |
    | Permitted Future/Undefined Version 6 | `17 03 09` | `"\x17\x03\x09"` | `%17%03%09` |
   
    **Why does this work?**
    Advanced filtering systems often avoid blocking traffic that looks like standard TLS (the protocol that secures HTTPS). By adding one of these 3-byte prefixes, you make the start of your connection data identical to a legitimate TLS connection, causing it to be "exempted" from further inspection. Using a variety of prefixes can also help prevent your traffic from standing out.

    </details>

    **Replace `your.domain.com` with your domain and `xYzc2vR+aB1eF5gHjK9LqQ==` with your secret.** Then, add the `prefix` line as shown below.

    ```yaml
    transport:
      $type: tcpudp

      tcp:
        $type: shadowsocks
        endpoint:
          $type: websocket
          url: wss://your.domain.com/wss
        # Add the prefix inside the tcp block to disguise TCP traffic
        prefix: "\x16\x03\x03"
        cipher: chacha20-ietf-poly1305
        secret: xYzc2vR+aB1eF5gHjK9LqQ==

      udp:
        $type: shadowsocks
        endpoint:
          $type: websocket
          url: wss://your.domain.com/wsp
        # Also add the same prefix inside the udp block for consistency
        prefix: "\x16\x03\x03"
        cipher: chacha20-ietf-poly1305
        secret: xYzc2vR+aB1eF5gHjK9LqQ==
    ```

    Save and close the file. You should be able to access your configuration file at `https://your.domain.com/outline-config.yaml`.

4.  **Import into Outline Client**:
    *   Open your Outline client.
    *   Click the "+" button to add a new server.
    *   Paste the public URL to your configuration file: `https://your.domain.com/outline-config.yaml`
    *   The client will fetch the file and configure the server automatically.

    Congratulations! You have successfully deployed a secure and robust Outline server using WebSocket obfuscation through an Nginx reverse proxy.

5.  **Optional Step: Optimize Network with TCP BBR**

    For users connecting from regions with high latency and network congestion, enabling Google's BBR congestion control algorithm on your server can significantly improve throughput and connection stability.

    Traditional algorithms often react to packet loss by drastically reducing speed. BBR is smarter; it actively models the network's actual bandwidth and latency to maintain a higher speed, even on less-than-perfect international links.

    Modern Debian systems (10, 11, 12) come with a kernel that supports BBR out of the box. Enabling it is safe and straightforward.

    #### **Prerequisite: Check Your Kernel Version**

    First, verify that your Linux kernel is version 4.19 or higher, which has BBR built-in.
    ```bash
    uname -r
    ```
    If you see a version like `5.10.0-23-amd64` or anything higher than `4.19`, you are ready to proceed.

    #### **Step 1: Enable BBR in System Configuration**

    We need to edit the system's core configuration file to tell the kernel to use BBR.

    Open the `sysctl.conf` file with a text editor:
    ```bash
    sudo nano /etc/sysctl.conf
    ```

    Scroll to the bottom of the file and add the following two lines:
    ```
    # Enable BBR congestion control
    net.core.default_qdisc=fq
    net.ipv4.tcp_congestion_control=bbr
    ```
    *   `net.core.default_qdisc=fq`: Enables the FQ (Fair Queuing) packet scheduler, which is recommended for BBR to work optimally.
    *   `net.ipv4.tcp_congestion_control=bbr`: Sets the default TCP congestion control algorithm to BBR.

    Save and close the file by pressing `Ctrl+X`, then `Y`, then `Enter`.

    #### **Step 2: Apply the New Configuration**

    After saving the file, you need to tell the system to load these new settings without rebooting.
    ```bash
    sudo sysctl -p
    ```
    You should see the two lines you just added printed back to the console, confirming they have been applied.

    #### **Step 3: Verify that BBR is Running**

    Now, let's verify that BBR is indeed the active congestion control algorithm.

    Run this command:
    ```bash
    sysctl net.ipv4.tcp_congestion_control
    ```
    The expected output should be:
    ```
    net.ipv4.tcp_congestion_control = bbr
    ```

    You can also perform a second check to see if the BBR kernel module is loaded:
    ```bash
    lsmod | grep bbr
    ```
    If you see a line containing `tcp_bbr`, it means the module is active.

    That's it! Your server is now using BBR, which should provide a faster and more stable connection experience for your users, especially over long-distance, congested network paths.

---

### **Bonus: Troubleshooting Common Connection Issues**

If you've followed all the steps but find that your client cannot connect to the server, don't worry. The issue is almost always a small typo or mismatch in the configuration files. Here are the most common problems and how to fix them.

The golden rule of troubleshooting is to **check the logs**. These files will tell you exactly what's going wrong.

*   **Check Nginx logs**: `sudo journalctl -u nginx -f`
*   **Check Outline server logs**: `sudo journalctl -u outline-ss-server -f`

Press `Ctrl+C` to stop viewing the logs. Look for any lines that say `[error]` or `[emerg]`.

#### **Problem 1: Path Mismatch**

This is the most common error. The path in your Nginx configuration must **exactly match** the path in your `outline-ss-server`'s `config.yaml`.

*   **Symptoms**: Your client will immediately fail to connect. In the Nginx access log (`/var/log/nginx/access.log`), you might see `404 Not Found` errors for your WebSocket path.

*   **How to Fix**:
    1.  **Check your Outline config**:
        ```bash
        sudo nano /etc/outline-ss-server/config.yaml
        ```
        Look at the `path:` values. For example: `path: "/wss"`.

    2.  **Check your Nginx config**:
        ```bash
        sudo nano /etc/nginx/sites-available/default
        ```
        Look at the `location` block. If your `config.yaml` uses `/wss-outline`, your Nginx location must also match it, for example: `location ~ ^/(wss|wsp)$ { ... }`.

    3.  **Ensure they are identical.** An exmaple would be you entered `path: "/ws"` in Outline config instead of `path: "/wss"`. Double check to ensure they are identical. Even a missing slash or a typo will cause it to fail. After fixing, restart both services:
        ```bash
        sudo systemctl restart outline-ss-server
        sudo systemctl restart nginx
        ```

#### **Problem 2: Port Mismatch**

This is the second most common error. The port Nginx proxies to must be the same one your `outline-ss-server` is listening on.

*   **Symptoms**: The client will time out. In the Nginx error log (`/var/log/nginx/error.log`), you will see errors like `connect() failed (111: Connection refused) while connecting to upstream`. This means Nginx is trying to forward the traffic, but no service is listening at the target address.

*   **How to Fix**:
    1.  **Check your Outline config**:
        ```bash
        sudo nano /etc/outline-ss-server/config.yaml
        ```
        Look at the `listen:` value. For example: `listen: - "127.0.0.1:10000"`.

    2.  **Check your Nginx config**:
        ```bash
        sudo nano /etc/nginx/sites-available/default
        ```
        Look at the `proxy_pass` directive inside your WebSocket location block. It must point to the same address and port: `proxy_pass http://127.0.0.1:10000;`.

    3.  **Ensure they are identical.** An example would be you entered `443` as the port number in your Outline config but `10000` in Nginx config. Make sure to modify the port numbers in config files to make them identical. After fixing, restart Nginx: `sudo systemctl restart nginx`.

#### **Problem 3: Firewall is Blocking the Port**

If you are using a firewall on your server (like `ufw`), you must explicitly allow traffic on port 443.

*   **Symptoms**: The client cannot connect at all. The connection attempt doesn't even seem to reach Nginx.

*   **How to Fix** (if using `ufw`):
    1.  **Allow HTTPS traffic**:
        ```bash
        sudo ufw allow 'Nginx Full'
        # Or more specifically:
        sudo ufw allow 443/tcp
        ```
    2.  **Check the firewall status**:
        ```bash
        sudo ufw status
        ```
        Ensure that port 443 is listed as `ALLOW`.

#### **Problem 4: Incorrect Secret or Client Configuration**

A simple copy-paste error in the client's YAML configuration file can also cause issues.

*   **Symptoms**: The client might connect but immediately disconnect, or it might show an authentication error.

*   **How to Fix**:
    1.  **Double-check the secret**: Ensure the `secret:` value in your server's `config.yaml` is the exact same one used in the client's `outline-config.yaml`.
    2.  **Double-check the domain**: Make sure the `url:` in the client's config (`wss://your.domain.com/wss`) points to the correct domain name that you have configured in Nginx and CloudFront/your DNS.
    3.  **Double-check the prefix**: If you added a `prefix`, ensure it's correctly formatted and identical in both the `tcp` and `udp` sections of the client YAML file.

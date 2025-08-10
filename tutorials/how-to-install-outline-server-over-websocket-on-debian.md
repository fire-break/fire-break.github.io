### **How to Install Outline Server with WebSocket and Nginx on Debian**

This tutorial will guide you through setting up the `outline-ss-server` to listen for WebSocket connections on a private port and using Nginx to proxy public traffic from port 443 to the service. This setup enhances security and allows you to run other web services on the same server.

#### **Prerequisites**

1.  **A Debian Server**: A fresh installation of Debian 11 or 12 is recommended.
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
    ```    Paste the following content into the file. **Remember to replace the `secret` value with the one you generated.**

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
    ```    Press `Ctrl+X`, then `Y`, then `Enter` to save and exit.

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
    ```    You should see a green `active (running)` status. Press `Q` to exit.

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

2.  **Paste the Client Configuration**:
    This format is specifically for the Outline client. **Replace `your.domain.com` with your domain and `xYzc2vR+aB1eF5gHjK9LqQ==` with your secret.**

    ```yaml
    transport:
      $type: tcpudp

      tcp:
        $type: shadowsocks
        endpoint:
          $type: websocket
          url: wss://your.domain.com/wss
        cipher: chacha20-ietf-poly1305
        secret: xYzc2vR+aB1eF5gHjK9LqQ==

      udp:
        $type: shadowsocks
        endpoint:
          $type: websocket
          url: wss://your.domain.com/wsp
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

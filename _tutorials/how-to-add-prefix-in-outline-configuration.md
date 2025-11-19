---
title: "How to add prefix in Outline configuration"
layout: default 
---

## **How to add prefix in Outline configuration**

**To make your connection more resilient against sophisticated network filtering**, we can also add a `prefix`. This small piece of data is added to the very beginning of your connection, making it look like standard, legitimate internet traffic (like a regular HTTPS connection). This helps it bypass automated systems that block unfamiliar-looking data, significantly increasing its reliability in harsh network environments.

According to research by [GFW Report](http://gfw.report/publications/usenixsecurity23/en/), a 3-byte sequence like `\x16\x03\x03` is sufficient to mimic a standard TLS handshake. This triggers an "exemption rule" in systems like the GFW, causing them to stop inspecting the traffic further. Using a minimal prefix is recommended because it's both effective and avoids creating a unique fingerprint that could be blocked in the future.

Here are some effective examples based on real-world research. You can replace the one in the example with any of these.

| Prefix Example (in YAML) | Description |
| :--- | :--- |
| `"\x16\x03\x03"` | Mimics a **TLS 1.2/1.3 Handshake**. This makes your connection look like the start of a modern HTTPS session. It's one of the most effective and common disguises. |
| `"POST "` | Mimics an **HTTP POST Request**. This makes your connection look like data being sent to a web server. It is simple and widely recognized as legitimate traffic. Note the space at the end. |
| `"\x16\x03\x01"` | Mimics a **TLS 1.0/1.1 Handshake**. This is slightly older but still very common across the internet, making it a reliable choice. |

### **Even more "prefix" options**

This Advanced TLS Prefix Reference Table provides the raw hexadecimal values, the format required for your YAML file (`YAML-encoded`), and the format required if you were adding it directly to an `ss://` link (`URL-encoded`).

#### **TLS Handshake Record Type (`0x16`)**

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

#### **TLS Application Data Record Type (`0x17`)**

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

### **Example of a Outline client configuration file (w/ prefix added)**

Continue from step 5 in [this tutorial](/tutorials/how-to-install-outline-server-over-websocket-on-debian.html).

1.  **Locate the Client YAML File**:
    Open the client configuration file in your website's root directory (or wherever you store that file).
    ```bash
    sudo nano /var/www/html/outline-config.yaml
    ```

2.  **Replace `your.domain.com` with your domain and `xYzc2vR+aB1eF5gHjK9LqQ==` with your secret.** Then, add the `prefix` line as shown below. (using the YAML-encoded prefix listed in the tables above.)

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

### **Example of a Outline static access key URI (w/ prefix added)**

A Static "Access Key" generated by Outline Manager looks like this,

``ss://Z34nthataITHiTNIHTohithITHbVBqQ1o3bkk@127.0.0.1:33142/?outline=1``

Add a URL-coded prefix at the end.

``ss://Z34nthataITHiTNIHTohithITHbVBqQ1o3bkk@127.0.0.1:33142/?outline=1&prefix=%16%03%03``

Add this modified access key to Outline client.

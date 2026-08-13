# Jellyfin Remote Access & Reverse Proxy Setup - Summary Guide

## Overview & Architecture
This document summarizes the network topology, troubleshooting process, root cause analysis, and resolution steps for exposing a self-hosted **Jellyfin** media server externally via an **Nginx** reverse proxy behind a double-NAT firewall configuration.

### Network Topology
* **Domain:** `jellyfin.datahivetech.dev`
* **Edge Routing:** ISP Router (Port Forwarding) → pfSense Firewall (NAT / WAN Port Forwarding)
* **Reverse Proxy:** Nginx Host (`10.0.40.12` / VLAN 40)
* **Application Server:** Jellyfin Host (`10.0.30.10:8096` / VLAN 30)

---

## 1. Issue Description
When attempting to access `http://jellyfin.datahivetech.dev` via web browsers (Safari, Zen Browser, Chrome), the connection resulted in a **"Connection Timed Out"** error. 

However, direct raw `curl` tests against `http://jellyfin.datahivetech.dev` yielded successful response chains:
```http
HTTP/1.1 302 Found
Location: web/

HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 9723
```

---

## 2. Root Cause Analysis
1. **Network & Routing Status:** The successful `curl` test over HTTP (Port 80) proved that DNS resolution, ISP port forwarding, pfSense NAT rules, inter-VLAN firewall routing (VLAN 40 to VLAN 30), and basic Nginx proxying were **100% operational**.
2. **The `.dev` TLD HSTS Restriction:** Google manages the `.dev` Top-Level Domain (TLD), which is included on the global **HSTS (HTTP Strict Transport Security) Preload List**. 
3. **Browser Enforcement:** Web browsers automatically enforce HTTPS for all `.dev` domains before sending network traffic. Even when manually typing `http://`, browsers locally rewrite the request to `https://` (Port 443). Since Port 443 was not yet forwarded or listening on Nginx, browser connections timed out.

---

## 3. Step-by-Step Resolution Plan

### Step 1: Optimize Nginx Reverse Proxy Configuration
Update your `/etc/nginx/conf.d/default.conf` to support WebSockets, disables buffering for streaming media, and sets essential reverse proxy headers.

```nginx
server {
    listen 80;
    listen [::]:80;
    
    server_name jellyfin.datahivetech.dev;

    location / {
        proxy_pass http://10.0.30.10:8096;
        
        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Protocol $scheme;
        proxy_set_header X-Forwarded-Host $http_host;

        # Disable buffering to prevent stream delays
        proxy_buffering off;

        # WebSocket support (Required for Jellyfin client synchronization)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Validate and reload Nginx:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

### Step 2: Configure Port 443 Forwarding
To satisfy HSTS requirements, forward incoming HTTPS traffic across your perimeter:

1. **ISP Router:** Create a port forwarding rule directing inbound **TCP 443** to the pfSense WAN IP address.
2. **pfSense Router:** 
   * Navigate to **Firewall > NAT > Port Forward**.
   * Add a rule on the **WAN** interface for **TCP Port 443** pointing to Nginx (`10.0.40.12`).
   * Ensure the associated firewall filter rule is auto-created/enabled.

---

### Step 3: Provision Let's Encrypt SSL/TLS Certificate
Install Certbot on the Nginx host (Ubuntu/Debian) to automatically provision an SSL certificate via Let's Encrypt.

1. **Install Certbot & Nginx plugin:**
   ```bash
   sudo apt update
   sudo apt install certbot python3-certbot-nginx -y
   ```

2. **Obtain Certificate & Auto-Configure Nginx:**
   ```bash
   sudo certbot --nginx -d jellyfin.datahivetech.dev
   ```

3. **Verification:**
   * Certbot uses HTTP-01 validation over Port 80 (already functional).
   * Upon completion, Certbot modifies `default.conf` to handle SSL listening on port 443 and auto-redirects Port 80 traffic to HTTPS.

---

## 4. Verification Checklist
* [x] Port 80 HTTP routing verified via `curl -IL http://jellyfin.datahivetech.dev`
* [x] Port 443 HTTPS rule added to ISP Router & pfSense
* [x] Certbot SSL certificate successfully issued
* [x] Secure web access verified via `https://jellyfin.datahivetech.dev` in browser

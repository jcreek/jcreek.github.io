---
tags:
  - home lab
  - cloudflare
  - self-hosting
---

# How I set up my home lab with Cloudflare

_2025-03-02_

## **Overview**

This guide walks through setting up my home lab using the domain `creeknet.uk`, managing **DNS, security, remote access, reverse proxy, and VPN** to ensure secure and private access to my services. This exists both as a record for myself and a guide for others looking to set up a similar home lab.

---

## **1. Cloudflare Setup**

### **1.1 Register Domain & Set Up DNS**

1. Log into [Cloudflare](https://dash.cloudflare.com) and **add `creeknet.uk`**. The `.uk` domains are amongst the cheapest, at around $5 per year.
2. Go to DNS > Add A Records for all publicly accessible services:
   These records allow external access to specific services.  
    ⚠️ **Cloudflare Proxy (Orange Cloud ☁️) should be OFF (DNS Only) for services that require non-HTTP ports (e.g., Plex, VPN).**

   | **Subdomain**          | **Record Type** | **Points To**    | **Proxy Status**       | **Purpose**               |
   | ---------------------- | --------------- | ---------------- | ---------------------- | ------------------------- |
   | `ai.creeknet.uk`       | A               | **My Public IP** | **DNS Only ☁️ (Gray)** | AI Server                 |
   | `home.creeknet.uk`     | A               | **My Public IP** | **DNS Only ☁️ (Gray)** | Home Assistant            |
   | `nas.creeknet.uk`      | A               | **My Public IP** | **DNS Only ☁️ (Gray)** | NAS Access                |
   | `plex.creeknet.uk`     | A               | **My Public IP** | **DNS Only ☁️ (Gray)** | Remote Plex Access        |
   | `rustdesk.creeknet.uk` | A               | **My Public IP** | **DNS Only ☁️ (Gray)** | Remote Desktop (RustDesk) |
   | `vpn.creeknet.uk`      | A               | **My Public IP** | **DNS Only ☁️ (Gray)** | WireGuard VPN             |

### **1.2 Enable Cloudflare Zero Trust for Web Services**

I did consider that for any web-facing applications that do not have their own authentication I could use Cloudflare Zero Trust to enforce authentication. To do so I would:

1. Go to **Zero Trust → Access → Applications**.
2. Add an application.
3. Configure **Access Policy** (Require login via OTP).

---

## **2. Setting Up Local DNS (Split DNS)**

To **ensure local services work even if the internet is down**, configure **Pi-hole (`10.0.0.6`)**:

- Add **Local DNS Records**:

  - Web-services should direct to Nginx:
    - ai.creeknet.uk → `10.0.0.20`
    - home.creeknet.uk → `10.0.0.13`
    - nas.creeknet.uk → `10.0.0.9`
    - plex.creeknet.uk → `10.0.0.11`
  - Non-web services should direct to the actual servers:
    - rustdesk.creeknet.uk → `10.0.0.23` (dockerised on the same server as nginx)
    - vpn.creeknet.uk → `10.0.0.5` (redundant if the internet is down but included for completeness)

- **Router DNS Settings**:
  - Primary DNS: `10.0.0.6` (Pi-hole)
  - Secondary DNS: `1.1.1.1` (Cloudflare)

---

## **3. Reverse Proxy with NGINX Proxy Manager**

### **3.1 Deploy NGINX Proxy Manager (Docker) on ProxRouter**

```yaml
version: "3"
services:
  npm:
    image: "jc21/nginx-proxy-manager:latest"
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81" # Admin UI
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

### **3.2 Configure Proxy Hosts**

- ai.creeknet.uk → `http://10.0.0.20:8080` (Open Web UI)
  - Provide SSL certificate through Let’s Encrypt
  - Enable Websockets Support
  - Force SSL
- home.creeknet.uk → `http://10.0.0.13:8123` (Home Assistant)

  - Provide SSL certificate through Let’s Encrypt
  - Enable Websockets Support
  - Force SSL
  - Advanced custom nginx config:

    ```text
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Connection "upgrade";
    proxy_redirect off;
    ```

- nas.creeknet.uk → `https://10.0.0.9:1001` (Asustor NAS)
  - Provide SSL certificate through Let’s Encrypt
- plex.creeknet.uk → `http://10.0.0.11:32400` (Plex Media Server)
  - Provide SSL certificate through Let’s Encrypt

# Deploy a Personal AdGuard Home DNS Server with Nginx Proxy Manager

This guide provides a comprehensive walkthrough for deploying a private, ad-blocking DNS server using AdGuard Home, secured with SSL certificates from Nginx Proxy Manager. The entire stack runs in Docker on a VPS.

## Prerequisites

1.  **A VPS Server:** Any small Debian/Ubuntu server is sufficient.
2.  **A Domain Name:** For this guide, we will use `domain.com`.
3.  **Cloudflare Account:** To manage your domain's DNS records.

## Step 1: Cloudflare DNS Configuration

This is a critical step. For DNS-over-TLS to work, you must bypass Cloudflare's proxy.

1.  Log in to your Cloudflare dashboard.
2.  Go to the DNS settings for `domain.com`.
3.  Create a new **A record**:
    *   **Type:** `A`
    *   **Name:** `adguard` (or any subdomain you prefer)
    *   **IPv4 address:** Your VPS IP address (e.g., `123.123.123.123`)
    *   **Proxy status:** **DNS only** (click the orange cloud so it turns grey). This is mandatory.



## Step 2: Server Preparation

Connect to your VPS via SSH and install docker.

```bash
curl -fsSL https://get.docker.com | sudo sh && apt install git -y
```

## Step 3: Create the Docker Compose File

Create a single file to manage both services.

```bash
# Create a directory for your project
mkdir adguard-stack && cd adguard-stack

# Create the compose file
nano docker-compose.yml
```

Paste the following content into `docker-compose.yml`. This file is configured for security, using a shared network so you don't need to expose AdGuard's web ports to the internet.

```yaml
services:
  # Nginx Proxy Manager - Our reverse proxy for SSL
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: npm
    restart: unless-stopped
    ports:
      - '80:80'    # Public HTTP
      - '443:443'  # Public HTTPS
      - '81:81'    # NPM Admin UI
    volumes:
      - npm-data:/data
      - npm-letsencrypt:/etc/letsencrypt
    networks:
      - npm-network

  # AdGuard Home - Our DNS Server
  adguardhome:
    image: adguard/adguardhome
    container_name: adguardhome
    restart: unless-stopped
    volumes:
      - adguard-data:/opt/adguardhome/work
      - adguard-config:/opt/adguardhome/conf
      - npm-letsencrypt:/etc/letsencrypt:ro # Mount certs read-only
    ports:
      # Port 53 is for standard DNS. On a VPS, this is a security risk unless you firewall it to specific IPs. Keep it commented out if you only plan to use encrypted DNS (recommended).
      # - "53:53/tcp"
      # - "53:53/udp"

      # Port 853 is for DNS-over-TLS
      - "853:853/tcp"

      # Port 3000 for the initial setup ONLY.
      - "3000:3000/tcp"
    networks:
      - npm-network

volumes:
  npm-data:
  npm-letsencrypt:
  adguard-data:
  adguard-config:

networks:
  npm-network:
    name: npm-network
```

## Step 4: Deploy and Configure Nginx Proxy Manager

1.  Launch the stack:
    ```bash
    docker compose up -d
    ```

2.  **Access the NPM Admin UI** by navigating to `http://<your-vps-ip>:81`.
    *   Default Admin User: `admin@example.com`
    *   Default Password: `changeme`
    *   You will be forced to change these credentials immediately.

3.  **Request a Wildcard SSL Certificate:** A wildcard certificate is needed to identify your specific devices later.
    *   Go to **Hosts** -> **Proxy Hosts** and click **Add Proxy Host**.
    *   **Details Tab:**
        *   **Domain Names:** `adguard.domain.com`
        *   **Scheme:** `http`
        *   **Forward Hostname / IP:** `adguardhome` (use the container name)
        *   **Forward Port:** `80` (the container's internal port)
        *   Enable **Block Common Exploits**.
    *   **SSL Tab:**
        *   **SSL Certificate:** "Request a new SSL Certificate"
        *   Add a second domain name for the wildcard: `*.adguard.domain.com`
        *   Enable **Force SSL**.
        *   Enter your email address and agree to the terms.
    *   Click **Save**. NPM will now obtain the certificate.

## Step 5: First-Time AdGuard Home Setup

1.  Access the AdGuard Home setup wizard by navigating to `http://<your-vps-ip>:3000`.

2.  Walk through the wizard:
    *   **Step 1:** Click "Get Started".
    *   **Step 2 (Web Interface):** Leave as `All interfaces` and Port `80`.
    *   **Step 2 (DNS server):** Leave as `All interfaces` and Port `53`.
    *   **Step 3:** Create your admin username and password.
    *   **Step 4:** Follow the setup instructions.
    *   **Step 5:** Click "Open Dashboard".

## Step 6: Final AdGuard Configuration

1.  **Access your secure dashboard** via `https://adguard.domain.com`.

2.  **Configure Encryption:**
    *   Go to **Settings** -> **Encryption settings**.
    *   Check **Enable encryption**.
    *   Your **Server name** should be `adguard.domain.com`.
    *   Under Certificates, select **Set paths to certificate and key**.
    *   **Certificate path:** `/etc/letsencrypt/live/npm-1/fullchain.pem`
    *   **Private key path:** `/etc/letsencrypt/live/npm-1/privkey.pem`
        *(Note: The number `npm-1` might be different if you have requested certs before. Verify the path if needed.)*
    *   Click **Save configuration**.

3.  **Configure Upstream DNS:**
    *   Go to **Settings** -> **DNS settings**.
    *   In "Upstream DNS servers", add a private provider like Quad9:
        ```
        tls://dns.quad9.net
        ```
    *   Click **Apply**.

## Step 7: Secure Your Server

Now that the setup is complete, you must close the temporary port `3000`.

1.  Edit your `docker-compose.yml` file (`nano docker-compose.yml`).
2.  Comment out or delete the port mapping for port 3000:
    ```yaml
    # - "3000:3000/tcp"
    ```
3.  Re-deploy the stack to apply the change:
    ```bash
    docker compose up -d
    ```

## Step 8: Connect Your Devices

Your private DNS server is ready!

*   **For Android or iOS (Private DNS):**
    *   Go to your device's network settings for Private DNS.
    *   Use the hostname: `adguard.domain.com`.

*   **For Device-Specific Filtering (Recommended):**
    1.  In AdGuard, go to **Settings -> Client settings** to create a client (e.g., Name: `My Phone`, Identifier: `my-phone`).
    2.  On your device, use the hostname: `my-phone.adguard.domain.com`. The wildcard certificate you created makes this possible.

You are now running a fully functional, private, and secure ad-blocking DNS server.

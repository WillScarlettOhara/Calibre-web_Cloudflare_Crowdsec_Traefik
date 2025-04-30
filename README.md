# Docker Traefik Stack with Cloudflare, CrowdSec & Let's Encrypt

This repository contains a Docker Compose configuration to set up a secure Traefik reverse proxy, integrating Cloudflare (Tunnel and DNS), CrowdSec for intrusion prevention, and Let's Encrypt for automatic TLS certificates. It includes examples for exposing services publicly and others only on the local network.

## Acknowledgements

* This README file and parts of the configuration setup were generated and refined with the assistance of Google's Gemini large language model.
* Inspiration and guidance for this setup were drawn from several excellent community resources, including:
    * LRVT's Security Blog: [Configuring CrowdSec with Traefik](https://blog.lrvt.de/configuring-crowdsec-with-traefik/)
    * Techno Tim: [Traefik 3 Docker Certificates Guide](https://technotim.live/posts/traefik-3-docker-certificates/)
    * Videos and guides by [@JimsGarage](https://www.youtube.com/@Jims-Garage)

## Core Components

* **Traefik v3:** Reverse proxy and load balancer.
* **Docker & Docker Compose:** For containerization and orchestration.
* **Cloudflare:**
    * **Tunnel:** To expose web services without opening firewall ports.
    * **DNS:** Used by Traefik for ACME DNS-01 challenges via the Cloudflare API to obtain Let's Encrypt certificates.
* **Let's Encrypt:** Provider of free, automated TLS certificates.
* **CrowdSec:** Open-source security agent (IPS) that analyzes logs (including Traefik's), detects malicious behavior, and blocks corresponding IPs via a Traefik Bouncer.
* **Example Services:**
    * Calibre-Web (Exposed publicly)
    * Pi-hole (Local access only)
    * Proxmox (Local access only)
    * Portainer (Local access only)
    * Traefik Dashboard (Local access only)

## Key Features

* Automatic route configuration via Docker labels or a dynamic configuration file.
* Automatic HTTPS for all services via Let's Encrypt (DNS-01 challenge with Cloudflare).
* Secure exposure via Cloudflare Tunnel (no incoming ports need to be opened).
* Detection and blocking of malicious IPs via CrowdSec and the Traefik bouncer.
* Correct handling of the real visitor IP behind Cloudflare using the `cloudflarewarp` Traefik plugin.
* Access restriction for specific services to the local network only via a Traefik middleware (`default-whitelist`).
* Default security HTTP headers applied to services.

## Prerequisites

* A server with Docker and Docker Compose installed.
* A Cloudflare account.
* A domain name managed by Cloudflare.
  ![image](https://github.com/user-attachments/assets/c5d3c1dc-1388-41d8-b708-b8b5d7e7c52e)
  ![image](https://github.com/user-attachments/assets/02a3c984-7718-4434-9031-2d951f8fc456)
* A **Cloudflare API Token** with the necessary permissions to edit DNS records for your zone (Permissions: Zone:Zone:Read, Zone:DNS:Edit).
* A **Cloudflare Tunnel Token** if using that method (created via the Cloudflare Zero Trust dashboard).
* A functioning Pi-hole instance (or other local DNS server) if you want to resolve local-only services by hostname from within your LAN.

## Installation and Configuration

1.  **Clone this repo:**
    ```bash
    git clone <repository_url>
    cd <repository_directory_name>
    ```

2.  **Create Docker Network:**
    This setup uses an external network named `proxy`. Create it if it doesn't exist:
    ```bash
    sudo docker network create proxy
    ```

3.  **Create Secret and Environment Files:**
    * **`.env` (at the root or in `traefik` and `cloudflared` directories):**
        * `TRAEFIK_DASHBOARD_CREDENTIALS`: Your credentials for the Traefik dashboard (format `user:hashed_password`). Use `htpasswd` to generate the hashed password (e.g., `echo $(htpasswd -nB user) | sed -e s/\\$/\\$\\$/g`).
        * `TUNNEL_TOKEN`: Your Cloudflare Tunnel token (obtained from Cloudflare).
    * **Cloudflare DNS API Token File:** Create a file (e.g., `traefik/cf-token`) containing **only** your Cloudflare DNS API token. The path to this file is referenced in `traefik/docker-compose.yaml` via the `secrets` section.
    * **CrowdSec Bouncer API Key File:** Create a file (e.g., `traefik/crowdsec-lapi-key`) containing **only** the API key generated for the Traefik bouncer (via `docker exec crowdsec cscli bouncers add traefik-plugin-bouncer`). The path is referenced in `traefik/docker-compose.yaml` via the `secrets` section.

4.  **Adapt Configurations:**
    * **Domain Name:** Search and replace all occurrences of `<DOMAIN_NAME>` (and its subdomains like `calibre.<DOMAIN_NAME>`) with **your own domain name** in the following files:
        * `traefik/config.yaml` (`Host(...)` rules)
        * `traefik/docker-compose.yaml` (`Host(...)` and `tls.domains` labels)
        * `calibre/docker-compose.yaml` (`Host(...)` labels)
        * `pihole/docker-compose.yaml` (`Host(...)` labels)
        * `portainer/docker-compose.yaml` (if applicable)
    * **ACME Email:** In `traefik/traefik.yaml`, replace `your-email@example.com` with your real email address for Let's Encrypt notifications.
    * **Local IPs:**
        * In `traefik/config.yaml`, adjust the `proxmox` service URL (`servers.url`) to match your Proxmox server's IP/port.
        * If you added Portainer via `config.yaml`, adjust its `servers.url` with your Docker host IP and Portainer's exposed port.
        * Check the local IP range in the `default-whitelist` middleware (`sourceRange`). If your local network is not `192.168.1.0/24` (or included in `192.168.0.0/16`), adapt it.
    * **Proxy Network CIDR:** In `traefik/traefik.yaml`, under the `https` entrypoint -> `forwardedHeaders -> trustedIPs`, replace `YOUR_ACTUAL_PROXY_SUBNET_HERE` with the result of `docker network inspect proxy` (look for `IPAM.Config[0].Subnet`).
    * **Volume Paths:** Check all volumes in the `docker-compose.yaml` files. Ensure the host paths (the part before the `:`) exist on your server or adapt them to your structure. Create directories if needed (e.g., `/home/will/docker/traefik/logs`, `/home/will/docker/crowdsec/data`, etc.).

5.  **Configure Local DNS (Pi-hole):** See the section below.

6.  **Start the Containers:**
    Launch each stack (or all together if you combine the compose files):
    ```bash
    # Example for starting Traefik
    cd traefik/
    docker compose up -d

    # Example for starting Crowdsec
    cd ../crowdsec/
    docker compose up -d

    # Etc. for other services...
    ```

## Local DNS Configuration (Pi-hole)

To access your services (especially those restricted to local access like Pi-hole admin, Proxmox, Portainer, and the Traefik dashboard) using their hostnames (e.g., `https://pihole.example.com`) from devices **within your local network**, you need to configure your local DNS server (Pi-hole in this case) to resolve these hostnames to the **local IP address of the server running Traefik**.

This is necessary because public DNS (Cloudflare) will resolve these names to Cloudflare IPs (for the tunnel), which won't work for accessing IP-whitelisted services directly from your LAN.

**Steps:**

1.  **Identify Traefik Host IP:** Find the local IP address of the machine running Docker and Traefik (e.g., `192.168.1.100`). Let's call this `<TRAEFIK_HOST_IP>`.
2.  **Login to Pi-hole:** Access your Pi-hole web admin interface. First time use `192.168.1.100:500`
3.  **Navigate:** Go to **Local DNS** -> **DNS Records**.
4.  **Add A Record(s):** Add an A record for each primary hostname pointing to your Traefik host's IP. It's often convenient to create one base A record and use CNAMEs for services.
    * **Example Base A Record:**
        * Domain: `docker.example.com` (Or choose another base, like `traefik.example.com`)
        * IP Address: `<TRAEFIK_HOST_IP>` (Replace with your actual IP, e.g., `192.168.1.100`)
    * **Example Direct A Record (Alternative):**
        * Domain: `calibre-web.example.com`
        * IP Address: `<TRAEFIK_HOST_IP>` (e.g., `192.168.1.100`)
5.  **Navigate:** Go to **Local DNS** -> **CNAME Records**.
6.  **Add CNAME Records:** Add CNAME records to point the friendly service names to the base A record you created (or directly to another hostname if preferred). This makes management easier if your Traefik host IP changes.
    * **Example CNAME Records (pointing to `docker.example.com`):**
        * Domain: `traefik.example.com` -> Target: `docker.example.com`
        * Domain: `pihole.example.com` -> Target: `docker.example.com`
        * Domain: `proxmox.example.com` -> Target: `docker.example.com`
        * Domain: `portainer.example.com` -> Target: `docker.example.com`
        * Domain: `calibre.example.com` -> Target: `docker.example.com` *(Or use your direct A record as target below)*
    * **Example based on your request (using a direct A record and then a CNAME):**
        * **(First add A Record):** Go to DNS Records -> Domain: `calibre-web.example.com`, IP: `192.168.1.100`
        * **(Then add CNAME Record):** Go to CNAME Records -> Domain: `calibre.example.com`, Target: `calibre-web.example.com`
7.  **Configure Clients:** Ensure that the devices on your local network (computers, phones when on WiFi, etc.) are configured to **use your Pi-hole instance as their DNS server**. This is typically set via your router's DHCP settings.

With these local DNS records in place, when you type `https://pihole.example.com` or `https://calibre.example.com` into your browser from a device on your local network using Pi-hole for DNS, the name will resolve to your Traefik server's local IP, allowing Traefik to route the request (and apply the IP whitelist correctly).

## Detailed Configuration

* **`traefik/traefik.yaml`:** Traefik's static configuration. Defines entrypoints (`http`, `https`), providers (Docker, File), the ACME certificate resolver (`cloudflare`), access log settings (JSON format), and loads plugins (`cloudflarewarp`, `crowdsec-bouncer`). The `forwardedHeaders` section under `https` is essential for getting the real IP behind the Cloudflare tunnel.
* **`traefik/config.yaml`:** Traefik's dynamic configuration. Defines:
    * Reusable middlewares:
        * `https-redirectscheme`: Redirects HTTP to HTTPS.
        * `default-headers`: Adds security headers (HSTS, X-Frame-Options, etc.).
        * `default-whitelist`: Restricts access to local IPs (used for Pi-hole, Proxmox, Portainer, Traefik Dashboard).
        * `cloudflarewarp`: Plugin to get the real visitor IP behind Cloudflare.
        * `crowdsec-bouncer`: Plugin to query CrowdSec and block banned IPs (reads API key from Docker secret).
    * Routers and services for applications *not* discovered via Docker (e.g., Proxmox, Portainer if installed via `docker run`).
* **Docker Labels (in application `docker-compose.yaml` files):** Method for Traefik to automatically discover and configure routes for Docker containers. Labels define routers (`Host(...)`), services (`loadbalancer.server.port`), middlewares to apply (referencing those in `config.yaml` with `@file`, or defining simple ones directly in labels), and TLS configuration.

## Troubleshooting / Useful Commands

* View container logs: `docker logs <container_name>` (e.g., `docker logs traefik`, `docker logs crowdsec`)
* Enter a container's shell: `docker exec -it <container_name> sh`
* View CrowdSec decisions (bans): `docker exec crowdsec cscli decisions list`
* View CrowdSec alerts: `docker exec crowdsec cscli alerts list -a`
* View CrowdSec CAPI status: `docker exec crowdsec cscli capi status`
* View CrowdSec metrics (parsing, scenarios): `docker exec crowdsec cscli metrics`
* Test Traefik log line parsing: `docker exec crowdsec cscli explain --log '<json_access_log_line>' --type traefik -v`
* Inspect a Docker network: `docker network inspect proxy`
* Force a log rotation: `sudo logrotate -f /etc/logrotate.conf`

## Testing CrowdSec Ban with Smartphone

You can simulate an attack from your smartphone to verify that CrowdSec detects it and the Traefik bouncer enforces the ban.

1.  **Use Mobile Data:** **Disconnect your phone from WiFi** and use your mobile data (4G/5G). This ensures the request comes from an external IP address not on your local whitelist. You can find your phone's current public IP by searching "what is my IP address" in its browser.
2.  **Run Simulation:** The easiest way to trigger detection reliably (especially for scenarios like `http-probing` which might require unique paths) is often using a terminal emulator like Termux on Android.
    * Install Termux (from F-Droid/GitHub is recommended) and `curl` (`pkg install curl`).
    * Run a loop targeting a **publicly exposed service** (like `calibre.example.com` in this setup) with **distinct paths** to trigger probing rules:
      ```bash
      # Request 15 DIFFERENT non-existent paths rapidly
      for i in $(seq 1 15); do curl -I -k "[https://calibre.example.com/test-probe-$i](https://calibre.example.com/test-probe-$i)"; echo "Req $i"; sleep 0.5; done
      ```
      *(Adjust count and sleep time if needed based on scenario capacity/leakspeed found via `cscli scenarios inspect ...`)*
3.  **Wait:** Allow 30-60 seconds for CrowdSec to process the logs.
4.  **Check Decisions:** Get a shell inside the CrowdSec container and list active decisions:
    ```bash
    docker exec -it crowdsec sh
    cscli decisions list
    ```
    Look for your phone's mobile IP address in the list with a `ban` action.
5.  **Verify Block:** Try accessing the targeted site (e.g., `https://calibre.example.com`) from your phone's browser (still on mobile data). You should receive a **"Forbidden" (403) error** from Traefik, indicating the bouncer blocked the request.
6.  **Unban:** Once finished, remove the ban from within the CrowdSec container shell:
    ```bash
    # Replace <YOUR_PHONE_IP> with the actual IP
    cscli decisions delete -i <YOUR_PHONE_IP>
    # Exit the container shell
    exit
    ```

## Disclaimer

This is an example configuration. Adapt it to your own security, performance, and service requirements. Ensure you understand what each part does before deploying it. Never expose secrets or sensitive data in your Git repository.

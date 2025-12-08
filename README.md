# Inception Infrastructure

## Table of Contents
- [Project Summary](#project-summary)
- [High-Level Topology](#high-level-topology)
- [Technology Stack](#technology-stack)
- [Compose Orchestration](#compose-orchestration)
  - [Networks](#networks)
  - [Volumes](#volumes)
  - [Global Environment Expectations](#global-environment-expectations)
- [Service Reference](#service-reference)
  - [NGINX](#nginx)
  - [WordPress (PHP-FPM)](#wordpress-php-fpm)
  - [MariaDB](#mariadb)
  - [Redis](#redis)
  - [FTP (vsftpd)](#ftp-vsftpd)
  - [Adminer](#adminer)
  - [Portainer](#portainer)
  - [Static Site](#static-site)
- [Configuration Files & Scripts](#configuration-files--scripts)
- [Building & Running the Stack](#building--running-the-stack)
  - [Day-to-day workflow](#day-to-day-workflow)
  - [Environment bootstrap](#environment-bootstrap)
  - [Log inspection](#log-inspection)
- [Security & TLS Notes](#security--tls-notes)
- [Data Persistence & Host Bindings](#data-persistence--host-bindings)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [Further Improvements](#further-improvements)

## Project Summary
This repository hosts a Docker Compose-based infrastructure inspired by the 42 "Inception" project. It defines a multi-container environment that delivers a production-style WordPress deployment with HTTPS termination, database, caching, administration, file transfer, Docker UI management, and a bonus static website. Every service is built from handcrafted Dockerfiles using Debian Bullseye (or closely related) base images, ensuring full ownership of the software stack and transparent reproducibility.

## High-Level Topology
```
            🌍 External Host (Your Browser, FTP Client, etc.)
            ────────────────────────────────────────────────
                               │
                               ▼
               ┌────────────────────────────────────┐
               │             NGINX (443)            │
               │------------------------------------│
               │ • Reverse proxy for WordPress PHP  │
               │ • Handles HTTPS traffic            │
               │ • Accessible from your host        │
               └────────────────┬───────────────────┘
                                │
        ┌───────────────────────┴───────────────────────┐
        │                                               │
        ▼                                               ▼
┌────────────────────────┐                 ┌────────────────────────┐
│    WordPress (php-fpm) │                 │     SITE (Static HTML) │
│────────────────────────│                 │────────────────────────│
│ • Listens on port 9000 │                 │ • Internal-only        │
│ • Not exposed outside  │                 │ • Served via NGINX     │
└───────────┬────────────┘                 └────────────────────────┘
            │
            ▼
┌────────────────────────┐
│      MariaDB (3306)    │
│────────────────────────│
│ • Stores WP data       │
│ • Internal-only        │
│ • Connected via        │
│   WORDPRESS_DB_HOST    │
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│        Redis (6379)    │
│────────────────────────│
│ • Internal cache layer │
│ • Used by WP plugin    │
└────────────────────────┘


┌────────────────────────┐
│       FTP (21,21100+)  │
│────────────────────────│
│ • Accessible externally│
│ • Linked to /var/www/html│
└────────────────────────┘

┌────────────────────────┐
│    Adminer (8080)      │
│────────────────────────│
│ • External DB browser  │
│ • Accessible via web   │
└────────────────────────┘

┌──────────────────────────────┐
│   Portainer (9000,9443)      │
│──────────────────────────────│
│ • Docker management UI       │
│ • Uses /var/run/docker.sock  │
└──────────────────────────────┘

              🕸 Shared Docker Network: `inception`
────────────────────────────────────────────────────────────
All containers can reach each other by name (DNS resolution).
Example: `wordpress` can reach `mariadb` using `DB_HOST=mariadb`
────────────────────────────────────────────────────────────
```
The ASCII diagram above is duplicated from `overview.txt` so that the README remains self-contained.

## Technology Stack
The stack intentionally avoids pre-built service images and instead provisions every component manually, forcing explicit knowledge of the tooling in play.

| Layer | Technology | Role | Key Details |
|-------|------------|------|-------------|
| Container Engine | Docker Engine & Docker Compose | Orchestration | Compose file is `srcs/docker-compose.yml`; all images are built from local Dockerfiles. |
| Base OS | Debian Bullseye | Common base image | Provides a minimal Linux environment for all custom images except the static site, which builds from `nginx:bullseye`. |
| Reverse proxy | NGINX | TLS terminator and routing | Serves HTTPS traffic, forwards PHP requests to WordPress, and proxies `/bonus/` to the static site. |
| Application runtime | PHP 7.4 (php-fpm) | WordPress execution | Provided via Debian packages; managed by php-fpm for better process isolation. |
| CMS | WordPress | Blogging platform | Automatically bootstrapped via `wp-cli`. |
| Database | MariaDB 10.x | WordPress relational store | Initialized with custom script that creates users and schema. |
| Cache | Redis | Object cache | Enabled via the Redis WordPress plugin for application-level caching. |
| File transfer | vsftpd | FTP server | Provides external access to the WordPress document root for file management. |
| DB admin | Adminer | Web-based DB UI | Runs on PHP's built-in web server and connects to MariaDB. |
| Docker management | Portainer | UI for Docker | Offers browser-based management of local containers. |
| Static site | NGINX (official image) | Bonus content | Ships a simple static HTML portfolio and is only reachable through the `/bonus/` path on the main NGINX server. |
| SSL tooling | OpenSSL | Certificate generation | Self-signed certificates are created at container startup. |
| Automation | GNU Make, shell scripts | Developer ergonomics | The Makefile and `build_script.sh` provide repeatable workflows. |

## Compose Orchestration
All orchestration happens through `srcs/docker-compose.yml`. Every service block builds a local Dockerfile, declares networks, ports, volumes, restart strategies, and dependencies.

### Networks
- **`inception` (bridge)** – A user-defined bridge connecting every container. Docker's built-in DNS allows addressing services by container name (e.g., `wordpress` can reach `mariadb:3306`).

### Volumes
| Volume | Driver | Host binding | Usage |
|--------|--------|--------------|-------|
| `mariadb_data` | `local` with `type=none` | `/home/tboussad/data/mariadb_data` | Persists MariaDB data directory across container restarts. |
| `wordpress_data` | `local` with `type=none` | `/home/tboussad/data/wordpress_data` | Stores WordPress core files, uploads, themes, and plugin state. |

> **Note:** The hard-coded host paths match the 42 school Linux environment. Adjust `device` paths if you run on another machine (e.g., map to `${PWD}/data/...`).

### Global Environment Expectations
Several services load secrets and configuration from a shared `.env` file (referenced via `env_file: .env`). This file is intentionally ignored by git to avoid leaking credentials. Required keys are enumerated in [Environment bootstrap](#environment-bootstrap).

## Service Reference
Each service is built from a bespoke Dockerfile under `srcs/requirements/...`. This section documents exactly what happens during image build and runtime initialization.

### NGINX
- **Context:** `srcs/requirements/nginx`
- **Base image:** `debian:bullseye`
- **Packages installed:** `nginx`, `openssl` (for self-signed TLS certificates).
- **Configuration:**
  - `conf/nginx.conf` places a single server block on port 443 with TLS 1.3 enforced.
  - It sets `root /var/www/wordpress;` and forwards PHP requests to `wordpress:9000` via `fastcgi_pass`.
  - Requests under `/bonus/` are proxied to the static site container (`http://site:80/`).
- **Entrypoint script:** `tools/nginx_setup.sh`
  - Creates `/etc/ssl/private` and `/etc/ssl/certs`.
  - Generates a self-signed certificate valid for 365 days using OpenSSL (`rsa:2048` key).
  - Runs `nginx -g "daemon off;"` so the container remains in the foreground.
- **Ports:** `443` exposed to the host.
- **Volumes:** Shares `wordpress_data` as read-only web root.
- **Dependencies:** Waits for `wordpress` and `site` to be ready (declared via `depends_on`).

### WordPress (PHP-FPM)
- **Context:** `srcs/requirements/wordpress`
- **Base image:** `debian:bullseye`
- **Packages installed:** `php`, `php-fpm`, `php-mysql`, `php-redis`, `curl`, `unzip`, `netcat`.
  - `php-redis` enables WordPress to communicate with Redis.
  - `curl` and `unzip` allow acquiring and unpacking WordPress core.
- **Additional artifacts:**
  - Downloads `wp-cli` Phar into `/usr/local/bin/wp` for headless administration.
  - Custom PHP-FPM pool configuration `www.conf` sets the listener to port 9000 and uses the `www-data` user.
  - Entrypoint script `tools/wordpress_entrypoint.sh` orchestrates installation.
- **Entrypoint workflow:**
  1. Downloads WordPress core into `/var/www/wordpress` if missing.
  2. Applies secure ownership (`www-data`) and permissions.
  3. Generates `wp-config.php` using values from the `.env` file (database credentials and host).
  4. Performs `wp core install` to provision the site, specifying admin user/email plus an additional editor account.
  5. Installs and activates the `redis-cache` plugin, injects Redis settings into `wp-config.php`, and enables cache with `wp redis enable`.
  6. Executes the container command (`php-fpm7.4 -F`) to run PHP-FPM in the foreground.
- **Ports:** Exposes `9000` internally (no host binding, it is consumed by NGINX over the Docker network).
- **Volumes:** Mounts `wordpress_data:/var/www/wordpress` for persistence and FTP sharing.
- **Environment variables consumed:**
  - `WORDPRESS_DB_NAME`, `WORDPRESS_DB_USER`, `WORDPRESS_DB_PASSWORD`, `WORDPRESS_DB_HOST`
  - `WP_SITE_URL`, `WP_SITE_TITLE`
  - `WP_ADMIN_USER`, `WP_ADMIN_PASSWORD`, `WP_ADMIN_EMAIL`
  - `WP_EDITOR_USER`, `WP_EDITOR_PASSWORD`, `WP_EDITOR_EMAIL`

### MariaDB
- **Context:** `srcs/requirements/mariadb`
- **Base image:** `debian:bullseye`
- **Packages installed:** `mariadb-server`.
- **Configuration:**
  - `conf/mariadb.conf` overrides defaults to listen on every interface (`0.0.0.0`) on port 3306.
  - The Dockerfile prepares data directories (`/run/mysqld`, `/var/lib/mysql`) and assigns appropriate ownership.
- **Entrypoint script:** `tools/mariadb_setup.sh`
  - Ensures secure permissions (`chmod 700 /var/lib/mysql`).
  - Detects an empty data directory and runs `mysql_install_db` on first boot.
  - Launches MariaDB in the background using `mysqld_safe`, waits for readiness via `mysqladmin ping`.
  - Executes SQL to create the WordPress database, a regular user (`$DB_USER`/`$DB_PASS`), and an elevated superuser (`$DB_SUPER`/`$DB_SUPER_PASSWORD`).
  - Flushes privileges and gracefully stops the temporary instance before starting the final foreground `mysqld_safe` process.
- **Ports:** 3306 (not published to host).
- **Volumes:** `mariadb_data:/var/lib/mysql`.
- **Healthcheck:** Uses `mysqladmin ping` every 10 seconds with a 5-second timeout to ensure readiness for dependent services.
- **Environment variables consumed:** `DB_NAME`, `DB_USER`, `DB_PASS`, `DB_SUPER`, `DB_SUPER_PASSWORD`.

### Redis
- **Context:** `srcs/requirements/bonus/redis`
- **Base image:** `debian:bullseye`
- **Packages installed:** `redis-server`.
- **Configuration:**
  - `conf/redis.conf` disables protected mode (allowing network connections inside the Docker network), listens on all interfaces, enforces a 256MB memory limit, and selects `allkeys-lru` eviction to maintain cache efficiency.
- **Command:** Runs `redis-server /etc/redis/redis.conf`.
- **Ports:** 6379 (internal only).
- **Purpose:** Provides a fast, in-memory cache that the WordPress Redis plugin uses for object caching.

### FTP (vsftpd)
- **Context:** `srcs/requirements/bonus/ftp`
- **Base image:** `debian:bullseye`
- **Packages installed:** `vsftpd`.
- **Configuration:**
  - `conf/vsftpd.conf` enables local user login, write access, chroot jail, and passive mode across ports 21100–21110.
  - Passive address is set to `0.0.0.0` so Docker handles NAT translation.
- **Entrypoint script:** `tools/ftp_setup.sh`
  - Ensures the secure chroot directory exists.
  - Creates a system user (`$FTPUSER`) whose home directory is `/var/www/wordpress`, aligning FTP operations with the WordPress document root shared via `wordpress_data`.
  - Sets the user password from `$FTPPASS` and takes ownership of the WordPress files for upload capability.
  - Starts `vsftpd` with the custom configuration.
- **Ports:**
  - Control channel: `21:21` (host exposed).
  - Passive range: `21100-21110` mapped directly for clients behind firewalls.
- **Volumes:** `wordpress_data` (shared with WordPress container).
- **Environment variables consumed:** `FTPUSER`, `FTPPASS`.

### Adminer
- **Context:** `srcs/requirements/bonus/adminer`
- **Base image:** `debian:bullseye`
- **Packages installed:** `php`, `php-mysqli`, `curl`.
- **Build steps:** Downloads the latest Adminer PHP single-file application into `/var/www/adminer/index.php`.
- **Runtime:** Executes `php -S 0.0.0.0:8080 -t /var/www/adminer`, leveraging PHP's built-in development server (sufficient for a bounded admin tool inside Docker).
- **Ports:** `8080:8080` so the tool is reachable from the host.
- **Usage:** Use your browser to connect to `https://localhost:8080` (or the mapped host) and log into the MariaDB instance at host `mariadb` with the credentials specified in `.env`.

### Portainer
- **Context:** `srcs/requirements/bonus/portainer`
- **Base image:** `debian:bullseye`
- **Packages installed:** `curl`, `ca-certificates`, `tar`.
- **Build steps:**
  - Downloads the Portainer Community Edition binary tarball (`2.21.3`) directly from GitHub.
  - Extracts contents to `/opt/portainer` and removes the archive.
- **Runtime:** Launches `/opt/portainer/portainer`.
- **Ports:**
  - `9443:9443` (HTTPS UI)
  - `9000:9000` (legacy HTTP UI)
- **Volumes:** Binds `/var/run/docker.sock` so Portainer can manage the Docker engine on the host. **This grants Docker root-level access—handle with care.**

### Static Site
- **Context:** `srcs/requirements/bonus/site`
- **Base image:** `nginx:bullseye`
- **Packages:** Updates the base image via `apt update && apt upgrade -y`.
- **Content:** Copies `tools/site.html` as `/usr/share/nginx/html/index.html`. The HTML file is a minimalist personal landing page.
- **Runtime:** Runs the default `nginx` foreground command.
- **Ports:** Exposes port 80 internally (no direct host binding; consumed by the main NGINX service under `/bonus/`).

## Configuration Files & Scripts
| Path | Purpose |
|------|---------|
| `overview.txt` | ASCII network topology diagram replicated in this README. |
| `Makefile` | Defines automation targets: `build`, `up`, `down`, `rebuild`. Uses `docker compose -f srcs/docker-compose.yml`. |
| `build_script.sh` | Convenience script that calls `make rebuild`, then prints logs for each container sequentially.| 
| `srcs/docker-compose.yml` | Central orchestration manifest. |
| `srcs/requirements/<service>/Dockerfile` | Service-specific image definitions. |
| `srcs/requirements/<service>/conf/*` | Runtime configuration files (e.g., nginx.conf, mariadb.conf, vsftpd.conf, redis.conf). |
| `srcs/requirements/<service>/tools/*` | Entrypoint scripts or static assets (WordPress entrypoint, nginx_setup.sh, etc.). |
| `srcs/requirements/wordpress/www.conf` | Custom PHP-FPM pool configuration. |
| `srcs/requirements/bonus/site/tools/site.html` | Static HTML served at `/bonus/`. |
| `.gitignore` | Added in this commit; keeps secrets, cache files, and runtime artefacts out of version control. |

## Building & Running the Stack
All commands assume Docker Engine 24.x+ and Docker Compose plugin 2.x+ are installed.

### Day-to-day workflow
1. **Build images**
   ```bash
   make build
   ```
2. **Start (rebuild + run in background)**
   ```bash
   make up
   ```
   The `--build` flag in the Makefile guarantees images stay in sync with Dockerfiles.
3. **Stop everything**
   ```bash
   make down
   ```
4. **Full rebuild** (stop, rebuild, start)
   ```bash
   make rebuild
   ```

### Environment bootstrap
Create a `.env` file in the repository root before launching Compose. Below is a comprehensive template covering every variable consumed across services:

```ini
# MariaDB
DB_NAME=wordpress
DB_USER=wp_user
DB_PASS=wp_password
DB_SUPER=wp_super
DB_SUPER_PASSWORD=wp_super_password

# WordPress core
WORDPRESS_DB_NAME=${DB_NAME}
WORDPRESS_DB_USER=${DB_USER}
WORDPRESS_DB_PASSWORD=${DB_PASS}
WORDPRESS_DB_HOST=mariadb
WP_SITE_URL=https://localhost
WP_SITE_TITLE=My Inception Site
WP_ADMIN_USER=admin
WP_ADMIN_PASSWORD=change_me_admin
WP_ADMIN_EMAIL=admin@example.com
WP_EDITOR_USER=editor
WP_EDITOR_PASSWORD=change_me_editor
WP_EDITOR_EMAIL=editor@example.com

# FTP access (shares wordpress_data volume)
FTPUSER=ftpuser
FTPPASS=change_me_ftp
```
Feel free to extend this file with more service-specific variables if you add new features.

### Log inspection
The helper script `./build_script.sh` rebuilds the stack and prints logs from every container with sleep intervals for readability. For ad-hoc checks you can still run `docker logs <service>` manually.

## Security & TLS Notes
- Certificates generated by `nginx_setup.sh` are **self-signed**. Browsers will show a warning unless you import the certificate or replace it with one generated via a trusted CA (e.g., Let’s Encrypt). Autogeneration keeps builds deterministic but is insecure for production use.
- Portainer mounts the Docker socket, granting administrative control over the host. Restrict access to the Portainer UI and rotate the admin password after first login.
- FTP transmits credentials in plaintext. Consider replacing it with SFTP (over SSH) if you need secure file transfer in production.
- Database credentials reside in `.env`; treat the file as secret material and never commit it (the new `.gitignore` enforces that).
- Redis `protected-mode` is disabled, so ensure the Docker network remains private.

## Data Persistence & Host Bindings
- `mariadb_data` and `wordpress_data` volumes bind into `/home/tboussad/data/...` to satisfy 42 school grading scripts. On other systems you can customize these paths (edit `device` in `docker-compose.yml`) to live under the repository, such as `${PWD}/data/mariadb`.
- The WordPress directory is deliberately shared with the FTP container so web uploads appear instantly for FTP users.
- MariaDB data directory persists between runs. If you need a clean slate, delete the host directory and restart the stack.

## Troubleshooting Checklist
- **Containers restart continuously** – Run `docker compose -f srcs/docker-compose.yml logs <service>` to inspect stack traces. For database issues verify `.env` credentials and confirm the host path is writable.
- **MariaDB fails healthcheck** – Ensure your host machine isn’t blocking port 3306 internally and that the volume directory has correct permissions.
- **WordPress setup hangs** – Confirm Redis and MariaDB containers are reachable (`docker exec -it wordpress ping mariadb`). Also verify DNS resolution through the `inception` network.
- **FTP passive mode fails** – Open firewall ports 21100–21110 or adjust the passive range in `vsftpd.conf`.
- **Browser warns about certificate** – Import `/etc/ssl/certs/nginx.crt` from inside the NGINX container or configure a trusted certificate.
- **Portainer cannot connect to Docker** – Ensure `/var/run/docker.sock` exists on the host and Docker is running with sufficient permissions.

## Further Improvements
- Provide scripts to switch volume host paths dynamically based on environment variables.
- Replace self-signed TLS with ACME automation (e.g., certbot using DNS-01 challenges).
- Add monitoring (Prometheus + Grafana) to observe service health.
- Replace FTP with SFTP or WebDAV for encrypted file transfers.
- Expand the static site with build tooling (Eleventy, Hugo) for richer bonus content.

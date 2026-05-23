# Raspberry Pi Home Server with Docker

![Docker](https://img.shields.io/badge/Docker-Ready-blue?logo=docker)
![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-4B-red?logo=raspberrypi)
![Maintained](https://img.shields.io/badge/Maintained-Yes-brightgreen)
![License](https://img.shields.io/badge/License-MIT-green)

This repository documents my Raspberry Pi home server setup, including Docker Compose files, service configurations, backup scripts, and notes from my own deployment.

The goal of this project is to keep everything self-hosted, easy to rebuild, and simple to maintain.

---

# Services Overview

## Networking & Access

| Service                                                     | Purpose                             | Repo                                                           |
| ----------------------------------------------------------- | ----------------------------------- | -------------------------------------------------------------- |
| [**Nginx Proxy Manager**](#nginx-proxy-manager)             | Reverse proxy with SSL support      | [↗︎](https://github.com/Nginxproxymanager/Nginx-Proxy-Manager) |
| [**WireGuard**](#wireguard-vpn)                             | Secure remote access to the LAN     | [↗︎](https://www.wireguard.com/)                               |
| [**Gluetun**](#gluetun)                                     | VPN routing for selected containers | [↗︎](https://github.com/qdm12/gluetun)                         |
| [**OpenSpeedTest**](#openspeedtest)                         | Local network speed testing         | [↗︎](https://openspeedtest.com/selfhosted-speedtest)           |
| [**UniFi Network Application**](#unifi-network-application) | UniFi controller                    | [↗︎](https://github.com/linuxserver/docker-unifi-network-application)|

## Security & Privacy

| Service                                    | Purpose                                  | Repo                                             |
| ------------------------------------------ | ---------------------------------------- | ------------------------------------------------ |
| [**AdGuard Home**](#adguard-home)          | DNS-level ad and telemetry blocking      | [↗︎](https://github.com/AdguardTeam/AdguardHome) |
| [**Vaultwarden**](#vaultwardenvaultwarden) | Lightweight self-hosted password manager | [↗︎](https://github.com/dani-garcia/vaultwarden) |

## Media Automation

| Service                        | Purpose                       | Repo                                                    |
| ------------------------------ | ----------------------------- | ------------------------------------------------------- |
| [**Overseerr**](#arr-stack)    | Media request management      | [↗︎](https://github.com/sct/overseerr)                  |
| [**Radarr**](#arr-stack)       | Movie automation              | [↗︎](https://github.com/Radarr/Radarr)                  |
| [**Sonarr**](#arr-stack)       | TV show automation            | [↗︎](https://github.com/Sonarr/Sonarr)                  |
| [**Prowlarr**](#arr-stack)     | Indexer management            | [↗︎](https://github.com/Prowlarr/Prowlarr)              |
| [**Flaresolverr**](#arr-stack) | Cloudflare challenge handling | [↗︎](https://github.com/FlareSolverr/FlareSolverr)      |
| [**qBittorrent**](#arr-stack)  | Torrent client                | [↗︎](https://github.com/linuxserver/docker-qbittorrent) |

## Infrastructure & Management

| Service                         | Purpose                     | Repo                                                 |
| ------------------------------- | --------------------------- | ---------------------------------------------------- |
| [**Portainer**](#portainer)     | Docker management UI        | [↗︎](https://github.com/Portainer/Portainer)         |
| [**Watchtower**](#watchtower)   | Automatic container updates | [↗︎](https://github.com/Containrrr/Watchtower)       |
| [**Filebrowser**](#filebrowser) | Web-based file manager      | [↗︎](https://github.com/hurlenko/filebrowser-docker) |
| [**Homepage**](#homepage)       | Service dashboard           | [↗︎](https://gethomepage.dev/)                       |

## Productivity

| Service                                     | Purpose                | Repo                                              |
| ------------------------------------------- | ---------------------- | ------------------------------------------------- |
| [**Obsidian LiveSync**](#obsidian-livesync) | Encrypted note syncing | [↗︎](https://github.com/vrtmrz/obsidian-livesync) |

## Automation

| Script                                       | Purpose                                               |
| -------------------------------------------- | ----------------------------------------------------- |
| [**cpu_temp.sh**](#cpu-temperature-alerts)               | Sends Discord alerts if CPU temperature gets too high |
| [**backup-obsidian.sh**](#obsidian-backup-script) | Backs up Obsidian LiveSync data to NAS                |
| [**docker-backup.sh**](#docker-git-backup-script)     | Pushes Docker configs to GitHub                       |

---

# Notes

* All services run inside Docker containers for portability and easier management.
* Most services are available internally through Nginx Proxy Manager.
* Privacy-sensitive services can be routed through Gluetun.
* Media services share the same storage structure to support hardlinking.

---

# Prerequisites

* A Raspberry Pi with an installed and updated OS
* A static IP address recommended for easier access & configuration

I use Raspberry Pi OS 64-bit based on Debian, but most Linux distributions should work with small adjustments.

---

# Installing Docker

Docker allows applications to run inside lightweight containers, making deployments easier to manage and rebuild.

Docker Compose lets you manage multiple containers using a single YAML file.

Installation steps vary depending on your OS. Refer to Docker's official documentation for detailed instructions.

## Docker on Debian

```yaml
curl -sSl https://get.docker.com | sh
```

To avoid running Docker as root:

```yaml
sudo usermod -aG docker ${USER}
```

Log out and back in for the group change to apply.

Verify Docker:

```yaml
docker run hello-world
```

Install Docker Compose:

```yaml
sudo apt install docker-compose-plugin
```

Verify Docker Compose:

```yaml
docker compose version
```

---

# Folder Structure

I keep each container or stack inside its own directory:

```text
~/containers/nginx-proxy-manager
~/containers/adguard
~/containers/arr-stack
```

Each directory contains its own `docker-compose.yml` file.

To start a stack:

```yaml
docker compose up -d
```

---

# NGINX Proxy Manager

Nginx Proxy Manager makes it easy to manage reverse proxies and SSL certificates through a web UI.

Example:

Instead of opening services using IP addresses like:

```text
http://192.168.1.10:3000
```

You can use internal domains such as:

```text
portainer.lan
adguard.lan
```

I followed Wolfgang's Channel on YouTube for the initial setup using deSEC DNS. [video](https://www.youtube.com/watch?v=qlcVx-k-02E)

```yaml
services:
  nginx_proxy_manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx_proxy_manager
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - ./config.json:/app/config/production.json
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    networks:
      - proxy
    restart: unless-stopped

networks:
  proxy:
    name: proxy
```

Access the web UI:

```text
http://<raspberrypi-ip>:81
```

---

# AdGuard Home

AdGuard Home acts as a DNS sinkhole, blocking ads, trackers, and telemetry requests across your network.

```yaml
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguard
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      # - "67:67/udp"
      # - "68:68/udp"
      - "3003:3000/tcp"
      - "8062:80/tcp"
      - "445:443"
      - "853:853/tcp"
      - "5443:5443"
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
    restart: unless-stopped
    networks:
      - adguard
      - proxy

networks:
  adguard:
    name: adguard

  proxy:
    external: true
```

Initial setup:

```text
http://<raspberrypi-ip>:3003
```

Main interface after setup:

```text
http://<raspberrypi-ip>:8062
```

I use DNS rewrites to point local domains such as `*.lan` to my reverse proxy.

---

# Portainer

Portainer provides a simple web UI for managing Docker containers, images, networks, and volumes.

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    volumes:
      - data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "9443:9443"
    networks:
      - portainer
      - proxy
    restart: unless-stopped

volumes:
  data:

networks:
  portainer:
    name: portainer
  proxy:
    external: true
```

Access:

```text
https://<raspberrypi-ip>:9443
```

---

# Vaultwarden

Vaultwarden is a lightweight self-hosted password manager, compatible with the Bitwarden apps and browser extensions.

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: bitwarden
    environment:
      - WEBSOCKET_ENABLED=true
    volumes:
      - ./data:/data
    ports:
      - "8085:80"
    restart: unless-stopped
    networks:
      - bitwarden
      - proxy

networks:
  bitwarden:
    name: bitwarden
    internal: true

  proxy:
    external: true
```

Vaultwarden should be exposed through HTTPS, otherwise some Bitwarden features may not work correctly.

I handle this using Nginx Proxy Manager.

---

# WireGuard VPN

WireGuard gives secure remote access to my LAN and internal services while away from home.

```yaml
services:
  wireguard:
    image: linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Stockholm
      - SERVERURL=auto
      - SERVERPORT=51820
      - PEERS=3
      - PEERDNS=192.168.1.10
      - INTERNAL_SUBNET=10.13.13.0
      - ALLOWEDIPS=0.0.0.0/0
    volumes:
      - ./config:/config
      - /lib/modules:/lib/modules
    ports:
      - "51820:51820/udp"
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv4.ip_forward=1
    restart: unless-stopped
    networks:
      - wireguard

networks:
  wireguard:
    name: wireguard
```

After setup, generate a client QR code with:

```yaml
docker exec -it wireguard /app/show-peer <peer-name>
```

You can scan the QR code directly in the WireGuard mobile app.

---

# Watchtower

Watchtower automatically checks for updated container images and redeploys containers.

```yaml
services:
  watchtower:
    image: nickfedor/watchtower:latest
    container_name: watchtower
    environment:
      TZ: Europe/Stockholm
      WATCHTOWER_SCHEDULE: "0 0 0 * * 0"
      WATCHTOWER_CLEANUP: "true"
      WATCHTOWER_INCLUDE_RESTARTING: "true"
      WATCHTOWER_DISABLE_CONTAINERS: watchtower,gluetun
    ports:
      - "86:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - watchtower
    restart: unless-stopped

networks:
  watchtower:
    name: watchtower
```

I had issues letting watchtower update itself or gluetun so watchtower ignores those two containers.

The docker compose above runs updates weekly.

---

# Filebrowser

Filebrowser is a lightweight web-based file manager.

```yaml
services:
  filebrowser:
    image: hurlenko/filebrowser
    container_name: filebrowser
    environment:
      - FB_BASE_URL=/filebrowser
    volumes:
      - ./DATA_DIR:/data
      - ./CONFIG_DIR:/config
    ports:
      - "444:8080"
    networks:
      - filebrowser
      - proxy
    restart: unless-stopped

networks:
  filebrowser:
    name: filebrowser
  proxy:
    external: true
```

If a default password is generated, you can view it with:

```yaml
docker logs -f filebrowser
```

---

# Obsidian LiveSync

Obsidian LiveSync allows encrypted syncing between devices using CouchDB for obsidian note taking.

```yaml
services:
  couchdb-obsidian-livesync:
    image: couchdb:latest
    container_name: obsidian-livesync
    environment:
      - TZ=Europe/Stockholm
      - COUCHDB_USER=admin
      - COUCHDB_PASSWORD=/run/secrets/couchdb_password
    secrets:
      - couchdb_password
    volumes:
      - ./data:/opt/couchdb/data
      - ./etc:/opt/couchdb/etc/local.d
    ports:
      - "5984:5984"
    networks:
      - obsidian
    restart: unless-stopped

networks:
  obsidian:
    name: obsidian

secrets:
  couchdb_password:
    file: ./secrets/couchdb_password.txt
```

I recommend following the dedicated setup guide for configuring the plugin and database made by Timely_Anteater_9330. [Guide](https://www.reddit.com/r/selfhosted/comments/1eo7knj/guide_obsidian_with_free_selfhosted_instant_sync/)

---

# Gluetun

Gluetun routes selected containers through a VPN tunnel.

I mainly use it for torrent-related containers.

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    secrets:
      - vpn_user
      - vpn_password
    volumes:
      - ./data:/gluetun
    environment:
      - TZ=Europe/Stockholm
      - VPN_SERVICE_PROVIDER=
      - VPN_TYPE=
      - OPENVPN_USER_FILE=/run/secrets/vpn_user
      - OPENVPN_PASSWORD_FILE=/run/secrets/vpn_password
      - SERVER_REGIONS=Switzerland
      - FIREWALL_OUTBOUND_SUBNETS=192.168.1.0/24
    ports:
      - "8081:8081"
      - "6881:6881"
      - "6881:6881/udp"
      - "9696:9696"
      - "8191:8191"
      - "8000:8000"
    restart: unless-stopped
    networks:
      - gluetun

networks:
  gluetun:
    name: gluetun

secrets:
  vpn_user:
    file: ./secrets/vpn_user.txt
  vpn_password:
    file: ./secrets/vpn_password.txt
```

Containers using:

```yaml
network_mode: "container:gluetun"
```

share the same VPN connection.

---

# arr Stack

The *arr stack automates downloading, organizing, and managing media.

My setup uses:

* Radarr
* Sonarr
* Prowlarr
* qBittorrent
* Overseerr
* FlareSolverr

I run these in a single Docker Compose stack because they depend heavily on each other.

---

# Hardlinks & Storage Layout

All containers reference the same root mount (`/mnt/nas`) so hardlinking works correctly.

Example structure:

```text
nas
├── downloads
│   ├── radarr
│   └── tv-sonarr
├── movies
└── tv-shows
```

Hardlinks let media managers move files instantly without duplicating storage usage.

You can verify hardlinks with:

```yaml
ls -l
```

or compare inode numbers:

```yaml
stat filename.mkv
```

If both files share the same inode, hardlinking works correctly.

---

```yaml
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=022
      - TZ=Europe/Stockholm
    volumes:
      - radarr_config:/config
      - /mnt/nas:/nas
    ports:
      - "7878:7878"
    networks:
      - arr
      - proxy
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=022
      - TZ=Europe/Stockholm
    volumes:
      - sonarr_config:/config
      - /mnt/nas:/nas
    ports:
      - "8989:8989"
    networks:
      - arr
      - proxy
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=022
      - TZ=Europe/Stockholm
      - WEBUI_PORT=8081
      - TORRENTING_PORT=6881
    volumes:
      - qbittorrent_config:/config
      - /mnt/nas/downloads:/nas/downloads
    network_mode: "container:gluetun"
    restart: always

  seerr:
    image: ghcr.io/seerr-team/seerr:latest
    container_name: seerr
    environment:
      - LOG_LEVEL=debug
      - TZ=Europe/Stockholm
    volumes:
      - seerr_config:/app/config
    ports:
      - "5055:5055"
    dns:
      - 1.1.1.1
      - 8.8.8.8
    networks:
      - arr
      - proxy
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=022
      - TZ=Europe/Stockholm
    volumes:
      - prowlarr_config:/config
    network_mode: "container:gluetun"
    restart: always

  flaresolverr:
    image: flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - LOG_HTML=${LOG_HTML:-false}
      - CAPTCHA_SOLVER=${CAPTCHA_SOLVER:-none}
      - TZ=Europe/Stockholm
    volumes:
      - flaresolverr_config:/config
    network_mode: "container:gluetun"
    restart: always

networks:
  arr:
    name: arr
  proxy:
    external: true

volumes:
  radarr_config:
  sonarr_config:
  seerr_config:
  prowlarr_config:
  flaresolverr_config:
  qbittorrent_config:
```

# OpenSpeedTest

```yaml
services:
  speedtest:
    image: openspeedtest/latest
    container_name: openspeedtest
    restart: unless-stopped
    ports:
      - "3002:3002"
      - "3001:3001"
    networks:
      - proxy

networks:
  proxy:
    external: true
```

# UniFi Network Application

```yaml
services:
  unifi-db:
    image: mongo:4.4.18
    container_name: unifi-db
    network_mode: host

    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_INITDB_ROOT_USERNAME}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_INITDB_ROOT_PASSWORD}
      - MONGO_USER=${MONGO_USER}
      - MONGO_PASS=${MONGO_PASS}
      - MONGO_DBNAME=${MONGO_DBNAME}
      - MONGO_AUTHSOURCE=${MONGO_AUTHSOURCE}

    volumes:
      - ./mongo/data:/data/db
      - ./init-mongo.sh:/docker-entrypoint-initdb.d/init-mongo.sh:ro

    restart: unless-stopped

  unifi-network-application:
    image: lscr.io/linuxserver/unifi-network-application:latest
    container_name: unifi
    network_mode: host

    depends_on:
      - unifi-db

    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}

      - MONGO_USER=${MONGO_USER}
      - MONGO_PASS=${MONGO_PASS}
      - MONGO_HOST=127.0.0.1
      - MONGO_PORT=27017
      - MONGO_DBNAME=${MONGO_DBNAME}
      - MONGO_AUTHSOURCE=${MONGO_AUTHSOURCE}

    volumes:
      - ./unifi:/config

    restart: unless-stopped
```

init-mongo.sh

```bash
#!/bin/bash

if which mongosh > /dev/null 2>&1; then
  mongo_init_bin='mongosh'
else
  mongo_init_bin='mongo'
fi
"${mongo_init_bin}" <<EOF
use ${MONGO_AUTHSOURCE}
db.auth("${MONGO_INITDB_ROOT_USERNAME}", "${MONGO_INITDB_ROOT_PASSWORD}")
db.createUser({
  user: "${MONGO_USER}",
  pwd: "${MONGO_PASS}",
  roles: [
    "clusterMonitor",
    { db: "${MONGO_DBNAME}", role: "dbOwner" },
    { db: "${MONGO_DBNAME}_stat", role: "dbOwner" },
    { db: "${MONGO_DBNAME}_audit", role: "dbOwner" },
    { db: "${MONGO_DBNAME}_restore", role: "dbOwner" }
  ]
})
EOF
```

.env

```shell
# ===== System =====
PUID=1000
PGID=1000
TZ=TZ

# ===== MongoDB root =====
MONGO_INITDB_ROOT_USERNAME=root
MONGO_INITDB_ROOT_PASSWORD=<strong password>

# ===== UniFi Mongo user =====
MONGO_USER=unifi
MONGO_PASS=<strong password>
MONGO_DBNAME=unifi
MONGO_AUTHSOURCE=admin

# ===== Mongo connection =====
MONGO_HOST=unifi-db
MONGO_PORT=27017
.env
```

# CPU Temperature Alerts

I use a simple yaml script with Discord webhooks to notify me if the Raspberry Pi overheats.

The script:

* Sends an initial alert when the CPU exceeds a threshold
* Sends additional alerts every 5°C increase
* Resets automatically once temperatures drop

I run the script using a systemd timer every 10 minutes.

```yaml
#!/bin/yaml

BASE_TEMP=60
STEP=5
STATE_FILE="/tmp/pi_temp_alert.state"

# get CPU temperature in Celsius (rounded down)
pi_temp=$(vcgencmd measure_temp | awk -F "[=.'']" '{print int($2)}')
this_pi=$(hostname)

discord_pi_webhook=

# If below base temp → reset state
if [[ "$pi_temp" -lt "$BASE_TEMP" ]]; then
  rm -f "$STATE_FILE"
  exit 0
fi

# First alert at or above BASE_TEMP
if [[ ! -f "$STATE_FILE" ]]; then
  echo "$pi_temp" > "$STATE_FILE"

  curl -H "Content-Type: application/json" -X POST \
    -d "{\"content\":\"🚨 ALERT! ${this_pi} CPU temp reached ${pi_temp}°C\"}" \
    "$discord_pi_webhook"
  exit 0
fi

last_alert_temp=$(cat "$STATE_FILE")
temp_diff=$((pi_temp - last_alert_temp))

# Alert only if temp increased by STEP
if [[ "$temp_diff" -ge "$STEP" ]]; then
  echo "$pi_temp" > "$STATE_FILE"

  curl -H "Content-Type: application/json" -X POST \
    -d "{\"content\":\"⚠️ UPDATE! ${this_pi} CPU temp increased to ${pi_temp}°C\"}" \
    "$discord_pi_webhook"
fi
```

---

# Obsidian Backup Script

This script:

* Stops the container
* Creates a timestamped backup using `rsync`
* Restarts the container
* Removes old backups automatically

Backups are stored on my NAS.

```yaml
#!/bin/yaml

# ── Config ────────────────────────────────────────────────────────────────────
COMPOSE_DIR="/home/admin/containers/obsidian-livesync"
BACKUP_ROOT="/mnt/nas/backups/obsidian-livesync"
RETENTION_DAYS=30
LOG_FILE="/var/log/obsidian-backup.log"
# ─────────────────────────────────────────────────────────────────────────────

TIMESTAMP=$(date +"%Y-%m-%d_%H-%M")
BACKUP_DIR="$BACKUP_ROOT/$TIMESTAMP"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "──────────────────────────────────────────"
log "Starting obsidian-livesync backup"

# Stop the container
log "Stopping container..."
docker compose -f "$COMPOSE_DIR/docker-compose.yml" stop
if [ $? -ne 0 ]; then
  log "ERROR: Failed to stop container. Aborting backup."
  exit 1
fi

# Sync data and config
log "Rsyncing to $BACKUP_DIR ..."
mkdir -p "$BACKUP_DIR/data" "$BACKUP_DIR/etc"
rsync -a --delete "$COMPOSE_DIR/data/" "$BACKUP_DIR/data/"
DATA_STATUS=$?
rsync -a --delete "$COMPOSE_DIR/etc/" "$BACKUP_DIR/etc/"
ETC_STATUS=$?

RSYNC_STATUS=$(( DATA_STATUS + ETC_STATUS ))

# Always restart the container, even if rsync failed
log "Restarting container..."
docker compose -f "$COMPOSE_DIR/docker-compose.yml" start

if [ $RSYNC_STATUS -ne 0 ]; then
  log "ERROR: rsync failed with exit code $RSYNC_STATUS."
  exit 1
fi

log "Backup complete → $BACKUP_DIR"

# Prune backups older than retention period
log "Pruning backups older than $RETENTION_DAYS days..."
find "$BACKUP_ROOT" -maxdepth 1 -type d -mtime +$RETENTION_DAYS -exec rm -rf {} +

log "Done."
```

---

# Docker Git Backup Script

I use GitHub as an off-site backup for my Docker Compose files.

The script:

* Adds changed files
* Creates a commit automatically
* Pushes updates to GitHub

Sensitive files are excluded using `.gitignore`.

```yaml
#!/bin/yaml
cd ~/containers
git add -A
git diff --cached --quiet && echo "No changes, skipping commit." && exit 0
git commit -m "Auto backup $(date '+%Y-%m-%d %H:%M')"
git push origin main
```

---

# Example `.gitignore`

```gitignore
# Ignore everything by default
*

# Allow folders
!*/

# Allow Docker Compose files
!*/docker-compose.yml
!*/docker-compose.yaml

# Ignore secrets and environment files
*.env
*.txt

# Ignore persistent application data
*/letsencrypt/*
*/DATA_DIR/*
*/data
*/work

# Ignore additional YAML configs
*.yaml

# Keep root files
!.gitignore
```

---

# NAS Mount

Example `/etc/fstab` entry for mounting a Synology NAS using NFS:

```fstab
192.168.1.50:/volume1/nas /mnt/nas nfs defaults,nofail,_netdev,x-systemd.automount,x-systemd.device-timeout=30 0 0```
---

# Final Notes

This setup has changed a lot over time and will probably continue evolving.

The repository mainly exists as:

* Documentation for my own rebuilds
* A backup of working configurations
* A reference for anyone building something similar

Feel free to adapt any part of it for your own setup.

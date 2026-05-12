# <img src="https://github.com/user-attachments/assets/39d7950d-8c68-4845-a20c-97ba42d940cd" width="60" alt="logo">Raspberry Pi Home Server with Docker

![Docker](https://img.shields.io/badge/Docker-Ready-blue?logo=docker)
![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-4B-red?logo=raspberrypi)
![Maintained](https://img.shields.io/badge/Maintained-Yes-brightgreen)
![License](https://img.shields.io/badge/License-MIT-green)

This repository documents my raspberry pi server setup, covering configuration steps, docker compose files, and other useful resources. 


## 🧩 Services Overview

This section lists all services included in this Raspberry Pi setup.

---

## 🌐 Networking & Access

| Service | Description | Repository |
|----------|-------------|-------------|
| **Nginx Proxy Manager** | Reverse proxy with SSL management and domain routing for internal services | [Repository ↗︎](https://github.com/Nginxproxymanager/Nginx-Proxy-Manager) |
| **WireGuard** | Secure VPN for remote access to LAN and internal services | [Repository ↗︎](https://www.wireguard.com/) |
| **Gluetun** | Routes selected container traffic through a VPN provider for privacy | [Repository ↗︎](https://github.com/qdm12/gluetun) |

---

## 🔐 Security & Privacy

| Service | Description | Repository |
|----------|-------------|-------------|
| **Adguard Home** | Network-wide DNS filtering and ad blocking via DNS sinkhole | [Repository ↗︎](https://github.com/AdguardTeam/AdguardHome) |
| **Vaultwarden** | Lightweight self-hosted password manager compatible with Bitwarden clients | [Repository ↗︎](https://github.com/dani-garcia/vaultwarden) |

---

## 📺 Media Automation (*arr Stack)

| Service | Description | Repository |
|----------|-------------|-------------|
| **Overseerr** | Media request management interface for Radarr and Sonarr | [Repository ↗︎](https://github.com/sct/overseerr) |
| **Radarr** | Automated movie downloading and organization | [Repository ↗︎](https://github.com/Radarr/Radarr) |
| **Sonarr** | Automated TV show downloading and organization | [Repository ↗︎](https://github.com/Sonarr/Sonarr) |
| **Prowlarr** | Indexer manager and proxy for *arr applications | [Repository ↗︎](https://github.com/Prowlarr/Prowlarr) |
| **Flaresolverr** | Bypass tool for Cloudflare-protected indexers | [Repository ↗︎](https://github.com/FlareSolverr/FlareSolverr) |
| **qBittorrent** | Torrent client used for downloading media (often routed via VPN) | [Repository ↗︎](https://github.com/linuxserver/docker-qbittorrent) |

---

## ⚙️ Infrastructure & Management

| Service | Description | Repository |
|----------|-------------|-------------|
| **Portainer** | Web-based Docker management interface for containers, networks, and volumes | [Repository ↗︎](https://github.com/Portainer/Portainer) |
| **Watchtower** | Automatically updates running Docker containers to latest images | [Repository ↗︎](https://github.com/Containrrr/Watchtower) |
| **Filebrowser** | Lightweight web file manager for browsing and managing files on the Pi | [Repository ↗︎](https://github.com/hurlenko/filebrowser-docker) |

---

## 🧠 Productivity

| Service | Description | Repository |
|----------|-------------|-------------|
| **Obsidian LiveSync** | Self-hosted encrypted sync service for Obsidian notes across devices | [Repository ↗︎](https://github.com/vrtmrz/obsidian-livesync) |

## 🤖  Automation

| Service | Description  |
|----------|-------------|
| **cpu_temp.sh** | Receive alert on discord if pi is overheating |
| **backup-obsidian.sh** | Backup obsidian database to NAS |
| **docker-backup.sh** | Backup docker compose files to github using git |

---

## 📦 Notes

- All services run inside Docker containers for isolation and portability.
- Most services are accessible internally via reverse proxy (Nginx Proxy Manager).
- VPN routing is used selectively for privacy-sensitive services (via Gluetun).
- Media automation services share a common storage structure to support hardlinking.

## Prerequisites
* Enough storage for your needs (Example: I used a 128GB microSD card with 20% used for all my services).
* Raspberry Pi with an installed and updated OS (I used [Raspberry Pi OS](https://www.raspberrypi.com/software/) 64-bit, based on Debian, but any OS should work as long as you know the commands for it).
* Static IP: Recommended for consistent access.


## Installing Docker
To get started, install ```docker``` and ```docker-compose```.

Docker is a tool that simplifies application deployment in lightweight containers. Containers share the same OS resources, so they’re more efficient than virtual machines.

Docker compose is a tool that simplifies the setup of multiple docker containers through YAML configuration files.

   Note: Installation varies by OS. Refer to [Docker's Official Site](https://docs.docker.com/desktop/install/debian/) for detailed instructions on your specific OS.
   
#### Docker on Debian 
``` Bash
curl -sSl https://get.docker.com | sh
```
To avoid running Docker commands as root, add your user to the Docker group:
``` Bash
sudo usermod -aG docker ${whoami}
```
You may need to log out and back in for this to take effect.

Verify that Docker is installed by running your first container:
``` Bash
sudo docker run hello-world
```
Install docker compose:
``` Bash
sudo apt install docker-compose-plugin
```
Verify that Docker Compose is installed:
``` Bash
docker compose version
```
To use docker compose I would recommend creating a directory for each container you will create. For example I have my setup similar to down below for each container/stack on my setup.

```docker/nginx-proxy-manager```

Within each respective container folder you create the ```docker-compose.yml``` file and paste in the docker compose configuration.

## NGINX Proxy Manager

NGINX Proxy Manager let’s you manage domains and control which application each domain points to. For example, you can create a domain name for Pi-hole (e.g., pihole.website.io) instead of using its IP address. This setup won’t be public; only devices within your LAN will be able to access these services.

   Tutorial: Credit to "Wolfgang's Channel" on youtube for this [video](https://www.youtube.com/watch?v=qlcVx-k-02E) guide (I used deSEC for DNS, which is free.)

``` Bash
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

Below is how you run all the docker compose configuration files:
``` Bash
docker compose up -d
```
Access NGINX Proxy Manager at ```http://<raspberrypi-ip>:81```

## Adguard Home

Adguard acts as a [DNS sinkhole](https://en.wikipedia.org/wiki/DNS_sinkhole), blocking ads and telemetry requests.

``` Bash
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
After running the docker compose yml you should be able to reach adguard through ```http://<raspberrypi_ip>:3003``` for initial setup and then ```http://<raspberrypi_ip>:8062```.

Use DNS Rewrite to point domains towards reverse proxy for example *.lan -> 192.168.1.10

## Portainer
Portainer is a GUI tool for managing Docker containers.

##### Portainer Compose File
``` Bash
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

Access Portainer at ```https://<raspberrypi-ip>:9443```

## Bitwarden/Vaultwarden
Bitwarden is a password manager and vaultwarden is a more lightweight option that you can host yourself. This works with the bitwarden app and extension.

``` Bash
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

Once the container Is up you should be able to reach bitwarden through ```http://<raspberrypi-ip>:8080```, although you won't be able to create an account or use it just yet. Bitwarden needs to go through HTTPS otherwise errors will occur. There are multiple ways of doing this, one way is through a reverse proxy which I found to be the easiest, I use NGINX Proxy Manager for this.

## Wireguard VPN
VPN to reach my DNS and my LAN from outside my network. There are different options out there but I choose wireguard and found it simple to configure.

``` Bash
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
The server side VPN is created, for the client side run the command below to get a QR code of the configuration for the client.
``` Bash
docker exec -it wireguard /app/show-peer {peer number or name}
```
To add more clients in the future edit the peers variable in the docker-compose file and recreate the container.

## Watchtower
Automatically monitors and updates your running Docker containers to keep them up to date with the latest images.

``` Bash
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
      WATCHTOWER_HTTP_API_METRICS: "true"
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
    internal: true
```

## Filebrowser
Lightweight web-based file manager that lets you browse, upload, and manage files on your Pi through a simple UI.
```Bash
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

Should either have a default login or you can check logs to see if it generates a temporary password. ```docker logs -f filebrowser```

## Obsidian-LiveSync

Obsidian is a note-taking app. Obsidian LiveSync is a self-hosted synchronization plugin you can run on a Raspberry Pi, enabling real-time, end-to-end encrypted syncing of your notes across multiple devices without relying on third-party cloud services.

```Bash
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

Next you will have to setup the database, I would recommend following this [Guide](https://www.reddit.com/r/selfhosted/comments/1eo7knj/guide_obsidian_with_free_selfhosted_instant_sync/)

## Gluetun

A secure VPN client container that routes traffic from other containers (like torrent or indexer services) through a VPN tunnel for privacy and IP protection.

``` Bash
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
      - VPN_SERVICE_PROVIDER=private internet access
      - VPN_TYPE=openvpn
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

## arr stack

The *arr family is a collection of media automation tools that manage downloading, organizing, and tracking movies and TV shows through indexers and torrent clients.

For my arr stack I run everything within the same docker compose configuration file. Seems logical since they are mostly dependent on eachother. The initial setup of all of these is pretty simple but if you have trouble or things you would like to optimize I would recommend using this guide [Trash Guide](https://trash-guides.info/).

#### Docker Compose & Storage Layout
My Docker Compose stack is slightly unique because I mount my NAS directly to my Raspberry Pi. If you’re following this setup, you’ll need to adjust the volume paths to match your own external or mounted storage.

One important concept to understand is [hardlinks](https://en.wikipedia.org/wiki/Hard_link). 

#### Hardlinks & File Structure

In short hardlinks allow a single file to appear in multiple locations without consuming additional disk space.

For example your torrent client downloads a movie to data/torrents/radarr, radarr then “moves” the file to data/media. Instead of copying the file (which would double storage usage), radarr creates a hardlink. The operation is instant and both paths point to the same data blocks on disk.

#### File Structure

The main importance when making your own file structure is that all containers reference the same root path which is data in this case. This is required for hardlinking to function.

```
nas
├── downloads
│  ├── radarr
│  ├── tv-sonarr
├── tv-shows
└── movies
```
#### Verify Hardlinking

You can verify hardlinking by checking linkcount on the files, should be more than 1.

```ls -l```

If you want more confirmation you can compare the inode number on both files in downloads directory and in the finished plex diretory which in above case would be movies or tv-shows. Use command on both locations of the file and compare inode number. Same inode number = hardlinked.

```stat filename.mkv```

``` Bash
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

## cpu_temp.sh

First create a new server in Discord where you will get your alerts.

On the raspberry pi create a file, you can call it anything, I named it cpu_temp.sh. Paste what's below into it and place the discord webhook link at the correct variable in the script:

NOTE: Credit to Dave McKay for the initial script and tutorial on how to do it [LINK](https://www.howtogeek.com/discord-slack-alert-raspberry-pi-too-hot/)

``` Bash
#!/bin/bash

BASE_TEMP=50
STEP=5
STATE_FILE="/tmp/pi_temp_alert.state"

# get CPU temperature in Celsius (rounded down)
pi_temp=$(vcgencmd measure_temp | awk -F "[=.'']" '{print int($2)}')
this_pi=$(hostname)

discord_pi_webhook=""

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

To test it and make sure it's working change the 45 in the if statement to something lower like 20. Run the file ./cpu_temp.sh and you should get a notification in Discord. 

Now to automate this I used systemd timers and used this as a reference on what to do [LINK](https://www.howtogeek.com/replace-cron-jobs-with-systemd-timers/).

Create a pialert.service and pialert.timer file at /etc/systemd/system

cpu-temp.service

``` Bash
[Unit]
Description=CPU Temperature Monitor

[Service]
Type=oneshot
ExecStart=/home/admin/scripts/cpu_temp.sh
```

cpu-temp.timer

``` Bash
[Unit]
Description=Run CPU temp monitor every 5 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=10min
Persistent=true

[Install]
WantedBy=timers.target
```

```sudo systemctl enable --now cpu-temp.timer```

## backup-obsidian.sh

backup-obsidian.sh

``` Bash
#!/bin/bash

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

```sudo nano /etc/systemd/system/obsidian-backup.service```

``` Bash
[Unit]
Description=Obsidian LiveSync Backup

[Service]
Type=oneshot
ExecStart=/home/admin/scripts/backup-obsidian.sh
```

```sudo nano /etc/systemd/system/obsidian-backup.timer```

``` Bash
[Unit]
Description=Nightly Obsidian Backup

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```sudo systemctl enable --now obsidian-backup.timer```

## Docker Compose Files Backup Script

This part requires a little more work. You'll have to use git.

Install
```apt-get install git```

Generate SSH Key
```ssh-keygen -t ed25519 -C "rpi-backup"```

Show SSH Key
"```cat ~/.ssh/id_ed25519.pub"```

Add it to GitHub, [Here](https://github.com/settings/keys) 

Test It
```ssh -T git@github.com"```

Here it depends if you already have an existing github repo or wants to create one. I already have one so my flow looks like this:

``` Bash
cd ~/containers
git init
git remote add origin git@github.com:github-username/github-repo.git
git add -A
git commit -m "New Raspberry Pi rebuild"
git branch -M main
git push -u origin main --force"
```
Also add a .gitignore file in the containers dir

``` Bash
# Ignore everything by default
*

# Allow folders
!*/

# Allow docker compose files
!*/docker-compose.yml
!*/docker-compose.yaml

# Ignore all env files
*.env
# Ignore all txt files
*.txt
# Ignore Docker volumes / letsencrypt / DATA_DIR folders
*/letsencrypt/*
*/DATA_DIR/*
*/data
*/work
# Ignore raw YAML configs (we will push only sanitized examples)
*.yaml

# Allow root files
!.gitignore
```

## docker-backup.sh

``` Bash
#!/bin/bash
cd ~/containers
git add -A
git diff --cached --quiet && echo "No changes, skipping commit." && exit 0
git commit -m "Auto backup $(date '+%Y-%m-%d %H:%M')"
git push origin main
### Systemd Timers
```

```sudo nano /etc/systemd/system/docker-backup.service```

``` Bash
[Unit]
Description=Git backup of Docker configs

[Service]
Type=oneshot
WorkingDirectory=/home/admin/containers
ExecStart=/home/admin/scripts/docker-backup.sh

sudo nano /etc/systemd/system/docker-backup.timer

[Unit]
Description=Hourly Docker Config Backup

[Timer]
OnCalendar=Sun *-*-* 03:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

```sudo systemctl enable --now docker-backup.timer```


## Random

/etc/fstab

NASIP:/volume1/NAS  /mnt/nas  nfs  defaults,_netdev,x-systemd.automount  0  0


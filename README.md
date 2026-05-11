# <img src="https://github.com/user-attachments/assets/39d7950d-8c68-4845-a20c-97ba42d940cd" width="60" alt="logo">Raspberry Pi Home Server with Docker

![Docker](https://img.shields.io/badge/Docker-Ready-blue?logo=docker)
![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-4B-red?logo=raspberrypi)
![Maintained](https://img.shields.io/badge/Maintained-Yes-brightgreen)
![License](https://img.shields.io/badge/License-MIT-green)

This repository documents my raspberry pi server setup, covering configuration steps, docker compose files, and other useful resources. 

## 🧩 Services Overview

List of all services included in this Raspberry Pi setup, along with short descriptions and repository links for reference.

| Service | Description | Repository |
|----------|--------------|-------------|
| **[Nginx Proxy Manager](#nginx-proxy-manager)** | Manages reverse proxy configurations and SSL certificates through a simple web interface, allowing you to route domains to local services. | [Repository ↗︎](https://github.com/Nginxproxymanager/Nginx-Proxy-Manager) |
| **[Portainer](#portainer)** | Web-based GUI for managing Docker containers, images, networks, and volumes. | [Repository ↗︎](https://github.com/Portainer/Portainer) |
| **[Pi-hole](#pi-hole--cloudflared)** | Network-wide ad blocker acting as a DNS sinkhole to block unwanted domains. | [Repository ↗︎](https://github.com/Pi-Hole/Pi-Hole) |
| ↳ **[Cloudflared](#pi-hole--cloudflared)** | Provides DNS over HTTPS (DoH) for encrypted DNS queries. | [Repository ↗︎](https://github.com/Cloudflare/Cloudflared) |
| **[Vaultwarden](#bitwardenvaultwarden)** | Lightweight self-hosted Bitwarden-compatible password manager. | [Repository ↗︎](https://github.com/Dani-Garcia/Vaultwarden) |
| **[WireGuard](#wireguard-vpn)** | Fast, secure VPN solution for remote access to your network and DNS. | [Repository ↗︎](https://www.wireguard.com/) |
| **[Watchtower](#watchtower)** | Automatically monitors and updates running Docker containers with the latest images. | [Repository ↗︎](https://github.com/Containrrr/Watchtower) |
| **[Filebrowser](#filebrowser)** | Lightweight web-based file manager to browse, upload, and organize files. | [Repository ↗︎](https://github.com/hurlenko/filebrowser-docker) |
| **[Obsidian LiveSync](#obsidian-livesync)** | Self-hosted sync service for Obsidian, enabling encrypted note synchronization across devices. | [Repository ↗︎](https://github.com/vrtmrz/obsidian-livesync) |
| **[Grafana / Pi Monitoring](#grafana)** | Visualizes system and Docker metrics via Prometheus and Grafana dashboards. | [Repository ↗︎](https://github.com/oijkn/Docker-Raspberry-PI-Monitoring) |
| **[Gluetun](#gluetun)** | VPN client container that routes traffic from other containers securely through supported VPN providers. | [Repository ↗︎](https://github.com/qdm12/gluetun) |
| **[*arr Stack](#arr-stack)** | Suite of media automation tools for managing movies and TV shows. | — |
| ↳ **[Overseerr](#arr-stack)** | Media request management interface for Radarr and Sonarr. | [Repository ↗︎](https://github.com/sct/overseerr) |
| ↳ **[Radarr](#arr-stack)** | Automatically downloads and organizes movies. | [Repository ↗︎](https://github.com/Radarr/Radarr) |
| ↳ **[Sonarr](#arr-stack)** | Automatically downloads and organizes TV shows. | [Repository ↗︎](https://github.com/Sonarr/Sonarr) |
| ↳ **[Prowlarr](#arr-stack)** | Indexer manager and proxy for *arr apps. | [Repository ↗︎](https://github.com/Prowlarr/Prowlarr) |
| ↳ **[Flaresolverr](#arr-stack)** | Handles Cloudflare protection for indexers that require JavaScript solving. | [Repository ↗︎](https://github.com/FlareSolverr/FlareSolverr) |
| ↳ **[qBittorrent](#arr-stack)** | Torrent client used for downloading media, typically routed through Gluetun VPN. | [Repository ↗︎](https://github.com/linuxserver/docker-qbittorrent) |
| ↳ **[Maintainerr](#arr-stack)** | Janitor for your stack. | [Repository ↗︎](https://github.com/Maintainerr/Maintainerr) |
| **[Pi Temperature Alert](#receive-discord-alerts-for-raspberry-pi-overheating)** | Receive Discord Temperature Alert. | — |




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
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx_proxy_manager
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./config.json:/app/config/production.json
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    restart: unless-stopped
    networks:
      proxy:
        ipv4_address: 172.20.0.2

networks:
  proxy:
    name: proxy
    ipam:
      config:
        - subnet: 172.20.0.0/16
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
    container_name: portainer
    image: portainer/portainer-ce:latest
    ports:
      - 9443:9443
    volumes:
      - data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      portainer:
        ipv4_address: 172.30.0.2
      proxy:
        ipv4_address: 172.20.0.3
    restart: unless-stopped

networks:
  portainer:
    name: portainer
    ipam:
      config:
        - subnet: 172.30.0.0/16
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
    container_name: vaultwarden
    environment:
      - WEBSOCKET_ENABLED=true
    volumes:
      - ./data:/data
    ports:
      - 8080:80
    restart: unless-stopped
    networks:
      bitwarden:
        ipv4_address: 172.70.0.2
      proxy:
        ipv4_address: 172.20.0.5

networks:
  bitwarden:
    name: bitwarden
    ipam:
      config:
        - subnet: 172.70.0.0/16
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
      - PEERS=3                                #Amount of available peers
      - PEERDNS=172.20.0.4                     #IP of pihole container
      - INTERNAL_SUBNET=10.13.13.0
      - ALLOWEDIPS=0.0.0.0/0                   #Allows all IPs to connect to VPN
    volumes:
      - /home/pi/wireguard/config:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv4.ip_forward=1
    restart: unless-stopped
    networks:
      wireguard:
        ipv4_address: 172.40.0.2
      proxy:
        ipv4_address: 172.20.0.7

networks:
  wireguard:
    name: wireguard
    ipam:
      config:
        - subnet: 172.40.0.0/16
  proxy:
     external: true
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
    image: containrrr/watchtower:latest
    container_name: watchtower
    environment:
      TZ: Europe/Stockholm
      #WATCHTOWER_ROLLING_RESTART: 'true'
      #WATCHTOWER_MONITOR_ONLY: 'true'
      WATCHTOWER_SCHEDULE: '0 0 0 * * 0'           #Cron expression (Set at 12:00 AM, only on Sunday)
      WATCHTOWER_CLEANUP: 'true'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    networks:
      watchtower:
        ipv4_address: 172.120.0.2

networks:
  watchtower:
    name: watchtower
    ipam:
      config:
        - subnet: 172.120.0.0/16
```

## Filebrowser
Lightweight web-based file manager that lets you browse, upload, and manage files on your Pi through a simple UI.
```Bash
services:
  filebrowser:
    container_name: filebrowser
    image: hurlenko/filebrowser
    user: "${UID}:${GID}"
    ports:
      - 444:8080
    volumes:
      - ./DATA_DIR:/data
      - ./CONFIG_DIR:/config
    environment:
      - FB_BASEURL=/filebrowser
    networks:
      filebrowser:
        ipv4_address: 172.110.0.2
      proxy:
        ipv4_address: 172.20.0.6
    restart: unless-stopped

networks:
  filebrowser:
    name: filebrowser
    ipam:
      config:
        - subnet: 172.110.0.0/16
  proxy:
     external: true
```

Should either have a default login or you can check logs to see if it generates a temporary password. ```docker logs -f filebrowser```

## Obsidian-LiveSync

Obsidian is a note-taking app. Obsidian LiveSync is a self-hosted synchronization plugin you can run on a Raspberry Pi, enabling real-time, end-to-end encrypted syncing of your notes across multiple devices without relying on third-party cloud services.

```Bash
services:
   couchdb-obsidian-livesync:
    container_name: obsidian-livesync
    image: couchdb:latest
    secrets:
      - couchdb_password
    environment:
      - TZ=Europe/Stockholm
      - COUCHDB_USER=admin
      - COUCHDB_PASSWORD_FILE=/run/secrets/couchdb_password
    volumes:
      - ./data:/opt/couchdb/data
      - ./etc:/opt/couchdb/etc/local.d
    ports:
      - "5984:5984"
    networks:
      obsidian:
        ipv4_address: 172.60.0.2
    restart: unless-stopped
  
networks:
  obsidian:
    name: obsidian
    ipam:
      config:
        - subnet: 172.60.0.0/16
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
    secrets:
      - vpn_user
      - vpn_password
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    volumes:
      - ./data:/gluetun
    environment:
      - TZ=Europe/Stockholm
      - VPN_SERVICE_PROVIDER=                                     #VPN Provider, check Gluetun documentation
      - VPN_TYPE=openvpn
      - OPENVPN_USER_FILE=/run/secrets/vpn_user
      - OPENVPN_PASSWORD_FILE=/run/secrets/vpn_password
      - SERVER_REGIONS=                                           #Which region/country you want VPN to connect
      - FIREWALL_OUTBOUND_SUBNETS=172.100.0.0/24,192.168.1.0/24   #Needed to allow one of these for *arr stack to work, either container LAN or actual LAN.
    ports:
      - 8081:8081                                                 #For qbittorrent
      - 6881:6881                                                 #For qbittorrent
      - 6881:6881/udp
      - 9696:9696                                                 #For Prowlarr
      - 8191:8191                                                 #For verr
    restart: unless-stopped
    networks:
      gluetun:
        ipv4_address: 172.90.0.2

networks:
  gluetun:
    driver: bridge
    name: gluetun
    ipam:
      config:
        - subnet: 172.90.0.0/16
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
data
├── Downloads
│  ├── radarr
│  ├── tv-sonarr
├── Temporary TV Shows & Anime
└── Temporary Movies
```
#### Verify Hardlinking

You can verify hardlinking by checking linkcount

```ls -l```

If you want more confirmation you can compare the inode number when writing below command on both locations of the file, same inode number = hardlinked.

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
      - /mnt/BigBoi/data:/data
    ports:
      - 7878:7878
    restart: unless-stopped
    networks:
      arr:
        ipv4_address: 172.100.0.2

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
      - /mnt/BigBoi/data:/data
    ports:
      - 8989:8989
    restart: unless-stopped
    networks:
      arr:
        ipv4_address: 172.100.0.3
    
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
      - /mnt/BigBoi/data/Downloads:/data/Downloads
    healthcheck:
      test: ping -c 2 8.8.8.8 || exit 1
      interval: 60s
      timeout: 5s
      retries: 2
      start_period: 60s
    restart: on-failure
    network_mode: "container:gluetun"
    
  overseerr:
    image: lscr.io/linuxserver/overseerr:latest
    container_name: overseerr
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=022
      - TZ=Europe/Stockholm
    volumes:
      - overseerr_config:/config
    ports:
      - 5055:5055
    restart: unless-stopped
    networks:
      arr:
        ipv4_address: 172.100.0.4

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
    healthcheck:
      test: ping -c 2 8.8.8.8 || exit 1
      interval: 60s
      timeout: 50s
      retries: 2
      start_period: 60s
    restart: on-failure
    network_mode: "container:gluetun"
      
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
    healthcheck:
      test: ["CMD-SHELL", "curl -s --head --fail http://www.google.com || exit 1"]
      interval: 60s
      timeout: 5s
      retries: 2
      start_period: 60s
    restart: on-failure
    network_mode: "container:gluetun"

  maintainerr:
    image: ghcr.io/maintainerr/maintainerr:latest
    container_name: maintainerr
    volumes:
      - type: bind
        source: ./data
        target: /opt/data
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=022
      - TZ=Europe/Stockholm
    ports:
      - 6246:6246
    restart: unless-stopped
    networks:
      arr:
        ipv4_address: 172.100.0.5

networks:
  arr:
    driver: bridge
    name: arr
    ipam:
      config:
        - subnet: 172.100.0.0/16

volumes:
  radarr_config:
  sonarr_config:
  overseerr_config:
  prowlarr_config:
  flaresolverr_config:
  qbittorrent_config:
```

## Receive Discord Alerts for Raspberry Pi Overheating

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
    -d "{\"content\":\" ALERT! ${this_pi} CPU temp reached ${pi_temp}°C\"}" \
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

pialert.service

``` Bash
Description="Runs pi alert script"
Requires=cpu_temp.sh

[Service]
Type=simple
ExecStart=/home/admin/pi-alert/cpu_temp.sh
User=admin                                  #change this to your user
```

pialert.timer

``` Bash
[Unit]
Description="Timer for the pialert.service"

[Timer]
Unit=pialert.service
OnBootSec=5min
OnUnitActiveSec=10min                       #how often it should run

[Install]
WantedBy=timers.target
```


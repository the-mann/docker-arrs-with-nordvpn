# docker-arrs-with-nordvpn
This is a Docker Stack to deploy Sonarr, Radarr, Jackett, and qBittorrent. With Jacket and qBittorrent connecting to NordVPN.

All apps connect to a shared volume that connects to a NAS SMB share using CIFS.

## Sources/Referances
- ~~NordVPN: https://github.com/bubuntux/nordvpn~~
- NordLynx: https://github.com/bubuntux/nordlynx
  - https://github.com/bubuntux/nordlynx/issues/39
  - https://github.com/bubuntux/nordlynx/discussions/28
  - https://forum.openwrt.org/t/instruction-config-nordvpn-wireguard-nordlynx-on-openwrt/89976
  - https://sleeplessbeastie.eu/2019/02/18/how-to-use-public-nordvpn-api/
- qBittorrent: https://hub.docker.com/r/linuxserver/qbittorrent
- ~~Jackett: https://hub.docker.com/r/linuxserver/jackett/~~
- Prowlarr: https://wiki.servarr.com/prowlarr
- Sonarr: https://hub.docker.com/r/linuxserver/sonarr
- Radarr: https://hub.docker.com/r/linuxserver/radarr
- Servarr Wiki: https://wiki.servarr.com/

##Docker Environment Variables:
| Variable | Notes |
| ----------- | ----------- |
| nordlynx_pkey | Generated via NordVPN Container |
| nasUser | The service username on NAS host |
| nasPass | The service password on NAS host |
| LAN | Whitelist your local area network so the apps connect via NordVPN can be accessed. Example: 192.168.1.0/24 |
| smbSharePath | Network path to your SMB share Example: //192.168.1.18/Media |

## Docker Compose File
```yaml:docker-compose.yml
version: "3.4"
networks:
  sith-network:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: "172.30.172.0/24"
    enable_ipv6: false
services:
  nordlynx:
    image: ghcr.io/bubuntux/nordlynx
    container_name: nordlynx
    cap_add:
      - NET_ADMIN
    environment:
      - PRIVATE_KEY=$nordlynx_pkey #required
      - QUERY=filters\[country_id\]=228&filters\[servers_groups\]\[identifier\]=legacy_p2p #https://api.nordvpn.com/v1/servers/recommendations?filters\[country_id\]=228&filters\[servers_groups\]\[identifier\]=legacy_p2p
      - NET_LOCAL=192.168.1.0/24
      - DNS=1.1.1.1,1.0.0.1,8.8.8.8
      - RECONNECT=14400
      - TZ=$timezone
    ports:
      - 9080:9080 #qbittorrent
      # - 9117:9117 #jackett
      - 9696:9696 #prowlarr
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=1
    networks:
      - sith-network
    restart: unless-stopped
  qbittorrent:
    image: ghcr.io/linuxserver/qbittorrent
    container_name: qbittorrent
    cap_add:
      - NET_ADMIN
    network_mode: service:nordlynx
    depends_on:
      nordlynx:
        condition: service_healthy
    environment:
      - PUID=1000
      - PGID=997
      - TZ=$timezone
      - WEBUI_PORT=9080
    volumes:
      - qbit_config:/config
      - media:/media
    restart: unless-stopped
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:develop
    container_name: prowlarr
    cap_add:
      - NET_ADMIN
    network_mode: service:nordlynx
    depends_on:
      nordlynx:
        condition: service_healthy
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=$timezone
    volumes:
      - prowlarr_config:/config
    restart: unless-stopped
  sonarr:
    image: ghcr.io/linuxserver/sonarr
    container_name: sonarr
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=1000
      - PGID=997
      - TZ=$timezone
    volumes:
      - sonarr_config:/config
      - media:/media 
    ports:
      - 8989:8989
    networks:
      - sith-network
    restart: unless-stopped
  radarr:
    image: ghcr.io/linuxserver/radarr
    container_name: radarr
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=1000
      - PGID=997
      - TZ=$timezone
    volumes:
      - radarr_config:/config
      - media:/media 
    ports:
      - 7878:7878
    networks:
      - sith-network
    restart: unless-stopped
  readarr:
    image: lscr.io/linuxserver/readarr:nightly
    container_name: readarr
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=$timezone
    volumes:
      - readarr_config:/config
      - media:/media
    ports:
      - 8787:8787
    networks:
      - sith-network
    restart: unless-stopped
volumes:
  qbit_config:
  prowlarr_config:
  sonarr_config:
  radarr_config:
  readarr_config:
  media:
    driver_opts:
      type: cifs
      o: username=$nasUser,password=$nasPass,file_mode=0777,dir_mode=0777,noperm
      device: $nasMediaPath
```

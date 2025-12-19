# Homelab Overview 

- Container subnet: `10.10.50.0/24` (`lab_net`, external Docker network)
- Host IPs: `192.168.0.10` (LAN), `10.10.50.1/24` (gateway for lab_net)
- Router: static route `10.10.50.0/24 via 192.168.0.10`
- DNS chain: NPM (10.10.50.12) → AdGuard (10.10.50.13) → Internet (1.1.1.1/1.0.0.1)
- Project name: `homelab`
- Storage mounts in compose:
  - Media/download root: `/warehouse`
  - NVMe (general fast storage): `/mnt/nvme`
  - Transcode (RAM): `/mnt/ramtranscode`
- GPU: Intel iGPU via `/dev/dri` (Jellyfin, Tdarr node)
- VPN namespace: Gluetun (Mullvad WireGuard) shared by all “Arr”/download apps

---

## Network Topology
- Host NIC `enp3s0`: `192.168.0.10` (LAN), `10.10.50.1/24` (lab_net gateway)
- Docker external network `lab_net` (ipvlan L3): `10.10.50.0/24`, gw `10.10.50.1`
- Tailscale: host network (no lab_net IP)
- Service discovery: Docker DNS on `lab_net`; app → DB by hostname.

---

## Core / Ingress / DNS
**Portainer**  
- IP: 10.10.50.10  
- Port: 9443/tcp (published)  
- Volumes: `/var/run/docker.sock:/var/run/docker.sock`  
- Role: Docker UI  
- Notes: Runs standalone stack.

**NPM (Nginx Proxy Manager)**  
- IP: 10.10.50.12  
- Ports: 80/443/81 (published)  
- Volumes: `./npm-data:/data`, `./npm-letsencrypt:/etc/letsencrypt`  
- Role: Reverse proxy/SSL  
- Notes: DNS points to AdGuard.

**AdGuard Home**  
- IP: 10.10.50.13  
- Ports: 53/tcp+udp (published), 3000/tcp (setup)  
- Volumes: `./adguard-work`, `./adguard-conf`  
- Role: DNS filtering/resolver  
- Notes: Upstream 1.1.1.1/1.0.0.1.

**Tailscale**  
- Network: host  
- Volumes: `./tailscale-varlib`, `/dev/net/tun`  
- Role: Remote access/exit-node  
- Notes: No lab_net IP.

---

## VPN-Bound Stack (via Gluetun)
**Gluetun**  
- IP: 10.10.50.14  
- Ports (published on host): 8080, 5050, 9696, 7878, 8989, 6969, 6767, 8787, 5055, 8191, 64898  
- Role: VPN namespace (Mullvad WG)  
- Notes: Healthcheck; others use `network_mode: container:gluetun`.

**qBittorrent**  
- IP: shares Gluetun  
- Port: 8080 (via Gluetun)  
- Volumes: `./qbittorrent-config:/config`, `/warehouse:/warehouse`  
- Role: BitTorrent client.

**SABnzbd**  
- IP: shares Gluetun  
- Port: 5050 (via Gluetun)  
- Volumes: `./sabnzbd-config:/config`, `/warehouse:/warehouse`  
- Role: Usenet client.

**Radarr**  
- IP: shares Gluetun  
- Port: 7878 (via Gluetun)  
- Volumes: `./radarr-config:/config`, `/warehouse:/warehouse`  
- Role: Movies automation.

**Sonarr**  
- IP: shares Gluetun  
- Port: 8989 (via Gluetun)  
- Volumes: `./sonarr-config:/config`, `/warehouse:/warehouse`  
- Role: TV automation.

**Whisparr**  
- IP: shares Gluetun  
- Port: 6969 (via Gluetun)  
- Volumes: `./whisparr-config:/config`, `/warehouse:/warehouse`  
- Role: Adult/audiobooks automation.

**Bazarr**  
- IP: shares Gluetun  
- Port: 6767 (via Gluetun)  
- Volumes: `./bazarr-config:/config`, `/warehouse:/warehouse`  
- Role: Subtitles.

**Recyclarr**  
- IP: shares Gluetun  
- Port: 8787 (via Gluetun)  
- Volumes: `./recyclarr-config:/config`, `/warehouse:/warehouse`  
- Role: Sync quality profiles.

**Prowlarr**  
- IP: shares Gluetun  
- Port: 9696 (via Gluetun)  
- Volumes: `./prowlarr-config:/config`  
- Role: Indexer manager.

**Jellyseerr**  
- IP: shares Gluetun  
- Port: 5055 (via Gluetun)  
- Volumes: `./jellyseerr-config:/app/config`  
- Role: Request front-end.

**FlareSolverr**  
- IP: shares Gluetun  
- Port: 8191 (via Gluetun)  
- Role: Anti-cloudflare helper.

---

## Media Serve
**Jellyfin**  
- IP: 10.10.50.30  
- Port: 8096 (published)  
- Volumes: `./jellyfin-config:/config`, `/warehouse:/warehouse`, `/mnt/ramtranscode:/mnt/ramtranscode`, `/mnt/nvme:/mnt/nvme`  
- Role: Media server  
- Notes: VAAPI via `/dev/dri`; transcode in RAM disk.

**Jellystat**  
- IP: 10.10.50.32  
- Port: 5056 (published)  
- Volumes: `./jellystat-config:/config`  
- Role: Jellyfin analytics.

**Jellystat DB (Postgres)**  
- IP: 10.10.50.132  
- Port: 5432 (internal)  
- Volumes: `./jellystat-db-data:/var/lib/postgresql/data`  
- Role: DB for Jellystat.

**Tdarr Server**  
- IP: 10.10.50.33  
- Ports: 8265/8266 (published)  
- Volumes: `./tdarr-server:/app/server`, `./tdarr-config:/app/configs`, `./tdarr-logs:/app/logs`, `/warehouse:/warehouse`  
- Role: Transcode orchestrator.

**Tdarr Node**  
- IP: 10.10.50.34  
- Ports: none published  
- Volumes: `./tdarr-node-config:/app/configs`, `./tdarr-node-logs:/app/logs`, `/warehouse:/warehouse`  
- Role: Worker (uses `/dev/dri`).

**Kometa**  
- IP: 10.10.50.35  
- Port: none (runs on schedule/CLI)  
- Volumes: `./kometa-config:/config`, `/warehouse:/warehouse`  
- Role: Library metadata/collections automation.

---

## Apps Core
**Homepage**  
- IP: 10.10.50.40  
- Port: internal (expose if desired)  
- Volumes: `./homepage-config:/app/config`, `/var/run/docker.sock:/var/run/docker.sock:ro`  
- Role: Dashboard.

**Glances**  
- IP: 10.10.50.41  
- Port: internal (61208 if exposed)  
- Volumes: `/var/run/docker.sock:ro`, `/etc/os-release:ro`, `/proc:/host/proc:ro`, `/sys:/host/sys:ro`  
- Role: Monitoring.

**Bookstack**  
- IP: 10.10.50.42  
- Port: 6875 (published)  
- Volumes: `./bookstack-config:/config`  
- Role: Wiki/docs.

**Bookstack DB (MariaDB)**  
- IP: 10.10.50.142  
- Port: 3306 (internal)  
- Volumes: `./bookstack-db-data:/var/lib/mysql`  
- Role: DB for Bookstack.

**Authentik**  
- IP: 10.10.50.43  
- Port: 9000 (published)  
- Volumes: `./authentik-media:/media`, `./authentik-custom-templates:/templates`  
- Role: SSO/IdP.

**Authentik DB (Postgres)**  
- IP: 10.10.50.143  
- Port: 5432 (internal)  
- Volumes: `./authentik-db-data:/var/lib/postgresql/data`  
- Role: DB for Authentik.

**Authentik Redis**  
- IP: 10.10.50.144  
- Port: 6379 (internal)  
- Volumes: `./authentik-redis-data:/data`  
- Role: Cache/queue for Authentik.

---

## Home Automation
**Home Assistant**  
- IP: 10.10.50.50  
- Port: 8123 (published)  
- Volumes: `./homeassistant-config:/config`, `/etc/localtime:/etc/localtime:ro`  
- Role: Home automation  
- Notes: On its own IP; discovery may be limited vs host network.

---

## Games
**Pterodactyl Panel**  
- IP: 10.10.50.61  
- Port: 8088 (published)  
- Volumes: `./pterodactyl-config:/app`, `/warehouse:/warehouse`, `/mnt/nvme:/mnt/nvme`  
- Role: Game panel.

**Wings**  
- IP: 10.10.50.62  
- Port: 2022 (published) + add game ports as needed  
- Volumes: `/var/lib/pterodactyl:/var/lib/pterodactyl`, `/var/run/docker.sock:/var/run/docker.sock`, `/warehouse:/warehouse`, `/mnt/nvme:/mnt/nvme`  
- Role: Game daemon.

---

## Special Paths (1:1)
- General media/download root: `/warehouse`
- Fast storage (NVMe): `/mnt/nvme`
- Transcode: `/mnt/ramtranscode`
- Tdarr media root: `/warehouse`

---

## Exposure / Ingress Summary
- Public-facing (published): 80/81/443 (NPM), 9443 (Portainer), 53 TCP/UDP (AdGuard), Gluetun ports (Arr stack) if allowed, 8096 (Jellyfin), 5056 (Jellystat), 8265/8266 (Tdarr), 6875 (Bookstack), 9000 (Authentik), 8123 (HA), 8088 (Pterodactyl), 2022 (+ game ports) for Wings.
- Restrict Gluetun ports and other services at the firewall if you do not need external reachability.
- Reverse proxy via NPM for HTTP(S) services on lab_net.

---

## Notes / Best Practices
- Keep Gluetun ports minimal; only expose what you need.
- Use `dns: ["10.10.50.13"]` on services if you want them filtered by AdGuard.
- Strong secrets for all DBs, Authentik, and WireGuard keys.
- Consider docker-socket-proxy for Portainer/Homepage/Glances if avoiding raw docker.sock exposure.
- Back up all `*-config` and DB volumes (Bookstack DB, Authentik DB, Jellystat DB).
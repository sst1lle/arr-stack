# 🎬 Arr-Stack Media Server

Een volledig geautomatiseerde media server stack draaiend op een **Raspberry Pi 5 (8GB)** met NVMe SSD, gebouwd met Docker Compose.

## 📦 Stack overzicht

| Service | Functie | Poort |
|---|---|---|
| **Gluetun** | VPN container (Mullvad WireGuard) | — |
| **qBittorrent** | Download client | `8080` |
| **Prowlarr** | Indexer manager | `9696` |
| **FlareSolverr** | Cloudflare bypass proxy | `8191` |
| **Radarr** | Film automatisering | `7878` |
| **Sonarr** | Serie automatisering | `8989` |
| **Jellyseerr** | Aanvraag interface | `5055` |
| **Jellyfin** | Media server | `8096` |

## 🖥️ Hardware

- **Raspberry Pi 5** — 8GB RAM
- **NVMe SSD** — 235GB (OS + Docker images + config)
- **Externe SSD** — 1.9TB via `/mnt/ssd` (media opslag)

## 🏗️ Architectuur & beslissingen

### Waarom Gluetun?
In Nederland zijn torrent sites geblokkeerd door ISP's. In plaats van een systeem-brede VPN is gekozen voor **Gluetun** als VPN container. Alleen de containers die door Gluetun's network namespace gaan (qBittorrent, Prowlarr, FlareSolverr) gebruiken de VPN. Radarr, Sonarr, Jellyseerr en Jellyfin draaien op het normale netwerk — dit heet een **split tunnel**.

```
Internet
    │
    ├── Gluetun (Mullvad WireGuard)
    │       ├── qBittorrent   ← torrent verkeer via VPN
    │       ├── Prowlarr      ← indexer zoekopdrachten via VPN
    │       └── FlareSolverr  ← Cloudflare bypass via VPN
    │
    └── Normaal netwerk
            ├── Radarr
            ├── Sonarr
            ├── Jellyseerr
            └── Jellyfin
```

### Waarom FlareSolverr?
Veel torrent indexers zoals 1337x zijn beschermd door Cloudflare DDoS protection. FlareSolverr draait een headless Chrome browser die de Cloudflare challenge oplost. Prowlarr stuurt requests via FlareSolverr voor sites die dit nodig hebben via een tag-systeem.

**Belangrijk:** FlareSolverr moet ook via de VPN lopen, anders kan Cloudflare je thuis-IP zien en blijft de blokkade actief.

### Waarom twee FlareSolverr instanties?
De originele FlareSolverr draait al voor het Huursignal project in een apart Docker netwerk. Omdat containers in verschillende netwerken elkaar niet kunnen bereiken, draait er een tweede instantie (`flaresolverr-arr`) die via Gluetun gaat specifiek voor de arr-stack.

### Network namespace probleem
qBittorrent en Prowlarr draaien via `network_mode: service:gluetun` — ze hebben geen eigen netwerk maar delen Gluetun's netwerk. Dit betekent:
- Ze zijn **niet bereikbaar via containernaam** (`http://qbittorrent:8080` werkt niet)
- Ze zijn bereikbaar via het **Gluetun IP** (`172.19.0.5`) of het **Pi IP** (`192.168.200.217`)
- Radarr en Sonarr gebruiken `http://192.168.200.217:8080` om qBittorrent te bereiken
- FlareSolverr is bereikbaar via `http://localhost:8191` vanuit Prowlarr (zelfde namespace)

### DHT binding
qBittorrent moet expliciet gebonden worden aan de VPN interface (`10.70.22.191`) via **Tools → Options → Advanced → Optional IP address**. Zonder dit bindt qBittorrent aan het Docker netwerk IP en werkt DHT niet — waardoor torrents niet kunnen verbinden met peers.

## 🗂️ Mappenstructuur

```
/mnt/ssd/
├── media/
│   ├── movies/        ← Radarr plaatst films hier
│   ├── series/        ← Sonarr plaatst series hier
│   └── music/
├── downloads/
│   ├── complete/      ← Afgeronde downloads
│   └── incomplete/    ← Bezig met downloaden
├── config/
│   ├── radarr/
│   ├── sonarr/
│   ├── prowlarr/
│   ├── jellyseerr/
│   └── qbittorrent/
└── jellyfin/
    ├── config/
    └── cache/
```

## 🚀 Installatie

### 1. Vereisten
- Docker + Docker Compose
- Mullvad account met WireGuard config
- Cloudflare account met domein (voor publieke toegang)
- Tailscale (voor remote toegang)

### 2. Environment variabelen instellen

Maak een `.env` bestand aan in de `arr-stack` map:

```env
MULLVAD_PRIVATE_KEY=jouw-private-key-hier
MULLVAD_ADDRESS=10.x.x.x/32
```

### 3. Mappen aanmaken

```bash
sudo mkdir -p /mnt/ssd/{media/{movies,series,music},downloads/{complete,incomplete},config/{radarr,sonarr,prowlarr,jellyseerr,qbittorrent}}
sudo chown -R $USER:$USER /mnt/ssd
```

### 4. Stack starten

```bash
cd arr-stack
docker compose up -d
```

### 5. Configuratie volgorde

1. **qBittorrent** (`http://<pi-ip>:8080`)
   - Downloads → save path: `/downloads/complete`, incomplete: `/downloads/incomplete`
   - Advanced → Optional IP address: `10.70.22.191` (jouw Mullvad VPN IP)
   - Categorieën aanmaken: `radarr` en `sonarr`

2. **Prowlarr** (`http://<pi-ip>:9696`)
   - Indexer Proxies → Add → FlareSolverr → host: `http://localhost:8191`
   - Indexers toevoegen: YTS, EZTV, 1337x (met FlareSolverr tag)
   - Apps → Add Radarr + Sonarr → Sync

3. **Radarr** (`http://<pi-ip>:7878`)
   - Media Management → Root folder: `/movies`, hardlinks aan
   - Download Clients → qBittorrent → host: `<pi-ip>`, port: `8080`

4. **Sonarr** (`http://<pi-ip>:8989`)
   - Media Management → Root folder: `/tv`, hardlinks aan
   - Download Clients → qBittorrent → host: `<pi-ip>`, port: `8080`

5. **Jellyseerr** (`http://<pi-ip>:5055`)
   - Koppel Jellyfin, Radarr en Sonarr

## 🌐 Toegang

### Lokaal netwerk
Alle services bereikbaar via `http://192.168.200.217:<poort>`

### Via Tailscale (remote)
Installeer Tailscale op je apparaat en verbind met hetzelfde Tailscale netwerk als de Pi. Dan bereik je alle services via het Tailscale IP van de Pi:

```bash
# Tailscale IP opvragen
tailscale ip
```

Gebruik dan `http://<tailscale-ip>:<poort>` voor alle services.

### Via Cloudflare Tunnel (publiek)
Jellyfin is publiek bereikbaar via `https://jellyfin.huursignal.com` dankzij Cloudflare Tunnel.

Config staat in `/etc/cloudflared/config.yml`:

```yaml
tunnel: huursignal
credentials-file: /home/stille/.cloudflared/<tunnel-id>.json
ingress:
  - hostname: huursignal.com
    service: http://localhost:5000
  - hostname: jellyfin.huursignal.com
    service: http://192.168.200.217:8096
  - service: http_status:404
```

Tunnel beheren:
```bash
sudo systemctl restart cloudflared
sudo systemctl status cloudflared
```

## 🔄 Automatische flow

```
Jellyseerr → Radarr/Sonarr → Prowlarr → qBittorrent → /mnt/ssd/media → Jellyfin
```

1. Gebruiker vraagt film aan via Jellyseerr
2. Jellyseerr stuurt aanvraag naar Radarr
3. Radarr vraagt Prowlarr om te zoeken
4. Prowlarr doorzoekt alle indexers (via VPN, via FlareSolverr indien nodig)
5. qBittorrent downloadt de torrent (via Mullvad VPN)
6. Radarr hernoemt en verplaatst naar `/mnt/ssd/media/movies`
7. Jellyfin scant automatisch en voegt film toe aan bibliotheek

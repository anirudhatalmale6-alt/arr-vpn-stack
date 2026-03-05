# Full ARR Media Stack — Gluetun VPN (PIA/OpenVPN)

A complete Docker Compose media stack that routes all torrent and indexer traffic through a Gluetun VPN container with a built-in kill-switch.

## Stack Overview

| Service       | Port  | Behind VPN? | Purpose                          |
|--------------|-------|-------------|-----------------------------------|
| Gluetun      | —     | IS the VPN  | VPN gateway (PIA/OpenVPN)         |
| qBittorrent  | 8080  | YES         | Torrent client                    |
| Prowlarr     | 9696  | YES         | Indexer manager                   |
| FlareSolverr | 8191  | YES         | Cloudflare bypass for indexers    |
| Radarr       | 7878  | No          | Movie management                  |
| Sonarr       | 8989  | No          | TV show management                |
| Lidarr       | 8686  | No          | Music management                  |
| Readarr      | 8787  | No          | Book & audiobook management       |
| Bazarr       | 6767  | No          | Subtitle management               |
| Jellyfin     | 8096  | No          | Media server                      |
| Jellyseerr   | 5055  | No          | Media request manager             |

**Why only qBittorrent/Prowlarr/FlareSolverr are behind VPN:**
- They handle all the sensitive traffic (torrents, indexer searches)
- The *arr apps only talk to Prowlarr and qBittorrent over Docker's internal network
- Keeping them off the VPN means faster metadata fetches and no disruption if VPN reconnects

## Setup (5 minutes)

### Step 1: Create project folder

```bash
mkdir -p ~/arr-stack && cd ~/arr-stack
```

### Step 2: Copy files

Place `docker-compose.yml` and `.env.example` in this folder.

### Step 3: Configure environment

```bash
cp .env.example .env
nano .env
```

Fill in your PIA credentials:
```
PIA_USER=p1234567
PIA_PASS=your_password_here
PIA_REGION=Netherlands
```

Find your user/group IDs:
```bash
id -u    # PUID
id -g    # PGID
```

### Step 4: Create media directories

```bash
mkdir -p downloads media/movies media/tv media/music media/books
```

### Step 5: Launch the stack

```bash
docker compose up -d
```

Wait ~60 seconds for the VPN to connect and health check to pass.

### Step 6: Verify VPN is working

```bash
# Check Gluetun's external IP (should be VPN, NOT your real IP)
docker exec gluetun wget -qO- https://ipinfo.io

# Check qBittorrent sees the VPN IP too
docker exec qbittorrent wget -qO- https://ipinfo.io

# Compare with your real IP
curl -s https://ipinfo.io
```

If the IPs from gluetun/qbittorrent are different from your real IP — you're protected!

## Web UIs

After startup, access these from your browser:

| Service       | URL                                | Default Login                    |
|--------------|-------------------------------------|----------------------------------|
| qBittorrent  | `http://<your-ip>:8080`            | admin / check logs (see below)   |
| Prowlarr     | `http://<your-ip>:9696`            | Set on first launch              |
| FlareSolverr | `http://<your-ip>:8191`            | No login needed                  |
| Radarr       | `http://<your-ip>:7878`            | Set on first launch              |
| Sonarr       | `http://<your-ip>:8989`            | Set on first launch              |
| Lidarr       | `http://<your-ip>:8686`            | Set on first launch              |
| Readarr      | `http://<your-ip>:8787`            | Set on first launch              |
| Bazarr       | `http://<your-ip>:6767`            | Set on first launch              |
| Jellyfin     | `http://<your-ip>:8096`            | Set on first launch              |
| Jellyseerr   | `http://<your-ip>:5055`            | Set on first launch              |

Get qBittorrent's temporary password:
```bash
docker logs qbittorrent 2>&1 | grep "temporary password"
```

## CasaOS Integration

This compose file is fully compatible with CasaOS:
1. In CasaOS, go to **Apps** → **Custom Install** → **Import docker-compose**
2. Paste the contents of `docker-compose.yml`
3. Make sure the `.env` file is in the same directory CasaOS uses
4. Or: set the environment variables directly in the CasaOS UI for each container

**Alternative:** Run from terminal alongside CasaOS:
```bash
cd ~/arr-stack
docker compose up -d
```
The containers will appear in CasaOS dashboard automatically.

## Connecting the Apps Together

### 1. Prowlarr → qBittorrent (Download Client)
- In Prowlarr: Settings → Download Clients → Add → qBittorrent
- Host: `localhost` (they share the same network via Gluetun)
- Port: `8080`

### 2. Prowlarr → FlareSolverr
- In Prowlarr: Settings → Indexers → Add → FlareSolverr
- Host: `http://localhost:8191`
- (FlareSolverr runs on the same Gluetun network as Prowlarr)

### 3. Prowlarr → All *arr Apps (App Sync)
- In Prowlarr: Settings → Apps → Add each one:
  - **Radarr:** Prowlarr Server `http://localhost:9696`, Radarr Server `http://radarr:7878`
  - **Sonarr:** Prowlarr Server `http://localhost:9696`, Sonarr Server `http://sonarr:8989`
  - **Lidarr:** Prowlarr Server `http://localhost:9696`, Lidarr Server `http://lidarr:8686`
  - **Readarr:** Prowlarr Server `http://localhost:9696`, Readarr Server `http://readarr:8787`
- Get API keys from each app: Settings → General → API Key

### 4. *arr Apps → qBittorrent (Download Client)
- In Radarr/Sonarr/Lidarr/Readarr: Settings → Download Clients → Add → qBittorrent
- Host: `gluetun` (they reach qBit through Docker network via Gluetun's hostname)
- Port: `8080`

### 5. Bazarr → Radarr + Sonarr
- In Bazarr: Settings → Radarr / Sonarr
- Radarr: `http://radarr:7878` + API key
- Sonarr: `http://sonarr:8989` + API key

### 6. Jellyseerr → Jellyfin + Radarr + Sonarr
- On first launch, Jellyseerr will walk you through connecting Jellyfin, Radarr, and Sonarr

## Kill-Switch & DNS Leak Prevention

Built into Gluetun by default:
- **Kill-switch:** `FIREWALL=on` — blocks all traffic if VPN drops
- **DNS-over-TLS:** `DOT=on` with Cloudflare — prevents DNS leaks
- **Malicious blocking:** `BLOCK_MALICIOUS=on`

If VPN disconnects, qBittorrent, Prowlarr, and FlareSolverr **cannot reach the internet at all** until VPN reconnects. No IP leaks.

## Troubleshooting

### qBittorrent Web UI not loading
```bash
# Check if Gluetun is healthy
docker inspect gluetun | grep -i health

# Check Gluetun logs for connection issues
docker logs gluetun --tail 50

# Make sure VPN is connected before qBit starts
docker restart gluetun && sleep 30 && docker restart qbittorrent
```

### VPN won't connect
```bash
# Check your credentials
docker logs gluetun 2>&1 | grep -i "auth\|error\|fail"

# Try a different region in .env
PIA_REGION=Switzerland
```

### Port conflicts
Change the ports in `.env` — no need to touch the compose file.

## Updating

```bash
docker compose pull    # Pull latest images
docker compose up -d   # Recreate with new images
```

## Stopping

```bash
docker compose down           # Stop all containers
docker compose down -v        # Stop and remove volumes (WARNING: deletes configs)
```

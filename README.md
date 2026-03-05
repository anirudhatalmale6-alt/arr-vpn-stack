# ARR Media Stack — Gluetun VPN (PIA/OpenVPN)

A Docker Compose stack that routes all torrent and indexer traffic through a Gluetun VPN container with a built-in kill-switch.

## Stack Overview

| Service      | Port  | Behind VPN? | Purpose               |
|-------------|-------|-------------|------------------------|
| Gluetun     | —     | IS the VPN  | VPN gateway (PIA/OpenVPN) |
| qBittorrent | 8080  | YES         | Torrent client          |
| Prowlarr    | 9696  | YES         | Indexer manager         |
| Radarr      | 7878  | No          | Movie management        |
| Jellyfin    | 8096  | No          | Media server            |
| Jellyseerr  | 5055  | No          | Media request manager   |

**Why Radarr/Jellyfin/Jellyseerr are NOT behind VPN:**
- They only communicate with Prowlarr and qBittorrent over Docker's internal network
- Keeping them off the VPN means faster metadata fetches and no disruption if VPN reconnects
- qBittorrent + Prowlarr handle all the sensitive traffic and ARE behind VPN

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
mkdir -p downloads media/movies media/tv media/music
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

- **qBittorrent:** `http://<your-server-ip>:8080`
  - Default login: `admin` / check container logs for temp password:
    ```bash
    docker logs qbittorrent 2>&1 | grep "temporary password"
    ```
- **Prowlarr:** `http://<your-server-ip>:9696`
- **Radarr:** `http://<your-server-ip>:7878`
- **Jellyfin:** `http://<your-server-ip>:8096`
- **Jellyseerr:** `http://<your-server-ip>:5055`

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

### 2. Prowlarr → Radarr (App Sync)
- In Prowlarr: Settings → Apps → Add → Radarr
- Prowlarr Server: `http://localhost:9696`
- Radarr Server: `http://radarr:7878`
- Get API key from Radarr: Settings → General → API Key

### 3. Radarr → qBittorrent (Download Client)
- In Radarr: Settings → Download Clients → Add → qBittorrent
- Host: `gluetun` (Radarr reaches qBit through Docker network via Gluetun's hostname)
- Port: `8080`

### 4. Jellyseerr → Jellyfin + Radarr
- On first launch, Jellyseerr will walk you through connecting Jellyfin and Radarr

## Kill-Switch & DNS Leak Prevention

Built into Gluetun by default:
- **Kill-switch:** `FIREWALL=on` — blocks all traffic if VPN drops
- **DNS-over-TLS:** `DOT=on` with Cloudflare — prevents DNS leaks
- **Malicious blocking:** `BLOCK_MALICIOUS=on`

If VPN disconnects, qBittorrent and Prowlarr **cannot reach the internet at all** until VPN reconnects. No IP leaks.

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

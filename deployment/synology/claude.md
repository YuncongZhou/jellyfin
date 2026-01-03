# Claude Code: Synology DS920+ Jellyfin Automated Setup Guide

This file guides Claude Code through automating Jellyfin deployment on a Synology DS920+ NAS. Follow these steps sequentially, confirming each phase with the user before proceeding.

## Prerequisites Check

Before starting, verify:
1. SSH access to the Synology NAS is enabled (Control Panel → Terminal & SNMP → Enable SSH)
2. User has admin credentials for SSH
3. Container Manager (Docker) is installed from Package Center
4. SHR volume is configured (2x20TB HDDs on `/volume1`)

Ask the user for:
- NAS IP address
- SSH username and password
- Timezone (e.g., `America/New_York`, `Europe/London`)
- Desired Jellyfin admin username

## Phase 1: Directory Structure Setup

Connect via SSH and create the folder structure:

```bash
# Create Docker config directories
sudo mkdir -p /volume1/docker/{jellyfin/config,jellyfin/cache,sonarr,radarr,prowlarr,bazarr,shoko,autoscan,qbittorrent}

# Create data directories for hardlinks
sudo mkdir -p /volume1/data/torrents/{movies,tv}
sudo mkdir -p /volume1/data/media/{movies,tv,anime}

# Set permissions (replace 1026:100 with user's UID:GID from 'id username')
sudo chown -R 1026:100 /volume1/docker /volume1/data
sudo chmod -R 755 /volume1/docker /volume1/data
```

To get the correct UID/GID, run: `id <username>`

## Phase 2: Hardware Transcoding Permissions

Create the boot task for `/dev/dri` access:

1. This must be done via DSM UI: Control Panel → Task Scheduler → Create → Triggered Task → User-defined script
2. Settings:
   - Task: `Fix Intel GPU Permissions`
   - User: `root`
   - Event: `Boot-up`
   - Script: `chmod 777 /dev/dri/*`

Verify GPU devices exist:
```bash
ls -la /dev/dri/
# Should show: card0, renderD128
```

## Phase 3: Docker Compose Deployment

Create `/volume1/docker/docker-compose.yml`:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: "1026:100"  # UPDATE: Use actual UID:GID
    environment:
      - TZ=America/New_York  # UPDATE: User's timezone
      - JELLYFIN_PublishedServerUrl=http://NAS_IP  # UPDATE: NAS IP
    volumes:
      - /volume1/docker/jellyfin/config:/config:rw
      - /volume1/docker/jellyfin/cache:/cache:rw
      - /volume1/data/media/movies:/media/movies:ro
      - /volume1/data/media/tv:/media/tv:ro
      - /volume1/data/media/anime:/media/anime:ro
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
      - /dev/dri/card0:/dev/dri/card0
    ports:
      - 8096:8096
      - 8920:8920
      - 7359:7359/udp
    restart: unless-stopped
    networks:
      - media

  shoko_server:
    image: ghcr.io/shokoanime/server:latest
    container_name: shoko_server
    shm_size: 256m
    environment:
      - PUID=1026  # UPDATE
      - PGID=100   # UPDATE
      - TZ=America/New_York  # UPDATE
    ports:
      - 8111:8111
    volumes:
      - /volume1/docker/shoko:/home/shoko/.shoko
      - /volume1/data/media/anime:/mnt/anime
    restart: unless-stopped
    networks:
      - media

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1026  # UPDATE
      - PGID=100   # UPDATE
      - TZ=America/New_York  # UPDATE
    volumes:
      - /volume1/docker/sonarr:/config
      - /volume1/data:/data
    ports:
      - 8989:8989
    restart: unless-stopped
    networks:
      - media

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1026  # UPDATE
      - PGID=100   # UPDATE
      - TZ=America/New_York  # UPDATE
    volumes:
      - /volume1/docker/radarr:/config
      - /volume1/data:/data
    ports:
      - 7878:7878
    restart: unless-stopped
    networks:
      - media

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1026  # UPDATE
      - PGID=100   # UPDATE
      - TZ=America/New_York  # UPDATE
    volumes:
      - /volume1/docker/prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped
    networks:
      - media

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      - PUID=1026  # UPDATE
      - PGID=100   # UPDATE
      - TZ=America/New_York  # UPDATE
    volumes:
      - /volume1/docker/bazarr:/config
      - /volume1/data/media:/media
    ports:
      - 6767:6767
    restart: unless-stopped
    networks:
      - media

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1026  # UPDATE
      - PGID=100   # UPDATE
      - TZ=America/New_York  # UPDATE
      - WEBUI_PORT=8080
    volumes:
      - /volume1/docker/qbittorrent:/config
      - /volume1/data/torrents:/data/torrents
    ports:
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
    networks:
      - media

  autoscan:
    image: ghcr.io/niniyas/autoscan:latest
    container_name: autoscan
    environment:
      - PUID=1026  # UPDATE
      - PGID=100   # UPDATE
    volumes:
      - /volume1/docker/autoscan:/config
      - /volume1/data/media:/media
    ports:
      - 3468:3468
    restart: unless-stopped
    networks:
      - media

networks:
  media:
    driver: bridge
```

Deploy with:
```bash
cd /volume1/docker && sudo docker-compose up -d
```

## Phase 4: Post-Deployment Configuration

### 4.1 Jellyfin Initial Setup
1. Access `http://NAS_IP:8096`
2. Complete wizard: language, admin user, library paths
3. Configure transcoding: Dashboard → Playback → Transcoding
   - Hardware acceleration: Intel QuickSync (QSV)
   - Enable: H264, HEVC, MPEG2, VC1, VP8, VP9 hardware decoding
   - Enable: HEVC 10-bit, VP9 10-bit
   - Enable: VPP Tone Mapping
   - **Disable**: Low-power encoding, Low-power decoding

### 4.2 Install Jellyfin Plugins
Navigate to Dashboard → Plugins → Repositories, add these:

| Plugin | Repository URL |
|--------|----------------|
| Shokofin | `https://raw.githubusercontent.com/ShokoAnime/Shokofin/metadata/stable/manifest.json` |
| Intro Skipper | `https://raw.githubusercontent.com/intro-skipper/intro-skipper/master/manifest.json` |
| jellyfin-ani-sync | `https://raw.githubusercontent.com/vosmiic/jellyfin-ani-sync/master/manifest.json` |

Then install from Catalog: Shoko, Intro Skipper, AniSync, Playback Reporting, Webhook

### 4.3 Shoko Server Setup
1. Access `http://NAS_IP:8111`
2. Create AniDB account at anidb.net (required for API)
3. Add import folder: `/mnt/anime`
4. Start file hashing (runs in background)

### 4.4 Configure Shokofin in Jellyfin
1. Dashboard → Plugins → Shoko → Settings
2. Host: `http://shoko_server:8111` (Docker network) or `http://NAS_IP:8111`
3. Enable VFS mode
4. Restart Jellyfin

### 4.5 Autoscan Configuration
Create `/volume1/docker/autoscan/config.json`:

```json
{
  "minimum_age": 10,
  "scan_delay": 180,
  "triggers": {
    "sonarr": {
      "priority": 1
    },
    "radarr": {
      "priority": 1
    }
  },
  "targets": {
    "jellyfin": {
      "url": "http://jellyfin:8096",
      "token": "YOUR_JELLYFIN_API_KEY"
    }
  }
}
```

Get Jellyfin API key: Dashboard → API Keys → Create

### 4.6 Configure Sonarr/Radarr Webhooks
In each *arr app: Settings → Connect → Add → Webhook
- URL: `http://autoscan:3468/triggers/sonarr` (or `/triggers/radarr`)
- Triggers: On Import, On Upgrade, On Rename

### 4.7 Root Folder Configuration
- **Sonarr**: Settings → Media Management → Root Folders → Add `/data/media/tv`
- **Radarr**: Settings → Media Management → Root Folders → Add `/data/media/movies`
- **qBittorrent**: Downloads path → `/data/torrents/`

## Phase 5: Verification Checklist

Run these checks to confirm successful setup:

```bash
# Check all containers running
sudo docker ps --format "table {{.Names}}\t{{.Status}}"

# Verify Jellyfin hardware transcoding
sudo docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo

# Check GPU access
ls -la /dev/dri/

# Test Jellyfin API
curl -s "http://localhost:8096/System/Info/Public" | jq .
```

Expected `vainfo` output should show:
- `vainfo: VA-API version: 1.x.x`
- `vainfo: Driver version: Intel iHD driver`
- Multiple `VAProfile` entries for H264, HEVC, VP9

## Troubleshooting Commands

```bash
# View container logs
sudo docker logs jellyfin --tail 100
sudo docker logs shoko_server --tail 100

# Restart all containers
cd /volume1/docker && sudo docker-compose restart

# Rebuild containers (after compose changes)
cd /volume1/docker && sudo docker-compose up -d --force-recreate

# Check disk usage
df -h /volume1

# Fix permissions if needed
sudo chown -R 1026:100 /volume1/docker /volume1/data
```

## Quick Reference: Service URLs

| Service | URL | Default Credentials |
|---------|-----|---------------------|
| Jellyfin | `http://NAS_IP:8096` | Set during wizard |
| Shoko Server | `http://NAS_IP:8111` | Set during wizard |
| Sonarr | `http://NAS_IP:8989` | None (set auth in settings) |
| Radarr | `http://NAS_IP:7878` | None (set auth in settings) |
| Prowlarr | `http://NAS_IP:9696` | None (set auth in settings) |
| Bazarr | `http://NAS_IP:6767` | None (set auth in settings) |
| qBittorrent | `http://NAS_IP:8080` | admin / adminadmin |
| Autoscan | `http://NAS_IP:3468` | N/A |

## Notes for Claude Code

1. **Always ask for user confirmation** before executing SSH commands on the NAS
2. **Substitute placeholders**: Replace `NAS_IP`, `1026:100`, timezone values with user-provided data
3. **Check prerequisites** before each phase
4. **Verify success** after each major step using the provided check commands
5. **The boot task for /dev/dri must be created via DSM UI** - cannot be automated via SSH
6. Refer to `README.md` in this directory for detailed explanations of each component

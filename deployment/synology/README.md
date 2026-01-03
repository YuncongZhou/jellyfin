# Comprehensive Jellyfin customization guide for Synology DS920+

Your optimal setup combines **Docker-deployed Jellyfin with Intel Quick Sync hardware transcoding**, **Shoko Anime Server for metadata** (solving your anime collection challenges), **Sonarr/Radarr for file organization**, **autoscan for real-time library updates**, and **Kodi + Jellyfin on Google TV with Infuse on Apple TV** for the best playback experience. This guide provides production-ready configurations and addresses every customization area you specified.

## Installation decision: Docker wins definitively

**Docker containers are the clear choice** for Jellyfin on DS920+. No official Synology native package exists—only a community SynoCommunity package that lacks hardware transcoding support and receives delayed updates. Docker offers full Intel Quick Sync access via device passthrough, immediate access to official images, container isolation for security, and easy rollback via image versioning.

Between the two main images, **jellyfin/jellyfin (official)** edges out linuxserver/jellyfin for your DS920+ because it has native Intel QSV support without requiring additional Docker Mods. The linuxserver image requires `DOCKER_MODS=linuxserver/mods:jellyfin-opencl-intel` for OpenCL tone mapping—extra complexity for no benefit on Gemini Lake processors.

## Storage configuration: SHR on 2x20TB HDDs

This guide assumes **SHR (Synology Hybrid RAID)** with **2x20TB HDDs**, providing approximately 20TB usable storage with single-disk fault tolerance. SHR is the recommended RAID type for Synology—it offers the flexibility to expand with different drive sizes later while maintaining redundancy.

**Volume layout for media server workloads**:
- `/volume1` - Your primary SHR volume on the HDDs (media storage, downloads, Docker containers)
- Consider creating a separate SSD volume if you install M.2 NVMe drives for databases and cache

**Recommended shared folder structure**:
```
/volume1/
├── docker/           # Container configs (consider SSD for databases)
│   ├── jellyfin/
│   │   ├── config/   # Jellyfin database and settings
│   │   └── cache/    # Transcoding cache
│   ├── sonarr/
│   ├── radarr/
│   ├── shoko/
│   └── ...
├── data/             # Single parent for hardlinks
│   ├── torrents/     # Active downloads
│   │   ├── movies/
│   │   └── tv/
│   └── media/        # Organized library
│       ├── movies/
│       ├── tv/
│       └── anime/
└── media/            # Alternative: direct media folders
    ├── movies/
    ├── tv/
    └── anime/
```

**SHR performance considerations**: HDDs are fine for media streaming (sequential reads), but Jellyfin's SQLite database benefits significantly from SSD storage. If you add M.2 drives to the DS920+'s NVMe slots, create a separate volume and mount `/config` there, or enable SSD read-write caching for the HDD volume.

## Hardware transcoding capabilities

Your DS920+'s **Intel Celeron J4125** (Gemini Lake Refresh, UHD Graphics 600) supports hardware decode/encode for H.264, HEVC 8-bit/10-bit, VP9 8-bit, and MPEG-2. It cannot handle AV1 (requires 11th Gen+) and low-power encoding modes cause stability issues—disable these explicitly. Here's your production Docker Compose:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: 1026:100  # Your UID:GID from 'id username'
    environment:
      - TZ=Your/Timezone
      - JELLYFIN_PublishedServerUrl=http://YOUR_NAS_IP
    volumes:
      - /volume1/docker/jellyfin/config:/config:rw  # SSD recommended
      - /volume1/docker/jellyfin/cache:/cache:rw    # SSD strongly recommended
      - /volume1/media/movies:/media/movies:ro
      - /volume1/media/tv:/media/tv:ro
      - /volume1/media/anime:/media/anime:ro
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
      - /dev/dri/card0:/dev/dri/card0
    ports:
      - 8096:8096
      - 8920:8920
      - 7359:7359/udp
    restart: unless-stopped
```

**Critical DSM 7 fix**: Synology blocks non-root access to `/dev/dri`. Create a triggered boot task (Control Panel → Task Scheduler → Triggered Task → Boot-up) running as root: `chmod 777 /dev/dri/*`. Without this, hardware transcoding silently fails with "No VA display found" errors.

After deployment, configure transcoding at Dashboard → Playback → Transcoding: select **Intel QuickSync (QSV)**, enable hardware decoding for H264/HEVC/MPEG2/VC1/VP8/VP9, enable HEVC and VP9 10-bit, enable VPP Tone Mapping, but **disable low-power encoding and decoding**—Gemini Lake doesn't handle these properly.

## Solving anime metadata: Shoko Server is the answer

Anime metadata failures stem from fundamental database conflicts. **AniDB uses absolute episode numbering** (Episode 1-500 for long series), **TVDB forces western season structures** (S01E01-S22E37), and TMDB uses yet another division scheme. When your fansub files are named with absolute numbers but Jellyfin defaults to TVDB, episodes land in wrong seasons or fail entirely. OVAs, specials, and movies have inconsistent categorization across all databases.

The **Shoko Anime Server + Shokofin plugin combination** solves this definitively. Shoko uses ED2K file hashing to match your files against AniDB's database of millions of known file hashes—**no filename parsing needed**. Even `[SubGroup]Show-EP01[1080p].mkv` identifies correctly. Shokofin then creates a Virtual File System (VFS) with proper structures via symlinks, meaning you don't reorganize your existing files.

Deploy Shoko Server alongside Jellyfin:

```yaml
services:
  shoko_server:
    image: ghcr.io/shokoanime/server:latest
    container_name: shoko_server
    shm_size: 256m
    environment:
      - PUID=1026
      - PGID=100
      - TZ=Your/Timezone
    ports:
      - 8111:8111
    volumes:
      - /volume1/docker/shoko:/home/shoko/.shoko
      - /volume1/media/anime:/mnt/anime
    restart: always
```

Access Shoko at `http://NAS_IP:8111`, create an AniDB account (required for API access), add your anime import folder, and let it hash your collection. Initial hashing takes hours for large libraries but only runs once per file.

Install Shokofin in Jellyfin: Dashboard → Plugins → Manage Repositories → add `https://raw.githubusercontent.com/ShokoAnime/Shokofin/metadata/stable/manifest.json`, then install "Shoko" from the Anime section. Configure connection to your Shoko Server, enable VFS, and let Shokofin handle library structure.

For watch status sync to MyAnimeList/AniList/Kitsu, add **jellyfin-ani-sync**: repository URL `https://raw.githubusercontent.com/vosmiic/jellyfin-ani-sync/master/manifest.json`. This syncs your viewing progress across anime tracking services.

If you prefer a lighter solution without Shoko, the built-in **AniDB and AniList plugins** work for smaller, well-organized collections—but expect manual intervention for long-running series and multi-season anime.

## Torrent filename parsing with the *arr ecosystem

Jellyfin expects specific naming: `Movie Name (Year)/Movie Name (Year).mkv` for movies and `Series Name (Year)/Season XX/Series Name SxxExx.mkv` for shows. Scene releases like `Movie.Name.2023.1080p.BluRay.x264-GROUP.mkv` often mismatch without proper folder structure. Characters `< > : " / \ | ? *` break parsing entirely.

**Sonarr and Radarr automate this completely**. They monitor for releases, download via your torrent client, rename to Jellyfin-compatible formats, and organize into proper folder structures. The TRaSH Guides naming schemes include database IDs for guaranteed matching:

Radarr folder format: `{Movie CleanTitle} ({Release Year}) [tmdbid-{TmdbId}]`
Sonarr folder format: `{Series TitleYear} [tvdbid-{TvdbId}]`

For hardlinks to work (allowing continued seeding while media appears in your library), all downloads and media must share a parent directory:

```yaml
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    volumes:
      - /volume1/docker/sonarr:/config
      - /volume1/data:/data  # Single parent for hardlinks
    ports:
      - 8989:8989

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    volumes:
      - /volume1/docker/radarr:/config
      - /volume1/data:/data
    ports:
      - 7878:7878

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    volumes:
      - /volume1/docker/qbittorrent:/config
      - /volume1/data/torrents:/data/torrents
    ports:
      - 8080:8080
```

Structure your `/volume1/data` as:
```
/volume1/data/
├── torrents/
│   ├── movies/
│   └── tv/
└── media/
    ├── movies/
    └── tv/
```

Add **Bazarr** for automatic subtitle downloads (integrates with OpenSubtitles, Subscene) and **Prowlarr** as a centralized indexer manager that syncs to all *arr apps.

## Real-time indexing with autoscan and API triggers

Jellyfin's built-in scanning is slow because each item triggers multiple metadata API calls and ffprobe runs. Version 10.11 introduced performance regressions—photo libraries went from 26 seconds to 86 minutes in some cases. Full library scans trigger even for single file additions.

The solution is **webhook-triggered scanning via autoscan** rather than scheduled or filesystem-monitored scans:

```yaml
services:
  autoscan:
    image: ghcr.io/niniyas/autoscan:latest
    container_name: autoscan
    volumes:
      - /volume1/docker/autoscan:/config
      - /volume1/media:/media
    ports:
      - 3468:3468
```

Configure autoscan's `config.json`:
```json
{
  "ENABLE_JOE": true,
  "JELLYFIN_EMBY": "jellyfin",
  "JOE_API_KEY": "your-jellyfin-api-key",
  "JOE_HOST": "http://jellyfin:8096",
  "SERVER_SCAN_DELAY": 180,
  "SERVER_PATH_MAPPINGS": {
    "/media/Movies/": ["/movies/"],
    "/media/TV/": ["/tv/"]
  }
}
```

In Sonarr/Radarr, add webhooks (Settings → Connect → Webhook) pointing to `http://autoscan:3468/YOUR_SERVER_PASS` with On Import, On Upgrade, and On Rename triggers. When Sonarr/Radarr imports media, autoscan batches notifications and triggers targeted Jellyfin scans.

For manual API triggering:
```bash
# Full library refresh
curl -X POST "http://localhost:8096/Library/Refresh" \
  -H "Authorization: MediaBrowser Token=YOUR_API_KEY"

# Refresh specific library by ID
curl -X POST "http://localhost:8096/Items/LIBRARY_ID/Refresh" \
  -H "Authorization: MediaBrowser Token=YOUR_API_KEY"
```

**Move Jellyfin's database to SSD** for dramatic performance improvement. The `/config` volume containing `library.db` benefits enormously from SSD—place it on your DS920+'s M.2 cache drives configured as a separate volume, or ensure your Docker volume has read-write SSD caching enabled.

## Client setup for Sony A95L and Apple TV

### Google TV (Sony A95L): Kodi + Jellyfin is the best option

The official Jellyfin Android TV app has **significant HDR/Dolby Vision issues**—DV Profile 7/8 sometimes plays as SDR or HDR10, black screens occur with some DV content, and critically, **there's no option to allow ASS/SSA subtitles in direct play** on the TV client. Your anime subtitles with styled fonts will force transcoding.

**Kodi with the Jellyfin add-on** solves everything:
- Plays virtually any codec natively
- **Proper ASS/SSA subtitle rendering** with fonts and styling (critical for anime)
- Reliable HDR/DV passthrough
- Two-way watched status sync
- Remote control via Jellyfin apps

Setup: Install Kodi, download the Jellyfin repository ZIP from jellyfin.org, install via Add-ons → Install from ZIP, then install "Jellyfin for Kodi" and optionally the "Kodi Sync Queue" server plugin for real-time sync.

**Findroid** is an alternative using mpv for playback with excellent ASS subtitle support, but its TV navigation is awkward—designed for mobile.

### Apple TV: Infuse Pro is worth the $99

Swiftfin (official Jellyfin tvOS client) has critical issues: **TrueHD 7.1/Atmos doesn't work**, Dolby Vision displays incorrectly when Apple TV is set to DV mode, and the tvOS version significantly lags behind the iOS version. A major tvOS update is in development but has no release date.

**Infuse Pro** ($99 lifetime) handles everything:
- Direct plays virtually all formats including 4K HDR10/HDR10+/Dolby Vision
- TrueHD/Atmos and DTS-HD MA passthrough
- **Excellent ASS/SSA subtitle rendering** for anime
- Polished native Apple TV interface
- Watched status sync with Jellyfin

Setup: Settings → Add Files → Media Servers → Jellyfin → enter credentials. Install InfuseSync plugin on Jellyfin server for better synchronization. Use Direct Mode for large libraries (faster updates, smaller footprint).

### Maximizing direct play across both platforms

Transcoding triggers to avoid:
- **ASS/SSA subtitles** on official Jellyfin apps (use Kodi/Infuse instead)
- PGS bitmap subtitles often require burn-in
- TrueHD/DTS-HD on devices without passthrough
- AV1 on your DS920+ (no hardware decode)
- Bitrates over **80 Mbps** stress some clients

For anime specifically, clients using mpv or VLC playback engines (Findroid, Infuse, Kodi) render styled ASS subtitles correctly. Official Jellyfin apps don't.

## Common issues and critical fixes

### Database problems

**"Database is locked"** errors come from plugin incompatibilities or improper shutdowns. Stop Jellyfin, remove problematic plugins, delete stale `.db-shm` and `.db-wal` files, ensure only one container accesses the data directory.

**Corrupted database** recovery:
```bash
sqlite3 library.db ".recover" | sqlite3 library-recovered.db
sqlite3 library-recovered.db "PRAGMA integrity_check"
# Replace original if integrity_check passes
```

Community repair tool: https://github.com/MadGoatHaz/JellyFin-DB-Fix

### Subtitle burn-in failures

Hardware-accelerated subtitle burn-in on HEVC with QSV/VAAPI fails with FFmpeg code 251. Workaround: use software transcoding for problematic files or disable subtitles. This primarily affects PGS/VOBSUB bitmap subtitles requiring burn-in.

### Version-specific issues

**Jellyfin 10.11.x** has tracked performance issues (GitHub #15685) and database migration problems (#15686). If running 10.11 RC versions, avoid upgrading from ≤RC5 to RC7 (library breakage)—use RC8+.

**10.9.x networking issues**: "External request received, no matching external bind address found" after upgrade. Verify Published Server URL and bind addresses in network settings.

## Advanced customization for power users

### Essential plugins worth installing

| Plugin | Repository URL | Purpose |
|--------|----------------|---------|
| **Intro Skipper** | `https://raw.githubusercontent.com/intro-skipper/intro-skipper/master/manifest.json` | Netflix-style skip intros via audio fingerprinting |
| **Jellyfin-Enhanced** | `https://github.com/n00bcodr/Jellyfin-Enhanced` | Keyboard shortcuts, TMDB reviews, Jellyseerr integration |
| **Playback Reporting** | Built-in catalog | Watch statistics and history |
| **Webhook** | Built-in catalog | Discord/Slack/Gotify notifications |

### Jellyseerr for media requests

Deploy Jellyseerr (`fallenbagel/jellyseerr` on port 5055) for user media requests that auto-route to Sonarr/Radarr. This replaces Ombi with full Jellyfin integration. Users can request content, admins approve, and the *arr apps handle acquisition and organization automatically.

### Jellystat for Tautulli-like monitoring

https://github.com/CyferShepard/Jellystat provides session monitoring, watch history per user/library, and activity statistics—essentially Tautulli for Jellyfin. Requires PostgreSQL backend.

### Source code modifications

Fork `jellyfin/jellyfin` (C#/.NET backend) or `jellyfin/jellyfin-web` (TypeScript/React frontend). Build with `dotnet build`. Plugin development template: https://github.com/jellyfin/jellyfin-plugin-template. Key interfaces: `IScheduledTask` for background jobs, `IMetadataSaver` for custom formats, `ControllerBase` for REST APIs.

### Webhook events available

Playback (start/stop/pause/progress), authentication (success/failure), library changes (item added/updated/deleted), user management, system events. Configure at Dashboard → Plugins → Webhook with destination templates for Discord, Slack, Home Assistant, or generic HTTP endpoints.

## Complete deployment checklist

1. **Install Container Manager** from Package Center
2. **Create boot task** for `/dev/dri` permissions
3. **Deploy Jellyfin** via Docker Compose with hardware transcoding enabled
4. **Deploy Shoko Server** for anime metadata (if large anime collection)
5. **Deploy Sonarr/Radarr/Prowlarr/Bazarr** for media organization
6. **Deploy autoscan** for real-time library updates
7. **Configure webhooks** from *arr apps to autoscan
8. **Install Shokofin + jellyfin-ani-sync** plugins for anime
9. **Install Intro Skipper** plugin for skip functionality
10. **Configure clients**: Kodi on Sony A95L, Infuse on Apple TV
11. **Optional**: Deploy Jellyseerr for user requests, Jellystat for monitoring

## Key resources

- Jellyfin Synology documentation: https://jellyfin.org/docs/general/installation/advanced/synology/
- Intel hardware transcoding guide: https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/
- Shoko Anime documentation: https://docs.shokoanime.com/
- TRaSH Guides for *arr setup: https://trash-guides.info/
- Awesome Jellyfin plugin list: https://github.com/awesome-jellyfin/awesome-jellyfin
- Dr Frankenstein's Synology guides: https://drfrankenstein.co.uk/

## Conclusion

The Jellyfin ecosystem has matured significantly, but optimal configuration requires deliberate choices at every layer. **Docker with jellyfin/jellyfin official image** provides the cleanest hardware transcoding path on DS920+. **Shoko Server fundamentally solves anime metadata** through file hashing rather than filename parsing—a paradigm shift that eliminates the AniDB/TVDB/TMDB conflicts plaguing anime collections. **Client selection matters enormously**: official apps have significant codec and subtitle limitations that Kodi and Infuse solve completely. **Event-driven scanning via autoscan** outperforms both scheduled scans and filesystem monitoring for Docker deployments. The combination of these components—each solving a specific problem well—creates a media server that rivals commercial offerings while remaining fully under your control.

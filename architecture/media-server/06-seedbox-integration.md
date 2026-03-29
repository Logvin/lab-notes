# 06 - Seedbox Integration

The seedbox is a remote server dedicated to downloading. It handles the bandwidth-intensive work so your media server can focus on serving content.

## Why Use a Seedbox

- **Bandwidth:** Multi-gigabit connections, purpose-built for high-throughput transfers
- **IP separation:** Your media server's IP never appears in torrent swarms
- **ISP compliance:** No torrent traffic on your media server's network
- **Always-on seeding:** Maintain ratio requirements without consuming your server resources
- **Dedicated storage:** Temporary storage for downloads before transfer

## Seedbox Providers

Common options:

| Provider | Storage | Bandwidth | SSH Access | Price Range |
|----------|---------|-----------|------------|-------------|
| Feral Hosting | 1-6TB | 10Gbps | Yes | $15-40/mo |
| Whatbox | 1-4TB | 10Gbps | Yes | $15-35/mo |
| Ultra.cc | 1-4TB | 20Gbps | Yes | $10-30/mo |
| Seedbox.io | 1-3TB | 10Gbps | Yes | $10-25/mo |

**Requirements:**
- SSH access (for rsync and scripting)
- qBittorrent (or similar client with API)
- Enough storage for active downloads + seeding buffer
- Public IP or port forwarding for the torrent client

## qBittorrent Configuration

### Categories

Set up categories in qBittorrent that match your Sonarr/Radarr download clients:

| Category | Save Path | Used By |
|----------|-----------|---------|
| `sonarr` | `/downloads/sonarr/` | Sonarr |
| `radarr` | `/downloads/radarr/` | Radarr |

### Seeding Limits

Configure seeding behavior:

- **Ratio limit:** 1.0 (seed until 1:1 ratio, then stop)
- **Time limit:** 72 hours (or whatever your tracker requires)
- **Action when limits reached:** Pause torrent

These limits balance being a good community member with not filling your seedbox.

### Connection to Sonarr/Radarr

Sonarr and Radarr connect to the seedbox's qBittorrent API. If the seedbox doesn't allow direct port access to the WebUI, use an SSH tunnel.

## SSH Tunnel Proxy

When the seedbox doesn't expose the qBittorrent WebUI port directly, set up a persistent SSH tunnel:

```bash
# On your media server, create a tunnel to the seedbox's qBit WebUI
ssh -L 8090:localhost:8080 seeduser@seedbox.example.com -N -f
```

This forwards `localhost:8090` on your media server to `localhost:8080` on the seedbox (where qBittorrent's WebUI listens).

For a production setup, use autossh for automatic reconnection:

```bash
sudo apt install autossh

# autossh with monitoring port
autossh -M 20000 \
    -L 8090:localhost:8080 \
    seeduser@seedbox.example.com \
    -N -f \
    -o "ServerAliveInterval 30" \
    -o "ServerAliveCountMax 3"
```

Or create a systemd service:

```ini
[Unit]
Description=SSH tunnel to seedbox qBittorrent
After=network-online.target
Wants=network-online.target

[Service]
User=mediauser
ExecStart=/usr/bin/autossh -M 20000 \
    -L 8090:localhost:8080 \
    seeduser@seedbox.example.com \
    -N \
    -o "ServerAliveInterval 30" \
    -o "ServerAliveCountMax 3" \
    -o "ExitOnForwardFailure yes"
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Then configure Sonarr/Radarr to use `localhost:8090` as the qBittorrent host.

## Automated Pipeline Script

The ExtractAndTransfer script runs on the seedbox via cron. It has three phases:

### Phase 1: Scan

Query the qBittorrent API for completed downloads:

```bash
# Get list of completed torrents in the 'sonarr' category
curl -s "http://localhost:8080/api/v2/torrents/info?filter=completed&category=sonarr"
```

### Phase 2: Stage

For each completed download:
- Check if files need extraction (RAR archives)
- Extract to a staging directory
- Track which torrents have been processed (avoid re-extracting)

### Phase 3: Transfer

rsync staged files to the media server:

```bash
rsync -avP --remove-source-files \
    /staging/sonarr/ \
    rsync://mediauser@192.0.2.10/staging/sonarr/
```

The `--remove-source-files` flag cleans up the seedbox staging area after successful transfer. The original download directory is untouched (for continued seeding).

See [`config-examples/extract-and-transfer.sh.example`](config-examples/extract-and-transfer.sh.example) for the full script.

## Cron Scheduling

Run the pipeline every 10 minutes:

```bash
# crontab -e (on the seedbox)
*/10 * * * * /home/seeduser/scripts/extract-and-transfer.sh >> /home/seeduser/logs/pipeline.log 2>&1
```

## Monitoring and Health Checks

Things that can go wrong:

| Issue | Detection | Response |
|-------|-----------|----------|
| rsync connection refused | Non-zero exit code | Alert, check media server rsync daemon |
| Disk full on seedbox | `df` check in script | Pause qBittorrent, alert |
| Disk full on media server | rsync fails | Alert, check staging cleanup |
| qBittorrent API unreachable | curl fails | Alert, check qBit service |
| Extraction fails | unrar exit code | Alert, manual review |

Add health checks to the beginning of the script:

```bash
# Check seedbox disk usage
DISK_USAGE=$(df --output=pcent /downloads | tail -1 | tr -d ' %')
if [ "$DISK_USAGE" -gt 90 ]; then
    echo "WARNING: Seedbox disk usage at ${DISK_USAGE}%"
    # Optionally pause qBittorrent
    curl -s "http://localhost:8080/api/v2/torrents/pause?hashes=all"
fi

# Check media server is reachable
if ! rsync rsync://mediauser@192.0.2.10/ > /dev/null 2>&1; then
    echo "ERROR: Cannot reach media server rsync daemon"
    exit 1
fi
```

## Next Steps

With the seedbox pipeline in place, see [07 - Cloud Sync](07-cloud-sync.md) for moving media from local storage to the cloud.

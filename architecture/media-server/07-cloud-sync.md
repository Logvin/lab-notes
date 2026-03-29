# 07 - Cloud Sync

Media files start on local storage after import. Cloud sync moves them to encrypted cloud backends, freeing local disk space while keeping everything accessible through rclone mounts and MergerFS.

## Strategy

The sync follows a simple lifecycle:

```
Download → Import locally → Sync to cloud → Verify → Clean local copy
```

Local storage is a staging area for fresh media. Cloud storage is the permanent home. MergerFS presents both as a single path, so Plex doesn't care where the file physically lives.

## Why Multiple Cloud Providers

Relying on a single cloud provider is a single point of failure:

| Risk | Mitigation |
|------|-----------|
| Provider shuts down | Media spread across 2-3 providers |
| Account suspension | Majority of library survives on other providers |
| API rate limits | Load distributed across providers |
| Price increases | Flexibility to shift to the cheapest option |
| Storage quota limits | Aggregate capacity across providers |

A practical split:

| Provider | Use Case | Typical Cost |
|----------|----------|-------------|
| Google Drive (Team/Workspace) | Primary storage, largest library | $12-20/mo |
| Backblaze B2 | Secondary, overflow | $5/TB/mo |
| Blomp (Swift) | Tertiary, free tier | Free (with limits) |

## rclone copy vs sync

**Always use `rclone copy`, not `rclone sync`.**

- `rclone copy` copies files from source to destination that don't exist at the destination
- `rclone sync` makes the destination match the source, **deleting** files at the destination that don't exist at the source

If you accidentally run `rclone sync` from an empty local directory to your cloud storage, it will delete everything in the cloud. Use `copy`. Always.

```bash
# Safe: copy new files to cloud
rclone copy /local/storage/mounts/local/TV\ Shows/ gdrive-crypt:TV\ Shows/ \
    --config /local/storage/rclone.conf \
    --transfers 4 \
    --checkers 8 \
    --log-file /local/storage/logs/cloud-sync.log \
    --log-level INFO \
    --stats 1m

# DANGEROUS: sync (deletes from destination)
# rclone sync ... # DO NOT USE unless you know exactly what you're doing
```

## Nightly Sync Script

Create a script that runs nightly to push local media to the cloud:

```bash
#!/usr/bin/env bash
# /local/storage/scripts/nightly-cloud-sync.sh
#
# Syncs local media to the primary cloud backend (Google Drive, encrypted).
# Runs nightly via cron.

set -euo pipefail

RCLONE_CONFIG="/local/storage/rclone.conf"
LOG_DIR="/local/storage/logs"
LOCAL_MEDIA="/local/storage/mounts/local"
CLOUD_REMOTE="gdrive-crypt:"
DATE=$(date +%Y-%m-%d)

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "${LOG_DIR}/cloud-sync-${DATE}.log"
}

log "Starting nightly cloud sync"

# Sync TV Shows
log "Syncing TV Shows..."
rclone copy "${LOCAL_MEDIA}/TV Shows/" "${CLOUD_REMOTE}TV Shows/" \
    --config "${RCLONE_CONFIG}" \
    --transfers 4 \
    --checkers 8 \
    --log-file "${LOG_DIR}/cloud-sync-${DATE}.log" \
    --log-level INFO \
    --stats 5m \
    --exclude ".partial~" \
    --exclude "*.part"

# Sync Movies
log "Syncing Movies..."
rclone copy "${LOCAL_MEDIA}/Movies/" "${CLOUD_REMOTE}Movies/" \
    --config "${RCLONE_CONFIG}" \
    --transfers 4 \
    --checkers 8 \
    --log-file "${LOG_DIR}/cloud-sync-${DATE}.log" \
    --log-level INFO \
    --stats 5m \
    --exclude ".partial~" \
    --exclude "*.part"

log "Nightly cloud sync complete"
```

Cron entry:

```bash
# Run at 3 AM daily
0 3 * * * /local/storage/scripts/nightly-cloud-sync.sh
```

## VFS Cache Refresh After Upload

After uploading files to a cloud remote, the rclone FUSE mount may not immediately see them. The VFS cache has its own view of what's on the remote.

Force a cache refresh:

```bash
# If rclone RC (remote control) API is enabled on the mount
rclone rc vfs/refresh --rc-addr localhost:5572 recursive=true

# Or restart the rclone mount service
sudo systemctl restart rclone-gdrive
```

For mounts with the RC API enabled, add `--rc --rc-addr localhost:5572` to the mount command. Each mount should use a different port.

## Blomp/Swift: Chunking Large Files

Blomp uses OpenStack Swift, which has a 5GB maximum object size. The rclone chunker remote handles this transparently:

- Files under 5GB are stored as-is
- Files over 5GB are split into 5GB chunks with metadata
- rclone reassembles them on read

This is transparent to MergerFS and Plex. They see normal files.

The chunker adds some overhead:
- Listing directories is slightly slower (metadata lookups)
- Large file reads have marginally higher latency (reassembly)
- Shorter VFS cache age (30m vs 1h) compensates for the added complexity

## Monitoring Disk Usage

Local disk can fill up fast if cloud sync falls behind or downloads spike. Set up alerts:

```bash
#!/usr/bin/env bash
# /local/storage/scripts/check-disk-usage.sh

THRESHOLD=85
USAGE=$(df /local --output=pcent | tail -1 | tr -d ' %')

if [ "${USAGE}" -gt "${THRESHOLD}" ]; then
    echo "ALERT: /local disk usage at ${USAGE}% (threshold: ${THRESHOLD}%)"
    # Send notification via your preferred method:
    # - Alertmanager webhook
    # - Email
    # - Discord/Slack webhook
fi
```

Run via cron every hour:

```bash
0 * * * * /local/storage/scripts/check-disk-usage.sh
```

## Cleaning Up Local Copies

Once a file is confirmed in cloud storage, the local copy can be removed. But be careful:

1. **Verify the cloud copy exists:** `rclone lsf gdrive-crypt:"TV Shows/Show Name/Season 01/episode.mkv"`
2. **Verify it's accessible via the mount:** `ls /local/storage/mounts/gdrive/TV Shows/Show Name/Season 01/episode.mkv`
3. **Only then** remove the local copy

A safe cleanup script:

```bash
#!/usr/bin/env bash
# For each file in local media, check if it exists in cloud, then remove local copy

RCLONE_CONFIG="/local/storage/rclone.conf"
LOCAL="/local/storage/mounts/local"
CLOUD="gdrive-crypt:"

find "${LOCAL}" -type f -name "*.mkv" -o -name "*.mp4" | while read -r file; do
    relative="${file#${LOCAL}/}"
    if rclone lsf --config "${RCLONE_CONFIG}" "${CLOUD}${relative}" > /dev/null 2>&1; then
        echo "Cloud copy confirmed, removing local: ${relative}"
        rm "${file}"
    else
        echo "No cloud copy yet, keeping local: ${relative}"
    fi
done
```

**Never run this automatically without testing.** Run it manually a few times first, review the output, then consider automating it.

## Next Steps

See [08 - Networking & Security](08-networking-and-security.md) for locking down the server and configuring zero-trust access.

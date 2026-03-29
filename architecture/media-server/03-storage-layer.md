# 03 - Storage Layer

This is the most important chapter in the guide. The storage layer is what makes a media server with terabytes of content actually work on a server with only ~640GB of local disk.

## The Problem

You have more media than fits on local storage. Cloud storage is cheap and virtually unlimited, but it's slow and has API rate limits. You need a system that:

1. Presents one unified filesystem to Plex
2. Writes new files locally (fast)
3. Reads existing files from cloud (cheap)
4. Encrypts everything at rest in the cloud

The solution: **rclone FUSE mounts** for each cloud backend + **MergerFS** to combine them into a single path.

## Why Encrypted Cloud Storage

- **Privacy:** Your cloud provider cannot see filenames or content
- **Portability:** If a provider shuts down or changes terms, your encrypted data is meaningless to them
- **Redundancy:** Spread across multiple providers so no single failure loses everything
- **Cost:** Cloud storage is dramatically cheaper than adding physical drives to a dedicated server

## rclone Remote Configuration

rclone uses a layered remote model. You define a base remote (the cloud provider), then wrap it in encryption:

```
provider remote  →  crypt remote  →  FUSE mount
   (plain)           (encrypted)     (local path)
```

For providers with object size limits (like OpenStack Swift), add a chunker layer:

```
provider remote  →  chunker remote  →  crypt remote  →  FUSE mount
   (plain)           (5GB chunks)      (encrypted)     (local path)
```

### Example: Google Drive (Encrypted)

```ini
[gdrive]
type = drive
scope = drive
team_drive = YOUR_TEAM_DRIVE_ID
token = {"access_token":"...","token_type":"Bearer","refresh_token":"...","expiry":"..."}

[gdrive-crypt]
type = crypt
remote = gdrive:media
password = YOUR_ENCRYPTED_PASSWORD
password2 = YOUR_ENCRYPTED_SALT
```

### Example: Backblaze B2 (Encrypted)

```ini
[backblaze]
type = b2
account = YOUR_B2_ACCOUNT_ID
key = YOUR_B2_APP_KEY

[backblaze-crypt]
type = crypt
remote = backblaze:your-bucket-name
password = YOUR_ENCRYPTED_PASSWORD
password2 = YOUR_ENCRYPTED_SALT
```

### Example: Swift/Blomp (Chunked + Encrypted)

Some providers (Blomp uses OpenStack Swift) have a 5GB max object size. The chunker layer splits large files:

```ini
[blomp-swift]
type = swift
user = your-email@example.com
key = YOUR_SWIFT_KEY
auth = https://authenticate.blomp.com/v3
tenant = your-email@example.com
auth_version = 3

[blomp-chunker]
type = chunker
remote = blomp-swift:media
chunk_size = 5G
hash_type = none

[blomp-crypt]
type = crypt
remote = blomp-chunker:
password = YOUR_ENCRYPTED_PASSWORD
password2 = YOUR_ENCRYPTED_SALT
```

## rclone FUSE Mounts as systemd Services

Each cloud backend gets its own systemd service. This ensures mounts come up at boot and can be managed independently.

See [`config-examples/rclone-mount.service.example`](config-examples/rclone-mount.service.example) for the full service file.

Key mount options explained:

```bash
rclone mount gdrive-crypt: /local/storage/mounts/gdrive \
    --config /local/storage/rclone.conf \
    --allow-other \                    # Let other users (Plex, Docker) access
    --dir-cache-time 72h \             # Cache directory listings for 72 hours
    --vfs-cache-mode full \            # Full read/write caching
    --vfs-cache-max-size 80G \         # Cap cache at 80GB
    --vfs-cache-max-age 1h \           # Evict cached files after 1 hour
    --vfs-read-ahead 128M \            # Read-ahead buffer for streaming
    --buffer-size 64M \                # Transfer buffer
    --poll-interval 15s \              # Check for remote changes
    --log-file /local/storage/logs/rclone-gdrive.log \
    --log-level INFO
```

### VFS Cache Tuning

Different backends deserve different cache sizes based on how heavily they're accessed:

| Backend | Cache Size | Cache Age | Rationale |
|---------|-----------|-----------|-----------|
| Google Drive | 80G | 1h | Primary backend, most content, most access |
| Backblaze B2 | 10G | 1h | Secondary backend, less frequent access |
| Blomp/Swift | 10G | 30m | Tertiary, chunker adds latency so shorter cache |

The VFS cache lives on local disk. The total cache across all backends must fit within your available local storage, leaving room for staging and local media.

## MergerFS: Unified Filesystem

MergerFS takes multiple mount points and presents them as one directory. Plex sees a single library path, but the files are actually spread across local storage and multiple cloud backends.

### How It Works

```
/local/storage/mounts/local      (RW)   ─┐
/local/storage/mounts/gdrive     (NC)    ├─→  /local/storage/plexdrive
/local/storage/mounts/backblaze  (NC)    │        (unified view)
/local/storage/mounts/blomp      (NC)   ─┘
```

- **RW** (read-write): Local storage. New files are written here.
- **NC** (no create): Cloud mounts. Existing files are readable, but new files are never created here by MergerFS.

### Configuration Explained

```bash
mergerfs \
    -o use_ino \                          # Use inode numbers from source
    -o allow_other \                      # Allow Docker/Plex access
    -o func.getattr=newest \              # If file exists in multiple branches, use newest
    -o category.create=ff \               # Create on First Found (local, since it's listed first)
    -o cache.files=auto-full \            # Kernel page cache for open files
    -o cache.readdir=true \               # Cache directory reads
    -o cache.attr=30 \                    # Cache file attributes for 30 seconds
    -o cache.entry=30 \                   # Cache directory entries for 30 seconds
    -o dropcacheonclose=true \            # Drop cache when file is closed
    /local/storage/mounts/backblaze=NC:/local/storage/mounts/gdrive=NC:/local/storage/mounts/blomp=NC:/local/storage/mounts/local=RW \
    /local/storage/plexdrive
```

**`category.create=ff`** is critical. It means "first found"... new files go to the first branch that has space. Since `local` is the only RW branch, all writes go to local storage. Cloud branches are read-only from MergerFS's perspective.

See [`config-examples/mergerfs.service.example`](config-examples/mergerfs.service.example) for the full service file.

## systemd Service Chain

The boot order matters. rclone mounts must be up before MergerFS starts, and MergerFS must be up before Docker containers that depend on it.

```
network-online.target
  ├── rclone-gdrive.service        → /local/storage/mounts/gdrive
  ├── rclone-backblaze.service     → /local/storage/mounts/backblaze
  ├── rclone-blomp.service         → /local/storage/mounts/blomp
  └── prepare-for-mergerfs.service → (sleeps 10s, then signals ready)
         └── mergerfs.service      → /local/storage/plexdrive
                └── plexdrive-shared.service → mount --make-rshared
```

### Why the delay service?

rclone mounts take a few seconds to initialize their VFS caches and connect to the remote. If MergerFS starts before they're ready, it sees empty mount points and Plex gets confused. The `prepare-for-mergerfs` service is a simple `ExecStart=/bin/sleep 10` with `Type=oneshot` and `RemainAfterExit=yes`. It waits for all rclone services, then signals that MergerFS can start.

### Why `mount --make-rshared`?

Docker containers use bind mounts to access `/local/storage/plexdrive`. By default, FUSE mounts are not visible inside Docker bind mounts. `mount --make-rshared /local/storage/plexdrive` propagates the mount into container namespaces.

```ini
# plexdrive-shared.service
[Unit]
Description=Make plexdrive mount shared for Docker
After=mergerfs.service
Requires=mergerfs.service

[Service]
Type=oneshot
ExecStart=/bin/mount --make-rshared /local/storage/plexdrive
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

## Gotcha: fusermount on Restart

If MergerFS or an rclone mount crashes and you restart the service, the old FUSE mount may still be registered. The service should clean up before starting:

```ini
ExecStartPre=-/bin/fusermount -uz /local/storage/mounts/gdrive
```

The `-` prefix before the command tells systemd "don't fail if this command fails." Without it, the service won't start if there's nothing to unmount.

## Next Steps

With storage configured, move on to [04 - Docker Services](04-docker-services.md) to set up the containerized media stack.

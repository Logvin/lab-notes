# 10 - Disaster Recovery

Your server will die. A drive will fail, a provider will have an outage, or you'll fat-finger a command. The question is how fast you can rebuild.

## What to Back Up

Not everything needs backing up. Media files live in the cloud. What you need are the configs and scripts that make everything work together.

### Critical (back up off-server)

| Item | Location | Why |
|------|----------|-----|
| `rclone.conf` | `/local/storage/rclone.conf` | Contains all remote definitions, encryption keys. Without this, your cloud media is unrecoverable. |
| Docker compose files | Various `/data/compose/` dirs | Service definitions, environment variables |
| SWAG/NGINX configs | `/local/storage/swag_config/nginx/` | Proxy rules, security headers, Access validation |
| systemd service files | `/etc/systemd/system/rclone-*.service`, `mergerfs.service` | Mount configuration, boot chain |
| Automation scripts | `/local/storage/scripts/` | Cloud sync, cleanup, monitoring |
| Prometheus configs | `/local/storage/prometheus_config/` | Scrape targets, alert rules |
| Alertmanager config | `/local/storage/alertmanager_config/` | Notification routing |
| SSH keys | `~/.ssh/` | Access to seedbox, other servers |
| Cron jobs | `crontab -l` output | Scheduled tasks |
| UFW rules | `sudo ufw status numbered` output | Firewall configuration |

### Important (nice to have)

| Item | Location | Why |
|------|----------|-----|
| Sonarr/Radarr databases | `*_config/` dirs | Watch history, quality profiles, custom formats. Rebuilding is tedious but possible. |
| Tautulli database | `/local/storage/tautulli_config/` | Watch history. Sentimental but not critical. |
| Plex metadata | `/local/storage/plex_config/` | Watch status, custom metadata. Large but recoverable from a fresh scan. |

### Not Needed

| Item | Why Skip |
|------|----------|
| Media files | They're in cloud storage. Re-mount and they're back. |
| Docker images | `docker pull` rebuilds them. |
| OS packages | Reinstall from package manager. |
| rclone VFS cache | Rebuilt automatically on mount. |

## Backup Strategy

### rclone.conf (Most Critical)

This file contains your encryption passwords. If you lose it, your cloud media is gone forever. The files are encrypted, and without the keys, they're random noise.

```bash
# Encrypt and back up rclone.conf
gpg --symmetric --cipher-algo AES256 /local/storage/rclone.conf

# Store the encrypted file:
# 1. Local backup drive
# 2. Password manager (as a secure note or attachment)
# 3. A separate cloud storage account (not one of your media backends)
# 4. Print the encryption passwords and store in a safe (seriously)
```

### Automated Config Backup

```bash
#!/usr/bin/env bash
# /local/storage/scripts/backup-configs.sh
#
# Backs up all critical configs to a tarball.
# Run weekly via cron, store off-server.

set -euo pipefail

BACKUP_DIR="/local/storage/backups"
DATE=$(date +%Y-%m-%d)
BACKUP_FILE="${BACKUP_DIR}/config-backup-${DATE}.tar.gz"

mkdir -p "${BACKUP_DIR}"

tar czf "${BACKUP_FILE}" \
    /local/storage/rclone.conf \
    /local/storage/scripts/ \
    /local/storage/swag_config/nginx/ \
    /local/storage/prometheus_config/ \
    /local/storage/alertmanager_config/ \
    /etc/systemd/system/rclone-*.service \
    /etc/systemd/system/mergerfs.service \
    /etc/systemd/system/plexdrive-shared.service \
    /etc/systemd/system/prepare-for-mergerfs.service \
    /etc/rsyncd.conf \
    2>/dev/null

echo "Backup created: ${BACKUP_FILE}"
echo "Size: $(du -h "${BACKUP_FILE}" | cut -f1)"

# Keep only last 4 backups locally
ls -t "${BACKUP_DIR}"/config-backup-*.tar.gz | tail -n +5 | xargs -r rm

# Upload to a separate backup location
# rclone copy "${BACKUP_FILE}" backup-remote:server-configs/
```

## Rebuild Order

Order matters. Each layer depends on the one before it.

### Phase 1: Base System (30 minutes)

1. Provision new server (Ubuntu 24.04 LTS)
2. Create users (admin + mediauser)
3. Install packages: docker, rclone, fuse3, mergerfs, rsync
4. Configure SSH (keys, disable password auth)
5. Set up UFW firewall rules

### Phase 2: Storage Layer (1 hour)

1. Create directory structure: `/local/storage/{mounts,staging,cache,logs,scripts}`
2. Restore `rclone.conf` from backup
3. Install rclone systemd services
4. Start rclone mounts, verify each one connects
5. Install MergerFS systemd service chain
6. Start MergerFS, verify unified mount
7. Set up `mount --make-rshared`

**Test:** `ls /local/storage/plexdrive/` should show your media library.

### Phase 3: Docker Services (1 hour)

1. Restore docker-compose files
2. Start SWAG first (needs to be up for SSL/proxy)
3. Start arr stack (Sonarr, Radarr, Prowlarr, Overseerr)
4. Start Plex
5. Start monitoring stack (Prometheus, Grafana, Alertmanager)
6. Start Portainer

**Test:** Access `https://media.example.com/apps/sonarr` through Cloudflare Access.

### Phase 4: Automation (30 minutes)

1. Restore automation scripts
2. Set up cron jobs
3. Restore rsync daemon config
4. Verify seedbox can rsync to staging
5. Start SSH tunnel to seedbox (if needed)

**Test:** Trigger a manual rsync from the seedbox.

### Phase 5: Verification (30 minutes)

1. Play a movie on Plex (test cloud mount → MergerFS → Plex)
2. Request something in Overseerr (test full pipeline)
3. Check Grafana dashboards (test monitoring)
4. Verify Cloudflare Access is protecting all `/apps/*` paths
5. Run a port scan against your server (verify UFW)

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| MergerFS starts before rclone mounts | Empty media library in Plex | Check systemd service chain, add delay service |
| Docker can't see FUSE mounts | "No such file or directory" in containers | Run `mount --make-rshared` on the MergerFS path |
| rclone mount fails after crash | "Transport endpoint is not connected" | `fusermount -uz /path` then restart the service |
| Wrong file permissions | Sonarr/Radarr can't import | Check PUID/PGID match mediauser, check group membership |
| SWAG can't get SSL cert | 403 from Let's Encrypt | Verify Cloudflare API token has DNS edit permissions |
| Cloudflare Access not enforced | Apps accessible without auth | Check Access policy covers `/apps/*`, verify JWT validation |
| Seedbox rsync rejected | "Connection refused" on port 873 | Check UFW allows seedbox IP, check rsync daemon is running |

## Estimated Recovery Time

| Scenario | Recovery Time |
|----------|--------------|
| Single container crash | 1 minute (auto-restart) |
| rclone mount drop | 5 minutes (restart service) |
| MergerFS failure | 10 minutes (restart chain) |
| Full server rebuild from backup | 3-4 hours |
| Full server rebuild from scratch | 8-12 hours |
| Lost rclone.conf with no backup | **Unrecoverable** (cloud media lost) |

The last one is why backing up `rclone.conf` is the single most important thing in this entire guide.

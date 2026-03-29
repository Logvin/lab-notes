# 04 - Docker Services

Every application runs in a Docker container. No exceptions. This gives you isolation, reproducibility, and the ability to blow away a broken service without affecting anything else.

## Why Docker for Everything

- **Isolation:** Each service has its own filesystem, dependencies, and network namespace
- **Portability:** `docker compose up` on a new server and you're running
- **Updates:** Pull a new image, recreate the container, done
- **Rollback:** Keep the old image, roll back in seconds
- **Resource management:** CPU and memory limits per container

## SWAG: The Front Door

SWAG (Secure Web Application Gateway) from linuxserver.io is an NGINX reverse proxy bundled with Let's Encrypt SSL. It's the only container that listens on public ports (80 and 443).

```yaml
swag:
  image: lscr.io/linuxserver/swag:latest
  container_name: swag
  environment:
    - PUID=1000          # mediauser UID
    - PGID=1000          # mediagroup GID
    - TZ=UTC
    - URL=media.example.com
    - VALIDATION=dns
    - DNSPLUGIN=cloudflare
    - ONLY_SUBDOMAINS=false
  volumes:
    - /local/storage/swag_config:/config
  ports:
    - "80:80"
    - "443:443"
  restart: unless-stopped
```

SWAG handles:
- SSL certificate provisioning and renewal (Let's Encrypt via Cloudflare DNS validation)
- Reverse proxying to internal services
- Cloudflare Access JWT token validation
- Rate limiting and security headers

## Subfolder Routing

Every service gets a subfolder path under the main domain. No subdomains.

| Service | Path | Internal Target |
|---------|------|-----------------|
| Sonarr | `/apps/sonarr` | `127.0.0.1:8989` |
| Radarr | `/apps/radarr` | `127.0.0.1:7878` |
| Prowlarr | `/apps/prowlarr` | `127.0.0.1:9696` |
| Tautulli | `/apps/tautulli` | `127.0.0.1:8181` |
| Overseerr | `/apps/overseerr` | `127.0.0.1:5055` |
| Portainer | `/apps/portainer` | `127.0.0.1:9443` |

Each application must be configured with a URL base matching its subfolder path. In Sonarr, for example, set `Settings → General → URL Base` to `/apps/sonarr`.

See [`config-examples/swag-subfolder.conf.example`](config-examples/swag-subfolder.conf.example) for the NGINX config.

### Why Not Subdomains?

- One SSL certificate covers `media.example.com` (no wildcard needed)
- One Cloudflare Access policy covers all `/apps/*` paths
- One DNS A record instead of many
- Simpler mental model: everything is at one domain
- No CORS headaches between services

## The arr Stack

### Sonarr (TV Shows)

```yaml
sonarr:
  image: lscr.io/linuxserver/sonarr:latest
  container_name: sonarr
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=UTC
  volumes:
    - /local/storage/sonarr_config:/config
    - /local/storage/plexdrive:/media         # MergerFS mount (read)
    - /local/storage/staging:/staging         # Download staging (write)
    - /local/storage/mounts/local:/local-media  # Local storage (write)
  ports:
    - "127.0.0.1:8989:8989"
  restart: unless-stopped
```

### Radarr (Movies)

```yaml
radarr:
  image: lscr.io/linuxserver/radarr:latest
  container_name: radarr
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=UTC
  volumes:
    - /local/storage/radarr_config:/config
    - /local/storage/plexdrive:/media
    - /local/storage/staging:/staging
    - /local/storage/mounts/local:/local-media
  ports:
    - "127.0.0.1:7878:7878"
  restart: unless-stopped
```

### Prowlarr (Indexers)

```yaml
prowlarr:
  image: lscr.io/linuxserver/prowlarr:latest
  container_name: prowlarr
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=UTC
  volumes:
    - /local/storage/prowlarr_config:/config
  ports:
    - "127.0.0.1:9696:9696"
  restart: unless-stopped
```

### Overseerr (Requests)

```yaml
overseerr:
  image: sctx/overseerr:latest
  container_name: overseerr
  environment:
    - TZ=UTC
  volumes:
    - /local/storage/overseerr_config:/app/config
  ports:
    - "127.0.0.1:5055:5055"
  restart: unless-stopped
```

## Plex Media Server

Plex uses host networking for discovery and DLNA. It's the one container that doesn't bind to 127.0.0.1.

```yaml
plex:
  image: lscr.io/linuxserver/plex:latest
  container_name: plex
  network_mode: host
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=UTC
    - VERSION=docker
  volumes:
    - /local/storage/plex_config:/config
    - /local/storage/plexdrive:/media:ro    # MergerFS mount (read-only)
  restart: unless-stopped
```

Plex's libraries point to subdirectories within `/media` (which is the MergerFS mount). For example:
- `/media/TV Shows`
- `/media/Movies`
- `/media/Anime`

## Monitoring Stack

### Prometheus + Grafana + Alertmanager

```yaml
prometheus:
  image: prom/prometheus:latest
  container_name: prometheus
  volumes:
    - /local/storage/prometheus_config:/etc/prometheus
    - prometheus_data:/prometheus
  ports:
    - "127.0.0.1:9090:9090"
  restart: unless-stopped

grafana:
  image: grafana/grafana:latest
  container_name: grafana
  environment:
    - GF_SERVER_ROOT_URL=https://media.example.com/apps/grafana
    - GF_SERVER_SERVE_FROM_SUB_PATH=true
  volumes:
    - grafana_data:/var/lib/grafana
  ports:
    - "127.0.0.1:3000:3000"
  restart: unless-stopped

alertmanager:
  image: prom/alertmanager:latest
  container_name: alertmanager
  volumes:
    - /local/storage/alertmanager_config:/etc/alertmanager
  ports:
    - "127.0.0.1:9093:9093"
  restart: unless-stopped

node-exporter:
  image: prom/node-exporter:latest
  container_name: node-exporter
  volumes:
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
    - /:/rootfs:ro
  command:
    - '--path.procfs=/host/proc'
    - '--path.sysfs=/host/sys'
    - '--path.rootfs=/rootfs'
  ports:
    - "127.0.0.1:9100:9100"
  restart: unless-stopped

cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  container_name: cadvisor
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker:/var/lib/docker:ro
  ports:
    - "127.0.0.1:8080:8080"
  restart: unless-stopped
```

### Tautulli (Plex Analytics)

```yaml
tautulli:
  image: ghcr.io/tautulli/tautulli:latest
  container_name: tautulli
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=UTC
  volumes:
    - /local/storage/tautulli_config:/config
  ports:
    - "127.0.0.1:8181:8181"
  restart: unless-stopped
```

## Portainer (Docker Management)

```yaml
portainer:
  image: portainer/portainer-ce:latest
  container_name: portainer
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - portainer_data:/data
  ports:
    - "127.0.0.1:9443:9443"
  restart: unless-stopped
```

Portainer gives you a web UI for managing containers, viewing logs, and deploying compose stacks. It's entirely optional but very convenient for quick troubleshooting.

## Network Architecture

All containers bind to `127.0.0.1` except Plex (host networking) and SWAG (public-facing). This means:

- No container is directly accessible from the internet
- All traffic flows through SWAG
- SWAG proxies to localhost ports
- Inter-container communication happens over localhost

For containers that need to talk to each other (Sonarr → Prowlarr, for example), they can use `http://127.0.0.1:PORT` or Docker's internal DNS if on the same bridge network.

## Critical: Bind Mounts and FUSE

Docker containers accessing MergerFS paths need the mount to be shared. Without `mount --make-rshared`, the FUSE mount is invisible inside the container.

Additionally, use `:shared` on the bind mount in Docker Compose for MergerFS paths:

```yaml
volumes:
  - /local/storage/plexdrive:/media:shared
```

Or use `:ro` (read-only) for containers that should never write to the media library.

See [`config-examples/docker-compose.example.yml`](config-examples/docker-compose.example.yml) for the complete compose file.

## Next Steps

With services running, move on to [05 - Media Pipeline](05-media-pipeline.md) to understand the automated acquisition flow.

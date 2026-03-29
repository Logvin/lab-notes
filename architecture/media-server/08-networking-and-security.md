# 08 - Networking & Security

Security is layers. No single control is sufficient, but stacked together they create a setup that's very difficult to compromise.

## Architecture Overview

```
Internet
    │
    ▼
Cloudflare (CDN + WAF + DDoS protection)
    │
    ▼
Cloudflare Access (zero-trust authentication)
    │
    ▼
UFW Firewall (only allows Cloudflare IPs on 80/443)
    │
    ▼
SWAG/NGINX (reverse proxy + JWT validation)
    │
    ▼
Docker containers (bound to 127.0.0.1)
```

Every layer blocks a different class of attack. An attacker needs to bypass all of them.

## UFW Configuration

Start with deny-all, then whitelist only what's needed:

```bash
# Reset to defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH (restrict to your IP if possible)
sudo ufw allow 22/tcp

# Plex (direct connection for clients)
sudo ufw allow 32400/tcp

# HTTP/HTTPS - ONLY from Cloudflare IP ranges
# IPv4 ranges (check https://www.cloudflare.com/ips-v4 for current list)
sudo ufw allow from 173.245.48.0/20 to any port 80,443 proto tcp
sudo ufw allow from 103.21.244.0/22 to any port 80,443 proto tcp
sudo ufw allow from 103.22.200.0/22 to any port 80,443 proto tcp
sudo ufw allow from 103.31.4.0/22 to any port 80,443 proto tcp
sudo ufw allow from 141.101.64.0/18 to any port 80,443 proto tcp
sudo ufw allow from 108.162.192.0/18 to any port 80,443 proto tcp
sudo ufw allow from 190.93.240.0/20 to any port 80,443 proto tcp
sudo ufw allow from 188.114.96.0/20 to any port 80,443 proto tcp
sudo ufw allow from 197.234.240.0/22 to any port 80,443 proto tcp
sudo ufw allow from 198.41.128.0/17 to any port 80,443 proto tcp
sudo ufw allow from 162.158.0.0/15 to any port 80,443 proto tcp
sudo ufw allow from 104.16.0.0/13 to any port 80,443 proto tcp
sudo ufw allow from 104.24.0.0/14 to any port 80,443 proto tcp
sudo ufw allow from 172.64.0.0/13 to any port 80,443 proto tcp
sudo ufw allow from 131.0.72.0/22 to any port 80,443 proto tcp

# rsync - ONLY from seedbox IP
sudo ufw allow from 192.0.2.20 to any port 873 proto tcp

# Enable the firewall
sudo ufw enable
```

**Why restrict HTTP/HTTPS to Cloudflare IPs?**

If someone discovers your server's real IP, they could bypass Cloudflare entirely... hitting SWAG directly, skipping Access authentication. By only allowing Cloudflare's IP ranges, direct connections are rejected at the firewall level.

## Cloudflare Access: Zero-Trust Gateway

Cloudflare Access replaces traditional app-level authentication. Users authenticate through Cloudflare before their request reaches your server.

### How It Works

1. User navigates to `media.example.com/apps/sonarr`
2. Cloudflare Access intercepts the request
3. User authenticates (email OTP, Google, GitHub, etc.)
4. Cloudflare issues a signed JWT cookie (`CF_Authorization`)
5. Request continues to your server with the JWT attached
6. SWAG validates the JWT before proxying to the app

### Access Policy Setup

Create an Access Application in the Cloudflare dashboard:

| Setting | Value |
|---------|-------|
| Application name | Media Server Apps |
| Session duration | 24 hours |
| Application domain | `media.example.com` |
| Path | `/apps/*` |

Policy:

| Rule | Action | Criteria |
|------|--------|----------|
| Allow | Allow | Email ends in `@example.com` |
| Allow | Allow | Specific email: `trusted-friend@gmail.com` |
| Block | Block | Everyone else |

This means only pre-approved email addresses can access any `/apps/*` path. No passwords to manage, no exposed login forms.

## SWAG/NGINX JWT Validation

Even though Cloudflare Access handles authentication, SWAG should validate the JWT token as a defense-in-depth measure. If someone bypasses Cloudflare (unlikely but possible), this layer stops them.

Add to your NGINX server block:

```nginx
# /config/nginx/site-confs/default.conf

# Cloudflare Access JWT validation
# Fetches the public key from your Access team domain
set $cf_access_team "your-team-name";

# Validate CF Access JWT on all /apps/ paths
location /apps/ {
    # Verify the CF_Authorization cookie contains a valid JWT
    # signed by your Cloudflare Access team
    access_by_lua_block {
        local jwt = require "resty.jwt"
        local validators = require "resty.jwt-validators"

        local token = ngx.var.cookie_CF_Authorization
        if not token then
            ngx.status = 403
            ngx.say("Access denied: no token")
            return ngx.exit(403)
        end

        -- Validate against Cloudflare's public keys
        -- (In practice, use the cf-access-auth module or similar)
    }

    # Proxy to internal services
    include /config/nginx/proxy.conf;
}
```

In practice, linuxserver's SWAG image includes Cloudflare Access authentication support. Check their documentation for the simplest integration path.

## Docker Container Isolation

Every container binds to `127.0.0.1`, not `0.0.0.0`:

```yaml
ports:
  - "127.0.0.1:8989:8989"    # Only accessible from localhost
  # NOT:
  # - "8989:8989"            # This exposes to ALL interfaces
```

This means:
- No container is directly reachable from the internet
- All access goes through SWAG
- Even if UFW is misconfigured, containers aren't exposed

Exceptions:
- **SWAG:** Binds to `0.0.0.0:80` and `0.0.0.0:443` (must be publicly reachable for Cloudflare)
- **Plex:** Uses host networking for DLNA/discovery (but Plex has its own auth)

## rsync Restrictions

The rsync daemon only accepts connections from the seedbox:

```ini
# /etc/rsyncd.conf
[staging]
    path = /local/storage/staging
    hosts allow = 192.0.2.20
    hosts deny = *
    read only = no
    uid = mediauser
    gid = mediagroup
```

Combined with UFW allowing port 873 only from the seedbox IP, this is double-locked.

## No Open Management Ports

Services like Portainer, Sonarr, Radarr, etc. are never exposed directly:

- They bind to `127.0.0.1`
- They're only accessible through SWAG reverse proxy
- SWAG routes are protected by Cloudflare Access
- No SSH tunneling needed for daily management

This means you manage everything through the web browser, authenticated via Cloudflare Access. No need to SSH in for routine tasks.

## Additional Hardening

### Fail2ban

Protects SSH from brute-force attempts:

```bash
sudo apt install fail2ban

# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 22
maxretry = 5
bantime = 3600
findtime = 600
```

### Automatic Security Updates

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

This auto-installs security patches. Critical for a 24/7 server.

### Docker Socket Protection

The Docker socket (`/var/run/docker.sock`) is root-equivalent access. Only Portainer should have it mounted, and Portainer is behind Cloudflare Access.

Never expose the Docker socket to the network. Never mount it into containers that don't absolutely need it.

## Security Checklist

Before going live:

- [ ] SSH: key-only, no root login, fail2ban enabled
- [ ] UFW: default deny, only Cloudflare IPs on 80/443
- [ ] Cloudflare: proxied DNS (orange cloud), Access policy on `/apps/*`
- [ ] SWAG: JWT validation, security headers, rate limiting
- [ ] Docker: all containers on 127.0.0.1 (except SWAG and Plex)
- [ ] rsync: restricted to seedbox IP only
- [ ] No exposed management ports
- [ ] Automatic security updates enabled
- [ ] No hardcoded secrets in configs (use environment variables)

## Next Steps

See [09 - Monitoring](09-monitoring.md) for setting up observability across the stack.

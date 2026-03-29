# 02 - Hardware & OS Setup

## Choosing a Provider

You want a dedicated server, not a VPS. The difference matters for media workloads.

| Provider | Pros | Cons |
|----------|------|------|
| **OVH/OVHcloud** | Good price/performance, data centers worldwide, unmetered bandwidth | Support can be slow, stock availability varies |
| **Hetzner** | Excellent value (especially auction servers), great network | EU-only data centers, some US latency |
| **SoYouStart** | OVH's budget brand, same hardware | Fewer config options |
| **Netcup** | Very affordable | Smaller company, less availability |

**Recommended specs:**

| Spec | Minimum | Recommended |
|------|---------|-------------|
| CPU | 8 cores (Xeon E-2200 series) | 16 cores (Xeon Silver or Ryzen) |
| RAM | 32GB ECC | 64GB ECC |
| Storage | 2x 500GB NVMe | 2x 1TB NVMe |
| Bandwidth | 1Gbps unmetered | Same |
| RAID | Software RAID1 | Same |

ECC RAM matters for a server that runs 24/7. Non-ECC works, but bit flips over months of uptime are real.

## Ubuntu 24.04 LTS Setup

Use the latest Ubuntu LTS. Most providers offer it as a one-click install option.

After initial provisioning:

```bash
# Update everything
sudo apt update && sudo apt upgrade -y

# Install essentials
sudo apt install -y \
    curl wget git htop tmux \
    ufw fail2ban \
    fuse3 mergerfs \
    rsync \
    unzip p7zip-full

# Set timezone
sudo timedatectl set-timezone UTC

# Set hostname
sudo hostnamectl set-hostname media-server
```

## RAID1 Configuration

Most providers configure RAID during provisioning. If you need to do it yourself:

Your provider will typically set up software RAID (mdadm) with two NVMe drives mirrored. The key partitions:

| Partition | RAID | Size | Mount | Purpose |
|-----------|------|------|-------|---------|
| md2 | RAID1 | 1G | /boot | Boot partition |
| md3 | RAID1 | 250G | / | Root filesystem |
| md5 | RAID1 | ~640G | /local | Data partition |

The data partition is where everything lives. 250G for root is generous... you could go smaller, but storage is cheap and running out of root space is painful.

```bash
# Verify RAID status
cat /proc/mdstat

# Should show all arrays as [UU] (both drives active)
# md5 : active raid1 nvme1n1p5[1] nvme0n1p5[0]
#       674201600 blocks super 1.2 [2/2] [UU]
```

## Partition Strategy

Keep root small and predictable. Put all data on a separate mount:

```
/           250G   OS, packages, Docker images
/boot         1G   Kernel, initramfs
/local      640G   All media data, configs, caches, scripts
```

Create the main data directory structure:

```bash
sudo mkdir -p /local/storage/{mounts,staging,cache,logs,scripts}
sudo mkdir -p /local/storage/mounts/{local,gdrive,backblaze,blomp}
```

## User Setup

Two users: one for administration, one for running media services.

```bash
# Create the media service user
sudo useradd -r -m -s /bin/bash mediauser

# Create a shared group for media files
sudo groupadd mediagroup

# Add both users to the shared group
sudo usermod -aG mediagroup mediauser
sudo usermod -aG mediagroup youradmin

# Add your admin user to the docker group
sudo usermod -aG docker youradmin

# Set ownership of the data directory
sudo chown -R mediauser:mediagroup /local/storage
sudo chmod -R 775 /local/storage
```

Why two users:
- Your admin user has sudo and SSH access. If it gets compromised, you want the blast radius to be containable.
- The media service user (mediauser) runs rclone mounts and handles media files. It has no sudo, no SSH access, no elevated privileges.
- Both share a group so file permissions work across services.

## SSH Hardening

Edit `/etc/ssh/sshd_config`:

```
# Disable password authentication (key-only)
PasswordAuthentication no
PubkeyAuthentication yes

# Disable root login
PermitRootLogin no

# Limit to your admin user
AllowUsers youradmin

# Change default port (optional but reduces noise)
# Port 2222

# Timeout idle sessions
ClientAliveInterval 300
ClientAliveCountMax 2
```

```bash
# Restart SSH
sudo systemctl restart sshd
```

Before you disable password auth, make sure your SSH key is already installed and working. Locking yourself out of a remote server is not fun.

## Install Docker

```bash
# Add Docker's official GPG key and repository
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin

# Verify
docker --version
docker compose version
```

## Install rclone

```bash
# Install latest rclone
curl https://rclone.org/install.sh | sudo bash

# Verify
rclone version
```

rclone is installed system-wide but the config file lives in the data directory (`/local/storage/rclone.conf`), owned by the media service user.

## Next Steps

With the OS and base packages installed, move on to [03 - Storage Layer](03-storage-layer.md) to set up your cloud storage mounts and MergerFS.

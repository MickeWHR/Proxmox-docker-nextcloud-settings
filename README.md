# Proxmox-docker-nextcloud-settings
A description of how I deployed Nextcloud for future references.

This guide describes my take on an installation of **Nextcloud All-in-One (AIO)** inside a **Debian VM running on Proxmox**, using:
- **ZFS mirror on SSDs** for system, VM disks, and Docker performance "rpool"
- **ZFS mirror on HDDs** for Nextcloud user data and backups "nc"
- **VirtioFS** mounts between the Proxmox host and the VM for efficient, low-overhead file sharing

### ✅ Benefits
- Clear separation of system, Docker, and user data
- Easy dataset-level backup and snapshot control (ZFS)
- Simple recovery or migration via bind-mounts
- Minimal performance loss — SSD for containers, HDD for large data

### ⚠️ Warnings
- Nextcloud internal backup doesn't do external folders, have to solve it inside VM or preferaby through proxmox.
- Do **not** let `NEXTCLOUD_DATADIR` and `NEXTCLOUD_MOUNT` point to the same path — it breaks AIO confinement rules.
- Always test mounts before starting containers.
- Avoid snapshotting large datasets while AIO backups are running.

## 🧩 Proxmox Configuration
### 1. **Proxmox host (ZFS layout)**
| Purpose | ZFS Pool | Path | Notes |
|----------|-----------|------|------|
| System / VM disks | `rpool` (SSD mirror) | `/rpool/data` | Default Proxmox pool |
| Nextcloud user data | `nc` (HDD mirror) | `/nc/userdata` | Created via `zfs create nc/userdata` |
| Nextcloud backups | `nc` (HDD mirror) | `/nc/backup` | Created via `zfs create nc/backup` |

### 2. Create **Directory Mappings** in the Datacenter view  
This step makes your ZFS datasets available as shareable storage objects inside Proxmox.

In **Proxmox Web UI**:
1. Go to **Datacenter → Storage → Add → Directory**  
2. Fill in the following:

| Setting | Value |
|----------|--------|
| **ID** | `nc-userdata` |
| **Directory** | `/nc/userdata` |
| **Content** | *Unchecked (none needed)* |
| **Nodes** | *(select your host)* |
| **Enable** | ✅ |

Repeat for the backup directory:
| Setting | Value |
|----------|--------|
| **ID** | `nc-backup` |
| **Directory** | `/nc/backup` |

You should now see both `nc-userdata` and `nc-backup` listed under **Datacenter → Storage**.

## 🧩 VM configuration (Debian 12 / Proxmox 8.4)

### **VirtioFS mappings (Proxmox → VM)**
| Proxmox Path | Directory ID | VM Path | Purpose |
|---------------|--------------|----------|----------|
| `/nc/userdata` | `nc-userdata` | `/mnt/ncdata/vm-userdata` | Nextcloud user data |
| `/nc/backup` | `nc-backup` | `/mnt/ncdata/vm-backup` | Nextcloud AIO backup target |

### 1. Add VirtioFS shares to VM
In **Proxmox GUI → VM → Hardware → Add → VirtioFS**:
- **Directory ID:** `nc-userdata` → `/nc/userdata`
- **Directory ID:** `nc-backup` → `/nc/backup`

### 2. Mount them inside the VM
Edit `/etc/fstab` in the Debian VM and add to the end of file:
nc-userdata   /mnt/ncdata/vm-userdata  virtiofs  rw,nofail,noatime  0 0
nc-backup     /mnt/ncdata/vm-backup    virtiofs  rw,nofail,noatime  0 0

Run systemctl daemon-reload

## Startup of Nextcloud-AIO wih Docker run command
sudo docker run \
  --init \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart=always \
  --publish 80:80 \
  --publish 8080:8080 \
  --publish 8443:8443 \
  --env NEXTCLOUD_DATADIR=/mnt/ncdata/vm-userdata \
  --env NEXTCLOUD_MOUNT=/mnt/ \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  --volume /mnt/ncdata/vm-userdata:/mnt/d-userdata \
  --volume /mnt/ncdata/vm-backup:/mnt/d-backup \
  ghcr.io/nextcloud-releases/all-in-one:latest

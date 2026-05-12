# Disaster Recovery

This document describes how to recover the homelab in case of hardware failure or total loss.

## Recovery objectives

- Recovery time: hours to days acceptable. Goal is not zero downtime, but avoiding restart from scratch.
- Recovery point: up to 1 week of changes may be lost (weekly Proxmox backup schedule).

## Recovery scenarios

### Single drive failure (ZFS mirror)
ZFS mirror handles this automatically. Replace failed drive, run `zpool replace`. No service interruption.

### Proxmox host failure (disk/motherboard, hardware preserved)
Reinstall Proxmox, restore configs, restore LXC containers from backup on `enclosure-backup`.

### Total loss (theft, fire, flood)
Acquire new hardware (any x86_64 with sufficient RAM/storage), install Proxmox, restore from offsite backup.
Currently a gap: offsite backup not yet configured. See Known Gaps section.

## Backup inventory

| What | Where | Retention |
|------|-------|-----------|
| LXC containers (11) | `/enclosure/backup/dump/` on Proxmox host | 5 latest, weekly Sun 01:00 |
| Paperless documents | OneDrive `/paperless-backup/` | Daily 02:00, rclone sync |
| UniFi config | UniFi cloud account | Automatic |
| Secrets (vault pw, root pw, SSH keys) | Bitwarden "Homelab DR" folder | Manual |
| Code/configs | GitHub (homelab, homelab-ansible, homelab-kubernetes) | Push-driven |

## Critical credentials

All stored in Bitwarden under "Homelab DR":
- Ansible vault password (decrypts all secrets in repo)
- Proxmox root password
- Ansible SSH private key (manages all containers)

## Hardware requirements for rebuild

Minimum specs to run current workload:
- x86_64 CPU with virtualization extensions
- 16GB RAM (current usage allows for K8s VMs to be running)
- 2 storage drives for ZFS mirror (3TB each, or larger)
- 1 boot drive (NVMe or SSD, 256GB or larger)
- Gigabit network

Current hardware: ASUS PN52, AMD Ryzen 9 5900HX, 16GB RAM, 2 of 3TB HDD ZFS mirror.

## Recovery procedure

### Phase 1: Proxmox host

1. Install Proxmox VE from ISO (https://www.proxmox.com/en/downloads)
2. Set root password (use Bitwarden value if same hardware, else new)
3. Configure network to use 192.168.1.11/24, gateway 192.168.1.1
4. Import ZFS pool: `zpool import enclosure`
5. Recreate storage entries via Datacenter, Storage:
   - `lvm`: LVM on second NVMe, content: rootdir, images
   - `enclosure`: ZFS pool, content: rootdir, images
   - `enclosure-backup`: Directory `/enclosure/backup`, content: backup

### Phase 2: LXC containers

For each container in `/enclosure/backup/dump/`:

1. In Proxmox UI: select storage `enclosure-backup`, Backups
2. Select latest `vzdump-lxc-XXX-*.tar.zst`
3. Restore, choose target storage (lvm for OS disk)
4. After restore, verify mount points in `/etc/pve/lxc/XXX.conf` reference correct ZFS subvolumes

Recovery order (dependency-driven):
1. ansible (107), needed for re-configuration
2. paperless-ngx (100), primary service
3. prometheus (108), grafana (109), monitoring
4. github-runner (110), CI/CD
5. Media stack (101-106): sabnzbd, prowlarr, sonarr, radarr, jellyfin, bazarr

### Phase 3: Kubernetes VMs (optional, learning environment)

K8s VMs (800-802) are not currently backed up. If needed, rebuild via Ansible: ansible-playbook playbooks/kubernetes.yml

### Phase 4: SSH access restoration

From your workstation, restore the ansible_key from Bitwarden:
1. Copy private key content to `~/.ssh/ansible_key`
2. `chmod 600 ~/.ssh/ansible_key`
3. SSH to ansible LXC: `ssh -i ~/.ssh/ansible_key root@192.168.1.19`

### Phase 5: GitHub access

1. On ansible LXC, generate new SSH key: `ssh-keygen -t ed25519 -f ~/.ssh/github_key`
2. Add public key to GitHub account settings
3. Test: `ssh -T -i ~/.ssh/github_key git@github.com`

### Phase 6: Verification

| Service | URL | Verify |
|---------|-----|--------|
| paperless | http://192.168.1.12:8000 | Login, see documents |
| sabnzbd | http://192.168.1.13:7777 | Login, see queue |
| prowlarr | http://192.168.1.14:9696 | Login, see indexers |
| sonarr | http://192.168.1.15:8989 | Login, see series |
| radarr | http://192.168.1.16:7878 | Login, see movies |
| jellyfin | http://192.168.1.17:8096 | Login, libraries scan |
| bazarr | http://192.168.1.18:6767 | Login, see subtitles |
| prometheus | http://192.168.1.21:9090 | Targets all up |
| grafana | http://192.168.1.22:3000 | Dashboards load |

## Known gaps

- No offsite backup of LXC containers. All backups currently on the Proxmox host. Total-loss scenarios would require rebuilding from scratch (with code/configs from GitHub).
- K8s VMs not backed up. Learning environment, rebuild via Ansible takes about 15min.
- Proxmox host config not versioned. Network and storage configs documented here, but not automatically backed up.

## Maintenance

- Quarterly: verify a single LXC backup actually restores correctly (test on a different VMID).
- After significant changes: update this document.

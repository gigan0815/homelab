# LXC Inventory

Snapshot of the Proxmox LXC containers, captured 2026-10-23 from `pct list` and `/etc/pve/lxc/*.conf`. Re-capture after any container change. MAC addresses are omitted.

Common to all containers: unprivileged, `onboot: 1`, root disk on `lvm` (volume group `nvme-two`), single `eth0` on `vmbr0` using DHCP. The static IPs in the [README](../README.md) are expected to be UniFi DHCP reservations; this snapshot does not verify them.

## Containers

| VMID | Hostname | OS | Cores | RAM | Swap | Root disk | Bind mount (host → container) | Other settings |
|------|----------|----|-------|-----|------|-----------|-------------------------------|----------------|
| 100 | paperless-ngx | debian | 2 | 3072 MB | 512 MB | 64G | `/mnt/nas/paperless` → `/mnt/USBHDD` | `keyctl`, `nesting` |
| 101 | sabnzbd | debian | 2 | 2048 MB | 512 MB | 5G | `/mnt/nas/media` → `/mnt/USBHDD` | |
| 102 | prowlarr | debian | 2 | 1024 MB | 512 MB | 4G | `/mnt/nas/media` → `/mnt/USBHDD` | |
| 103 | sonarr | debian | 2 | 1024 MB | 512 MB | 4G | `/mnt/nas/media` → `/mnt/USBHDD` | `fuse=1`, `/dev/net/tun` passed through |
| 104 | radarr | debian | 2 | 1024 MB | 512 MB | 4G | `/mnt/nas/media` → `/mnt/USBHDD` | `fuse=1`, `/dev/net/tun` passed through |
| 105 | jellyfin | ubuntu | 2 | 2048 MB | 512 MB | 16G | `/mnt/nas/media` → `/mnt/USBHDD` | GPU passthrough: `/dev/dri/renderD128` (gid 993), `/dev/dri/card0` (gid 44); timezone `Europe/Berlin` |
| 106 | bazarr | debian | 2 | 1024 MB | 512 MB | 4G | `/mnt/nas/media` → `/mnt/USBHDD` | timezone `Europe/Berlin` |
| 107 | ansible | debian | 1 | 512 MB | 512 MB | 8G | none | firewall enabled |
| 108 | prometheus | debian | 1 | 1024 MB | 512 MB | 8G | none | firewall enabled |
| 109 | grafana | debian | 1 | 1024 MB | 512 MB | 8G | none | firewall enabled |
| 110 | github-runner | debian | 1 | 1024 MB | 512 MB | 8G | none | firewall enabled |

## Notes

- **`/mnt/USBHDD` is a legacy name.** It is the in-container mount point, not a USB disk. The source is an NFS share from TrueNAS, mounted on the Proxmox host under `/mnt/nas/`. The Ansible `paperless_backup` role and the DR procedure use this path inside the container.
- **Jellyfin GPU GIDs are host-specific.** `gid=993` and `gid=44` must match the groups on the restored host, or hardware transcoding breaks.
- **Bind mounts use `backup=0`.** Media and paperless data are excluded from vzdump. Paperless data is covered by its OneDrive export; media has no off-box copy.
- **VMID 111 does not exist.** `playbooks/unifi_controller.yml` targets it via `unifi_lxc_id`.
- **No VMs on the host.** `qm list` is empty, so the Kubernetes nodes and the Windows DC described in the README are not deployed here.

## Backup job

| Setting | Value |
|---------|-------|
| Schedule | `sun 01:00` |
| VMIDs | 100–110 |
| Mode / compression | `snapshot` / `zstd` |
| Storage | `nas-backups` (NFS `192.168.1.6:/mnt/enclosure/backups`) |
| Retention | `keep-last=4,keep-monthly=6` (same in `jobs.cfg` and `storage.cfg`) |

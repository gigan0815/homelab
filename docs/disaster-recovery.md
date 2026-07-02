# Disaster Recovery

This document describes how to recover the homelab in case of hardware failure or total loss.

## Recovery objectives

- Recovery time: hours to days acceptable. Goal is not zero downtime, but avoiding restart from scratch.
- Recovery point: up to 1 week of changes may be lost (weekly Proxmox backup schedule).

## Recovery scenarios

### Single drive failure (NAS ZFS mirror)
ZFS mirror handles this automatically. Replace failed drive in the TrueNAS box, run `zpool replace` (via TrueNAS UI: Storage → pool → Replace Disk, or CLI). No service interruption — Proxmox LXCs keep working since they mount over NFS and don't see the underlying disk change.

**Current status (as of 2026-07-02): one mirror member is already defective and awaiting replacement.** The drive with serial `WD-WCAWZ2181253` (`ata-WDC_WD30EZRX-00MMMB0_WD-WCAWZ2181253`) has a confirmed, repeatable read failure at LBA `5,860,378,112`, in unallocated space (~99.997% through the disk), so `zpool status` stays clean (`No known data errors`). The other member (`WD-WCAWZ0415134`) is confirmed healthy. Replacement is market-constrained (open-ended); the plan is monitor-and-replace-when-available, not urgent replacement. Notes for whoever handles this:
- **Identify drives by `by-id`/serial, never `/dev/sdX`** — the letters have already swapped once and drift on any reboot/hardware move.
- **Do not force-repair the bad sector** (`dd` to that LBA): it sits near the ZFS vdev labels on a live member; a raw write risks turning a contained defect into a real pool problem for a cosmetic (quieter SMART) gain. The `zpool replace` handles it properly.
- **This drive's firmware under-reports self-test LBAs by 2³² in the *legacy* log.** Read LBAs from `smartctl -x` (extended log), never `smartctl -l selftest`.
- **Escalate to urgent** (accelerate sourcing / consider `zpool offline`) if `Reallocated_Sector_Ct` > 0, `Current_Pending_Sector` > 1, a second/different bad LBA appears, or anything shows in `zpool status -v`.
- Replacement command: `zpool replace enclosure 53fed87e-1385-454e-a202-642647b0e61c <new-device-by-id>`
- **Monitoring/alerting for this is live:** monthly SMART long test (Cron Job, both drives) + TrueNAS Alert Service → Discord on Failed Selftest / Uncorrected Errors / Pool-not-healthy, tested 2026-07-02. The monthly long test will report the known read failure *every month* — that recurring "Failed Selftest" alert is expected; do not mute it, or a genuinely new failure would be missed.

### NAS host failure (motherboard/PSU/boot drive, data drives preserved)
Move the two data drives to replacement NAS hardware (any x86_64 box with 2 SATA ports + enough RAM), install TrueNAS SCALE, import the `enclosure` pool via the TrueNAS **web GUI** (not CLI — this registers the pool correctly in TrueNAS's config DB). Recreate NFS shares, deploy node_exporter (Custom App, see NAS rebuild steps below), point Proxmox's `/etc/fstab` NFS mounts and `nas-backups` PVE storage at the new IP if it changed. LXCs relying on NFS mounts will be down for the duration (media stack, paperless-ngx); Proxmox itself and NVMe-only containers (ansible, prometheus, grafana, github-runner, k8s) are unaffected.

### Proxmox host failure (disk/motherboard, hardware preserved)
Reinstall Proxmox, restore configs, restore LXC containers from backup on `nas-backups` (NFS, on the NAS — not local to Proxmox anymore). NAS-dependent containers need NFS mounts re-added to `/etc/fstab` and the `nas-backups` NFS storage re-registered before backups are reachable.

### Total loss (theft, fire, flood)
Two separate pieces of hardware to rebuild: Proxmox host and NAS. Acquire replacement hardware for both (any x86_64 with sufficient RAM/storage per requirements below), rebuild NAS first (so `nas-backups` is reachable), then rebuild Proxmox and restore from it.
Currently a gap: offsite backup not yet configured (neither host). If both are lost simultaneously with no ZFS data drives salvageable, LXC container data is unrecoverable — only code/configs (GitHub) and paperless documents (OneDrive) survive. See Known Gaps section.

## Backup inventory

| What | Where | Retention |
|------|-------|-----------|
| LXC containers (100-110) | `nas-backups` (NFS export `/mnt/enclosure/backups` on TrueNAS, `192.168.1.6`) | 5 latest, weekly Sun 01:00 |
| Paperless documents | OneDrive `/paperless-backup/` | Daily 02:00, rclone sync |
| UniFi config | UniFi cloud account | Automatic |
| Secrets (vault pw, root pw, SSH keys) | Bitwarden "Homelab DR" folder | Manual |
| Code/configs | GitHub (homelab, homelab-ansible, homelab-kubernetes) | Push-driven |

Note: backups now live on separate hardware from the Proxmox host (the NAS), which is an improvement over the old single-box setup — a Proxmox host failure no longer risks the backups themselves. However, it also means Proxmox backups are unreachable if the NAS is down, and the NAS pool itself has no off-box copy (see Known Gaps).

## Critical credentials

All stored in Bitwarden under "Homelab DR":
- Ansible vault password (decrypts all secrets in repo)
- Proxmox root password
- Ansible SSH private key (manages all containers)

## Hardware requirements for rebuild

**Proxmox host:**
- x86_64 CPU with virtualization extensions
- 16GB RAM (current usage allows for K8s VMs to be running)
- 1 boot/VM-storage drive (NVMe or SSD, 256GB or larger) — no longer needs local bulk storage, LXCs consume NFS from the NAS
- Gigabit network

**NAS (TrueNAS):**
- x86_64 CPU (current: Intel i5-6400)
- 16GB RAM
- 2 storage drives for ZFS mirror (3TB each, or larger) — native SATA, not USB (see lessons learned)
- 1 boot drive (128GB+ for TrueNAS SCALE itself)
- Gigabit network

Current hardware: Proxmox on ASUS PN52 (AMD Ryzen 9 5900HX, 16GB RAM); NAS on Intel i5-6400, 16GB DDR4, 2x 3TB WD ZFS mirror.

## Recovery procedure

### Phase 1: NAS (rebuild first — Proxmox backups and NFS mounts depend on it)

1. Install TrueNAS SCALE on the NAS hardware (fresh or replacement).
2. Configure network: static/DHCP-reserved at 192.168.1.6 (managed via UniFi DHCP reservation, not TrueNAS-side static IP — see lessons learned).
3. If data drives survived: physically install them, then **import the `enclosure` pool via the TrueNAS web GUI** (Storage → Import Pool) — not CLI `zpool import`, so TrueNAS registers the pool correctly in its own config DB. If drives did not survive, this pool and all data on it is lost; skip to creating a fresh pool once new drives are installed.
4. Recreate NFS shares (Shares → Add, or Sharing → NFS): media (`subvol-101-disk-0`), paperless (`subvol-100-disk-0`), and backups (`enclosure/backups`) — network `192.168.1.0/24`, maproot user/group `root`, read-write.
5. Recreate the scrub schedule (Storage → Storage Health → Scheduled Scrub: monthly, 1st @00:00) and the SMART long-test cron job (System Settings → Advanced → Cron Jobs: `smartctl -t long /dev/disk/by-id/<drive>` for each drive, monthly, offset from the scrub).
6. Redeploy node_exporter as a Custom App (Apps → Discover Apps → Custom App): image `prom/node-exporter`, Host Network enabled, host-path mounts `/proc`→`/host/proc` and `/sys`→`/host/sys` (read-only; **do not** attempt to mount host `/` — TrueNAS blocks this), command args `--path.procfs=/host/proc --path.sysfs=/host/sys` (watch for stray whitespace in the args field — it will crash-loop the container with a cryptic "unexpected flag" error).
7. Verify: `curl -s http://192.168.1.6:9100/metrics | grep node_zfs_zpool_state` shows `enclosure` state `online=1`.

### Phase 2: Proxmox host

1. Install Proxmox VE from ISO (https://www.proxmox.com/en/downloads)
2. Set root password (use Bitwarden value if same hardware, else new)
3. Configure network to use 192.168.1.11/24, gateway 192.168.1.1
4. Recreate storage entries via Datacenter, Storage:
   - `lvm`: LVM on NVMe, content: rootdir, images
   - `nas-backups`: NFS export from `192.168.1.6:/mnt/enclosure/backups`, content: VZDump backup file
5. Add NFS mounts to `/etc/fstab`:
   ```
   192.168.1.6:/mnt/enclosure/subvol-101-disk-0  /mnt/nas/media      nfs  defaults,_netdev,nfsvers=3  0  0
   192.168.1.6:/mnt/enclosure/subvol-100-disk-0  /mnt/nas/paperless  nfs  defaults,_netdev,nfsvers=3  0  0
   ```
   Then `mount -a` and verify with `mount | grep nas`.

### Phase 3: LXC containers

For each container in `nas-backups`:

1. In Proxmox UI: select storage `nas-backups`, Backups
2. Select latest `vzdump-lxc-XXX-*.tar.zst`
3. Restore, choose target storage (`lvm` for OS disk)
4. For containers 100-106, after restore add the `mp0` bind mount: `mp0: /mnt/nas/paperless,mp=/mnt/USBHDD,backup=0` (CT 100) or `mp0: /mnt/nas/media,mp=/mnt/USBHDD,backup=0` (CT 101-106) — **mount NFS on the Proxmox host and bind-mount into the container; do not mount NFS inside an unprivileged LXC directly.**

Recovery order (dependency-driven):
1. ansible (107), needed for re-configuration
2. paperless-ngx (100), primary service
3. prometheus (108), grafana (109), monitoring
4. github-runner (110), CI/CD
5. Media stack (101-106): sabnzbd, prowlarr, sonarr, radarr, jellyfin, bazarr

### Phase 4: Kubernetes VMs (optional, learning environment)

K8s VMs (800-802) are not currently backed up. If needed, rebuild via Ansible: ansible-playbook playbooks/kubernetes.yml

### Phase 5: SSH access restoration

From your workstation, restore the ansible_key from Bitwarden:
1. Copy private key content to `~/.ssh/ansible_key`
2. `chmod 600 ~/.ssh/ansible_key`
3. SSH to ansible LXC: `ssh -i ~/.ssh/ansible_key root@192.168.1.19`

Note: this key is for the Debian LXCs only. TrueNAS uses a separate `truenas_admin` account (not root SSH, not this key) — recover TrueNAS access via its own console/UI setup, not via Ansible.

### Phase 6: GitHub access

1. On ansible LXC, generate new SSH key: `ssh-keygen -t ed25519 -f ~/.ssh/github_key`
2. Add public key to GitHub account settings
3. Test: `ssh -T -i ~/.ssh/github_key git@github.com`

### Phase 7: Verification

| Service | URL | Verify |
|---------|-----|--------|
| TrueNAS | http://192.168.1.6 | Pool `enclosure` online, NFS shares active |
| paperless | http://192.168.1.12:8000 | Login, see documents |
| sabnzbd | http://192.168.1.13:7777 | Login, see queue |
| prowlarr | http://192.168.1.14:9696 | Login, see indexers |
| sonarr | http://192.168.1.15:8989 | Login, see series |
| radarr | http://192.168.1.16:7878 | Login, see movies |
| jellyfin | http://192.168.1.17:8096 | Login, libraries scan |
| bazarr | http://192.168.1.18:6767 | Login, see subtitles |
| prometheus | http://192.168.1.21:9090 | Targets all up, including `truenas` |
| grafana | http://192.168.1.22:3000 | Dashboards load, ZFS Pool Health alert shows Normal |

## Known gaps

- No offsite backup of either host. LXC backups live on the NAS, paperless documents live on OneDrive, but the ZFS pool itself and Proxmox VM/LXC configs have no off-site copy. Total-loss scenarios where both the Proxmox host and NAS are destroyed together (e.g. same building, fire/flood) lose all LXC data — only code/configs (GitHub) and paperless documents (OneDrive) survive.
- K8s VMs not backed up. Learning environment, rebuild via Ansible takes about 15min.
- Proxmox host config not versioned. Network and storage configs documented here, but not automatically backed up.
- NAS is a new dependency for the media stack and paperless-ngx: if the NAS is down, those LXCs lose their data mount (Proxmox itself and NVMe-only containers — ansible, prometheus, grafana, github-runner, k8s — are unaffected, since they don't consume NFS).
- One ZFS mirror member is currently defective (serial `WD-WCAWZ2181253`, contained read failure in unallocated space) and awaiting a market-constrained replacement. The mirror is not currently degraded and `zpool status` is clean, but the pool is running without effective redundancy margin on that member until the drive is replaced. Monitoring + Discord alerting are in place (see Single drive failure above). A ZFS mirror is redundancy, not a backup — and the media dataset has no off-box copy regardless.
- TrueNAS config (pool layout, NFS share definitions, cron jobs, node_exporter app config) is **not version-controlled** — it lives only in TrueNAS's own config database. A NAS host-failure rebuild currently means manually recreating shares/schedules/app from this document, not restoring a config file. TrueNAS does support exporting a system config backup (System Settings → General → Manage Configuration) — **not currently scheduled**, worth adding as a follow-up.

## Maintenance

- Quarterly: verify a single LXC backup actually restores correctly (test on a different VMID).
- Monthly (automated): NAS pool scrub and SMART long self-tests (see NAS / Storage in the main README).
- After significant changes: update this document.

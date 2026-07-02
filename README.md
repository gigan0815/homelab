# homelab

A single-node Proxmox setup running self-hosted services for personal and family use. The focus is on learning by doing - setting up, maintaining and monitoring real infrastructure rather than just reading about it.

Infrastructure automation is handled via Ansible.
Repository: [homelab-ansible](https://github.com/gigan0815/homelab-ansible)

## Hardware

| Component | Details |
| --- | --- |
| Device | ASUS PN52 Mini PC |
| CPU | AMD Ryzen 9 5900HX |
| RAM | 16GB |
| Storage | NVMe/SSD only (bulk storage moved to dedicated NAS, see below) |
| Hypervisor | Proxmox VE |

## Network

| Device | IP | Role |
| --- | --- | --- |
| UniFi Express 6 | 192.168.1.1 | Gateway / Router / DHCP Server |
| USW Flex Mini - HAR | 192.168.1.4 | Main switch to patch panel |
| USW Flex Mini - Büro | 192.168.1.5 | Office switch |
| AP Wohnzimmer | 192.168.1.2 | Access Point |
| AP Obergeschoss | 192.168.1.3 | Access Point |
| NAS (TrueNAS) | 192.168.1.6 | NAS / storage |

## Services

| Service | IP | Port | Description |
| --- | --- | --- | --- |
| paperless-ngx | 192.168.1.12 | 8000 | Document management system |
| sabnzbd | 192.168.1.13 | 7777 | Binary newsreader |
| prowlarr | 192.168.1.14 | 9696 | Indexer aggregator |
| sonarr | 192.168.1.15 | 8989 | Series library manager |
| radarr | 192.168.1.16 | 7878 | Movie library manager |
| jellyfin | 192.168.1.17 | 8096 | Media server |
| bazarr | 192.168.1.18 | 6767 | Subtitle manager |
| ansible | 192.168.1.19 | - | Ansible control node |
| - | 192.168.1.20 | - | Reserved, fallback IP for UniFi Express 6 |
| prometheus | 192.168.1.21 | 9090 | Metrics collection |
| grafana | 192.168.1.22 | 3000 | Metrics visualization |
| github-runner | 192.168.1.23 | - | GitHub Actions self-hosted runner |

## Storage / NAS

Bulk storage runs on a dedicated **TrueNAS SCALE** box, separate from the Proxmox host. Originally the 2x 3TB ZFS mirror lived in a USB enclosure attached directly to the PN52 (see Roadmap); the enclosure's USB bridge proved unreliable, so the drives were moved to native SATA on standalone NAS hardware.

| Component | Details |
| --- | --- |
| Device | Intel i5-6400, 16GB DDR4 |
| OS | TrueNAS SCALE |
| IP | 192.168.1.6 |
| Pool | `enclosure` (legacy name, kept — too much downstream config references it), 2x 3TB WD mirror, ZFS |

Proxmox LXCs consume NAS storage over **NFS**, mounted on the Proxmox host and bind-mounted into each unprivileged container (NFS is not mounted inside the LXCs directly):

| NFS export | Consumer |
| --- | --- |
| `/mnt/enclosure/subvol-101-disk-0` (media) | *arr stack + jellyfin |
| `/mnt/enclosure/subvol-100-disk-0` (paperless) | paperless-ngx |
| `/mnt/enclosure/backups` (pve-backups) | Proxmox backup job |

Monthly scrub and monthly SMART long self-tests are scheduled on TrueNAS (offset from each other to avoid overlapping I/O load). Pool health is monitored via node_exporter (deployed on TrueNAS as a Docker app through TrueNAS's own Apps system, not Ansible) feeding a Grafana alert — see Alerting section below.



## Automation

Infrastructure is managed via Ansible. The control node runs as a dedicated LXC container on the Proxmox host.

Repository: [homelab-ansible](https://github.com/gigan0815/homelab-ansible)

Current playbooks:

- `update.yml` - updates all LXC containers + Discord notification on reboot
- `paperless_backup.yml` - exports paperless-ngx and syncs to OneDrive
- `node_exporter.yml` - deploys Node Exporter on all containers
- `prometheus.yml` - deploys Prometheus on the monitoring container
- `grafana.yml` - deploys Grafana on the monitoring container
- `kubernetes.yml` - deploys Kubernetes cluster on VMs
- `github_runner.yml` - deploys GitHub Actions self-hosted runner

The NAS (TrueNAS SCALE) is **not** Ansible-managed — it's an appliance OS with its own update mechanism and Apps/Docker system for deploying services (e.g. node_exporter). It's listed in the Ansible inventory for reference only and explicitly excluded from `update.yml` and `node_exporter.yml`. NFS shares are configured through the TrueNAS UI, not `/etc/exports`.

## Kubernetes

3-node cluster running on Proxmox VMs for learning purposes.
Cluster setup is automated via Ansible, see [homelab-ansible](https://github.com/gigan0815/homelab-ansible).
Kubernetes manifests are managed in a separate repository, see [homelab-kubernetes](https://github.com/gigan0815/homelab-kubernetes).

| Node | IP | Role |
| --- | --- | --- |
| k8smaster | 192.168.1.80 | control-plane |
| k8sworker1 | 192.168.1.81 | worker |
| k8sworker2 | 192.168.1.82 | worker |

## Windows Server / Active Directory

Windows Server 2022 VM running as a Domain Controller for the `homelab.local` domain. Built to practice Microsoft Server administration, Active Directory Domain Services, DNS, and Group Policy via PowerShell.

| Node | IP | Role |
| --- | --- | --- |
| dc01 | 192.168.1.50 | Domain Controller, DNS server |

See [docs/windows-ad-lab.md](https://github.com/gigan0815/homelab/blob/main/docs/windows-ad-lab.md) for the full setup, OU structure, users, groups, and operational commands.

## CI/CD

GitHub Actions workflows run on a self-hosted runner deployed on the github-runner LXC container.

- `homelab-ansible` repository runs Ansible syntax checks on all playbooks
- `homelab-kubernetes` repository validates all Kubernetes manifests with kubeconform

## Backup

| Service | Destination | Schedule | Method |
| --- | --- | --- | --- |
| paperless-ngx | OneDrive | Daily 02:00 | document_exporter + rclone |
| LXC containers (100-110) | `nas-backups` (NFS, TrueNAS) | Sun 01:00 | Proxmox vzdump, keep-last=5 (validated 2026-07-02; retention aligned in both `storage.cfg` and `jobs.cfg`) |

Storage redundancy is provided via ZFS mirror (2x 3TB drives) on the dedicated NAS — see Storage / NAS section above. Media library has no off-box backup (accepted risk, content is re-downloadable).

## Alerting

Alerts are configured in Grafana and sent to the `#homelab-alerts` Discord channel.

| Alert | Condition | Pending period |
| --- | --- | --- |
| Container unreachable | Node Exporter not responding | 2m |
| High CPU usage | CPU > 80% | 15m |
| High RAM usage | RAM > 85% | 5m |
| High SWAP usage | SWAP > 50% | 10m |
| High disk usage | Disk > 85% | 5m |
| ZFS Pool Health | Pool state (`enclosure` on TrueNAS) degraded/faulted/suspended/unavail | 0s (immediate) |

TrueNAS also alerts natively (independent of Grafana) to Discord via a TrueNAS Alert Service. Relevant alert classes are set to route at WARNING or above: **Hardware → Failed Selftest** (a failed SMART long test — the primary early-warning signal for the defective drive, since its bad LBA is in unallocated space and does not show in `zpool status`), **Hardware → Uncorrected Errors Detected**, and **Storage → Pool Status Is Not Healthy** (CRITICAL). Delivery is a Discord webhook configured as a "Slack"-type service with `/slack` appended to the webhook URL, tested end-to-end 2026-07-02. See the NAS runbook for the defective-drive monitoring detail and the note not to mute the (expected, monthly-recurring) Failed Selftest alert.

## Architecture

[![Architecture](https://github.com/gigan0815/homelab/raw/main/docs/architecture.svg)](https://github.com/gigan0815/homelab/blob/main/docs/architecture.svg)

## Disaster Recovery

See [docs/disaster-recovery.md](https://github.com/gigan0815/homelab/blob/main/docs/disaster-recovery.md) for the recovery procedure, backup inventory, and known gaps.

See [docs/windows-ad-lab.md](https://github.com/gigan0815/homelab/blob/main/docs/windows-ad-lab.md) for the Windows Server / Active Directory lab setup.

## Roadmap

- [ ] Expand Ansible roles for all services
- [x] Re-enable PVE backup job pointed at `nas-backups` (validated 2026-07-02)
- [x] Migrate ZFS pool off Proxmox host (USB enclosure) to dedicated TrueNAS NAS
- [x] Add Ansible role for Kubernetes cluster setup
- [x] CI/CD pipeline with GitHub Actions
- [x] Migrate USB RAID to ZFS mirror
- [x] Set up Kubernetes cluster (learning environment)
- [x] Deploy monitoring stack (Prometheus + Grafana)
- [x] Windows Server AD lab (Domain Controller, OU structure, users, groups, GPO)

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Layout conventions

- **`.env` is required and untracked.** Every service reads `CONFIG_PATH`, `DATA_PATH`, `TZ`, `PUID`, `PGID`, `UMASK`, `RESTART_POLICY` from it. Do not hardcode paths in `compose.yaml`.
- **Config vs data.** Each service's state goes under `${CONFIG_PATH}/<service>`. Media and downloads live under a single `${DATA_PATH}` tree, mounted as `/data` in every *arr container so hardlinks work between `/data/usenet` (SABnzbd downloads) and `/data/media/*` (library). Keep new services on the same `/data` mount, not a separate path, or imports fall back to copying.
- **`profiles: [donotstart]`** marks services that are kept in the file but not run by default. Add new experimental services there rather than deleting them.

## Infrastructure (not in the repo)

The stack runs in a Proxmox VM named `servarr` (VM 100). `DATA_PATH` is an NFS mount served by a separate OpenMediaVault VM (VM 101) from a USB-attached 8 TB (7.3 TiB) ext4 drive. Disk-space questions therefore need checking on the OMV VM, not here; this Mac only holds the compose file.

Docker is not installed on this Mac. The VM is reachable as `ssh servarr`, with the deployed checkout at `~/servarr` (real `.env` lives there). Validate local edits there before committing with `ssh servarr 'cd ~/servarr && docker compose -f - config -q' < compose.yaml`; deploy with `git pull && docker compose up -d --remove-orphans`.

Home Assistant is deliberately **not** in this stack: it runs as Home Assistant OS in its own Proxmox VM (VM 104, `homeassistant`), kept separate because it is internet-facing. Don't add it back to `compose.yaml`.

- **LinkTap** is integrated via the gateway's local HTTP API using the HACS integration `sh00t2kill/linktap_local_http_component` (no MQTT broker, no cloud). The gateway has a static IP (`192.168.68.3`) set in its own web UI. The integration has no UI to change that IP; to change it, `ha core stop`, edit `"Local IP Address of Gateway"` in `.storage/core.config_entries`, then `ha core start`.
- **Google Home** connects through HA's manual `google_assistant` integration (Cloud-to-Cloud project in the Google Home Developer Console, service-account key at `/config/SERVICE_ACCOUNT.json`, `report_state: true`). Nabu Casa is intentionally not used.
- **Public access** is Tailscale Funnel via the HA Tailscale app (`share_homeassistant: funnel`); HA trusts `127.0.0.1` as a reverse proxy.

### Network

AT&T gateway (`192.168.1.254`, IP passthrough to the Deco) → TP-Link Deco mesh as router on `192.168.68.0/22`, gateway and DNS `192.168.68.1`, DHCP `.68.150–.240`. Static devices use `192.168.68.2–.149`, all `/22`:

| Device | IP | Where the address is set |
|---|---|---|
| pve (Proxmox host) | `.68.86` | `/etc/network/interfaces` (vmbr0), `/etc/hosts`, `/etc/resolv.conf` |
| homeassistant (VM 104) | `.68.95` | `ha network update enp6s18 …` (HA web is on port 80) |
| omv (VM 101) | `.68.112` | OMV web UI only (it regenerates netplan) |
| servarr (VM 100) | `.68.128` | `/etc/netplan/01-static.yaml` |
| dagster (VM 102) | `.68.242` | `/etc/netplan/01-static.yaml` |
| LinkTap gateway | `.68.3` | its web UI (reboot to apply) |

- **Keep the Deco on its default subnet.** On every boot it compares the LAN with the WAN DNS (`192.168.1.254`) and silently renumbers the LAN if it judges a conflict, stranding every static device; custom DNS on the Deco doesn't avoid it. This took everything offline on 2026-10-17 when the LAN was `192.168.0.0/24`.
- **NFS exports from OMV** (edit only in the OMV UI, Services → NFS): `/export/data` → servarr `/data`, `/export/dagster` → dagster `/mnt/nfs/dagster` (both clients `192.168.68.0/22`, mounted via `/etc/fstab` with `x-systemd.automount`), and `/export/proxmox` → pve storage `omv-backup` (client `192.168.68.86/32`, `no_root_squash`). Changing OMV's IP means updating both fstabs and `/etc/pve/storage.cfg`.
- **Reaching a VM whose network is broken:** every running VM has the QEMU guest agent, so `qm guest exec <id> -- <cmd>` works from pve; long-running commands (`netplan apply`, mounts) exceed its timeout, so launch them with `nohup … &`. servarr and dagster also have `serial0: socket` for `qm terminal`.
- **SSH aliases** (`~/.ssh/config` on this Mac): `pve`, `servarr` by LAN IP; `dagster` and `cube` (VM 103, normally stopped) by Tailscale IP. `omv` has no key set up.

### Proxmox host and backups

The Proxmox host is reachable as `ssh pve` (root, 192.168.68.86). Its `local-lvm` thin pool is overcommitted, so check `lvs pve/data` usage before adding VM disks, and give new VM disks `discard=on,ssd=1` so guest trims free pool space.

Backups: a nightly vzdump job (02:30, snapshot, zstd, keep 1 daily + 1 weekly) backs up VMs 100, 102 and 104 to the `omv-backup` NFS storage, which is OMV's `/export/proxmox` share on the USB drive (exported only to the host). The OMV VM itself is excluded because it serves that share, and the media under `DATA_PATH` has no backup.

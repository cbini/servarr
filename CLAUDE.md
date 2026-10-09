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

- **LinkTap** is integrated via the gateway's local HTTP API using the HACS integration `sh00t2kill/linktap_local_http_component` (no MQTT broker, no cloud).
- **Google Home** connects through HA's manual `google_assistant` integration (Cloud-to-Cloud project in the Google Home Developer Console, service-account key at `/config/SERVICE_ACCOUNT.json`, `report_state: true`). Nabu Casa is intentionally not used.
- **Public access** is Tailscale Funnel via the HA Tailscale app (`share_homeassistant: funnel`); HA trusts `127.0.0.1` as a reverse proxy.

The Proxmox `local-lvm` thin pool is overcommitted, so check `lvs pve/data` usage before adding VM disks.

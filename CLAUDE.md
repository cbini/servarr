# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single Docker Compose stack (`compose.yaml`) for a home media server. There is no application code, build, or test suite. The only artifact is the compose file; secrets and paths live in an untracked `.env`.

## Commands

```sh
docker compose up -d                          # start default services
docker compose --profile donotstart up -d X   # start an opt-in service (see below)
docker compose pull && docker compose up -d   # update images
trunk check                                   # lint (yamllint, prettier, checkov, trufflehog)
trunk fmt
```

## Layout conventions

- **`.env` is required and untracked.** Every service reads `CONFIG_PATH`, `DATA_PATH`, `TZ`, `PUID`, `PGID`, `UMASK`, `RESTART_POLICY` from it. Do not hardcode paths in `compose.yaml`.
- **Config vs data.** Each service's state goes under `${CONFIG_PATH}/<service>`. Media and downloads live under a single `${DATA_PATH}` tree, mounted as `/data` in every *arr container so hardlinks work between `/data/usenet` (SABnzbd downloads) and `/data/media/*` (library). Keep new services on the same `/data` mount, not a separate path, or imports fall back to copying.
- **`profiles: [donotstart]`** marks services that are kept in the file but not run by default (readmeabook, actualbudget, recyclarr, audiobookrequest, audiobookshelf, whisparr, stash). Add new experimental services there rather than deleting them.
- **Images.** *arr apps use `ghcr.io/hotio/*`; plex and sabnzbd use `lscr.io/linuxserver/*`. Plex and stash use `network_mode: host`; everything else publishes a fixed port.

## Infrastructure (not in the repo)

The stack runs in a Proxmox VM named `servarr` (VM 100). `DATA_PATH` is an NFS mount served by a separate OpenMediaVault VM (VM 101) from a USB-attached 8 TB (7.3 TiB) ext4 drive. Disk-space questions therefore need checking on the OMV VM, not here; this Mac only holds the compose file.

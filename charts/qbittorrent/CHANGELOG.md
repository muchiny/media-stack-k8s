# Changelog

All notable changes to the qBittorrent chart will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-11-07

### Changed
- Image officielle `ghcr.io/qbittorrent/docker-qbittorrent-nox` (équipe qBittorrent) à la place de `lscr.io/linuxserver/qbittorrent`
- Conteneur en uid 1000, `runAsNonRoot`, rootfs en lecture seule, aucune capability (CHOWN, KILL, SETGID et SETUID retirées)
- Variables `QBT_LEGAL_NOTICE` et `QBT_WEBUI_PORT` à la place de `PUID`, `PGID` et `WEBUI_PORT`
- Fichiers réorganisés selon `--profile=/config` (`qBittorrent/config/` et `qBittorrent/data/`)

### Removed
- Volume `session` (monté sur `/config/qBittorrent/data`, resté vide)

## [1.0.0] - 2024-01-01

### Added
- Initial release
- qBittorrent deployment with LinuxServer image
- Anti-seeding configuration
- Init container waiting for DNS proxy
- NodePort services for WebUI (30080) and torrent (30881)
- Persistent storage for config and downloads
- tmpfs for session data (reduces SD card wear)
- NetworkPolicy with DNS proxy egress
- PodDisruptionBudget for high availability
- Health probes (startup, liveness, readiness)
- Priority class support (media-high)
- ARM64 node selector for Raspberry Pi 5

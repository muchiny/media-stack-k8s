# Changelog

All notable changes to the Plex chart will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2024-01-01

### Added
- Initial release
- Plex Media Server deployment with LinuxServer image
- Hardware transcoding support via /dev/dri (privileged mode)
- hostNetwork for DLNA/GDM discovery
- Persistent storage for config and media
- Memory-backed transcode directory (emptyDir)
- NetworkPolicy for media traffic
- PodDisruptionBudget for high availability
- Health probes (startup, liveness, readiness)
- Priority class support (media-high)
- ARM64 node selector for Raspberry Pi 5

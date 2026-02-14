# Changelog

All notable changes to the Cloudflared chart will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2024-01-01

### Added
- Initial release
- DNS-over-HTTPS proxy deployment
- Fixed ClusterIP (10.43.48.123) for CoreDNS integration
- NetworkPolicy for secure DNS traffic
- PodDisruptionBudget for high availability
- Health probes (startup, liveness, readiness)
- Priority class support (media-critical)
- ARM64 node selector for Raspberry Pi 5

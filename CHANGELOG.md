# Changelog

All notable changes to this role are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

### Fixed

- The hardened systemd unit emitted `ReadWritePaths=` for the textfile
  collector directory even when the textfile collector was disabled. Because
  the role only creates that directory when the collector is enabled, systemd
  refused to set up the unit's mount namespace and the service never came up —
  silently, since the converge itself still reported success.
- `node_exporter_arch` was hardcoded to `amd64`, so arm64 and armv7 hosts were
  handed a binary they could not execute. It is now derived from
  `ansible_facts.architecture`, and preflight fails early on an architecture
  upstream publishes no build for.
- `meta/main.yml` had no Galaxy `namespace`, which made `ansible-compat` reject
  the computed role name and stopped molecule from running at all.
- Both CI workflows installed `ansible>=2.9,<2.11`, which can no longer resolve
  against current `ansible-lint` or `molecule`; the molecule job also used the
  `molecule[docker]` extra that was removed in molecule 5.

### Security

- The release archive is now verified against the `sha256sums.txt` published
  with each upstream release by default, instead of being installed unverified.
  `node_exporter_archive_checksum` still pins an exact digest.
- The archive is staged in a root-owned directory instead of `/tmp`, where a
  local user could pre-create or symlink the predictable download target.
- Added `ProtectHostname`, `ProtectClock`, `LockPersonality`, `RestrictRealtime`,
  `RestrictSUIDSGID` and `RemoveIPC` to the hardened unit.
- Documented that the metrics endpoint is unauthenticated and, on the default
  wildcard bind, readable by anyone who can reach port 9100. Preflight now
  prints a reminder, suppressible with
  `node_exporter_warn_on_wildcard_listen: false`.

### Added

- `tests/render.yml`, a container-free test that renders the systemd unit
  across several variable combinations and asserts on the output.
- A `no-textfile` molecule scenario covering the regression above.
- `node_exporter_release_base_url` for pointing the role at an internal mirror.

### Changed

- The molecule default scenario now covers Rocky Linux 9 and Ubuntu 22.04 on
  multi-arch images, replacing the end-of-life CentOS 8 platform.
- Molecule verification additionally asserts the service is enabled at boot,
  runs as a non-root account, and has the expected sandboxing in effect.

## [1.0.0] - 2021-03-14

### Added

- Initial release of the node_exporter role.
- Download and versioned installation of the node_exporter release archive.
- Dedicated system user and group for the service.
- Templated systemd unit with configurable listen address and collectors.
- Optional systemd sandboxing directives.
- Optional checksum verification of the downloaded archive.
- Preflight assertions for required variables.
- Molecule default scenario and a syntax-check test playbook.

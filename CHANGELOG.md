# Changelog

All notable changes to this role are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

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

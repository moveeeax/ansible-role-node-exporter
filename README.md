# ansible-role-node-exporter

![CI](https://github.com/mtarassov/ansible-role-node-exporter/workflows/CI/badge.svg)
![Molecule](https://github.com/mtarassov/ansible-role-node-exporter/workflows/Molecule/badge.svg)

Ansible role that installs the [Prometheus node_exporter](https://github.com/prometheus/node_exporter)
and runs it as a systemd service. The release archive is downloaded from
GitHub, unpacked into a versioned directory, and exposed through a `current`
symlink so that upgrades and rollbacks are a matter of flipping the link.

## Requirements

- Ansible 2.9 or newer.
- A target host running systemd (see supported platforms in `meta/main.yml`).

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `node_exporter_version` | `1.1.2` | Release version to install. |
| `node_exporter_arch` | `amd64` | Architecture used to build the archive name. |
| `node_exporter_download_url` | GitHub release URL | Full download URL for the release archive. |
| `node_exporter_install_dir` | `/opt/node_exporter` | Base installation directory. |
| `node_exporter_version_dir` | `<install_dir>/node_exporter-<version>` | Versioned directory the archive is unpacked into. |
| `node_exporter_binary` | `<install_dir>/current/node_exporter` | Path to the binary the unit runs. |
| `node_exporter_user` | `node_exporter` | System user the service runs as. |
| `node_exporter_group` | `node_exporter` | System group the service runs as. |
| `node_exporter_listen_address` | `0.0.0.0:9100` | Address passed to `--web.listen-address`. |
| `node_exporter_metrics_path` | `/metrics` | Path passed to `--web.telemetry-path`. |
| `node_exporter_enabled_collectors` | `[systemd, textfile]` | Collectors enabled with `--collector.<name>`. |
| `node_exporter_disabled_collectors` | `[]` | Collectors disabled with `--no-collector.<name>`. |
| `node_exporter_extra_flags` | `[]` | Additional raw flags appended to `ExecStart`. |

## Example Playbook

```yaml
---
- name: Deploy node_exporter
  hosts: monitoring
  become: true
  roles:
    - role: ansible-role-node-exporter
      vars:
        node_exporter_version: "1.1.2"
        node_exporter_listen_address: "0.0.0.0:9100"
        node_exporter_enabled_collectors:
          - systemd
          - textfile
          - processes
```

## License

MIT

## Author Information

Michael Tarassov

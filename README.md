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
- Network access from the target host to `github.com`, or an internal mirror
  configured through `node_exporter_release_base_url`.

## Security

Read this before pointing the role at anything reachable from a network you do
not control.

**The metrics endpoint is unauthenticated and describes your host in detail.**
node_exporter exposes mounted filesystems and their free space, network
interfaces and addresses, kernel and distribution version, systemd unit names
and states, and — if you enable the `processes` collector — what is running.
Anyone who can open a TCP connection to port 9100 can read all of it. There is
no authentication in front of it by default.

The role ships upstream's `0.0.0.0:9100` default so it works out of the box,
which means **an unfirewalled host is exposing this to the whole network**. Do
one of the following:

- Bind to a management address, so the port is not reachable from anywhere
  else:

  ```yaml
  node_exporter_listen_address: "10.0.0.5:9100"
  ```

- Or keep the wildcard bind and restrict port 9100 to your Prometheus server
  with a host firewall or security group. This role does **not** manage
  firewall rules for you.

Once you have done either, silence the reminder the role prints during
preflight with `node_exporter_warn_on_wildcard_listen: false`.

For TLS and basic auth, node_exporter supports a web-config file; pass it
through `node_exporter_extra_flags` (`--web.config` on 1.x, renamed
`--web.config.file` in 1.5).

**Downloads are checksum-verified by default.** The archive is checked against
the `sha256sums.txt` published with each upstream release before it is
unpacked, so a tampered or truncated download fails the play instead of being
installed. Pin an exact digest with `node_exporter_archive_checksum` if you
prefer not to trust the checksum file fetched at run time. The archive is
staged in a root-owned directory rather than `/tmp`, so local users cannot
pre-plant or symlink the download target.

If you serve releases from a mirror, override `node_exporter_release_base_url`
rather than `node_exporter_download_url` alone — otherwise the archive comes
from your mirror while the checksum file is still fetched from GitHub. Mirrors
must keep the upstream archive filename, since that is the key looked up in
`sha256sums.txt`.

**The service does not run as root.** It runs as the dedicated `node_exporter`
system account with `nologin` as its shell, and the unit is sandboxed by
default (`ProtectSystem=strict`, `NoNewPrivileges`, `PrivateTmp` and friends).
Set `node_exporter_harden_service: false` to drop the sandboxing directives if
a collector you need is incompatible with them.

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `node_exporter_version` | `1.1.2` | Release version to install. |
| `node_exporter_arch` | detected from the host | Architecture used to build the archive name, mapped from `ansible_facts.architecture` (e.g. `aarch64` → `arm64`). Override to cross-install. |
| `node_exporter_release_base_url` | GitHub release URL | Base URL for the archive and checksum file. Point at an internal mirror if needed. |
| `node_exporter_archive` | `node_exporter-<version>.linux-<arch>.tar.gz` | Release archive filename. |
| `node_exporter_download_url` | `<release_base_url>/<archive>` | Full download URL for the release archive. |
| `node_exporter_checksum_url` | `<release_base_url>/sha256sums.txt` | Checksum file the archive is verified against. Matched on the basename of the download URL. |
| `node_exporter_archive_checksum` | `""` | Exact digest to pin instead, e.g. `sha256:<hex>`. Takes precedence over the checksum URL. |
| `node_exporter_install_dir` | `/opt/node_exporter` | Base installation directory. |
| `node_exporter_download_dir` | `<install_dir>/archives` | Root-owned directory the archive is staged in. |
| `node_exporter_version_dir` | `<install_dir>/node_exporter-<version>` | Versioned directory the archive is unpacked into. |
| `node_exporter_binary` | `<install_dir>/current/node_exporter` | Path to the binary the unit runs. |
| `node_exporter_user` | `node_exporter` | System user the service runs as. |
| `node_exporter_group` | `node_exporter` | System group the service runs as. |
| `node_exporter_listen_address` | `0.0.0.0:9100` | Address passed to `--web.listen-address`. See [Security](#security). |
| `node_exporter_warn_on_wildcard_listen` | `true` | Print a preflight reminder when listening on every interface. |
| `node_exporter_metrics_path` | `/metrics` | Path passed to `--web.telemetry-path`. |
| `node_exporter_textfile_directory` | `/var/lib/node_exporter/textfile_collector` | Directory scraped by the textfile collector. Only created when `textfile` is in the enabled list. |
| `node_exporter_enabled_collectors` | `[systemd, textfile]` | Collectors enabled with `--collector.<name>`. |
| `node_exporter_disabled_collectors` | `[]` | Collectors disabled with `--no-collector.<name>`. |
| `node_exporter_extra_flags` | `[]` | Additional raw flags appended to `ExecStart`. |
| `node_exporter_harden_service` | `true` | Apply systemd sandboxing directives to the unit. |

### A note on collectors

node_exporter already enables a large set of collectors by default.
`node_exporter_enabled_collectors` only *adds* `--collector.<name>` flags, so
removing an entry does not switch that collector off — it only stops the role
managing anything associated with it. To actually turn a collector off, list it
in `node_exporter_disabled_collectors`, which renders `--no-collector.<name>`.

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
        # Bind to this host's management address rather than every interface.
        node_exporter_listen_address: "{{ ansible_default_ipv4.address }}:9100"
        node_exporter_enabled_collectors:
          - systemd
          - textfile
          - processes
```

## Testing

```bash
pip install "ansible-core>=2.16" ansible-lint yamllint molecule "molecule-plugins[docker]"
ansible-galaxy collection install community.docker ansible.posix

yamllint .
ansible-lint
ansible-playbook --syntax-check -i tests/inventory tests/test.yml

# Renders the systemd unit across several variable combinations and asserts on
# the result. No container or root required.
ansible-playbook -i tests/inventory tests/render.yml

# Full converge, idempotence and verify against real systemd in containers.
molecule test -s default      # Rocky Linux 9 and Ubuntu 22.04
molecule test -s no-textfile  # regression: hardened unit, textfile collector off
```

## License

MIT

## Author Information

Michael Tarassov

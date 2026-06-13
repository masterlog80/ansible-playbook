# ansible-copilot

An Ansible playbook for bootstrapping a fleet of Linux nodes (Debian/Ubuntu-based) from a fresh OS image to a fully configured K3S-ready state. Covers system updates, package installation, hardening, user management, networking, and shell environment setup.

---

## Features

- 🔄 **System update** – `apt update` + dist-upgrade on every run
- 📦 **Package installation** – installs a curated set of packages defined in `group_vars/all.yml` (htop, btop, curl, git, zsh, iperf3, open-iscsi, cifs-utils, and more)
- 💾 **Disable swap** – `swapoff -a` + removes swap entries from `/etc/fstab` + masks systemd swap units (required for K3S)
- ☸️ **K3S installation** – downloads and runs the official K3S install script; version and extra flags configurable via variables
- 🕐 **Timezone** – sets the system clock to `Asia/Tokyo`
- 🌐 **DNS & Domain** – configures `systemd-resolved` with custom DNS servers and search domain `salvetti.info`
- 🔌 **UPS udev rule** – creates `/etc/udev/rules.d/99-ups.rules` for the APC UPS USB device
- 🔇 **Disable sysstat** – stops and masks the sysstat service
- 👤 **User management** – deletes all regular non-root users, creates user `lorenzo` with passwordless sudo
- 🔑 **SSH key import** – imports SSH public keys from GitHub account `masterlog80` for user `lorenzo`
- 🔒 **Root hardening** – locks the root password and sets `PermitRootLogin no` in `sshd_config`
- 🖥️ **Oh My Zsh** – installs Oh My Zsh, Powerlevel10k theme, and plugins `zsh-syntax-highlighting`, `zsh-autosuggestions`, `colorize`, `last-working-dir`, `git`
- 🔐 **Getty autologin** – configures automatic login as `lorenzo` on the local console
- 🌐 **Static IP** – deploys a per-host Netplan configuration (defined in `host_vars/`)
- 🏷️ **Hostname** – sets the system hostname per host

---

## Quick Start

### Prerequisites

**Control machine** (where you run Ansible):

```bash
pip install ansible
ansible-galaxy collection install community.general ansible.posix
```

**Managed hosts:**
- Debian 11+ or Ubuntu 20.04+
- SSH access with a user that has `sudo` rights
- Python 3 available at `/usr/bin/python3`

### Clone & run

```bash
git clone https://github.com/masterlog80/ansible-copilot.git
cd ansible-copilot
```

Edit `inventory/hosts.ini` to point at your nodes:

```ini
[nodes]
node1  ansible_host=192.168.10.10  ansible_user=ubuntu
node2  ansible_host=192.168.10.11  ansible_user=ubuntu
node3  ansible_host=192.168.10.12  ansible_user=ubuntu
```

Run a dry-run first:

```bash
ansible-playbook site.yml --check --diff
```

Apply:

```bash
ansible-playbook site.yml
```

Target a single host:

```bash
ansible-playbook site.yml --limit node1
```

### Uninstall / rollback

The playbook is not destructive by default, but to undo specific tasks you can run individual tags (see below) or manually revert on the affected host.

---

## Repository Structure

```
.
├── site.yml                   # Main playbook — run this
├── inventory/
│   └── hosts.ini              # Managed hosts + connection settings
├── group_vars/
│   └── all.yml                # Variables shared by every host
├── host_vars/
│   ├── node1.yml              # Per-host network config for node1
│   ├── node2.yml              # Per-host network config for node2
│   └── node3.yml              # Per-host network config for node3
└── templates/
    ├── netplan.yml.j2          # Netplan static-IP template
    ├── resolved.conf.j2        # systemd-resolved DNS template
    └── zshrc.j2               # .zshrc template (Oh My Zsh + Powerlevel10k)
```

---

## Configuration

### `group_vars/all.yml` — shared variables

| Variable | Default | Description |
|----------|---------|-------------|
| `packages` | see file | List of apt packages to install on all nodes |
| `timezone` | `Asia/Tokyo` | System timezone |
| `dns_servers` | `192.168.10.3`, `192.168.10.4`, `1.1.1.1` | DNS servers for `systemd-resolved` |
| `dns_search_domain` | `salvetti.info` | DNS search domain |
| `admin_user` | `lorenzo` | Name of the admin user to create |
| `admin_github_user` | `masterlog80` | GitHub username from which SSH public keys are imported |
| `k3s_version` | `""` (latest) | K3S version to install (e.g. `v1.29.3+k3s1`) |
| `k3s_install_flags` | `""` | Extra flags for the K3S installer (e.g. `--disable traefik`) |
| `subnet_prefix` | `24` | Subnet prefix length for static IP |
| `gateway` | `192.168.10.1` | Default gateway |

### `host_vars/<node>.yml` — per-host network config

Override `static_ip` and optionally `gateway` / `subnet_prefix` per node:

```yaml
static_ip: 192.168.10.21
gateway: 192.168.10.1
subnet_prefix: 24
```

---

## Tags

Run only specific parts of the playbook with `--tags`:

| Tag | Task |
|-----|------|
| `update` | System update + dist-upgrade |
| `packages` | Package installation |
| `swap` | Disable swap |
| `k3s` | K3S installation |
| `timezone` | Set timezone |
| `dns` | Configure systemd-resolved |
| `users` | User management |
| `ssh` | SSH hardening |
| `zsh` | Oh My Zsh + Powerlevel10k |
| `autologin` | Getty autologin |
| `network` | Static IP via Netplan |
| `hostname` | Set hostname |

Example — only apply networking and hostname:

```bash
ansible-playbook site.yml --tags network,hostname
```

---

## License

MIT

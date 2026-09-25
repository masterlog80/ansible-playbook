# ansible-playbook

An Ansible playbook for bootstrapping a fleet of Linux nodes (Debian/Ubuntu-based) from a fresh OS image to a fully configured, role-specific state. Covers system updates, package installation, hardening, user management, networking, shell environment setup — plus a per-node choice of **K3S** or **Docker** (or both, or neither) driven entirely by inventory group membership.

---

## Design: common baseline + role groups

One playbook (`site.yml`), three plays:

| Play | Runs on | What it does |
|------|---------|---------------|
| `System bootstrap (common baseline)` | `hosts: all` | Updates, packages, swap, timezone, NTP, DNS, UPS udev rule, sysstat, users, SSH hardening, zsh, autologin, static IP, hostname — every managed node gets this |
| `K3S role` | `hosts: k3s_nodes` | Installs K3S — only on hosts in the `[k3s_nodes]` inventory group |
| `Docker role` | `hosts: docker_nodes` | Installs Docker Engine + Compose plugin — only on hosts in the `[docker_nodes]` inventory group |

**Which device gets what is decided by inventory group membership, not by writing a separate playbook per device.** A host can be in `k3s_nodes`, `docker_nodes`, both, or neither (baseline-only) — see `inventory/hosts.ini`.

To add a new role later (e.g. `monitoring_nodes`), add a new play at the bottom of `site.yml` gated on a new group and a matching `group_vars/monitoring_nodes.yml`, the same way `k3s_nodes` / `docker_nodes` are done. No forking of the whole playbook required.

---

## Features

- 🔄 **System update** – `apt update` + dist-upgrade on every run
- 📦 **Package installation** – curated package set from `group_vars/all.yml`
- 💾 **Disable swap** – required for K3S, applied to every node regardless of role
- ☸️ **K3S** (`k3s_nodes` group) – downloads and runs the official K3S install script; version and extra flags configurable via `group_vars/k3s_nodes.yml`
- 🐳 **Docker** (`docker_nodes` group) – adds Docker's official apt repo + GPG key, installs `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`, and adds the admin user to the `docker` group
- 🕐 **Timezone** – sets the system clock to `Asia/Tokyo`
- 🌐 **DNS & Domain** – configures `systemd-resolved` with custom DNS servers and search domain `salvetti.info`
- 🔌 **UPS udev rule** – creates `/etc/udev/rules.d/99-ups.rules` for the APC UPS USB device
- 🔇 **Disable sysstat** – stops and masks the sysstat service
- 👤 **User management** – deletes all regular non-root users, creates user `lorenzo` with passwordless sudo
- 🔑 **SSH key import** – imports SSH public keys from GitHub account `masterlog80` for user `lorenzo`
- 🔒 **Root hardening** – sets `PermitRootLogin no` in `sshd_config`
- 🖥️ **Oh My Zsh** – installs Oh My Zsh, Powerlevel10k theme, `zsh-syntax-highlighting`, `zsh-autosuggestions`
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
git clone https://github.com/masterlog80/ansible-playbook.git
cd ansible-playbook
```

Edit `inventory/hosts.ini` — add the host under `[nodes]`, then add it under whichever role group(s) match what it should run:

```ini
[nodes]
node1  ansible_host=192.168.10.10  ansible_user=ubuntu
node2  ansible_host=192.168.10.11  ansible_user=ubuntu
node3  ansible_host=192.168.10.12  ansible_user=ubuntu

[k3s_nodes]
node1
node2

[docker_nodes]
node3
```

Also add a `host_vars/<name>.yml` for the new host (see `host_vars/node1.yml` for the shape) so it has a static IP/hostname.

Run a dry-run first:

```bash
ansible-playbook site.yml --check --diff
```

Apply everything (baseline + whichever roles each host belongs to):

```bash
ansible-playbook site.yml
```

Target a single host:

```bash
ansible-playbook site.yml --limit node1
```

Target just a role (e.g. re-run Docker install across all Docker-role hosts only):

```bash
ansible-playbook site.yml --tags docker
```

### Uninstall / rollback

The playbook is not destructive by default, but to undo specific tasks you can run individual tags (see below) or manually revert on the affected host.

---

## Repository Structure

```
.
├── site.yml                   # Main playbook — run this (3 plays: baseline, k3s, docker)
├── inventory/
│   └── hosts.ini              # Managed hosts + role groups (k3s_nodes / docker_nodes)
├── group_vars/
│   ├── all.yml                # Variables shared by every host
│   ├── k3s_nodes.yml          # K3S version/flags — applies only to [k3s_nodes]
│   └── docker_nodes.yml       # Docker packages/channel — applies only to [docker_nodes]
├── host_vars/
│   ├── node1.yml               # Per-host network config for node1
│   ├── node2.yml               # Per-host network config for node2
│   └── node3.yml               # Per-host network config for node3
└── templates/
    ├── netplan.yml.j2          # Netplan static-IP template
    ├── resolved.conf.j2        # systemd-resolved DNS template
    └── zshrc.j2                # .zshrc template (Oh My Zsh + Powerlevel10k)
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
| `subnet_prefix` | `24` | Subnet prefix length for static IP |
| `gateway` | `192.168.10.1` | Default gateway |

### `group_vars/k3s_nodes.yml` — K3S-role variables

| Variable | Default | Description |
|----------|---------|-------------|
| `k3s_version` | `""` (latest) | K3S version to install (e.g. `v1.29.3+k3s1`) |
| `k3s_install_flags` | `""` | Extra flags for the K3S installer (e.g. `--disable traefik`) |

### `group_vars/docker_nodes.yml` — Docker-role variables

| Variable | Default | Description |
|----------|---------|-------------|
| `docker_channel` | `stable` | Docker apt repo channel (`stable`, `test`, `nightly`) |
| `docker_packages` | see file | Docker packages installed from the official apt repo |
| `docker_group_members` | `["{{ admin_user }}"]` | Users added to the `docker` group |

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

| Tag | Scope | Task(s) |
|-----|-------|---------|
| `baseline` | all hosts | Everything in the common bootstrap play |
| `update` | all hosts | System update + dist-upgrade |
| `packages` | all hosts | Package installation |
| `swap` | all hosts | Disable swap |
| `timezone` | all hosts | Set timezone |
| `ntp` | all hosts | Configure Chrony / systemd-timesyncd |
| `dns` | all hosts | Configure systemd-resolved |
| `ups` | all hosts | UPS udev rule |
| `sysstat` | all hosts | Disable sysstat |
| `users` | all hosts | User management |
| `ssh` | all hosts | SSH hardening |
| `zsh` | all hosts | Oh My Zsh + Powerlevel10k |
| `autologin` | all hosts | Getty autologin |
| `network` | all hosts | Static IP via Netplan |
| `hostname` | all hosts | Set hostname |
| `k3s` | `k3s_nodes` only | K3S installation |
| `docker` | `docker_nodes` only | Docker installation |

Example — only apply networking and hostname:

```bash
ansible-playbook site.yml --tags network,hostname
```

Example — (re)install Docker on Docker-role hosts only, without touching the baseline:

```bash
ansible-playbook site.yml --tags docker
```

---

## Adding a new role

The K3S and Docker roles both follow the same recipe. To add another one (monitoring agent, NFS client, whatever), repeat these five steps — no new playbook file, no forking `site.yml`.

**Worked example: adding a `monitoring_nodes` role that installs a Prometheus node-exporter.**

1. **Add a group to `inventory/hosts.ini`** and put the relevant hosts in it (a host can belong to more than one role group):

   ```ini
   [monitoring_nodes]
   node1
   node3
   ```

2. **Add `group_vars/monitoring_nodes.yml`** for anything the role needs to be configurable:

   ```yaml
   ---
   node_exporter_version: "1.8.2"
   node_exporter_port: 9100
   ```

3. **Add a new play at the bottom of `site.yml`**, gated on the group, tagged with the role name:

   ```yaml
   # =============================================================================
   # PLAY 4 — Monitoring role (hosts in [monitoring_nodes])
   # =============================================================================
   - name: Monitoring role
     hosts: monitoring_nodes
     become: true
     tags: [monitoring]

     tasks:
       - name: Download node_exporter
         ansible.builtin.get_url:
           url: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
           dest: /tmp/node_exporter.tar.gz
         tags: [monitoring]

       # ... unpack, install as a systemd service, etc.
   ```

   Tag every task in the new play with the role name (`monitoring`), the same way `k3s` and `docker` tag theirs — that's what makes `--tags monitoring` work later.

4. **(Optional) Add per-host overrides** in `host_vars/<name>.yml` if a specific monitoring host needs something the group default doesn't cover.

5. **Document it**: add a row to the Features list above, a row to the `group_vars/monitoring_nodes.yml` table under Configuration, and a row to the Tags table below (`monitoring` | `monitoring_nodes` only | node_exporter installation).

That's the whole pattern. `site.yml` stays one file; `--tags <role>` and inventory group membership are what actually decide which devices get which software.

### Rule of thumb: role play vs. baseline task

- If **every** node should get it regardless of what else it runs → it belongs in the common baseline play (`hosts: all`), tagged `baseline` (+ a specific tag like `zsh`, `dns`, etc.).
- If only **some** nodes should get it → it's a new role: new group, new play, new `group_vars/<group>.yml`, own tag. Don't add `when: some_flag_is_true` conditionals to the baseline play to fake this — a real inventory group is easier to reason about, easier to select from Semaphore UI (point a template's inventory/tag at the group), and keeps `--tags <role>` working correctly.

---

## Running from Semaphore UI

If you're driving this playbook from a UI (e.g. [ansible-docker](https://github.com/masterlog80/ansible-docker)'s Semaphore UI stack) rather than the CLI, the role groups map directly onto task templates:

- One template running `site.yml` with no `--tags` / `--limit` set → applies baseline + whichever role each host in the inventory belongs to.
- A second template with `--tags k3s` (or a Survey Variable for the tag) → touches only `k3s_nodes`.
- A third with `--tags docker` → touches only `docker_nodes`.

Target IP/host selection then comes from which inventory (or `--limit`/survey variable) each template is pointed at, rather than from separate playbooks.

---

## License

MIT

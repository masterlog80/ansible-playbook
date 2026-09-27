# ansible-playbook

An Ansible playbook for bootstrapping a fleet of Linux nodes (Debian/Ubuntu-based) from a fresh OS image to a fully configured, role-specific state. Covers system updates, package installation, hardening, user management, networking, shell environment setup — plus a per-node choice of **K3S** or **Docker** (or both, or neither) driven entirely by inventory group membership.

---

## Design: common baseline + role groups

One playbook (`site.yml`), seven plays:

| Play | Runs on | What it does |
|------|---------|---------------|
| `System bootstrap (common baseline)` | `hosts: all` | Updates, packages (with `apt --fix-broken` before/after), swap, timezone, NTP, DNS, udev rules (GSM modem, UPS OMRON, UPS APC), disabling sysstat/nvmf-autoconnect, users, SSH hardening, zsh, autologin, static IP, hostname — every managed node gets this |
| `K3S role` | `hosts: k3s_nodes` | Installs K3S — only on hosts in the `[k3s_nodes]` inventory group |
| `Docker role` | `hosts: docker_nodes` | Installs Docker Engine + Compose plugin — only on hosts in the `[docker_nodes]` inventory group |
| `K3S control-plane (master) role` | `hosts: k3s_master` | Installs K3S with control-plane flags, taints the node, installs FluxCD + Helm — only on hosts in the `[k3s_master]` inventory group |
| `role_1` | `hosts: role_1` | Placeholder role — only on hosts in the `[role_1]` inventory group |
| `role_2` | `hosts: role_2` | Placeholder role — only on hosts in the `[role_2]` inventory group |
| `role_3` | `hosts: role_3` | Placeholder role — only on hosts in the `[role_3]` inventory group |

`role_1` / `role_2` / `role_3` are scaffolding — each currently runs a single `debug` task so the pattern is provable end-to-end. Replace the placeholder task in each play with real work (and rename the group/tag/`group_vars` file to something meaningful) once you know what that role should actually do.

**Which device gets what is decided by inventory group membership, not by writing a separate playbook per device.** A host can be in any combination of `k3s_nodes`, `docker_nodes`, `k3s_master`, `role_1`, `role_2`, `role_3` — or none of them (baseline-only) — see `inventory/hosts.ini` and "Adding a host to a role" below. Don't put the same host in both `k3s_nodes` and `k3s_master` — see the `k3s_master` note in `inventory/hosts.ini`.

To add a new role later (e.g. `monitoring_nodes`), add a new play at the bottom of `site.yml` gated on a new group and a matching `group_vars/monitoring_nodes.yml`, the same way the roles above are done. No forking of the whole playbook required.

---

## Features

- 🔄 **System update** – `apt update` + dist-upgrade on every run
- 📦 **Package installation** – curated package set from `group_vars/all.yml`, wrapped in `apt --fix-broken` (before and after) in case any packages are left in a broken state
- 💾 **Disable swap** – required for K3S, applied to every node regardless of role
- ☸️ **K3S** (`k3s_nodes` group) – downloads and runs the official K3S install script; version and extra flags configurable via `group_vars/k3s_nodes.yml`
- ☸️ **K3S control-plane** (`k3s_master` group) – installs K3S with `--disable traefik --disable servicelb --disable metrics-server`, taints the node `node-role.kubernetes.io/control-plane=:NoSchedule`, persists `KUBECONFIG=/etc/rancher/k3s/k3s.yaml` in `/etc/environment`, and installs FluxCD + Helm
- 🐳 **Docker** (`docker_nodes` group) – adds Docker's official apt repo + GPG key, installs `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`, and adds the admin user to the `docker` group
- 🕐 **Timezone** – sets the system clock to `Asia/Tokyo`
- 🌐 **DNS & Domain** – configures `systemd-resolved` with custom DNS servers and search domain `salvetti.info`, disables the DNS stub listener (frees port 53 for a local DNS server like dnsmasq — see [Baeldung](https://www.baeldung.com/linux/dnsmasq-systemd-resolved-conflicts)), and repoints `/etc/resolv.conf` at systemd-resolved's non-stub `resolv.conf` so resolution doesn't break once the stub listener is off
- 🔌 **udev rules** – `/etc/udev/rules.d/90-qualcomm.rules` (GSM modem, auto-switches a Qualcomm USB stick out of storage mode via `usb_modeswitch`), `/etc/udev/rules.d/90-omron-ups.conf` (OMRON UPS — symlinks to `/dev/ups`, group `nut`), `/etc/udev/rules.d/99-ups.rules` (APC UPS — symlinks to `/dev/ups-network`)
- 🔇 **Disable unwanted services** – stops and masks `sysstat` (and clears `/var/log/sysstat/*`) and `nvmf-autoconnect`
- 👤 **User management** – deletes all regular non-root users other than the admin user, creates user `lorenzo` with passwordless sudo
- 🔑 **SSH key import** – imports SSH public keys from GitHub account `masterlog80` for both `lorenzo` and `root`
- 🔒 **SSH lockdown** – `PermitRootLogin no` (root never accepts SSH connections — root access is only ever obtained via `{{ admin_user }}` + `sudo`/`become`) and `PasswordAuthentication no` (no password SSH login for any user once the first-run key is installed). See "SSH access" below for the two-phase credential model
- 🔓 **sshd MaxAuthTries for `lorenzo`** – raises `MaxAuthTries` to 20 for that user only, to avoid sshd dropping the connection with "Too many authentication failures" before it reaches a valid key/password (common when an SSH agent offers many keys)
- 🖥️ **Oh My Zsh** – installs Oh My Zsh + Powerlevel10k theme for **every user in `zsh_users`** (default: the admin user and `root`), with the plugin set: `kubectl`, `kubectx`, `kube-ps1`, `git`, `zsh-syntax-highlighting`, `zsh-autosuggestions`, `docker`, `docker-compose`, `colorize`, `last-working-dir`. `zsh-syntax-highlighting` and `zsh-autosuggestions` are cloned from their upstream repos (not bundled with Oh My Zsh); the rest ship with Oh My Zsh itself
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
- Initial SSH access as `root` using the device's default password. After the first successful run, use the matching private SSH key for subsequent runs.
- Python 3 available at `/usr/bin/python3`

### Clone & run

```bash
git clone https://github.com/masterlog80/ansible-playbook.git
cd ansible-playbook
```

Edit `inventory/hosts.ini` — add the host under `[nodes]`, then add it under whichever role group(s) match what it should run:

```ini
[nodes]
node1  ansible_host=192.168.10.10  ansible_user=root
node2  ansible_host=192.168.10.11  ansible_user=root
node3  ansible_host=192.168.10.12  ansible_user=root

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

### SSH access: connect as `{{ admin_user }}`, never root

**Root never accepts SSH connections at all** (`PermitRootLogin no`) — root access on a managed node is only ever obtained via `{{ admin_user }}` (`lorenzo`) + `sudo`/`become`, never a direct root login. This is why every play in `site.yml` uses `become: true`: Ansible connects as `{{ admin_user }}` and escalates to root per-task via passwordless sudo, rather than connecting as root directly.

**Two-phase credential model, matching `inventory/hosts.ini`'s `ansible_user={{ admin_user }}`:**

1. **First run on a brand-new node — username/password credential**
   `{{ admin_user }}` must already exist on the node with a known password before Ansible can reach it at all (e.g. created during OS install/imaging — this playbook's own user-creation task can't help here, since Ansible needs to log in before it can run anything). Configure Semaphore's SSH credential as:
   - **User:** `{{ admin_user }}`
   - **Password:** the account's current password
   This run adds `{{ admin_user }}`'s GitHub public key (`ansible.posix.authorized_key`, fetched from `https://github.com/<admin_github_user>.keys`), grants passwordless sudo, and then locks down `sshd` (`PermitRootLogin no`, `PasswordAuthentication no`).
2. **Every run after that — private-key credential**
   Once the key is installed and password auth is disabled, switch Semaphore's SSH credential for that node to:
   - **User:** `{{ admin_user }}`
   - **Private key:** the key matching the public key published at `https://github.com/<admin_github_user>.keys`
   Root privilege for any task is obtained via `become`/`sudo`, not by SSHing in as root — which is no longer possible anyway.

Switching Semaphore's credential from password to private key after the first run is a manual step done in Semaphore's credential configuration — this playbook doesn't (and can't) automate that switch itself, since it happens outside the SSH session the playbook runs in.

No password or private key is committed to this repository.

> **Important:** the private key used by Semaphore/Ansible must correspond to a public
> key currently published at `https://github.com/<admin_github_user>.keys`.
> If your existing private key is not the one published on GitHub, the
> `authorized_key` task must instead be changed to use that exact public key.

### Uninstall / rollback

The playbook is not destructive by default, but to undo specific tasks you can run individual tags (see below) or manually revert on the affected host.

---

## Repository Structure

```
.
├── site.yml                   # Main playbook — run this (7 plays: baseline, k3s, docker, k3s_master, role_1, role_2, role_3)
├── inventory/
│   └── hosts.ini              # Managed hosts + role groups (k3s_nodes / docker_nodes / k3s_master / role_1 / role_2 / role_3)
├── group_vars/
│   ├── all.yml                # Variables shared by every host
│   ├── k3s_nodes.yml          # K3S version/flags — applies only to [k3s_nodes]
│   ├── docker_nodes.yml       # Docker packages/channel — applies only to [docker_nodes]
│   ├── k3s_master.yml         # K3S control-plane flags, FluxCD/Helm versions — applies only to [k3s_master]
│   ├── role_1.yml             # Placeholder-role vars — applies only to [role_1]
│   ├── role_2.yml             # Placeholder-role vars — applies only to [role_2]
│   └── role_3.yml             # Placeholder-role vars — applies only to [role_3]
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
| `admin_github_user` | `masterlog80` | GitHub username from which SSH public keys are imported for `lorenzo` and `root` |
| `zsh_users` | `{{ admin_user }}` + `root` (name + home dir pairs) | Users who get zsh/Oh My Zsh/Powerlevel10k/plugins. Add an entry (`name`, `home`) to extend it to another user |
| `zsh_plugins` | see file — `kubectl`, `kubectx`, `kube-ps1`, `git`, `zsh-syntax-highlighting`, `zsh-autosuggestions`, `docker`, `docker-compose`, `colorize`, `last-working-dir` | Oh My Zsh plugins enabled in every `zsh_users` member's `.zshrc` |
| `zsh_external_plugins` | `zsh-syntax-highlighting`, `zsh-autosuggestions` (name + repo pairs) | Plugins not bundled with Oh My Zsh, cloned from their upstream repo instead |
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

### `group_vars/k3s_master.yml` — K3S control-plane variables

| Variable | Default | Description |
|----------|---------|-------------|
| `k3s_master_version` | `""` (latest) | K3S version to install on control-plane nodes |
| `k3s_master_install_flags` | `--disable traefik --disable servicelb --disable metrics-server` | Flags passed to the K3S installer |
| `k3s_master_taint_effect` | `NoSchedule` | Taint effect applied to `node-role.kubernetes.io/control-plane` |
| `fluxcd_version` | `latest` | FluxCD CLI version to install (e.g. `v2.3.0`) |
| `helm_version` | `latest` | Helm version to install (e.g. `v3.15.3`) |

### `group_vars/role_1.yml` / `role_2.yml` / `role_3.yml` — placeholder-role variables

| Variable | Default | Description |
|----------|---------|-------------|
| `node_role` | `role_1` / `role_2` / `role_3` | Informational tag surfaced in the placeholder task's debug message |

Each file is a stub — add real variables here as you flesh out what the role does.

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
| `packages` | all hosts | Package installation (incl. `apt --fix-broken` before/after) |
| `swap` | all hosts | Disable swap |
| `timezone` | all hosts | Set timezone |
| `ntp` | all hosts | Configure Chrony / systemd-timesyncd |
| `dns` | all hosts | Configure systemd-resolved (incl. stub-listener disable + resolv.conf repoint) |
| `ups` | all hosts | UPS udev rules (OMRON + APC) |
| `modem` | all hosts | GSM modem udev rule (Qualcomm) |
| `udev` | all hosts | All three udev rules together (GSM modem + both UPS rules) |
| `sysstat` | all hosts | Disable sysstat + clear `/var/log/sysstat/*` |
| `nvmf` | all hosts | Disable nvmf-autoconnect |
| `users` | all hosts | User management |
| `ssh` | all hosts | SSH hardening (incl. MaxAuthTries for `{{ admin_user }}`) |
| `zsh` | all hosts | Oh My Zsh + Powerlevel10k + plugins, for every user in `zsh_users` |
| `autologin` | all hosts | Getty autologin |
| `network` | all hosts | Static IP via Netplan |
| `hostname` | all hosts | Set hostname |
| `k3s` | `k3s_nodes` only | K3S installation |
| `docker` | `docker_nodes` only | Docker installation |
| `k3s_master` | `k3s_master` only | K3S control-plane install, taint, KUBECONFIG, FluxCD, Helm |
| `fluxcd` | `k3s_master` only | FluxCD install only |
| `helm` | `k3s_master` only | Helm install only |
| `role_1` | `role_1` only | Placeholder task |
| `role_2` | `role_2` only | Placeholder task |
| `role_3` | `role_3` only | Placeholder task |

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

## Adding a host to a role (or to several at once)

Assigning a host to a role is entirely an `inventory/hosts.ini` edit — you never touch `site.yml` to onboard a device, only to define a new *kind* of role (see above).

### Single role

Add the host under `[nodes]`, then under the one role group it should run. To make `node4` a Docker host:

```ini
[nodes]
node1  ansible_host=192.168.10.10  ansible_user=ubuntu
node4  ansible_host=192.168.10.13  ansible_user=ubuntu

[docker_nodes]
node3
node4
```

Then add `host_vars/node4.yml` with at least its `static_ip` (see the existing `host_vars/node1.yml` for the shape). Run `ansible-playbook site.yml --limit node4` and it gets the common baseline + Docker — nothing else.

### Multiple roles on the same host — yes, this is supported

**A single host can be listed in as many role groups as you want.** Ansible evaluates `site.yml` play by play, top to bottom, and each play's `hosts:` pattern is checked independently — so a host matching more than one group simply gets every one of those plays applied, in the order they appear in `site.yml` (baseline first, then `k3s`, `docker`, `role_1`, `role_2`, `role_3`).

Example — `node5` should run **both** K3S and `role_1`:

```ini
[nodes]
node5  ansible_host=192.168.10.14  ansible_user=ubuntu

[k3s_nodes]
node1
node2
node5

[role_1]
node5
```

Running `ansible-playbook site.yml --limit node5` applies: common baseline → K3S role → role_1's tasks, in that order, in one run. No special syntax, no combined group needed — just list the hostname under every group it belongs to.

If you *do* want a shorthand for "always run these two roles together," Ansible supports group-of-groups:

```ini
[k3s_and_role1:children]
k3s_nodes
role_1
```

This doesn't change what runs (that's still driven by `k3s_nodes` / `role_1` membership and their plays) — it just gives you `--limit k3s_and_role1` as a convenience alias for "everything currently in either group."

### Checking what a host will actually run

Before applying, you can see which groups (and therefore which plays) a host resolves to:

```bash
ansible-inventory -i inventory/hosts.ini --host node5
ansible node5 -i inventory/hosts.ini --list-hosts -m ping   # confirms it's reachable
ansible-playbook site.yml --limit node5 --list-tags          # shows which tags/plays would run
```

---

## Running from Semaphore UI

Two credentials, matching the two-phase model in "SSH access" above:

- **Bootstrap credential (first run on a new node):** User `{{ admin_user }}` + password
- **Normal credential (every run after):** User `{{ admin_user }}` + private key

Root privilege for any task comes from `become`/`sudo` inside the playbook, not from which SSH credential is used — both credentials connect as `{{ admin_user }}`, never root.

No password or private key is committed to this repository.

The role groups map directly onto task templates:

- One template running `site.yml` with no `--tags` / `--limit` set → applies baseline + whichever role each host in the inventory belongs to.
- A second template with `--tags k3s` (or a Survey Variable for the tag) → touches only `k3s_nodes`.
- A third with `--tags docker` → touches only `docker_nodes`.
- A fourth with `--tags k3s_master` → touches only `k3s_master`.

Target IP/host selection then comes from which inventory (or `--limit`/survey variable) each template is pointed at, rather than from separate playbooks.

---

## License

MIT

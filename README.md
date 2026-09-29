# ansible-playbook

An Ansible playbook that bootstraps a fleet of Debian/Ubuntu nodes from a fresh OS image to a fully configured, role-specific state — system updates, hardening, users, networking, shell setup — plus a per-node choice of **K3S**, **Docker**, both, or neither, driven entirely by inventory group membership.

> 📋 Literal, in-order list of every operation → [FLOW.md](FLOW.md)
> ⚙️ Semaphore UI setup for this repo → [SETTINGS.md](SETTINGS.md)

---

## Design

One playbook (`site.yml`), one play per role. A host's role(s) come from its inventory group — no forking the playbook per device.

| Play | Group | Does |
|---|---|---|
| Baseline | `all` | Updates, packages, swap, timezone, NTP, DNS, udev rules, disabled services, users, SSH hardening, zsh, console login, static IP, hostname |
| K3S | `k3s_nodes` | Installs K3S |
| Docker | `docker_nodes` | Installs Docker Engine + Compose |
| K3S control-plane | `k3s_master` | K3S with control-plane flags, taint, FluxCD, Helm |
| `role_1` / `role_2` / `role_3` | matching group | Placeholders — replace with real work when needed |

A host can be in any combination of groups (baseline-only is fine too). See "Adding a role" below to add another.

---

## Features

- 🔄 **Updates** – `apt update` + dist-upgrade, wrapped in `apt --fix-broken`
- 📦 **Packages** – curated list in `group_vars/all/vars.yml`
- 💾 **Swap disabled** – required for K3S, applied to every node
- ☸️ **K3S** – official installer, version/flags configurable
- ☸️ **K3S control-plane** – `--disable traefik/servicelb/metrics-server`, taints the node, installs FluxCD + Helm
- 🐳 **Docker** – official apt repo (auto-selects `ubuntu`/`debian`), Compose plugin, admin user added to the `docker` group
- 🕐 **Timezone / NTP** – Chrony or systemd-timesyncd, auto-detected or forced
- 🌐 **DNS** – `systemd-resolved` with custom servers, stub listener disabled for local-resolver compatibility
- 🔌 **udev rules** – GSM modem (Qualcomm) + UPS (OMRON, APC)
- 🔇 **Disabled services** – `sysstat` (+ its logs) and `nvmf-autoconnect`
- 👤 **User management** – deletes stray non-admin users, creates the admin user, sets root's password
- 🔒 **SSH lockdown** – key-only, no root login, `AllowUsers`, and the companion hardening that closes what those two settings alone leave open. See "Security" below
- 🔐 **Console login** – password-authenticated (no autologin)
- 🖥️ **Oh My Zsh** – Powerlevel10k + a full plugin set, for every user in `zsh_users`
- 🌐 **Static IP** – per-host Netplan config

---

## Quick Start

### Prerequisites

| Requirement | Notes |
|---|---|
| Ansible + collections | `pip install ansible && ansible-galaxy collection install -r requirements.yml` |
| Managed hosts | Debian 11+ or Ubuntu 20.04+, Python 3 at `/usr/bin/python3` |
| First SSH access | `root` + the node's built-in default password (see "Security") |

### Clone & run

```bash
git clone https://github.com/masterlog80/ansible-playbook.git
cd ansible-playbook
```

Add the host to `inventory/hosts.ini` under `[nodes]` plus whichever role group(s) it should run, and add a `host_vars/<name>.yml` (see `host_vars/node1.yml`) for its static IP/hostname:

```ini
[nodes]
node4  ansible_host=192.168.10.13  ansible_user=lorenzo

[docker_nodes]
node4
```

```bash
ansible-playbook site.yml --check --diff     # dry run
ansible-playbook site.yml                    # apply baseline + roles for every host
ansible-playbook site.yml --limit node4      # just one host
ansible-playbook site.yml --tags docker      # just one role, all hosts in it
```

---

## Security

**Two-phase auth:** a brand-new node only has a known `root` password, so bootstrap connects as `root` + that password. That one run creates the admin user (`lorenzo`), installs its GitHub SSH key, sets both passwords, and locks `sshd` down for good — after that, `root` can never SSH in again, and every later run connects as `lorenzo` + private key.

`lorenzo`'s sudo is `ALL=(ALL) ALL` + `Defaults:lorenzo rootpw` — sudo asks for the **root** password, not passwordless. Trade-off: whatever runs Ansible must hold both the SSH key and the root password, but nobody — including someone at the physical console — reaches root without knowing that password.

| sshd setting | Value |
|---|---|
| `PermitRootLogin` | `no` (after bootstrap) |
| `PasswordAuthentication`, `KbdInteractiveAuthentication`, `ChallengeResponseAuthentication`, `PermitEmptyPasswords` | `no` |
| `PubkeyAuthentication` | `yes` |
| `AllowUsers` | `lorenzo` only |

**Secrets — never committed in plaintext:**

| Variable | Lives in | Used for |
|---|---|---|
| `root_password_hash`, `admin_user_password_hash` | encrypted `group_vars/all/vault.yml` | Node passwords (crypt hashes) |
| `ansible_become_password` | `--ask-become-pass`, Semaphore's Become-password field, or the vault | sudo, every run after bootstrap |

```bash
openssl passwd -6 'the-root-password'          # generate a hash for each password
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
$EDITOR group_vars/all/vault.yml               # paste the hashes in
ansible-vault encrypt group_vars/all/vault.yml
ansible-playbook site.yml --ask-vault-pass --ask-become-pass
```

The run fails fast if either hash is missing or isn't a real crypt hash. `additional_authorized_keys` in the vault gives a fallback if GitHub is unreachable when fetching `lorenzo`'s key.

---

## Configuration

`group_vars/all/vars.yml` (shared) — key variables:

| Variable | Default | Description |
|---|---|---|
| `packages` | see file | apt packages on every node |
| `timezone` | `Asia/Tokyo` | |
| `dns_servers` / `dns_search_domain` | — | `systemd-resolved` config |
| `admin_user` / `admin_github_user` | `lorenzo` / `masterlog80` | Admin account + key source |
| `zsh_users` | `lorenzo` + `root` | Who gets Oh My Zsh |
| `subnet_prefix` / `gateway` | `24` / `192.168.10.1` | Static IP defaults |

Per-role files: `group_vars/k3s_nodes.yml` (`k3s_version`, `k3s_install_flags`), `docker_nodes.yml` (`docker_channel`, `docker_packages`), `k3s_master.yml` (control-plane flags, `fluxcd_version`, `helm_version`). `host_vars/<node>.yml` overrides `static_ip` per host.

---

## Tags

| Tag | Scope | Covers |
|---|---|---|
| `baseline` | all | everything in the common play |
| `packages`, `swap`, `timezone`, `ntp`, `dns`, `udev`, `sysstat`, `nvmf`, `users`, `ssh`, `zsh`, `console`, `network`, `hostname` | all | that one baseline section |
| `k3s` | `k3s_nodes` | K3S install |
| `docker` | `docker_nodes` | Docker install |
| `k3s_master`, `fluxcd`, `helm` | `k3s_master` | control-plane / just Flux / just Helm |
| `role_1` / `role_2` / `role_3` | matching group | placeholder task |

```bash
ansible-playbook site.yml --tags network,hostname
```

---

## Adding a role

1. Add a group to `inventory/hosts.ini`.
2. Add `group_vars/<group>.yml` for its variables.
3. Add a new play to `site.yml`, `hosts: <group>`, tagged with the role name.
4. Document it (Features, Configuration, Tags above).

A host can belong to several groups at once — every matching play runs, in the order plays appear in `site.yml`. Check what a host will run with:

```bash
ansible-inventory -i inventory/hosts.ini --host node5
ansible-playbook site.yml --limit node5 --list-tags
```

---

## Project Structure

```
.
├── ansible.cfg
├── requirements.yml
├── site.yml
├── inventory/hosts.ini
├── group_vars/
│   ├── all/{vars.yml, vault.yml.example}
│   ├── k3s_nodes.yml / docker_nodes.yml / k3s_master.yml
│   └── role_1.yml / role_2.yml / role_3.yml
├── host_vars/node1.yml ...
└── templates/{netplan,resolved,zshrc}.j2
```

---

## Linting

```bash
pip install ansible ansible-lint
ansible-playbook site.yml --syntax-check
ansible-lint   # clean at the 'production' profile
```

---

## License

MIT

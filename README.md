# ansible-copilot

> Ansible playbook for bootstrapping a fleet of Linux nodes (Debian/Ubuntu-based).

---

## What the playbook does

| # | Task |
|---|------|
| 1 | **System update** – `apt update` + dist-upgrade |
| 2 | **Package installation** – installs a curated set of packages (see `group_vars/all.yml`) |
| 3 | **Disable swap** – `swapoff -a` + removes swap entries from `/etc/fstab` |
| 4 | **K3S installation** – downloads and runs the official K3S install script |
| 5 | **Timezone** – sets the clock to `Asia/Tokyo` |
| 6 | **DNS & Domain** – configures `systemd-resolved` with DNS `192.168.10.3`, `192.168.10.4`, `1.1.1.1` and search domain `salvetti.info` |
| 7 | **UPS udev rule** – creates `/etc/udev/rules.d/99-ups.rules` for the APC UPS USB device |
| 8 | **Disable sysstat** – stops and masks the sysstat service |
| 9 | **User management** – deletes all regular (non-root) users, creates user `lorenzo` with passwordless sudo |
| 10 | **SSH key** – imports the SSH public keys from GitHub account `masterlog80` for user `lorenzo` |
| 11 | **Harden root** – locks the root password and sets `PermitRootLogin no` in `sshd_config` |
| 12 | **Oh My Zsh** – installs Oh My Zsh, Powerlevel10k theme, and plugins `zsh-syntax-highlighting`, `zsh-autosuggestions`, `colorize`, `last-working-dir`, `git` |
| 13 | **Getty autologin** – configures `/lib/systemd/system/getty@.service.d/override.conf` for automatic login as `lorenzo` |
| 14 | **Static IP** – deploys a Netplan configuration per host (defined in `host_vars/`) |

---

## Repository layout

```
.
├── site.yml                   # Main playbook – run this
├── inventory/
│   └── hosts.ini              # Managed hosts + connection settings
├── group_vars/
│   └── all.yml                # Variables shared by every host
├── host_vars/
│   ├── node1.yml              # Per-host network config for node1
│   ├── node2.yml              # Per-host network config for node2
│   └── node3.yml              # Per-host network config for node3
└── templates/
    ├── netplan.yml.j2         # Netplan static-IP template
    ├── resolved.conf.j2       # systemd-resolved DNS template
    └── zshrc.j2               # .zshrc template (Oh My Zsh + Powerlevel10k)
```

---

## Prerequisites

### Control machine (where you run Ansible)

```bash
# Python 3.8+ required
pip install ansible

# Required collections
ansible-galaxy collection install community.general ansible.posix
```

### Managed hosts

- Debian 11+ or Ubuntu 20.04+ (the playbook relies on `apt` and `netplan`)
- SSH access with a user that has `sudo` rights
- Python 3 available (`/usr/bin/python3`)

---

## Quick start

### 1. Clone this repository

```bash
git clone https://github.com/masterlog80/ansible-copilot.git
cd ansible-copilot
```

### 2. Edit the inventory

Open `inventory/hosts.ini` and add / replace hosts:

```ini
[nodes]
node1  ansible_host=192.168.10.10  ansible_user=ubuntu
node2  ansible_host=192.168.10.11  ansible_user=ubuntu
```

If your SSH key is not the default one, add `ansible_ssh_private_key_file=~/.ssh/id_rsa` to each line.

### 3. Configure per-host static IPs

Each host listed in the inventory must have a corresponding file in `host_vars/`.
Copy an existing example and adjust the values:

```bash
cp host_vars/node1.yml host_vars/mynode.yml
```

```yaml
# host_vars/mynode.yml
static_ip: 192.168.10.20
network_interface: eth0   # use `ip link` on the target to find the right name
gateway: 192.168.10.1
subnet_prefix: 24
```

### 4. Run the playbook

```bash
# Dry-run (no changes applied)
ansible-playbook -i inventory/hosts.ini site.yml --check

# Full run
ansible-playbook -i inventory/hosts.ini site.yml

# Run against a single host
ansible-playbook -i inventory/hosts.ini site.yml --limit node1

# Prompt for the sudo password instead of relying on passwordless sudo
ansible-playbook -i inventory/hosts.ini site.yml --ask-become-pass
```

---

## How to customise the playbook

### Change the package list

Edit `group_vars/all.yml` and modify the `packages` list:

```yaml
packages:
  - htop
  - btop
  # add or remove packages here
```

### Change the K3S version or install flags

In `group_vars/all.yml`:

```yaml
k3s_version: "v1.29.3+k3s1"   # pin to a specific version; leave "" for latest
k3s_install_flags: "--disable traefik"   # extra flags passed to the K3S installer
```

The K3S install script is downloaded from `https://get.k3s.io`.
If you want to use a **custom script** instead, replace the two K3S tasks in `site.yml`
with your own `ansible.builtin.script` or `ansible.builtin.shell` task.

### Change the DNS servers or search domain

In `group_vars/all.yml`:

```yaml
dns_servers:
  - 192.168.10.3
  - 192.168.10.4
  - 1.1.1.1

dns_search_domain: salvetti.info
```

### Change the timezone

In `group_vars/all.yml`:

```yaml
timezone: Asia/Tokyo   # any tz identifier accepted by timedatectl
```

### Change the admin user or GitHub key source

In `group_vars/all.yml`:

```yaml
admin_user: lorenzo
admin_github_user: masterlog80   # GitHub username whose public keys are imported
```

### Add a new host

1. Add an entry to `inventory/hosts.ini`:
   ```ini
   node4  ansible_host=192.168.10.13  ansible_user=ubuntu
   ```
2. Create `host_vars/node4.yml`:
   ```yaml
   static_ip: 192.168.10.13
   network_interface: eth0
   gateway: 192.168.10.1
   subnet_prefix: 24
   ```
3. Run the playbook with `--limit node4` to bootstrap only the new host.

### Change the static IP of an existing host

Edit the corresponding `host_vars/<hostname>.yml` and re-run the playbook.
Netplan is applied with `netplan apply`, so the change takes effect immediately
**without a reboot** (though a reboot is recommended to confirm persistence).

---

## Notes

- **Netplan files**: the playbook writes `/etc/netplan/01-ansible-static.yaml`.
  Any pre-existing DHCP configuration in other Netplan files may conflict —
  remove or rename them before running the playbook.
- **Powerlevel10k prompt**: after first login as `lorenzo`, run `p10k configure`
  to generate a `~/.p10k.zsh` personalisation file.
- **colorize plugin**: requires `python3-pygments` which is included in the
  package list.
- **K3S idempotency**: the install script is skipped if `/usr/local/bin/k3s`
  already exists. To force a re-install, remove that binary first.
- **Root login**: after disabling root password and SSH login, ensure `lorenzo`
  (or another sudoer) can reach the host before locking yourself out.

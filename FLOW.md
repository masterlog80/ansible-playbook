# FLOW.md — What `site.yml` actually does, in order

This document walks through every operation `site.yml` performs, in the exact order it
performs them. It exists so you can answer "what will running this playbook actually do
to a node?" without having to read the whole file. For *why* the playbook is structured
the way it is (common baseline + role groups, tags, adding a new role, adding a host to a
role), see [README.md](README.md) — this file is the literal, in-order operational trace,
not the design rationale.

Every operation below runs with `become: true` (root privilege for the task, obtained via
`sudo` from `{{ admin_user }}` on every run except the one-time bootstrap — see README's
"SSH access" section).

---

## PLAY 1 — System bootstrap (common baseline)

**Runs on:** every managed host, regardless of role (`hosts: all`).

### 1. System update
1. **Update apt cache** (`apt update`, cache treated as valid for 1 hour)
2. **Dist-upgrade all packages** (`apt dist-upgrade`, with `autoremove` + `autoclean`)

### 2. Package installation
3. **Fix broken packages (pre-install)** — `apt --fix-broken` equivalent, in case a previous run or manual change left packages in a broken state
4. **Install required packages** — the full list in `group_vars/all.yml`'s `packages` (includes `zsh`, `usb-modeswitch`, `ca-certificates`, `curl`, `git`, `python3-pygments`, and more — see that file for the complete list)
5. **Fix broken packages (post-install)** — same `--fix-broken` pass, run again after installation

### 3. Disable swap
6. **Disable all swap immediately** (`swapoff -a`)
7. **Remove swap entries from `/etc/fstab`** (comments them out, doesn't delete the line)
8. **Disable systemd swap units** (masks any `.swap` unit so swap can't come back via systemd)

### 4. Timezone
9. **Set timezone** to `group_vars/all.yml`'s `timezone` (default `Asia/Tokyo`)

### 5. NTP (Chrony or systemd-timesyncd — auto-detected or forced)
10. **Gather service facts** to detect which NTP implementation (if any) is already active
11. **Select NTP client automatically** — if `ntp_client: auto` (the default), keeps whichever of Chrony/systemd-timesyncd is already running, or picks Chrony if neither is
12. **Select configured NTP client** — if `ntp_client` is explicitly set instead of `auto`, uses that
13. **Validate selected NTP client** — asserts the resolved value is one of `chrony` / `systemd-timesyncd`
14. **Install Chrony** (only if Chrony was selected)
15. **Install systemd-timesyncd** (only if that was selected)
16. **Deploy Chrony configuration** (`templates/chrony.conf.j2` → `/etc/chrony/chrony.conf`; only if Chrony selected) → notifies **Restart chrony**
17. **Ensure Chrony is running and enabled** (only if selected)
18. **Deploy systemd-timesyncd configuration** (`templates/systemd-timesyncd.conf.j2`; only if that's selected) → notifies **Restart systemd-timesyncd**
19. **Ensure systemd-timesyncd is running and enabled** (only if selected)
20. **Disable systemd-timesyncd** (only if Chrony was the one actually selected, and timesyncd happens to be active)
21. **Disable Chrony** (the mirror image of the previous task, if systemd-timesyncd was selected instead)

### 6. DNS & Domain (systemd-resolved)
22. **Deploy systemd-resolved configuration** (`templates/resolved.conf.j2` → `/etc/systemd/resolved.conf`) — sets `DNS=` (from `dns_servers`), `Domains=` (from `dns_search_domain`), and `DNSStubListener=no` (frees port 53 for a local resolver like dnsmasq) → notifies **Restart systemd-resolved**
23. **Ensure systemd-resolved is running and enabled**
24. **Point `/etc/resolv.conf`** at systemd-resolved's non-stub `resolv.conf` (`/run/systemd/resolve/resolv.conf`) instead of the default stub-resolv.conf — necessary companion to step 22's `DNSStubListener=no`, otherwise DNS resolution breaks once the stub listener is off

### 7. udev rules
25. **Deploy GSM modem udev rule** (Qualcomm, `idVendor=05c6 idProduct=1000`) → `/etc/udev/rules.d/90-qualcomm.rules`, runs `usb_modeswitch` on match → notifies **Reload udev rules** + **Trigger udev**
26. **Deploy UPS OMRON udev rule** (`idVendor=0590 idProduct=0081`) → `/etc/udev/rules.d/90-omron-ups.conf`, symlinks to `/dev/ups`, group `nut` → notifies the same two handlers
27. **Deploy UPS APC udev rule** (`idVendor=051d idProduct=0002`) → `/etc/udev/rules.d/99-ups.rules`, symlinks to `/dev/ups-network` → notifies the same two handlers

### 8. Disable unwanted services
28. **Stop and disable `sysstat`**
29. **Check if `/var/log/sysstat` exists**
30. **Delete sysstat log files** (`rm -rf /var/log/sysstat/*`, only if that directory exists)
31. **Stop and disable `nvmf-autoconnect`**

### 9. User management
32. **Create admin user** `{{ admin_user }}` (`lorenzo`) — shell `/bin/zsh`, home directory created
33. **Grant `{{ admin_user }}` passwordless sudo** (`/etc/sudoers.d/lorenzo`, validated with `visudo -cf`)
34. **Add SSH public key from GitHub** for `{{ admin_user }}` — fetched from `https://github.com/<admin_github_user>.keys`
35. **Set root's shell to zsh** (for local/console use only — root never gets SSH access, see step 36+)
36. **Gather list of non-root / non-admin regular users** (any account with UID ≥ 1000 that isn't `{{ admin_user }}`)
37. **Gracefully stop processes** for users about to be deleted (`SIGTERM`)
38. **Wait briefly** (3 seconds) for those processes to exit
39. **Force-kill remaining processes** for the same users (`SIGKILL`)
40. **Remove those users and their home directories**

### 10. SSH lockdown
41. **Disable root SSH login entirely** (`PermitRootLogin no`) → notifies **Restart sshd**. After this task completes on a given node, root can never SSH in again — the *only* time root ever connects over SSH is the one-time bootstrap run on a brand-new node, before this task first executes.
42. **Disable SSH password authentication** (`PasswordAuthentication no`) → **Restart sshd**
43. **Ensure public-key authentication is enabled** (`PubkeyAuthentication yes`, explicit) → **Restart sshd**
44. **Disable keyboard-interactive authentication** (`KbdInteractiveAuthentication no` — closes a PAM gap that `PasswordAuthentication no` alone doesn't close) → **Restart sshd**
45. **Disable challenge-response authentication** (`ChallengeResponseAuthentication no`, legacy alias for the same thing, for older OpenSSH) → **Restart sshd**
46. **Disable empty-password logins** (`PermitEmptyPasswords no`) → **Restart sshd**
47. **Restrict SSH logins to `{{ admin_user }}` only** (`AllowUsers lorenzo`) → **Restart sshd**
48. **Raise `MaxAuthTries` to 20 for `{{ admin_user }}`** (a `Match User lorenzo` block appended at the true end of `sshd_config`) — avoids sshd dropping the connection with "too many authentication failures" before it reaches a valid key, which can happen when an SSH agent offers many keys → **Restart sshd**

### 11. Zsh — for every user in `zsh_users` (default: `{{ admin_user }}` and `root`)
49. **Check if Oh My Zsh is already installed** for each user
50. **Install Oh My Zsh** (official installer, `RUNZSH=no CHSH=no` so it doesn't try to switch shells interactively) — only for users where it isn't already present
51. **Clone Powerlevel10k theme** into each user's `.oh-my-zsh/custom/themes/`
52. **Clone external zsh plugins** (`zsh-syntax-highlighting`, `zsh-autosuggestions` — not bundled with Oh My Zsh) for each user
53. **Deploy `.zshrc`** (`templates/zshrc.j2`) for each user — enables the plugin set in `zsh_plugins`: `kubectl`, `kubectx`, `kube-ps1`, `git`, `zsh-syntax-highlighting`, `zsh-autosuggestions`, `docker`, `docker-compose`, `colorize`, `last-working-dir`

### 12. Getty autologin (for `{{ admin_user }}`, local console only)
54. **Ensure getty override directory exists** (`/lib/systemd/system/getty@.service.d`)
55. **Ensure getty override directory exists** (`/etc/systemd/system/getty@.service.d` — the canonical one)
56. **Configure getty autologin** (`/lib` copy) → notifies **Reload systemd**
57. **Configure getty autologin** (`/etc` copy, canonical) → **Reload systemd**

### 13. Static IP
58. **Deploy Netplan static-IP configuration** (`templates/netplan.yml.j2`, using each host's `static_ip`/`gateway`/`subnet_prefix` from `host_vars/`) → notifies **Apply netplan**

### 14. Hostname
59. **Set hostname** to each host's `hostname` var
60. **Ensure hostname is in `/etc/hosts`** (`127.0.1.1 <hostname>`)

---

## PLAY 2 — K3S role (`hosts: k3s_nodes`, tag `k3s`)

Runs after PLAY 1, only on hosts in the `[k3s_nodes]` inventory group.

1. **Download K3S install script** (`https://get.k3s.io` → `/tmp/k3s-install.sh`)
2. **Run K3S install script** — `INSTALL_K3S_VERSION` / `INSTALL_K3S_EXEC` from `group_vars/k3s_nodes.yml` (`k3s_version`, `k3s_install_flags`); skipped if `/usr/local/bin/k3s` already exists

## PLAY 3 — Docker role (`hosts: docker_nodes`, tag `docker`)

Runs after PLAY 1, only on hosts in the `[docker_nodes]` inventory group.

1. **Ensure `/etc/apt/keyrings` exists**
2. **Download Docker's official GPG key** → `/etc/apt/keyrings/docker.asc`
3. **Determine dpkg architecture**
4. **Add Docker's official apt repository** (channel from `docker_channel`, default `stable`)
5. **Install Docker Engine + Compose plugin** (`docker_packages`: `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`)
6. **Ensure Docker service is running and enabled**
7. **Add users to the `docker` group** (`docker_group_members`, default: just `{{ admin_user }}`)

## PLAY 4 — K3S control-plane / master role (`hosts: k3s_master`, tag `k3s_master`)

Runs after PLAY 3, only on hosts in the `[k3s_master]` inventory group.

1. **Download K3S install script**
2. **Run K3S install script** with control-plane flags (`k3s_master_install_flags`, default `--disable traefik --disable servicelb --disable metrics-server`)
3. **Wait for the control-plane node to be Ready** (`k3s kubectl wait --for=condition=Ready`, retried up to 6× with a 10s delay)
4. **Taint the control-plane node** (`node-role.kubernetes.io/control-plane=:NoSchedule` by default, from `k3s_master_taint_effect`, applied with `--overwrite` so re-runs are idempotent)
5. **Persist `KUBECONFIG`** in `/etc/environment` (`KUBECONFIG=/etc/rancher/k3s/k3s.yaml`)
6. **Download FluxCD install script**
7. **Run FluxCD install script** (version from `fluxcd_version`, default latest)
8. **Download Helm install script**
9. **Run Helm install script** (version from `helm_version`, default latest)

## PLAYS 5–7 — `role_1` / `role_2` / `role_3` (placeholder roles)

Each runs after PLAY 4, only on hosts in its matching inventory group (`[role_1]`, `[role_2]`, `[role_3]`). Currently each is a single `debug` task proving the pattern — see README's "Adding a new role" section for how to replace these with real work.

---

## Notes on ordering that matter

- **SSH lockdown (baseline step 41) runs *after* user creation and key import (steps 32–34).** This is deliberate: the very first run on a brand-new node connects as `root` + password (since `{{ admin_user }}` doesn't exist yet to connect as), and that run needs to create `{{ admin_user }}`, install its key, and grant it sudo *before* disabling root SSH — otherwise the node would be locked out mid-run. See README's "SSH access" section for the full two-phase credential model.
- **The `Match User {{ admin_user }}` MaxAuthTries block (step 48) is always the last thing appended to `sshd_config`.** Any future task that edits `sshd_config` must come *before* it (or use `insertafter`/an anchor to a known earlier directive) — anything appended after it via a bare `blockinfile`/`lineinfile` default (EOF) would be incorrectly scoped inside that `Match` block instead of applying globally. Step 47 (`AllowUsers`) already does this correctly, anchored with `insertafter: '^PasswordAuthentication'`.
- **A host can belong to more than one role group** (`k3s_nodes`, `docker_nodes`, `k3s_master`, `role_1`–`role_3`), and gets every matching play in the order shown above (PLAY 2 → 3 → 4 → 5 → 6 → 7). Don't put the same host in both `k3s_nodes` and `k3s_master` — whichever K3S install runs first wins the install flags (the second is skipped via its `creates:` guard).

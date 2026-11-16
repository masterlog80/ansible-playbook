# FLOW.md — What `site.yml` actually does, in order

This document walks through every operation `site.yml` performs, in the exact order it
performs them. It exists so you can answer "what will running this playbook actually do
to a node?" without having to read the whole file. For *why* the playbook is structured
the way it is (common baseline + role groups, tags, adding a new role, adding a host to a
role), see [README.md](README.md) — this file is the literal, in-order operational trace,
not the design rationale.

Every operation below runs with `become: true` (root privilege for the task, obtained via
`sudo` from `{{ admin_user }}` — authenticating with the root password — on every run except the one-time bootstrap; see README's
"SSH access" section).

---

## PLAY 1 — System bootstrap (common baseline)

**Runs on:** every managed host, regardless of role (`hosts: all`).

### 1. System update
1. **Update apt cache** (`apt update`, cache treated as valid for 1 hour)
2. **Dist-upgrade all packages** (`apt dist-upgrade`, with `autoremove` + `autoclean`)

### 2. Package installation
3. **Fix broken packages (pre-install)** — `apt --fix-broken` equivalent, in case a previous run or manual change left packages in a broken state
4. **Install required packages** — the full list in `group_vars/all/vars.yml`'s `packages` (includes `sudo`, `zsh`, `usb-modeswitch`, `ca-certificates`, `curl`, `git`, `python3-pygments`, and more — see that file for the complete list)
5. **Fix broken packages (post-install)** — same `--fix-broken` pass, run again after installation

### 3. Disable swap
6. **Disable all swap immediately** (`swapoff -a`)
7. **Remove swap entries from `/etc/fstab`** (comments them out, doesn't delete the line)
8. **List systemd swap units** (`systemctl list-units --type swap`, read-only)
9. **Mask and stop each systemd swap unit** so swap can't come back via systemd

### 4. Timezone
10. **Set timezone** to `group_vars/all/vars.yml`'s `timezone` (default `Asia/Tokyo`)

### 5. NTP (Chrony or systemd-timesyncd — auto-detected or forced)
11. **Gather service facts** to detect which NTP implementation (if any) is already active
12. **Select NTP client automatically** — if `ntp_client: auto` (the default), keeps whichever of Chrony/systemd-timesyncd is already running, or picks Chrony if neither is
13. **Select configured NTP client** — if `ntp_client` is explicitly set instead of `auto`, uses that
14. **Validate selected NTP client** — asserts the resolved value is one of `chrony` / `systemd-timesyncd`
15. **Install Chrony** (only if Chrony was selected)
16. **Install systemd-timesyncd** (only if that was selected)
17. **Deploy Chrony configuration** (`templates/chrony.conf.j2` → `/etc/chrony/chrony.conf`; only if Chrony selected) → notifies **Restart chrony**
18. **Ensure Chrony is running and enabled** (only if selected)
19. **Deploy systemd-timesyncd configuration** (`templates/systemd-timesyncd.conf.j2`; only if that's selected) → notifies **Restart systemd-timesyncd**
20. **Ensure systemd-timesyncd is running and enabled** (only if selected)
21. **Disable systemd-timesyncd** (only if Chrony was the one actually selected, and timesyncd happens to be active)
22. **Disable Chrony** (the mirror image of the previous task, if systemd-timesyncd was selected instead)

### 6. DNS & Domain (systemd-resolved)
23. **Deploy systemd-resolved configuration** (`templates/resolved.conf.j2` → `/etc/systemd/resolved.conf`) — sets `DNS=` (from `dns_servers`), `Domains=` (from `dns_search_domain`), and `DNSStubListener=no` (frees port 53 for a local resolver like dnsmasq) → notifies **Restart systemd-resolved**
24. **Ensure systemd-resolved is running and enabled**
25. **Point `/etc/resolv.conf`** at systemd-resolved's non-stub `resolv.conf` (`/run/systemd/resolve/resolv.conf`) instead of the default stub-resolv.conf — necessary companion to step 22's `DNSStubListener=no`, otherwise DNS resolution breaks once the stub listener is off

### 7. udev rules
26. **Deploy GSM modem udev rule** (Qualcomm, `idVendor=05c6 idProduct=1000`) → `/etc/udev/rules.d/90-qualcomm.rules`, runs `usb_modeswitch` on match → notifies **Reload udev rules** + **Trigger udev**
27. **Deploy UPS OMRON udev rule** (`idVendor=0590 idProduct=0081`) → `/etc/udev/rules.d/90-omron-ups.conf`, symlinks to `/dev/ups`, group `nut` → notifies the same two handlers
28. **Deploy UPS APC udev rule** (`idVendor=051d idProduct=0002`) → `/etc/udev/rules.d/99-ups.rules`, symlinks to `/dev/ups-network` → notifies the same two handlers

### 8. Disable unwanted services
29. **Stop and disable `sysstat`** (a missing service is tolerated; any other failure is not)
30. **Check if `/var/log/sysstat` exists**
31. **Find sysstat log files** (only if that directory exists)
32. **Delete sysstat log files** (each file found; the directory itself is kept)
33. **Stop and disable `nvmf-autoconnect`** (missing service tolerated, same as above)

### 9. User management
34. **Ensure password hashes are provided** — asserts `root_password_hash` and `admin_user_password_hash` exist and are crypt hashes (`$6$...`), failing fast with instructions otherwise (they come from the encrypted `group_vars/all/vault.yml`)
35. **Create admin user** `{{ admin_user }}` (`lorenzo`) — shell `/bin/zsh`, home directory created, password set from `admin_user_password_hash` (used for local console login only)
36. **Set root password** from `root_password_hash`
37. **Grant the admin user sudo** (`/etc/sudoers.d/lorenzo`, validated with `visudo -cf`): `ALL=(ALL) ALL` plus `Defaults:lorenzo rootpw` — sudo authenticates with the **root** password; there is no `NOPASSWD`
38. **Add SSH public key from GitHub** for `{{ admin_user }}` — fetched from `https://github.com/<admin_github_user>.keys` (a fetch failure is tolerated only when `additional_authorized_keys` provides fallback keys; otherwise the run stops here, before SSH password auth is disabled)
39. **Add additional SSH public keys** for `{{ admin_user }}` from `additional_authorized_keys` (no-op when the list is empty)
40. **Set root's shell to zsh** (for local/console use only — root never gets SSH access after bootstrap)
41. **Read the passwd database** (`getent`)
42. **Determine non-root / non-admin regular users to remove** — any account with UID ≥ 1000 other than `nobody` and `{{ admin_user }}` (e.g. a default `ubuntu`/`orangepi` user)
43. **Gracefully stop processes** for users about to be deleted (`SIGTERM`; "no matching process" is not an error)
44. **Wait briefly** (3 seconds) for those processes to exit
45. **Force-kill remaining processes** for the same users (`SIGKILL`; same tolerance)
46. **Remove those users and their home directories**

### 10. SSH lockdown
47. **Disable root SSH login entirely** (`PermitRootLogin no`) → notifies **Restart sshd**. After this task completes on a given node, root can never SSH in again — the *only* time root ever connects over SSH is the one-time bootstrap run on a brand-new node, before this task first executes.
48. **Disable SSH password authentication** (`PasswordAuthentication no`) → **Restart sshd**
49. **Ensure public-key authentication is enabled** (`PubkeyAuthentication yes`, explicit) → **Restart sshd**
50. **Disable keyboard-interactive authentication** (`KbdInteractiveAuthentication no` — closes a PAM gap that `PasswordAuthentication no` alone doesn't close) → **Restart sshd**
51. **Disable challenge-response authentication** (`ChallengeResponseAuthentication no`, legacy alias for the same thing, for older OpenSSH) → **Restart sshd**
52. **Disable empty-password logins** (`PermitEmptyPasswords no`) → **Restart sshd**
53. **Restrict SSH logins to `{{ admin_user }}` only** (`AllowUsers lorenzo`) → **Restart sshd**
54. **Raise `MaxAuthTries` to 20 for `{{ admin_user }}`** (a `Match User lorenzo` block appended at the true end of `sshd_config`) — avoids sshd dropping the connection with "too many authentication failures" before it reaches a valid key, which can happen when an SSH agent offers many keys → **Restart sshd**

### 11. Zsh — for every user in `zsh_users` (default: `{{ admin_user }}` and `root`)
55. **Check for the actual Oh My Zsh startup script** (`.oh-my-zsh/oh-my-zsh.sh`) for each user, rather than merely checking whether the directory exists
56. **Remove an incomplete Oh My Zsh directory** when its startup script is missing; this lets the official installer perform a clean installation
57. **Install Oh My Zsh** (official installer, `RUNZSH=no CHSH=no` so it doesn't try to switch shells interactively) only when the startup script is missing
58. **Verify the startup script exists** and fail with a clear diagnostic if installation did not complete
59. **Clone Powerlevel10k theme** (pinned to `powerlevel10k_version`) into each user's `.oh-my-zsh/custom/themes/`
60. **Clone external zsh plugins** (`zsh-syntax-highlighting`, `zsh-autosuggestions` — not bundled with Oh My Zsh; each pinned to a release tag) for each user
61. **Deploy `.zshrc`** (`templates/zshrc.j2`) for each user — enables the plugin set in `zsh_plugins`: `kubectl`, `kubectx`, `kube-ps1`, `git`, `zsh-syntax-highlighting`, `zsh-autosuggestions`, `docker`, `docker-compose`, `colorize`, `last-working-dir`

### 12. Local console login (password-authenticated)
60. **Remove getty autologin override** (`/lib/systemd/system/getty@.service.d/override.conf`, if present) → notifies **Reload systemd**
61. **Remove getty autologin override** (`/etc/systemd/system/getty@.service.d/override.conf`, if present) → **Reload systemd**

After this, the physical console shows the stock `login:` / `Password:` prompt, checked against `{{ admin_user }}`'s password hash (step 2 of section 9). SSH is unaffected — it remains key-only.

### 13. Static IP
62. **Deploy Netplan static-IP configuration** (`templates/netplan.yml.j2`, using each host's `static_ip`/`gateway`/`subnet_prefix` from `host_vars/`) → notifies **Apply netplan**

### 14. Hostname
63. **Set hostname** to each host's `hostname` var
64. **Ensure hostname is in `/etc/hosts`** (`127.0.1.1 <hostname>`)

---

## PLAY 2 — K3S role (`hosts: k3s_nodes`, tag `k3s`)

Runs after PLAY 1, only on hosts in the `[k3s_nodes]` inventory group.

1. **Download K3S install script** (`https://get.k3s.io` → `/tmp/k3s-install.sh`)
2. **Run K3S install script** — `INSTALL_K3S_VERSION` / `INSTALL_K3S_EXEC` from `group_vars/k3s_nodes.yml` (`k3s_version`, `k3s_install_flags`); skipped if `/usr/local/bin/k3s` already exists

## PLAY 3 — Docker role (`hosts: docker_nodes`, tag `docker`)

Runs after PLAY 1, only on hosts in the `[docker_nodes]` inventory group.

1. **Ensure `/etc/apt/keyrings` exists**
2. **Determine Docker apt repo distro path** — `ubuntu` on Ubuntu hosts, `debian` otherwise (Docker publishes a separate repo per distro)
3. **Download Docker's official GPG key** → `/etc/apt/keyrings/docker.asc` (from the distro-specific URL)
4. **Determine dpkg architecture**
5. **Install `python3-debian`** (required by the `deb822_repository` module)
6. **Remove legacy `docker.list`** left by earlier revisions of this playbook (avoids apt duplicate-source warnings)
7. **Add Docker's official apt repository** as a `docker.sources` deb822 file (channel from `docker_channel`, default `stable`)
8. **Install Docker Engine + Compose plugin** (`docker_packages`: `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`)
9. **Ensure Docker service is running and enabled**
10. **Add users to the `docker` group** (`docker_group_members`, default: just `{{ admin_user }}`)

## PLAY 4 — K3S control-plane / master role (`hosts: k3s_master`, tag `k3s_master`)

Runs after PLAY 3, only on hosts in the `[k3s_master]` inventory group.

1. **Download K3S install script**
2. **Run K3S install script** with control-plane flags (`k3s_master_install_flags`, default `--disable traefik --disable servicelb --disable metrics-server`)
3. **Wait for the control-plane node to be Ready** (`k3s kubectl wait --for=condition=Ready --timeout=15s`, retried up to 12× with a 10s delay — roughly a 5-minute overall budget)
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

- **SSH lockdown (baseline steps 47–54) runs *after* user creation, password setup, sudo rules and key import (steps 34–39).** This is deliberate: the very first run on a brand-new node connects as `root` + password (since `{{ admin_user }}` doesn't exist yet to connect as), and that run needs to create `{{ admin_user }}`, install its key, set the passwords and sudo rules *before* disabling root SSH — otherwise the node would be locked out mid-run. See README's "SSH access" section for the full two-phase credential model.
- **The `Match User {{ admin_user }}` MaxAuthTries block (step 54) is always the last thing appended to `sshd_config`.** Any future task that edits `sshd_config` must come *before* it (or use `insertafter`/an anchor to a known earlier directive) — anything appended after it via a bare `blockinfile`/`lineinfile` default (EOF) would be incorrectly scoped inside that `Match` block instead of applying globally. Step 53 (`AllowUsers`) already does this correctly, anchored with `insertafter: '^PasswordAuthentication'`.
- **A host can belong to more than one role group** (`k3s_nodes`, `docker_nodes`, `k3s_master`, `role_1`–`role_3`), and gets every matching play in the order shown above (PLAY 2 → 3 → 4 → 5 → 6 → 7). Don't put the same host in both `k3s_nodes` and `k3s_master` — whichever K3S install runs first wins the install flags (the second is skipped via its `creates:` guard).


## PLAY 8 — NVMe storage role (`nvme_present`)

**Runs on:** hosts explicitly listed in `[nvme_present]`. The default target is `/dev/nvme0n1`, mounted at `/mnt/K3S`.

1. Verify that the target exists and is a block device.
2. Install `parted` and `e2fsprogs`.
3. Inspect the partition table. A new GPT table is created only when the disk is blank; an existing GPT partition 1 is reused. Other layouts fail safely.
4. Create partition 1 spanning the disk only if it does not already exist.
5. Create an ext4 filesystem only if the partition has no filesystem signature; an existing filesystem is not force-formatted.
6. Check that the partition is not mounted at a different path.
7. Create `/mnt/K3S`, add/update its `/etc/fstab` entry, and ensure it is mounted with ext4 and `defaults`. Re-runs leave an already-correct mount unchanged.

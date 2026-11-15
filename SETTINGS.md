# SETTINGS.md — Configuring Semaphore UI to run this repo

This document covers the Semaphore-side setup needed to run `ansible-playbook` (this
repo, private) against your nodes: deploying Semaphore on K3S, giving it access to the
private repo, the credentials for the two-phase bootstrap model, the secret variables,
and the task templates. For what the playbook itself does, see
[README.md](README.md) and [FLOW.md](FLOW.md).

---

## 1. Deploying Semaphore on K3S

Semaphore UI's own supported install methods are Docker, a VM image, Snap, or a
Debian/RPM package — there's no first-party Helm chart. If you want it running as a pod
on K3S rather than as a plain Docker container, use a community Helm chart such as
[`cloudhippie/ansible-semaphore`](https://artifacthub.io/packages/helm/cloudhippie/ansible-semaphore),
or write a small Deployment + PVC + Service yourself from the
[official Docker image](https://hub.docker.com/r/semaphoreui/semaphore) (`semaphoreui/semaphore`).
Either way you'll need:

- A **database**: SQLite (single file, on a PVC) is enough for one person's homelab;
  Postgres/MySQL if you want more than one replica or heavier concurrent use.
- A **PVC** for Semaphore's own config/database, so task history and credentials survive
  pod restarts.
- **Ingress or a NodePort/LoadBalancer Service** to reach the UI.

This part is genuinely up to your cluster's conventions — the rest of this document
assumes you already have a reachable Semaphore instance and covers only what's specific
to running *this* repo from it.

---

## 2. Create a Project

In Semaphore: **New Project**. Name it whatever you like (e.g. `homelab`). Everything
below (Key Store entries, Repository, Inventory, Variable Group, Environment, Task
Templates) is scoped to this one project.

---

## 3. Key Store

Create these credentials under **Key Store**:

| Name | Type | Login / contents | Used for |
|------|------|------------------|----------|
| `git-deploy-key` | SSH Key | Private half of a read-only GitHub deploy key | Cloning this private repository |
| `bootstrap-root` | Login with password | Login: `root`; password: factory password (e.g. `orangepi`) | First run on a brand-new node only |
| `lorenzo-key` | SSH Key | Login: `lorenzo`; private key without a passphrase | Normal runs after bootstrap |
| `lorenzo-key-protected` | SSH Key | Login: `lorenzo`; private key and its passphrase | Optional alternative for normal runs |
| `lorenzo-sudo` | Login with password | Login: `root`; password: the new root password set by Ansible | Sudo/become after bootstrap |

You only need one Lorenzo SSH-key entry per Inventory. Store an encrypted key's passphrase
in that SSH Key credential's **Passphrase** field.

### 3a. Creating the Git deploy key

Generate a dedicated key pair on your workstation:

```bash
ssh-keygen -t ed25519 -C "semaphore-ansible-playbook" -f semaphore_deploy_key -N ""
```

Add the public half as a **read-only Deploy key** on this repository under GitHub →
Settings → Deploy keys. Paste the private half into `git-deploy-key`. Never commit the
private key.

### 3b. Two-stage node credentials

**Bootstrap:** the `bootstrap-root` credential is selected as **User Credentials** in
the Bootstrap Inventory. It connects as `root` with the factory password. Leave
**Sudo Credentials** empty because the connection is already root.

**After bootstrap:** select `lorenzo-key` (or `lorenzo-key-protected`) as **User
Credentials**, and `lorenzo-sudo` as **Sudo Credentials**. The sudo credential contains
the new root password, because the playbook configures `Defaults:lorenzo rootpw`.
It is not Lorenzo's own password.

### 3c. Which public keys are installed?

The playbook fetches all public keys published by
`https://github.com/masterlog80.keys`. If both public keys are registered in your GitHub
account, both are installed in `lorenzo`'s `authorized_keys`. Any other public key can
be added through the optional `additional_authorized_keys` variable. Never put private
keys in the repository or Variable Group.

---

## 4. Repository

**Repository → New Repository**:
- **URL:** `git@github.com:masterlog80/ansible-playbook.git`
- **Branch:** `main`
- **Access Key:** `git-deploy-key`

---

## 5. Inventories

Create **two Inventory entries**, both pointing to the same repository file but using
different credentials.

### 5a. Bootstrap Inventory (new nodes)

| Field | Value |
|------|-------|
| Name | `Bootstrap nodes` |
| User Credentials | `bootstrap-root` |
| Sudo Credentials | Leave empty |
| Type | File |
| Path to Inventory file | `inventory/hosts.ini` |
| Repository | Leave empty |

### 5b. Normal Inventory (already bootstrapped nodes)

| Field | Value |
|------|-------|
| Name | `Homelab nodes` |
| User Credentials | `lorenzo-key` (or `lorenzo-key-protected`) |
| Sudo Credentials | `lorenzo-sudo` |
| Type | File |
| Path to Inventory file | `inventory/hosts.ini` |
| Repository | Leave empty |

Leaving Repository empty makes Semaphore resolve the relative inventory path from the
repository selected in the Task Template.

Let the Inventory credential determine the SSH login. Avoid a conflicting
`ansible_user` in the inventory or host variables, or the intended bootstrap/normal
login may be overridden.

---

## 6. Variable Group — the vault secrets

Rather than shipping an `ansible-vault`-encrypted `group_vars/all/vault.yml` and giving
Semaphore the vault password, define the same variables directly as secret variables in
a **Variable Group** — simpler for a UI-driven tool, and avoids managing a vault
password as a separate credential:

**Variable Groups → New → `playbook-secrets`**, add as secret variables:

| Variable | Value |
|----------|-------|
| `root_password_hash` | Output of `openssl passwd -6 'the-root-password'` |
| `admin_user_password_hash` | Output of `openssl passwd -6 'the-lorenzo-console-password'` |

(If you'd rather keep using the encrypted `vault.yml` from the repo instead, that also
works — see README's "Secrets" section — but the Variable Group avoids the extra vault
password entirely.)

---

## 7. Environment

**Environments → New Environment → `default`**, attach the `playbook-secrets` Variable
Group created above. Task Templates reference this Environment so every run has access
to the two password hashes.

---

## 8. Task Templates

Create two Ansible Playbook templates. Both use Repository `ansible-playbook`,
Environment `default`, and Playbook Filename `site.yml`.

| Template | Inventory | Purpose |
|----------|-----------|---------|
| **Bootstrap (new node)** | `Bootstrap nodes` | Run once, limited to exactly one new node |
| **Full configuration** | `Homelab nodes` | Normal baseline + role configuration after bootstrap |

Do not use `lorenzo-key` for the first run: the account does not exist yet on a factory
node.

---

## 9. Onboarding a new node

1. Add the node to `inventory/hosts.ini` under `[nodes]` and its role group(s). Add
   `host_vars/<name>.yml` with its correct network and hostname settings.
2. Confirm the Bootstrap Inventory uses `bootstrap-root` and the node's factory password.
3. Run **Bootstrap (new node)** limited to that one hostname. The playbook creates
   `lorenzo`, sets both account passwords, grants sudo, installs the GitHub-published
   SSH keys (plus optional additional keys), then disables root SSH and SSH password login.
4. Test a fresh SSH session as `lorenzo` using the selected private key.
5. From then on, use **Full configuration** with the Normal Inventory. It connects as
   `lorenzo` and supplies the root password for sudo.

Do not run the Bootstrap template again after root SSH has been disabled. For an already
bootstrapped node, use the Normal Inventory from the start.

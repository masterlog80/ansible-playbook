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

## 3. Key Store — four entries

`ansible-playbook` is private, and the bootstrap model needs two *different* SSH
credentials (root+password, then `{{ admin_user }}`+key) plus a separate git-access
credential. Create all four under **Key Store**:

| Name | Type | Contents | Used for |
|------|------|----------|----------|
| `git-deploy-key` | SSH Key | A **read-only deploy key** for `masterlog80/ansible-playbook` (see 3a) | Cloning the repo itself |
| `bootstrap-root` | Login/Password | User `root`, password = the node's built-in default password | The one-time bootstrap run on a brand-new node |
| `lorenzo-key` | SSH Key | User `{{ admin_user }}` (`lorenzo`), the private key matching the public key published at `https://github.com/<admin_github_user>.keys`, **Become password** = the plaintext root password | Every run after bootstrap |

### 3a. Creating the git deploy key

```bash
ssh-keygen -t ed25519 -C "semaphore-ansible-playbook" -f semaphore_deploy_key -N ""
```

Add `semaphore_deploy_key.pub` as a **Deploy key** (read-only is enough) on
`masterlog80/ansible-playbook` under GitHub → repo → Settings → Deploy keys. Paste
`semaphore_deploy_key` (the private half) into the `git-deploy-key` Key Store entry.
A PAT with `repo` scope works too if you'd rather not manage a deploy key, but a
single-repo read-only deploy key is the narrower-scoped option.

### 3b. Why two SSH credentials for the nodes

This mirrors the model in README's "SSH access" section: a truly fresh node only has a
known password for `root`; `{{ admin_user }}` doesn't exist yet to have a password of
its own. The bootstrap run creates `{{ admin_user }}`, installs its key, and locks
`sshd` down (including disabling root SSH for good) — after that, only `lorenzo-key`
works, and `bootstrap-root` can never connect to that node again.

---

## 4. Repository

**Repository → New Repository**:
- **URL:** `git@github.com:masterlog80/ansible-playbook.git`
- **Branch:** `main`
- **Access Key:** `git-deploy-key`

---

## 5. Inventory

**Inventory → New Inventory**:
- **Type:** File
- **Path:** `inventory/hosts.ini` (the repo's own inventory — edit it via a PR like any
  other change, per README's "Adding a host to a role")

---

## 6. Variable Group — the vault secrets

Rather than shipping an `ansible-vault`-encrypted `group_vars/all/vault.yml` and giving
Semaphore the vault password, define the same variables directly in a **Variable Group**.

**Variable Groups → New → `playbook-secrets`**. Under **Secrets**, add both values
with the secret type set to **Variable** (Ansible extra variable), NOT **Environment
variable**. This distinction matters: environment secrets are only exported to the
process environment and do not automatically become Ansible playbook variables.

| Variable | Value |
|----------|-------|
| `root_password_hash` | **Variable** secret; output of `openssl passwd -6` |
| `admin_user_password_hash` | **Variable** secret; output of `openssl passwd -6` |

(If you'd rather keep using the encrypted `vault.yml` from the repo instead, that also
works — see README's "Secrets" section — but the Variable Group avoids the extra vault
password entirely.)

---

## 7. K3S agent token — keep it in Semaphore

The K3S agent join token must **never** be committed to this public repository or
placed in `group_vars`, inventory, or plaintext template variables.

In **Variable Groups**, create (or edit) a group named `k3s-agent-secrets`. Under
**Secrets**, add one secret with:

| Field | Value |
|---|---|
| Name | `K3S_AGENT_TOKEN` |
| Secret type | **Environment variable** |
| Value | The K3S server node's join token |

Paste the token directly into Semaphore's masked secret field. Do not put it in a
task message, extra-vars field, or shell command. Semaphore exports this secret to
the Ansible task process as `K3S_AGENT_TOKEN`; the playbook reads it with Ansible's
environment lookup and suppresses output for the installer task.

## 8. Attach the Variable Groups

Attach `playbook-secrets` to the Ansible task template under **Variable Groups**. Also attach `k3s-agent-secrets` to the agent template (and any full-run template that includes `[k3s_nodes]`).
In older Semaphore UI/API terminology this association may be labelled **Environment**.
Confirm that the template is actually using this group; otherwise Ansible will not
receive the two password hashes.

---

## 9. Task Templates

Create one template per operation. All of them use **Repository:** `ansible-playbook`,
**Inventory:** the one from step 5, **Variable Groups:** `playbook-secrets` (and `k3s-agent-secrets` for agent runs), **Playbook:**
`site.yml` — only the **SSH Key**, **tags**, and (for bootstrap) inventory override
differ:

| Template | SSH Key | Extra options |
|----------|---------|----------------|
| **Bootstrap (new node)** | `bootstrap-root` | `--limit <new-node>` (run once per new node); override the connecting user to `root` for this template |
| **Full run (baseline + roles)** | `lorenzo-key` | none — applies baseline plus whichever role(s) each host belongs to |
| **K3S agents** | `lorenzo-key` | `--tags k3s_nodes` (requires `k3s-agent-secrets`) |
| **Docker only** | `lorenzo-key` | `--tags docker` |
| **K3S control-plane only** | `lorenzo-key` | `--tags k3s_master` |

Add more templates the same way for any new role you add (see README's "Adding a new
role") — one row per tag, `lorenzo-key` credential, no other changes.

---

## 10. Onboarding a new node — the actual click-through

1. Add the node to `inventory/hosts.ini` and a matching `host_vars/<name>.yml` (PR, per
   README).
2. Run **Bootstrap (new node)**, limited to that host.
3. From then on, use **Full run** (or a role-specific template) for that node — it now
   authenticates with `lorenzo-key`, and `bootstrap-root` will no longer be able to
   reach it (root SSH is disabled by the bootstrap run itself).

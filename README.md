# doet — Nutanix VM Provisioning + ESET Deployment

Automated infrastructure pipeline for **SLTN → Doetinchem (ESET ICAP)** built
entirely with **Ansible / AWX**. All secrets (Prism credentials, sltnadmin data)
and dynamic targets (Nutanix host, Image Name) are natively injected straight
into the playbooks by AWX without needing third-party vault plugins in the code.

---

## Architecture

```
AWX Workflow Template
│
├── Job 1 ── create-vms.yml          (hosts: localhost)
│    └─ Prism Central API via nutanix.ncp
│         └─ 4× Ubuntu 24.04 VMs with cloud-init
│
├── Job 2 ── general-server-config.yml   (hosts: doet_icap, SSH)
│    └─ OS baseline: timezone, NTP verify, UFW, apt upgrade
│
└── Job 3 ── install-eset.yml            (hosts: doet_icap, SSH)
     └─ Download + unattended ESET install, enable eraagent
```

---

## Project Structure

```text
doet/
├── ansible.cfg                   # host_key_checking=False, production inventory path
├── collections/
│   └── requirements.yml          # AWX native collection dependencies
├── inventories/
│   └── production/
│       ├── hosts.yml             # doet_icap group, ansible_host IP assignments
│       └── group_vars/
│           ├── all.yml           # (formerly vm_definitions.yml) Single source of truth for VM specs & networks
│           └── doet_icap.yml     # UFW ports, ESET installer vars
├── playbooks/
│   ├── create-vms.yml            # Job 1
│   ├── general-server-config.yml # Job 2
│   ├── install-eset.yml          # Job 3
│   └── templates/
│       └── cloud-init.j2         # Jinja2 cloud-init template
├── roles/                        # (Future placeholder for abstracted tasks)
└── README.md
```

### Variable Ownership

| Namespace | File | Used by |
|---|---|---|
| `vm_*` | `inventories/production/group_vars/all.yml` | Auto-loaded for all playbooks assigned to `production` inventory |
| `ansible_*`, `eset_*`, `ufw_*` | `inventories/production/group_vars/doet_icap.yml` | Auto-loaded for Jobs 2 & 3 |
| Secrets / Dynamic Vars | AWX at runtime | Injected via Custom Credentials and Surveys |

> `group_vars/all.yml` is the **single source of truth** for all network, hardware, and VM settings. `group_vars/doet_icap.yml` holds only connection and application-level vars. Removing `vars_files:` from playbooks in favor of implicit inventory `group_vars` allows you to effortlessly spin up a `staging/` directory in the future without changing a single line of playbook code.

---

## VM Specifications

| VM Name                  | IP           | CPU           | RAM   | OS Disk | Data Disk        |
|--------------------------|--------------|---------------|-------|---------|------------------|
| doet-gropicap01-test     | 10.128.8.41  | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset |
| doet-gropicap02-test     | 10.128.8.42  | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset |
| doet-gropicap03-test     | 10.128.8.43  | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset |
| doet-gropicap04-test     | 10.128.8.44  | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset |

**Network:** subnet `NFSC-VLAN779` · gateway `10.128.8.1` · DNS `10.128.8.3/4` ·
NTP `10.128.8.3/4` (fallback `ntp.ubuntu.com`) · timezone `Europe/Amsterdam`

---

## Prerequisites

### AWX Collections (install on execution environment or AWX EE)

```bash
ansible-galaxy collection install \
  nutanix.ncp:2.4.0 \
  community.general
```

### SSH Key
Place the Ansible SSH private key at `~/.ssh/ansible-key` on the AWX execution
node (or inject via a standard AWX Machine Credential).

---

## AWX Setup

Rather than hardcoding credentials or using lookup plugins in the playbook, we instruct AWX to natively inject secrets into the playbook's execution memory as `extra_vars`. 

### 1 — Custom Credential Type (Nutanix Auth)
Create an AWX Custom Credential Type that securely injects the Prism Central credentials into `nutanix_username` and `nutanix_password`

**Input configuration (YAML)**
```yaml
fields:
  - id: nutanix_user
    label: Nutanix Username
    type: string
  - id: nutanix_pass
    label: Nutanix Password
    type: string
    secret: true
required:
  - nutanix_user
  - nutanix_pass
```

**Injector configuration (YAML)**
```yaml
extra_vars:
  nutanix_username: "{% raw %}{{ nutanix_user }}{% endraw %}"
  nutanix_password: "{% raw %}{{ nutanix_pass }}{% endraw %}"
```
*(You will bind this credential to the `doet-create-vms` Job Template).*

### 2 — Custom Credential Type (SLTN Admin)
Create another Custom Credential Type for the `sltnadmin` password hash.

**Input configuration (YAML)**
```yaml
fields:
  - id: sltnadmin_hash
    label: SLTN Admin Password Hash
    type: string
    secret: true
required:
  - sltnadmin_hash
```

**Injector configuration (YAML)**
```yaml
extra_vars:
  sltnadmin_password_hash: "{% raw %}{{ sltnadmin_hash }}{% endraw %}"
```
*(You will bind this credential to the `doet-create-vms` Job Template).*

### 3 — Dynamic Targets via AWX Surveys
Because you have multiple Nutanix servers and may want to pick images flexibly, you can override the defaults in `vars/vm_definitions.yml` dynamically:

Add a **Survey** to the `doet-create-vms` Job Template with the following prompts:
1. **Target Nutanix Host**
   - **Answer Variable Name:** `nutanix_host`
   - **Question Type:** Multiple Choice
   - **Choices:** `prism-1.doetinchem.nl`, `prism-2.doetinchem.nl`
2. **Base Image Name**
   - **Answer Variable Name:** `vm_image_name`
   - **Question Type:** Text
   - *(Note: You only need the human-readable image name. The playbook dynamically resolves this into the Prism UUID.)*
3. **Target Subnet**
   - **Answer Variable Name:** `vm_subnet_name`

### 4 — Job Templates

| # | Name                          | Playbook                     | Inventory    | Hosts      |
|---|-------------------------------|------------------------------|--------------|------------|
| 1 | `doet-create-vms`             | `playbooks/create-vms.yml`             | `production` | localhost  |
| 2 | `doet-general-server-config`  | `playbooks/general-server-config.yml`  | `production` | doet_icap  |
| 3 | `doet-install-eset`           | `playbooks/install-eset.yml`           | `production` | doet_icap  |

### 5 — Workflow Template

Create an AWX Workflow Template and chain:
```
[doet-create-vms] ──on success──► [doet-general-server-config] ──on success──► [doet-install-eset]
```
`set_stats` in Job 1 natively exports the dynamic IPs for downstream visibility in the Workflow.

---

## Cloud-Init Design Decisions

| Topic          | Decision |
|----------------|----------|
| Static IP      | Native `network:` key (Network Config v2) — **not** `write_files` + `netplan apply` |
| Gateway        | `routes: [{to: default, via: ...}]` — `gateway4` is **forbidden** in Ubuntu 24.04 |
| NTP            | Native `ntp:` key with `ntp_client: systemd-timesyncd` — single source of truth |
| chrony         | Must be **absent** (purged in Job 2 if found) |
| Data disk      | Native cloud-init `fs_setup` + `mounts` — idempotent, no `wipefs`/`mkfs` in runcmd |
| hostname/fqdn  | Derived inline from `item.name` + `vm_domain` — no redundant fields in VM list |

---

## Idempotency

| Stage          | Guard |
|----------------|-------|
| VM creation    | `ntnx_vms_v2 state: present` |
| Disk format    | `fs_setup overwrite: false` |
| ESET install   | `creates: /opt/eset/esets/sbin/eraagent` |
| UFW rules      | `community.general.ufw` is idempotent |

---

## Security Notes

- `no_log: true` on **every task** that touches fetched secrets.
- Secrets are **never written to disk** — they are seamlessly passed into Ansible memory space via AWX Custom Credentials.
- `ansible-vault` is **not needed** — AWX manages the encrypted at-rest states natively.
- SSH password auth is **disabled** (`ssh_pwauth: false` in cloud-init).
- Root login is **disabled** (`disable_root: true` in cloud-init).
- UFW default-deny inbound (ports 22 + 1344 allowed) applied **before** ESET installation.

---

## Quick Syntax Check

```bash
# Lint all playbooks
ansible-lint playbooks/*.yml

# Dry-run Job 2 against a single host using the new dynamic inventories structure
ansible-playbook playbooks/general-server-config.yml --limit doet-gropicap01-test --check
```

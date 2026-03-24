# doet — Nutanix VM Provisioning + ESET Deployment

Automated infrastructure pipeline for **SLTN → Doetinchem (ESET ICAP)** built
entirely with **Ansible / AWX**. No Terraform, no Vault — secrets are managed
exclusively by **Keeper Secrets Manager**.

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
│    └─ OS baseline: timezone, NTP, UFW, apt upgrade
│
└── Job 3 ── install-eset.yml            (hosts: doet_icap, SSH)
     └─ Download + unattended ESET install, enable eraagent
```

---

## Project Structure

```
doet/
├── ansible.cfg                   # host_key_checking=False, inventory path
├── inventory/
│   └── hosts.yml                 # doet_icap group (10.128.8.41–44)
├── group_vars/
│   └── doet_icap.yml             # non-secret group vars
├── templates/
│   ├── cloud-init.j2             # Jinja2 cloud-init template (per VM)
│   └── timesyncd.conf.j2         # systemd-timesyncd config template
├── vars/
│   └── vm_definitions.yml        # VM list, specs, network, Keeper UIDs
├── create-vms.yml                # Job 1
├── general-server-config.yml     # Job 2
└── install-eset.yml              # Job 3
```

---

## VM Specifications

| VM Name                  | IP           | CPU           | RAM   | OS Disk | Data Disk       |
|--------------------------|--------------|---------------|-------|---------|-----------------|
| doet-gropicap01-test     | 10.128.8.41  | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset|
| doet-gropicap02-test     | 10.128.8.42  | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset|
| doet-gropicap03-test     | 10.128.8.43  | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset|
| doet-gropicap04-test     | 10.128.8.44  | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset|

**Network:** subnet `NFSC-VLAN779` · gateway `10.128.8.1` · DNS `10.128.8.3/4` ·
NTP `10.128.8.3/4` (fallback `ntp.ubuntu.com`) · timezone `Europe/Amsterdam`

---

## Prerequisites

### AWX Collections (install on execution environment or AWX EE)

```bash
ansible-galaxy collection install \
  nutanix.ncp:2.4.0 \
  keeper_security.keeper_secrets_manager \
  community.general
```

### SSH Key

Place the Ansible SSH private key at `~/.ssh/ansible-key` on the AWX execution
node (or inject via AWX Machine Credential).

### Keeper Secrets Manager — Record UIDs

Update `vars/vm_definitions.yml` with the real Keeper record UIDs:

```yaml
keeper_nutanix_record_uid:   "XXXXXXXXXXXXXXXXXXXX"   # Prism Central login/password
keeper_sltnadmin_record_uid: "YYYYYYYYYYYYYYYYYYYY"   # sltnadmin SHA-512 hash + SSH pubkey
```

The **sltnadmin** Keeper record must contain:
- `field: password` → SHA-512 password hash (e.g. from `openssl passwd -6`)
- `field: keyPair`  → SSH public key (ed25519)

---

## AWX Setup

### 1 — Custom Credential Type (Keeper)

**Input configuration (YAML)**
```yaml
fields:
  - id: ksm_config
    type: string
    label: KSM Config Token
    secret: true
required:
  - ksm_config
```

**Injector configuration (YAML)**
```yaml
env:
  KSM_CONFIG: "{{ ksm_config }}"
```

### 2 — Job Templates

| # | Name                        | Playbook                     | Inventory  | Hosts      |
|---|-----------------------------|------------------------------|------------|------------|
| 1 | `doet-create-vms`           | `create-vms.yml`             | (localhost)| localhost  |
| 2 | `doet-general-server-config`| `general-server-config.yml`  | hosts.yml  | doet_icap  |
| 3 | `doet-install-eset`         | `install-eset.yml`           | hosts.yml  | doet_icap  |

All three Job Templates attach:
- The **Keeper Custom Credential** (injects `KSM_CONFIG`)
- The **Machine Credential** referencing `~/.ssh/ansible-key`

### 3 — Workflow Template

Create an AWX Workflow Template and chain:
```
[doet-create-vms] ──on success──► [doet-general-server-config] ──on success──► [doet-install-eset]
```

`set_stats` in Job 1 exports `icap_vm_ips` for downstream visibility.

---

## Cloud-Init Design Decisions

| Topic          | Decision |
|----------------|----------|
| Static IP      | Native `network:` key (Network Config v2) — **not** `write_files` + `netplan apply` |
| Gateway        | `routes: [{to: default, via: ...}]` — `gateway4` is **forbidden** in Ubuntu 24.04 |
| NTP            | Native `ntp:` key with `ntp_client: systemd-timesyncd` |
| chrony         | Must be **absent** (purged in Job 2 if found) |
| Data disk      | `/dev/sdb` formatted ext4, UUID-based fstab entry with `nofail`, mounted at `/opt/eset` |

---

## Idempotency

| Stage         | Guard |
|---------------|-------|
| VM creation   | `ntnx_vms_v2 state: present` |
| ESET install  | `creates: /opt/eset/esets/sbin/eraagent` |
| UFW rules     | `community.general.ufw` is idempotent |
| timesyncd     | Handler only restarts on config change |

---

## Security Notes

- `no_log: true` is applied to **every task** that touches fetched secrets.
- Secrets are **never written to disk** — only held in Ansible facts for the play duration.
- `ansible-vault` is **not used** — Keeper replaces it entirely.
- SSH password auth is **disabled** (`ssh_pwauth: false` in cloud-init).
- Root login is **disabled** (`disable_root: true` in cloud-init).
- UFW default-deny is applied **before** ESET installation runs.

---

## Quick Syntax Check

```bash
# Lint all playbooks
ansible-lint create-vms.yml general-server-config.yml install-eset.yml

# Dry-run Job 2 against a single host
ansible-playbook general-server-config.yml --limit doet-gropicap01-test --check
```

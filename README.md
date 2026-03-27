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

### 1. The SSH Keypair
For seamless Ansible provisioning, the pipeline splits your SSH key across two places:
- **Public Key:** Go to `inventories/production/group_vars/all.yml` and paste the public half of your SSH key into the `sltnadmin_ssh_pubkey` variable. This is perfectly safe to commit to Git. Cloud-init will burn this into the VM's `~/.ssh/authorized_keys` file when it builds.
- **Private Key:** Do **NOT** commit this! Instead, go into AWX, click **Credentials → Add**, create a regular **Machine** credential, and paste the private key into the form. You will attach this Machine Credential to Jobs 2 and 3.

### 2. The `sltnadmin` Password Hash
Cloud-init requires a salted `SHA-512` hash of the user's password, rather than cleartext.

To generate this hash safely on any Linux or Mac system, run the following Python command (replace `YourPasswordHere` with your actual secure password):
```bash
python3 -c "import crypt; print(crypt.crypt('YourPasswordHere', crypt.mksalt(crypt.METHOD_SHA512)))"
```
You will get a long string starting with `$6$`. Copy that entire string — you will paste this into AWX later as the **SLTN Admin Password Hash**.

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

### 3 — Create the Actual Credentials
Creating the _Credential Type_ only creates a blueprint. You must now use those blueprints to input your actual secrets.

1. Go to **Credentials → Add**.
2. **Name:** `Nutanix API Runtime Account` (or similar)
3. **Credential Type:** Click the magnifying glass and select the **Nutanix Auth** type you made in Step 1.
4. Fill in the **Nutanix Username** and **Nutanix Password** fields that appear.
5. Click **Save**.

Repeat this process for the **sltnadmin** hash:
1. Go to **Credentials → Add**.
2. **Name:** `doet-sltnadmin-hash`
3. **Credential Type:** Select the **SLTN Admin** type you made in Step 2.
4. Input the long SHA-512 password hash.
5. Click **Save**.

### 4 — Setup Inventory

Syncing the Git project only downloads the files. You must create an Inventory to tell AWX where to look for variables.

1. Go to **Inventories** → **Add → Add Inventory**. Name it `doet-production` and **Save**.
2. Go to the **Sources** tab inside the new inventory and click **Add**.
3. Name it `doet-production-source`.
4. **Source:** Select `Project`, then select your synced Git project.
5. **Inventory file:** Select the `inventories/production/` folder from the drop-down.
6. Check **Overwrite** and **Update on Launch**, then **Save & Sync**.
*(This automatically injects `group_vars/all.yml` and `hosts.yml` into your AWX runs).*

### 5 — Create Job Templates

You must create a Job Template for each playbook manually so AWX knows how to execute them.

1. Go to **Templates → Add → Add Job Template**.
2. **Name:** `doet-create-vms`
3. **Job Type:** Run
4. **Inventory:** Select the `doet-production` inventory you just created.
5. **Project:** Select your synced Git project.
6. **Playbook:** Click the drop-down and select `playbooks/create-vms.yml`.
7. **Credentials:** Attach both the **Nutanix API Runtime Account** and **doet-sltnadmin-hash** Custom Credentials you created in Step 3.
8. **Save**.

Repeat this process for the other two playbooks, ensuring you attach your SSH key using standard Machine Credentials instead of Nutanix Auth:
- `doet-general-server-config` (pointing to `playbooks/general-server-config.yml`)
   - **Credentials:** Attach the standard **Machine Credential** you made in the Prerequisites containing your private SSH key.
- `doet-install-eset` (pointing to `playbooks/install-eset.yml`)
   - **Credentials:** Attach the standard **Machine Credential** containing your private SSH key.

### 6 — Dynamic Targets via AWX Surveys

Because you have multiple Nutanix servers and may want to pick images flexibly, you can override the defaults using a **Survey**.

1. Open the `doet-create-vms` Job Template you just saved.
2. Click the **Survey** tab at the top.
3. Click **Add** to add the following prompts:
   - **Target Nutanix Host**
     - **Answer Variable Name:** `nutanix_host`
     - **Question Type:** Multiple Choice (Single Select)
     - **Choices:** `prism-1.doetinchem.nl`, `prism-2.doetinchem.nl`
   - **Base Image Name**
     - **Answer Variable Name:** `vm_image_name`
     - **Question Type:** Text
     - *(Note: You only need the human-readable image name. The playbook dynamically resolves this into the Prism UUID.)*
   - **Target Subnet**
     - **Answer Variable Name:** `vm_subnet_name`
4. **Save** the questions, and ensure you flip the toggle to **Survey Enabled** at the top right of the screen!

### 7 — Workflow Template

A **Workflow Job Template** connects your standalone Job Templates together so they run automatically in sequence.

1. Go to **Templates → Add → Add Workflow Job Template**.
2. **Name:** `Doetinchem ESET Deployment Pipeline`
3. Select your **Organization** and click **Save**.
4. The **Workflow Visualizer** will automatically open. This is where you draw the sequence.
5. Click **Start**, then select the `doet-create-vms` Job Template from the list, and click **Save**.
6. Hover over the `doet-create-vms` node you just placed, and click the **(+)** icon to add the next step.
   - Set the **Run Type** condition to **On Success**.
   - Select the `doet-general-server-config` Job Template and click **Save**.
7. Hover over the `doet-general-server-config` node, click **(+)**, and set the **Run Type** condition to **On Success**.
   - Select the `doet-install-eset` Job Template and click **Save**.
8. Once your visualizer looks like a chain of 3 linked boxes, click **Save** at the top right to close the visualizer.

When you click **Launch** on your Workflow Template, it will:
1. Fire Job 1 to create the servers.
2. Natively export checking statuses and the IP endpoints (`set_stats`).
3. Fire Job 2 completely autonomously using SSH.
4. Fire Job 3 completely autonomously to deploy the final Software.

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

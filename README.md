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
│    └─ OS baseline: timezone, NTP verify, Nutanix Guest Tools (NGT), UFW, apt upgrade
│
└── Job 3 ── install-eset.yml            (hosts: doet_icap, SSH)
     └─ Download + unattended ESET install, enable eraagent
```

---

## Project Structure

```text
doet/
├── ansible.cfg                   # host_key_checking=False, test inventory path by default
├── collections/
│   └── requirements.yml          # AWX native collection dependencies
├── inventories/
│   ├── production/               # Production IPs & UUIDs
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── all.yml
│   └── test/                     # Test IPs & UUIDs
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml
├── playbooks/
│   ├── create-vms.yml
│   ├── general-server-config.yml
│   ├── install-eset.yml
│   ├── group_vars/
│   │   └── doet_icap.yml         # Shared App Config (ESET, UFW, SSH)
│   └── templates/
│       └── cloud-init.j2
└── README.md
```

### Variable Ownership

| Namespace | File | Used by |
|---|---|---|
| `vm_*` | `inventories/*/group_vars/all.yml` | Environment-specific network & hardware specs |
| `ansible_*`, `eset_*`, `ufw_*` | `playbooks/group_vars/doet_icap.yml` | Shared application/OS config for the `doet_icap` group |
| Secrets / Dynamic Vars | AWX at runtime | Injected via Custom Credentials and Surveys |

> **Single Source of Truth:** `all.yml` contains only variables that *change* per environment (UUIDs, Subnets, IPs). `doet_icap.yml` contains application variables that *remain the same* (UFW ports, ESET installers). This layout keeps the environment folders minimal and easy to clone for new projects.
>
> **Native Precedence:** Ansible allows you to selectively override these shared values. If a specific environment (e.g., Production) ever requires a different SSH private key path or unique ESET URL, simply recreate `doet_icap.yml` inside that environment's `group_vars/` folder. Inventory variables take precedence over playbook-level defaults.

---

## Test Environment Specifications

**Prism Central URL:** [slcd-waatnpc01.dpb.sltncloud.local](https://slcd-waatnpc01.dpb.sltncloud.local:9440) (adjustable via vars)

| VM Name                  | IP             | CPU           | RAM   | OS Disk | Data Disk        |
|--------------------------|----------------|---------------|-------|---------|------------------|
| doet-gropicap01-test     | 10.128.8.41    | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset |
| doet-gropicap02-test     | 10.128.8.42    | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset |
| doet-gropicap03-test     | 10.128.8.43    | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset |
| doet-gropicap04-test     | 10.128.8.44    | 2s × 3c = 6c  | 24 GB | 50 GB   | 300 GB /opt/eset |

**Network:** subnet `SBHP-Servers` (`7421ee02-9b91-4a57-946b-b91f756fedb9`) · gateway `10.128.8.1` · DNS `10.128.8.3 / 10.128.8.4` ·
NTP `10.128.8.3 / 10.128.8.4` (fallback `ntp.ubuntu.com`) · timezone `Europe/Amsterdam`

**Nutanix UUIDs (Test):**
- **Cluster:** `000648fa-6f4b-66b0-60a2-4cd98f907be6`
- **Base Image:** `29825f72-f2fb-449e-b7aa-9481a955fa2b` (noble-server-cloudimg-amd64.img)
- **Subnet eth0 (SBHP-Servers):** `7421ee02-9b91-4a57-946b-b91f756fedb9`

---

## Production Environment Specifications

**Prism Central URL:** [doet-gropnpc01.doetinchem-sc.sltncloud.local](https://doet-gropnpc01.doetinchem-sc.sltncloud.local:9440) (adjustable via vars)

> **Dual-NIC layout** — each VM has two network interfaces:
> - **eth0** (`DOET:DOET:EPG-DOET-vSphere-DCU VLAN 0`, `10.130.100.x`) — external / internet-facing. Carries the default route (`10.130.100.1`) and all DNS. **Ansible SSH connects on this interface.**
> - **eth1** (`NFSC VLAN 779`, `10.128.40.x`) — internal only. Traffic between Nutanix and ESET flows here. **No default route. No internet access.**

| VM Name              | eth0 IP (external / Ansible) | eth1 IP (internal) | CPU           | RAM   | OS Disk | Data Disk        |
|----------------------|------------------------------|--------------------|---------------|-------|---------|------------------|
| doet-gropicap01-prod | 10.130.100.191               | 10.128.40.31       | 2s × 6c = 12c | 24 GB | 50 GB   | 300 GB /opt/eset |
| doet-gropicap02-prod | 10.130.100.192               | 10.128.40.32       | 2s × 6c = 12c | 24 GB | 50 GB   | 300 GB /opt/eset |
| doet-gropicap03-prod | 10.130.100.193               | 10.128.40.33       | 2s × 6c = 12c | 24 GB | 50 GB   | 300 GB /opt/eset |
| doet-gropicap04-prod | 10.130.100.194               | 10.128.40.34       | 2s × 6c = 12c | 24 GB | 50 GB   | 300 GB /opt/eset |

**eth0 — External Network:**
Subnet `DOET:DOET:EPG-DOET-vSphere-DCU (VLAN 0)` (`26d75bac-450c-4f99-99c7-19397cec1807`) · gateway `10.130.100.1` · prefix `/24`
DNS `172.20.10.1 / 172.16.10.1` · NTP `172.20.10.1 / 172.16.10.1` (fallback `ntp.ubuntu.com`) · timezone `Europe/Amsterdam`

**eth1 — Internal Network:**
Subnet `NFSC (VLAN 779)` (`e16571d2-0199-4bef-8c06-13525de192db`) · no gateway (internal-only) · prefix `/24`

**Nutanix UUIDs (Production):**
- **Cluster:** `00062f9b-7a59-04cd-4837-1423f3232900`
- **Base Image:** `ed2a849c-5a85-49ed-ad51-c7c1f828334b` (noble-server-cloudimg-amd64.img)
- **Subnet eth0 (EPG-DOET-vSphere-DCU):** `26d75bac-450c-4f99-99c7-19397cec1807`
- **Subnet eth1 (NFSC):** `e16571d2-0199-4bef-8c06-13525de192db`

---

## Prerequisites

### AWX Prerequisites (install on execution environment or AWX EE)

**Ansible Collections:**
```bash
ansible-galaxy collection install \
  nutanix.ncp:2.4.0 \
  community.general
```

**Python Libraries:**
```bash
pip install \
  ntnx-v3-api-client \
  ntnx-vmm-py-client
```

### 1. The SSH Key-Pair

Authentication for the `sltnadmin` account should be handled via SSH keys rather than passwords. An **ED25519** key-pair is recommended for its high security and performance.

#### Generate the Key-Pair
Run this command on your management workstation:
```bash
ssh-keygen -t ed25519 -C "sltnadmin@doet-icap" -f ./id_ed25519_sltnadmin
```

#### Key Placement
For seamless Ansible provisioning, the pipeline splits your SSH key across two places:
-   **Public Key (`id_ed25519_sltnadmin.pub`)**: Copy the contents of this file and paste it into the `sltnadmin_ssh_pubkey` variable in `playbooks/group_vars/all.yml` (or your environment-specific file). Cloud-init will burn this into the VM's `~/.ssh/authorized_keys` file at boot.
-   **Private Key (`id_ed25519_sltnadmin`)**: **Do NOT commit this to Git!** Instead, upload it as a **Machine Credential** in AWX. You will attach this credential to the configuration and health-check jobs.

### 2. The `sltnadmin` Password Hash
Cloud-init requires a salted `SHA-512` hash of the user's password, rather than cleartext.

To generate this hash safely on any Linux or Mac system, use one of the following methods (replace `YourPasswordHere` with your actual secure password):

**Method A: Python (Universal)**
```bash
python3 -c "import crypt; print(crypt.crypt('YourPasswordHere', crypt.mksalt(crypt.METHOD_SHA512)))"
```

**Method B: mkpasswd (Common on Linux)**
```bash
echo "YourPasswordHere" | mkpasswd -m sha-512 -s
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
  nutanix_username: "{{ nutanix_user }}"
  nutanix_password: "{{ nutanix_pass }}"
```
*(You will bind this credential to the `doet-create-vms` Job Template).*

### 2 — Custom Credential Type (SLTN Admin)

This credential type securely injects the `sltnadmin` login secrets. It is designed to handle both **SSH Key-based** and **Password-based** authentication, ensuring connectivity even when `ssh_pwauth` is disabled.

**Input configuration (YAML)**
```yaml
fields:
  - id: admin_password
    label: SLTN Admin Cleartext Password
    type: string
    secret: true
  - id: admin_key
    label: SLTN Admin SSH Private Key
    type: string
    multiline: true
    secret: true
  - id: admin_hash
    label: SLTN Admin Password Hash (SHA-512)
    type: string
    secret: true
required:
  - admin_hash
```

**Injector configuration (YAML)**
```yaml
extra_vars:
  ansible_user: "sltnadmin"
  ansible_password: "{{ admin_password }}"
  ansible_ssh_private_key_file: "{{ admin_key }}"
  sltnadmin_password_hash: "{{ admin_hash }}"
```

> [!NOTE]
> If `ssh_pwauth` is set to `false`, the **SSH Private Key** field must be populated for Ansible to connect. If `true`, the **Cleartext Password** will be used for authentication.

### 3 — Create the Actual Credentials

Creating the _Credential Type_ only creates a blueprint. You must now use those blueprints to input your actual secrets. Since the test and production environments have different credentials, you should create one for each.

**For the Test Environment:**
1. Go to **Credentials → Add**.
2. **Name:** `Nutanix API - Test`
3. **Credential Type:** Select **Nutanix Auth**.
4. Fill in the **Nutanix Username** and **Nutanix Password** for the test environment.
5. Click **Save**.

**For the Production Environment:**
1. Go to **Credentials → Add**.
2. **Name:** `Nutanix API - Production`
3. **Credential Type:** Select **Nutanix Auth**.
4. Fill in the **Nutanix Username** and **Nutanix Password** for the production environment.
5. Click **Save**.

Repeat this process for the **sltnadmin** hash (create one for each environment if they differ, or one if it's the same).

### 4 — Setup Inventories (Test & Production)

Syncing the Git project only downloads the files. You must create two AWX Inventories so the pipeline knows how to map to both environments cleanly.

**First: Create the Test Inventory**
1. Go to **Inventories** → **Add → Add Inventory**.
2. **Name:** `doet-test` and click **Save**.
3. Go to the **Sources** tab inside the new inventory and click **Add**.
4. **Name:** `test-source`. **Source:** `Project` (Select your synced Git project).
5. **Inventory file:** Select `inventories/test/` from the drop-down.
6. Check **Overwrite** and **Update on Launch**, then **Save & Sync**.

**Second: Create the Production Inventory**
1. Go to **Inventories** → **Add → Add Inventory**.
2. **Name:** `doet-production` and click **Save**.
3. Go to the **Sources** tab and click **Add**.
4. **Name:** `production-source`. **Source:** `Project` (Select your synced Git project).
5. **Inventory file:** Select `inventories/production/` from the drop-down.
6. Check **Overwrite** and **Update on Launch**, then **Save & Sync**.

*(AWX can now invisibly swap all hardcoded IPs and specs just by alternating between these two!)*

### 5 — Create Job Templates

Rather than creating 6 duplicate playbooks for Test and Prod, you only need to create **3 Job Templates** and tell AWX to prompt you for the environment!

1. Go to **Templates → Add → Add Job Template**.
2. **Name:** `doet-create-vms`
3. **Job Type:** Run
4. **Inventory:** Select `doet-test` (as a placeholder), but immediately click the **Prompt on Launch** checkbox right next to it! This ensures AWX asks you whether to deploy to Test or Prod.
5. **Project:** Select your synced Git project.
6. **Playbook:** Click the drop-down and select `playbooks/create-vms.yml`.
7. **Credentials:** 
   - Click the glass and select **BOTH** the `Nutanix API` and `sltnadmin-hash` categories.
   - For each one, **CHECK the "Prompt on Launch" box** right next to the category label.
   - (Alternatively, you can select specific ones as defaults, but checking Prompt ensures you always select the right one for the chosen environment).
8. **Save**.

**Now, create the second Job Template:**
1. Go to **Templates → Add → Add Job Template**.
2. **Name:** `doet-general-server-config`
3. **Job Type:** Run
4. **Inventory:** Select `doet-test` (as a placeholder), and **CHECK the "Prompt on Launch" box** next to the Inventory field.
5. **Project:** Select your synced Git project.
6. **Playbook:** Select `playbooks/general-server-config.yml`.
7. **Credentials:** Attach the standard **Machine Credential** you made in the Prerequisites containing your private SSH key.
8. **Save**.

**Finally, create the third Job Template:**
1. Go to **Templates → Add → Add Job Template**.
2. **Name:** `doet-install-eset`
3. **Job Type:** Run
4. **Inventory:** Select `doet-test` (as a placeholder), and **CHECK the "Prompt on Launch" box** next to the Inventory field.
5. **Project:** Select your synced Git project.
6. **Playbook:** Select `playbooks/install-eset.yml`.
7. **Credentials:** Attach the standard **Machine Credential** containing your private SSH key.
8. **Save**.

### 6 — Dynamic Targets via AWX Surveys

All hardware specs and subnets are hardcoded inside the `inventories/` folders so they perfectly match their environments. The **only** thing we prompt for dynamically is the Image you wish to deploy.

1. Open the `doet-create-vms` Job Template you just saved.
2. Click the **Survey** tab at the top.
3. Click **Add** to add the following prompt:
   - **Base Image Name**
     - **Answer Variable Name:** `vm_image_name`
     - **Question Type:** Text
     - *(Note: The playbook dynamically resolves this into the Prism UUID.)*
   - **Base Image UUID** (Optional alternative)
     - **Answer Variable Name:** `vm_image_uuid`
     - **Question Type:** Text
4. **Save** the questions, and ensure you flip the toggle to **Survey Enabled** at the top right of the screen!
5. **CRUCIAL STEP:** Go back to the **Details** tab of the `doet-create-vms` Job Template, click Edit, and check the **Prompt on Launch** checkbox under the "Variables" or "Survey" section.

#### How to find an Image UUID in Nutanix manually
If the automatic `vm_image_name` lookup fails, you can bypass it using `vm_image_uuid`. Here is how to find your image's exact UUID:

**Method 1: Prism Central Web UI**
1. Log in to your Prism Central interface.
2. Navigate to **Compute & Storage** → **Images**.
3. Click on the name of the image you want to use.
4. Look at the URL in your browser address bar. It will look like: 
   `https://prism-central-ip/infrastructure/virtual_infrastructure/images/5d1b...-....-....-....`
5. The very last string of text in the URL is the UUID (`ext_id`).

**Method 2: Command Line (CVM)**
1. SSH into any Nutanix CVM as the `nutanix` user.
2. Run `acli image.list`
3. Copy the UUID next to the name of your image.

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

When you click **Launch** on your Workflow Template, AWX will ask you three things:
1. **Which Inventory?** — Choose either the `doet-production` or `doet-test` inventory. Everything else (subnets, IPs, hostnames) is instantly loaded from the `group_vars` linked to that choice! 
2. **Which Credentials?** — Choose the Nutanix and sltnadmin credentials that correspond to the environment you selected in step 1.
3. **Survey Prompts** — It will ask you for the `vm_image_name` (or `vm_image_uuid`).

After you hit Next:
1. Fire Job 1 to create the servers.
2. Natively export checking statuses and the IP endpoints (`set_stats`).
3. Fire Job 2 completely autonomously using SSH.
4. Fire Job 3 completely autonomously to deploy the final Software.

---

## Ansible CLI Usage

While this project is optimized for AWX, you can also execute the provisioning pipeline directly from your local terminal using the Ansible CLI.

### 1. Prerequisites
Ensure you have the Nutanix collection installed:
```bash
ansible-galaxy collection install nutanix.ncp:2.4.0
```

### 2. Execution

The execution is split into two stages. Step 1 (Provisioning) always uses the Nutanix API, while Steps 2 & 3 (Configuration) allow you to choose your authentication method.

#### Step 1: Provision the VMs (API-only)
```bash
ansible-playbook -i inventories/test/hosts.yml playbooks/create-vms.yml \
  -e "nutanix_username=YOUR_PC_USER" \
  -e "nutanix_password=YOUR_PC_PASS" \
  -e "sltnadmin_password_hash='YOUR_SHA512_HASH'"
```

#### Step 2: Configure & Verify (Choose your Auth)

**Option A: Using SSH Key-Pairs (Recommended)**
```bash
ansible-playbook -i inventories/test/hosts.yml playbooks/general-server-config.yml \
  --private-key ~/.ssh/id_ed25519_sltnadmin \
  -u sltnadmin
```

**Option B: Using Passwords**
```bash
ansible-playbook -i inventories/test/hosts.yml playbooks/general-server-config.yml \
  -u sltnadmin \
  -k -K
```
*(Note: `-k` prompts for the SSH password; `-K` prompts for the sudo password.)*

---

## Cloud-Init Design Decisions

| Topic          | Decision |
|----------------|----------|
| Static IP      | Custom Netplan via `write_files` + `netplan apply` in `runcmd` (bypasses schema validation) |
| Dual NIC       | `eth0` = external with default route + DNS matched on `ens3` (in dual-nic); `eth1` = internal only (no route, no DNS) matched on `ens4` — both set-name renamed in a single Netplan file |
| Gateway        | `routes: [{to: default, via: ...}]` on **eth0 only** — `gateway4` is **forbidden** in Ubuntu 24.04 |
| NTP            | Native `ntp:` key + `timesyncd` — correctly synced with `runcmd` task |
| Data disk      | Native cloud-init `fs_setup` + `mounts` — idempotent, no `wipefs`/`mkfs` in runcmd |
| hostname/fqdn  | Derived inline from `item.name` + `vm_domain` — no redundant fields in VM list |
| NGT Install    | Provision an additional empty `CDROM` in the disk spec. `cloud-init` claims the first drive (`/dev/sr0`); NGT requires the second empty drive (`/dev/sr1`) to successfully mount from Prism Central. |

---

## Post-Deployment ICAP Configuration

Once the Ansible pipeline finishes, the servers are built, firewalled, and running ESET. However, ICAP must be manually activated on both ends to connect them.

### 1. Activating ICAP on ESET
By default, ESET Server Security for Linux installs with the ICAP service turned off. You must enable it via your ESET PROTECT central console (recommended) or the local WebGUI:
1. Log into your **ESET PROTECT** console.
2. Select your newly deployed linux servers (`doet-gropicap...`) or their group.
3. Edit their Configuration Policy.
4. Navigate to **Server** → **ICAP**.
5. Toggle **Enable ICAP server** to **On**.
6. Ensure the listening port is set to `1344` (this is the port we automatically allowed through UFW via Ansible).
7. Apply the policy. ESET will now listen on port 1344 for incoming files.

### 2. Connecting Nutanix Files to ESET (Prism Central)
Now you must tell your Nutanix File Server to send files to your new ESET machines for virus scanning.
1. Log into **Prism Central**.
2. Navigate to **Infrastructure** → **Storage** → **File Server** (this may vary slightly depending on your AOS/PC version).
3. Select the target File Server and click **Update** (or select it and click **Antivirus**).
4. Navigate to the **Antivirus** or **ICAP** configuration tab.
5. Check the box to **Enable Antivirus**.
6. Under **ICAP Servers**, click **+ Add ICAP Server**.
7. Input the **internal IP addresses** of your deployed VMs and Port `1344`:
   - Nutanix communicates with ESET exclusively over the internal NIC (`eth1` in Production, `eth0` in Test).
   - Use `10.128.40.31` through `10.128.40.34` for Production (not the eth0 IPs).
8. Check the box to allow Nutanix to block/quarantine infected files based on ESET's response.
9. Click **Save / Update**. Nutanix will verify the connection to the ESET VMs.

---

## Expanding Disks in the Future

Both disks are designed to be trivially expandable if you ever run low on space!

### Expanding the OS Disk (50GB)
The Ubuntu cloud image is built with `growpart` natively active!
1. Go into Prism Central.
2. Edit the VM and increase the OS **Disk 1** (SCSI 0) size (e.g., to 80GB).
3. **Reboot the VM**. Cloud-init will automatically detect the new space, grow the partition, and expand the filesystem during boot. No manual CLI work required!

### Expanding the `/opt/eset` Disk (300GB)
Because the Ansible playbook formats this secondary disk as a raw `ext4` filesystem directly onto `/dev/sdb` (without creating rigid partition boundaries), expanding it is incredibly easy and can be done **entirely online**.

1. Go into Prism Central.
2. Edit the VM and increase the ESET **Disk 2** (SCSI 1) size (e.g., to 500GB).
3. If Linux doesn't pick up the new size automatically, trigger a rescan:
   ```bash
   echo 1 | sudo tee /sys/class/block/sdb/device/rescan
   ```
4. Expand the filesystem:
   ```bash
   sudo resize2fs /dev/sdb
   ```
It instantly grabs 100% of the newly added Nutanix space natively without any downtime.

---

## Idempotency

| Stage          | Guard |
|----------------|-------|
| VM creation    | `ntnx_vms_v2 state: present` |
| Disk format    | `fs_setup overwrite: false` |
| NGT install    | `ansible.builtin.stat: /usr/local/sbin/ngt_cli` |
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

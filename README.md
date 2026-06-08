# rhis-builder-bootstrap-init

Generates OEMDRV kickstart files to bootstrap RHEL 9 baremetal and virtual hosts
for the **rhis-builder** family of projects.

Kickstart files are delivered either directly to a USB-mounted OEMDRV volume
(lab method) or as ISO9660 images for iDRAC, iLO, or Redfish virtual media
(datacenter method). Disconnected (air-gapped) installations are supported
via the `disconnected: true` per-host flag, which skips CDN registration.

Supported host roles: `provisioner`, `idm`, `satellite`, `kvm`

> **Migrating from rhis-builder-baremetal-init?**
> This repository replaces the `bootstrap_init` role that was previously housed in
> [rhis-builder-baremetal-init](https://github.com/parmstro/rhis-builder-baremetal-init).
> The variable format is identical. Simply update your clone URL and `BAREMETAL_INIT`
> path references to point here.

---

## RHIS Infrastructure Bootstrap Workflow

This repository is **Phase 1** in the RHIS infrastructure lifecycle:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  rhis-builder-inventory                                                 │
│  Generate deployment configs (inventory, group_vars, host_vars, vault)  │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  rhis-builder-bootstrap-init   ← THIS REPO                             │
│  Generate OEMDRV kickstart files for each baremetal host                │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Baremetal Install                                                      │
│  Anaconda reads ks.cfg from OEMDRV → unattended RHEL 9 install          │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  rhis-provisioner container                                             │
│  Configure IdM (Phase 2) → Satellite (Phase 3) → AAP (Phase 4)         │
└─────────────────────────────────────────────────────────────────────────┘
```

> **Note:** All unified deployment configurations — group_vars, host_vars,
> inventory, and vault templates — are managed through
> [rhis-builder-inventory](https://github.com/parmstro/rhis-builder-inventory).
> Start there before running this repository.

---

## Prerequisites

### Provisioner node

The provisioner is the RHEL 9 system you run this playbook from. It can be the
system you are also bootstrapping (if you are rebuilding it), a laptop, or an
existing management node.

- RHEL 9 with `ansible-core` 2.14 or later
- `community.general` collection (installed via `requirements.yml`)
- Root or sudo access to mount the OEMDRV volume
- Network connectivity to the internet (for CDN registration in the kickstart — not required for disconnected environments)

Install the required collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

### Red Hat account

- Active Red Hat subscription
- CDN organization ID
- Activation key configured for RHEL 9 minimal install

> **Disconnected environments:** CDN registration is skipped for hosts with
> `disconnected: true`. No Red Hat account credentials are needed in the host
> entry for those hosts.

### SSH key pair

An SSH key pair for the automation user (`ansiblerunner` by convention). The
public key will be embedded in the kickstart so the provisioner can reach the
bootstrapped hosts immediately after install.

### Vault file

Sensitive values (passwords, keys, org credentials) must be stored in an
Ansible Vault encrypted file. See [Vault variables](#vault-variables) below.

---

## Workflow

### Step 1: Generate your deployment inventory

Use [rhis-builder-inventory](https://github.com/parmstro/rhis-builder-inventory)
to generate a consistent set of group_vars, host_vars, inventory, and vault
templates for your deployment domain.

**Do not clone rhis-builder-inventory directly.** Cloning leaves the upstream
remote in place and risks pushing your environment-specific configuration back
to the template repository. Instead, download the archive and initialise a fresh
repository pointed at your own remote:

```bash
wget https://github.com/parmstro/rhis-builder-inventory/archive/refs/heads/main.zip
unzip main.zip
mv rhis-builder-inventory-main rhis-builder-inventory
cd rhis-builder-inventory

git init -b main
git add --all
git commit -m "Initial commit"

# Point origin at your own repository — not the upstream template
git remote add origin https://github.com/<your_org_or_login>/rhis-builder-inventory.git
git remote -v
git push -u origin main
```

Copy and edit the base variables file for your domain:

```bash
cp inventory_basevars.yml your-domain_inventory_basevars.yml
```

Edit `your-domain_inventory_basevars.yml` to set your domain name, network
ranges, timezone, and the number of hosts of each type:

```yaml
global_domain_name: "your.domain"

default_network: "192.168.0.0"
default_network_prefix: "22"
default_network_mask: "255.255.252.0"

rhis_timezone: "America/New_York"
rhis_locale: "en"

rhis_aap_release_version: "2.6"

rhis_system_count:
  satellite: 1
  idm: 2
  aapcontroller: 1
  aaphub: 1
  capsule: 1
  quadlet: 2
```

Generate the deployment:

```bash
./inventory_update.sh -b your-domain_inventory_basevars.yml
```

This creates `deployments/your.domain/` containing:

```
deployments/your.domain/
├── group_vars/          # Group-level Ansible variables
├── host_vars/           # Per-host variable files
├── inventory/           # Ansible inventory
├── templates/           # Jinja2 configuration templates
├── vault_SAMPLES/       # Vault variable templates (copy and encrypt)
└── vars/                # Additional variable files
```

### Step 2: Prepare your vault file

Copy the vault sample and populate it with your secrets:

```bash
cp deployments/your.domain/vault_SAMPLES/rhis_builder_vault_SAMPLE.yml.j2 \
   /path/to/secure/vault/rhis_builder_vault.yml
```

At minimum, populate the following variables for this repository:

```yaml
encrypted_root_pass_vault:      # Generated with openssl passwd -6 (see below)
encrypted_grub_pass_vault:      # Generated with grub2-mkpasswd-pbkdf2 (see below)
encrypted_user_pass_vault:      # Generated with openssl passwd -6
user_sudoer_policy_vault:       # e.g. "ansiblerunner ALL=(ALL) NOPASSWD: ALL"
ssh_pub_key_vault:              # Contents of ~/.ssh/id_*.pub
cdn_organization_vault:         # Your Red Hat CDN organization ID
cdn_activation_key_vault:       # Your activation key name
```

> For disconnected hosts (`disconnected: true`), the `cdn_organization_vault`
> and `cdn_activation_key_vault` fields are not referenced in the kickstart and
> do not need to be set in the host entry.

Encrypt the vault file:

```bash
ansible-vault encrypt /path/to/secure/vault/rhis_builder_vault.yml
```

> **Note:** The variables listed above cover kickstart generation only. The full
> RHIS deployment (IdM, Satellite, AAP) requires many additional vault variables.
> See the `README.md` in
> [rhis-builder-inventory](https://github.com/parmstro/rhis-builder-inventory)
> for the complete vault variable reference and the `vault_SAMPLES/` directory
> in your generated deployment for the full template.

> **Security note:** Keep vault files outside of your project directory.
> Ensure `.gitignore` excludes any vault or credential files before committing.
> Consider enabling [secret scanning](https://docs.github.com/en/code-security/secret-scanning)
> on your repository.

### Step 3: Configure host variable files

Clone this repository and set up your host configuration files under
`group_vars/provisioner/`. The `.gitignore` excludes these files from commits.

```bash
git clone https://github.com/parmstro/rhis-builder-bootstrap-init.git
cd rhis-builder-bootstrap-init
mkdir -p group_vars/provisioner
```

Copy the sample and create one file per host group (or combine all hosts in one file):

```bash
cp bootstrap_init_vars.SAMPLE.yml group_vars/provisioner/your_init_vars.yml
```

Edit the file to define your hosts. Each entry in `bootstrap_init_hosts` generates
one kickstart file. See [Variable reference](#variable-reference) for all fields.

### Step 4: Generate kickstart files

Run the playbook, specifying which host list to use via `bootstrap_init_hosts`:

```bash
ansible-playbook -i inventory --limit provisioner main.yml \
  -e "vault_path=/path/to/secure/vault/rhis_builder_vault.yml" \
  -e "bootstrap_init_hosts={{ satellite_bootstrap_init_hosts }}" \
  --ask-vault-pass
```

The `bootstrap_init_hosts` extra variable selects a named list from your
`group_vars/provisioner/` file (e.g. `satellite_bootstrap_init_hosts`,
`idm_bootstrap_init_hosts`). Run the playbook once per host group, or pass
a combined list if generating all kickstarts in one pass.

### Step 5: Install the baremetal nodes

#### Method 1: USB boot (lab method)

Prepare the install media:

1. Download the RHEL 9 bootable DVD ISO from [access.redhat.com](https://access.redhat.com).
2. Write the ISO to a USB drive and verify it is bootable.
3. Format a second USB drive with an `xfs` or `ext4` filesystem and label it **OEMDRV**.
   Mount it at the path configured in `bootstrap_init_oem_dir` (default: `/mnt/OEMDRV`).
4. Run the playbook (Step 4). The `ks.cfg` file is written directly to the OEMDRV volume.
5. Unmount the OEMDRV drive.

Install the node:

1. Insert both USB drives into the target baremetal system.
2. Power on and boot from the RHEL 9 installer USB.
3. Anaconda detects the OEMDRV drive and reads `ks.cfg` automatically.
4. The system installs and reboots unattended. Remove the USB drives before the
   second boot, or the system will re-read the kickstart.

Repeat for each target host.

#### Method 2: ISO via virtual media (datacenter method)

This method attaches both the RHEL 9 installer ISO and an OEMDRV ISO as virtual
DVD drives via iDRAC, iLO, Redfish, or your hypervisor's virtual media facility.

Set `generate_oemdrv_iso: true` in your host entry:

```yaml
bootstrap_init_hosts:
  - hostname: "satellite1"
    ...
    generate_oemdrv_iso: true
```

Run the playbook. The role writes `ks.cfg` to `bootstrap_init_oem_dir` and then
generates an ISO9660 image at:

```
bootstrap_init_iso_dir/<rhis_role>_<hostname>.<domain>.oemdrv.iso
```

Attach both the RHEL 9 DVD ISO and this OEMDRV ISO as virtual media to the
target system and boot. Anaconda will detect both drives and automate the install.

The exact procedure for attaching virtual media depends on your hardware vendor.
See your BMC documentation for details.

#### Method 3: Disconnected (air-gapped) environments

For hosts in disconnected environments, set `disconnected: true` in the host entry.
This skips CDN subscription-manager registration and `dnf -y update` in the kickstart
`%post` section. The `org` and `activation_key` fields are not required for disconnected hosts.

```yaml
bootstrap_init_hosts:
  - hostname: "satellite1"
    domain: "highside.example.ca"
    disconnected: true         # DVD install — no CDN registration
    ...
    generate_oemdrv_iso: true  # Generate ISO for virtual media attachment
```

The host will still be fully installed; IdM and Satellite are later configured
to register hosts to the local Satellite (which receives content via Pulp import
from the connected satellite).

### Step 6: Continue with rhis-provisioner

Once all baremetal nodes are installed and reachable over SSH, proceed to the
[rhis-provisioner container](https://github.com/parmstro/rhis-provisioner-container)
to configure IdM, Satellite, and AAP.

When `inventory_update.sh` generated your deployment it also created two
container launch scripts in the `rhis-builder-inventory` root directory:

```
your.domain.24.sh    # AAP 2.4 — being phased out, use only if required
your.domain.25.sh    # AAP 2.5 and later — recommended
```

These scripts are pre-configured with all the correct deployment paths and
call `run_container.sh` internally — you do not need to pass any arguments:

```bash
cd rhis-builder-inventory
./your.domain.25.sh
```

**Why two scripts?** The `ansible.controller` collection introduced a breaking
API change in AAP 2.5 due to significant architectural changes in that release.
The two scripts launch different container images that bundle the correct
collection version for each AAP generation. The containers are otherwise
identical.

> **Recommendation:** Use the `.25.sh` script. AAP 2.4 support is being phased
> out and the 2.4-specific script will be removed in a future release of
> rhis-builder-inventory, after which a single launch script will be generated.

The container mounts your deployment configuration and executes the RHIS
provisioning playbooks in sequence: IdM → Satellite → AAP.

---

## Variable reference

### Role-level variables

These are set once per playbook run, either in `group_vars/provisioner/` or
passed as extra variables.

| Variable | Default | Description |
|---|---|---|
| `bootstrap_init_ks_path` | `ks.cfg` | Filename written to the OEMDRV volume |
| `bootstrap_init_oem_dir` | `/mnt/OEMDRV` | Path to the mounted OEMDRV volume |
| `bootstrap_init_iso_dir` | `/mnt/OEMDRV/ISO` | Directory for generated OEMDRV ISO files |
| `bootstrap_init_hosts` | `[]` | List of host definitions (required) |

### Per-host variables

Each entry in `bootstrap_init_hosts` defines one system.

#### Identity

| Variable | Required | Description |
|---|---|---|
| `rhis_role` | Yes | One of: `provisioner`, `idm`, `satellite`, `kvm` |
| `hostname` | Yes | Short hostname (no domain) |
| `domain` | Yes | DNS domain name |

#### Network

| Variable | Required | Description |
|---|---|---|
| `mac` | Yes | MAC address of the primary NIC (`xx:xx:xx:xx:xx:xx`) |
| `ipv4_address` | Yes | Static IPv4 address |
| `ipv4_netmask` | Yes | Dotted-decimal subnet mask (used by kickstart `network` directive) |
| `ipv4_prefix` | Yes | CIDR prefix length integer (used by NetworkManager keyfile) |
| `ipv4_gateway` | Yes | Default gateway |
| `name_server1` | Yes | Primary DNS server |
| `name_server2` | Yes | Secondary DNS server |

Network configuration is written as a NetworkManager keyfile in `%post` with
MAC-based interface matching. Legacy `ifcfg` profiles are removed during install.

#### Storage

| Variable | Required | Description |
|---|---|---|
| `boot_disk` | Yes | Disk path for `/boot` and `/boot/efi` (e.g. `/dev/nvme0n1`) |
| `root_disk` | Yes | Disk path for the root volume group (often same as `boot_disk`) |
| `additional_disks` | No | List of additional disk paths to extend the root VG |
| `filesystem` | No | `xfs` (default) or `ext4`. Use `ext4` for KVM hosts if online resize is needed |

Both `boot_disk` and `root_disk` are fully initialized by `clearpart`. If they
are the same disk, it is targeted once. This satisfies the IdM and Satellite
installation criteria that require a fully dedicated system.

#### Partition sizes (all in MiB)

The compliance layout below meets CIS Level 2, DISA-STIG, and PCI-DSS
partition requirements. Satellite requires a large `lv_var` — set to `1` to
instruct anaconda to grow it into all remaining space.

| Variable | Recommended (IdM/KVM) | Recommended (Satellite) | Description |
|---|---|---|---|
| `boot_mb` | `1024` | `1024` | `/boot` partition |
| `boot_efi_mb` | `2048` | `2048` | `/boot/efi` partition |
| `lv_root_mb` | `65536` | `65536` | `/` logical volume |
| `lv_home_mb` | `20480` | `20480` | `/home` logical volume |
| `lv_tmp_mb` | `6144` | `6144` | `/tmp` logical volume |
| `lv_var_tmp_mb` | `6144` | `6144` | `/var/tmp` logical volume |
| `lv_var_log_mb` | `6144` | `6144` | `/var/log` logical volume |
| `lv_var_log_audit_mb` | `6144` | `6144` | `/var/log/audit` logical volume |
| `lv_var_mb` | `1` | `1` | `/var` — set to `1` to grow into remaining space |

A 256 GiB minimum drive is recommended for IdM and KVM. Satellite should have
a 1 TiB or larger drive; downloading all standard RHEL repositories requires
approximately 900 GiB.

##### Disconnected and air-gapped environments

RHIS fully supports disconnected environments using Satellite's content
export/import workflow. Storage sizing for `/var` must account for the
additional space that the export and transfer process requires.

**Connected Satellite (exporting content)**

The connected Satellite needs at least **2x** the space required for the
exported content set in `/var` — 1x for the live Satellite content and 1x
for the export archive. For a full Library export covering RHEL 8, 9, and 10
content, a **2 TiB `/var`** is recommended.

**Disconnected Satellite (importing content)**

The space required on the disconnected host depends on how the transfer
media is presented:

| Import method | `/var` required | Notes |
|---|---|---|
| Mount transfer drive, import directly | **2x** export size | 1x tar extraction + 1x final content (tar files remain on the transfer drive) |
| Copy from transfer drive then import | **3x** export size | 1x tar files + 1x tar extraction + 1x final content |

The copy-then-import method requires the most headroom because all three
artifacts — the tar archives, the extracted content, and the final imported
content — coexist on `/var` simultaneously during the import process. When
mounting the transfer drive directly, the tar files never land on the
disconnected host's `/var`, reducing the requirement by one full copy.

When sizing the disconnected Satellite, set `lv_var_mb: 1` as usual to
consume all remaining disk space, and provision the drive accordingly before
running the kickstart.

#### User and access

| Variable | Required | Description |
|---|---|---|
| `username` | Yes | Automation user created during install |
| `user_enc_pass` | Yes | Encrypted password (vault reference) |
| `user_sudoer_policy` | Yes | Sudoers policy line (vault reference) |
| `ssh_pub_key` | Yes | SSH public key string for the user (vault reference) |

#### Credentials

| Variable | Required | Description |
|---|---|---|
| `root_enc_pass` | Yes | Encrypted root password (vault reference) |
| `grub_enc_pass` | No | Encrypted GRUB2 bootloader password (vault reference) |
| `org` | Connected only | Red Hat CDN organization ID (vault reference) — not required when `disconnected: true` |
| `activation_key` | Connected only | CDN activation key name (vault reference) — not required when `disconnected: true` |
| `disconnected` | No | Set to `true` to skip CDN registration and `dnf -y update` in kickstart |

The node is registered to the CDN during kickstart `%post` to pull the minimal
package set. IdM nodes are later re-registered to Satellite by `rhis-builder-idm`.

#### Delivery

| Variable | Default | Description |
|---|---|---|
| `generate_oemdrv_iso` | `false` | When `true`, generates an ISO9660 image instead of writing directly to the OEMDRV volume |

---

## Vault variables

Generate encrypted values with the following commands:

### Root and user passwords

```bash
openssl passwd -6
```

`openssl passwd -6` generates a SHA-512 hashed password (`$6$...`). Run it
without arguments and enter the password at the prompt to avoid the plaintext
appearing in your shell history. `openssl` is included by default on RHEL and
Fedora — no additional packages are required on the provisioner or your
workstation.

### GRUB2 bootloader password

```bash
grub2-mkpasswd-pbkdf2
```

Copy the full `grub.pbkdf2.sha512.…` string into your vault file.

### Vault file structure

```yaml
encrypted_root_pass_vault:      "$6$rounds=..."
encrypted_grub_pass_vault:      "grub.pbkdf2.sha512.10000...."
encrypted_user_pass_vault:      "$6$rounds=..."
user_sudoer_policy_vault:       "ansiblerunner ALL=(ALL) NOPASSWD: ALL"
ssh_pub_key_vault:              "ssh-ed25519 AAAA... ansiblerunner@provisioner"
cdn_organization_vault:         "12345678"
cdn_activation_key_vault:       "rhis-bootstrap"
```

---

## Linting

The project is validated against the `ansible-lint` production profile:

```bash
ansible-lint
```

The configuration is in `.ansible-lint` and `.yamllint`. The `bootstrap_init`
role must pass with 0 failures before merging.

Install ansible-lint under the same Python version as your system ansible-core:

```bash
python3.12 -m pip install --user ansible-lint
```

---

## Contributing

Fork it. Clone it. Configure it. Change it. Test it. Commit it. Create a PR.

If you have comments, tips, or suggestions, PRs are greatly appreciated.
Let's share the wealth!

ENJOY!

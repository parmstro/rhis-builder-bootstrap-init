# CLAUDE.md — rhis-builder-bootstrap-init

## Project purpose

Generates OEMDRV kickstart files to bootstrap RHEL 9 baremetal and virtual hosts
for the rhis-builder family of projects. Supported roles: `provisioner`, `idm`,
`satellite`, `kvm`. Kickstart files are delivered either directly to a USB-mounted
OEMDRV volume or as ISO9660 images for iDRAC/iLO/Redfish virtual media.

Disconnected (air-gapped) installations are supported via the `disconnected: true`
per-host flag, which skips CDN `subscription-manager register` and `dnf -y update`
in the kickstart `%post` section.

This repository contains a single role: `bootstrap_init`. It was extracted from
`rhis-builder-baremetal-init` on 2026-06-08.

## Variable conventions

All role variables use the `bootstrap_init_` prefix to avoid clashes when combined
with other rhis-builder projects.

Role-level defaults are in `roles/bootstrap_init/defaults/main.yml`.
Per-host variables are passed as a list under `bootstrap_init_hosts`.
Sensitive values (passwords, keys, org credentials) must come from an
Ansible Vault encrypted file, referenced as Jinja2 variables in the host entry.

## Task flow

```
main.yml
  └── validate_args.yml          (once, before loop)
  └── loop over bootstrap_init_hosts
        └── generate_ks.yml
              ├── validate_host.yml
              ├── ensure OEMDRV dir exists
              ├── template → ks.cfg.j2
              └── generate_oemdrv_iso.yml  (when generate_oemdrv_iso: true)
```

## Running the playbook

Add the bootstrap_init_hosts variable assignment to pick a host list from your group_vars variable files.

```bash
ansible-playbook -i inventory --limit provisioner main.yml \
  -e "vault_path=/path/to/vault.yml" \
  -e "bootstrap_init_hosts={{ idm_bootstrap_init_hosts }}" \
  --ask-vault-pass
```

The vault file must define: `encrypted_root_pass_vault`, `encrypted_grub_pass_vault`,
`encrypted_user_pass_vault`, `user_sudoer_policy_vault`, `ssh_pub_key_vault`.
For connected hosts also: `cdn_organization_vault`, `cdn_activation_key_vault`.

Variable files for each host are loaded via `vars_path` or `vars_dir`.
See `bootstrap_init_vars.SAMPLE.yml` for a complete example.

## Linting

```bash
ansible-lint          # uses .ansible-lint (production profile)
```

The project must pass the `production` profile with 0 failures before merging.

## Key design decisions

- Template renders directly to `bootstrap_init_oem_dir/bootstrap_init_ks_path` —
  no staging directory or intermediate copy step.
- Network configuration uses NM keyfile format (`/etc/NetworkManager/system-connections/`)
  with MAC-based interface matching. Legacy ifcfg files are removed in `%post`.
- `clearpart` targets both `boot_disk` and `root_disk` (deduplicated) to ensure
  full disk initialisation required by IDM and Satellite installation criteria.
- `ipv4_netmask` (dotted-decimal) is used by the kickstart `network` directive;
  `ipv4_prefix` (CIDR integer) is used by the NM keyfile. Both are required.
- `disconnected: true` suppresses both `subscription-manager register` and
  `dnf -y update` in `%post`. Validation of `org`/`activation_key` fields is
  also skipped for disconnected hosts.
- Collection dependencies belong in `requirements.yml`, not `meta/main.yml`.

## What NOT to put in this file

Do not add hostnames, IP addresses, MAC addresses, domain names, org names,
vault variable values, or any other environment-specific details here.
Those belong in gitignored `group_vars/`, `host_vars/`, and vault files.

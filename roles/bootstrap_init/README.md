### bootstrap_init

This role is a rename of the original baremetal_init role.

The role is being renamed because it really initializes more than just baremetal hosts.

We are using the OEMDRV method to provision hosts. When a system boots from a RHEL DVD ISO
and also discovers a volume labelled "OEMDRV", it will search the OEMDRV volume's root
directory looking for a file named "ks.cfg". If this can be interpreted as a kickstart file,
the boot will continue and anaconda will use ks.cfg to perform the installation. How we
provide the installation media and OEMDRV volume with the kickstart file is kind of
irrelevant. This could be the virtual CD/DVD drive of a VM, ISO files configured for an
iDRAC/iLO/Redfish subsystem, or any other volume that a system might use to start up.

So bootstrap_init it is!

bootstrap_init accepts all the same parameters as baremetal_init, plus it asks whether you
want to create an OEMDRV ISO file.

generate_oemdrv_iso:
 - false (default) - renders the kickstart file directly to the directory specified by
   "bootstrap_init_oem_dir"
 - true - generates an ISO9660 .iso file and stores it as a host-specific file in
   "bootstrap_init_iso_dir"

#### bootstrap_init parameters

bootstrap_init_ks_path: "ks.cfg"
bootstrap_init_oem_dir: "/mnt/OEMDRV"
bootstrap_init_iso_dir: "/mnt/OEMDRV/ISO"


bootstrap_init_hosts:
  - rhis_role: "idm"
    hostname: "idm1"
    domain: "example.ca"
    mac: "ff:ff:ff:ff:ff:fe"
    ipv4_address: "10.10.8.5"
    ipv4_netmask: "255.255.252.0"
    ipv4_prefix: 22
    ipv4_gateway: "10.10.8.1"
    name_server1: "10.20.8.5"
    name_server2: "10.20.8.6"
    boot_disk: "/dev/nvme0n1"
    root_disk: "/dev/nvme0n1"
    root_enc_pass: "{{ encrypted_root_pass_vault }}"
    grub_enc_pass: "{{ encrypted_grub_pass_vault }}"
    boot_mb: 1024
    boot_efi_mb: 2048
    lv_root_mb: 65536
    lv_home_mb: 20480
    lv_tmp_mb: 6144
    lv_var_tmp_mb: 6144
    lv_var_log_mb: 6144
    lv_var_log_audit_mb: 6144
    lv_var_mb: 1
    username: "ansiblerunner"
    user_enc_pass: "{{ encrypted_user_pass_vault }}"
    user_sudoer_policy: "{{ user_sudoer_policy_vault }}"
    ssh_pub_key: "{{ ssh_pub_key_vault }}"
    org: "{{ cdn_organization_vault }}"
    activation_key: "{{ cdn_activation_key_vault }}"
    generate_oemdrv_iso: false

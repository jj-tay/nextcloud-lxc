# nextcloud-lxc

Ansible playbook that installs and upgrades a native Nextcloud stack
(Apache + PHP-FPM + MariaDB + Redis) on a Debian 13 unprivileged Proxmox
LXC, with the Nextcloud data directory backed by an NFS export from a
co-located TrueNAS SCALE VM.

Design background: `docs/superpowers/specs/2026-09-09-nextcloud-lxc-design.md`.

## Prerequisites (manual, one-time)

These are on the Proxmox host and the TrueNAS VM, which are outside this
playbook's scope (it only configures the LXC).

1. **Proxmox**: create the `vmbr1` internal bridge (no physical uplink,
   no IP on the host side). Attach a second NIC on `vmbr1` to the TrueNAS
   VM, and add a second NIC to the Nextcloud LXC as `net1` on `vmbr1`
   (e.g. name `eth1`). Leave the LXC NIC's IPv4 and gateway empty — the
   playbook sets its static IP (`nextcloud_vmbr1_ip`, default
   `10.10.10.11/24`) automatically on first install, with no gateway.
2. **TrueNAS SCALE**:
   - Give the VM's `vmbr1` NIC the static IP `10.10.10.10/24` with **no
     gateway** (this must match `truenas_nfs_host`).
   - Create the `tank/nextcloud` dataset, NFS-share it, and set the
     export's mapall user/group to the UID/GID configured in
     `nextcloud_www_data_uid` (default `1000`).
   - Add `10.10.10.11` (the LXC's vmbr1 IP) to the export's allowed
     hosts/networks. Ansible never touches TrueNAS, so this stays manual.
3. **Proxmox LXC**: a Debian 13 container, reachable over SSH as `root`
   with a key Ansible can use.
4. **Proxmox host**: reachable over SSH as `root` from the Ansible
   machine. Every install run reads the LXC's config with `pct config`
   and, only if needed, sets the vmbr1 IP with `pct set`; upgrade runs
   also use it for `pct snapshot`.

## Setup

```bash
ansible-galaxy collection install -r requirements.yml

cp group_vars/all/vault.yml.example group_vars/all/vault.yml
# edit group_vars/all/vault.yml with real secrets, then:
ansible-vault encrypt group_vars/all/vault.yml
```

Edit `inventory.ini` and `roles/nextcloud/defaults/main.yml` for your
environment (LXC/Proxmox IPs, VMID, domain, TrueNAS export path, vmbr1
IPs, etc).

## Install / re-run

Safe to run repeatedly, including immediately after an upgrade — every
step is guarded so a re-run only confirms existing state:

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

## Upgrade

The upgrade play is tagged `upgrade` and additionally tagged `never`, so
it is *never* executed by a plain `ansible-playbook playbook.yml` run.
Trigger it explicitly, overriding the target version:

```bash
ansible-playbook playbook.yml --tags upgrade -e nextcloud_version=34.1.0 --ask-vault-pass
```

This will, in order: turn on maintenance mode, snapshot the LXC on the
Proxmox host, back up `config.php`, swap in the new release (carrying
over `config.php` and any third-party apps), run `occ upgrade`, then
turn maintenance mode back off. If any step fails, the play halts with
maintenance mode still on — roll back using the Proxmox snapshot taken
at the start of the run.

## Repo layout

```
nextcloud-lxc/
├── inventory.ini
├── playbook.yml
├── group_vars/all/vault.yml.example   # copy to vault.yml and encrypt
├── roles/nextcloud/
│   ├── tasks/main.yml                 # idempotent install
│   ├── tasks/network.yml              # vmbr1 NIC IP (via pct), imported by main.yml
│   ├── tasks/upgrade.yml              # tagged 'upgrade'
│   ├── templates/                     # vhost, php-fpm overrides
│   ├── defaults/main.yml              # non-secret vars
│   └── handlers/main.yml
└── docs/superpowers/specs/            # design docs
```

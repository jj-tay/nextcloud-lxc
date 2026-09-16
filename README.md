# nextcloud-lxc

Ansible playbook that installs and upgrades a native Nextcloud stack
(Apache + PHP-FPM + MariaDB + Redis) on a Debian 12 unprivileged Proxmox
LXC, with the Nextcloud data directory backed by an NFS export from a
co-located TrueNAS SCALE VM.

Design background: `docs/superpowers/specs/2026-09-09-nextcloud-lxc-design.md`.

## Prerequisites (manual, one-time)

1. **Proxmox**: create the `vmbr1` internal bridge (no physical uplink),
   and attach a second NIC to both the TrueNAS VM and the Nextcloud LXC
   on it.
2. **TrueNAS SCALE**: create the `tank/nextcloud` dataset, NFS-share it,
   and set the export's mapall user/group to the UID/GID configured in
   `nextcloud_www_data_uid` (default `1000`).
3. **Proxmox LXC**: a Debian 12 container, reachable over SSH as `root`
   with a key Ansible can use.

## Setup

```bash
ansible-galaxy collection install -r requirements.yml

cp group_vars/all/vault.yml.example group_vars/all/vault.yml
# edit group_vars/all/vault.yml with real secrets, then:
ansible-vault encrypt group_vars/all/vault.yml
```

Edit `inventory.ini` and `roles/nextcloud/defaults/main.yml` for your
environment (LXC/Proxmox IPs, VMID, domain, TrueNAS export path, etc).

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
│   ├── tasks/upgrade.yml              # tagged 'upgrade'
│   ├── templates/                     # vhost, php-fpm overrides
│   ├── defaults/main.yml              # non-secret vars
│   └── handlers/main.yml
└── docs/superpowers/specs/            # design docs
```

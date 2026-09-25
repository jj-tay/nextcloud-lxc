# nextcloud-lxc

Ansible playbook that provisions a Debian 13 unprivileged Proxmox LXC, then
installs and upgrades a native Nextcloud stack (Apache + PHP-FPM +
MariaDB + Redis) on it, with the Nextcloud data directory backed by an NFS
export from a co-located TrueNAS SCALE VM.

Design background: `docs/superpowers/specs/2026-09-09-nextcloud-lxc-design.md`.

## Prerequisites (manual, one-time)

These are on the Proxmox host, the TrueNAS VM and the Ansible controller.
The playbook creates the Nextcloud LXC itself, but never touches Proxmox
networking or TrueNAS.

1. **Proxmox**: create the `vmbr1` internal bridge (no physical uplink,
   no IP on the host side) and attach a second NIC on `vmbr1` to the
   TrueNAS VM. The LXC's own `vmbr1` NIC (`net1`, `eth1`, static
   `nextcloud_vmbr1_ip`, default `10.10.10.11/24`, no gateway) is created
   by the playbook.
2. **TrueNAS SCALE**:
   - Give the VM's `vmbr1` NIC the static IP `10.10.10.10/24` with **no
     gateway** (this must match `truenas_nfs_host`).
   - Create the `tank/nextcloud` dataset, NFS-share it, and set the
     export's mapall user/group to the UID/GID configured in
     `nextcloud_www_data_uid` (default `1000`).
   - Add `10.10.10.11` (the LXC's vmbr1 IP) to the export's allowed
     hosts/networks. Ansible never touches TrueNAS, so this stays manual.
3. **Proxmox API token** (used to create and start the LXC):
   - Datacenter → Permissions → Users: add a user, e.g. `ansible@pve`.
   - Datacenter → Permissions → API Tokens: add a token for it, e.g.
     `provision`. Copy the secret; it is only shown once.
   - Datacenter → Permissions → Add: grant these roles. If the token has
     *Privilege Separation* ticked, grant them to the token itself (user
     `ansible@pve!provision`); otherwise to the user.
     - `PVEVMAdmin` on `/vms` (create, configure, start, and the
       `nesting=1` feature flag)
     - `PVEDatastoreAdmin` on `/storage/local` (template download) and
       `/storage/local-lvm` (rootfs), or whatever
       `proxmox_lxc_template_storage` / `proxmox_lxc_rootfs_storage` are
     - `PVESDNUser` on `/sdn/zones/localnetwork` (to attach NICs to
       `vmbr0` / `vmbr1`)
   - Put the host, user, token ID and secret into your real
     `group_vars/all/vault.yml` as `vault_proxmox_api_host`,
     `vault_proxmox_api_user`, `vault_proxmox_api_token_id` and
     `vault_proxmox_api_token_secret` (see `vault.yml.example`). They are
     new, so an existing vault.yml needs them added:
     `ansible-vault edit group_vars/all/vault.yml`.
4. **Proxmox host**: still reachable over SSH as `root` from the Ansible
   machine. Every install run reads the LXC's config with `pct config`
   and, only if needed, sets the vmbr1 IP with `pct set`; upgrade runs
   also use it for `pct snapshot`.
5. **UniFi**: a Fixed IP (DHCP reservation) of `nextcloud_lxc_host`
   (default `192.168.10.51`) for the LXC's LAN MAC. Set
   `proxmox_lxc_lan_hwaddr` so the MAC is known before the container
   exists. Otherwise Proxmox picks a random MAC, and you add the
   reservation after the first run.
6. **Ansible controller**: `proxmoxer` (>= 2.3) and `requests` installed
   for the Python that runs Ansible (`pip install proxmoxer requests` in
   the same venv/pipx env), and the SSH public key in
   `proxmox_lxc_ssh_pubkey_file` (default `~/.ssh/id_ed25519.pub`). It is
   injected into the LXC's root account when the LXC is created.

## Setup

```bash
ansible-galaxy collection install -r requirements.yml

cp group_vars/all/vault.yml.example group_vars/all/vault.yml
# edit group_vars/all/vault.yml with real secrets, then:
ansible-vault encrypt group_vars/all/vault.yml
```

Edit for your environment:

- `inventory.ini`: Proxmox host IP, and the LXC's LAN IP (used by
  upgrade runs only).
- `group_vars/all/main.yml`: VMID, expected LAN IP, vmbr1 NIC/IP (shared
  by both roles).
- `roles/proxmox_lxc/defaults/main.yml`: Proxmox node name, hostname,
  cores/RAM/rootfs, template, bridges, LAN MAC.
- `roles/nextcloud/defaults/main.yml`: domain, TrueNAS export, PHP,
  Nextcloud version, etc.

## Provision + Install / re-run

One command creates the LXC (if needed) and installs Nextcloud on it:

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

1. **Provision** (play on `proxmox_api`, runs on the controller and talks
   to the Proxmox API): downloads the Debian template if it is missing,
   creates an unprivileged LXC at `nextcloud_lxc_vmid` with both NICs
   (LAN on `proxmox_lxc_lan_bridge` via DHCP, and `vmbr1` with the static
   `nextcloud_vmbr1_ip`), starts it, and waits for a LAN IP and for SSH.
   The IP the Proxmox API reports is passed to the next play with
   `add_host`.
2. **Install** (play on `nextcloud_lxc`, over SSH): the idempotent
   Nextcloud install, unchanged.

Safe to run repeatedly, including immediately after an upgrade — every
step is guarded so a re-run only confirms existing state. An existing
container at the VMID is never recreated, reconfigured or restarted; the
play only checks that it is named `proxmox_lxc_hostname` and is running.
Changing sizing variables later does **not** resize an existing container;
do that in Proxmox.

To only provision (e.g. to check the API token), run with
`--tags provision`.

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
├── playbook.yml                       # provision play, then install, then upgrade
├── group_vars/all/main.yml            # vars shared by both roles (VMID, vmbr1)
├── group_vars/all/vault.yml.example   # copy to vault.yml and encrypt
├── roles/proxmox_lxc/
│   ├── tasks/main.yml                 # create/start LXC via Proxmox API, add_host
│   └── defaults/main.yml              # node, sizing, template, bridges
├── roles/nextcloud/
│   ├── tasks/main.yml                 # idempotent install
│   ├── tasks/network.yml              # vmbr1 NIC IP (via pct), imported by main.yml
│   ├── tasks/upgrade.yml              # tagged 'upgrade'
│   ├── templates/                     # vhost, php-fpm overrides
│   ├── defaults/main.yml              # non-secret vars
│   └── handlers/main.yml
└── docs/superpowers/specs/            # design docs
```

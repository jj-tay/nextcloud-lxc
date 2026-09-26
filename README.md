# nextcloud-lxc

Ansible playbook that provisions a Debian 13 unprivileged Proxmox LXC, then
installs and upgrades a native Nextcloud stack (Apache + PHP-FPM +
MariaDB + Redis) on it, with the Nextcloud data directory backed by an NFS
export from a co-located TrueNAS SCALE VM.

NFS can't be mounted inside an unprivileged LXC (NFS has no Linux user
namespace support, so `mount` fails with "Operation not permitted" whatever
the AppArmor profile or feature flags). So the **Proxmox host** mounts the
export, and the container only gets a bind mount of it:

```
TrueNAS 10.10.10.10:/mnt/tank/nextcloud
  --NFS over vmbr1-->  Proxmox host 10.10.10.1: /mnt/nextcloud-data   (host /etc/fstab)
  --bind mount mp0-->  LXC: /mnt/ncdata                               (Nextcloud data dir)
```

The playbook sets all of this up (host vmbr1 IP, host NFS mount, `mp0`).
The LXC keeps only the `nesting=1` feature flag. So that the files TrueNAS
hands out as `1000:3000` show up as `www-data` inside the unprivileged LXC
(rather than `nobody:nogroup`), the playbook also maps exactly that UID
and GID 1:1 between host and container with `lxc.idmap`; every other ID
keeps the usual `100000` offset.

Design background: `docs/superpowers/specs/2026-09-09-nextcloud-lxc-design.md`.

## Prerequisites (manual, one-time)

Everything in this section lives outside the playbook: Ansible never
creates or fixes any of it, and never touches TrueNAS at all. A rebuild,
or a fresh TrueNAS, must go through it by hand, in order, before the
first run.

### Storage network and TrueNAS NFS export

1. **Proxmox host: create the `vmbr1` bridge.** Internal-only: no
   bridge ports (no physical uplink), and no IP set when you create it.
   The host's own vmbr1 IP is added later by the playbook (see step 6);
   the playbook only fails if the bridge itself is missing.

2. **Proxmox: give the TrueNAS VM a second NIC on `vmbr1`** (VM →
   Hardware → Add → Network Device, bridge `vmbr1`), separate from its
   LAN NIC.

3. **TrueNAS: give that second NIC the static IP `10.10.10.10/24`, no
   gateway.** It must match `truenas_nfs_host`.

   If TrueNAS's network-edit UI throws a "gateway unreachable"
   `CallError` while you do this, that is a known TrueNAS SCALE UI bug,
   not a real misconfiguration. The reliable fallback is `midclt` on the
   TrueNAS shell (`<ifname>` is the vmbr1 NIC's name inside TrueNAS):

   ```bash
   midclt call interface.update "<ifname>" '{"ipv4_dhcp": false, "aliases": [{"type": "INET", "address": "10.10.10.10", "netmask": 24}]}'
   ```

   Then apply it live with `ip addr add 10.10.10.10/24 dev <ifname>`
   (not persistent), or with a systemd-networkd drop-in for persistence.
   Avoid `midclt call interface.commit`: it re-evaluates *all*
   interfaces, including the DHCP LAN NIC, and can drop connectivity.

4. **TrueNAS: create a dedicated local user for the NFS export**, e.g.
   `nextcloud-nfs`:
   - SMB, TrueNAS, Shell and SSH access all unchecked; password disabled.
   - UID chosen manually (this install used `1000`).
   - "Create New Primary Group" checked. The group's GID will **not**
     automatically match the UID: TrueNAS assigns it independently (it
     landed on `3000` in this install).

   Record the UID and GID TrueNAS actually assigns. Everything after this
   must match them exactly, including `nextcloud_www_data_uid` /
   `nextcloud_www_data_gid` in `roles/nextcloud/defaults/main.yml`
   (defaults `1000` / `3000`), which the playbook uses for the `lxc.idmap`
   carve-out and host `/etc/subuid` / `/etc/subgid` entries.

5. **TrueNAS: create the dataset (e.g. `tank/nextcloud`) and NFS-share
   it.** The export path must match `truenas_nfs_export` (default
   `/mnt/tank/nextcloud`).
   - **Mapall User / Mapall Group**: the user and group from step 4.
   - **Networks**: add **both** the Nextcloud LXC's vmbr1 IP
     (`10.10.10.11/32`) **and** the Proxmox host's vmbr1 IP
     (`10.10.10.1/32`). The host's IP is the easy one to miss: it is not
     obvious that the host needs NFS access, but it is the host that
     mounts the export (see step 6).
   - **Critical, easily missed: chown the dataset.** A new dataset is
     owned by `root:root` with mode `0755`. Mapall only decides which
     UID/GID new writes are attributed to; it does **not** change the
     ownership of the dataset directory itself. Without this, no client
     can write to it, however correctly mapall is set. On the TrueNAS
     shell:

     ```bash
     chown nextcloud-nfs:nextcloud-nfs /mnt/tank/nextcloud
     ```

     or in the UI: Datasets → the dataset → Permissions → Edit, and set
     Owner user and group to `nextcloud-nfs`.

6. **Architecture note: why the Proxmox host needs vmbr1 and NFS access.**
   NFS cannot be mounted directly inside an unprivileged LXC (confirmed by
   testing and by Proxmox's own documentation: NFS has no support for
   Linux user namespaces). So the playbook mounts the export on the
   **Proxmox host** and bind-mounts that path into the LXC (see the
   diagram at the top). The host therefore needs its own static IP on
   vmbr1, `10.10.10.1/24` with no gateway (same pattern as TrueNAS's
   interface), and that IP must be in the export's allowed networks
   (step 5). The playbook sets this host IP itself
   (`proxmox_host_vmbr1_ip`, written to `/etc/network/interfaces`), along
   with the host NFS mount, `/etc/fstab` entry and `mp0` bind mount; you
   only need to make sure TrueNAS allows it.

7. **TrueNAS: confirm the NFS service is enabled and running** (System →
   Services, with "Start Automatically" on). A configured share does
   nothing while the service is off.

### Other prerequisites

8. **Proxmox API token** (used to create and start the LXC):
   - Datacenter → Permissions → Users: add a user, e.g. `ansible@pve`.
   - Datacenter → Permissions → API Tokens: add a token for it, e.g.
     `provision`. Copy the secret; it is only shown once.
   - Datacenter → Permissions → Add: grant these roles. If the token has
     *Privilege Separation* ticked, grant them to the token itself (user
     `ansible@pve!provision`); otherwise to the user.
     - `PVEVMAdmin` on `/vms` (create, configure, start, and the
       `nesting=1` feature flag)
     - `PVEDatastoreAdmin` on `/storage/local` (template download) and
       `/storage/local-zfs` (rootfs), or whatever
       `proxmox_lxc_template_storage` / `proxmox_lxc_rootfs_storage` are
     - `PVESDNUser` on `/sdn/zones/localnetwork` (to attach NICs to
       `vmbr0` / `vmbr1`)
   - Put the host, user, token ID and secret into your real
     `group_vars/all/vault.yml` as `vault_proxmox_api_host`,
     `vault_proxmox_api_user`, `vault_proxmox_api_token_id` and
     `vault_proxmox_api_token_secret` (see `vault.yml.example`). They are
     new, so an existing vault.yml needs them added:
     `ansible-vault edit group_vars/all/vault.yml`.
9. **Proxmox host**: reachable over SSH as `root` from the Ansible
   machine. Keep UID `1000` / GID `3000` (or whatever step 4 produced)
   unused on the host itself. For reference, every install run does the
   following there, each step only when something is missing or wrong:
   - sets the host's vmbr1 IP (`/etc/network/interfaces`, then
     `ifup vmbr1`),
   - mounts `truenas_nfs_host:truenas_nfs_export` at
     `nextcloud_host_nfs_mount` (default `/mnt/nextcloud-data`) and
     persists it in the host's `/etc/fstab`,
   - adds `root:1000:1` / `root:3000:1` to `/etc/subuid` / `/etc/subgid`
     and the `lxc.idmap` carve-out to `/etc/pve/lxc/<vmid>.conf`. Only
     the first time: the LXC is shut down, files it already owns as
     www-data (host `101000` / `103000`) are re-owned to `1000` / `3000`,
     and it is started again. Pre-existing, different `lxc.idmap` lines
     make the play fail rather than be overwritten.
   - reads the LXC's config with `pct config` and uses `pct set` to fix
     the LXC's vmbr1 IP (`net1`, `eth1`, static `nextcloud_vmbr1_ip`,
     default `10.10.10.11/24`, no gateway) and the `mp0` bind mount
     (`nextcloud_host_nfs_mount` → `nextcloud_data_mount`). If `mp0` had
     to be changed, the LXC is restarted (`pct reboot`) so it takes
     effect. Host-path bind mounts need `root@pam`, which is why this is
     done over SSH and not with the API token.

   Upgrade runs also use it for `pct snapshot`.
10. **UniFi**: a Fixed IP (DHCP reservation) of `nextcloud_lxc_host`
    (default `192.168.10.51`) for the LXC's LAN MAC. Set
    `proxmox_lxc_lan_hwaddr` so the MAC is known before the container
    exists. Otherwise Proxmox picks a random MAC, and you add the
    reservation after the first run.
11. **Ansible controller**: `proxmoxer` (>= 2.3) and `requests` installed
    for the Python that runs Ansible (`pip install proxmoxer requests` in
    the same venv/pipx env), and the SSH public key in
    `proxmox_lxc_ssh_pubkey_file` (default `~/.ssh/id_ed25519.pub`). It is
    injected into the LXC's root account when the LXC is created.

### Known environment gotchas

Not steps to do, but things to know when something breaks.

- **Proxmox mangles `#` lines in guest configs.** Leading `#` comment
  lines in `/etc/pve/lxc/<vmid>.conf` become the container's description,
  and on every config rewrite (start/stop, snapshots, GUI edits) Proxmox
  URL-encodes them (`:` → `%3A`) and moves them. So raw `lxc.*` lines,
  like the `lxc.idmap` entries this playbook writes, must never be wrapped
  in `#`-prefixed marker comments (e.g. `blockinfile` BEGIN/END markers):
  the markers stop matching, the block gets added again, and the
  duplicated idmap lines stop the container from starting. This playbook
  writes them as a plain file rewrite instead (`tasks/host_idmap.yml`).
- **MariaDB "Fatal error in defaults handling" is not MDEV-35904.**
  This message comes from MariaDB's option-file handling: a defaults file
  it was told to read is missing or unreadable (e.g.
  `/etc/mysql/debian.cnf`, seen after Debian 12 → 13 upgrades with
  MariaDB 11.8), or an option file has a bad entry. If MariaDB won't
  start, read `journalctl -u mariadb` in the LXC for the file or option it
  names, and fix that. Don't confuse it with MariaDB Jira MDEV-35904:
  with systemd v254+, `mariadb.service` logs *warnings* about the unset
  variables `MYSQLD_OPTS`, `_WSREP_NEW_CLUSTER` and
  `_WSREP_START_POSITION`. Those warnings are cosmetic, don't stop the
  server, and are fixed in MariaDB 11.8.4 and later, so this role applies
  no systemd override for them. (On older packages, a
  `mariadb.service.d` override setting the three variables to empty
  strings silences the warnings, but it does not fix a startup failure.)
- **The Debian 13 LXC template has no `sudo`**, which breaks every
  Ansible `become: true` / `become_user` task until it is installed. The
  role installs it right after the base packages, before the first
  `become_user` task (`roles/nextcloud/tasks/main.yml`, "Install sudo for
  become_user www-data").

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
- `roles/nextcloud/defaults/main.yml`: domain, TrueNAS export, host-side
  NFS mount point, the Proxmox host's vmbr1 IP, PHP, Nextcloud version,
  etc.

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
   Nextcloud install. Before anything references the data directory, it
   sets up the storage path on the Proxmox host (vmbr1 IP, NFS mount,
   `mp0` bind mount; see Prerequisites 6 and 9) and checks that
   `nextcloud_data_mount` is NFS-backed inside the LXC.

Safe to run repeatedly, including immediately after an upgrade — every
step is guarded so a re-run only confirms existing state. An existing
container at the VMID is never recreated by the provision play; it only
checks that it is named `proxmox_lxc_hostname` and is running. The
install play only restarts the LXC when it first applies the `lxc.idmap`
carve-out or when the `mp0` bind mount was missing or wrong.
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

### After a Proxmox host reboot

**Known limitation:** TrueNAS is a VM on the same host, so at boot the
host's fstab NFS mount runs before TrueNAS is up and fails, and the LXC
(`onboot`) can start with the *empty* host directory bind-mounted at
`/mnt/ncdata`. After any Proxmox host reboot, once TrueNAS is up, re-run
the playbook before assuming Nextcloud is healthy:

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

It mounts the export on the host and fails loudly if `/mnt/ncdata` is
still not NFS-backed inside the LXC. If it fails on that check, restart
the LXC (`pct reboot <vmid>`) so it picks up the now-live host mount,
then re-run.

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
│   ├── tasks/network.yml              # LXC's vmbr1 NIC IP (via pct), imported by main.yml
│   ├── tasks/host_idmap.yml           # 1:1 www-data UID/GID lxc.idmap, imported by main.yml
│   ├── tasks/host_network.yml         # Proxmox host's vmbr1 IP, imported by main.yml
│   ├── tasks/host_nfs.yml             # host NFS mount + mp0 bind mount, imported by main.yml
│   ├── tasks/upgrade.yml              # tagged 'upgrade'
│   ├── templates/                     # vhost, php-fpm overrides
│   ├── defaults/main.yml              # non-secret vars
│   └── handlers/main.yml
└── docs/superpowers/specs/            # design docs
```

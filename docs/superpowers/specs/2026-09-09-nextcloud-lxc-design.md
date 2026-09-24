# Nextcloud on Proxmox LXC — Ansible Provisioning Design

Status: approved, pending implementation plan
Date: 2026-09-09

## Goal

Ansible playbook that installs and upgrades a native Nextcloud stack
(Apache + PHP-FPM + MariaDB + Redis) on a Debian 13 unprivileged
Proxmox LXC, with the Nextcloud data directory backed by an NFS export
from a co-located TrueNAS SCALE VM. Run by hand or via cron — no
CI/CD. Must be safely re-runnable at any time, including immediately
after an upgrade.

## Non-goals / explicitly out of scope

- TLS/certificate management — a reverse proxy elsewhere terminates
  HTTPS; the LXC serves plain HTTP.
- Proxmox host network configuration (creating the `vmbr1` internal
  bridge and attaching the LXC's `net1` NIC to it) — one-time host setup,
  documented as a prerequisite, not automated. Only the IP on the LXC's
  existing `net1` is managed (see "Storage NIC" below).
- TrueNAS SCALE dataset/NFS-share creation and export permission
  mapping — one-time NAS-side setup, documented as a prerequisite.
- Docker/AIO — this is a native install by design.

## Component map

```
Proxmox host (bare metal)
├── vmbr0 (LAN)      ── Nextcloud LXC (Debian 13)  ── reverse proxy elsewhere terminates TLS
├── vmbr0 (LAN)      ── TrueNAS SCALE VM
└── vmbr1 (internal, no uplink) ── Nextcloud LXC (net1, 10.10.10.11/24) ↔ TrueNAS VM (10.10.10.10/24)
                                    NFS traffic for /mnt/ncdata rides here; no gateway on 10.10.10.0/24
```

Ansible touches two hosts:
- The Nextcloud LXC — all install/upgrade tasks.
- The Proxmox host — delegated tasks only:
  - every install run: `pct config` (read-only) and, only when the IP is
    missing or wrong, `pct set` on the LXC's `net1` IP;
  - every upgrade run: `pct snapshot` before anything is touched.

It never configures TrueNAS, and never creates or removes Proxmox bridges
or NICs.

## Decisions locked in

| Area | Decision | Why |
|---|---|---|
| Nextcloud version | 34.x | Supports PHP 8.4, which is Debian 13's default apt package — no third-party PHP repo (Sury) needed. |
| PHP | 8.4, from Debian 13's default repos | See above. |
| Web server | Apache + `mod_proxy_fcgi` → PHP-FPM | Closest match to Nextcloud's official docs; `.htaccess` support. |
| TLS | Terminated externally | Reverse proxy elsewhere; out of scope here. |
| LXC SSH | `root`, key-based auth | LXC is single-purpose; root is already the top of the privilege chain, so a separate sudo user + `become` adds a layer with no benefit here. |
| Nextcloud domain | `cloud.home.arpa` (placeholder) | `.home.arpa` is the IETF-reserved TLD for home networks (RFC 8375) — unlike `.local` (mDNS) or invented TLDs. |
| Data directory | `/mnt/ncdata` inside the LXC, NFS-mounted | TrueNAS manages the ZFS pool/dataset; the LXC just mounts the export. |
| NFS export | `tank/nextcloud` dataset → served at `/mnt/tank/nextcloud` | TrueNAS mounts pool/dataset paths under `/mnt/`. |
| Storage network | Isolated `vmbr1`, no physical uplink | Same-host VM/LXC traffic never leaves the bridge either way; isolating it keeps it off LAN broadcast traffic and allows jumbo frames. |
| Storage NIC IP | Set via `pct set <vmid> -net1 ...,ip=<nextcloud_vmbr1_ip>` from the Proxmox host, never with a `gw=` | Proxmox regenerates the container's network config from its netN entries on every boot, so in-container config (netplan/ifupdown/networkd files) would be overwritten or fight it. No gateway keeps the LAN NIC's default route the only one. |
| UID mapping | `www-data` forced to UID 1000 in the LXC; TrueNAS export mapall's to the matching UID | Unprivileged LXC UIDs mean nothing to TrueNAS by default; pinning both sides sidesteps NFS UID-mismatch permission issues. |
| Snapshots | `pct snapshot <vmid>` on the Proxmox host, delegated from the upgrade play | Belt-and-suspenders rollback point before any upgrade touches files. |

## Install path idempotency strategy

Ansible modules (`apt`, `mysql_user`, `mysql_db`, etc.) are naturally
idempotent and need no special handling. Two spots need an explicit
guard because the "obvious" implementation is destructive or errors on
a second run:

1. **Nextcloud download/unpack** — check for
   `/var/www/nextcloud/version.php` (or `occ status`) before
   downloading/extracting. Re-running install must never re-extract
   the tarball over a live install and clobber `config.php`.
2. **`occ maintenance:install`** — check `occ status --output=json`
   for `"installed":true` first. `occ` errors if you try to install
   over an existing instance, so this must be skipped entirely, not
   just retried.

Re-running the **full** playbook after an upgrade must not re-trigger
upgrade logic — the upgrade tasks live only in `tasks/upgrade.yml`
under the `upgrade` tag, so a normal `ansible-playbook playbook.yml`
run never executes that file. It only re-validates the two guards
above, which will both report "already satisfied."

## Data directory / NFS mount

- Install `nfs-common`.
- Force `www-data`'s UID/GID to `1000` (via `user`/`group` modules)
  **before** installing Apache/PHP-FPM, since those packages create
  `www-data` implicitly if it doesn't already exist.
- Before mounting, configure the storage NIC (see below) and wait until
  its IP is live and TrueNAS answers on TCP 2049 over it, so the mount
  never races the NIC. The mount carries `_netdev` so the boot-time fstab
  mount also waits for the network.
- Mount `/mnt/ncdata` via `ansible.posix.mount` with `state: mounted`
  (writes `/etc/fstab` and mounts immediately) pointing at
  `{{ truenas_nfs_host }}:{{ truenas_nfs_export }}`.
- Pre-flight check: verify `/mnt/ncdata` is an actual mounted NFS
  filesystem (not just an empty local directory) before proceeding,
  and fail loudly if not — Nextcloud's data must never silently land
  on the container's own rootfs.

## Storage NIC (`tasks/network.yml`)

Imported by `tasks/main.yml` before the NFS mount.

1. Delegated to Proxmox: `pct config <vmid>`, parse the `net1` line.
2. Assert `net1` exists and is on `vmbr1` — fail loudly otherwise (the NIC
   itself is a manual prerequisite).
3. If `ip` differs from `nextcloud_vmbr1_ip` or any `gw=` is present:
   `pct set <vmid> -net1 <existing keys, ip replaced, gw dropped>`.
   Existing keys (`name`, `bridge`, `hwaddr`, ...) are preserved so the
   MAC never changes. Otherwise nothing is run — safe to re-run.
4. On the LXC: wait for the IP on the interface, assert the default route
   does not use it, and `wait_for` TrueNAS on port 2049.

## Vault & credentials structure

```
roles/nextcloud/defaults/main.yml   # non-secret: versions, paths, domain, VMID, IPs
group_vars/all/vault.yml            # ansible-vault encrypted: db root pw, nc db pw, nc admin pw
```

Task files reference `{{ vault_nextcloud_db_password }}` etc. Nothing
secret ever lands in `defaults/main.yml` or git in plaintext.

## Upgrade path (`tasks/upgrade.yml`, tagged `upgrade`)

Steps run in strict sequence; Ansible's default behavior (a failed
task halts the play) satisfies the "fail loud, don't turn maintenance
mode back off" requirement — no explicit rescue/always block should
undo that.

1. `occ maintenance:mode --on`
2. Delegate to the Proxmox host: `pct snapshot <vmid>
   pre-upgrade-nextcloud-<version>`
3. Copy `config.php` to a timestamped backup path (belt-and-suspenders
   alongside the snapshot)
4. Download + unpack the new release tarball to a fresh path, then
   carry over `config.php`, the `data` symlink/mount (unaffected by
   the swap), and installed custom apps, before swapping
   `/var/www/nextcloud` over
5. `occ upgrade --no-interaction`
6. `occ maintenance:mode --off` — only reached if every prior step
   succeeded

## Variables (placeholders — fill in per environment)

| Variable | Placeholder | Notes |
|---|---|---|
| `nextcloud_lxc_host` | `192.168.1.50` | LAN IP, SSH as root |
| `nextcloud_domain` | `cloud.home.arpa` | vhost + trusted_domains |
| `nextcloud_data_mount` | `/mnt/ncdata` | inside LXC |
| `truenas_nfs_host` | `10.10.10.10` | TrueNAS VM's vmbr1 IP — not its LAN IP |
| `truenas_nfs_export` | `/mnt/tank/nextcloud` | derived from dataset `tank/nextcloud` |
| `nextcloud_www_data_uid` | `1000` | must match TrueNAS mapall target |
| `proxmox_host` | `192.168.1.10` | LAN IP, SSH as root, for `pct snapshot` |
| `nextcloud_lxc_vmid` | `100` | for `pct config` / `pct set` / `pct snapshot` |
| `nextcloud_vmbr1_netif` | `net1` | LXC's Proxmox NIC key on vmbr1 |
| `nextcloud_vmbr1_bridge` | `vmbr1` | asserted before setting the IP |
| `nextcloud_vmbr1_ip` | `10.10.10.11/24` | LXC's vmbr1 IP, no gateway |

## Prerequisites (manual, one-time, outside this playbook)

1. Proxmox: create `vmbr1` with no physical uplink; attach a second
   NIC to the TrueNAS VM on it, and add `net1` on it to the Nextcloud
   LXC with no IP/gateway (the playbook sets the IP).
2. TrueNAS SCALE: give the vmbr1 NIC `10.10.10.10/24` with no gateway;
   create the `tank/nextcloud` dataset, NFS-share it, and set the
   export's mapall user/group to UID/GID `1000` (or whatever
   `nextcloud_www_data_uid` is set to); allow `10.10.10.11` as an NFS
   client.
3. Proxmox LXC: Debian 13 container created, SSH reachable as root
   with a key Ansible can use.

## Repo structure

```
nextcloud-lxc/
├── inventory.ini
├── playbook.yml
├── group_vars/
│   └── all/
│       └── vault.yml          # ansible-vault encrypted
├── roles/
│   └── nextcloud/
│       ├── tasks/
│       │   ├── main.yml       # idempotent install
│       │   ├── network.yml    # vmbr1 NIC IP via pct, imported by main.yml
│       │   └── upgrade.yml    # tagged 'upgrade'
│       ├── templates/         # vhost, php.ini overrides, config.php template if needed
│       ├── defaults/main.yml  # non-secret vars
│       └── handlers/main.yml
└── README.md                  # install vs upgrade usage
```

## Open questions for implementation

None outstanding — all decisions above are confirmed. Next step is an
implementation plan (via the writing-plans skill).

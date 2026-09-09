# Nextcloud on Proxmox LXC — Ansible Provisioning Design

Status: approved, pending implementation plan
Date: 2026-09-09

## Goal

Ansible playbook that installs and upgrades a native Nextcloud stack
(Apache + PHP-FPM + MariaDB + Redis) on a Debian 12 unprivileged
Proxmox LXC, with the Nextcloud data directory backed by an NFS export
from a co-located TrueNAS SCALE VM. Run by hand or via cron — no
CI/CD. Must be safely re-runnable at any time, including immediately
after an upgrade.

## Non-goals / explicitly out of scope

- TLS/certificate management — a reverse proxy elsewhere terminates
  HTTPS; the LXC serves plain HTTP.
- Proxmox host network configuration (the `vmbr1` internal bridge) —
  one-time host setup, documented as a prerequisite, not automated.
- TrueNAS SCALE dataset/NFS-share creation and export permission
  mapping — one-time NAS-side setup, documented as a prerequisite.
- Docker/AIO — this is a native install by design.

## Component map

```
Proxmox host (bare metal)
├── vmbr0 (LAN)      ── Nextcloud LXC (Debian 12)  ── reverse proxy elsewhere terminates TLS
├── vmbr0 (LAN)      ── TrueNAS SCALE VM
└── vmbr1 (internal, no uplink) ── Nextcloud LXC (2nd NIC) ↔ TrueNAS VM (2nd NIC)
                                    NFS traffic for /mnt/ncdata rides here
```

Ansible touches two hosts:
- The Nextcloud LXC — all install/upgrade tasks.
- The Proxmox host — one delegated task per upgrade run, to trigger
  `pct snapshot` before anything is touched.

It never configures TrueNAS or Proxmox networking directly.

## Decisions locked in

| Area | Decision | Why |
|---|---|---|
| Nextcloud version | 34.x | Supports PHP 8.2, which is Debian 12's default apt package — no third-party PHP repo (Sury) needed. |
| PHP | 8.2, from Debian 12's default repos | See above. |
| Web server | Apache + `mod_proxy_fcgi` → PHP-FPM | Closest match to Nextcloud's official docs; `.htaccess` support. |
| TLS | Terminated externally | Reverse proxy elsewhere; out of scope here. |
| LXC SSH | `root`, key-based auth | LXC is single-purpose; root is already the top of the privilege chain, so a separate sudo user + `become` adds a layer with no benefit here. |
| Nextcloud domain | `cloud.home.arpa` (placeholder) | `.home.arpa` is the IETF-reserved TLD for home networks (RFC 8375) — unlike `.local` (mDNS) or invented TLDs. |
| Data directory | `/mnt/ncdata` inside the LXC, NFS-mounted | TrueNAS manages the ZFS pool/dataset; the LXC just mounts the export. |
| NFS export | `tank/nextcloud` dataset → served at `/mnt/tank/nextcloud` | TrueNAS mounts pool/dataset paths under `/mnt/`. |
| Storage network | Isolated `vmbr1`, no physical uplink | Same-host VM/LXC traffic never leaves the bridge either way; isolating it keeps it off LAN broadcast traffic and allows jumbo frames. |
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
- Mount `/mnt/ncdata` via `ansible.posix.mount` with `state: mounted`
  (writes `/etc/fstab` and mounts immediately) pointing at
  `{{ truenas_nfs_host }}:{{ truenas_nfs_export }}`.
- Pre-flight check: verify `/mnt/ncdata` is an actual mounted NFS
  filesystem (not just an empty local directory) before proceeding,
  and fail loudly if not — Nextcloud's data must never silently land
  on the container's own rootfs.

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
| `truenas_nfs_host` | `10.0.30.2` | internal vmbr1 IP |
| `truenas_nfs_export` | `/mnt/tank/nextcloud` | derived from dataset `tank/nextcloud` |
| `nextcloud_www_data_uid` | `1000` | must match TrueNAS mapall target |
| `proxmox_host` | `192.168.1.10` | LAN IP, SSH as root, for `pct snapshot` |
| `nextcloud_lxc_vmid` | `100` | for `pct snapshot <vmid> ...` |

## Prerequisites (manual, one-time, outside this playbook)

1. Proxmox: create `vmbr1` with no physical uplink; attach a second
   NIC to the TrueNAS VM and to the Nextcloud LXC on it.
2. TrueNAS SCALE: create the `tank/nextcloud` dataset, NFS-share it,
   and set the export's mapall user/group to UID/GID `1000` (or
   whatever `nextcloud_www_data_uid` is set to).
3. Proxmox LXC: Debian 12 container created, SSH reachable as root
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
│       │   └── upgrade.yml    # tagged 'upgrade'
│       ├── templates/         # vhost, php.ini overrides, config.php template if needed
│       ├── defaults/main.yml  # non-secret vars
│       └── handlers/main.yml
└── README.md                  # install vs upgrade usage
```

## Open questions for implementation

None outstanding — all decisions above are confirmed. Next step is an
implementation plan (via the writing-plans skill).

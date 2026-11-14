# pve-offsite-backup

- Nightly push proxmox backups
- Custom setup to fit into B2's 10GB free tier.
- Dedup with restic
- metrics for prometheus-node-exporter text exporter

| what | how |
|---|---|
| every running LXC | `vzdump --mode snapshot --stdout` streamed into restic (nothing staged on disk) |
| host config | `/etc` (incl. `/etc/pve`), `/root`, `/usr/local`, `/var/spool/cron`, copy of `/var/lib/pve-cluster/config.db` |
| OPNsense | `config.xml` from `https://<OPN_HOST>/api/core/backup/download/this` |

Retention: 7 daily, 4 weekly, 3 monthly, pruned once a week after a success.
Containers, host files, the cluster DB and config.xml are separate snapshots.
Failed item makes no snapshot, so failures never push out the last good copy.

Stopped containers are listed in the log but not backed up, their last snapshot stays.

It's capped at 10GB in order to stay below B2 free tier. Size is checked every night agaisnt the max.
Over the limit the run fails, so the healthcheck warns before the bucket is full.

## B2
1. Buckets → Create: private, no object lock.
   Bucket → Lifecycle Settings → **Keep only the last version of the file**.
2. Application Keys → Add: access to this bucket only, Read and Write. Note keyID + applicationKey.
3. The bucket page shows the S3 endpoint, e.g. `s3.eu-central-003.backblazeb2.com`.

## OPNsense
1. System → Access → Users → +: name `backup`, generate a scrambled password,
   privilege **Diagnostics: Configuration History** only. Save.
2. Edit it → API keys → + : the browser downloads key + secret.

That privilege also allows reverting/deleting config history through the API.
Don't let it leave the proxmox host.

`OPN_PIN` in env pins the GUI cert's pubkey, so the API key is only ever sent to the real fw.
It keeps working after the self-signed cert expires.
If you ever regenerate the cert, the OPNsense part fails until you recompute the pin (command in env.example).

## Install on PVE
```sh
apt install restic sqlite3
install -m 755 pve-offsite-backup pve-offsite-backup-sizes /usr/local/sbin/
install -d -m 700 /etc/pve-offsite-backup
install -m 600 env.example      /etc/pve-offsite-backup/env
install -m 600 lxc.conf.example /etc/pve-offsite-backup/lxc.conf
openssl rand -base64 32 > /etc/pve-offsite-backup/restic-password
chmod 600 /etc/pve-offsite-backup/restic-password
nano /etc/pve-offsite-backup/env
nano /etc/pve-offsite-backup/lxc.conf
```

**Keep the restic password, B2 key and env values in a password manager.**

## Dry run
Trial into a local repo; `/var/lib/vz`
```sh
export RESTIC_REPOSITORY=/var/lib/vz/restic-trial
pve-offsite-backup restic init
pve-offsite-backup
pve-offsite-backup restic snapshots --json | pve-offsite-backup-sizes   # per container, biggest first
```

Add excludes to `lxc.conf` for what the app can rebuild.
Usuals: the systemd journal, apt caches, app's generated previews and thumbnails, trickplay...
```sh
rm -rf /var/lib/vz/restic-trial; unset RESTIC_REPOSITORY
```

## Upload
```sh
install -m 644 pve-offsite-backup.service pve-offsite-backup.timer /etc/systemd/system/
systemctl daemon-reload                  # units must be installed before they can be started
pve-offsite-backup restic init           # creates the repo in B2
pve-offsite-backup 172                   # one small CT as a smoke test
systemctl start --no-block pve-offsite-backup && journalctl -fu pve-offsite-backup   # first full run
systemctl enable --now pve-offsite-backup.timer
systemctl list-timers pve-offsite-backup.timer
```

## Metrics
Needs `prometheus-node-exporter` on the host and `PROM_TEXTFILE_DIR` in env.
Every nightly run replaces `pve_offsite_backup.prom` there; trial and single-CT runs don't.

| metric (`pve_offsite_backup_...`) | |
|---|---|
| `repo_size_bytes`, `repo_size_limit_bytes` | used vs. `REPO_MAX_MB` |
| `last_run_timestamp_seconds`, `last_success_timestamp_seconds` | success = a night without any failure |
| `last_run_duration_seconds` | |
| `failed_items`, `failure{reason}` | what failed last night |
| `item_size_bytes{item,name}` | tar size per container |
| `item_added_bytes{item,name}` | what last night added per container, stored |
| `item_snapshot_timestamp_seconds{item,name}` | newest snapshot per container |

Alerts:
```
time() - pve_offsite_backup_last_success_timestamp_seconds > 26 * 3600
pve_offsite_backup_repo_size_bytes / pve_offsite_backup_repo_size_limit_bytes > 0.8
```
A run that dies early doesn't touch the file, so the first alert covers that too.
It doesn't cover the host being down; that's what the healthcheck is for.

## Commands
```sh
pve-offsite-backup restic snapshots                  # what's there
pve-offsite-backup restic stats --mode raw-data      # how much of the 10 GB is used
journalctl -u pve-offsite-backup --since yesterday   # last night's log, ends with "repo size: N MB of ..."
pve-offsite-backup restic snapshots --json | pve-offsite-backup-sizes   # what last night added, per container
pve-offsite-backup <pct-id>                          # one-off copy of skipped or stopped CTs (kept until you forget them)
```

When a container fails, the log line says so and vzdump's own reason is in
`/var/log/vzdump/lxc-<vmid>.log`.

## Restore
**One container** (`--unique 1` gives it a new MAC):
```sh
pve-offsite-backup restic dump --path /ct-<id>.tar latest /ct-<id>.tar > /var/lib/vz/dump/vzdump-lxc-<id>-restore.tar

# 9 is there to just show it's a temp.\
# all my lxc's are upriv
pct restore 9<id> /var/lib/vz/dump/vzdump-lxc-<id>-restore.tar --storage local-lvm --unprivileged 1 --unique 1

# replace the original instead: pct stop 172; pct restore 172 <tar> --storage local-lvm --unprivileged 1 --force 1
rm /var/lib/vz/dump/vzdump-lxc-172-restore.tar
```
Older version: `pve-offsite-backup restic snapshots --tag ct-<id>`, then use that ID instead of `latest`.
Paths excluded in `lxc.conf` come back as empty directories.

**OPNsense** (fresh install → System → Configuration → Backups → Restore):
```sh
pve-offsite-backup restic dump --tag opnsense latest /run/pve-offsite-backup/opnsense-config.xml > config.xml
```

**Host files**:
```sh
pve-offsite-backup restic restore latest --tag host --target /root/restore --include /etc/network/interfaces
```

**Restore**:
Restore `/etc/network/interfaces`
then the guest configs (`/etc/pve/lxc`, `/etc/pve/qemu-server`),
then the containers
then OPNsense's config.xml.

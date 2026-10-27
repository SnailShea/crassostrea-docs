# Operations

## Common tasks

| Task | Command |
| --- | --- |
| Update (to the newest release or commit, packages, redeploy, restart) | `sudo ./update.sh` |
| Apply changed settings | `sudo system/configure.sh` |
| State of all services | `systemctl list-units 'crassostrea*' postfix rspamd nginx valkey mariadb unbound 'clamd@*'` |
| Logs of a component | `journalctl -u postfix` (or `rspamd`, `crassostrea-web`, `crassostrea-netfilter`, ...) |
| Mail log | `journalctl SYSLOG_FACILITY=2`, or **Monitoring → Logs** in the web interface (also the other services' journal) |
| Re-write all filter maps | `sudo systemctl start crassostrea-job@sync` |
| Send due digests now | `sudo systemctl start crassostrea-job@send-digests` |
| Synchronise the directories now | `sudo systemctl start crassostrea-job@sync-directories` |
| Make a backup now / list them | `sudo system/backup.sh create` / `sudo system/backup.sh list` |
| Restore a backup | `sudo system/backup.sh restore NAME-OR-PATH`, see [Backups](backups.md) |
| Reset an administrator's password | `sudo helper-scripts/reset-admin.sh [username]` |
| Delete all learning data (Bayes, fuzzy hashes, neural networks) | `sudo helper-scripts/reset-learns.sh` |
| Turn SELinux denials into a local policy module | `sudo helper-scripts/selinux-local-policy.sh` |
| Cluster state | **Cluster** page, or `sudo system/cluster.sh status` |
| Documentation at `/docs/`: deploy, update, remove | `sudo system/docs.sh install\|update\|rebuild\|remove\|status`, see [Documentation on the gateway](installation.md#documentation-on-the-gateway) |
| Put a cluster node into maintenance / take it out | `sudo system/cluster.sh maintenance on\|off [hostname]`, see [Maintenance](high-availability.md#maintenance) |

## Services

| Unit | Purpose |
| --- | --- |
| `postfix` | SMTP (port 25), with a drop-in of the gateway |
| `rspamd` | Filtering |
| `clamd@crassostrea`, `clamav-freshclam` | ClamAV and its signature updates (unless `SKIP_CLAMD=y`) |
| `crassostrea-olefy` | Office macro scanner (unless `SKIP_OLEFY=y`) |
| `mariadb`, `valkey`, `unbound`, `nginx` | Database, key-value store, resolver, web server |
| `crassostrea-web` | Web interface and API (gunicorn, `127.0.0.1:8000`) |
| `crassostrea-web-internal` | Internal API for Rspamd (settings map, quarantine, mail log, reports; `127.0.0.1:8001`, not proxied) |
| `crassostrea-siem` | Sends events to the SIEM ([Monitoring](monitoring.md#siem-export)); idles while none is set |
| `crassostrea-journal` | Sends journal copies to the archive ([Journaling](journaling.md)); idles while none is set |
| `crassostrea-postfix-tlspol` | MTA-STS and DANE policies for outbound TLS |
| `crassostrea-netfilter` | Bans hosts in its own nftables table (`inet crassostrea`) |
| `crassostrea-logship` | Ships the mail log from journald to Valkey (web interface, netfilter), and adds the recipients Postfix refused and the delivery results to the mail log |
| `crassostrea-watchdog` | Health checks, restarts, notifications |
| `crassostrea-postfix-control.socket` | Root helper for mail queue actions, started per request |
| `crassostrea-logs.socket` | Root helper that reads the journal of the gateway's services for **Monitoring → Logs**, started per request |
| `crassostrea-postfix-reload.path` | Applies the relay networks, the message size limit, the rate limits and the PTR check to Postfix when they change (`postfix-apply`) |
| `crassostrea-timezone.path` | Sets the time zone chosen in the web interface (`set-timezone`, see [Time zone](#time-zone)) |
| `crassostrea-hostname.path`, `crassostrea-tls.path`, `crassostrea-network.path`, `crassostrea-backup-request.path` | Root services the web interface asks to change the hostname, certificates and TLS versions, IP addressing, and to make or delete backups; started while a request file in `/var/lib/crassostrea/web/` exists |
| `crassostrea-cluster-report`, `crassostrea-cluster` | Cluster nodes only: report the node's state every 10 seconds and carry out maintenance, Valkey master and membership changes (root), see [Cluster page](high-availability.md#cluster-page) |
| `valkey-sentinel`, `haproxy`, `keepalived`, `garb` | Cluster only, see [High availability](high-availability.md) |

Scheduled jobs (`systemctl list-timers 'crassostrea*'`):

| Timer | Schedule | Job |
| --- | --- | --- |
| `crassostrea-job-cleanup` | daily 03:15 | Expire the quarantine, the mail log, the audit log, reports, statistics and digest tokens; trim the Postfix log |
| `crassostrea-job-digests` | every 15 minutes | Send due quarantine digests |
| `crassostrea-job-sync` | every 10 minutes (every minute in a cluster) | Re-write the maps from the database |
| `crassostrea-job-directories` | every 15 minutes | Synchronise recipients from the [directories](directories.md) |
| `crassostrea-job-alerts` | every 5 minutes | [Alert mail](monitoring.md#alert-mail) |
| `crassostrea-job-accounts` | every 5 minutes | [Accounts for recipients of quarantined mail](quarantine.md#accounts-for-recipients-of-quarantined-mail), if turned on |
| `crassostrea-backup` | daily 02:30 | [Backup](backups.md), then delete old scheduled ones |
| `crassostrea-dmarc-report` | daily 01:15 | Send [DMARC reports](reports.md#sending-dmarc-reports), if turned on |
| `crassostrea-sa-rules` | daily | Update the SpamAssassin rules |
| `crassostrea-clamav-sigs` | every 12 hours | Update the Sanesecurity signatures |
| `crassostrea-acme` | daily | Obtain or renew the certificate |

Times of day are the servers' time zone.

## Time zone

**Settings → General → Time zone** sets the time zone of the whole gateway,
on every node of a cluster:

- The **web interface** shows times in it (with the zone's abbreviation, e.g.
  `CEST`), including the dashboard's charts and the digests. It stores them in
  UTC either way, so changing the zone changes nothing that was recorded.
- The **servers** switch to it: the next sync of the maps (at once on the node
  you save on, within the sync interval on the others) writes
  `/var/lib/crassostrea/web/timezone`, and `crassostrea-timezone.path` runs
  `set-timezone` as root, which sets it with `timedatectl` and restarts the
  services that keep the zone they started with: Rspamd, Postfix, ClamAV,
  Olefy, Unbound, nginx and the gateway's own services (the web interface too,
  so reload the page if it does not answer for a moment). Scheduled jobs run
  at their times in the new zone.
- **MariaDB and Valkey** are not restarted: restarting every Galera node at
  once could stop the database cluster. They use the new zone after their next
  restart, e.g. the next update.

The page shows the zone the servers use, and when the last change failed. With
the zone empty (the default) the servers keep theirs, and the web interface
shows UTC. A zone set on the command line (`timedatectl set-timezone`) stays
until one is chosen in the web interface. The setting is `timezone` (an IANA
name such as `Europe/Berlin`).

## Watchdog

`crassostrea-watchdog` checks nginx, the web interface, MariaDB, Valkey,
unbound, Postfix, the mail queue, postfix-tlspol, Rspamd, ClamAV, Olefy, the
certificate, the last certificate renewal, rate limits and bans, and in a
cluster the Galera state. When a check fails repeatedly it restarts the
affected units (not more often than every 6 minutes) and sends a
notice by mail and/or webhook ([Watchdog notices](monitoring.md#watchdog-notices)).
The thresholds are environment variables of the unit (`*_THRESHOLD`);
override them with a drop-in if needed.

In a cluster the watchdog never restarts the database (Galera rejoins a node by
itself, and a restart cannot help a node that lost the majority), and it
restarts haproxy instead of Valkey when Valkey is unreachable.

## Bans

netfilter watches the mail log and the web interface's failed logins, and
bans networks (`/32` for IPv4 and `/128` for IPv6 by default) after 10
attempts within 10 minutes, for 30 minutes, growing with every ban up to
10000 seconds. Private and loopback addresses, and in a cluster the members'
addresses, are never banned.

**Banned IPs** in the web interface (or `GET /api/v1/bans`) lists the bans and
lifts them. The options are a JSON document in Valkey; change them with
`valkey-cli`:

```bash
REDISPASS=$(sudo sed -n 's/^REDISPASS=//p' /etc/crassostrea/crassostrea.conf)
valkey-cli -a "$REDISPASS" --no-auth-warning GET F2B_OPTIONS
valkey-cli -a "$REDISPASS" --no-auth-warning SET F2B_OPTIONS \
  '{"ban_time": 3600, "max_ban_time": 86400, "ban_time_increment": true, "max_attempts": 5,
    "retry_window": 600, "netban_ipv4": 24, "netban_ipv6": 64, "manage_external": 0}'
```

Permanent bans and exceptions are the Valkey hashes `F2B_BLACKLIST` and
`F2B_WHITELIST`; the field names are networks or hostnames:

```bash
valkey-cli -a "$REDISPASS" --no-auth-warning HSET F2B_BLACKLIST 198.51.100.0/24 1
valkey-cli -a "$REDISPASS" --no-auth-warning HSET F2B_WHITELIST partner-mx.example.org 1
```

netfilter applies changes of the lists within a minute.

## Backup and restore

Backups are made every night and on demand, and restored with
`system/backup.sh restore`; see [Backups](backups.md). Besides them, keep
your own overrides (`/etc/postfix/custom_transport.pcre`,
`custom_postscreen_whitelist.cidr`, `dns_blocklists.cf`,
`/etc/rspamd/override.d/*.custom.inc`, `/etc/rspamd/rspamd.conf.local`) in
version control or copy them along: backups contain `extra.cf` but not these.

In a cluster, a lost node is simply re-joined (`system/cluster.sh bundle` on a
running node, `setup.sh --join` on the new host); it copies everything from the
others.

## Postscreen allow list

postscreen lets the outbound servers of large mail providers through without
its checks. The list is `data/conf/postfix/postscreen_access.cidr` in the
repository; `helper-scripts/update_postscreen_whitelist.sh` regenerates it
with postwhite (it clones spf-tools and postwhite from GitHub). Deploy the new
list with `system/configure.sh`. For your own exceptions use
`/etc/postfix/custom_postscreen_whitelist.cidr` instead.

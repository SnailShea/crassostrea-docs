# Operations

## Common tasks

| Task | Command |
| --- | --- |
| Update (git fast-forward, packages, redeploy, restart) | `sudo ./update.sh` |
| Apply changed settings | `sudo system/configure.sh` |
| State of all services | `systemctl list-units 'spamfilter*' postfix rspamd nginx valkey mariadb unbound 'clamd@*'` |
| Logs of a component | `journalctl -u postfix` (or `rspamd`, `spamfilter-web`, `spamfilter-netfilter`, ...) |
| Mail log | `journalctl SYSLOG_FACILITY=2`, or **Monitoring → Logs** in the web interface (also the other services' journal) |
| Re-write all filter maps | `sudo systemctl start spamfilter-job@sync` |
| Send due digests now | `sudo systemctl start spamfilter-job@send-digests` |
| Synchronise the directories now | `sudo systemctl start spamfilter-job@sync-directories` |
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
| `clamd@spamfilter`, `clamav-freshclam` | ClamAV and its signature updates (unless `SKIP_CLAMD=y`) |
| `spamfilter-olefy` | Office macro scanner (unless `SKIP_OLEFY=y`) |
| `mariadb`, `valkey`, `unbound`, `nginx` | Database, key-value store, resolver, web server |
| `spamfilter-web` | Web interface and API (gunicorn, `127.0.0.1:8000`) |
| `spamfilter-web-internal` | Internal API for Rspamd (settings map, quarantine, mail log, reports; `127.0.0.1:8001`, not proxied) |
| `spamfilter-siem` | Sends events to the SIEM ([Monitoring](monitoring.md#siem-export)); idles while none is set |
| `spamfilter-journal` | Sends journal copies to the archive ([Journaling](journaling.md)); idles while none is set |
| `spamfilter-postfix-tlspol` | MTA-STS and DANE policies for outbound TLS |
| `spamfilter-netfilter` | Bans hosts in its own nftables table (`inet spamfilter`) |
| `spamfilter-logship` | Ships the mail log from journald to Valkey (web interface, netfilter) |
| `spamfilter-watchdog` | Health checks, restarts, notifications |
| `spamfilter-postfix-control.socket` | Root helper for mail queue actions, started per request |
| `spamfilter-logs.socket` | Root helper that reads the journal of the gateway's services for **Monitoring → Logs**, started per request |
| `spamfilter-postfix-reload.path` | Applies the relay networks and the message size limit to Postfix when they change (`postfix-apply`) |
| `spamfilter-hostname.path`, `spamfilter-tls.path`, `spamfilter-network.path`, `spamfilter-backup-request.path` | Root services the web interface asks to change the hostname, certificates and TLS versions, IP addressing, and to make or delete backups; started while a request file in `/var/lib/spamfilter/web/` exists |
| `spamfilter-cluster-report`, `spamfilter-cluster` | Cluster nodes only: report the node's state every 10 seconds and carry out maintenance, Valkey master and membership changes (root), see [Cluster page](high-availability.md#cluster-page) |
| `valkey-sentinel`, `haproxy`, `keepalived`, `garb` | Cluster only, see [High availability](high-availability.md) |

Scheduled jobs (`systemctl list-timers 'spamfilter*'`):

| Timer | Schedule | Job |
| --- | --- | --- |
| `spamfilter-job-cleanup` | daily 03:15 | Expire the quarantine, the mail log, the audit log, reports, statistics and digest tokens; trim the Postfix log |
| `spamfilter-job-digests` | every 15 minutes | Send due quarantine digests |
| `spamfilter-job-sync` | every 10 minutes (every minute in a cluster) | Re-write the maps from the database |
| `spamfilter-job-directories` | every 15 minutes | Synchronise recipients from the [directories](directories.md) |
| `spamfilter-job-alerts` | every 5 minutes | [Alert mail](monitoring.md#alert-mail) |
| `spamfilter-backup` | daily 02:30 | [Backup](backups.md), then delete old scheduled ones |
| `spamfilter-dmarc-report` | daily 01:15 | Send [DMARC reports](reports.md#sending-dmarc-reports), if turned on |
| `spamfilter-sa-rules` | daily | Update the SpamAssassin rules |
| `spamfilter-clamav-sigs` | every 12 hours | Update the Sanesecurity signatures |
| `spamfilter-acme` | daily | Obtain or renew the certificate |

## Watchdog

`spamfilter-watchdog` checks nginx, the web interface, MariaDB, Valkey,
unbound, Postfix, the mail queue, postfix-tlspol, Rspamd, ClamAV, Olefy, the
certificate, the last certificate renewal, rate limits and bans, and in a
cluster the Galera state. When a check fails repeatedly it restarts the
affected units (not more often than every 6 minutes) and sends a
notification by mail (`WATCHDOG_NOTIFY_EMAIL`) and/or webhook
(`WATCHDOG_NOTIFY_WEBHOOK`). The thresholds are environment variables of the
unit (`*_THRESHOLD`); override them with a drop-in if needed.

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
REDISPASS=$(sudo sed -n 's/^REDISPASS=//p' /etc/spamfilter/spamfilter.conf)
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

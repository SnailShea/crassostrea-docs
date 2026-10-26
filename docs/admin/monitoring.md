# Monitoring

The gateway checks itself (the [watchdog](operations.md#watchdog) restarts
unhealthy services and notifies you). **Monitoring** shows the alert checks and
the state of every service (systemd unit, since when, in a cluster those of
the node you are logged in to), and has the [logs](#logs). **Settings → Alerts
& export** hands the state to the tools you already use: Prometheus, a SIEM
and alert mail.

When a check fails, a red badge with the number of problems appears next to
**Monitoring** in the sidebar on every page (full administrators); its tooltip
names them. The alert job records the state every 5 minutes, also without
alert recipients, and opening Monitoring updates it at once.

## Prometheus

`https://<hostname>/metrics` answers in the Prometheus text format to requests
with a bearer token. **Create a token** on the page; it is shown once (only
its hash is stored), and a new token replaces the old one. **Revoke** turns the
endpoint off.

```yaml
scrape_configs:
  - job_name: mail-gateway
    scheme: https
    authorization:
      credentials: sfm_...
    static_configs:
      - targets: ['filter.example.com']
```

| Metric | Labels | Meaning |
| --- | --- | --- |
| `sf_up` | `component` (database, valkey, rspamd) | 1 if it answers |
| `sf_service_active` | `unit` | 1 if the systemd unit is active |
| `sf_queue_messages` | `queue` (maildrop, incoming, active, deferred, hold) | Messages in the Postfix queue |
| `sf_queue_deferred_oldest_seconds` | | Age of the oldest deferred message |
| `sf_quarantine_messages`, `sf_mail_log_messages` | | Rows in the quarantine and the mail log |
| `sf_messages_1h`, `sf_messages_24h` | `direction`, `outcome` | Scanned messages (from the [statistics](mail-log.md#statistics)) |
| `sf_certificate_expiry_seconds` | | Until the certificate expires |
| `sf_backup_last_timestamp_seconds`, `sf_backup_last_failed` | | The newest backup, and whether the last one failed |
| `sf_directory_sync_ok` | `directory` | 1 if the last synchronisation worked |

Rspamd's own metrics (scan counts, symbol hits, Bayes) are at
`https://<hostname>/rspamd/metrics`, where the Rspamd UI is reachable
(`RSPAMD_UI_ALLOW`).

In a cluster each node answers with its own queue and services; scrape every
node by its hostname. Every node also answers with the cluster as its nodes
report it (see the [Cluster page](high-availability.md#cluster-page)):

| Metric | Labels | Meaning |
| --- | --- | --- |
| `sf_cluster_votes` | `kind` (total, quorum, present) | Votes of all members, needed for a majority, in the database cluster |
| `sf_cluster_problems` | `level` (critical, warning) | Problems on Monitoring → Cluster |
| `sf_cluster_member_reporting`, `sf_cluster_member_report_age_seconds` | `member` | Whether a node reported in the last minute, and when |
| `sf_cluster_member_maintenance` | `member` | 1 if the node is in maintenance |
| `sf_cluster_member_database_synced` | `member` | 1 if its database is synced in the primary component |
| `sf_cluster_member_valkey_master`, `sf_cluster_member_valkey_lag_bytes` | `member` | The Valkey master; how far a replica is behind |
| `sf_cluster_member_queue_messages` | `member`, `queue` | Each node's Postfix queue |
| `sf_cluster_witness_up` | `component` (sentinel, arbitrator) | Whether the witness takes part |

## SIEM export

Every scanned message (from the mail log) and every [audit log](administrators.md#audit-log)
entry can be sent to a SIEM, one event per line:

| Setting | Values |
| --- | --- |
| Host, port | The receiver; empty host: off. Port 514 by default |
| Protocol | TCP (default), TCP with TLS (certificate checked unless turned off), UDP |
| Format | JSON, one object per line; or syslog (RFC 5424, facility local0) with the JSON as message |
| Events | Mail log, audit log (both on) |

Mail events carry time, node, direction, queue ID, Message-ID, sender,
recipients, `From`, subject, score, action, client, size, whether it had a
virus, and the names of the matching rules. Audit events carry time, node,
who, address, action and its description, target, result and message.

Events are queued in Valkey (at most 100,000; the oldest are dropped while the
SIEM is away) and sent by `crassostrea-siem.service`, so a slow or missing
receiver never holds up mail. **Send a test event** checks the connection.

## Alert mail

Mail to the alert recipients (one per line) when a check fails:

| Check | Fails when |
| --- | --- |
| Deferred mail | the deferred queue has at least *n* messages (default 100) |
| Mail waiting | the oldest deferred message waits longer than *n* hours (default 6) |
| Certificate | it expires within *n* days (default 14), or there is none |
| Backup | the last backup failed, or the newest is older than *n* hours (default 36) |
| Directories | a [directory](directories.md) does not synchronise |
| Services | a unit of the gateway is not running |

In a cluster the deferred mail, mail waiting and services checks are made for
every node, from its report (e.g. *Deferred mail (mx2.example.com)*), and the
problems of the [Cluster page](high-availability.md#cluster-page) are checks of
their own (a node that stopped reporting, the database, Valkey, the witness,
the floating address, versions, disk space, maintenance). The checks of a node
that stopped reporting keep their state until it reports again.

`crassostrea-job-alerts.timer` checks every 5 minutes, from one node of a
cluster. A mail goes out when a check starts failing, again every 24 hours
while it fails, and once when it is solved. The page shows the checks as they
are now. **Send a test alert** checks the delivery. Alerts come from
`alerts@<hostname>` through the local Postfix. API:
`GET /api/v1/monitoring/checks`, settings `alert_*` and `siem_*`.

The watchdog's notifications (`WATCHDOG_NOTIFY_*`) are independent: it covers
the services themselves and works without the web interface and the database.

## Logs

**Monitoring → Logs** shows the most recent lines of one service at a time,
newest first; the row of names above the list switches between them:

| Service | Covers |
| --- | --- |
| Postfix | The mail log: connections, deliveries, rejections. In a cluster from all nodes, with the node's name (from Valkey, written by `crassostrea-logship`) |
| Rspamd | Scans, maps, Lua errors |
| ClamAV | `clamd@crassostrea`, `clamav-freshclam`, the Sanesecurity updates, Olefy |
| Web interface | `crassostrea-web`, `crassostrea-web-internal` |
| nginx | The web server's journal (its access log stays in `/var/log/nginx`) |
| Database | MariaDB, in a cluster also the Galera arbitrator |
| Valkey | Valkey, in a cluster also Sentinel and haproxy |
| DNS resolver | unbound |
| Postfix service | `postfix.service` itself, the MTA-STS/DANE policy service, reloads |
| Bans | `crassostrea-netfilter`, `crassostrea-logship` |
| Watchdog | `crassostrea-watchdog` |
| Certificates | `crassostrea-acme`, the certificate changes of the web interface |
| Jobs | Cleanup, digests, map sync, directories, alerts, backups, DMARC reports, rule updates |
| SIEM & journaling | `crassostrea-siem`, `crassostrea-journal` |
| Root services | Hostname and network changes, mail queue actions, the log reader |
| Cluster | `crassostrea-cluster`, `crassostrea-cluster-report`, keepalived (only in a cluster) |

Filter by text (the message or the unit) and, except for the Postfix log, by
level (warnings and errors only, for example). The page updates itself every
10 seconds and shows the last 500 lines.

The journal is readable only by root: `crassostrea-logs.socket` starts a small
root helper (`system/libexec/log-reader`) per request, which reads only the
units it lists itself; the web interface only names the service. Apart from
the Postfix log, a cluster node shows its own journal: open the page on each
node, by its name.

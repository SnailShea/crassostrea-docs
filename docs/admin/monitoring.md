# Monitoring

The gateway checks itself (the [watchdog](operations.md#watchdog) restarts
unhealthy services and notifies you). **Monitoring** hands its state
to the tools you already use: Prometheus, a SIEM and alert mail.

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
| `sf_cluster_problems` | `level` (critical, warning) | Problems on the Cluster page |
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
SIEM is away) and sent by `spamfilter-siem.service`, so a slow or missing
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

`spamfilter-job-alerts.timer` checks every 5 minutes, from one node of a
cluster. A mail goes out when a check starts failing, again every 24 hours
while it fails, and once when it is solved. The page shows the checks as they
are now. **Send a test alert** checks the delivery. Alerts come from
`alerts@<hostname>` through the local Postfix. API:
`GET /api/v1/monitoring/checks`, settings `alert_*` and `siem_*`.

The watchdog's notifications (`WATCHDOG_NOTIFY_*`) are independent: it covers
the services themselves and works without the web interface and the database.

# Backups

## What a backup contains

A backup is a `tar.gz` file in `/var/lib/spamfilter/backups/`:

| Part | Content |
| --- | --- |
| `manifest.json` | When and where it was made, and what it contains |
| `database.sql` | The database: domains, accounts, lists, settings, DKIM keys, audit log, reports, statistics. The quarantine's messages by default, the mail log's only if chosen |
| `valkey.rdb` | Valkey's data: Bayes and fuzzy hashes, bans, greylisting, link protection |
| `files/` | `spamfilter.conf`, the certificate, the ACME account and DNS-01 credentials, `/etc/postfix/extra.cf`, the cluster PKI |

!!! warning
    A backup holds every password and private key of the gateway. The backup
    directory is readable by root and the web interface only; keep downloaded
    copies as safe as the server itself.

## Making backups

- **Every night** around 02:30 (`spamfilter-backup.timer`) a scheduled backup
  is made. The newest 7 scheduled backups are kept (1 to 90, on the page);
  older ones are deleted.
- **On demand**: **Back up now** on **Settings → Backups**, or
  `sudo system/backup.sh create`. Manual backups are kept until you delete
  them.

The page lists the backups with their size and contents, downloads them
(recorded in the [audit log](administrators.md#audit-log)) and deletes them.
Its settings choose whether the quarantine and the mail log's messages are
included (the mail log can make backups much larger). API:
`/api/v1/backups`.

The web interface cannot read the configuration or the database dump it would
need: it asks the root service `spamfilter-backup-request.path`, which makes
or deletes the backup and records the outcome. A failed backup is shown on the
page, in the metrics and as an [alert](monitoring.md#alert-mail).

Copy backups off the server regularly (download them, or fetch them with
`scp`/`rsync` as root): a backup on the gateway does not help if the gateway is
lost.

## Restoring

Restoring stops the services, so it is done on the server, as root:

```bash
sudo system/backup.sh list
sudo system/backup.sh restore backup-20261021-023012-scheduled.tar.gz   # a name in the backup directory
sudo system/backup.sh restore /root/backup.tar.gz                        # or a path
```

It shows what the backup contains and asks for confirmation (`--yes` skips
the question), then:

1. saves the current configuration, files and database to
   `/var/lib/spamfilter/restore-<time>/`;
2. stops Postfix, Rspamd and the web interface;
3. puts back the files, the database and Valkey's data (not when Valkey uses
   append-only files);
4. runs `system/configure.sh`, which renders the configuration of the restored
   settings (hostname included) and starts everything.

### Moving the gateway to a new server

Install the new server with `setup.sh` (any hostname), copy a backup over and
restore it there. The restored `spamfilter.conf` brings the hostname,
passwords and certificate along. Then move the MX records or the address.

### Clusters

A cluster member is not restored: the other members hold the same data. To
recover a whole cluster from a backup, restore it on one node as a single
gateway, then build the cluster again ([High availability](high-availability.md)).

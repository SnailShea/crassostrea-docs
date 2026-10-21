# Web interface

The web interface is at `https://<hostname>/`. Administrators see the full
menu; [domain administrators](administrators.md) see the pages of their
domains only; mailbox users see only their own quarantine and settings (see
[Quarantine and digests](quarantine.md#mailbox-users)).

## Pages

The menu lists groups of related pages; a group with several pages shows them
as tabs at the top. Domain administrators see only the groups and tabs of
their pages, and no tabs where only one page is left.

| Menu | Tabs | Purpose |
| --- | --- | --- |
| Dashboard | | Scan counters of Rspamd, quarantine and domain counts, health of database, Valkey and Rspamd, recent mail, state of all services; in a cluster also the database and Valkey replication state |
| Statistics | Statistics | Scanned messages over time by outcome, see [Mail log and statistics](mail-log.md#statistics) |
| | DMARC & TLS reports | Reports about your domains, see [DMARC and TLS reports](reports.md) |
| **Mail** | | |
| Quarantine | | Rejected mail, see [Quarantine and digests](quarantine.md) |
| Mail log | Mail log | Every scanned message with its score breakdown, text, HTML and headers; download, queue again, see [Mail log and statistics](mail-log.md) |
| | Mail queue | Messages waiting for delivery: flush, deliver now, hold, release hold, delete. In a cluster: the queue of the node you are logged in to |
| | Postfix log | The most recent mail log lines, searchable; in a cluster from all nodes, with the node's name |
| | Journaling | Copies of all mail to an archive, see [Journaling](journaling.md) |
| **Filter** | | |
| Domains | Domains | Protected domains, their recipients, lists and DKIM keys, see [Domains and delivery](domains.md) |
| | Directories | Recipients from LDAP, Active Directory or Microsoft 365, see [Recipient directories](directories.md) |
| | Mailbox users | Accounts of recipients |
| Protection | Impersonation, Link protection, Attachments, Content rules | See [Threat protection](protection.md) |
| Lists & training | Allow & block lists | Global entries, see [Filtering](filtering.md#allow-and-block-lists) |
| | Training | Teach Bayes and the fuzzy storage; reports from users, see [Filtering](filtering.md#training) |
| Outbound | Relay networks | The servers that may send outbound mail through the gateway, see [Domains and delivery](domains.md#outbound-mail) |
| | Outbound limits, Data loss prevention | See [Outbound mail](outbound.md) |
| **System** | | |
| Settings | General | Global thresholds, message limits and mail log retention, quarantine and digests, administrators (two-factor requirement, audit log retention) |
| | Branding | Name, colour, logo, footer, see [Branding](#branding) |
| | Backups | Backups, see [Backups](backups.md) |
| Network | Hostname & IP | The gateway's hostname, addresses and IP families, see [Hostname, IP addressing and DNS](network.md) |
| | Certificates & TLS | Let's Encrypt or your own certificate, TLS versions, see [Certificates and TLS](certificates.md) |
| | DNS & ports | Checks of the records and connections the gateway needs, see [Hostname, IP addressing and DNS](network.md#dns-and-ports) |
| | Banned IPs | Hosts blocked by netfilter, with unban |
| Monitoring | Monitoring | Prometheus token, SIEM export, alert mail, see [Monitoring](monitoring.md) |
| | Cluster | Every member's state, the cluster's problems, maintenance and the Valkey master, see [High availability](high-availability.md#cluster-page); on a single gateway how to set up a cluster |
| Access | Administrators | Accounts and their roles, see [Administrators and audit log](administrators.md) |
| | Audit log | Who changed what, see [Administrators and audit log](administrators.md#audit-log) |
| | API keys | Keys for the [REST API](api.md) |
| | API documentation | Swagger UI of the API |
| Rspamd UI | | `https://<hostname>/rspamd/`, password `RSPAMD_PASSWORD` |

The addresses of the pages did not change with the menu (e.g. `/settings/networks`
for the relay networks, `/ip/` for the hostname and IP addressing).

The hostname, IP addressing and Certificates & TLS change only a single
gateway; in a cluster they show the state and explain what to do instead.

## Live pages

The dashboard, statistics, mail log, quarantine, mail queue, Postfix log,
banned IPs, audit log, journaling, outbound limits, link protection, cluster, and the
status parts of IP addressing, Certificates & TLS and Backups update
themselves every 5 to 60 seconds. A line above the content shows when it was
last updated and has a **Pause** button. Updates also pause while you have
selected entries (a checked box) or are working in the list, and while the
browser tab is hidden.

Actions in lists (queue actions, unban, quarantine actions, removing list
entries, enabling or deleting accounts) apply without reloading the page; the
result message appears above the list.

Everything also works without JavaScript, with normal page loads.

## Your account

Click your username in the menu to change your password and, as an
administrator, to set up [two-factor login](administrators.md#two-factor-login).
Logins end 8 hours after they began (`WEB_SESSION_LIFETIME`); see
[Logins](administrators.md#logins).

## API keys

**API keys** creates keys for the REST API. A key is shown once; only a hash
is stored. Options:

- **Read-only**: only GET requests.
- **Allowed networks**: comma separated addresses or networks the key may be
  used from; empty allows every address.

The list shows when each key was last used. Delete keys you no longer need.
API keys always have full access; only full administrators manage them.

## Branding

**Settings → Branding** sets the product name, an accent colour, a logo, a
favicon, a footer text (shown on all pages and the login page, e.g. support
contact details) and the default appearance (light, dark or following the
system). The logo is shown above the name in the menu. The theme toggle in
the menu lets each user override the appearance.

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
| Dashboard | | The average [processing time](mail-log.md#processing-time) (scan and delivery), for full administrators the [mail queue by direction and the use of CPU, memory and storage](mail-log.md#the-gateway-on-the-dashboard); quarantine and domain counts, the busiest domains and the [statistics](mail-log.md#statistics) of the scanned mail, hourly or daily; for full administrators also the health of the database, Valkey and Rspamd and, in a cluster, its state. The mail log lists the messages |
| **Mail** | | |
| Quarantine | | Rejected mail, see [Quarantine and digests](quarantine.md) |
| Mail log | Mail log | Every scanned message with its score breakdown, text, HTML and headers; download, queue again, see [Mail log and statistics](mail-log.md) |
| | Mail queue | Messages waiting for delivery: flush, deliver now, hold, release hold, delete. In a cluster: the queue of the node you are logged in to |
| | Journaling | Copies of all mail to an archive, see [Journaling](journaling.md) |
| DMARC & TLS reports | | Reports about your domains, see [DMARC and TLS reports](reports.md) |
| **Filter** | | |
| Domains | Domains | Protected domains, their recipients, lists and DKIM keys, see [Domains and delivery](domains.md) |
| | Directories | Recipients from LDAP, Active Directory or Microsoft 365, see [Recipient directories](directories.md) |
| Protection | Impersonation, Link protection, Attachments, Content rules, Rate limits | See [Threat protection](protection.md) |
| Lists & training | Allow list | Global allow entries, see [Filtering](filtering.md#allow-and-block-lists) |
| | Block list | Global block entries |
| | Training | Teach Bayes and the fuzzy storage; reports from users, see [Filtering](filtering.md#training) |
| Outbound | Outbound limits, Data loss prevention | See [Outbound mail](outbound.md) |
| **System** | | |
| Settings | General | Global thresholds and the greylisting switch, message limits and mail log retention, quarantine and digests (and accounts for recipients of quarantined mail), virus scanning, the PTR check and the Spamhaus DQS key, administrators (two-factor requirement, audit log retention, allowed networks, sign-in: login lifetime and throttling), the web server (HTTP to HTTPS, Rspamd UI networks, trusted reverse proxies), the time zone. Links at the top jump to each section; the save button stays in view and says when there are unsaved changes, and leaving the page with unsaved changes asks first |
| | Branding | Name, colour, logo, footer, see [Branding](#branding) |
| | Relay networks | The servers that may send outbound mail through the gateway, see [Domains and delivery](domains.md#outbound-mail) |
| | Alerts & export | Prometheus token, SIEM export, alert mail, the watchdog's notices, see [Monitoring](monitoring.md) |
| | Cluster | Create a cluster or join one (single gateway); join information, maintenance, the Valkey master, removing members (cluster), see [High availability](high-availability.md#setting-up-a-cluster) |
| | Backups | Backups, see [Backups](backups.md) |
| Network | Hostname & IP | The gateway's hostname, and the addresses and IP families of each network interface, see [Hostname, IP addressing and DNS](network.md) |
| | Certificates & TLS | Let's Encrypt or your own certificate, TLS versions, see [Certificates and TLS](certificates.md) |
| | DNS & ports | Checks of the records and connections the gateway needs, see [Hostname, IP addressing and DNS](network.md#dns-and-ports) |
| | Banned IPs | Hosts blocked by netfilter, with unban |
| Monitoring | Monitoring | The deployed version and the update channel, the alert checks and the state of every service, see [Monitoring](monitoring.md) |
| | Logs | The Postfix log (all nodes) and the journal of the other services, one service at a time, see [Monitoring](monitoring.md#logs) |
| | Cluster | Every member's state and the cluster's problems, see [High availability](high-availability.md#cluster-page) |
| Users | Administrators | Accounts and their roles, see [Administrators and audit log](administrators.md) |
| | Mailbox users | Accounts of recipients, see [Quarantine and digests](quarantine.md#mailbox-users) |
| Access | Audit log | Who changed what, see [Administrators and audit log](administrators.md#audit-log) |
| | API keys | Keys for the [REST API](api.md) |
| | API documentation | Swagger UI of the API |
| Documentation | | This documentation at `https://<hostname>/docs/`, if it is [deployed on the gateway](installation.md#documentation-on-the-gateway) |
| Rspamd UI | | `https://<hostname>/rspamd/`, password `RSPAMD_PASSWORD` |

The addresses of the pages did not change with the menu (e.g. `/settings/networks`
for the relay networks, `/ip/` for the hostname and IP addressing).

The hostname, IP addressing and Certificates & TLS change only a single
gateway; in a cluster they show the state and explain what to do instead.

## In-place actions

Pages do not update themselves: reload a page to see what changed since it
was opened (a running backup, a staging test, new mail).

Actions in lists (queue actions, unban, quarantine actions, removing list
entries, enabling or deleting accounts) apply without reloading the page; the
result message appears above the list.

Everything also works without JavaScript, with normal page loads.

## Your account

Click your username in the menu to change your password and, as an
administrator, to set up [two-factor login](administrators.md#two-factor-login).
Logins end 8 hours after they began (Settings → General, Sign-in); see
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
the menu lets each user override the appearance. The API documentation
follows the same light or dark appearance and accent colour.

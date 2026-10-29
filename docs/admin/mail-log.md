# Mail log and statistics

## Mail log

**Mail log** lists every message Rspamd scanned for the protected domains and
from the relay networks, newest first: time, direction, sender, recipients,
subject, score and action. Filter it by domain, direction (inbound/outbound)
and action, or search the subject, sender, recipients, `From` header,
Message-ID, queue ID or client address.

### Delivery

The mail log also shows what Postfix did with a message after it was scanned.
The list marks each message **delivered** (every recipient's server accepted
it), **deferred** (Postfix is still retrying a recipient) or **bounced** (a
recipient's server refused it, or Postfix gave up after the queue lifetime,
5 days), and **bounce sent** when the gateway sent the sender a non-delivery
notice. The **delivery** filter finds them (`delivery=` in
`GET /api/v1/mail-log`).

A message's page has a **Delivery** card: for each recipient the latest
attempt with the receiving server, the DSN code and its answer (e.g.
`550 5.1.1 User unknown`), the number of attempts and when the last one was;
whether Postfix gave up; and the notices the gateway sent the sender (bounce,
delay warning, delivery status notification) with their own queue ID and
whether they arrived. A rejected message has no delivery. The API returns the
same as `deliveries` of `GET /api/v1/mail-log/{id}`.

The notices themselves are not mail log entries: Postfix creates them without
a scan. `crassostrea-logship` finds the delivery lines in the Postfix log of
each node and reports them to the internal API; they are matched to messages
by queue ID (Postfix uses long queue IDs, which are never reused) and kept as
long as the mail log; Postfix's delay of each delivered recipient goes to
the [processing time](#processing-time). Delivery results start with the
update that added them.

### Refused recipients

For domains that validate recipients (a recipient list, a directory, or
verification with the destination server), Postfix refuses an address that
does not exist during the SMTP dialogue, before the message is sent; Rspamd
never sees it. The mail log shows these too, one row per refused recipient:
action **reject**, the sender, the recipient, the client address and Postfix's
reason ("User unknown in relay recipient table", or "undeliverable address:
..." from the destination server) instead of a subject. Choose **refused
recipients** in the action filter to see only them (`action=refused` in
`GET /api/v1/mail-log`, whose entries carry the reason as `refused`).

They have no message: no score, content, download or queueing again, and
their page links to the Postfix log lines of the recipient. They are kept as
long as the mail log, go to the SIEM with the reason, are not journaled, and
and are not counted in the statistics, so a dictionary attack does not
inflate them. Only permanent refusals
(5xx) are logged; a temporary one (the destination server could not be asked)
is retried by the sender. `crassostrea-logship` finds them in the Postfix log on
each node and reports them to the internal API.

A message's page shows:

- its details: queue ID, Message-ID, client address, size, scan time;
- the **score breakdown**: every Rspamd rule that matched, with its score and
  details, highest first, and for common rules what they mean;
- the **text** and, for messages with an HTML body, a switch between the plain
  text and the **HTML** (see below);
- attachments and all headers;
- links to its quarantine entries (if it was rejected) and to its lines in the
  Postfix log (Monitoring → Logs).

Actions:

| Action | Effect |
| --- | --- |
| Download | The message as scanned (`.eml`) |
| Queue again | Deliver it once more to some or all of its recipients, without a second scan. It is marked with an `X-Mail-Log-Requeued` header; outbound mail is DKIM-signed for the sender's domain. A message with a virus needs an explicit confirmation |

### HTML view

The HTML of a message (here and in the quarantine) is shown sanitized in a
sandboxed frame: no scripts, no forms, no links that can be followed and no
remote content. Images embedded in the message are shown; images from the
internet are not loaded, so opening a message does not tell the sender that
it was read. Use it to judge what a recipient
would have seen; the plain text view stays the default.

### Retention and size

Messages are kept for **7 days** by default, up to 31 (**Settings → General**,
Messages), and deleted by the daily cleanup. Every scanned message is stored in
full, so plan disk space: the mail log is usually the largest table. In a
cluster it is in the replicated database, on every node. Backups leave the
mail log's messages out unless you include them (see [Backups](backups.md)).

The mail log is no archive; for long-term copies of all mail see
[Journaling](journaling.md).

## Message limits

**Settings → General** (Messages) applies to inbound and outbound mail:

| Setting | Default | Effect |
| --- | --- | --- |
| Maximum message size | 50 MiB (at most 100) | Postfix's `message_size_limit`: larger messages are refused during the SMTP dialogue. Applied by `crassostrea-postfix-reload.service` within seconds |
| Maximum attachment size | 0 (no limit) | Rspamd rejects messages with a larger attachment (its decoded size), symbol `ATTACHMENT_TOO_LARGE` |

Every accepted message is scanned in full, by ClamAV too: up to 128 MiB per
message and file, 90 seconds per scan, archives to ClamAV's default depth and
number of files. A message ClamAV cannot scan completely is not let through
silently: it is scored `CLAM_SCAN_INCOMPLETE` (+4).

## Statistics

The **dashboard** shows the scanned messages over time, by outcome:

| Outcome | Rspamd actions |
| --- | --- |
| Clean | no action |
| Marked as spam | add header, rewrite subject |
| Greylisted or deferred | greylist, soft reject |
| Rejected | reject |

and the number of viruses, as totals and as a bar chart. The dropdown of the
**Over time** card switches between **hourly** (the last 24 hours, one bar
per hour) and **daily** (the last 30 days, one bar per day); the totals, the
processing time and the busiest domains follow it. Domain administrators see
their domains only. `GET /api/v1/statistics` also has 7 days (per 6 hours)
and 12 months (per 30 days), by direction and domain.

The counts are hourly totals per domain, direction and outcome, added as the
mail log stores each message, and kept for **two years** (setting
`stats_retention_days`, through the API) — much longer than the mail log. The
first start after the update that introduced them fills them from the
existing mail log once.

Mail refused before Rspamd saw it (postscreen, unknown recipients, unknown
domains) is not counted; the Postfix log (Monitoring → Logs) shows it.

### Processing time

The dashboard's **Processing time** card shows two averages of the last 24
hours or 30 days (the **Over time** card's view); `GET /api/v1/statistics`
also has them per hour or day:

- **Scan**: Rspamd's time per message (the *Scan time* of a mail log entry),
  counted with the statistics, so it covers the whole period. Hours from
  before the update that added it have no scan time and are left out.
- **Delivery**: from the message's arrival at the gateway until the next
  server (the domain's destination host, or the recipient's server for
  outbound mail) accepted it, per delivered recipient: Postfix's `delay` of
  the [delivery](#delivery). It includes the time deferred mail waited for a
  retry, so one recipient that was unreachable for hours raises it. It is
  counted per hour as Postfix delivers, kept as long as the statistics, and
  starts with the update that added it.

### The gateway on the dashboard

The dashboard starts with the processing time and, for full administrators,
the mail queue and the system load of the node they are logged in to; the
quarantine, domain and health cards and the busiest domains follow, then the
statistics and the chart. The page does not update itself; reload it.

- **Mail queue**: the messages Postfix is delivering, the deferred ones and
  those on hold, inbound and outbound, with their size and how long the
  oldest deferred one waits. The direction is the mail log's; mail it does
  not know (notices to senders, mail no longer in the mail log) is inbound
  when it goes to a protected domain. Mail queue (Mail log → Mail queue)
  lists the messages.
- **System load**: the use of the CPUs (as the page loads), of memory (what
  is not available to programs) and
  of each filesystem of `/`, `/var`, `/var/lib/mysql` and
  `/var/spool/postfix` (as `df` counts it). The meters turn amber from 75%
  and red from 90%. In a cluster the alert mail also warns of a filesystem
  90% full (Monitoring).

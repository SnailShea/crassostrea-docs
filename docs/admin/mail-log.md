# Mail log and statistics

## Mail log

**Mail log** lists every message Rspamd scanned for the protected domains and
from the relay networks, newest first: time, direction, sender, recipients,
subject, score and action. Filter it by domain, direction (inbound/outbound)
and action, or search the subject, sender, recipients, `From` header,
Message-ID, queue ID or client address. The list updates itself like the other
live pages.

It replaces the earlier *Scan history*, which only kept Rspamd's last scans in
Valkey: the mail log keeps each message in full, in the database.

A message's page shows:

- its details: queue ID, Message-ID, client address, size, scan time;
- the **score breakdown**: every Rspamd rule that matched, with its score and
  details, highest first;
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
| Maximum message size | 50 MiB (at most 100) | Postfix's `message_size_limit`: larger messages are refused during the SMTP dialogue. Applied by `spamfilter-postfix-reload.service` within seconds |
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

and the number of viruses, as totals and as a bar chart, for the last 24 hours
(one bar per hour), 7 days (per 6 hours), 30 days (per day) or 12 months (per
30 days). Filter by direction and domain; the busiest domains, the quarantine
counts and the recent mail follow the filter. Domain administrators see their
domains only. The same data is available as `GET /api/v1/statistics`; the
former Statistics page (`/statistics/`) leads to the dashboard.

The counts are hourly totals per domain, direction and outcome, added as the
mail log stores each message, and kept for **two years** (setting
`stats_retention_days`, through the API) — much longer than the mail log. The
first start after the update that introduced them fills them from the
existing mail log once.

Mail refused before Rspamd saw it (postscreen, unknown recipients, unknown
domains) is not counted; the Postfix log (Monitoring → Logs) shows it.

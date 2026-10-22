# Quarantine and digests

## What is quarantined

When Rspamd rejects a message, a copy is stored in the quarantine, one entry
per recipient, if:

- the recipient's domain has "Quarantine rejected mail" enabled (inbound), or
  the message came from a relay network and "Quarantine rejected outbound
  mail" is enabled (outbound), or a [data loss prevention](outbound.md#data-loss-prevention)
  rule with "Reject and quarantine" rejected it;
- it was not rejected by an explicit block list entry (global, domain or
  personal block lists, the IP block list, bad ASNs or subjects): such mail is
  not worth keeping;
- it is not larger than the "Maximum size of quarantined mail" setting
  (default 10 MiB).

Quarantined mail is deleted after the retention time (Settings, default 14
days) by the daily cleanup job.

## For administrators

**Quarantine** lists all messages, newest first, with filters for domain,
direction (inbound/outbound), status (released or not) and a search over
subject, sender, recipient and queue ID. The list refreshes itself every 30
seconds while nothing is selected. [Domain administrators](administrators.md)
see only their domains' messages.

A message's page shows its headers, the text, its attachments and the Rspamd
symbols that led to the rejection. Messages with an HTML body can switch to
the **HTML**, shown sanitized in a sandboxed frame without scripts, links,
forms or remote content (see [HTML view](mail-log.md#html-view)). The
[mail log](mail-log.md) has the same message with its full score breakdown.
Actions:

| Action | Effect |
| --- | --- |
| Release | Delivers the message to its recipient, with an `X-Quarantine-Released` header |
| Release & learn as ham | Also trains Bayes that it is not spam |
| Learn as spam & delete | Trains Bayes and stores the fuzzy hash, then deletes it |
| Delete | Removes it |
| Download | The original message (`.eml`) |

**The quarantine of one mailbox**: the quarantine count of each mailbox user
(**Users → Mailbox users**), and **Quarantine** on a mailbox user's page, open
the list for that address only: its inbound mail, including mail to its
`+tag` sub-addresses (`/quarantine/?user=alice@example.com`, any address works,
with or without an account). The mailbox user's page also shows its latest
quarantined messages. Everything else works as on the full list. API:
`GET /api/v1/quarantine?mailbox=alice@example.com`.

Several messages can be released, learned or deleted at once from the list.
Messages with a virus must be released one by one, with an explicit
confirmation.

Released mail is re-injected into Postfix on a local listener that skips the
filter. Only a header is added, so the original DKIM signature stays valid;
an `X-Quarantine-Released` header that arrives with incoming mail is removed so
it cannot be forged. Released outbound mail is DKIM-signed for the sender's
domain.

## Mailbox users

Mailbox users are accounts for recipients who manage their own quarantine.
Create them under **Users → Mailbox users** (or with the API); the address's domain
must be configured. They log in with their e-mail address and see:

- **My quarantine**: their inbound quarantined mail, including mail to their
  `+tag` sub-addresses. They can view, release and delete it, but not release
  viruses (administrators can) and not train the filter.
- **My settings**: their own thresholds (1 to 50), digest interval and
  personal allow/block list of senders.

They never see outbound mail, other addresses or any gateway settings.
Deleting a mailbox user keeps their quarantined mail for administrators and
invalidates their digest links.

**Send password link** on a mailbox user's page mails them a link to choose a
new password themselves (also `POST /api/v1/mailbox-users/{id}/password-link`).
The link works once, for "Password links valid for" days (Settings → General,
default 7); a newer link replaces the older one. Like the digest links, its
token is kept out of the web server's log and never sent to other sites.

### Accounts for recipients of quarantined mail

With **Create accounts for recipients of quarantined mail** (Settings →
General, Quarantine and digests; off by default), a recipient of inbound
quarantined mail who has no mailbox user gets one, when their mailbox exists
on the domain's destination server, and a mail with a link to set their
password. How the gateway knows the mailbox exists depends on the domain's
[recipient check](domains.md):

| Recipient check | The mailbox exists when |
| --- | --- |
| Recipient list, directory | The address is an active recipient |
| Verify with the destination server | Always: Postfix verified it before it accepted the mail |
| Accept all | The destination server accepts `RCPT TO` for it (`MAIL FROM:<>`, nothing is sent). A server that also accepts a made-up address of the domain cannot tell, and no account is created |

Every 5 minutes (`spamfilter-job-accounts.timer`, `flask provision-accounts`,
one node of a cluster) the mail quarantined in the last day is looked at, at
most 100 recipients a run; an address the destination server refused or could
not answer for is asked again after a day. Sub-addresses count as their
mailbox (`alice+news@` → `alice@`). The account has a random password nobody
knows until the recipient sets one; it is marked **invited** in the list until
then. If the mail cannot be handed to Postfix, no account is created and the
next run tries again. Created accounts are recorded in the audit log
(`mailbox_user.auto_create`). Settings: `mailbox_auto_create`,
`mailbox_invite_days`.

## Digests

Recipients with new quarantined mail receive a summary at the digest
interval: never, hourly, daily or weekly. The interval is taken from the
mailbox user's setting, else the domain's, else the global one (Settings,
default daily). A recipient gets a digest only when there is new mail since
the last one.

Each digest contains a **link** with a personal token that opens the
recipient's quarantine **without an account**, for "Digest links valid for"
days (default 7). Opening the link changes nothing: mail scanners that follow
links cannot release or delete anything; that needs a click on the page.
The token is never sent to other sites, never cached and is removed from the
web server and application logs.

The digest sender address is the "Digest sender" setting, by default
`quarantine@<hostname>`. Digests are sent every 15 minutes by
`spamfilter-job-digests.timer`; **Send due digests** (Settings → General, Quarantine and digests) sends the due
ones immediately. Links of disabled mailbox users stop working. Set `WEB_PUBLIC_URL` if the links must use another name than the
gateway's hostname (for example in a cluster).

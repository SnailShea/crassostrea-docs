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
subject, sender, recipient and queue ID. [Domain administrators](administrators.md)
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

**The sender** of an inbound message can be put on an allow or block list in
the same step, under **Sender** on the message's page:

| Action | Effect |
| --- | --- |
| Release & allow sender | Delivers the message and adds its envelope sender to an allow list |
| Block sender & delete | Adds the envelope sender to a block list and deletes the message |

The list is chosen next to the buttons: **this recipient** (the personal list
of the recipient's mailbox user, offered when there is one), **the recipient's
domain**, or **every domain** (the global list; not for domain
administrators). A sender that is listed already is no error. The entry is
noted as coming from the quarantined message, and the filter uses it within a
minute. Bounces (empty sender) and outbound mail cannot be listed. If the
release fails, no entry is added. API: `allow_sender` (`mailbox`, `domain` or
`global`) in `POST /api/v1/quarantine/{id}/release`, and
`POST /api/v1/quarantine/{id}/block-sender` (`scope`, `delete`).

**Why it was held back** lists the rules that added score, in plain language
(also on the mailbox users' pages, see [Checking a
decision](filtering.md#checking-a-decision)).

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
  viruses (administrators can) and not train the filter. **Release & allow
  sender** and **Block sender & delete** put the sender on their personal
  lists in the same step; a message's page explains in plain words why it was
  held back.
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

**Forgot your password?** on the login page lets mailbox users ask for such a
link themselves: they enter their address and get the mail if it has an
active account. The page answers the same for any address, so it does not tell
who has an account; at most 3 links an hour are sent to an address and 10 are
requested from one client address. The current password keeps working until a
new one is set. Administrators reset theirs on the server
(`helper-scripts/reset-admin.sh`) or have another administrator do it.

Recipients who reach their quarantine through a digest link can use **Release
& allow sender** and **Block sender & delete** too, when their address has a
mailbox user.

### Importing accounts from another mail system

`POST /api/v1/mailbox-users` takes either `password` (clear text, hashed by
the gateway) or `password_hash`: the hash the previous system stored, so
users keep their passwords. Accepted formats, with or without the
Dovecot/LDAP scheme prefix:

| Format | Examples |
| --- | --- |
| crypt(3) | SHA512-CRYPT `$6$`, SHA256-CRYPT `$5$`, MD5-CRYPT `$1$`, bcrypt `$2y$`/`$2b$` (`{BLF-CRYPT}`), yescrypt `$y$`, scrypt `$7$`; prefixes `{CRYPT}`, `{SHA512-CRYPT}`, `{MD5-CRYPT}`, ... |
| Argon2 | `$argon2id$...`, `{ARGON2ID}`, Django's `argon2$...` |
| Salted digests | `{SSHA512}`, `{SSHA256}`, `{SSHA}`, `{SMD5}` |
| Plain digests | `{SHA512}`, `{SHA256}`, `{SHA}`, `{PLAIN-MD5}`, `{LDAP-MD5}` |
| PBKDF2 | Django's `pbkdf2_sha256$...`, Dovecot's `{PBKDF2}`, Werkzeug's `pbkdf2:sha256:...` |
| Clear text | `{PLAIN}`, `{CLEARTEXT}`: hashed at once, never stored |

Digests may be Base64 or hex (`{SSHA256.HEX}`, or recognised by length). That
covers Dovecot, Postfix Admin, mailcow, iRedMail, Modoboa and OpenLDAP, among
others. Exports without a hash (Microsoft Exchange and 365, Google) cannot
keep passwords: create those accounts without one and send a
[password link](#mailbox-users), or turn on [accounts for
recipients](#accounts-for-recipients-of-quarantined-mail).

An unknown format, or one whose cost would make a login slow (bcrypt above 16,
more than a million SHA-crypt rounds or PBKDF2 iterations, Argon2 above
256 MiB), is refused with the reason. The first successful login replaces an
imported hash with the gateway's own (scrypt); until then `password_format` of
the account (and its page) shows the imported format. `PATCH` takes
`password_hash` too.

```bash
curl -H "$KEY" -H "Content-Type: application/json" \
     -d '{"email": "alice@example.com", "password_hash": "{SHA512-CRYPT}$6$5Yd1...$Fq..."}' \
     $API/mailbox-users
```

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

Every 5 minutes (`crassostrea-job-accounts.timer`, `flask provision-accounts`,
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
`crassostrea-job-digests.timer`; **Send due digests** (Settings → General, Quarantine and digests) sends the due
ones immediately. Links of disabled mailbox users stop working. Set `WEB_PUBLIC_URL` if the links must use another name than the
gateway's hostname (for example in a cluster).

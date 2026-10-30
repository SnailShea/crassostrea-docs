# Filtering

## Thresholds

Three thresholds decide what happens to a message, see the
[Overview](overview.md#score-and-actions). They must satisfy
*greylist ≤ mark as spam < reject*.

They exist at three levels; the most specific one that is set wins:

1. **Mailbox user** (on the user's page, or by the user under "My settings")
2. **Domain** (on the domain page)
3. **Global** (Settings)

Mailbox users can only choose values from 1 to 50 for themselves;
administrators can set any value from -100 to 10000.

Rspamd decides once per message. A user's thresholds therefore only apply to
messages whose envelope recipients **all** belong to that user (the address
and its `+tag` sub-addresses); a domain's thresholds only apply when every
recipient is in that domain. A message to several users or domains is filtered
with the next more general settings. One recipient's settings never decide for
another recipient.

### Turning greylisting off

**Greylisting** in **Settings → General → Spam thresholds** turns
greylisting off for the whole gateway, including domains and mailbox users with
their own greylist threshold: no mail is deferred for its score any more, and
a score above the greylist threshold has no action until it reaches mark as
spam. Rspamd follows within a minute; it then neither checks nor stores
senders in Valkey (`GREYLIST_CHECK`, `GREYLIST_SAVE`). The greylist thresholds
stay stored and apply again when greylisting is turned back on. The setting is
`greylisting` (`PATCH /api/v1/settings`), written to
`/etc/rspamd/custom/scanning.map` for `rspamd.local.lua`. Rate limits and the
PTR check can still defer mail.

## Allow and block lists

| Level | Managed under | Applies to |
| --- | --- | --- |
| Global | Lists & training → Allow list, Block list | Every message |
| Domain | the domain's page | Block: messages with at least one recipient in the domain. Allow: messages whose recipients are all in the domain |
| Personal | the mailbox user's page or "My settings" | Messages whose recipients all belong to that user |

Entry types:

- **Sender**: matches the envelope sender and the `From` header address.
  Accepted forms: `user@example.org`, `*@example.org`, `@example.org` or
  `example.org` (the whole domain), and `*` wildcards such as
  `*@*.example.org`. A pattern that would match every domain is refused.
- **IP** (global and domain lists only): an address or network in CIDR
  notation.

Precedence, from strongest to weakest:

1. Global block and allow entries (they add ±2050 to the score)
2. Domain block entries
3. Personal block entries
4. Personal allow entries
5. Domain allow entries
6. Personal, then domain, then global thresholds

!!! note "Allow entries are not a blank cheque"
    Sender allow entries (at every level) are ignored when the sender fails
    its own DMARC or SPF policy, so a forged sender address cannot use them.
    IP allow entries are not affected: an address cannot be forged in an SMTP
    connection. Viruses are always rejected.

## What Rspamd checks

Rspamd combines many modules, configured in the repository's
`data/conf/rspamd/`. Among them:

- **Bayes** classifier and **fuzzy hashes** (local storage plus a public
  fuzzy storage), trained by you (see below) and automatically from very clear
  cases
- DNS blocklists and URL blocklists (Spamhaus with a DQS key where needed:
  Settings → General, Scanning; without one the public mirrors, which refuse
  cloud providers' networks)
- SPF, DKIM, DMARC and ARC
- header and MIME heuristics, bad attachments and file types, phishing URLs
- **ClamAV** (viruses, plus Sanesecurity signatures) and **Olefy** (Office
  macros), unless disabled
- SpamAssassin rule sets, updated daily
- greylisting and rate limits

The gateway adds its own checks, configured in the web interface: content
rules, an attachment policy, hash lookups of attachments, impersonation and
link protection ([Threat protection](protection.md)), and for outbound mail
limits per sender and data loss prevention ([Outbound mail](outbound.md)).
Message and attachment size limits are in
[Mail log and statistics](mail-log.md#message-limits).

**postscreen** runs before all of this: it checks connecting hosts against
weighted DNS blocklists and for protocol violations, and turns away most spam
bots cheaply. Adjust its lists in `/etc/postfix/dns_blocklists.cf`.

## Training

The Bayes classifier and the local fuzzy storage learn from your decisions:

- **Training** page: upload or paste a raw message (with all headers) and learn
  it as spam or ham.
- Quarantine: **Learn as spam & delete**, or **Release & learn as ham**.
- API: `POST /api/v1/training/spam` and `/training/ham` with the raw message.
- Users report from their mail client, see below.

Learning as spam also stores the message's fuzzy hash, so copies of the same
campaign are recognised. Bayes starts scoring after a few learned messages of
each kind; the more you train (in both directions), the better it gets.
`helper-scripts/reset-learns.sh` deletes all learned data.

### Reports from users

Users can report spam (and wrongly caught mail) from their mail client: they
forward the message **as an attachment** to `spam@<hostname>` or
`ham@<hostname>`, and every attached message (`message/rfc822` or `.eml`) is
learned. In Microsoft 365, set these addresses as the reporting mailbox of the
Report button (Microsoft Defender: User reported settings); Outlook's "Forward
as attachment" works everywhere.

Turn it on under **Training → Reports from users** (off by default):

- Only users of your domains can report: the sender's domain must be a
  protected domain, and the report must come from a relay network, pass SPF
  for that domain, or carry a DKIM signature of it. Others are refused, so
  outsiders cannot teach the filter.
- At most 50 messages per reporter and hour (configurable).
- A short confirmation goes back to the reporter (can be turned off); refused
  reporters get none, as their address may be forged.
- Inline forwards carry the reporter's headers instead of the original's and
  are not learned; the confirmation says how to forward as an attachment.

The page lists the recent reports with their result. API:
`GET /api/v1/user-reports`, settings `user_reports_*`.

## Checking a decision

The **Mail log** shows every scanned message with its score breakdown: every
rule that matched, with its score and details ([Mail log and
statistics](mail-log.md)). Common rules are explained in plain words below
their name (SPF and DMARC failures, block lists, the statistical filter,
missing reverse DNS, the gateway's own checks, ...). Quarantined messages show
the same, and **Why it was held back** lists the explained rules that added
score, highest first; mailbox users see that list on their messages too.
The Rspamd UI at `/rspamd/` (password `RSPAMD_PASSWORD`) has Rspamd's own
history.

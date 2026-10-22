# Outbound mail

Your own mail servers can send outbound mail through the gateway. Setting it
up (relay networks, smart host, SPF, DKIM) is described in
[Domains and delivery](domains.md#outbound-mail). This page covers the
controls for outbound mail.

Outbound mail is scanned like inbound mail. Rejected outbound mail is
quarantined for administrators (Settings: "Quarantine rejected outbound mail",
on by default); mailbox users never see it.

## Outbound limits

Inbound mail has its own [rate limits](protection.md#rate-limits).

**Outbound limits** keeps a compromised account or device from sending a flood
of spam through the gateway, which would get its address on block lists.

- **Every sender**: the default number of recipients each envelope sender may
  send to per minute, hour or day. 0 means no limit (the default).
- **Limits for a sender or a domain**: a limit for one address, or for each
  sender of a domain on its own. An address's limit wins over its domain's,
  which wins over the default; 0 exempts them.

A message counts once per recipient. Bounces (empty sender) count per sending
server. Over the limit, mail is **deferred** (`451`) and the sending server
retries later; a single message with more recipients than the limit is
rejected. The counts are kept in Valkey, so a cluster counts together.

The page lists the senders that were deferred in the last 7 days and the
busiest senders of the current period; **Reset** lets a sender send again at
once. API: `/api/v1/outbound-limits`, `/api/v1/outbound-senders`, the default
in the settings (`outbound_limit`, `outbound_limit_period`).

## Data loss prevention

**Data loss prevention** rules stop outbound mail that holds data which must
not leave the organisation. A rule finds:

| Detector | Finds |
| --- | --- |
| Payment card numbers | 13 to 19 digits with a valid check digit (Luhn) and a known first digit |
| IBANs | the country's length and a valid checksum |
| US social security numbers | `123-45-6789`, without impossible ranges |
| Keywords | words or phrases, anywhere, ignoring case |
| Regular expression | your PCRE expression |

in the subject, the text and text attachments (text files, CSV, JSON, XML; not
Office documents or PDFs). It needs a minimum number of **different** matches
(a card number quoted twice counts once), can be limited to a sender or
domain, and then:

| Action | Effect |
| --- | --- |
| Reject and quarantine | Rejected; the sender is told the rule's name; the message is kept in the outbound quarantine (even if outbound quarantine is off) for an administrator to release |
| Reject | Rejected only |
| Deliver with X-DLP-Match | Delivered with a header naming the rules |

The page tests the rules on a pasted text. The symbols are `OUTBOUND_DLP` and
`OUTBOUND_DLP_QUARANTINE`. API: `/api/v1/dlp-rules`.

## Other controls

- [Attachment policy](protection.md#attachment-policy) and
  [content rules](protection.md#content-rules) can apply to outbound mail too.
- [Journaling](journaling.md) sends copies of outbound mail to an archive.
- [DMARC reports](reports.md) show which servers send mail in your domains'
  names, and whether they pass DKIM and SPF.

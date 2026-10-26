# Threat protection

Besides Rspamd's own scoring ([Filtering](filtering.md)), the gateway has
checks you configure in the web interface. All of them run inside Rspamd;
their settings are written as maps to `/etc/rspamd/custom/` and take effect
within a minute.

| Check | Page | Default |
| --- | --- | --- |
| [Attachment policy](#attachment-policy) | Attachments | On, with a list of dangerous file types |
| [Hash lookups](#hash-lookups) | Attachments | Off |
| [Impersonation](#impersonation) | Impersonation | Lookalike and spoofing checks on; name check needs protected people |
| [Link protection](#link-protection) | Link protection, domain settings | Off (per domain) |
| [Content rules](#content-rules) | Content rules | None |
| [Rate limits](#rate-limits) | Rate limits | 50 simultaneous connections per host; the others off |
| [Virus scanning](#virus-scanning) | Settings → General | On (unless installed with `SKIP_CLAMD=y`) |
| [Hosts without reverse DNS](#hosts-without-reverse-dns) | Settings → General | Accepted |

Rules that reject a message work like any other rejection: the sender is told
during the SMTP dialogue, and inbound mail of domains with a quarantine is
quarantined, where it can be released. Each check names itself in the mail
log's score breakdown.

## Attachment policy

**Attachments** lists file extensions that are always rejected, whatever the
spam score: by default executables, scripts, shortcuts, disk images and
installers (`exe`, `js`, `lnk`, `iso`, `msi`, `scr`, `vbs` and more; the page
shows the full list).

- The **last extension** counts, so `invoice.pdf.exe` is an `exe`; trailing
  dots and spaces are ignored, as Windows ignores them.
- **Files inside archives** (zip, rar, 7z, ...) are checked too, one level
  deep (on by default).
- **Password-protected archives** can be rejected: the antivirus cannot look
  into them, and they are a common way to slip malware past it (off by
  default).
- **Outbound mail** from the relay networks is only checked if you turn it on.
- **Exempt senders**: envelope sender addresses or domains the policy does not
  apply to, e.g. a software vendor.

The symbol is `ATTACHMENT_BLOCKED`, with the file name. Independently of the
policy, Rspamd scores risky types it knows, ClamAV scans the files and Olefy
checks Office macros.

## Hash lookups

On the same page, the hash of every attachment can be looked up in databases
of known malware. Each source adds its score when it knows the file (at the
reject threshold the message is rejected and quarantined):

| Source | Needs | Known malware when |
| --- | --- | --- |
| Team Cymru Malware Hash Registry | nothing (DNS) | at least a share of antivirus engines detect it (default 10 %) |
| MalwareBazaar (abuse.ch) | a free Auth-Key | it is a sample in the database |
| VirusTotal | an API key | at least a number of engines detect it (default 3) |

Only hashes leave the gateway, never files. A file nobody has reported yet
passes, so this adds to ClamAV rather than replacing it. Verdicts are cached
for a day. The free VirusTotal key allows 4 requests a minute and 500 a day:
set the per-minute limit to your key's, attachments beyond it are not looked
up. The API keys are stored in the database and copied to Valkey, not into a
map file, and are never shown again (leave the field empty to keep a key).

The page lists recent detections (`GET /api/v1/hash-detections`) and looks up
a hash you enter. Sandboxes that open unknown files need a commercial service
or appliance and are not included.

## Impersonation

**Impersonation** adds score to inbound mail that pretends to come from your
organisation. Three checks, each with its own score (0 turns it off; a score
at or above the reject threshold rejects and quarantines):

| Check | Symbol | Default score | Flags |
| --- | --- | --- | --- |
| Name of a protected person | `IMPERSONATION_NAME` | 6 | The `From` display name is one of the protected people, but the address is outside your domains and none of theirs |
| Look-alike domain | `LOOKALIKE_DOMAIN` | 6 | A `From` domain imitating one of yours, see below |
| Own domain without authentication | `SPOOFED_OWN_DOMAIN` | 8 | Your domain in `From`, but neither DMARC, DKIM nor SPF pass for it |

- **Protected people**: managers, finance and HR staff whose names fraudsters
  use, with the private addresses they really write from. Names match as whole
  words, ignoring case and punctuation, also written "Doe, Jane".
- **Look-alikes** of a protected domain (`example.com`): look-alike characters
  (`examp1e.com`, `exarnple.com`, Cyrillic letters), one or two typing errors
  (`exampel.com`), another suffix (`example.net`), the name joined with others
  (`example-secure.com`), or the domain in front of another
  (`example.com.evil.net`). Very short names are only compared by look-alike
  characters.
- **Protected domains** are the gateway's domains plus "More domains to
  protect" (e.g. a parent company). **Legitimate look-alikes** (your other
  domains) are exempt.
- **Spoofing**: domains that publish DMARC `p=reject` or `p=quarantine` are
  already handled by Rspamd's DMARC check; this also covers `p=none` and
  domains without DMARC.

Mail from the relay networks is not checked. The page's test shows what a
display name and address would trigger. API: `/api/v1/protected-people`,
`POST /api/v1/impersonation/test`, scores in the settings.

## Link protection

With link protection a domain's inbound mail gets its links rewritten to
`https://<hostname>/l/<id>`. When a recipient clicks, the gateway checks the
link **at that moment** and redirects to it, or shows a warning page. A site
that turns malicious after the message arrived is still blocked.

- Turn it on per domain (domain settings: "Check links when they are
  clicked"). A message is rewritten only when all its recipients are in such
  domains.
- A click is checked against your **blocked sites** and the domain block lists
  Spamhaus DBL (with `SPAMHAUS_DQS_KEY` through DQS), SURBL and URIBL. Verdicts
  are cached for 10 minutes.
- The warning page can offer a way on ("open it anyway"), off by default.
- Not rewritten: signed or encrypted mail (S/MIME, PGP), outbound mail,
  rejected mail, messages larger than the message size limit, links to exempt
  hosts and to the gateway itself. HTML links and plain-text URLs are
  rewritten; images are not.
- The original URLs are kept in Valkey for 90 days (configurable); older links
  show an error page. Only links the gateway rewrote can be opened, so it
  cannot be abused as an open redirector.
- The mail log keeps each message as it arrived, before rewriting.

The page lists recent clicks (all or blocked) and checks a link you enter. API:
`GET /api/v1/link-clicks`, `POST /api/v1/link-check`.

!!! note
    The link endpoint `/l/` must be reachable by your recipients wherever they
    read mail, like the web interface. Rewriting changes the message body, so
    the sender's DKIM signature no longer verifies downstream; the gateway
    has checked it already.

## Content rules

**Content rules** are your own rules for what a message says or where it comes
from. A rule looks at one part of the message:

| Part | Matched against |
| --- | --- |
| Subject | the decoded subject |
| From | display name and address, as `Name <address>` |
| Envelope sender | the SMTP sender |
| A recipient | each envelope recipient |
| A header | each value of the named header |
| The text | the text parts |
| A link | each URL |
| An attachment name | each file name |

with **contains** or **is** (both ignoring the case of letters A to Z) or a
**regular expression** (PCRE; `(?i)` ignores case). It applies to inbound,
outbound or all mail, and then:

| Action | Effect | Symbol |
| --- | --- | --- |
| Add a score | Adds its score to the spam score; negative favours such mail | `CONTENT_RULE` |
| Reject | Rejects the message with your message (or a default); wins over the other rules | `CONTENT_RULE_REJECT` |
| Deliver with X-Content-Rule | Adds a header naming the rule, e.g. for a rule in the mail server | `CONTENT_RULE_TAG` |

Rules are checked before Rspamd's other checks, so a reject rule always
applies. Test them on a pasted message (headers, a blank line, the body) with
an envelope sender and recipients. API: `/api/v1/content-rules`.

## Rate limits

**Protection → Rate limits** limits how much inbound mail a sending server may
deliver and a recipient may receive, so a flood or a misbehaving server cannot
swamp the gateway or a mailbox. Mail over a limit is **deferred** with a
temporary error (4xx): the sending server retries later and nothing is lost.
Two layers, 0 meaning no limit:

| Layer | Limits | Counted |
| --- | --- | --- |
| SMTP connections (Postfix) | Simultaneous connections (Postfix's default 50), connections, messages and recipients per minute, per sending host | By Postfix's `anvil`, during the SMTP dialogue, on each node of a cluster separately |
| Messages (Rspamd) | Messages from one IP address, messages to one recipient, bounces (empty sender) to one recipient, each per minute, hour or day | In Valkey as messages are scanned: for all nodes of a cluster together |

Bounce limits catch *backscatter*: bounces of mail someone else sent with
forged addresses of your domains.

**Exempt networks** (one per line, e.g. partner servers that send a lot) have
no rate limits in either layer. The relay networks never have any: their
senders have [outbound limits](outbound.md#outbound-limits) instead.

A deferred message counts nowhere. **Deferred in the last 7 days** lists the
servers and recipients that reached a limit, with **Reset**, which clears their
counts so their mail is accepted again at once. The rejection names the limit,
e.g. `451 4.7.1 Rate limit of 100 messages from this server per hour reached,
try again later`; Postfix's own limits log `too many connections` or
`Message delivery request rate limit exceeded` in the Postfix log.

The limits are settings (`smtp_*_limit`, `inbound_*_limit`, `inbound_*_period`,
`rate_limit_exempt` in `PATCH /api/v1/settings`); `GET /api/v1/rate-limits/hits`
and `POST /api/v1/rate-limits/reset` list and reset the deferred. Postfix applies
its limits within seconds (`crassostrea-postfix-reload.path`, `postfix-apply`,
kept by `configure.sh`), Rspamd within a minute.

## Virus scanning

ClamAV scans every message (with the Sanesecurity signatures); a virus
rejects the message (`VIRUS_FOUND`), and inbound mail of domains with a
quarantine is quarantined, where only administrators can release it. **Scan
mail for viruses** in **Settings → General → Scanning** turns the scan off and
on again without reinstalling: Rspamd skips the ClamAV check (`CLAM_VIRUS`)
within a minute, clamd keeps running and keeps its signatures up to date. The
attachment policy, the hash lookups and Olefy still apply. The setting is
`virus_scanning` (`PATCH /api/v1/settings`), written to
`/etc/rspamd/custom/scanning.map` for `rspamd.local.lua`.

A gateway installed with `SKIP_CLAMD=y` has no ClamAV at all; the page says so,
and the setting changes nothing there.

## Hosts without reverse DNS

Almost every mail server has a reverse DNS name (a PTR record) for its address;
infected PCs and dial-up ranges often do not. **Hosts without a reverse DNS
name (PTR)** in **Settings → General → Scanning** decides what Postfix does
with them, before the message is transferred:

| Setting | Postfix answers |
| --- | --- |
| Accept (default) | Nothing special: Rspamd adds a little score (`RDNS_NONE`) |
| Defer | `450`: a real server retries, and passes once its PTR exists |
| Reject | `550`: the mail is refused |

A temporary DNS failure always defers, whatever the setting, so a broken
resolver does not reject mail. The relay networks and the IP addresses and
networks of the global [allow list](filtering.md#allow-and-block-lists) are
exempt: add a partner's server there when it has no PTR. The rejection is
logged in the Postfix log as `cannot find your reverse hostname`.

Postfix's `reject_unknown_reverse_client_hostname` does the check
(`sf_ptr_check` at the end of `smtpd_client_restrictions`, with
`unknown_client_reject_code`), applied within seconds by
`crassostrea-postfix-reload.path` and kept by `configure.sh`. The setting is
`ptr_check` (`off`, `defer` or `reject`).


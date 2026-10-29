# Overview

## What Crassostrea does

Crassostrea is the host your domains' MX records point to. For every message
it receives it:

1. accepts the connection in **postscreen**, which turns away most spam bots
   before they reach Postfix (DNS blocklists, protocol checks);
2. checks in **Postfix** that the recipient's domain is protected and, if the
   domain asks for it, that the recipient exists;
3. scans the message with **Rspamd**, which asks **ClamAV** (viruses) and
   **Olefy** (Office macros), applies your thresholds and allow/block lists,
   and runs the gateway's own checks: content rules, the attachment policy,
   hash lookups, impersonation and, for outbound mail, limits and data loss
   prevention ([Threat protection](protection.md), [Outbound mail](outbound.md));
4. depending on the score, **delivers** it to the domain's mail server
   (unchanged, marked as spam, or with its links rewritten for
   [link protection](protection.md#link-protection)), **greylists** it, or
   **rejects** it;
5. records it in the **mail log** and keeps a copy of rejected mail in the
   **quarantine**, from where administrators and recipients can release it.

Your mail server keeps the mailboxes. Crassostrea stores mail only in the
quarantine, the [mail log](mail-log.md) (7 days by default, at most 31) and
its queue; for long-term copies it [journals](journaling.md) to an archive.

## Score and actions

Rspamd adds up a score for every message. Three thresholds decide what
happens:

| Score reaches | Action |
| --- | --- |
| Greylist (default 7) | The sender is asked to try again later; real mail servers do, many spam bots do not |
| Mark as spam (default 8) | Delivered with spam headers (`X-Spam: Yes`), so the mail server can file it as junk |
| Reject (default 15) | Refused during the SMTP dialogue and stored in the quarantine |

The thresholds can be set globally, per domain and per mailbox user; the most
specific value wins. See [Filtering](filtering.md).

## Components

| Component | Purpose |
| --- | --- |
| Postfix with postscreen | Receives and sends mail (port 25) |
| Rspamd | Spam scoring, DKIM signing, quarantine export |
| ClamAV, Olefy | Virus and macro scanning (optional) |
| MariaDB | Domains, recipients, lists, accounts, quarantine, mail log, statistics, reports, audit log, settings |
| Valkey | Rspamd's learning data (Bayes, fuzzy), greylisting, rate limits, outbound counts, rewritten links, bans, the Postfix log for the UI, queues of the SIEM and journal senders |
| unbound | Local validating DNS resolver (DNS blocklists refuse public resolvers) |
| nginx | HTTPS for the web interface and the API |
| Web interface (`crassostrea-web`) | Administration, mailbox user self-service, REST API, link protection, metrics |
| Root services (`crassostrea-hostname`, `-tls`, `-network`, `-backup`, ...) | Changes the web interface may not make itself: hostname, certificates, IP addressing, backups |
| SIEM and journal senders (`crassostrea-siem`, `crassostrea-journal`) | Events to a SIEM, copies to an archive |
| netfilter (`crassostrea-netfilter`) | Bans hosts after repeated protocol errors or failed logins, and scanners and exploit attempts at once |
| Watchdog (`crassostrea-watchdog`) | Health checks, restarts, notifications |
| acme.sh (`crassostrea-acme`) | Let's Encrypt certificate |

All of them run as systemd services on one host. In a cluster each node runs
the full set; see [High availability](high-availability.md).

## Inbound and outbound mail

**Inbound** mail is mail for your protected domains, from anywhere.

**Outbound** mail is mail your own servers send through the gateway. Only
**relay networks** you configure may do this. Outbound mail is scanned too,
limited per sender, checked by data loss prevention rules, signed with the
domain's DKIM key, and rejected outbound mail is quarantined for
administrators. See [Domains and delivery](domains.md#outbound-mail) and
[Outbound mail](outbound.md).

## Who uses what

- **Administrators** use the web interface (and API keys) and see everything.
- **Domain administrators** see and manage only the mail of their domains
  ([Administrators](administrators.md)).
- **Mailbox users** log in with their e-mail address and see only their own
  inbound quarantine and personal filter settings. They can report spam from
  their mail client.
- **Recipients without an account** receive quarantine digests with a link
  that opens their quarantine for a limited time.

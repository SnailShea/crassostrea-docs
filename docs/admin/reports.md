# DMARC and TLS reports

Mailbox providers send **aggregate reports** to the address a domain publishes:

- **DMARC reports** (RFC 7489): which servers sent mail with your domain in
  `From`, how much, and whether it passed DKIM and SPF. They show forgeries of
  your domains, and your own services that still need DKIM or SPF.
- **TLS reports** (RFC 8460, TLS-RPT): whether other providers could connect
  to your MX (this gateway) with TLS, and why not (e.g. an expired certificate,
  or an MTA-STS policy that does not match).

The gateway receives them at `reports@<hostname>` and shows them on **DMARC &
TLS reports**.

## Receiving reports

Publish the report address in each domain's records. The page shows, per
domain, what to publish and checks what is published:

| Record | Value |
| --- | --- |
| `_dmarc.<domain>` TXT | `v=DMARC1; p=none; rua=mailto:reports@<hostname>`; add the `rua` to an existing record, keeping its policy |
| `<domain>._report._dmarc.<hostname>` TXT | `v=DMARC1`, in the zone of the gateway's hostname: receivers send DMARC reports to an address outside the domain only when that domain allows it |
| `_smtp._tls.<domain>` TXT | `v=TLSRPTv1; rua=mailto:reports@<hostname>` |

Reports arrive about once a day per provider. Postfix accepts mail for
`reports@<hostname>` (of the hostname's addresses only this one and the
[reporting addresses](filtering.md#reports-from-users)), Rspamd hands it
unfiltered to the web interface, which unpacks the attachments (zip, gzip or
plain; size-limited) and stores the reports about domains of this gateway
once each. Anyone can send to the address, so reports are only shown, never
acted on, and XML with a `DOCTYPE` is refused.

In a cluster each node accepts reports at its own hostname; list several
addresses in `rua` (comma separated) so reports arrive while a node is down.

## Reading them

The page has a table of all domains for the last 7, 30 or 90 days: DMARC
reports, messages, the share that **passed DMARC**, TLS reports and failed TLS
sessions. A domain's details show:

- **Failing DMARC**: sources (IP address and `From` domain) whose mail passed
  neither aligned DKIM nor SPF, by number of messages. Unknown sources are
  forgeries; known ones (a newsletter service, a ticket system) need DKIM or
  SPF for your domain before you tighten the policy to `p=quarantine` or
  `p=reject`.
- **Passing DMARC**: the sources that do it right.
- **TLS failures**: the result type, the receiving MX and the number of
  sessions.
- The DNS records above, and the reports themselves.

Reports are kept 180 days by default (Settings on the page). API:
`/api/v1/reports/summary`, `/reports/dmarc`, `/reports/tls`.

## Sending DMARC reports

As a receiver, the gateway can send DMARC aggregate reports about the mail it
received to the domains that ask for them (their `rua`). It is good practice
and helps others find forgeries of their domains. Turn it on in the page's
settings. Rspamd collects the data all the time (and forgets it after two
days); `crassostrea-dmarc-report.timer` sends yesterday's reports daily at
about 01:15 from `dmarc-noreply@<hostname>`, from one node of a cluster.

Sending **TLS reports** about the gateway's own deliveries needs Postfix 3.10
or later, which EL10 does not ship yet.

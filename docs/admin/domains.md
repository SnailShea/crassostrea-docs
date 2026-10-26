# Domains and delivery

## Protecting a domain

1. **Domains → Add domain** in the web interface (or `POST /api/v1/domains`).
2. Make sure the domain's mail server accepts mail from the gateway, and
   ideally **only** from the gateway: otherwise spammers deliver directly to it
   and bypass the filter.
3. Point the domain's **MX record** to the gateway's hostname. In a cluster,
   add one MX record per node.
4. If the mail server sends outbound mail through the gateway, add its
   address as a **relay network** (see below) and include the gateway in the
   domain's SPF record.

Mail for a domain is accepted as soon as the domain exists and is active.
**DNS & ports** (Network) then checks the domain's MX, DKIM, SPF and DMARC
records from the gateway ([DNS and ports](network.md#dns-and-ports)).

## Domain settings

| Setting | Meaning |
| --- | --- |
| Domain | The domain name. It cannot be changed later (delete and re-add instead) |
| Description | Free text |
| Active | Inactive domains are rejected like unknown domains |
| Downstream server, port | Where clean mail is delivered: a hostname or IP address, port 25 by default |
| Look up MX records | Deliver to the MX hosts of the downstream server name instead of the name itself (not possible with an IP address) |
| Recipient validation | Which recipients are accepted, see below |
| Directory | With validation from a directory: which one |
| Quarantine rejected mail | Keep rejected mail of this domain in the quarantine |
| Check links when they are clicked | [Link protection](protection.md#link-protection) for the domain's inbound mail |
| Quarantine digest | Digest interval for the domain's recipients; empty uses the global setting |
| Thresholds | Greylist, mark as spam and reject scores for this domain; empty values use the global settings |

## Recipient validation

Spam is often sent to random addresses. Rejecting unknown recipients at the
gateway avoids accepting it at all (and avoids backscatter: bounces to forged
senders).

| Mode | Behaviour |
| --- | --- |
| Accept all recipients | Every address of the domain is accepted; the downstream server decides |
| Only recipients from the list | Only addresses on the domain's recipient list are accepted. Manage the list on the domain page or with the API (addresses or local parts, one per line) |
| Verify with the downstream server | Postfix asks the downstream server whether the address exists (SMTP call-ahead) before accepting a message, and caches the answer |
| Only recipients from a directory | Like the list, but the list is synchronised every 15 minutes from LDAP, Active Directory or Microsoft 365, see [Recipient directories](directories.md). Addresses added by hand stay |

The recipient list is only used in "list" and "directory" mode, but can be
prepared before switching to it. It shows for each address whether it was
added by hand or by the directory.

## Testing the destination

**Test destination** on the domain page talks SMTP to the downstream server
(each of its MX hosts with "use MX") the way the gateway delivers, without
sending anything: connect, EHLO, STARTTLS, `MAIL FROM:<>`, then `RCPT TO` for
an address you enter (optional) and for a made-up address of the domain. It
shows per host:

- whether it answers and accepts the sender;
- **TLS**: offered or not, and whether its certificate is trusted (Postfix
  encrypts without checking it, like most mail servers);
- your address: accepted, deferred or refused;
- the made-up address: a server that **accepts** it accepts any address, so
  "Verify with the downstream server" cannot reject wrong recipients there;
  use a recipient list or a directory instead;
- the SMTP dialogue.

Save changes to the delivery settings before testing them. API:
`POST /api/v1/domains/{id}/test-destination` with an optional `address`.

## Domain allow and block lists

On the domain page you manage entries that apply to this domain only:

- **Sender** entries match the envelope sender and the `From` header:
  `user@example.org`, `*@example.org` (also written `@example.org` or
  `example.org`) or patterns like `*@*.example.org`.
- **IP** entries match the sending host's address or network.

**Block** entries apply when at least one recipient of a message is in the
domain. **Allow** entries apply only when every recipient is in the domain,
so a domain cannot allow mail for recipients of other domains on the same
message. Global entries win over domain entries; see [Filtering](filtering.md).

## DKIM signing

Outbound mail from relay networks is signed with the sender domain's DKIM key.
On the domain page (or `PUT /api/v1/domains/{id}/dkim`):

- **Generate** a key (2048 bit recommended, or 4096 bit) with a selector, or
  **import** an existing unencrypted RSA key in PEM format.
- Publish the shown **TXT record** at `<selector>._domainkey.<domain>`. The
  page also shows it in zone file syntax, split into 255 byte strings.
- **Rotate** by creating a key with a new selector: publish its record first,
  then replace the key. Delete old records once no signed mail is in transit.

Outbound mail released from the quarantine is signed with the same key.

## Outbound mail

Your own mail servers can send outbound mail through the gateway:

1. **Settings → Relay networks**: add the address or network of each server.
   Only add networks you control: the gateway relays their mail anywhere.
2. Configure the gateway as the server's smart host (port 25).
3. Add the gateway to the SPF record of the sending domains and set up DKIM.

Outbound mail is scanned like inbound mail. Rejected outbound mail is
quarantined for administrators (Settings: "Quarantine rejected outbound mail",
on by default); mailbox users never see outbound mail. Limits per sender and
data loss prevention rules are described in [Outbound mail](outbound.md).

The relay networks are also written for Postfix, as
`/var/lib/crassostrea/postfix/relay_networks.cidr`
(`crassostrea-postfix-reload.path` reloads Postfix when it changes; earlier
versions used `/etc/postfix/dynamic/`).

## Deleting a domain

Deleting a domain also deletes its recipients, list entries, mailbox users and
DKIM key; mail for it is no longer accepted. Quarantined mail stays until it
expires. Only full administrators add and delete domains; [domain
administrators](administrators.md) edit the domains assigned to them.

# Troubleshooting

## First checks

```bash
systemctl --failed
systemctl list-units 'crassostrea*' postfix rspamd nginx valkey mariadb unbound 'clamd@*'
journalctl -u <unit> -n 100
sudo system/cluster.sh status        # in a cluster
```

**Monitoring** shows the same service list and the alert checks, and
**Monitoring → Logs** the journal of each service; the dashboard shows the
health of the database, Valkey and Rspamd. **DNS & ports** checks the records
and connections the gateway needs.

## Mail is not accepted

| Symptom (in the Postfix log) | Cause and fix |
| --- | --- |
| `Relay access denied` | The recipient's domain is not configured or not active |
| `Recipient address rejected: User unknown` | Recipient validation: the address is not on the domain's list, or the downstream server rejects it ("verify" mode). With a directory: the address is not in it, or the last synchronisation was refused (Directories page) |
| `Message size exceeds fixed limit` | Larger than the maximum message size (Settings → General, Messages) |
| `451 ... Sending limit ... reached` (outbound) | The sender reached its [outbound limit](outbound.md#outbound-limits); reset it on the page |
| `451 ... Rate limit of ... reached` (inbound) | A sending server or recipient reached an [inbound rate limit](protection.md#rate-limits): reset it on the Rate limits page, raise the limit, or exempt the server's network |
| `too many connections from`, `... rate limit exceeded` | Postfix's SMTP connection limits (Rate limits page) |
| `Temporary lookup failure`, 4xx | The database is not reachable; in a cluster this node lost the majority (see [High availability](high-availability.md#failures)) |
| `blocked using <list>` (postscreen) | The sender is on a DNS blocklist |
| Connection refused or timeout on port 25 | firewalld, the provider's port 25 block, or Postfix not running |
| Connections from one sender's address time out, nothing in the log | netfilter banned the address; check **Banned IPs** |

Send a test message and follow it in the Postfix log (**Monitoring → Logs**, search for the
address or queue ID) and the **Mail log**.

## Mail is accepted but not delivered

- **Mail queue**: messages with a delay reason. Typical causes: the downstream
  server refuses the gateway, wrong host or port in the domain settings, or
  TLS problems. "Deliver now" retries immediately.
- Clean mail goes to the downstream server; check that it accepts mail from
  the gateway's address.

## Legitimate mail is rejected or marked

1. Find the message in the **Quarantine** (rejected) or the **Mail log**, and
   look at the rules with the highest scores. Rejections by the gateway's own
   checks name them: `ATTACHMENT_BLOCKED`, `CONTENT_RULE_REJECT`,
   `OUTBOUND_DLP`, `IMPERSONATION_*`, `LOOKALIKE_DOMAIN`,
   `SPOOFED_OWN_DOMAIN`, `HASH_*`, `ATTACHMENT_TOO_LARGE`; adjust the
   corresponding page ([Threat protection](protection.md),
   [Outbound mail](outbound.md)).
2. Release it and learn it as ham.
3. If a sender is always affected, add a sender allow entry (personal, domain
   or global). It does not help senders that fail their own DMARC or SPF
   policy: they need to fix their DNS.
4. Raise the thresholds of the domain or user if the scores are generally
   close.

## Spam is not caught

- Learn missed spam (Training page, the quarantine, or let users
  [report it](filtering.md#reports-from-users)).
- Check that DNS blocklists work: `dig +short 2.0.0.127.zen.spamhaus.org`
  should answer `127.0.0.2`. If it does not, the resolver is not the local
  unbound, or Spamhaus blocks your network (set `SPAMHAUS_DQS_KEY`).
- Lower the domain's thresholds.

## Web interface

| Symptom | Cause and fix |
| --- | --- |
| Browser warns about the certificate | Let's Encrypt has not issued one yet: `journalctl -u crassostrea-acme` |
| 404 for every page | The web interface was opened by IP address or by a name that is not configured. nginx answers only to the hostname, `ADDITIONAL_SAN` and the host of `WEB_PUBLIC_URL`; add the name (Certificates & TLS, or `crassostrea.conf` and `configure.sh`) |
| 502 Bad Gateway | `crassostrea-web` is not running: `journalctl -u crassostrea-web` |
| "Too many failed attempts" | Login throttling; wait 15 minutes, or `helper-scripts/reset-admin.sh` (also clears the throttling) |
| Lost the admin password | `sudo helper-scripts/reset-admin.sh admin` |
| Mail queue page: "not reachable" | `crassostrea-postfix-control.socket` is not active |
| Hostname, IP addressing, Certificates & TLS or Backups: "is crassostrea-...path installed?" | The root service is missing: run `system/configure.sh`, check `systemctl status crassostrea-hostname.path crassostrea-tls.path crassostrea-network.path crassostrea-backup-request.path` |
| An IP addressing change came back as "reverted" | The test after the change found no internet access or no name resolution for an enabled family; the page shows the log |
| A Let's Encrypt name cannot be applied | It needs a passed staging test of exactly these names in the last 24 hours; the failed test's log on the page says why (DNS, port 80) |
| Link protection shows "cannot be checked right now" | Valkey is not reachable |
| Directory: "would lose ... recipients" | The synchronisation found far fewer addresses than before; check the filter and permissions, then "Synchronise, remove anyway" if it is right |

## Certificates

`journalctl -u crassostrea-acme` shows why a request failed. The most common
cause is that `http://<name>/.well-known/acme-challenge/` does not reach the
gateway: DNS, port 80 in the firewall, or a proxy in front of it. Test with
`LE_STAGING=y` to avoid Let's Encrypt's rate limits.

## Reports and journaling

| Symptom | Cause and fix |
| --- | --- |
| No DMARC reports arrive | The `_report._dmarc` authorisation record in the gateway's zone is missing (DMARC & TLS reports page), or the `rua` points elsewhere. Reports arrive about daily |
| A report is listed as "not for a domain of this gateway" | The reported domain is not configured on the gateway |
| Journal copies wait | The archive refuses them or cannot be reached; the Journaling page shows the last error. They are sent when it accepts them |
| User reports are "refused" | The reporter is not a user of a protected domain as far as the gateway can tell (no relay network, SPF or DKIM), reporting is off, or the hourly limit was reached |

## SELinux

```bash
sudo ausearch -m AVC -ts recent
```

A service that fails with "Permission denied" on a database or network
connection although the configuration is right may run in the wrong SELinux
domain: `ps -eZ | grep crassostrea` should show `unconfined_service_t`, not
`init_t`. `update.sh` (or `system/install.sh`) restores the file labels.

If a denial blocks something legitimate, report it, and meanwhile build a local
module with `sudo helper-scripts/selinux-local-policy.sh`.

## Logs page

| Symptom | Cause and fix |
| --- | --- |
| "The log reader (crassostrea-logs.socket) is not reachable" | `sudo systemctl enable --now crassostrea-logs.socket` (or run `system/configure.sh`) |
| A service shows no lines | It has not logged since the journal was rotated, or the level filter hides its lines |
| Only this node's lines | Expected for everything but the Postfix log: open the page on the other node |

## Documentation on the gateway

| Symptom | Cause and fix |
| --- | --- |
| `/docs/` sends you to the login page although you are logged in | The login ended, or two-factor login must still be set up: log in again |
| `/docs/` answers 404 | The site was not built: `sudo system/docs.sh status`, then `rebuild`; the build's messages show why |
| "No documentation to build" in `configure.sh` | `DOCS_SOURCE` is empty and there is no `crassostrea-docs` next to the gateway's checkout: `sudo system/docs.sh install SOURCE` |
| `/docs/` shows an old version | `sudo system/docs.sh update` (or `rebuild` for a local directory with uncommitted changes) |

## Cluster

| Symptom | Cause and fix |
| --- | --- |
| A new node does not join (`mariadb` times out) | The other members do not let it in yet: they apply the new membership within a minute (`journalctl -u crassostrea-cluster` on them); check the firewall zone `crassostrea-cluster` |
| Join: "The join information is invalid, used or expired" | It works once, for an hour: create a new one on a member |
| Join: "The member's certificate does not match" | The member's certificate changed since the join information was made (a renewal): create a new one |
| Join: the member "does not answer" | The new gateway cannot reach the member's cluster address on its HTTPS port |
| Join stops after "added this gateway to the cluster" | The member did not hand out the bundle: `journalctl -u crassostrea-cluster` on it. The members already list the new gateway: join it with a bundle (`system/cluster.sh bundle`, `setup.sh --join`) or remove it under Settings → Cluster |
| A member shows "no longer in the cluster's membership" | It was removed while it was running: stop it (`system/cluster.sh leave`) and reinstall it to use it again |
| `cluster.sh status`: database `non-Primary` | The node cannot reach a majority: network between the members, or too many members down |
| MariaDB does not start after a power loss | Expected: [recover](high-availability.md#recovery-after-all-nodes-were-down) with `system/cluster.sh bootstrap` |
| Valkey: "no Sentinel answers" | Sentinel is down on the members, or port 26379 is blocked |
| The floating address does not move | keepalived needs VRRP between the nodes and a shared network segment; `journalctl -u keepalived` |
| Monitoring → Cluster: a node "has not reported" | Its database is not reachable or out of the majority, or `crassostrea-cluster-report` stopped: `journalctl -u crassostrea-cluster-report` on it |
| Monitoring → Cluster: "the cluster service does not report" | `crassostrea-cluster` stopped on that node: `journalctl -u crassostrea-cluster` |
| Maintenance "requested but not in effect" | The node's `crassostrea-cluster-report` or `crassostrea-cluster` does not run, or the node is down: it applies the request when it returns |
| A node in maintenance still gets mail | Mail already queued is delivered; new connections get 421. Relay clients that only know this node queue the mail until maintenance ends |
| Maintenance refused: "is needed to accept mail" | No other node is reporting, in sync and out of maintenance. Fix that first, or use `--force` |

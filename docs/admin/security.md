# Security

## Exposed services

Only these listen on public addresses:

| Port | Service |
| --- | --- |
| 25 | Postfix (postscreen) |
| 80 | nginx: ACME challenges and the redirect to HTTPS |
| 443 | nginx: web interface, API, Rspamd UI at `/rspamd/`, link protection at `/l/`, Prometheus metrics at `/metrics` |

Every other service listens on loopback only. In a cluster the replication
ports are additionally open, but only to the other members' addresses and only
with TLS and certificates of the cluster CA. `configure.sh` opens the ports in
firewalld.

Restrict the Rspamd UI with `RSPAMD_UI_ALLOW`, and consider allowing the web
interface only from your networks with firewalld rich rules or an upstream
firewall; mailbox users, digest links and link protection need it from
wherever your users are.

nginx answers only to the gateway's names: `CRASSOSTREA_HOSTNAME`, the
certificate's `ADDITIONAL_SAN` and the host of `WEB_PUBLIC_URL`. A request for
any other name, or for an IP address, gets 404 (scanners that walk address
ranges see nothing); only ACME challenges are answered for every name, since
a name is validated before it is configured.

What answers without a login:

- `/l/<id>`: only links the gateway rewrote (unknown ids get an error page,
  so it is no open redirector);
- `/metrics`: only with the bearer token created under Settings → Alerts & export;
- digest links: only with their token;
- links to set a mailbox user's password (`/account/set-password/<token>`):
  only with their token, once, until they expire.

Administrator logins can be limited to networks you choose, see
[Access by network](administrators.md#access-by-network).

Port 25 also accepts mail for three addresses of the gateway's own hostname,
which is not filtered but handed to the web interface: `reports@` (DMARC and
TLS reports, only displayed) and `spam@`/`ham@` (learned only when reporting
is on and the sender is verified as a user of a protected domain). Every
other address of the hostname is rejected.

## Secrets

| Secret | Where |
| --- | --- |
| Database, Valkey and Rspamd passwords, session key, initial admin password | `/etc/spamfilter/spamfilter.conf` (root and group `spamfilter` only) |
| TLS key | `/etc/spamfilter/ssl/key.pem` (0600) |
| DKIM private keys | the database (and Valkey, for Rspamd) |
| Directory bind passwords, Microsoft 365 client secrets | the database; never shown again in the interface or the API |
| MalwareBazaar and VirusTotal keys | the database and Valkey (for Rspamd), not in map files |
| Two-factor secrets | the database; recovery codes as hashes |
| Cluster CA key and member keys | `/etc/pki/spamfilter/` |
| API keys, the metrics token, digest link and password link tokens | stored as hashes only |
| Backups | `/var/lib/spamfilter/backups/` (root and group `spamfilter`; files 0640): they contain all of the above |

Database dumps and Valkey dumps therefore contain secrets; protect backups
accordingly. Join bundles contain everything needed to join the cluster:
transfer them securely and delete them afterwards.

## Hardening built in

- **Services never run from the repository**: `configure.sh` copies the code
  to root-owned locations (`/usr/local/lib/spamfilter`,
  `/usr/local/libexec/spamfilter`), so the account owning the checkout cannot
  change what runs as root.
- The web interface runs as the unprivileged `spamfilter` user with systemd
  sandboxing. Mail queue actions go through a small root helper that accepts
  only fixed actions with validated queue IDs. Changes it cannot make itself
  (hostname, certificates and TLS versions, IP addressing, backups) are
  requests in `/var/lib/spamfilter/web/` that root services validate again
  before acting.
- **SELinux** stays enforcing; the needed port and file labels and booleans
  are set (the programs systemd starts are labelled `bin_t`, so they run as
  services, `unconfined_service_t`, instead of in systemd's own domain
  `init_t`, which may not connect to the database).
  `helper-scripts/selinux-local-policy.sh` turns remaining denials into a local
  module if you need it.
- **Logins** are throttled per address and per account, failed attempts and
  invalid API keys are reported to netfilter, sessions are bound to the client
  and end 8 hours after the login (`WEB_SESSION_LIFETIME`), and cookies are
  HTTPS-only with CSRF protection. Administrators can use (or be required to
  use) **two-factor login**; **domain administrators** reach only their
  domains.
- The **audit log** records every change, login and backup download, and
  can be sent to a SIEM.
- **Digest links** never act on a GET request, are not sent to other sites and
  are removed from the web server's and the application's logs.
- **HTML of messages** is shown only sanitized, in a sandboxed frame without
  scripts, forms, followable links or remote content.
- **Relay networks** are the only clients that may send outbound mail. Do not
  add networks you do not control.
- The HAProxy listener (`HAPROXY_LISTENER`) trusts the PROXY header of its
  clients: expose it to your proxy only.
- **Pinned downloads**: acme.sh and postfix-tlspol are cloned at a fixed tag
  and verified against the expected commit, smtp-cli against a checksum.

## Mailbox users

Mailbox users only reach their own inbound quarantine and personal settings.
Their personal allow and block lists and thresholds only apply to messages
whose recipients all belong to them, so one user's settings never affect mail
for others.

## Reporting vulnerabilities

See `SECURITY.md` in the repository.

# Configuration reference

## The configuration file

All settings of the host live in **`/etc/spamfilter/spamfilter.conf`**. It
contains `KEY=value` lines and is read like a systemd `EnvironmentFile`:
values are taken literally, one pair of surrounding quotes is removed, nothing
is expanded or executed. Quote values that contain spaces.

`setup.sh` creates the file; it is readable by root and the `spamfilter`
group only, because it contains all passwords.

**After a change**, apply it:

```bash
sudo /opt/crassostrea/system/configure.sh
```

`configure.sh` re-renders the configuration of every service and restarts
them. Settings that are managed in the web interface (domains, thresholds,
lists, digests, relay networks, and the features of the Filter, Mail and
System pages) are not in this file; they are in the database.

A few settings of this file can also be changed in the web interface, by root
services that edit it and apply the change: `CRASSOSTREA_HOSTNAME`
([Hostname](network.md#changing-the-hostname)), `IPV4_ENABLED` and
`IPV6_ENABLED` ([IP addressing](network.md#ip-addressing)), and
`ADDITIONAL_SAN`, `SKIP_LETS_ENCRYPT`, `HTTPS_TLS_MIN` and `SMTP_TLS_MIN`
([Certificates and TLS](certificates.md)). Later `configure.sh` runs keep
their values. In a cluster these pages change nothing; edit the file on each
member.

## Settings

### Gateway

| Setting | Default | Meaning |
| --- | --- | --- |
| `CRASSOSTREA_HOSTNAME` | asked by setup | The gateway's fully qualified hostname: the MX target of your domains, the certificate name and Postfix's `myhostname`. In a cluster the name of this member |
| `LOG_LINES` | `9999` | Mail log lines kept for the web interface |

### Web interface

| Setting | Default | Meaning |
| --- | --- | --- |
| `WEB_ADMIN_PASSWORD` | random | Password of the initial `admin` account, used only while the database has no administrator |
| `WEB_SECRET_KEY` | random | Signs session cookies. Changing it logs everybody out. Must be the same on all members of a cluster |
| `WEB_SECURE_COOKIES` | `y` | Send session cookies over HTTPS only. Set `n` only for tests without TLS |
| `WEB_PUBLIC_URL` | `https://CRASSOSTREA_HOSTNAME` (with `HTTPS_PORT` if not 443) | Base URL of the links in quarantine digests. Its host is one of the names nginx answers to |
| `RSPAMD_PASSWORD` | random | Password of the Rspamd controller, used by the web interface and for the Rspamd UI at `/rspamd/` |
| `WEB_LOGIN_MAX_FAILURES` | `10` | Failed logins or API key attempts per client IP within 15 minutes before further ones are refused |
| `WEB_LOGIN_MAX_ACCOUNT_FAILURES` | `30` | Failed logins per account (from all addresses) within 15 minutes; reaching it locks the account for the rest of the window |
| `WEB_SESSION_LIFETIME` | `28800` (8 hours) | Seconds after which a login ends, however active it is; closing the browser ends it earlier |

### Database and Valkey

| Setting | Default | Meaning |
| --- | --- | --- |
| `DBNAME`, `DBUSER` | `crassostrea` | MariaDB database and user (letters, digits and `_`) |
| `DBPASS` | random | Database password (letters and digits) |
| `REDISPASS` | random | Valkey password (letters and digits) |

`configure.sh` sets the database user's password from `DBPASS` on every run,
so you can change it here and apply it.

### Web server

| Setting | Default | Meaning |
| --- | --- | --- |
| `HTTP_PORT`, `HTTPS_PORT` | `80`, `443` | Ports of nginx. Other ports are labelled for SELinux automatically |
| `HTTP_BIND`, `HTTPS_BIND` | empty (all) | One local address to listen on |
| `HTTP_REDIRECT` | `y` | Redirect HTTP to HTTPS (ACME challenges are still answered over HTTP) |
| `TRUSTED_PROXIES` | empty | Comma separated reverse proxies in front of nginx whose `X-Forwarded-For` header is trusted |
| `RSPAMD_UI_ALLOW` | empty (everyone) | Comma separated networks allowed to open the Rspamd UI at `/rspamd/` (also its `/rspamd/metrics`) |
| `HTTPS_TLS_MIN` | `1.2` | Oldest TLS version of HTTPS: `1.2`, `1.1` or `1.0`. The older ones also relax nginx's OpenSSL policy (security level 0, SHA-1) |

### Network

| Setting | Default | Meaning |
| --- | --- | --- |
| `IPV4_ENABLED` | `true` | Use IPv4 on the network interface (the host's addresses, Postfix, unbound, nginx). `true`/`false`; at least one of the two |
| `IPV6_ENABLED` | detected | The same for IPv6. Replaces `ENABLE_IPV6` of earlier versions, which `configure.sh` renames in the file |

Loopback keeps both families whatever is set. See
[IP addressing](network.md#ip-addressing).

### Certificates

| Setting | Default | Meaning |
| --- | --- | --- |
| `ADDITIONAL_SAN` | empty | Comma separated additional names for the certificate. nginx answers only to these, the hostname and the host of `WEB_PUBLIC_URL`; other names and IP addresses get 404 |
| `SKIP_LETS_ENCRYPT` | `n` | `y`: keep the self-signed or your own certificate in `/etc/spamfilter/ssl/{cert,key}.pem` |
| `ACME_DNS_CHALLENGE` | `n` | `y`: validate with DNS-01 instead of HTTP-01 |
| `ACME_DNS_PROVIDER` | `dns_xxx` | The acme.sh DNS API, e.g. `dns_cf`; credentials go into `/etc/spamfilter/acme/dns-01.conf` |
| `ACME_ACCOUNT_EMAIL` | `me@example.com` | Contact address of the Let's Encrypt account (the placeholder registers without one) |
| `SKIP_HTTP_VERIFICATION` | `n` | `y`: skip the check that every name reaches this host over HTTP before requesting |
| `LE_STAGING` | `n` | `y`: use the Let's Encrypt staging environment (for tests) |
| `SMTP_TLS_MIN` | `1.2` | Oldest TLS version of inbound SMTP (`1.2`, `1.1`, `1.0`); outbound SMTP keeps 1.2 |

See [Certificates](certificates.md).

### Filter components

| Setting | Default | Meaning |
| --- | --- | --- |
| `SKIP_CLAMD` | `n` (`y` offered on small hosts) | `y`: no ClamAV virus scanning |
| `SKIP_OLEFY` | `n` | `y`: no Olefy macro scanning of Office documents |
| `SPAMHAUS_DQS_KEY` | empty | Spamhaus Data Query Service key. Without it, the Spamhaus lists are disabled on networks that Spamhaus blocks |
| `MANAGE_RESOLVER` | `y` | Make the local unbound the system resolver (`/etc/resolv.conf`, NetworkManager stops managing it). The original file is kept as `/etc/resolv.conf.spamfilter-orig` |
| `HAPROXY_LISTENER` | `n` | `y`: an additional SMTP listener that expects the PROXY protocol, for an HAProxy in front of the gateway |
| `HAPROXY_LISTENER_BIND` | `127.0.0.1:10025` | Address of that listener. Only expose it to the proxy: its clients can claim any source address |

### Watchdog

| Setting | Default | Meaning |
| --- | --- | --- |
| `USE_WATCHDOG` | `y` | Run `spamfilter-watchdog`, which checks the services and restarts unhealthy ones |
| `WATCHDOG_NOTIFY_EMAIL` | unset | Comma separated recipients of notifications (no spaces, no quotes), sent from `watchdog@CRASSOSTREA_HOSTNAME` |
| `WATCHDOG_NOTIFY_WEBHOOK` | unset | URL that receives notifications as JSON POST |
| `WATCHDOG_NOTIFY_WEBHOOK_BODY` | unset | JSON body in single quotes; `${SUBJECT}` and `${BODY}` are replaced |
| `WATCHDOG_NOTIFY_BAN` | `n` | Notify about every banned IP (with a whois lookup) |
| `WATCHDOG_NOTIFY_START` | `y` | Notify when the watchdog starts |
| `WATCHDOG_SUBJECT` | `Watchdog ALERT` | Subject prefix of notification mails |
| `WATCHDOG_VERBOSE` | `n` | Verbose watchdog logging |

### High availability

| Setting | Default | Meaning |
| --- | --- | --- |
| `HA_NODES` | empty (single gateway) | Full cluster nodes: comma separated `hostname=IPv4 address` pairs |
| `HA_WITNESS` | empty | The vote-only witness: `hostname=IPv4 address` |
| `HA_VIP` | empty | Floating address with prefix for the web interface, e.g. `192.0.2.10/24` |
| `HA_VIP_INTERFACE` | the interface with the node's address | Network interface for `HA_VIP` |
| `HA_VIP_ROUTER_ID` | `51` | VRRP router ID (1-255), unique in the network |

Change membership with `system/cluster.sh members`, see
[High availability](high-availability.md).

## Overrides that survive configure.sh

`configure.sh` overwrites the files it generates. Lasting changes go into:

| File | Purpose |
| --- | --- |
| `/etc/postfix/extra.cf` | Postfix `main.cf` settings, appended last (starts with `myhostname`) |
| `/etc/postfix/custom_transport.pcre` | Additional transport rules |
| `/etc/postfix/custom_postscreen_whitelist.cidr` | Addresses postscreen always lets through or always rejects |
| `/etc/postfix/dns_blocklists.cf` | The postscreen DNS blocklists and their weights; delete it to restore the defaults |
| `/etc/rspamd/override.d/*.custom.inc` | Rspamd overrides |
| `/etc/rspamd/rspamd.conf.local` | Created once, then left alone |

Files in the repository's `data/conf/` are copied to `/etc` on every run:
change those in the repository, not in `/etc`.

**Hooks**: executable files in the repository's `data/hooks/postfix/`,
`data/hooks/rspamd/` and `data/hooks/unbound/` run after the component's
configuration was rendered, e.g. to add a setting `configure.sh` does not
know.

# Configuration reference

## The configuration file

Most settings are made in the web interface (or the REST API) and kept in the
database: they are the same on every member of a cluster and take effect
without a `configure.sh` run (see [Settings of the web interface](#settings-of-the-web-interface)).
What remains in **`/etc/crassostrea/crassostrea.conf`** is what the services
need before the database answers (names, passwords), what differs per
cluster member, and what changes installed packages or listening ports. It
contains `KEY=value` lines and is read like a systemd `EnvironmentFile`:
values are taken literally, one pair of surrounding quotes is removed, nothing
is expanded or executed. Quote values that contain spaces.

`setup.sh` creates the file; it is readable by root and the `crassostrea`
group only, because it contains all passwords.

**After a change**, apply it:

```bash
sudo /opt/crassostrea/system/configure.sh
```

`configure.sh` re-renders the configuration of every service and restarts
them.

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

### Web interface

| Setting | Default | Meaning |
| --- | --- | --- |
| `WEB_ADMIN_PASSWORD` | random | Password of the initial `admin` account, used only while the database has no administrator |
| `WEB_SECRET_KEY` | random | Signs session cookies. Changing it logs everybody out. Must be the same on all members of a cluster |
| `WEB_SECURE_COOKIES` | `y` | Send session cookies over HTTPS only. Set `n` only for tests without TLS |
| `WEB_PUBLIC_URL` | `https://CRASSOSTREA_HOSTNAME` (with `HTTPS_PORT` if not 443) | Base URL of the links in quarantine digests. Its host is one of the names nginx answers to |
| `RSPAMD_PASSWORD` | random | Password of the Rspamd controller, used by the web interface and for the Rspamd UI at `/rspamd/` |

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
| `HTTPS_TLS_MIN` | `1.2` | Oldest TLS version of HTTPS: `1.2`, `1.1` or `1.0`. The older ones also relax nginx's OpenSSL policy (security level 0, SHA-1) |

### Network

| Setting | Default | Meaning |
| --- | --- | --- |
| `IPV4_ENABLED` | `true` | The gateway uses IPv4 (Postfix, unbound, nginx). `true`/`false`; at least one of the two. The IP addressing page sets it: `true` while any interface has IPv4 |
| `IPV6_ENABLED` | detected | The same for IPv6. |

Loopback keeps both families whatever is set. See
[IP addressing](network.md#ip-addressing).

### Certificates

| Setting | Default | Meaning |
| --- | --- | --- |
| `ADDITIONAL_SAN` | empty | Comma separated additional names for the certificate. nginx answers only to these, the hostname and the host of `WEB_PUBLIC_URL`; other names and IP addresses get 404 |
| `SKIP_LETS_ENCRYPT` | `n` | `y`: keep the self-signed or your own certificate in `/etc/crassostrea/ssl/{cert,key}.pem` |
| `ACME_DNS_CHALLENGE` | `n` | `y`: validate with DNS-01 instead of HTTP-01 |
| `ACME_DNS_PROVIDER` | `dns_xxx` | The acme.sh DNS API, e.g. `dns_cf`; credentials go into `/etc/crassostrea/acme/dns-01.conf` |
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
| `MANAGE_RESOLVER` | `y` | Make the local unbound the system resolver (`/etc/resolv.conf`, NetworkManager stops managing it). The original file is kept as `/etc/resolv.conf.crassostrea-orig` |
| `HAPROXY_LISTENER` | `n` | `y`: an additional SMTP listener that expects the PROXY protocol, for an HAProxy in front of the gateway. Its postscreen keeps its own cache (`postscreen_cache_haproxy`), so a client that passed on port 25 is new there |
| `HAPROXY_LISTENER_BIND` | `127.0.0.1:10025` | Address of that listener. Only expose it to the proxy: its clients can claim any source address |

### Watchdog

| Setting | Default | Meaning |
| --- | --- | --- |
| `USE_WATCHDOG` | `y` | Run `crassostrea-watchdog`, which checks the services and restarts unhealthy ones. Its notices are set in the web interface |
| `WATCHDOG_VERBOSE` | `n` | Verbose watchdog logging |

### Documentation

See [Documentation on the gateway](installation.md#documentation-on-the-gateway);
change them with `system/docs.sh`.

| Setting | Default | Meaning |
| --- | --- | --- |
| `DOCS_ENABLED` | `n` | `y`: build this documentation and serve it at `/docs/` to logged-in administrators |
| `DOCS_SOURCE` | empty | The `crassostrea-docs` repository: a git URL or a directory (no spaces or quotes). Empty: the checkout next to the gateway's (`../crassostrea-docs`), else `https://github.com/SnailShea/crassostrea-docs.git` |
| `DOCS_REF` | empty | Branch or tag of a git URL; empty: its default branch |

### High availability

| Setting | Default | Meaning |
| --- | --- | --- |
| `HA_NODES` | empty (single gateway) | Full cluster nodes: comma separated `hostname=IPv4 address` pairs |
| `HA_WITNESS` | empty | The vote-only witness: `hostname=IPv4 address` |
| `HA_VIP` | empty | Floating address with prefix for the web interface, e.g. `192.0.2.10/24` |
| `HA_VIP_INTERFACE` | the interface with the node's cluster address | Network interface the floating address is added to, e.g. the public one when the cluster addresses are on a private network. VRRP always runs on the interface with the cluster address ([Cluster and public network](high-availability.md#cluster-and-public-network)) |
| `HA_VIP_ROUTER_ID` | `51` | VRRP router ID (1-255), unique in the network |

Settings → Cluster in the web interface sets `HA_NODES` and `HA_WITNESS` on
every member when nodes join or are removed (from the cluster's membership in
the database); on the command line `system/cluster.sh members`. See
[High availability](high-availability.md).

## Settings of the web interface

Settings of services outside the web interface that are made in it (also
with `PATCH /api/v1/settings`), not in `crassostrea.conf`. The web interface
runs unprivileged, so it hands the ones of other services
over, and root services check them again before applying them on every
member of a cluster:

| Setting | Where | Default | Applied by |
| --- | --- | --- | --- |
| How long a login lasts | Settings → General, Administrators (Sign-in) | 480 minutes | the web interface |
| Failed logins per address, per account, and how long they count | Settings → General, Administrators (Sign-in) | 10, 30, 15 minutes | the web interface |
| Redirect HTTP to HTTPS | Settings → General, Web server | on | `crassostrea-nginx.service` (ACME challenges are still answered over HTTP) |
| Networks of the Rspamd UI (`/rspamd/`, also `/rspamd/metrics`) | Settings → General, Web server | any | `crassostrea-nginx.service` |
| Reverse proxies whose `X-Forwarded-For` is trusted | Settings → General, Web server | none | `crassostrea-nginx.service` |
| Spamhaus Data Query Service key | Settings → General, Scanning | none | `crassostrea-spamhaus.service` (Postfix's postscreen and Rspamd) |
| Postfix log lines kept for the Logs page | Settings → General, Messages | 9999 | `crassostrea-logship.service` (from Valkey) |
| The watchdog's notices: recipients, webhook and its body, subject, at its start, for bans | Settings → Alerts & export, Watchdog notices | none, start on | `crassostrea-watchdog.service` (from Valkey, at every notice) |
| What `update.sh` installs: the newest release or the newest commit of a branch | Monitoring → Version | the newest release | `update.sh` (reads the database) |

`crassostrea-nginx.service` keeps the previous configuration when nginx
refuses a value, `crassostrea-spamhaus.service` the previous Rspamd rules
when Rspamd refuses them; both log to the journal
(`journalctl -u crassostrea-nginx -u crassostrea-spamhaus`).

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

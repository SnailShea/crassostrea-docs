# Hostname, IP addressing and DNS

## Changing the hostname

**Network → Hostname & IP** (Gateway hostname) renames a single gateway, for example when
the MX and PTR records were created under another name. A root service
(`spamfilter-hostname.path`, `system/libexec/set-hostname`):

1. sets `CRASSOSTREA_HOSTNAME` in `spamfilter.conf` (later `configure.sh` runs
   keep it) and Postfix's `myhostname`;
2. updates the sender of DMARC reports;
3. restarts the services that read the name, the web interface included
   (reload the page if it does not answer for a moment);
4. requests a certificate for the new name.

A name that does not resolve to the gateway yet is accepted with a warning: the
current certificate stays until Let's Encrypt can validate the new name, which
is retried daily. Also `PUT /api/v1/hostname`.

In a cluster the hostname identifies the member (Galera, keepalived, the
certificates of the cluster CA) and is not changed here: replace the member
instead (`system/cluster.sh leave`, `members`, `bundle`, then `setup.sh
--join` on a host with the new name).

## IP addressing

**Network → Hostname & IP** (single gateway) shows and changes the network
interface's configuration through NetworkManager:

- per family (IPv4, IPv6): **manual** addresses with prefix length, gateway and
  DNS servers, or **automatic** (DHCP; for IPv6 SLAAC/DHCPv6);
- whether the gateway uses **IPv4** and **IPv6** at all (`IPV4_ENABLED`,
  `IPV6_ENABLED` in `spamfilter.conf`). Off means the interface has no address
  of that family and Postfix, unbound and nginx do not use it. Loopback keeps
  both: the services talk to each other over `127.0.0.1`. **Only one family
  can be off.**

The DNS servers are not used while the gateway resolves names itself
(`MANAGE_RESOLVER=y`, the default).

After **Save**, the root service `spamfilter-network.path`
(`system/libexec/network-apply`) applies the change, reconfigures the services
for the families, and **tests** that every enabled family reaches the internet
(1.1.1.1 and 9.9.9.9, or their IPv6 counterparts, by ICMP or TCP) and that
names resolve. If a test fails, the previous settings come back, and the page
shows the outcome with the log. This takes up to a minute.

!!! warning
    The page warns when you are connected over the family you are about to
    change. If the address you use changes, open the web interface at the new
    address; if a change locks you out anyway, the automatic test restores the
    old settings only when the internet is unreachable, not when your own
    network is.

In a cluster the members' addresses identify them and are not changed here.
Also `GET/PUT /api/v1/ip-config`.

## DNS and ports

**Network → DNS & ports** lists what the gateway needs from DNS and the network
and checks it from the server on every visit:

| Area | Checks |
| --- | --- |
| Hostname | A/AAAA records pointing to this server, reverse DNS (PTR) matching the name in both directions, a CAA record that allows Let's Encrypt |
| Each active domain | MX records pointing to the gateway or a cluster member (backup MX hosts that bypass the filter are flagged), the DKIM record matching the signing key, and with relay networks SPF (evaluated for the server's outgoing address) and DMARC |
| Ports | SMTP, HTTP and HTTPS accepting connections on the server |
| Connections | DNS, each domain's downstream server, Let's Encrypt and the other cluster members being reachable |

Lookups and connections run in parallel with short timeouts. A firewall in
front of the server cannot be seen from inside: the page explains how to test
port 25 from another network. Outgoing SMTP to the internet is not tested, as
that would contact third-party servers.

The [DMARC & TLS reports](reports.md) page checks the report records of each
domain.

# Hostname, IP addressing and DNS

## Changing the hostname

**Network → Hostname & IP** (Gateway hostname) renames a single gateway, for example when
the MX and PTR records were created under another name. A root service
(`crassostrea-hostname.path`, `system/libexec/set-hostname`):

1. sets `CRASSOSTREA_HOSTNAME` in `crassostrea.conf` (later `configure.sh` runs
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

**Network → Hostname & IP** (single gateway) shows and changes the
configuration of the server's network interfaces through NetworkManager. With
several interfaces, a table lists them (addresses, which has the default
route); choose one to change it (`?device=` in the address). Bridges such as
`docker0`, loopback and interfaces NetworkManager does not manage are left out.

For each interface and family (IPv4, IPv6):

- **on or off**: off means the interface has no address of that family;
- **manual** addresses with prefix length, an optional gateway and DNS servers,
  or **automatic** (DHCP; for IPv6 SLAAC/DHCPv6). An interface of an internal
  network needs no gateway.

Either family, or both, can be off on any interface **as long as another
interface keeps an address**; the last interface with an address keeps at
least one family. The gateway's services use a family if any interface has it
(`IPV4_ENABLED`, `IPV6_ENABLED` in `crassostrea.conf`, set accordingly): Postfix,
unbound and nginx. Loopback keeps both: the services talk to each other over
`127.0.0.1`.

The DNS servers are not used while the gateway resolves names itself
(`MANAGE_RESOLVER=y`, the default).

After **Save**, the root service `crassostrea-network.path`
(`system/libexec/network-apply`) checks the request again against the other
interfaces, applies it, reconfigures the services for the families, and
**tests** that names resolve and that every family with a way out reaches the
internet (1.1.1.1 and 9.9.9.9, or their IPv6 counterparts, by ICMP or TCP): a
family that has a default route, or a gateway or automatic configuration on
some interface. A family only on internal interfaces is not tested, but at
least one family must reach the internet. If a test fails, the previous
settings of the interface come back, and the page shows the outcome with the
log. This takes up to a minute.

!!! warning
    The page warns when you are connected over the family you are about to
    change. If the address you use changes, open the web interface at the new
    address; if a change locks you out anyway, the automatic test restores the
    old settings only when the internet is unreachable, not when your own
    network is.

In a cluster the members' addresses identify them and are not changed here.
Also `GET/PUT /api/v1/ip-config` (`device` chooses the interface; without it,
the one with the default route).

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

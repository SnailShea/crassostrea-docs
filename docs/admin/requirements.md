# Requirements

## Host

- A dedicated host with **EL10**: AlmaLinux 10, Rocky Linux 10 or RHEL 10.
  x86_64 is tested; other architectures may work. EL10 needs an x86-64-v3
  CPU (Intel Haswell, AMD Excavator or newer); in a VM, give it the host's
  CPU model (e.g. "host" in Proxmox or libvirt), not a generic one. The
  [installation ISO](installation.md#installation-iso) installs AlmaLinux 10
  for you.
- **SELinux enforcing** and **firewalld** are supported and recommended.
- Root access (the installer is run with `sudo`).
- **RAM**: **8 GiB recommended**. A gateway with ClamAV uses about 4 GiB; the
  rest is headroom for mail peaks, large messages and the page cache of the
  database. It starts with 2 GiB, and setup warns below 8 GiB; on hosts with
  2.5 GiB or less it offers to disable ClamAV. The
  [witness](high-availability.md#quorum) of a cluster needs far less.
- **Disk**: the quarantine and the mail log are stored in the database. Plan
  for the mail log's retention (default 7 days, at most 31) times your daily
  mail volume, the quarantine's (default 14 days) times your daily rejected
  volume, the nightly backups in `/var/lib/crassostrea/backups/` (7 by
  default, without the mail log's messages unless you include them), plus
  about 1 GiB for ClamAV signatures and Rspamd data.

Use the host for the gateway only. All services share it, and anything that
can connect to the host's loopback interface can reach the internal ports
(they still require their passwords).

## Network

- **Port 25** reachable from the internet (inbound mail). Many cloud
  providers block port 25 by default; ask them to open it.
- **Ports 80 and 443** for the web interface and Let's Encrypt (HTTP-01). With
  DNS-01 validation port 80 is only needed for the redirect to HTTPS.
- **Outgoing** connections to the internet: TCP 25 (outbound mail, recipient
  verification), 53 (DNS, the local resolver queries the root servers; also
  the DNS block lists and the malware hash registry), 80 and 443 (packages,
  signatures, rule and map downloads, Let's Encrypt; Microsoft 365,
  MalwareBazaar and VirusTotal if you use them), 873 (rsync, Sanesecurity
  signatures) and UDP to the public fuzzy hash storage of rspamd.com, which
  Rspamd queries for every message.
  An LDAP directory needs 389/636 (or 3268/3269) to its servers.
- The link endpoint `/l/` of [link protection](protection.md#link-protection),
  like the web interface, must be reachable from wherever your users read
  mail.
- Your downstream mail servers must accept mail from the gateway, and should
  accept mail **only** from it, otherwise spammers bypass the filter.

## DNS

- A hostname for the gateway (e.g. `filter.example.com`) with **A** (and
  **AAAA** if you use IPv6) records.
- A matching **PTR** record (reverse DNS) for its address; many receivers
  reject mail from hosts without one.
- Later, per protected domain: an **MX** record pointing to the gateway, and
  the gateway in the domain's **SPF** record if it relays outbound mail.

!!! warning "DNS blocklists and public resolvers"
    DNS blocklists (Spamhaus and others) refuse queries that come through
    large public resolvers. Crassostrea therefore makes its local unbound the
    system resolver (`MANAGE_RESOLVER=y`). If your host's network is in an
    autonomous system that Spamhaus blocks (common for cloud providers), get a
    free Spamhaus Data Query Service key; setup tells you when this applies.

## Cluster

A cluster survives the failure of a member from three votes on: three full
nodes, or two full nodes and a small witness host (it can start with one node
and grow). The members need fixed IPv4 addresses on which they reach
each other, ideally on a private network, and low latency between them (the
same region). See [High availability](high-availability.md).

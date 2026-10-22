# Installation

## Install

On a fresh EL10 host:

```bash
sudo dnf install -y git
git clone <repository URL> /opt/crassostrea && cd /opt/crassostrea
sudo ./setup.sh
```

`setup.sh` does everything in one run:

1. **Checks the host**: root, EL10, architecture, memory, that no other
   service uses ports 25, 80 or 443, and HTTPS access to the internet.
2. **Installs its own prerequisites** (`curl`, `gawk`, `iproute`) if they are
   missing.
3. **Creates the configuration** `/etc/spamfilter/spamfilter.conf`: it asks
   for the gateway's hostname (the name your MX records will point to),
   generates random passwords and keys, detects IPv6, offers to disable ClamAV
   on small hosts, asks whether to serve this
   [documentation](#documentation-on-the-gateway) on the gateway, and checks
   whether Spamhaus blocks your network.
4. **Checks the hostname**: it warns if the name does not resolve, or resolves
   to an address that is not on this host (expected behind NAT).
5. **Installs** (`system/install.sh`): enables EPEL, CRB and the official
   Rspamd repository, installs the packages, creates the `spamfilter` user and
   the Python environment of the web interface, builds the pinned helper tools,
   sets up SELinux, and runs `system/configure.sh`, which writes the
   configuration of every service, opens the firewall and starts everything.
6. **Prints a summary** with the address of the web interface and the initial
   `admin` password.

Running `setup.sh` again is safe: an existing configuration is kept and the
installation is brought up to date.

### Unattended installation

```bash
sudo ./setup.sh --hostname filter.example.com --yes
```

| Option | Effect |
| --- | --- |
| `--hostname FQDN` | The gateway's hostname instead of asking |
| `--yes` | No questions; needs `--hostname` for a new configuration; disables ClamAV on hosts with 2.5 GiB RAM or less |
| `--config-only` | Stop after writing the configuration, e.g. to edit it before installing; run `setup.sh` again to install |
| `--reconfigure` | Create a new configuration with new passwords even if one exists; the old file is kept as `spamfilter.conf.bak` (copy over the settings you had changed) |
| `--docs` | Also serve the [documentation](#documentation-on-the-gateway), from the `crassostrea-docs` checkout next to the gateway's (`/opt/crassostrea-docs`) |
| `--docs-source SOURCE` | The same, from `SOURCE`: a git URL or a directory of the `crassostrea-docs` repository |
| `--no-docs` | Do not serve it, and do not ask |
| `--ha`, `--ha-*`, `--join` | Clusters, see [High availability](high-availability.md) |

## First steps

1. **Log in** at `https://<hostname>/` as `admin` with the password from the
   summary (it is also `WEB_ADMIN_PASSWORD` in `spamfilter.conf`; it is only
   used while no administrator exists). Change it under your account page
   (click your username).
2. Until Let's Encrypt has issued a certificate, the gateway uses a self-signed
   one and your browser warns. `spamfilter-acme.timer` requests the
   certificate shortly after the installation if the hostname resolves to the
   gateway and port 80 is reachable; `sudo systemctl start spamfilter-acme`
   requests it right away. See [Certificates](certificates.md).
3. **Add a domain** (Domains → Add domain), see
   [Domains and delivery](domains.md).
4. Make sure the downstream mail server accepts mail from the gateway, and
   ideally only from it.
5. **Point the domain's MX record** to the gateway's hostname.
6. Open **Network → DNS & ports**: it checks the hostname's records, each
   domain's MX, DKIM, SPF and DMARC, and the ports, from the server.
7. Send a test message from outside and follow it under **Mail log** and
   the Postfix log (**Monitoring → Logs**).
8. Check the nightly [backups](backups.md) and copy them off the server, and
   connect [monitoring](monitoring.md).

The hostname, the IP addressing and the certificate can later be changed in
the web interface ([Hostname, IP addressing and DNS](network.md),
[Certificates and TLS](certificates.md)).

## Documentation on the gateway

The gateway can serve this documentation at `https://<hostname>/docs/`, to
logged-in administrators only (domain administrators included, not mailbox
users). The menu of the web interface then has a **Documentation** link.
Anyone else is sent to the login page and back to the page they asked for.

It is built from the `crassostrea-docs` repository with MkDocs, in its own
Python environment (`/usr/local/lib/spamfilter/docs-venv`), into
`/usr/local/share/spamfilter/docs`. The build loads no web fonts; pages load
nothing from elsewhere except the scripts of Mermaid diagrams.

**At installation**: answer *yes* to the question of `setup.sh`, or use
`--docs` / `--docs-source SOURCE`. With the default source, clone the docs
next to the gateway:

```bash
git clone <docs repository URL> /opt/crassostrea-docs
git clone <repository URL> /opt/crassostrea && cd /opt/crassostrea
sudo ./setup.sh --docs
```

**Later**, on an installed gateway:

| Task | Command |
| --- | --- |
| Deploy it | `sudo system/docs.sh install [SOURCE] [--ref BRANCH-OR-TAG]` |
| Get new commits of the source and rebuild if they changed it | `sudo system/docs.sh update` |
| Build again from the source as it is | `sudo system/docs.sh rebuild` |
| Stop serving it and remove the site | `sudo system/docs.sh remove` |
| Source, last build, whether it is up to date | `sudo system/docs.sh status` |

`SOURCE` is a git URL (cloned to `/var/lib/spamfilter/docs-src`) or a
directory; without it, `DOCS_SOURCE` or the checkout next to the gateway's is
used. The settings are `DOCS_ENABLED`, `DOCS_SOURCE` and `DOCS_REF` in
`spamfilter.conf` ([Configuration reference](configuration.md#documentation)).
`update.sh` also fetches the documentation's repository (a local checkout is
fast-forwarded as its owner), and `configure.sh` rebuilds the site whenever
its source changed. A failed build only warns: the previous site stays, and
the gateway is not affected.

In a cluster every node has its own setting and site.

## Updating

```bash
cd /opt/crassostrea
sudo ./update.sh
```

`update.sh` fast-forwards the git checkout (as the owner of the checkout) and,
if it is deployed, the documentation's repository, then runs `system/install.sh`: packages, Python packages, pinned tools, the
deployment of the code and the configuration, the database migration, and a
restart of the services. Options: `--skip-git` (no `git pull`), `--force` (no
confirmation).

Updates may add services and timers; `configure.sh` enables them. Most new
filtering features are off until you turn them on. On by default are the
[attachment policy](protection.md#attachment-policy) (a list of dangerous file
types is rejected) and the [impersonation](protection.md#impersonation)
checks for look-alike domains and spoofing of your own domains: review them
after updating from a version without them.

In a cluster, update the members one at a time.

## Uninstalling

There is no uninstaller. Crassostrea uses distribution packages and writes its
configuration into the services' configuration directories; reinstall the host
to remove it completely. To stop it:

```bash
sudo systemctl disable --now 'spamfilter*' postfix rspamd nginx 'clamd@spamfilter'
```

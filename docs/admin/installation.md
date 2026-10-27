# Installation

## Install

On a fresh EL10 host:

```bash
sudo dnf install -y git
git clone https://github.com/SnailShea/crassostrea.git /opt/crassostrea && cd /opt/crassostrea
sudo ./setup.sh
```

`setup.sh` does everything in one run:

1. **Checks the host**: root, EL10, architecture, memory, that no other
   service uses ports 25, 80 or 443, and HTTPS access to the internet.
2. **Installs its own prerequisites** (`curl`, `gawk`, `iproute`) if they are
   missing.
3. **Creates the configuration** `/etc/crassostrea/crassostrea.conf`: it asks
   for the gateway's hostname (the name your MX records will point to),
   generates random passwords and keys, detects IPv6, offers to disable ClamAV
   on small hosts, asks whether to serve this
   [documentation](#documentation-on-the-gateway) on the gateway, and checks
   whether Spamhaus blocks your network.
4. **Checks the hostname**: it warns if the name does not resolve, or resolves
   to an address that is not on this host (expected behind NAT).
5. **Installs** (`system/install.sh`): enables EPEL, CRB and the official
   Rspamd repository, installs the packages, creates the `crassostrea` user and
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
| `--reconfigure` | Create a new configuration with new passwords even if one exists; the old file is kept as `crassostrea.conf.bak` (copy over the settings you had changed) |
| `--docs` | Also serve the [documentation](#documentation-on-the-gateway), from the `crassostrea-docs` checkout next to the gateway's (`/opt/crassostrea-docs`), else from [GitHub](https://github.com/SnailShea/crassostrea-docs) |
| `--docs-source SOURCE` | The same, from `SOURCE`: a git URL or a directory of the `crassostrea-docs` repository |
| `--no-docs` | Do not serve it, and do not ask |
| `--ha`, `--ha-*`, `--join` | Clusters, see [High availability](high-availability.md) |

## Installation ISO

Instead of preparing an EL10 host first, boot the **installation ISO**: it
installs AlmaLinux 10 from the ISO itself and the gateway's source in
`/opt/crassostrea`, and sets the gateway up at the first boot. Get it from the
repository's releases (an ISO is attached to each `v*` tag), or as an artifact
of the **Installation ISO** workflow (Actions → Run workflow), or build it:

```bash
iso/build.sh                     # the checked-out branch, into iso/out/
iso/build.sh --ref v1.4.0 --origin https://github.com/SnailShea/crassostrea.git
```

The build downloads the newest AlmaLinux 10 minimal ISO (checked against
AlmaLinux's signed checksums) and needs only git, curl, gpg and podman or
docker, without root. The installed checkout keeps its git history and pulls
updates from `--origin` (the repository's `origin` by default), so
[`update.sh`](#updating) works as on a host installed by hand.

Write the ISO to a USB stick or attach it to a VM, and boot it (BIOS or
UEFI):

- **Interactive** (default): the installer asks for the disk and the root
  password or an administrator account (and lets you change the network and
  time zone). At the first boot the console offers to set up the gateway now
  (`setup.sh`, which asks for the hostname), to configure the network first
  (`nmtui`; give the gateway a fixed address), or to skip and run
  `/opt/crassostrea/setup.sh` later over SSH.
- **Unattended**: press `e` on the boot entry and add parameters to the
  `linux` line:

  | Parameter | Effect |
  | --- | --- |
  | `crassostrea.hostname=FQDN` | The first boot runs `setup.sh --hostname FQDN --yes` (again at each boot until it succeeds) |
  | `crassostrea.wipe=sda` | **Erases** that disk and partitions it automatically (`first`: the first disk that is not the installation medium) |
  | `crassostrea.sshkey="ssh-ed25519 AAAA..."` | Root logs in with this SSH key only, without a password |
  | `crassostrea.timezone=Europe/Berlin` | The time zone (default UTC) |

  With `wipe` and `sshkey`, nothing is asked and the machine reboots when the
  installation is done.

The first boot needs Internet access, like `setup.sh` (packages, PyPI, virus
signatures); its output is kept in `/var/log/crassostrea-firstboot.log`. The
same hardware requirements apply as for a [manual installation](requirements.md).

## First steps

1. **Log in** at `https://<hostname>/` as `admin` with the password from the
   summary (it is also `WEB_ADMIN_PASSWORD` in `crassostrea.conf`; it is only
   used while no administrator exists). Change it under your account page
   (click your username).
2. Until Let's Encrypt has issued a certificate, the gateway uses a self-signed
   one and your browser warns. `crassostrea-acme.timer` requests the
   certificate shortly after the installation if the hostname resolves to the
   gateway and port 80 is reachable; `sudo systemctl start crassostrea-acme`
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
Python environment (`/usr/local/lib/crassostrea/docs-venv`), into
`/usr/local/share/crassostrea/docs`. The build loads no web fonts; pages load
nothing from elsewhere except the scripts of Mermaid diagrams.

**At installation**: answer *yes* to the question of `setup.sh`, or use
`--docs` / `--docs-source SOURCE`. Without a source, the gateway uses a
`crassostrea-docs` checkout next to its own (`/opt/crassostrea-docs`) if there
is one, else it clones
[crassostrea-docs](https://github.com/SnailShea/crassostrea-docs) from GitHub
and follows it: `update.sh` fetches its new commits.

```bash
git clone https://github.com/SnailShea/crassostrea.git /opt/crassostrea && cd /opt/crassostrea
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

`SOURCE` is a git URL (cloned to `/var/lib/crassostrea/docs-src`) or a
directory; without it, `DOCS_SOURCE` or the checkout next to the gateway's is
used. The settings are `DOCS_ENABLED`, `DOCS_SOURCE` and `DOCS_REF` in
`crassostrea.conf` ([Configuration reference](configuration.md#documentation)).
`update.sh` also fetches the documentation's repository (a local checkout is
fast-forwarded as its owner if it tracks a remote branch; otherwise, or if it
has diverged, it is built as it is), and `configure.sh` rebuilds the site whenever
its source changed. A failed build only warns: the previous site stays, and
the gateway is not affected.

In a cluster every node has its own setting and site.

## Updating

```bash
cd /opt/crassostrea
sudo ./update.sh
```

`update.sh` brings the git checkout up to date (as the owner of the checkout)
and, if it is deployed, the documentation's repository, then runs
`system/install.sh`: packages, Python packages, pinned tools, the deployment of
the code and the configuration, new database tables and columns, and a restart
of the services.

What it updates to is the **update channel** of **Monitoring → Version**
(API: settings `update_channel` and `update_branch`; `update.sh` reads them
from the database):

| Channel | `update.sh` moves the checkout to |
| --- | --- |
| `release` (default) | The newest release: the highest tag `vX.Y.Z` (pre-releases such as `v1.2.0-rc1` are skipped). A checkout that is newer than the newest release (on the branch's later commits) stays where it is: it never goes back to an older version. Before the first release it stays where it is too |
| `main` | The newest commit of a branch (the **Branch** field, default `main`), fast-forward only. For development and testing gateways |

A cluster's members share the channel. A witness has no database and takes
the newest release (or `--channel main` and the branch `main`).

Options: `--channel release|main` (this time, instead of the channel of the
web interface),
`--skip-git` (nothing from the git remote), `--force` (no confirmation). The
version a gateway runs, its deployment time and its channel are on the
**Monitoring** page, with a link to the release notes when it runs a release;
the changes of each release are in the repository's
[CHANGELOG.md](https://github.com/SnailShea/crassostrea/blob/main/CHANGELOG.md).

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
sudo systemctl disable --now 'crassostrea*' postfix rspamd nginx 'clamd@crassostrea'
```

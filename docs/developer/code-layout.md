# Code layout

```
setup.sh                  first installation, cluster creation (--ha) and joining (--join)
update.sh                 git fast-forward, then system/install.sh
iso/                      the installation ISO (see Installation)
  build.sh                stage the source, fetch and verify the AlmaLinux ISO, run assemble.sh
  assemble.sh             kickstart, GRUB entries (BIOS, UEFI, EFI image) and payload into the ISO (xorriso, mtools)
  crassostrea.ks          kickstart: base system, boot parameters (%pre), source and first-boot unit (%post)
  firstboot.sh            first-boot wizard on tty1 (crassostrea-firstboot.service): runs setup.sh
.github/workflows/iso.yml builds the ISO (Run workflow, tags v*), checks iso/ on pull requests
system/
  install.sh              repositories, packages, user, venv, pinned tools, SELinux; calls configure.sh
  configure.sh            deploys the code and renders every service's configuration
  cluster.sh              cluster status, join bundles, membership changes, maintenance, bootstrap, leave
  docs.sh                 deploy, update or remove the documentation at /docs/
  backup.sh               backups: create, list, restore
  lib/common.sh           shared helpers: logging, write_file/create_file, load_env_file, load_config
  lib/cluster.sh          cluster helpers: HA_* parsing and validation, cluster CA, bundles
  lib/conf.sh             conf_set: edit one setting of crassostrea.conf
  lib/tls.sh              oldest TLS versions of nginx and Postfix, the legacy OpenSSL configuration
  lib/ipfamily.sh         IPV4_ENABLED/IPV6_ENABLED for Postfix, unbound and nginx
  lib/nginx.sh            the names nginx answers to (other names and addresses: 404), the web server settings
  lib/spamhaus.sh         the Spamhaus blocklists of Postfix and Rspamd (DQS key or the public mirrors)
  lib/docs.sh             the documentation at /docs/: fetch, build with MkDocs, nginx locations
  libexec/                helpers deployed to /usr/local/libexec/crassostrea
    postfix-control         root helper for mail queue actions (socket-activated)
    log-reader              root helper that reads the journal of fixed units (socket-activated, Logs page)
    postfix-apply           relay networks, message size limit, rate limits, PTR check into Postfix
    set-hostname, set-timezone, tls-apply, network-apply   root services of the web UI (path units)
    nginx-apply, spamhaus-apply   web server settings, Spamhaus DQS key (path units)
    backup                  create, prune and request handling of backups
    dmarc-report            daily DMARC aggregate reports (rspamadm dmarc_report)
    logship                 journald -> Valkey (mail log, netfilter channel; nginx's access log -> netfilter channel), refused recipients and deliveries -> internal API
    acme                    acme.sh wrapper (also --staging-test)
    sa-rules, clamav-sigs   SpamAssassin rules, Sanesecurity signatures
    valkey-role             cluster: Valkey's role at start (and priority 0 in maintenance)
    cluster-node            cluster: probes for the Cluster page, maintenance, Valkey master changes,
                            membership changes, join bundles for joining gateways
    cluster-setup           a single gateway creating or joining a cluster (Cluster page)
  keepalived/             health check for the floating address
  systemd/                units, timers, socket and path units; dropins/ for distribution units
  tmpfiles.d/             runtime directories
data/
  conf/                   service configuration copied by configure.sh
    postfix/                main.cf, master.cf (rendered with additions), postscreen allow list
    rspamd/                 local.d, override.d, custom maps, lua/:
                              rspamd.local.lua        selectors and the checks of the web UI
                              dlp_detectors.lua, impersonation.lua    detectors (also in Python)
                              *.redis.lua             Valkey scripts (outbound and inbound limits, hash lookups)
    clamav/, mysql/, unbound/
  hooks/                  administrator hooks run after rendering a component
  web/                    the web application
    requirements*.txt       Python dependencies (pinned)
    gunicorn.conf.py        gunicorn settings, token-redacting access log
    crassostrea/
      __init__.py             create_app("ui" | "internal")
      config.py               settings from the environment (crassostrea.conf)
      models.py               SQLAlchemy models, settings defaults
      cli.py                  flask commands (see Script reference) and new columns of existing tables
      auth.py                 login, two-factor login, logout, account page, throttling
      internal.py             internal API for Rspamd: settings map, quarantine, mail log, reports, user reports
      forms.py, validators.py WTForms forms; input normalisation shared with the services
      navigation.py           the sidebar's groups and the tabs of their pages (base.html)
      services/               business rules and backends, one module per feature (see Design decisions)
      views/                  admin pages, mailbox user portal, digest links, link protection (/l/),
                              the join endpoint of a cluster (/cluster/join, cluster_join.py),
                              /metrics, render_live, protected_blueprint and the domain scope
      api/                    REST API: __init__ (auth, helpers), accounts, configuration, operations, spec
      templates/              Jinja templates; _*_live.html and */_live.html are live region partials
      static/                 app.css, app.js; swagger-init.js and swagger-theme.css (Swagger UI in
                              the colours of app.css, light and dark)
    tests/                  pytest suite
services/
  netfilter/              ban daemon (main.py, modules/HostNFTables.py)
  olefy/                  Office macro scanner (oletools) for Rspamd
  watchdog/               health checks and notifications
  requirements.txt        Python dependencies of netfilter and olefy
helper-scripts/           administrator tools (reset-admin, reset-learns, SELinux, CAA, postscreen list)
tests/system/             system tests with the real EL10 binaries in namespaces
```

## Where things are installed

| Path | Content |
| --- | --- |
| `/etc/crassostrea/` | `crassostrea.conf`, `units.env`, TLS certificate, DNS-01 credentials, OpenSSL configuration for legacy TLS versions |
| `/etc/pki/crassostrea/` | cluster CA and member certificate |
| `/usr/local/lib/crassostrea/` | deployed web app, netfilter, olefy, the Python venv, `lib/` (shell libraries of the root services) |
| `/usr/local/libexec/crassostrea/` | deployed helpers and the watchdog |
| `/usr/local/share/crassostrea/acme.sh/` | pinned acme.sh |
| `/usr/local/bin/postfix-tlspol` | built from a pinned commit |
| `/etc/systemd/system/crassostrea-*` | units, timers, drop-ins |
| `/var/lib/crassostrea/` | acme state, ACME webroot, rendered cluster templates, configuration hashes; `web/` (requests and settings of the web app: also `web-access`, `spamhaus`, `timezone`), `tls/`, `network/` (outcomes of the root services), `cluster/` (probe and maintenance state of `cluster-node`), `postfix/` (relay networks, message size limit, the maintenance map), `backups/`, `restore-<time>/` |
| `/var/lib/crassostrea/postfix/` | `relay_networks.cidr`, `message_size_limit` (written by the web app) |
| `/etc/rspamd/custom/` | Rspamd maps (static ones from the repository, the rest written by the web app) |

## Naming

The project is called Crassostrea; many technical names still say
`crassostrea` (units, paths, the Python package, the `crassostrea` user and
group, the `SF_` prefix of script variables). They are kept for compatibility
with existing installations.

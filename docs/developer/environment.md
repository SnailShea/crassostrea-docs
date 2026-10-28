# Development environment

Pick the setup that matches what you change:

| You change | Use |
| --- | --- |
| Web interface, API, services, templates | [Local web app](#local-web-app) |
| `configure.sh`, `install.sh`, `data/conf/`, units | [Dry-run rendering](#dry-run-rendering), then a [VM](#a-full-gateway-in-a-vm) |
| Cluster behaviour | [System tests](testing.md#system-tests), then [VMs](#a-cluster-of-vms) |
| netfilter, watchdog, logship | Their tests, then a VM |

## Local web app

Any Linux with Python 3.12 works for the web application. You need a Valkey
(or Redis) server for login throttling, job locks and the logs page; on EL10
or Fedora `dnf install valkey`, elsewhere any Redis-compatible server.

```bash
cd data/web
python3 -m venv .venv
.venv/bin/pip install -r requirements-dev.txt

# a throwaway Valkey without persistence
valkey-server --port 6399 --save "" --daemonize yes

export DATABASE_URL=sqlite:////tmp/crassostrea-dev.db
export WEB_SECRET_KEY=dev-secret WEB_SECURE_COOKIES=n WEB_ADMIN_PASSWORD=admin-password
export RSPAMD_MAP_DIR=/tmp/crassostrea-maps POSTFIX_DYNAMIC_DIR=/tmp/crassostrea-postfix
export REDIS_PORT_INTERNAL=6399
# requests to the root services and their outcome
export HOSTNAME_REQUEST=/tmp/crassostrea-web/hostname TLS_REQUEST=/tmp/crassostrea-web/tls-request.json \
       TLS_UPLOAD_DIR=/tmp/crassostrea-web/tls-upload TLS_STATUS=/tmp/crassostrea-web/tls-status.json \
       IP_REQUEST=/tmp/crassostrea-web/ip-request.json IP_STATUS=/tmp/crassostrea-web/ip-status.json \
       BACKUP_REQUEST=/tmp/crassostrea-web/backup-request.json BACKUP_DIR=/tmp/crassostrea-backups \
       SF_CONF=/tmp/crassostrea-web/crassostrea.conf
mkdir -p "$RSPAMD_MAP_DIR" "$POSTFIX_DYNAMIC_DIR" /tmp/crassostrea-web /tmp/crassostrea-backups

.venv/bin/flask --app crassostrea init-db
.venv/bin/flask --app crassostrea run --debug
```

Open `http://127.0.0.1:5000/` and log in as `admin` / `admin-password`.

What works without the rest of the gateway: everything that only needs the
database and Valkey (domains, lists, accounts, settings, API keys, the API,
the quarantine and mail log with test data, statistics, the rule pages, live
regions). Pages that talk to Rspamd (dashboard counters, training), the
Postfix control socket (mail queue), systemd or NetworkManager (IP
addressing) show a "not reachable" message instead, which is also how they
behave when those services are down. Pages that ask a root service (hostname,
Certificates & TLS, IP addressing, backups) write their request files, and
nothing applies them. DNS & ports, directories, link checks and hash lookups
make real network requests.

Useful settings (environment variables, see `crassostrea/config.py`):

| Variable | Purpose |
| --- | --- |
| `DATABASE_URL` | SQLAlchemy URL; SQLite for development, MariaDB on the gateway (built from `DBUSER`, `DBPASS`, `DBNAME`) |
| `REDIS_HOST`, `REDIS_PORT_INTERNAL`, `REDISPASS` | Valkey |
| `RSPAMD_URL`, `RSPAMD_PASSWORD` | Rspamd controller (`http://127.0.0.1:11334`) |
| `RSPAMD_MAP_DIR`, `POSTFIX_DYNAMIC_DIR` | Where maps are written (on the gateway `/etc/rspamd/custom`, `/var/lib/crassostrea/postfix`) |
| `POSTFIX_CONTROL_SOCKET` | Socket of the queue helper |
| `POSTCONF`, `TLS_POLICY_CHECK_DOMAIN` | The TLS policy lookups check (`services/tlspolicy.py`): Postfix's `postconf` (`/usr/sbin/postconf`; without it the check is left out) and the domain it looks up (`gmail.com`) |
| `HOSTNAME_REQUEST`, `TLS_REQUEST`, `TLS_UPLOAD_DIR`, `TLS_STATUS`, `IP_REQUEST`, `IP_STATUS`, `BACKUP_REQUEST`, `BACKUP_DIR` | Request files of the root services and their outcomes (on the gateway below `/var/lib/crassostrea/`) |
| `SF_CONF` | `crassostrea.conf`, read for the settings the pages show (`services/sysconf.py`) |
| `HA_NODES`, `HA_WITNESS` | Make the app behave like a cluster node (dashboard card, node names, job locks) |

The internal API (what Rspamd fetches) is a second app:

```bash
.venv/bin/flask --app "crassostrea:create_app('internal')" run --port 8001
curl http://127.0.0.1:8001/internal/rspamd/settings
# a DMARC report as Rspamd would hand it over
curl --data-binary @report.eml -H 'Content-Type: message/rfc822' http://127.0.0.1:8001/internal/reports
```

Test data: create domains and users in the UI or with the API, and add
quarantine items from a Python shell (`flask --app crassostrea shell`) with the
`QuarantineItem` model, as the tests do.

## Dry-run rendering

`configure.sh` can render the configuration of every service into a scratch
directory, without root and without changing the system:

```bash
WORK=$(mktemp -d)
export SF_ROOT=$WORK/root SF_CONF=$WORK/root/etc/crassostrea/crassostrea.conf
export SF_DRYRUN=y SF_ASN_RESPONSE=200

# a configuration, created like setup.sh does (it insists on root: a user namespace is enough)
unshare -r ./setup.sh --hostname gw.example.com --yes --config-only

system/configure.sh           # prints the commands it would run as [dry-run]
find "$SF_ROOT/etc" -type f   # the rendered files
```

`SF_ASN_RESPONSE` replaces the Spamhaus ASN check (`200`: not blocked,
`503`: blocked). For a cluster member, add `--ha-nodes ... --ha-witness ...`
to `setup.sh`, or use `tests/system/render.sh`, which renders a whole test
cluster.

Compare the rendering before and after your change, e.g. with
`git stash`, to see exactly what changes on installed gateways. A change that
should not affect single gateways must leave their rendering identical.

You can check rendered files with the real tools: `nginx -t -c ...`,
`haproxy -c -f ...`, `postconf -c <dir>`, `rspamadm configtest`,
`unbound-checkconf`, `systemd-analyze verify`. The system tests show how to run
the binaries without installing them.

## A full gateway in a VM

For changes to installation, configuration rendering, SELinux, units or mail
flow, use a disposable EL10 VM (AlmaLinux 10 or Rocky Linux 10, 2 vCPU, 8 GiB
RAM, 20 GiB disk; 4 GiB works with ClamAV, without headroom):

```bash
sudo dnf install -y git
git clone <your fork> /opt/crassostrea && cd /opt/crassostrea
sudo ./setup.sh --hostname gw.test.example --yes
```

- Without public DNS, set `SKIP_LETS_ENCRYPT=y` (self-signed certificate) or
  use `LE_STAGING=y` with a real name.
- Deploy a change with `sudo system/configure.sh`: it copies the code from the
  checkout and restarts the services.
- Send test mail with `swaks` (EPEL) from another host or the VM itself, e.g.
  `swaks --to user@protected.test --server <vm>`; a GTUBE message
  (`XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X`)
  is always rejected and lands in the quarantine.
- A second VM, or a container with a simple SMTP sink, can play the
  downstream mail server.
- Snapshots make it cheap to test `setup.sh` from scratch.

## A cluster of VMs

Three VMs on one private network (two nodes and a witness, or three nodes):

```bash
# node 1
sudo ./setup.sh --hostname mx1.test.example --yes --ha \
  --ha-nodes mx1.test.example=10.0.0.1,mx2.test.example=10.0.0.2 \
  --ha-witness w.test.example=10.0.0.3 --ha-vip none
# copy /root/crassostrea-join-*.tar to the others, then on each:
sudo ./setup.sh --join /root/crassostrea-join-<hostname>.tar
sudo system/cluster.sh status
```

Test failures by stopping services (`systemctl stop mariadb`, `valkey`),
powering off nodes, and blocking the network between them with
`firewall-cmd`.

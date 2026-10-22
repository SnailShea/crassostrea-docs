# Testing

There are three layers of tests. Run the ones that cover your change before
you submit it.

| Layer | Where | Runs on | Covers |
| --- | --- | --- | --- |
| Unit and integration tests | `data/web/tests/` | any Linux with Python 3.12 | Web app, API, services, maps, internal API, the Postfix control helper, the Valkey scripts of Rspamd |
| System tests | `tests/system/` | EL10 host, no root | Rendered configuration with the real binaries: cluster, nginx, netfilter, `app.js` |
| Manual tests | a VM or VMs | EL10 | Installation, SELinux, units, mail flow, keepalived |

## Web application tests

```bash
cd data/web
python3 -m venv .venv && .venv/bin/pip install -r requirements-dev.txt
.venv/bin/python -m pytest -q
.venv/bin/python -m pytest -q tests/test_api_configuration.py -k dkim   # a subset
```

The suite needs no services: it uses SQLite, fakeredis (with Lua through
`lupa`, for redis-py's locks and the Rspamd scripts), and fakes for Rspamd,
SMTP, DNS, LDAP (ldap3's mock server), Microsoft Graph and systemd. Fixtures in `tests/conftest.py`:

| Fixture | Provides |
| --- | --- |
| `config` | Test configuration, map directories in `tmp_path` |
| `fake_redis` | A fakeredis server, patched into `store.get_redis` |
| `app` | The UI app with an initialised database (`init-db` ran) |
| `client`, `logged_in` | Test clients, the second logged in as `admin` |
| `api` | `api("POST", "/domains", json=...)`: calls the REST API with a read/write key |
| `make_api_key(read_only, allow_from)` | A key for your own client |
| `internal_app` | The internal API app on the same database |

`tests/test_pages.py` has `backends_up` and `backends_down` fixtures that fake
Rspamd, the mail queue and systemd with realistic data or failures, and
renders every page with both.

Guidelines:

- Test behaviour through the UI or the API like a user would, and check the
  effect where it matters (database rows, rendered maps via `_read(config,
  "file.map")`, the settings map via `render_rspamd_settings()`).
- Patch backends at the service module that calls them (e.g.
  `monkeypatch.setattr(postfix_control, "queue_action", ...)`).
- A new API route must be added to `api/spec.py`;
  `test_every_api_route_is_documented` fails otherwise.
- Security rules deserve a test of their own (for example
  `test_personal_entries_are_never_global`).

Other test files cover the live regions (`test_live.py`, fragments and
in-place forms), the cluster features of the app (`test_cluster.py`), the
Postfix control helper and the log reader over a real unix socket
(`test_postfix_control.py`, `test_logs.py`), the decisions of `network-apply`
with several interfaces (`test_network_apply.py`), building a cluster
(`test_cluster_build.py`: the join handshake, and `cluster-setup` against a TLS
server with a pinned certificate) and `cluster-node` applying a membership with
stubbed commands (`test_cluster_node.py`),
and one file per feature (`test_maillog.py`, `test_stats.py`,
`test_domain_admins.py`, `test_directory.py`, `test_links.py`, ...).

Some tests exercise the Rspamd side directly:

- `test_outbound.py` and `test_hashcheck.py` run the Valkey scripts
  (`data/conf/rspamd/lua/*.redis.lua`) in fakeredis, as Rspamd calls them;
- `test_dlp.py` and `test_impersonation.py` run the Lua detectors with
  `rspamadm lua` on the samples the Python detectors are tested with, and are
  skipped where Rspamd is not installed.

Tests that depend on time (two-factor codes, login lifetimes) must not assume
that no step boundary passes between two requests: reuse a code instead of
computing "the next one" again, and keep margins of seconds, not one.

## System tests

`tests/system/` runs what `configure.sh` renders with the real EL10 binaries,
without installing anything and without root. The binaries are downloaded with
`dnf download` and extracted into a work directory; each test runs in an
unprivileged user and network namespace (`unshare -rn`), where the cluster
members get their own loopback addresses (10.0.0.x).

```bash
tests/system/run.sh                 # all
tests/system/run.sh valkey galera   # selected
```

| Test | Checks |
| --- | --- |
| `render` | `setup.sh --ha --config-only`, join bundles and dry-run rendering of two nodes and a witness, and of a one-node cluster |
| `valkey` | TLS-only replication with client certificates, Sentinel discovery and quorum, haproxy routing and pub/sub, failover, the old master returning as a replica, the read-only main configuration; `cluster-node`: its probe, making a node the master, maintenance (421 map, master handed over, priority 0 kept by `valkey-role`) and back |
| `galera` | Bootstrap, the arbitrator, a full state transfer with mariabackup over TLS, multi-master writes, refusal of a node with a foreign certificate, failure and rejoin, loss of quorum |
| `acme` | Two nginx instances answering each other's ACME challenges without loops; 404 for other names and addresses; `/docs/` only after the web UI (a stub) allows it, otherwise the login page; the configuration of a one-node cluster passes `nginx -t` |
| `netfilter` | Bans with real nftables on a single gateway, and shared bans between two nodes |
| `js` | `static/app.js` in jsdom: in-place forms, confirmations, refresh, pausing, errors |

Requirements: an EL10 host with access to the distribution repositories,
unprivileged user namespaces, `nft`, `curl`, `openssl`, `python3` with
`python3-nftables`, and `node`/`npm` for `js`. Work files and logs go to
`$SF_TEST_WORK` (default `${TMPDIR:-/tmp}/crassostrea-system-tests`);
delete it to download and render from scratch. Every test prints `ok`/`FAIL`
lines and a failure count.

Differences from a real cluster, documented in each script: all members run on
one host with their own addresses, files live below the work directory, and the
database servers run as (namespace) root, so the state transfer authenticates
as root instead of the `mysql` system user.

These tests found real bugs that unit tests could not: Galera settings landing
in the client section of `my.cnf`, helper scripts without their execute bit,
Valkey copying its settings into its main file on failover, Sentinel failovers
stalling for minutes, and netfilter wiping the shared ban list when one node
stopped. When you change the cluster code, run them.

## Rendering checks

For changes to `configure.sh` or `data/conf/`, render before and after the
change (see [Dry-run rendering](environment.md#dry-run-rendering)) and compare
the trees. Single gateways must render identically unless the change is meant
for them. Check the rendered files with the services' own tools where
possible (`nginx -t`, `haproxy -c`, `postconf`, `rspamadm configtest`,
`unbound-checkconf`, `keepalived --config-test`, `systemd-analyze verify`).

Shell scripts: `bash -n` at least; `shellcheck -x` where available.

## Rspamd checks

The Lua of the gateway's checks (`rspamd.local.lua`) is not covered by the
web app's tests beyond the scripts and detectors above. After changing it:

1. `rspamadm configtest` against a copy of the configuration (copy
   `/etc/rspamd` to a scratch directory, overlay the repository's
   `data/conf/rspamd/`, and pass `--var=CONFDIR=... --var=LOCAL_CONFDIR=...`);
2. run a throwaway Rspamd on that copy with other ports (override
   `worker-normal.inc`, `worker-controller.inc`, `worker-proxy.inc`; a scratch
   `valkey-server`), raise the action thresholds so test mail is not rejected
   for missing DNS, write the maps the web app would write, and send messages
   to `/checkv2` with `IP`, `From` and `Rcpt` headers. `Flags: milter,body_block`
   returns a rewritten body;
3. for milter behaviour (header actions, body replacement), talk to the proxy
   worker with a small milter client.

Never point such an Rspamd at a real gateway's internal API
(`127.0.0.1:8001`): on a development host that is also a gateway it would
write into the live database.

## Manual tests

Some things need a real host: the installation itself, SELinux in enforcing
mode, firewalld, systemd ordering and sandboxing, keepalived moving the
address, real ACME issuance, and mail flow end to end. Use a VM (see
[Development environment](environment.md#a-full-gateway-in-a-vm)) and check
`ausearch -m AVC -ts recent` after exercising the feature.

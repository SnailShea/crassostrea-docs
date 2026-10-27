# Design decisions

This page explains why things are built the way they are. Read it before
changing the areas it covers.

## Host packages instead of containers

Crassostrea runs on the host with distribution packages, not in containers,
because:

- a mail gateway is a single-purpose host; containers would add a second update
  channel (images) next to the OS and a privileged control path (a container
  API) for the web UI's queue actions and restarts;
- EL10 ships everything needed (Postfix, Valkey, MariaDB with Galera,
  nginx, haproxy, keepalived; Rspamd and ClamAV from their repositories), with
  security updates through `dnf` and SELinux policies;
- systemd provides what an orchestrator would (dependencies, restarts,
  sandboxing, timers, socket activation for the root helpers).

Consequences: services share one host and the loopback interface, so the
internal services still require passwords; the code that runs as root is
deployed root-owned (next section).

## Root-owned deployment

The repository is usually owned by an unprivileged user. If services ran from
it, that user could change code that runs as root. `configure.sh` therefore
copies the web app, netfilter, olefy and the helpers to
`/usr/local/lib/crassostrea` and `/usr/local/libexec/crassostrea` (owned by
root), and the units only reference those copies. `update.sh` runs `git` as the
checkout's owner and everything else as root.

## One configuration file, never sourced

`/etc/crassostrea/crassostrea.conf` is both the scripts' configuration and the
units' `EnvironmentFile`. Scripts read it with `load_env_file`, which parses
`KEY=value` lines the way systemd does (one pair of quotes removed, nothing
expanded or executed) instead of sourcing it, so a value with `$(...)` or
spaces behaves the same in shell and in the units and cannot execute code.
`load_config` validates the required settings and that the secrets that end
up in other configuration formats contain only letters and digits.

## Rendering instead of templating engines

`configure.sh` writes every file with plain `cat` and here-documents through
`write_file` (temporary file, `mv`, `restorecon`), which makes the generated
content visible in one place and needs no extra tools. Rules:

- `write_file` for files the script owns (overwritten every run),
  `create_file` for files the administrator may edit (created once);
- everything below `$SF_ROOT`, and nothing that changes the system when
  `SF_DRYRUN=y`, so the whole rendering runs in tests;
- the script is idempotent and safe to run any time;
- hooks in `data/hooks/<component>/` cover what it does not know.

## Mail queue access for the web UI

The web UI runs unprivileged but must list, show and act on the Postfix queue.
`crassostrea-postfix-control.socket` is a unix socket only the `crassostrea`
group may open; systemd starts a root helper per connection
(`Accept=yes`). The helper reads one JSON request, allows only fixed actions,
validates every queue ID (`^[0-9A-Za-z]{6,32}$`) and passes IDs to `postsuper`
on standard input, never as arguments. Postfix reloads after relay network
changes are done by a path unit watching the file, not by the web UI.

The Logs page reads the journal the same way: `crassostrea-logs.socket` starts
`log-reader` as root per connection. The web UI names a source ("rspamd",
"jobs", ...); the helper maps it to its own list of units and runs
`journalctl` without a shell, with the number of lines and the priority
validated. A compromised web UI can therefore read the logs of the gateway's
services, not the rest of the journal. The Postfix log stays in Valkey
(logship), so it shows every node of a cluster.

## The database is the source of truth

Everything administrators configure lives in MariaDB. Derived state is
rewritten from it:

- **Postfix** reads `domains` and `recipients` with MySQL lookup maps, so no
  reload is needed.
- **Rspamd** needs two kinds of input:
    - *map files* for global data (global lists, protected domains, relay
      networks), written by `maps.sync_all()` only when their content changed
      (an unchanged file keeps its inode, so no reloads are triggered);
    - the *settings map* for per-domain and per-user rules, served by the
      internal API with `Last-Modified` from the `config_updated_at` setting,
      which `touch_config()` increases with every relevant change.
- **DKIM keys** are copied to Valkey, where Rspamd's signing module reads them.

The `sync` job repeats `sync_all()` periodically, so a failed write heals
itself.

## Per-recipient rules in a per-message filter

Rspamd decides once per message, but thresholds and allow lists belong to
recipients. Applying one recipient's rules to a message for several recipients
would let one user's allow list decide for everybody. The settings rules
therefore use custom selectors (`data/conf/rspamd/lua/rspamd.local.lua`):

- `sf_rcpts_owned_by("user@domain")` matches only if **every** envelope
  recipient is the user or one of their `+tag` addresses;
- `sf_rcpts_in_domain("domain")` matches only if **every** recipient is in the
  domain.

Domain **block** rules use a plain `rcpt` condition (one recipient in the
domain is enough). Rule priorities make block beat allow and specific beat
general; global lists are map symbols with ±2050 points and win over all
settings rules. If Rspamd cannot parse a selector, the rule loses its only
condition and never matches; it never silently degrades to "any recipient".

Personal list entries are `list_entries` rows with a `mailbox_user_id` and no
`domain_id`. "Global" therefore means *no domain and no mailbox user*; always
use `filters.global_list_entries()` rather than filtering on `domain_id`
alone. Personal entries once leaked into the global maps exactly that way.

## One service layer for UI and API

Business rules live in `data/web/crassostrea/services/`:

- `accounts.py`: administrators (roles, domains), mailbox users, personal
  lists, thresholds;
- `filters.py`: domains, recipients, global and domain lists, relay networks,
  settings (grouped per page: `SETTINGS`, `BACKUP_SETTINGS`, ...,
  `ALL_SETTINGS` for the API; `SECRET_SETTINGS` are write-only), DKIM keys;
- one module per feature: `maillog.py`, `stats.py`, `reports.py`,
  `user_reports.py`, `journal.py`, `siem.py`, `monitoring.py`, `audit.py`,
  `twofactor.py`, `directory.py`, `outbound.py`, `dlp.py`, `content_rules.py`,
  `impersonation.py`, `links.py`, `hashcheck.py`, `backups.py`,
  `certificates.py`, `hostname.py`, `ipconfig.py`, `netcheck.py`,
  `html_preview.py`;
- `quarantine.py`, `digest.py`, `maps.py`, `postfix_control.py`, `store.py`,
  `sysconf.py` (reads `crassostrea.conf`).

Views and API routes parse input (forms or JSON), call a service function,
commit, and call `sync_all()` when filter configuration changed. Services raise
`ServiceError(message, field, status)`; the UI shows it next to the field, the
API answers with the status and the field name. Services never commit, so a
failed rule leaves nothing behind after the caller's rollback.

## Web UI: server-rendered, enhanced in place

The UI is server-rendered Jinja with a strict CSP (`script-src 'self'`) and
works without JavaScript. `static/app.js` adds:

- **live regions**: a page puts its changing part into a
  `<div data-live data-live-refresh="seconds">` that includes a partial
  template. `views.render_live(page, partial)` renders only the partial when
  the request carries `X-Live-Fragment: 1`; the partial then also shows the
  flashed messages;
- **in-place forms** (`data-async`): the script posts the form with the header;
  the handler's usual redirect back to the page keeps the header, so the
  response is the updated region with its message. POST handlers stay
  unchanged;
- anything that is not a fragment (an expired session redirecting to the
  login page) is shown as a normal page, and a failed request is reported,
  never repeated.

The refresh pauses while entries are selected, while focus is in the region,
and while the tab is hidden.

The menu is data, `navigation.py`: sections of groups, each a list of pages
(tabs) with the endpoints that make them active and whether domain
administrators see them. `base.html` renders the sidebar (one entry per
group, linking to its first visible page) and, on a page of a group with more
than one visible page, the tabs above it. Grouping pages changes only this
list; URLs stay as they are. POST endpoints that render a page on errors are
listed with that page, so the right group stays active.

## REST API

- Authentication by API key only; browser sessions are ignored, so there is
  no CSRF exposure and no CSRF token handling in the API.
- Keys are stored hashed, can be read-only and limited to networks; failed
  attempts count towards login throttling and are reported to netfilter.
- Routes use the service layer; every route must be documented in
  `api/spec.py`, which a test enforces.

## Quarantine and digests

- Released mail gets one prepended header and is otherwise unchanged, so the
  sender's DKIM signature stays valid. The re-injection listener has no milter;
  outbound releases are therefore signed by the web app itself, with the same
  key and the same header list as Rspamd (`dkim.SIGN_HEADERS`).
- Digest links contain a random token (stored hashed) instead of requiring an
  account. GET requests never change anything (mail scanners prefetch links),
  responses carry `Referrer-Policy: no-referrer` and `Cache-Control: no-store`,
  and nginx and gunicorn redact the token from their logs.
- The **HTML view** never puts message HTML into the page: it is sanitized
  with nh3 (no scripts, event handlers, forms, frames or links; images only
  when embedded in the message, as raster `data:` URIs) and served from its
  own URL into a sandboxed iframe whose CSP forbids scripts and remote loads,
  so whatever the sanitizer missed cannot run or load anything either.

## Mail log in the database

Rspamd's own history in Valkey keeps a few thousand entries, without the
message. The mail log stores every scanned message in full in MariaDB (the metadata exporter posts it to the internal
API), so it survives restarts, is shared in a cluster, can be searched,
opened, downloaded and queued again. The price is disk space; retention is
capped at 31 days, and `models.MAX_MESSAGE_MB` sizes Rspamd's `max_message`,
the internal API's request limit and MariaDB's `max_allowed_packet` together.

Statistics are not computed from the mail log: `stats.record()` adds the
message to an hourly counter (an upsert, `ON DUPLICATE KEY UPDATE`, safe with
concurrent workers and Galera) so they outlive the mail log.

## Root services for the web UI

The web UI runs unprivileged, but some pages change the system: the
hostname, the time zone, certificates and TLS versions, IP addressing,
backups, the Postfix settings, the web server's access rules and the
Spamhaus key. They all follow one pattern:

1. the web UI validates the input and writes a **request file** into
   `/var/lib/crassostrea/web/` (owned by `crassostrea`);
2. a **path unit** (`PathExists=`, or `PathChanged=` for the Postfix files,
   the time zone, the web server settings and the Spamhaus key, which stay)
   starts a root service: `set-hostname`, `set-timezone`, `tls-apply`,
   `network-apply`, `backup request`, `postfix-apply`, `nginx-apply`,
   `spamhaus-apply` in `system/libexec/`;
3. the service **consumes** the request first (so the path unit does not loop
   on a bad one), **validates it again** (the web UI is not trusted), applies
   it, and writes its outcome as JSON to a root-owned directory the web UI
   reads (`/var/lib/crassostrea/tls/`, `network/`, `timezone/`, `backups/status.json`);
4. settings the host needs before the database answers go into
   `crassostrea.conf` with `conf_set` (`system/lib/conf.sh`), so later
   `configure.sh` runs keep them. The others stay in the database, and the
   files the web UI writes on every sync (`services/serverconf.py`,
   `timezone.sync`, the Postfix files) are what `configure.sh` reads too:
   the web server settings and the Spamhaus key render through the same
   functions (`system/lib/nginx.sh`, `spamhaus.sh`) in both.

The shell libraries these services share with `configure.sh` (`conf.sh`,
`tls.sh`, `ipfamily.sh`) are deployed to `/usr/local/lib/crassostrea/lib/`, so
both render the same way. Services that test and roll back
(`network-apply`) keep the previous state until the test passed. On a
cluster member the pages refuse: the hostname, addresses and certificates
identify the member.

## Rspamd checks of the web UI

The gateway's own checks are Lua in `rspamd.local.lua`, configured by maps
the web app writes (`maps.build_rspamd_maps`), so they change within a minute
without reloading Rspamd:

- **JSON maps with callbacks** (`dlp_rules.json`, `content_rules.json`,
  `impersonation.json`, `attachment_policy.json`, `link_protection.json`,
  `attachment_analysis.json`, `intake.json`) are parsed once per change, not
  per message. Secrets never go into map files, which every local user can
  read: the hash lookup keys are copied to Valkey like the DKIM keys.
- **Prefilters for verdicts**: Rspamd stops running normal checks once a
  message has reached the reject score. A check whose decision must not depend
  on the spam score (content rules, attachment policy, DLP, outbound limits)
  is a prefilter that sets a pre-result. Checks that only add score
  (impersonation, hash lookups) are normal checks with dependencies (e.g. on
  `DMARC_CHECK`).
- **The direction first**: the quarantine and the mail log file a message as
  inbound or outbound by `RCPT_PROTECTED_DOMAIN` and `RELAY_NETWORK`. The
  multimap rules that set them are normal checks, which a high score skips, so
  the prefilter `SF_MAIL_DIRECTION` sets both before anything else. Without it
  a message rejected early was not quarantined.
- **Valkey scripts in files** (`outbound_limit.redis.lua`,
  `inbound_limit.redis.lua`, `hash_prepare.redis.lua`), loaded with `load_redis_script_from_file`: one
  atomic call per message, and the web app's tests run the same file in
  fakeredis.
- **Detectors in Lua and Python**: the DLP and impersonation detectors exist
  in `dlp_detectors.lua`/`impersonation.lua` for Rspamd and in
  `services/dlp.py`/`impersonation.py` for the test boxes of the UI. Tests
  run both on the same samples (with `rspamadm lua`, skipped where Rspamd is
  not installed); keep them alike.

## Link protection

- **No signing key**: a rewritten link's id is the start of the URL's SHA-256,
  and the URL is stored in Valkey (`sf_link:<id>`, with a TTL) by Rspamd when
  it rewrites. A signed token would need a key readable by Rspamd and the web
  app and turn a leaked key into an open redirector; an id the gateway did not
  store leads nowhere. If Valkey cannot store the links, the message stays
  unchanged.
- **Rewriting through the milter**: Rspamd's `lua_mime` replaces the text
  parts and `task:set_message()` sets the new message. Through the milter only
  the **body** is replaced (`SMFIR_REPLBODY`); header changes (a single-part
  message's new `Content-Type` and `Content-Transfer-Encoding`,
  `X-Link-Protection`) must be milter header actions of their own, or a base64
  body re-encoded as quoted-printable would be corrupted.
- Signed and encrypted messages are left alone (a changed body breaks
  S/MIME and PGP signatures).

## Mail for the gateway itself

`reports@`, `spam@` and `ham@` of the hostname receive reports and user
reports. A Postfix `pipe` to a script would run in a confined SELinux domain
that may not connect to the web app; Rspamd already posts to the internal API.
So Postfix accepts the addresses with `inline:` tables on `$myhostname` in
`main.cf` (expanded by Postfix, so a hostname change needs no other files) and
discards them; an Rspamd prefilter marks such mail "no action" and a
postfilter posts it to `/internal/reports` or `/internal/learn/<kind>` with
the envelope and the client. The web app authenticates user reports itself
(relay network, SPF, DKIM), since the unfiltered mail skipped Rspamd's checks.

## Directories: synchronisation, not lookups

Postfix could query LDAP directly, but every message would then depend on the
directory, Microsoft 365 has no LDAP, and a cluster would query from every
node. Instead `sync-directories` copies the addresses into `recipients`
(`source = "directory"`) every 15 minutes, under a job lock. Postfix keeps
using the MySQL maps. A result that removes more than half of a domain's
directory recipients is refused unless forced: an empty or truncated result
is more often a broken filter, an expired secret or a permission problem.

## Queues for slow receivers

The SIEM export and journaling must not slow down mail or the audit log.
Events and mail log ids are pushed onto Valkey lists (`sf_siem`, capped;
`sf_journal`) in the request that produced them, and long-running senders
(`siem-forward`, `journal-send`, one per node, `BLPOP` on the shared lists)
deliver them, putting them back while the receiver is away. Journal copies go
through the re-injection listener without the milter, so they are not scanned
or logged again.

## Accounts

- **Two-factor login** is TOTP (RFC 6238) implemented in
  `services/twofactor.py`, with the last accepted time step stored against
  replays and recovery codes stored as SHA-256 hashes (they are random, a slow
  hash adds nothing). The login is completed only after the code
  (`2fa_pending` in the session); `require_two_factor` sends administrators to
  the setup while the setting requires it. API keys are unaffected.
- **Domain administrators** are `Admin.role = "domain"` with
  `admin_domains`. `protected_blueprint(..., domain_admins=True)` admits them;
  every view of such a blueprint limits its queries with `scope()` and checks
  single objects with `require_scope()` (404, as if the object did not exist).
  New views in those blueprints must do the same.
- The **audit log** hooks into every request (`after_request`) instead of
  every view: the endpoint, the URL rule with non-sensitive view arguments
  (never the path, which may contain a digest token), the submitted fields
  without secrets (a name pattern: password, secret, token, key, code, ...),
  and the outcome from the status and flashed messages. It writes on its own
  connection, so a rollback of the request keeps the entry, and a failing
  audit log never breaks a request. GET requests are not recorded except
  explicit events (backup downloads).

## Cluster

### Database: Galera

Every node needs fast, local access to the configuration (Postfix lookups for
every message) and the web UI must work on every node. A multi-master Galera
cluster gives each node a full local copy that accepts writes, so no
connection routing is needed and Postfix keeps using the local socket. A node
without quorum refuses queries, which makes Postfix answer 4xx and senders
retry another MX: the safe failure mode for mail.

State transfers use `mariabackup` (the donor stays writable) with `encrypt=4`
(socat with TLS and CA verification); Galera's own traffic uses
`socket.ssl_*`. The donor authenticates as the `mysql` system user through
`unix_socket`, so no SST password exists.

### Valkey: Sentinel behind a local haproxy

Many clients use Valkey: Rspamd, the web app, netfilter, logship, the watchdog
and the shell helpers. Making each of them Sentinel-aware would mean changes in
code we do not control. Instead every node runs haproxy on `127.0.0.1:6379`,
which health-checks all Valkey nodes over TLS (`role:master`) and forwards to
the first master (`balance first`, so never to two). On failover haproxy closes
the connections to the demoted master, so subscribers such as netfilter
reconnect.

Two details matter:

- **`valkey-role`** runs before every Valkey start and asks the Sentinels for
  the current master. A node that was down during a failover therefore comes
  back as a replica instead of a second master with old data.
- **CONFIG REWRITE**: after every role change Sentinel makes Valkey rewrite
  its main configuration file, and Valkey then copies *all* settings,
  including those from included files, into it. Those copies would override
  later changes (password, bind address, the switch back to a single gateway).
  In a cluster Valkey therefore starts from `/etc/crassostrea-valkey.conf`,
  which it cannot write; the rewrite fails harmlessly and `valkey-role` decides
  the role at start.

Sentinel's `failover-timeout` is 10 s: a failover attempt that finds no replica
with fresh state is retried after twice that, instead of after minutes.

### Quorum and membership

Every member has one vote; Galera and Sentinel both need a majority. With two
nodes, a witness (arbitrator `garbd` plus a Sentinel, no data) provides the
third vote. A cluster may have one or two votes while it grows (it then stops
on a failure, like a Proxmox cluster of one or two nodes); `ha_parse` warns.

Membership is configuration (`HA_NODES`, `HA_WITNESS`) on each member and a
row of the shared database (`cluster_membership`), which is the source of
truth: `cluster-report` compares them and asks the root service
`cluster-node` to apply a difference (`configure.sh --no-restart`, then a
haproxy and keepalived reload and Sentinel's quorum). Galera learns members
dynamically, so no database restart is needed. The witness has no database:
it is changed on the command line.

### Building a cluster in the web interface

Modelled on Proxmox: create on one gateway, join the others with a join
information. The information carries the member's cluster address, the
SHA-256 of its HTTPS certificate and a one-time secret (hashed in
`cluster_join_tokens`, one hour). The joining gateway's root service
`cluster-setup` connects to the member's cluster address with the member's
name as SNI and accepts the TLS connection only with that fingerprint: no CA
is involved, so a self-signed certificate or a name that does not resolve
there yet still works, and nobody in between can read the join bundle.
`POST /cluster/join` (no login; the secret authenticates) adds the gateway to
the membership on the first call and answers `202` until the member's
`cluster-node` has applied it (the firewall lets the new node in) and written
the join bundle; then `200` with the bundle, once. The bundle never leaves the
member otherwise, and the web interface, which cannot read the CA key, only
hands out what root wrote into a directory of its group. The joining side then
runs the same `setup.sh --join` as on the command line, so there is one way to
join, whichever starts it; creating runs `setup.sh --ha`.

Members restart MariaDB, Valkey and Sentinel only when their rendered
configuration changed (hashes in `/var/lib/crassostrea/config-hashes`), because
every restart removes a vote and may cause a failover.

### Trust: a private CA and join bundles

`setup.sh --ha` creates a CA on the first node. Full nodes keep the CA key and
issue their own certificate (name and IP address); the witness receives its
certificate in its bundle. A join bundle is a tar file with the configuration
and the CA, created per member, so passwords and keys travel only the way the
administrator chooses. `ha_unbundle` accepts only the expected file names.

### Shared state and single runners

- Jobs that must run once (`send-digests`, `cleanup`, `init-db`) take a lock in
  Valkey (`store.job_lock`); in a cluster a job without Valkey does not run at
  all.
- netfilter on every node bans what the logs of all nodes report (logship
  publishes every node's log lines). `F2B_ACTIVE_BANS` is the shared list: a
  node never clears it on start or stop, restores it on start, and lifts its
  own nftables rule when the entry disappears, whichever node removed it.
- Certificates are per node; nginx forwards ACME challenge requests it cannot
  answer to the other nodes (marked with a header so they never forward
  again), so names shared by several nodes validate.
- keepalived uses `nopreempt`, so the floating address stays where it is when
  a failed node returns.

### Monitoring and maintenance

Each node writes its own row of `cluster_reports` every 10 seconds; rows of
different nodes never conflict in Galera, and the database is where every node
can read every other node's state (Valkey would lose it on a failover, and a
TTL would also lose when a dead node was last seen). What needs the cluster
certificate's key (Valkey's and Sentinel's own view over TLS) or root (the
floating address) is probed by the root service `cluster-node` into a file
the report includes, so the unprivileged web app never gets the key.

What the administrators want of a node is a row of `cluster_control`,
written on any node. The node's own `cluster-report` compares it with what
`cluster-node` reports and writes a request file (the pattern of the other
root services); `cluster-node` keeps the state in effect on every round, so a
restart of Postfix or Valkey does not end maintenance. A node that is down
gets the request when it returns.

Maintenance uses mechanisms each component already has: a `421` from a CIDR
map in `smtpd_client_restrictions` (senders try the next MX), Valkey's
`replica-priority 0` (Sentinel never promotes it) plus a Sentinel failover if
it is the master, and a failing keepalived health check. Making a node the
master sets its priority to 1, waits until Sentinel has read it, and asks for a
failover.

The problems on the Cluster page have stable keys (`node@mx2...`, `valkey`,
...): the alert mail treats each as a check, so a problem that is solved is
reported as solved, and the checks of a node that stopped reporting keep
their state instead of recovering.

### Deliberate limits

- Cluster addresses are IPv4.
- Mail queues stay per node; a failed node's queued mail waits for it.
- `crassostrea.conf` is kept in sync by the administrator, not replicated.

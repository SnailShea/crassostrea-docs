# Script reference

## Entry points

| Script | Usage |
| --- | --- |
| `setup.sh` | `[--hostname FQDN] [--yes] [--reconfigure] [--config-only] [--ha] [--ha-nodes LIST] [--ha-witness H=A\|none] [--ha-vip A/P\|none] [--ha-vip-interface IF] [--join FILE]` |
| `update.sh` | `[--channel release\|main] [--skip-git] [--force]` |
| `system/install.sh` | no options; run by `setup.sh` and `update.sh` |
| `system/configure.sh` | `[--no-restart]`: deploy and render only |
| `system/cluster.sh` | `status`, `bundle HOSTNAME [FILE]`, `members --nodes LIST [--witness H=A \| --no-witness]` (also writes the cluster's membership: the other nodes apply it), `members --standalone`, `maintenance on\|off [NODE] [--force]`, `bootstrap [--force]`, `leave` |
| `system/backup.sh` | `create`, `list`, `restore NAME-OR-PATH [--yes]` |
| `system/docs.sh` | `install [SOURCE] [--ref REF]`, `update`, `rebuild`, `remove`, `status` |
| `iso/build.sh` | `[--ref REF] [--origin URL] [--base-iso PATH] [--out DIR] [--cache DIR] [--engine podman\|docker\|none]`, see [Installation ISO](../admin/installation.md#installation-iso) |

## Helpers in /usr/local/libexec/crassostrea

| Helper | Started by | Does |
| --- | --- | --- |
| `postfix-control` | `crassostrea-postfix-control.socket`, per connection | Mail queue actions of the web UI |
| `log-reader` | `crassostrea-logs.socket`, per connection | The journal of a source (`{"source", "lines", "priority"}`) for Monitoring → Logs; the units of each source are listed in the helper only |
| `postfix-apply` | `crassostrea-postfix-reload.path` | Message size limit, the SMTP rate limits (`rate_limits`: only the four `smtpd_client_*_limit` parameters, whole numbers) and the PTR check (`ptr_check`: `off`, `defer` or `reject` into `sf_ptr_check` and `unknown_client_reject_code`) into `main.cf`, reload for the relay networks, the rate limit exceptions and `client_exempt.cidr` |
| `set-hostname` | `crassostrea-hostname.path` (`/var/lib/crassostrea/web/hostname`) | Hostname change |
| `set-timezone` | `crassostrea-timezone.path` (`web/timezone`, kept) | `timedatectl set-timezone` for a name of the zone database, then `try-restart` of the services that read the zone at start (not MariaDB, Valkey); outcome in `/var/lib/crassostrea/timezone/status.json` |
| `nginx-apply` | `crassostrea-nginx.path` (`web/web-access`, kept) | The web server settings (`http_redirect=`, `rspamd_ui_networks=`, `trusted_proxies=`) into the files nginx includes (`/etc/nginx/crassostrea-https-redirect.conf`, `-rspamd-access.conf`, `-real-ip.conf`, rendered by `system/lib/nginx.sh` like `configure.sh` does), `nginx -t` and reload; the previous files stay if nginx refuses them |
| `spamhaus-apply` | `crassostrea-spamhaus.path` (`web/spamhaus`, kept) | The Spamhaus DQS key (letters and digits) into Postfix's `sf_spamhaus_dnsbl` (which `postscreen_dnsbl_sites` ends with) and `/etc/postfix/dnsbl_reply.map`, and Rspamd's `custom/dqs-rbl.conf` (`system/lib/spamhaus.sh`); without a key the public mirrors if they answer this network; `rspamadm configtest`, then reloads |
| `tls-apply` | `crassostrea-tls.path` (`web/tls-request.json`) | `le-test`, `le-apply`, `manual-apply`, `versions`; outcome in `/var/lib/crassostrea/tls/status.json` |
| `network-apply` | `crassostrea-network.path` (`web/ip-request.json`) | IP addressing of one interface with test and rollback; checks the rule against the other interfaces and sets `IPV4_ENABLED`/`IPV6_ENABLED` from all of them; outcome in `/var/lib/crassostrea/network/status.json` |
| `backup` | `crassostrea-backup.timer` (`create scheduled`, `prune`), `crassostrea-backup-request.path` (`request`), `backup.sh` (`create manual`) | Backups; outcome in `/var/lib/crassostrea/backups/status.json` |
| `dmarc-report` | `crassostrea-dmarc-report.timer` | `rspamadm dmarc_report` if the web UI allows it, under a job lock |
| `acme` | `crassostrea-acme.timer`; `--staging-test NAME...` by `tls-apply` | Certificate |
| `cluster-node` | `crassostrea-cluster.service` (long-running, cluster nodes) | Every 10 s: probes Valkey, Sentinel and the floating address into `/var/lib/crassostrea/cluster/probe.json`, keeps maintenance in effect; applies `web/cluster-request` (`maintenance=on\|off`, `promote=TOKEN`, `members=`/`witness=`: the membership) and `web/cluster-bundle` (membership and `host=`: writes that host's join bundle to `/var/lib/crassostrea/cluster-bundles/`) |
| `cluster-setup` | `crassostrea-cluster-setup.path` (`web/cluster-setup.json`) | A single gateway creating (`setup.sh --ha`) or joining a cluster (join information: pinned HTTPS to the member, then `setup.sh --join`); outcome in `/var/lib/crassostrea/cluster-setup/status.json` |
| `logship`, `sa-rules`, `clamav-sigs`, `valkey-role` | their units | see Code layout |

Every root service of the web UI consumes its request file first and
validates it again; see [Design decisions](design.md#root-services-for-the-web-ui).
| `tests/system/run.sh` | `[render] [valkey] [galera] [keepalived] [acme] [netfilter] [js]` |

## Environment variables of the scripts

| Variable | Used by | Effect |
| --- | --- | --- |
| `SF_CONF` | all | Path of the configuration (default `/etc/crassostrea/crassostrea.conf`) |
| `SF_ROOT` | `configure.sh`, `setup.sh`, cluster helpers | Prefix for every file written; renders into a scratch tree |
| `SF_DRYRUN=y` | `configure.sh` | Do not run commands that change the system (systemctl, mysql, semanage, chown, ...), print them instead |
| `SF_SRC` | `common.sh` | Repository root (default: derived from the script's location) |
| `SF_ASN_RESPONSE` | `configure.sh` in dry runs | Result of the Spamhaus ASN check (`200` or `503`) |
| `SF_GALERA_BOOTSTRAP=y` | `install.sh`, `configure.sh` | Create a new database cluster instead of joining (set by `setup.sh --ha`) |
| `SKIP_CLAMD` | `setup.sh` | Preset the ClamAV question (`y`/`n`) |
| `SF_TEST_WORK` | system tests | Work directory |

## Shell library

`system/lib/common.sh` (sourced by all scripts):

| Function | Purpose |
| --- | --- |
| `log`, `warn`, `die` | Messages on stderr; `die` exits |
| `p PATH` | `PATH` below `$SF_ROOT` |
| `dry`, `run CMD...` | Dry-run test; run a system-changing command unless dry |
| `write_file PATH MODE [OWNER]` | Write stdin atomically (temp file, mv, restorecon) |
| `install_file SRC DEST MODE [OWNER]` | Copy a file with `write_file` |
| `create_file PATH MODE [OWNER]` | Write only if the file does not exist |
| `sync_tree SRC DEST` | Mirror a directory (deploy) |
| `load_env_file FILE`, `load_config` | Read `KEY=value` files safely; read and validate the configuration |
| `require_el10` | Root and EL10 check |
| `selinux_port TYPE PORT` | Label a TCP port |
| `run_hooks COMPONENT` | Run `data/hooks/COMPONENT/*` |

`system/lib/cluster.sh` (sourced by `common.sh`):

| Function | Purpose |
| --- | --- |
| `ha_enabled`, `ha_parse` | Is this a cluster; parse and validate `HA_*` into `HA_NODE_NAMES`, `HA_NODE_ADDRS`, `HA_WITNESS_*`, `HA_ROLE`, `HA_SELF_ADDR` |
| `ha_voters`, `ha_quorum` | Number of votes, majority |
| `ha_member_addrs`, `ha_peer_addrs`, `ha_node_peer_addrs` | Address lists |
| `pki_create_ca`, `pki_issue`, `pki_install_node` | Cluster CA and member certificates |
| `ha_bundle HOST FILE`, `ha_unbundle FILE` | Join bundles |
| `galera_status VAR`, `valkey_tls HOST PORT ARGS` | Query the local Galera state, a member's Valkey or Sentinel |

Also sourced by `common.sh`, and deployed to `/usr/local/lib/crassostrea/lib/`
for the root services:

| Library | Functions |
| --- | --- |
| `conf.sh` | `conf_set KEY VALUE`: change (or add) one setting in `crassostrea.conf` |
| `tls.sh` | `tls_valid_min`, `tls_legacy`, `tls_nginx_protocols`, `tls_postfix_protocols`, `tls_postfix_config_file`, `tls_legacy_cnf`, `tls_nginx_dropin`: `HTTPS_TLS_MIN`/`SMTP_TLS_MIN` for nginx and Postfix |
| `nginx.sh` | `nginx_server_names`, `nginx_names_conf`, `nginx_update_names`: the names nginx answers to (`/etc/nginx/crassostrea-server-names.conf`, included by the servers of the web UI); the root services rewrite it when the hostname or `ADDITIONAL_SAN` change |
| `docs.sh` | `docs_fetch [--update]`, `docs_build [--force]`, `docs_remove`, `docs_nginx_locations`: the documentation at `/docs/` (`DOCS_*`); the site is rebuilt when the hash of its source (`.sf-docs-source` in the site) changes |
| `ipfamily.sh` | `ip_enabled`, `ip_postfix_inet_protocols`, `ip_postfix_address_preference`, `ip_unbound_lines`: `IPV4_ENABLED`/`IPV6_ENABLED` for the services |

## Flask commands

Run as the `crassostrea` user with the gateway's environment, e.g.
`systemctl start crassostrea-job@sync`, or in development with
`flask --app crassostrea <command>`:

| Command | Purpose |
| --- | --- |
| `init-db` | Create tables and new columns (`cli._MIGRATIONS`), defaults, the initial admin, the maps (runs before the web UI starts) |
| `sync` | Re-write the maps and the Valkey copies (DKIM and hash lookup keys) from the database |
| `cleanup` | Expire the quarantine, mail log, audit log, reports, statistics and digest tokens, trim the Postfix log |
| `send-digests` | Send due quarantine digests |
| `provision-accounts` | Create mailbox users for recipients of quarantined mail and mail them a link to set a password (when `mailbox_auto_create` is on) |
| `sync-directories` | Synchronise the recipients of the directories |
| `check-alerts` | Evaluate the alert checks and mail changes |
| `siem-forward` | Send queued events to the SIEM (runs until stopped) |
| `journal-send` | Send queued journal copies (runs until stopped) |
| `cluster-report` | Cluster nodes: report this node's state into `cluster_reports` every 10 s and write requests for `cluster-node` from `cluster_control` (runs until stopped) |
| `create-admin USERNAME` | Create an administrator (asks for the password) |
| `reset-password [USERNAME]` | Reset (or create) an administrator, clear login throttling |
| `reset-2fa USERNAME` | Turn off an administrator's two-factor login |
| `admin-networks [--clear] [--add NETWORK]...` | Show or change the networks administrators may log in from |

The account commands are recorded in the audit log as `cli`.

## Internal API

`crassostrea-web-internal` on `127.0.0.1:8001`, for Rspamd and `logship` only
(loopback or `INTERNAL_CLIENTS`):

| Endpoint | Caller | Content |
| --- | --- | --- |
| `GET /internal/rspamd/settings` | Rspamd settings map | Per-domain and per-user rules |
| `POST /internal/mail-log` | metadata exporter | Every message for a protected domain or from a relay network |
| `POST /internal/quarantine` | metadata exporter | Rejected messages |
| `POST /internal/refused-recipient` | `logship` | A recipient Postfix refused because it does not exist (JSON: `time`, `ip`, `sender`, `rcpt`, `reason`), for the mail log |
| `POST /internal/delivery` | `logship` | A delivery attempt (`kind` delivery: `qid`, `rcpt`, `status`, `dsn`, `relay`, `detail`, `delay` in seconds or null), a notice to the sender (`bounce_notice`, `delay_notice`, `dsn_notice` with `notice_qid`) or an expired message (`expired`), stored in `mail_delivery` |
| `POST /internal/reports` | `SF_INTAKE_EXPORT` | Mail to `reports@<hostname>` |
| `POST /internal/learn/spam`, `/internal/learn/ham` | `SF_INTAKE_EXPORT` | Mail to `spam@`/`ham@<hostname>`, with `X-Envelope-From`, `X-Client-IP`, `X-Relay-Network` |

## Valkey keys

| Key | Owner | Content |
| --- | --- | --- |
| `POSTFIX_MAILLOG` | logship | Mail log entries (JSON) for the web UI |
| `POSTFIX_LOG_LINES` | web app | Lines of `POSTFIX_MAILLOG` logship keeps (Settings) |
| `WATCHDOG_NOTIFY` | web app | Hash of the watchdog's notice settings: `email` (comma separated), `webhook`, `webhook_body`, `subject`, `notify_start`, `notify_ban` (`y`/`n`) |
| `F2B_CHANNEL` | logship, web app | Pub/sub channel netfilter watches |
| `F2B_OPTIONS`, `F2B_REGEX` | netfilter, web app | Ban options and rules (`{"id": {"regex": ..., "attempts": n}}`, `services/store.py`) |
| `F2B_ACTIVE_BANS`, `F2B_PERM_BANS`, `F2B_QUEUE_UNBAN` | netfilter, web app | Bans and unban requests |
| `F2B_WHITELIST`, `F2B_BLACKLIST` | administrator | Never ban, always ban |
| `WEB_LOGIN_FAILURES*` | web app | Login throttling counters |
| `SF_JOB_LOCK:<job>` | web app | Job locks |
| `DKIM_PRIV_KEYS`, `DKIM_SELECTORS` | web app | DKIM keys for Rspamd |
| `ACME_FAIL_TIME` | acme | Last failed renewal, read by the watchdog |
| `sf_rl:<seconds>:<sender>:<window>`, `sf_rl_hit:<sender>`, `sf_rl_hits` | Rspamd, web app | Outbound limit counters and deferred senders |
| `sf_irl:<kind>:<seconds>:<key>:<window>`, `sf_irl_hit:<kind>:<key>`, `sf_irl_hits` | Rspamd, web app | Inbound rate limit counters (kind ip, rcpt, bounce) and the deferred |
| `sf_link:<id>`, `sf_link_verdict:<host>`, `sf_link_clicks` | Rspamd, web app | Link protection: URLs, cached verdicts, clicks |
| `sf_hashcheck_keys`, `sf_hash:<source>:<sha256>`, `sf_vt_rate:<minute>`, `sf_hash_hits` | web app, Rspamd | Hash lookups: API keys, verdicts, VirusTotal rate, detections |
| `sf_siem`, `sf_journal`, `sf_journal_status` | web app | Queues of the SIEM and journal senders |
| `sf_alert_state` | web app | Failing alert checks and when they were mailed |
| `sf_user_reports:<sender>:<hour>` | web app | Reporting limit per user |
| `SF_JOB_LOCK:dmarc-report` | dmarc-report | One sender of DMARC reports per cluster |
| `dmarc_idx*`, `dmarc_rpt*` | Rspamd | DMARC report data (two days) |
| others | Rspamd | Bayes, fuzzy, greylisting, rate limits, history, neural |

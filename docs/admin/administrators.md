# Administrators and audit log

## Administrators

**Users → Administrators** manages the accounts of the web interface. There
are two roles:

| Role | Can |
| --- | --- |
| Administrator | Everything |
| Domain administrator | Only the mail of the domains assigned to them: their quarantine (inbound, and outbound mail sent from their domains), the mail log, their domains' settings, recipients, lists and DKIM keys, their mailbox users, the dashboard's statistics and DMARC/TLS reports |

Domain administrators cannot add or delete domains, and see none of the
system pages (settings, mail queue, logs, monitoring, global lists and filters,
certificates, backups, API keys, other administrators, ...); opening one
leads to the quarantine. They re-queue mail from the mail log only to
recipients in their domains (outbound mail of their domains to any of its
recipients). Messages and items of other domains do not exist for them.

Choose the role and the domains when creating an account, or click an
administrator to change them. Rules:

- Nobody can disable, delete or change the role of their own account.
- At least one active **full** administrator must remain; domain
  administrators do not count.
- `helper-scripts/reset-admin.sh [username]` resets a password from the
  shell (and re-creates the account if needed).

API: `/api/v1/admins` with `role` (`admin` or `domain`) and `domains` (a list
of domain names). API keys always have full access.

## Logins

Logins are throttled: after `WEB_LOGIN_MAX_FAILURES` failures from one
address, or `WEB_LOGIN_MAX_ACCOUNT_FAILURES` failures for one account, within
15 minutes further attempts are refused. Failed logins are also reported to
netfilter, which bans addresses that keep trying.

A login ends 8 hours after it began (`WEB_SESSION_LIFETIME`), however active
you are; live pages do not keep it alive. Closing the browser or logging out
ends it earlier.

### Access by network

**Administrator access from** (Settings → General, Administrators) limits the
networks administrators may use the web interface from: networks or addresses,
one per line (`192.0.2.0/24`, `2001:db8::/48`, `198.51.100.7`); empty allows
any. From another address an administrator login is refused before the
password is checked, with the same answer whether the username exists or not,
and a logged-in administrator whose address changes to one outside the list is
logged out. The documentation at `/docs/` follows the same rule.

- **Mailbox users** log in from anywhere; their logins use an address, which
  administrator usernames cannot contain.
- **API keys** are not affected; each key has its own allowed networks.
- The settings page shows your address and refuses a list without it, so you
  cannot lock yourself out by saving. If you are locked out anyway (a changed
  office address), allow your address or empty the list on the server:

```bash
# the same user and environment as the web interface; --clear allows any address again
sudo systemd-run --quiet --wait --pipe --collect --uid=spamfilter --gid=spamfilter \
  --property=EnvironmentFile=/etc/spamfilter/spamfilter.conf \
  --setenv=PYTHONPATH=/usr/local/lib/spamfilter/web --working-directory=/usr/local/lib/spamfilter/web \
  /usr/local/lib/spamfilter/venv/bin/flask --app spamfilter admin-networks --add 203.0.113.0/24
```

Without options the command shows the list.

The address is the one nginx sees. Behind a load balancer or a reverse proxy,
make sure it passes the client's address (`X-Forwarded-For`), or the list
matches the proxy. The setting is `admin_allowed_networks`.

## Two-factor login

Administrators can protect their login with codes from an authenticator app
(TOTP: Google Authenticator, Microsoft Authenticator, 1Password, ...):

1. Your account page (click your username) → **Two-factor login → Set up**.
2. Scan the QR code (or enter the key), and enter a code to confirm.
3. Store the ten **recovery codes** shown once: each logs in once without the
   app.

From then on the login asks for a code after the password. A code is accepted
once; codes of one 30-second step before or after are accepted (clock drift).
The account page creates new recovery codes, and turns two-factor login off
(with the password and a code).

**Settings → General → Administrators → Require two-factor login** makes it mandatory:
administrators without it must set it up at their next page view. It does not
apply to mailbox users or API keys.

Someone who lost their phone and recovery codes needs another administrator
(**Reset 2FA** on the Administrators page), the API
(`POST /api/v1/admins/{id}/reset-2fa`) or the server:

```bash
# the same user and environment as the web interface (as helper-scripts/reset-admin.sh does)
sudo systemd-run --quiet --wait --pipe --collect --uid=spamfilter --gid=spamfilter \
  --property=EnvironmentFile=/etc/spamfilter/spamfilter.conf \
  --setenv=PYTHONPATH=/usr/local/lib/spamfilter/web --working-directory=/usr/local/lib/spamfilter/web \
  /usr/local/lib/spamfilter/venv/bin/flask --app spamfilter reset-2fa USERNAME
```

## Audit log

**Access → Audit log** records every change made in the web interface or
through the API, every login (failed ones under the name that was tried) and
the account commands of `flask --app spamfilter`:

| Column | Content |
| --- | --- |
| Who | an administrator, `user:<address>` (mailbox user), `link:<address>` (digest link), `api:<key name>` or `cli` |
| IP | the client address |
| Action | what was done (e.g. "Settings: add network", "Quarantine: action (release)") and the request |
| Result | ok or failed, with the message shown to the user |
| Details | the submitted fields, without passwords, keys, codes and tokens |

Reading pages is not recorded, except downloads of backups. Filter by person
and result, or search. Entries cannot be changed or deleted in the interface;
they are kept 365 days by default (**Settings → General → Administrators**, 30 to 3650)
and can be sent to a SIEM ([Monitoring](monitoring.md#siem-export)). API:
`GET /api/v1/audit-log`.

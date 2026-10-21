# Recipient directories

A domain can take its valid recipients from a directory instead of a list you
maintain: an LDAP server (Active Directory, OpenLDAP) or Microsoft 365. Mail to
other addresses of the domain is rejected during the SMTP dialogue.

## How it works

Every 15 minutes (`spamfilter-job-directories.timer`) the gateway reads all
addresses of each directory and copies those of the domains that use it into
their **recipient lists**, marked as coming from the directory. Addresses you
added by hand stay. Postfix only reads the recipient lists, so:

- mail keeps flowing while a directory is down (with the last synchronised
  list);
- a cluster gets the lists through the replicated database; one node
  synchronises at a time.

A synchronisation is **refused**, and the list kept, when the directory returns
no address at all, or when a domain would lose more than half of its directory
recipients (with at least four): that is more often a broken filter or
permission than real. The refusal is shown on the Directories page (and sent
as an [alert](monitoring.md#alert-mail)); **Synchronise, remove anyway** forces
it.

## Setting up

1. **Filter → Directories → Add directory**, LDAP or Microsoft 365 (see
   below).
2. **Test**: reads the directory and shows how many addresses it has per
   domain, with examples, without changing anything.
3. Set the domain's recipient validation to **Only recipients from a
   directory** and choose the directory (Domains → the domain).
4. **Synchronise now**, then check the domain's recipient list.

### LDAP and Active Directory

| Field | Meaning |
| --- | --- |
| Servers | One or more URLs, e.g. `ldaps://dc1.example.com ldaps://dc2.example.com`; the next is tried if one does not answer. Port 636 for `ldaps://`, 389 for `ldap://`; for an Active Directory forest the global catalog (3269/3268) |
| STARTTLS | For `ldap://` URLs |
| Check the server certificate | Against the system's CA certificates. Add an internal CA to `/etc/pki/ca-trust/source/anchors/` and run `update-ca-trust`. Turn off only for a test |
| Bind DN, password | A read-only account; empty for an anonymous bind |
| Base DN | Where to search, e.g. `DC=example,DC=com` |
| Filter | Default `(|(mail=*)(proxyAddresses=*))`. To leave out disabled AD accounts: `(&(|(mail=*)(proxyAddresses=*))(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` |
| Address attributes | Default `mail proxyAddresses`; of `proxyAddresses` only `smtp:` entries count. OpenLDAP: e.g. `mail mailAlternateAddress` |

The search is paged, so large directories work.

### Microsoft 365

Register an application in Microsoft Entra ID, give it the Microsoft Graph
**application** permissions `User.Read.All` and `Group.Read.All`, grant admin
consent, and create a client secret. Enter the tenant ID or domain
(`example.onmicrosoft.com`), the application (client) ID and the secret. The
addresses of all users (shared mailboxes included) and groups are read.

Client secrets expire: renew the secret in Entra ID in time, or the
synchronisation fails (and alerts).

## Commands and API

- `flask --app spamfilter sync-directories` (as the job does:
  `sudo systemctl start spamfilter-job@sync-directories`)
- `/api/v1/directories` with `/test` and `/sync` (`{"force": true}`);
  passwords and secrets can be set but are never returned
- Deleting a directory is refused while a domain uses it; its addresses stay in
  the recipient lists.

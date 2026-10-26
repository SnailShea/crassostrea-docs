# Certificates and TLS

One certificate covers `CRASSOSTREA_HOSTNAME` and the names in
`ADDITIONAL_SAN`. nginx (web interface) and Postfix (STARTTLS) use it from
`/etc/crassostrea/ssl/cert.pem` and `key.pem`.

Until a certificate has been issued, `configure.sh` creates a self-signed one,
so both services start right away.

## In the web interface

**Network → Certificates & TLS** shows the installed certificate (names,
issuer, validity, key) and manages it on a single gateway. A root service
(`crassostrea-tls.path`, `system/libexec/tls-apply`) carries out the changes,
writes them to `crassostrea.conf` (so `configure.sh` keeps them) and records
the outcome, which the page shows with its log. In a cluster each member has
its own certificate and settings: the page only shows them.

### Let's Encrypt names

The hostname is always on the certificate. Additional names (one per line;
wildcards with DNS-01 only) are added in two steps:

1. **Test with staging**: the names are validated against Let's Encrypt's
   staging environment, which has no rate limits and issues untrusted
   certificates. A name that does not point to the gateway, or cannot be
   reached over HTTP, fails here instead of in production.
2. **Apply**: only names that passed a staging test in the last 24 hours can
   be applied. They become `ADDITIONAL_SAN`, Let's Encrypt manages the
   certificate again, and the production certificate is requested right away.

### Your own certificate

**Upload files**: the certificate (optionally followed by its chain), the
private key without a passphrase, and optionally the chain. **Check files**
validates them before anything changes:

- errors (cannot be used): the files are not PEM, the key has a passphrase or
  does not belong to the certificate, an RSA key has fewer than 2048 bits or
  the key is neither RSA nor ECDSA, the certificate is expired or not yet
  valid, the certificates are out of order or one of the chain did not issue
  the one before it;
- warnings (can be used, clients may complain): the certificate expires soon,
  does not name the hostname, or is not trusted by the system's certificate
  authorities (self-signed, a private CA, a missing intermediate).

**Apply certificate** installs it for HTTPS and SMTP and turns off Let's
Encrypt (`SKIP_LETS_ENCRYPT=y`): renew it yourself before it expires. The
[alert mail](monitoring.md#alert-mail) and the watchdog warn in time.

### TLS versions

TLS 1.2 and 1.3 are always accepted. TLS 1.1 and 1.0 can be turned on
separately for HTTPS (the web interface) and inbound SMTP
(`HTTPS_TLS_MIN`, `SMTP_TLS_MIN`), for old clients that cannot use anything
newer. They are insecure. EL10's system crypto policy refuses them in every
program (its LEGACY policy too), so a service that accepts them gets its own
OpenSSL configuration (security level 0, SHA-1 signatures): nginx through
`OPENSSL_CONF`, Postfix through `tls_config_file`. Outbound SMTP keeps TLS
1.2 or newer. Sending mail servers that fail the TLS handshake usually retry
without TLS. SSL 3.0 is not available: OpenSSL 3 no longer includes it.

API: `GET /api/v1/tls` and the `/tls/...` endpoints.

## Let's Encrypt (default)

`crassostrea-acme.timer` runs `crassostrea-acme.service` shortly after boot and
daily. It uses acme.sh (a pinned version installed by `install.sh`) and:

1. checks for every name that it reaches this host over HTTP (a test file in
   the challenge directory); names that do not are skipped, and if the
   hostname itself does not, nothing is requested;
2. requests or renews the certificate (renewal happens when it is due);
3. installs it and reloads nginx and restarts Postfix.

Request it immediately with `sudo systemctl start crassostrea-acme`, and follow
it with `journalctl -u crassostrea-acme`. A failure is reported by the watchdog.
`/usr/local/libexec/crassostrea/acme --staging-test NAME...` runs the staging
test of the web interface from the shell.

### HTTP-01 (default)

Let's Encrypt fetches a token from
`http://<name>/.well-known/acme-challenge/`. Port 80 must be reachable from the
internet and every name must resolve to the gateway (or, in a cluster, to any
of its nodes: nodes answer each other's challenges).

### DNS-01

Needed for wildcard names, or when port 80 is not reachable:

```bash
ACME_DNS_CHALLENGE=y
ACME_DNS_PROVIDER=dns_cf        # an acme.sh DNS API
```

Put the provider's credentials into `/etc/crassostrea/acme/dns-01.conf`
(created with an example), e.g. for Cloudflare:

```bash
CF_Token=...
CF_Zone_ID=...
```

See the acme.sh documentation for the variables of other providers. Run
`system/configure.sh` after changing the configuration.

### Other settings

| Setting | Effect |
| --- | --- |
| `ACME_ACCOUNT_EMAIL` | Registers the account with this contact address |
| `LE_STAGING=y` | Uses the staging environment (untrusted certificates, for tests without rate limits) |
| `SKIP_HTTP_VERIFICATION=y` | Skips the reachability check |

To force a new certificate (e.g. after changing `ADDITIONAL_SAN` in the file),
create `/var/lib/crassostrea/acme/force_renew` and start `crassostrea-acme`.

## Your own certificate without the web interface

Set `SKIP_LETS_ENCRYPT=y`, put the certificate chain and key into
`/etc/crassostrea/ssl/cert.pem` and `key.pem` (key mode 0600), and reload:

```bash
sudo systemctl reload nginx && sudo systemctl restart postfix
```

The watchdog warns before the certificate expires.

## CAA record

`helper-scripts/generate_caa_record.py` prints a CAA record that allows only
your Let's Encrypt account to issue certificates for the name:

```bash
python3 helper-scripts/generate_caa_record.py --account-key \
  /var/lib/crassostrea/acme/ca/acme-v02.api.letsencrypt.org/directory/account.key
```

**DNS & ports** checks that the hostname's CAA records allow Let's Encrypt
([DNS and ports](network.md#dns-and-ports)).

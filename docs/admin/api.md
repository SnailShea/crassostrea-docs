# REST API

Everything the web interface manages, except branding, can be automated with
the REST API at `https://<hostname>/api/v1/`. The interactive documentation
(Swagger UI, OpenAPI 3) is at **API documentation** in the web interface
(`/api/docs/`), in the light or dark appearance of the web interface; the
machine-readable document is at `/api/v1/openapi.json`.

The API applies the same rules as the web interface: both use the same
service layer, so validation and side effects (map updates, DKIM copies,
digest token revocation) are identical.

## Authentication

Create a key under **API keys** and send it in a header:

```bash
curl -H "X-API-Key: sfk_..." https://filter.example.com/api/v1/domains
# or
curl -H "Authorization: Bearer sfk_..." https://filter.example.com/api/v1/domains
```

Read-only keys may only use GET. A key can be limited to source networks.
Browser sessions are not accepted by the API, so it is not exposed to CSRF.
Invalid keys count towards the per-address login throttling and are reported
to netfilter.

## Conventions

- Request and response bodies are JSON (`Content-Type: application/json`),
  except the raw message downloads and the training endpoints
  (`message/rfc822`), the certificate upload (`multipart/form-data`) and the
  backup download (`application/gzip`).
- Lists are paginated: `?page=1&per_page=50` (up to 500), answered as
  `{"items": [...], "total": n, "page": 1, "per_page": 50}`.
- `PATCH` changes only the given fields; `null` resets an override to the
  default (thresholds, digest intervals).
- Unknown fields are rejected.

| Status | Meaning |
| --- | --- |
| 200, 201, 202, 204 | OK, created, accepted (applied shortly), deleted |
| 400 | Malformed request (not JSON, bad query parameter) |
| 401, 403 | Missing or invalid key; read-only key or address not allowed |
| 404 | Not found |
| 409 | Conflict: duplicate, last administrator, virus release without confirmation |
| 422 | Validation failed; `field` names the offending field |
| 429 | Too many failed authentication attempts |
| 502 | A directory could not be read, or its result was refused |
| 503 | A service of the gateway (Postfix, Rspamd, Valkey) failed, or a root service is not installed |

Errors look like `{"error": "This domain already exists", "field": "name"}`.

## Endpoints

| Area | Endpoints |
| --- | --- |
| Domains | `GET/POST /domains`, `GET/PATCH/DELETE /domains/{id}`, `POST /domains/{id}/test-destination` |
| Recipients | `GET/POST /domains/{id}/recipients`, `DELETE /domains/{id}/recipients/{rid}` |
| Domain lists | `GET/POST /domains/{id}/list-entries`, `DELETE /domains/{id}/list-entries/{eid}` |
| DKIM | `GET/PUT/DELETE /domains/{id}/dkim` |
| Global lists | `GET/POST /list-entries`, `DELETE /list-entries/{eid}` |
| Relay networks | `GET/POST /relay-networks`, `DELETE /relay-networks/{id}` |
| Directories | `GET/POST /directories`, `GET/PATCH/DELETE /directories/{id}`, `POST /directories/{id}/test`, `POST /directories/{id}/sync` |
| Outbound limits | `GET/POST /outbound-limits`, `GET/PATCH/DELETE /outbound-limits/{id}`, `GET /outbound-senders`, `POST /outbound-senders/reset` |
| Rate limits | `GET /rate-limits/hits`, `POST /rate-limits/reset` (the limits are settings) |
| Data loss prevention | `GET/POST /dlp-rules`, `GET/PATCH/DELETE /dlp-rules/{id}`, `POST /dlp-rules/test` |
| Content rules | `GET/POST /content-rules`, `GET/PATCH/DELETE /content-rules/{id}`, `POST /content-rules/test` |
| Impersonation | `GET/POST /protected-people`, `GET/PATCH/DELETE /protected-people/{id}`, `POST /impersonation/test` |
| Link protection | `GET /link-clicks`, `POST /link-check` |
| Attachments | `GET /hash-detections`, `POST /hash-lookup` |
| Settings | `GET/PATCH /settings` (the global settings of all pages; API keys of the hash lookups are write-only, shown as `..._set`) |
| Hostname | `GET/PUT /hostname` |
| IP addressing | `GET/PUT /ip-config` (`device`: the interface) |
| Certificates and TLS | `GET /tls`, `POST /tls/letsencrypt/test`, `POST /tls/letsencrypt/apply`, `POST /tls/certificate`, `POST /tls/certificate/apply`, `PUT /tls/versions` |
| Quarantine | `GET /quarantine`, `GET/DELETE /quarantine/{id}`, `GET /quarantine/{id}/raw`, `POST /quarantine/{id}/release` (`allow_sender`), `POST /quarantine/{id}/block-sender`, `POST /quarantine/{id}/learn-spam` |
| Mail log | `GET /mail-log` (filters incl. `action=refused`, `delivery=delivered\|deferred\|bounced`), `GET /mail-log/{id}` (with `deliveries`), `GET /mail-log/{id}/raw`, `POST /mail-log/{id}/requeue` |
| Statistics | `GET /statistics` |
| Reports | `GET /reports/summary`, `GET /reports/dmarc`, `GET /reports/tls` |
| Journaling | `GET /journal/status` |
| Mail queue | `GET /queue`, `GET /queue/{queue_id}`, `POST /queue/actions` |
| Bans | `GET /bans`, `POST /bans/unban` |
| Training | `POST /training/spam`, `POST /training/ham`, `GET /user-reports` |
| Audit log | `GET /audit-log` |
| Backups | `GET/POST /backups`, `DELETE /backups/{name}`, `GET /backups/{name}/download` |
| Monitoring | `GET /monitoring/checks` (metrics for Prometheus: `/metrics`, see [Monitoring](monitoring.md)) |
| Cluster | `GET /cluster`, `PUT /cluster/nodes/{node}/maintenance`, `POST /cluster/nodes/{node}/valkey-master`, `DELETE /cluster/nodes/{node}`, `DELETE /cluster/witness`, `POST /cluster/join-information`; on a single gateway `POST /cluster/create`, `POST /cluster/join` |
| Administrators | `GET/POST /admins`, `GET/PATCH/DELETE /admins/{id}`, `POST /admins/{id}/reset-2fa` |
| Mailbox users | `GET/POST /mailbox-users`, `GET/PATCH/DELETE /mailbox-users/{id}`, `POST /mailbox-users/{id}/password-link` |
| Personal lists | `GET/POST /mailbox-users/{id}/list-entries`, `DELETE /mailbox-users/{id}/list-entries/{eid}` |

The field details are in the Swagger UI.

## Examples

```bash
KEY="X-API-Key: sfk_..."; API=https://filter.example.com/api/v1

# add a domain that only accepts listed recipients
curl -H "$KEY" -H "Content-Type: application/json" \
     -d '{"name": "example.com", "destination_host": "mail.internal", "recipient_check": "list"}' \
     $API/domains
curl -H "$KEY" -H "Content-Type: application/json" \
     -d '{"addresses": ["alice", "bob@example.com"]}' $API/domains/1/recipients

# stricter thresholds for one domain
curl -X PATCH -H "$KEY" -H "Content-Type: application/json" \
     -d '{"add_header_score": 6, "reject_score": 12}' $API/domains/1

# block a sender for everyone
curl -H "$KEY" -H "Content-Type: application/json" \
     -d '{"list_type": "block", "kind": "sender", "value": "*@spam.example"}' $API/list-entries

# release everything quarantined for one recipient
for id in $(curl -s -H "$KEY" "$API/quarantine?rcpt=alice@example.com" | jq '.items[].id'); do
  curl -s -X POST -H "$KEY" $API/quarantine/$id/release > /dev/null
done

# import a user with the password hash of the previous mail system
curl -H "$KEY" -H "Content-Type: application/json" \
     -d '{"email": "bob@example.com", "password_hash": "{BLF-CRYPT}$2y$10$..."}' $API/mailbox-users

# release a message and allow its sender for the recipient's domain
curl -H "$KEY" -H "Content-Type: application/json" -d '{"allow_sender": "domain"}' $API/quarantine/42/release

# generate a DKIM key and print the DNS record
curl -X PUT -H "$KEY" -H "Content-Type: application/json" \
     -d '{"selector": "sf2026", "key_size": 2048}' $API/domains/1/dkim | jq -r '.dns_name, .dns_value'

# learn a message as spam
curl -H "$KEY" -H "Content-Type: message/rfc822" --data-binary @spam.eml $API/training/spam
```

## Notes

- **Quarantine release** delivers the message and removes it from the
  quarantine; the answer is the message as it was, with `notes`. When Postfix
  does not take it (503), it stays. Messages with a virus need
  `{"allow_virus": true}`. The release header names `api:<key name>`.
- **Unban** (`{"network": "203.0.113.0/24"}`) is applied by netfilter within
  seconds. Permanent bans come from netfilter's denylist, see
  [Operations](operations.md#bans).
- **Mail queue and training** act on the node that answers the request. In a
  cluster use a node's own hostname to reach its queue.
- **Cluster**: `GET /cluster` answers `{"cluster": false, ...}` on a single
  gateway, with the progress of creating or joining a cluster (`setup`) and the
  server's IPv4 addresses. Creating and joining answer `202`; the gateway's
  services restart while they run. Maintenance (`{"enabled": true}`) is refused with `409` if no other
  node would accept mail (`"force": true` overrides); the node applies it
  within seconds (`maintenance_applied`).
- **Settings** changes of thresholds and of the filter features are applied by
  Rspamd within a minute.
- **Hostname, IP addressing, certificates, TLS versions and backups** are
  carried out by root services: the request answers `202` and the change
  follows within seconds (IP addressing: up to a minute, with a test); poll
  the `GET` endpoint for the outcome. All of them answer `409` while another
  change runs; hostname, IP addressing and certificates also on cluster
  members, which keep their own.
- **Administrators** have a `role` (`admin` or `domain`) and the `domains` of
  a domain administrator; API keys themselves always have full access.
- **Backup downloads** are recorded in the audit log.

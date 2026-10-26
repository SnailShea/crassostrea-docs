# Crassostrea documentation

The documentation of [Crassostrea](https://github.com/SnailShea/crassostrea), a spam filter gateway for
EL10, built with [MkDocs](https://www.mkdocs.org/) and the Material theme.

- `docs/admin/`: administrator guide (installation, configuration, domains
  and directories, filtering and threat protection, outbound mail,
  quarantine, mail log and statistics, reports, journaling, administrators,
  API, certificates and TLS, network, high availability, operations, backups,
  monitoring)
- `docs/developer/`: developer guide (architecture, design decisions, code
  layout, development environment, testing, contributing)

## On the gateway

The gateway can build and serve this documentation at
`https://<hostname>/docs/` for its administrators: `setup.sh --docs` (from a
checkout of this repository next to the gateway's, e.g.
`/opt/crassostrea-docs`, else from this repository on GitHub),
`setup.sh --docs-source <git URL or directory>`, or
later `system/docs.sh install`. See "Documentation on the gateway" in the
installation guide.

## Build and preview

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/mkdocs serve          # http://127.0.0.1:8000, reloads on changes
.venv/bin/mkdocs build --strict # static site in site/, fails on broken links
```

## Writing

- One topic per page; link instead of repeating.
- Check facts against the code: setting names, defaults, commands and paths
  are taken from the repository and must change with it.
- Write for administrators in the administrator guide (what to do, what
  happens), for contributors in the developer guide (how it works, why).

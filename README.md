# Crassostrea documentation

The documentation of [Crassostrea](../crassostrea), a spam filter gateway for
EL10, built with [MkDocs](https://www.mkdocs.org/) and the Material theme.

- `docs/admin/`: administrator guide (installation, configuration, domains
  and directories, filtering and threat protection, outbound mail,
  quarantine, mail log and statistics, reports, journaling, administrators,
  API, certificates and TLS, network, high availability, operations, backups,
  monitoring)
- `docs/developer/`: developer guide (architecture, design decisions, code
  layout, development environment, testing, contributing)

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

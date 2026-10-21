# Migrating from the Docker version

Earlier versions ran as a Docker Compose stack derived from mailcow:
dockerized, and used mailcow's names: the `mysql-mailcow` container, the
database `mailcow` and the configuration file `mailcow.conf`. Moving to the
current version means a new EL10 host (or a reinstalled one) and copying the
data over.

1. **Dump the database** on the old host:

    ```bash
    docker compose exec mysql-mailcow mariadb-dump -uroot -p"$DBROOT" mailcow > crassostrea.sql
    ```

2. **Create the configuration** on the new host without installing:

    ```bash
    sudo ./setup.sh --hostname filter.example.com --config-only
    ```

    Copy the values of `WEB_SECRET_KEY`, `RSPAMD_PASSWORD` and the other
    settings you had changed from the old `mailcow.conf` into
    `/etc/spamfilter/spamfilter.conf`. The old `MAILCOW_HOSTNAME` becomes
    `CRASSOSTREA_HOSTNAME`. Drop the Docker-only settings. Keep the new
    database and Valkey passwords.

3. **Install** with `sudo ./setup.sh`, then import the dump into the new
   database (`DBNAME`, `crassostrea` by default) and restart the web
   interface:

    ```bash
    sudo mariadb crassostrea < crassostrea.sql
    sudo systemctl restart spamfilter-web
    ```

4. **Optional**: copy the Valkey data (`dump.rdb` of the `redis-vol-1`
   volume) to `/var/lib/valkey/` while `valkey` is stopped, to keep the Bayes
   and fuzzy learning data. DKIM keys are copied to Valkey from the database
   anyway.

5. Move the MX records (or the address) to the new host, and check that the
   downstream servers accept mail from its address.

What the Docker version had and this version does not: see "Differences from
mailcow" in the repository's README.

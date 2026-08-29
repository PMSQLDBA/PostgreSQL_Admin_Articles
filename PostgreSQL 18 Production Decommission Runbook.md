# PostgreSQL 18 Production Decommission Runbook - 29th August, 2026 - Saturday

## Phase 0 — Rules before starting

Do **not** run `pg_dropcluster`, `apt purge`, or `rm -rf` until all of these are true:

* application owners approve the shutdown;
* application writes are stopped;
* replication dependencies are understood;
* every required database is backed up;
* globals are backed up;
* configuration is captured;
* tablespaces are documented;
* backups have been restore-tested;
* SHA-256 checksums pass;
* backups have been copied off this server;
* checksums pass again on the destination.

`pg_dropcluster` is particularly important: Ubuntu documents that it removes the cluster's data, WAL, configuration and tablespace directories. ([Ubuntu Manpages][1])

---

# Phase 1 — Become root and create secure backup directories

```bash
sudo -i

umask 077

install -d -m 0700 -o postgres -g postgres /DBBackups2026
install -d -m 0700 -o postgres -g postgres /DBBackups2026/decommission_info
install -d -m 0700 -o postgres -g postgres /DBBackups2026/config_backup
```

Verify:

```bash
ls -ld \
  /DBBackups2026 \
  /DBBackups2026/decommission_info \
  /DBBackups2026/config_backup
```

Do not continue if this filesystem does not have enough free space.

```bash
df -h /DBBackups2026
df -i /DBBackups2026
```

---

# Phase 2 — Confirm exactly which server you are on

```bash
hostname -f | tee /DBBackups2026/decommission_info/hostname.txt
date -Is | tee /DBBackups2026/decommission_info/decommission_date.txt

cat /etc/os-release \
  > /DBBackups2026/decommission_info/os-release.txt

uname -a \
  > /DBBackups2026/decommission_info/uname.txt
```

Now PostgreSQL:

```bash
pg_lsclusters | tee /DBBackups2026/decommission_info/clusters.txt
```

Expected cluster based on your source material:

```text
18/main
```

Capture service status:

```bash
systemctl status postgresql@18-main --no-pager \
  > /DBBackups2026/decommission_info/postgresql_service.txt
```

Capture versions:

```bash
/usr/lib/postgresql/18/bin/postgres --version \
  | tee /DBBackups2026/decommission_info/postgres_version.txt

/usr/lib/postgresql/18/bin/pg_dump --version \
  | tee /DBBackups2026/decommission_info/pg_dump_version.txt

/usr/lib/postgresql/18/bin/pg_restore --version \
  | tee /DBBackups2026/decommission_info/pg_restore_version.txt

/usr/lib/postgresql/18/bin/psql --version \
  | tee /DBBackups2026/decommission_info/psql_version.txt
```

**STOP if this isn't the intended server/cluster.**

---

# Phase 3 — Capture package inventory

Before uninstalling anything:

```bash
dpkg -l | grep -Ei \
'postgres|libpq|postgis|pgbouncer|pgpool|repmgr' \
> /DBBackups2026/decommission_info/postgresql_components.txt
```

Also:

```bash
dpkg -l | awk '
$1=="ii" &&
(
 $2 ~ /^postgresql/ ||
 $2 ~ /^libpq/ ||
 $2 ~ /^pgbouncer/ ||
 $2 ~ /^pgpool/ ||
 $2 ~ /^repmgr/
)
{print $0}' \
> /DBBackups2026/decommission_info/packages_before_removal.txt
```

Review:

```bash
cat /DBBackups2026/decommission_info/packages_before_removal.txt
```

---

# Phase 4 — Capture PostgreSQL configuration

First capture the normal Debian/Ubuntu configuration:

```bash
cp -a /etc/postgresql/18/main \
  /DBBackups2026/config_backup/

cp -a /etc/postgresql-common \
  /DBBackups2026/config_backup/ 2>/dev/null || true
```

Now capture actual runtime locations:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -Atc "SHOW data_directory;" \
  > /DBBackups2026/decommission_info/data_directory.txt

sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -Atc "SHOW config_file;" \
  > /DBBackups2026/decommission_info/config_file.txt

sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -Atc "SHOW hba_file;" \
  > /DBBackups2026/decommission_info/hba_file.txt

sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -Atc "SHOW ident_file;" \
  > /DBBackups2026/decommission_info/ident_file.txt
```

Capture all configuration source files PostgreSQL actually knows about:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT DISTINCT sourcefile
FROM pg_settings
WHERE sourcefile IS NOT NULL
ORDER BY sourcefile;" \
> /DBBackups2026/decommission_info/config_source_files.txt
```

Capture non-default settings:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT name,
       setting,
       unit,
       source,
       sourcefile
FROM pg_settings
WHERE source <> 'default'
ORDER BY name;" \
> /DBBackups2026/decommission_info/non_default_settings.txt
```

---

# Phase 5 — Capture the actual database inventory

**Do not hard-code the final database list.**

Capture every connectable non-template database:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -Atc "
SELECT datname
FROM pg_database
WHERE datallowconn
  AND NOT datistemplate
ORDER BY datname;" \
> /DBBackups2026/decommission_info/databases.txt
```

Review it:

```bash
cat /DBBackups2026/decommission_info/databases.txt
```

Based on your previous run, you should expect at least your known databases and normally `postgres`. Your previous five user databases were `alpha`, `appdb`, `migrationdb`, `ramdb`, and `testdb`. 

Do **not** proceed if an unexpected database appears until you know what it is.

Capture sizes:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT datname,
       pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
WHERE datallowconn
  AND NOT datistemplate
ORDER BY pg_database_size(datname) DESC;" \
> /DBBackups2026/decommission_info/database_sizes.txt
```

---

# Phase 6 — Capture roles

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -c '\du+' \
> /DBBackups2026/decommission_info/roles.txt
```

---

# Phase 7 — Capture and validate tablespaces

This is **critical before `pg_dropcluster`**.

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT oid,
       spcname,
       pg_tablespace_location(oid) AS location
FROM pg_tablespace
ORDER BY spcname;" \
> /DBBackups2026/decommission_info/tablespaces.txt
```

Review:

```bash
cat /DBBackups2026/decommission_info/tablespaces.txt
```

Also:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -c '\db+'
```

### STOP condition

If any custom tablespace points outside the normal PostgreSQL data directory, identify exactly what owns that directory.

Do not assume an external tablespace directory is safe to delete. `pg_dropcluster` removes cluster tablespace directories as part of deleting the cluster. ([Ubuntu Manpages][1])

---

# Phase 8 — Capture extensions for every database

```bash
while IFS= read -r DB
do
    echo "===== DATABASE: $DB ====="

    sudo -u postgres /usr/lib/postgresql/18/bin/psql \
      -X -d "$DB" -c '\dx'

done < /DBBackups2026/decommission_info/databases.txt \
> /DBBackups2026/decommission_info/extensions.txt
```

Review:

```bash
cat /DBBackups2026/decommission_info/extensions.txt
```

Record any extension whose OS package would be required during restoration.

---

# Phase 9 — Fully check replication

Check whether this server is a standby:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -c "
SELECT pg_is_in_recovery();"
```

Save:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT pg_is_in_recovery();" \
> /DBBackups2026/decommission_info/recovery_status.txt
```

Check whether this server is receiving WAL:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT * FROM pg_stat_wal_receiver;" \
> /DBBackups2026/decommission_info/wal_receiver.txt
```

`pg_stat_wal_receiver` shows information about a WAL receiver connected to its upstream server. ([PostgreSQL][2])

Check downstream replicas:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT * FROM pg_stat_replication;" \
> /DBBackups2026/decommission_info/replication.txt
```

Replication slots:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT slot_name,
       slot_type,
       database,
       active,
       active_pid
FROM pg_replication_slots
ORDER BY slot_name;" \
> /DBBackups2026/decommission_info/replication_slots.txt
```

Logical subscriptions:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT subname,
       subenabled,
       subslotname
FROM pg_subscription;" \
> /DBBackups2026/decommission_info/subscriptions.txt
```

Publications must be checked **inside each database**:

```bash
while IFS= read -r DB
do
    echo "===== DATABASE: $DB ====="

    sudo -u postgres /usr/lib/postgresql/18/bin/psql \
      -X -d "$DB" -P pager=off -c "
SELECT pubname,
       puballtables
FROM pg_publication
ORDER BY pubname;"

done < /DBBackups2026/decommission_info/databases.txt \
> /DBBackups2026/decommission_info/publications.txt
```

### STOP

If you find:

* standby recovery,
* WAL receiver,
* downstream replicas,
* replication slots,
* subscriptions,
* publications,

do not decommission until the replication topology and application impact are approved.

---

# Phase 10 — Check scheduled/background database activity

Check active sessions:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT datname,
       usename,
       client_addr,
       application_name,
       state,
       count(*)
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY datname,
         usename,
         client_addr,
         application_name,
         state
ORDER BY datname, usename;"
```

Also determine whether your environment uses:

* cron jobs,
* systemd timers,
* ETL jobs,
* CI/CD jobs,
* backup agents,
* monitoring that writes,
* `pg_cron`,
* application schedulers.

These are environment-specific, so no generic command can safely certify all of them.

---

# Phase 11 — Start the maintenance window

At this point:

```text
Application owner approval
        ↓
Maintenance window starts
        ↓
Stop application services
        ↓
Stop ETL/batch/scheduled writers
        ↓
Prevent application reconnects
        ↓
Verify connections
        ↓
Final backup
```

Do **not** start final backups while applications are still allowed to write.

PostgreSQL's `pg_dump` creates a consistent dump of an individual database and does not block normal readers/writers. ([PostgreSQL][3]) That's useful normally, but during a final decommission we want the entire environment intentionally frozen.

---

# Phase 12 — Verify application traffic is stopped

Run:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -P pager=off -c "
SELECT pid,
       datname,
       usename,
       client_addr,
       application_name,
       state,
       backend_start,
       xact_start,
       query_start
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
ORDER BY datname, usename;"
```

Investigate every remaining session.

Do not blindly terminate sessions until you know what they are.

Once application owners confirm the environment is frozen, continue.

---

# Phase 13 — Re-capture final database inventory

This protects you from backing up yesterday's database list.

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/psql \
  -X -d postgres -Atc "
SELECT datname
FROM pg_database
WHERE datallowconn
  AND NOT datistemplate
ORDER BY datname;" \
> /DBBackups2026/decommission_info/databases_FINAL.txt
```

Review:

```bash
cat /DBBackups2026/decommission_info/databases_FINAL.txt
```

Compare:

```bash
diff -u \
 /DBBackups2026/decommission_info/databases.txt \
 /DBBackups2026/decommission_info/databases_FINAL.txt
```

A difference is not automatically wrong, but **you must explain it before continuing**.

---

# Phase 14 — Final globals backup

Use the PostgreSQL 18 utility:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/pg_dumpall \
  -h localhost \
  -p 5432 \
  -U postgres \
  --globals-only \
  --file=/DBBackups2026/postgresql_globals_FINAL.sql
```

`pg_dumpall` is the appropriate tool because global objects such as roles and tablespaces are not saved by an individual `pg_dump`. ([PostgreSQL][4])

Check:

```bash
test -s /DBBackups2026/postgresql_globals_FINAL.sql \
  && echo "PASS: globals backup exists and is non-empty" \
  || {
      echo "FAIL: globals backup missing/empty"
      exit 1
  }
```

### Security warning

The globals dump can contain role password information. PostgreSQL provides `--no-role-passwords` specifically when you do not want those values included. ([PostgreSQL][5])

For a true rebuild/decommission package, preserving role credentials may be required by your recovery policy, so protect this file as sensitive.

---

# Phase 15 — Final database backups

Use the final dynamically generated database list.

```bash
sudo -u postgres bash <<'EOF'
set -euo pipefail

BACKUP_DIR="/DBBackups2026"
DBLIST="$BACKUP_DIR/decommission_info/databases_FINAL.txt"

PGDUMP="/usr/lib/postgresql/18/bin/pg_dump"
PGRESTORE="/usr/lib/postgresql/18/bin/pg_restore"

while IFS= read -r DB
do
    [ -n "$DB" ] || continue

    echo
    echo "========================================"
    echo "FINAL BACKUP: $DB"
    echo "========================================"

    "$PGDUMP" \
      -h localhost \
      -p 5432 \
      -U postgres \
      -Fc \
      -v \
      -f "$BACKUP_DIR/${DB}_FINAL.dump" \
      "$DB"

    test -s "$BACKUP_DIR/${DB}_FINAL.dump"

    "$PGRESTORE" \
      -l "$BACKUP_DIR/${DB}_FINAL.dump" \
      > "$BACKUP_DIR/decommission_info/${DB}_archive_contents.txt"

    echo "ARCHIVE READABLE: $DB"

done < "$DBLIST"

echo
echo "ALL FINAL DATABASE DUMPS COMPLETED."
EOF
```

Custom format (`-Fc`) is appropriate and is specifically designed for use with `pg_restore`. ([PostgreSQL][3])

### Important

`pg_restore -l` means:

> archive catalog is readable.

It does **not** mean:

> complete restore proven.

PostgreSQL documents `-l` as listing the archive table of contents. ([PostgreSQL][6])

---

# Phase 16 — Verify every expected dump exists

```bash
while IFS= read -r DB
do
    FILE="/DBBackups2026/${DB}_FINAL.dump"

    if [ ! -s "$FILE" ]; then
        echo "FAIL: Missing/empty backup: $FILE"
        exit 1
    fi

    echo "PASS: $FILE"

done < /DBBackups2026/decommission_info/databases_FINAL.txt
```

Then:

```bash
ls -lh /DBBackups2026/*_FINAL.dump
```

---

# Phase 17 — Perform a REAL restore test

This is the part your original procedure was missing.

The **preferred production method** is to restore these backups onto a separate PostgreSQL 18 validation server.

Why separate?

Because creating test databases on the production server:

* requires additional disk,
* changes the server being decommissioned,
* can interfere with extensions/tablespaces,
* is weaker evidence than independent recovery.

Copy the dumps and globals to the validation PostgreSQL 18 server.

Make sure required extensions/packages exist there first.

For each database:

```bash
createdb -T template0 restore_test_alpha
```

Then:

```bash
pg_restore \
  --exit-on-error \
  --verbose \
  -d restore_test_alpha \
  alpha_FINAL.dump
```

PostgreSQL explicitly supports restoring custom-format archives this way, and `--exit-on-error` makes the restore stop when an SQL error occurs. ([PostgreSQL][6])

Repeat for:

```text
alpha
appdb
migrationdb
ramdb
testdb
postgres
```

**only for databases actually present in `databases_FINAL.txt`.**

Do not assume those exact six names; the file is authoritative.

After each restore, verify at minimum:

```sql
SELECT current_database();

SELECT count(*)
FROM pg_class
WHERE relkind IN ('r','p');

SELECT count(*)
FROM pg_namespace;
```

For critical databases, application-specific validation should also be performed.

### GO/NO-GO

If **any restore fails**, stop.

Do not decommission the source server.

---

# Phase 18 — Generate checksums

Back on the source server:

```bash
cd /DBBackups2026 || exit 1

sha256sum *_FINAL.dump postgresql_globals_FINAL.sql \
  > FINAL_SHA256SUMS.txt
```

Verify immediately:

```bash
sha256sum -c FINAL_SHA256SUMS.txt
```

Every entry must say:

```text
OK
```

If anything says `FAILED`, stop.

---

# Phase 19 — Copy the complete package off-host

Copy:

```text
/DBBackups2026/
```

to your approved backup/storage system.

This includes:

```text
*_FINAL.dump
postgresql_globals_FINAL.sql
FINAL_SHA256SUMS.txt
config_backup/
decommission_info/
```

Do **not** proceed based merely on the copy command reporting success.

---

# Phase 20 — Verify checksums ON THE DESTINATION

On the destination storage/server:

```bash
cd /path/to/copied/DBBackups2026

sha256sum -c FINAL_SHA256SUMS.txt
```

Every database dump and globals file must return:

```text
OK
```

This is a mandatory checkpoint.

---

# Phase 21 — Preserve restore-test evidence

Document:

```text
Database
Backup filename
Backup SHA256
Restore test date
PostgreSQL validation version
Restore result
Validation result
Person/team approving
```

Do not rely only on terminal history.

---

# Phase 22 — Final go/no-go

Before destroying PostgreSQL, explicitly confirm:

```text
[ ] Correct server confirmed
[ ] Correct cluster confirmed: 18/main

[ ] Application maintenance approved
[ ] Application writes stopped
[ ] ETL/batch/schedulers stopped
[ ] Application reconnects controlled
[ ] Remaining connections reviewed

[ ] Final database inventory captured
[ ] Every final database dumped
[ ] postgres database handled if applicable

[ ] Globals backup successful
[ ] Roles captured
[ ] Configuration captured
[ ] Extensions captured

[ ] pg_is_in_recovery checked
[ ] WAL receiver checked
[ ] Downstream replication checked
[ ] Replication slots checked
[ ] Subscriptions checked
[ ] Publications checked

[ ] Tablespace paths reviewed
[ ] External tablespace ownership confirmed

[ ] Every archive readable
[ ] Real restore test successful
[ ] Application/data validation successful

[ ] Source SHA256 verification successful
[ ] Backup copied off-host
[ ] Destination SHA256 verification successful

[ ] Application owner approved destruction
[ ] DBA/infrastructure owner approved destruction
```

### DO NOT CONTINUE

unless every applicable item is satisfied.

---

# Phase 23 — Remove temporary `.pgpass`

Only after the backup and restore validation is finished:

```bash
rm -f /var/lib/postgresql/.pgpass
```

Verify:

```bash
test ! -f /var/lib/postgresql/.pgpass \
  && echo "PASS: .pgpass removed"
```

---

# Phase 24 — Stop PostgreSQL

```bash
systemctl stop postgresql@18-main
systemctl stop postgresql
```

Check:

```bash
pg_lsclusters
```

Check processes:

```bash
ps -ef | grep '[p]ostgres' || echo "PASS: No postgres processes"
```

Check listening sockets:

```bash
ss -lntp | grep -i postgres || echo "PASS: No PostgreSQL listener shown"
```

The `18/main` cluster should show stopped.

Do not proceed if server processes remain unexplained.

---

# Phase 25 — One final backup check BEFORE destruction

On the source:

```bash
cd /DBBackups2026 || exit 1
sha256sum -c FINAL_SHA256SUMS.txt
```

Again:

```text
OK
```

Confirm the independently stored copy is still available.

---

# Phase 26 — Final tablespace review BEFORE `pg_dropcluster`

Read:

```bash
cat /DBBackups2026/decommission_info/tablespaces.txt
```

This is your last chance to catch an external tablespace path.

Remember: `pg_dropcluster` removes the files belonging to the cluster, including tablespace directories. ([Ubuntu Manpages][1])

Only continue after those locations have been explicitly approved for destruction.

---

# Phase 27 — Remove PostgreSQL cluster

This is the **irreversible destructive boundary**.

```bash
pg_dropcluster --stop 18 main
```

Verify:

```bash
pg_lsclusters
```

`18/main` must no longer appear.

Do **not** execute this command until the previous phases are complete.

---

# Phase 28 — Build exact PostgreSQL package-removal list

```bash
dpkg -l | awk '
$1=="ii" && $2 ~ /^postgresql/ {print $2}' \
> /tmp/postgresql_packages_to_remove.txt
```

Review:

```bash
cat /tmp/postgresql_packages_to_remove.txt
```

If this file is unexpectedly empty or contains unexpected packages, stop and investigate.

---

# Phase 29 — Simulate package purge

Do **not** immediately use `-y`.

First:

```bash
xargs -r apt-get -s purge \
  < /tmp/postgresql_packages_to_remove.txt
```

Read the entire proposed removal.

Then simulate autoremove:

```bash
apt-get -s autoremove --purge
```

Make sure APT isn't proposing to remove something required by another application.

---

# Phase 30 — Purge PostgreSQL packages

Only after reviewing the simulation:

```bash
xargs -r apt-get purge -y \
  < /tmp/postgresql_packages_to_remove.txt
```

Then run another simulation:

```bash
apt-get -s autoremove --purge
```

If correct:

```bash
apt-get autoremove -y --purge
apt-get autoclean
```

---

# Phase 31 — Review `libpq` separately

Do **not** blindly remove it.

```bash
dpkg -l | awk '
$1=="ii" && $2 ~ /^libpq/ {print}'
```

Other software may use PostgreSQL client libraries.

Only remove `libpq` if you have verified no other software needs it.

---

# Phase 32 — Review PgBouncer/Pgpool/repmgr separately

```bash
dpkg -l | awk '
$1=="ii" &&
(
 $2 ~ /^pgbouncer/ ||
 $2 ~ /^pgpool/ ||
 $2 ~ /^repmgr/
)
{print}'
```

If any appear, determine whether they:

```text
belonged exclusively to this cluster
```

or:

```text
are used by something else
```

Do not blindly delete them.

---

# Phase 33 — Inspect residual PostgreSQL directories

Do **not** immediately run broad `rm -rf` commands.

First:

```bash
for DIR in \
  /var/lib/postgresql \
  /etc/postgresql \
  /var/log/postgresql \
  /usr/lib/postgresql \
  /usr/lib/pgsql \
  /var/lib/pgsql
do
    echo
    echo "===== $DIR ====="

    if [ -e "$DIR" ]; then
        find "$DIR" -maxdepth 3 -ls
    else
        echo "Not present"
    fi
done
```

Review what remains.

Only remove residual directories after confirming that:

* no installed package owns required files there;
* no other PostgreSQL cluster uses them;
* no application uses them;
* they contain no required backup data.

### NEVER DELETE

```text
/DBBackups2026
```

Your original runbook correctly protected this directory. 

---

# Phase 34 — PGDG repository

Check:

```bash
ls -l /etc/apt/sources.list.d/*pgdg* 2>/dev/null
```

If this machine will **never again need PGDG PostgreSQL packages**, remove the applicable repository configuration.

For example, depending on what actually exists:

```bash
rm -f /etc/apt/sources.list.d/pgdg.list
rm -f /etc/apt/sources.list.d/pgdg.sources
```

Then:

```bash
apt-get update
```

Don't remove unrelated repository files.

---

# Phase 35 — Reload systemd

```bash
systemctl daemon-reload
systemctl reset-failed
```

---

# Phase 36 — Final decommission verification

### Processes

```bash
if pgrep -a postgres; then
    echo "FAIL: PostgreSQL processes still exist"
else
    echo "PASS: No postgres processes"
fi
```

### Clusters

```bash
if command -v pg_lsclusters >/dev/null 2>&1; then
    pg_lsclusters
else
    echo "PASS: pg_lsclusters no longer installed"
fi
```

There must not be an unexpected cluster.

### Services

```bash
systemctl list-unit-files \
  | grep -Ei 'postgres|pgbouncer|pgpool|repmgr' \
  || echo "PASS: No related service unit files found"
```

Investigate anything returned.

### PostgreSQL packages

```bash
dpkg -l | awk '
$1=="ii" &&
(
 $2 ~ /^postgresql/ ||
 $2 ~ /^pgbouncer/ ||
 $2 ~ /^pgpool/ ||
 $2 ~ /^repmgr/
)
{print}'
```

Expected result: nothing, unless you intentionally retained a related component.

### libpq

```bash
dpkg -l | awk '
$1=="ii" && $2 ~ /^libpq/ {print}'
```

`libpq` remaining is **not automatically a failure**.

It may legitimately be required by another application.

### PostgreSQL 18 server binary

Do not rely on `command -v postgres`.

Use:

```bash
if [ -x /usr/lib/postgresql/18/bin/postgres ]; then
    echo "FAIL: PostgreSQL 18 server binary still exists"
else
    echo "PASS: PostgreSQL 18 server binary removed"
fi
```

### psql

```bash
command -v psql || echo "INFO: psql not found"
```

`psql` remaining is not necessarily a failure if PostgreSQL client packages were intentionally retained.

### Port 5432

```bash
ss -lntp | grep ':5432' \
  || echo "PASS: Nothing listening on TCP 5432"
```

If something is listening, investigate it rather than assuming it is PostgreSQL.

---

# Phase 37 — Final backup verification

The local decommission package should remain untouched:

```bash
ls -lah /DBBackups2026
```

Verify:

```bash
cd /DBBackups2026 || exit 1
sha256sum -c FINAL_SHA256SUMS.txt
```

Everything must still return:

```text
OK
```

And separately confirm that the **off-host copy still exists and passes the same checksums**.

---

# Final expected evidence package

You should end with something similar to:

```text
/DBBackups2026/
│
├── alpha_FINAL.dump
├── appdb_FINAL.dump
├── migrationdb_FINAL.dump
├── ramdb_FINAL.dump
├── testdb_FINAL.dump
├── postgres_FINAL.dump             # if postgres was in final inventory
│
├── postgresql_globals_FINAL.sql
├── FINAL_SHA256SUMS.txt
│
├── config_backup/
│   └── main/
│       ├── postgresql.conf
│       ├── pg_hba.conf
│       └── pg_ident.conf
│
└── decommission_info/
    ├── hostname.txt
    ├── decommission_date.txt
    ├── os-release.txt
    ├── clusters.txt
    ├── postgres_version.txt
    ├── pg_dump_version.txt
    ├── pg_restore_version.txt
    ├── databases.txt
    ├── databases_FINAL.txt
    ├── database_sizes.txt
    ├── roles.txt
    ├── tablespaces.txt
    ├── extensions.txt
    ├── recovery_status.txt
    ├── wal_receiver.txt
    ├── replication.txt
    ├── replication_slots.txt
    ├── subscriptions.txt
    ├── publications.txt
    ├── non_default_settings.txt
    ├── config_source_files.txt
    └── packages_before_removal.txt
```

## The exact execution boundary I recommend

Everything through **Phase 26** is preparation, backup, validation, and shutdown work.

**Phase 27 is the point of no return:**

```bash
pg_dropcluster --stop 18 main
```

Do not cross that boundary merely because `pg_dump` completed. PostgreSQL confirms that a custom-format dump is designed for restoration with `pg_restore`, while `pg_restore -l` only lists archive contents. 

A real restore using `pg_restore --exit-on-error` provides substantially stronger recovery evidence. ([PostgreSQL][3])

For your production change, we therefore use one simple rule:

**No successful independent restore test + no verified off-host checksum = no `pg_dropcluster`.**

[1]: https://manpages.ubuntu.com/manpages/noble/man1/pg_dropcluster.1.html "Ubuntu Manpage: pg_dropcluster - completely delete a PostgreSQL cluster"
[2]: https://www.postgresql.org/docs/18/monitoring-stats.html "PostgreSQL: Documentation: 18: 27.2. The Cumulative Statistics System"
[3]: https://www.postgresql.org/docs/18/app-pgdump.html "PostgreSQL: Documentation: 18: pg_dump"
[4]: https://www.postgresql.org/docs/18/app-pg-dumpall.html "PostgreSQL: Documentation: 18: pg_dumpall"
[5]: https://www.postgresql.org/docs/current/app-pg-dumpall.html "PostgreSQL: Documentation: 18: pg_dumpall"
[6]: https://www.postgresql.org/docs/current/app-pgrestore.html "PostgreSQL: Documentation: 18: pg_restore"

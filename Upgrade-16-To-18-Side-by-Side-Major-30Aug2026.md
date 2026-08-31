**Ubuntu 26.04**: **side-by-side PostgreSQL 16 → PostgreSQL 18 major-version upgrade**. 

For Ubuntu package-managed clusters: 

Recommended using Ubuntu/Debian's `pg_upgradecluster --method=upgrade`, which uses PostgreSQL `pg_upgrade` underneath (Ubuntu's split configuration/data-directory layout). ([Ubuntu Manpages][1])

# PostgreSQL 16 to PostgreSQL 18 Side-by-Side Major Upgrade

## Technical Standard Operating Procedure — Ubuntu 26.04

**Source Database:** PostgreSQL 16.x
**Target Database:** PostgreSQL 18.x
**Operating System:** Ubuntu 26.04 LTS
**Upgrade Type:** Major Version Upgrade
**Upgrade Method:** Side-by-Side using `pg_upgrade` through `pg_upgradecluster --method=upgrade`
**Recommended Environment:** Production / Enterprise PostgreSQL Server

---

# 1. Purpose

This SOP provides a controlled procedure for upgrading a PostgreSQL 16 database cluster to PostgreSQL 18 on Ubuntu 26.04.

The objective is to:

* Keep PostgreSQL 16 available during preparation.
* Install PostgreSQL 18 alongside PostgreSQL 16.
* Perform extensive compatibility checks before downtime.
* Use PostgreSQL's `pg_upgrade` technology instead of dump/restore.
* Minimize production downtime.
* Preserve the PostgreSQL 16 cluster until PostgreSQL 18 has been fully validated.
* Provide a clear rollback path.

PostgreSQL major releases cannot be upgraded by simply installing newer binaries over the existing server. A major upgrade requires a supported migration method such as:

* `pg_dump` / `pg_restore`
* `pg_upgrade`
* Logical replication

For this SOP, `pg_upgrade` is used.

PostgreSQL 18 `pg_upgrade` supports upgrades from PostgreSQL 9.2 and later, so PostgreSQL 16 can be upgraded directly to PostgreSQL 18.

There is no requirement to perform:

```text
PostgreSQL 16
     ↓
PostgreSQL 17
     ↓
PostgreSQL 18
```

The supported path is:

```text
PostgreSQL 16
     ↓
PostgreSQL 18
```

However, the PostgreSQL 17 and PostgreSQL 18 migration/release notes should both be reviewed because compatibility changes introduced in PostgreSQL 17 also apply when PostgreSQL 17 is skipped.

---

# 2. Upgrade Architecture

## Before Upgrade

```text
Application
     |
     | TCP 5432
     v
PostgreSQL 16
Cluster: 16/main

Data:
/var/lib/postgresql/16/main

Configuration:
/etc/postgresql/16/main
```

## During Preparation

```text
                     Ubuntu 26.04
                          |
             +------------+------------+
             |                         |
      PostgreSQL 16               PostgreSQL 18
        Source                      Target
             |                         |
       16/main                     18/main
             |
        Port 5432

PG16 Data:
 /var/lib/postgresql/16/main

PG16 Config:
 /etc/postgresql/16/main

PG18 Data:
 /var/lib/postgresql/18/main

PG18 Config:
 /etc/postgresql/18/main
```

## After Successful Upgrade

```text
Application
     |
     | TCP 5432
     v
PostgreSQL 18
Cluster: 18/main

PostgreSQL 16
Retained temporarily
for rollback
```

Ubuntu's PostgreSQL cluster-management tools keep the old cluster after upgrade instead of automatically deleting it. This is desirable for controlled production validation.

---

# 3. Important Ubuntu Architecture Difference

On Ubuntu/Debian PostgreSQL installations, configuration files and database files are normally separated.

PostgreSQL 16:

```text
Configuration:
/etc/postgresql/16/main

Data:
/var/lib/postgresql/16/main
```

PostgreSQL 18:

```text
Configuration:
/etc/postgresql/18/main

Data:
/var/lib/postgresql/18/main
```

This is different from many source-built PostgreSQL installations where configuration files reside directly inside `PGDATA`.

Therefore, blindly copying generic `pg_upgrade` commands from another Linux distribution is not recommended on Ubuntu.

Ubuntu provides:

```bash
pg_upgradecluster
```

which integrates with Ubuntu's PostgreSQL cluster-management framework.

For a fast PostgreSQL major upgrade, explicitly use:

```text
--method=upgrade
```

Otherwise, `pg_upgradecluster` can use a dump-based upgrade method depending on its version/default behavior.

---

# 4. Major Upgrade Phases

The production upgrade is divided into the following phases:

```text
Phase 1  - Planning
Phase 2  - PostgreSQL 16 Prechecks
Phase 3  - Backup and Recovery Validation
Phase 4  - PostgreSQL 18 Installation
Phase 5  - Extension Compatibility
Phase 6  - Side-by-Side Preparation
Phase 7  - Upgrade Compatibility Testing
Phase 8  - Maintenance Window
Phase 9  - Major Upgrade
Phase 10 - PostgreSQL 18 Configuration Review
Phase 11 - Post-Upgrade Tasks
Phase 12 - Application Validation
Phase 13 - Monitoring and Performance Validation
Phase 14 - Backup and HA Validation
Phase 15 - Rollback / Decommission
```

---

# 5. Phase 1 — Planning and Prerequisites

Before touching production, confirm:

```text
[ ] PostgreSQL 16 is healthy
[ ] PostgreSQL 18 packages are available
[ ] PostgreSQL 18 supports every required extension
[ ] Full backup is available
[ ] Restore capability has been tested
[ ] Adequate filesystem space exists
[ ] Application owners approved downtime
[ ] DBA rollback plan exists
[ ] Monitoring maintenance window is scheduled
[ ] Backup jobs are accounted for
[ ] Replication architecture is documented
[ ] PgBouncer/proxies are documented
[ ] Cron/batch/ETL jobs are documented
```

Do not begin a production major upgrade based only on `pg_upgrade --check`.

The entire platform should be reviewed.

---

# 6. Verify Ubuntu Version

Run:

```bash
lsb_release -a
```

or:

```bash
cat /etc/os-release
```

Confirm Ubuntu 26.04.

Example:

```text
VERSION_ID="26.04"
```

---

# 7. Verify PostgreSQL 16

Run:

```bash
sudo -u postgres psql -c "SELECT version();"
```

Also:

```bash
/usr/lib/postgresql/16/bin/postgres --version
```

Check server version:

```bash
sudo -u postgres psql -c "SHOW server_version;"
```

Record the results.

---

# 8. Check Existing PostgreSQL Clusters

Ubuntu provides:

```bash
pg_lsclusters
```

Example:

```text
Ver Cluster Port Status Owner    Data directory
16  main    5432 online postgres /var/lib/postgresql/16/main
```

Confirm that you are upgrading the correct cluster.

Never assume that the cluster is named `main`.

If the environment contains:

```text
16/main
16/prod
16/reporting
```

each cluster requires separate planning.

---

# 9. Verify Current Data Directory

Run:

```bash
sudo -u postgres psql -c "SHOW data_directory;"
```

Expected example:

```text
/var/lib/postgresql/16/main
```

---

# 10. Verify Configuration Locations

Run:

```bash
sudo -u postgres psql -c "SHOW config_file;"
sudo -u postgres psql -c "SHOW hba_file;"
sudo -u postgres psql -c "SHOW ident_file;"
```

Typical Ubuntu result:

```text
/etc/postgresql/16/main/postgresql.conf
/etc/postgresql/16/main/pg_hba.conf
/etc/postgresql/16/main/pg_ident.conf
```

Document these locations.

---

# 11. Capture Database Inventory

Run:

```bash
sudo -u postgres psql -c "\l+"
```

Capture database sizes:

```bash
sudo -u postgres psql -c "
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS database_size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
"
```

Save this information for post-upgrade comparison.

---

# 12. Capture Role Inventory

Run:

```bash
sudo -u postgres psql -c "\du+"
```

Also create a reference copy of global objects:

```bash
sudo -u postgres pg_dumpall \
--globals-only \
> /DBBackups2026/pg16_globals_before_upgrade.sql
```

This is an additional recovery/reference artifact.

---

# 13. Check Tablespaces

Run:

```bash
sudo -u postgres psql -c "
SELECT
    spcname,
    pg_tablespace_location(oid)
FROM pg_tablespace
ORDER BY spcname;
"
```

Example:

```text
pg_default
pg_global
app_data_ts   /PGDATA/app_data
app_index_ts  /PGINDEX/app_index
```

Document all custom tablespaces.

Check ownership and permissions at the OS level.

Example:

```bash
ls -ld /PGDATA
ls -ld /PGINDEX
```

---

# 14. Check Extensions

Run in every relevant database:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY extname;
```

Common examples:

```text
pg_stat_statements
pgcrypto
uuid-ossp
pgaudit
PostGIS
TimescaleDB
pg_partman
```

External modules must have PostgreSQL 18-compatible binaries installed before the major upgrade.

`pg_upgrade` can verify many PostgreSQL compatibility conditions, but it cannot guarantee binary compatibility of third-party extensions.

---

# 15. Check Shared Preload Libraries

Run:

```bash
sudo -u postgres psql -c "SHOW shared_preload_libraries;"
```

Example:

```text
pg_stat_statements,pgaudit
```

Every required PostgreSQL 18 library must be installed before starting PostgreSQL 18 with that configuration.

Do not copy PostgreSQL 16 `.so` binary libraries into the PostgreSQL 18 library directory.

Install PostgreSQL 18-compatible packages instead.

---

# 16. Check PostgreSQL Data Checksums

This is especially important for PostgreSQL 18.

Run:

```bash
sudo -u postgres psql -c "SHOW data_checksums;"
```

Possible result:

```text
on
```

or:

```text
off
```

PostgreSQL 18 changed `initdb` so newly initialized clusters have data checksums enabled by default.

`pg_upgrade` requires compatible checksum settings between the source and destination clusters.

Therefore:

```text
PG16 checksums ON
        ↓
PG18 checksums ON
```

or:

```text
PG16 checksums OFF
        ↓
PG18 checksums OFF
```

A mismatch must be corrected before upgrade.

PostgreSQL documents `--no-data-checksums` specifically for initializing PostgreSQL 18 clusters when upgrading an older non-checksum cluster.

---

# 17. Check Locale and Encoding

Run:

```bash
sudo -u postgres psql -c "
SELECT
    datname,
    pg_encoding_to_char(encoding) AS encoding,
    datcollate,
    datctype
FROM pg_database
ORDER BY datname;
"
```

Also examine:

```bash
sudo -u postgres \
/usr/lib/postgresql/16/bin/pg_controldata \
/var/lib/postgresql/16/main
```

Do not casually change locale, encoding or cluster-level compatibility characteristics during a major upgrade.

Major upgrade and locale migration should generally be treated as separate changes unless specifically tested.

---

# 18. Check Collation Versions

Run:

```sql
SELECT
    datname,
    datcollversion
FROM pg_database
ORDER BY datname;
```

Also check collations where applicable:

```sql
SELECT
    collname,
    collprovider,
    collversion
FROM pg_collation
WHERE collversion IS NOT NULL;
```

OS/library upgrades can change collation implementations.

Any collation-version warnings seen after upgrade must be investigated before production acceptance.

Indexes dependent on changed collations can require rebuilding.

---

# 19. Check Replication

Physical replication:

```bash
sudo -u postgres psql -c "
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
"
```

Replication slots:

```bash
sudo -u postgres psql -c "
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn
FROM pg_replication_slots;
"
```

Check publications:

```sql
SELECT * FROM pg_publication;
```

Check subscriptions:

```sql
SELECT * FROM pg_subscription;
```

Logical replication requires its own upgrade planning.

The normal `pg_upgrade` procedure is not a complete logical-replication migration procedure.

For physical standbys, choose beforehand whether you will:

```text
Option A:
Rebuild the PostgreSQL 18 standby from the upgraded primary.

or

Option B:
Use PostgreSQL's documented standby-upgrade procedure.
```

For most environments, rebuilding standbys after primary upgrade is operationally simpler.

---

# 20. Check Archiving

Run:

```bash
sudo -u postgres psql -c "SHOW archive_mode;"
sudo -u postgres psql -c "SHOW archive_command;"
```

Check:

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

Do not assume PostgreSQL 16 archive configuration should be blindly copied into PostgreSQL 18.

Review it.

---

# 21. Capture Non-Default Parameters

Run:

```sql
SELECT
    name,
    setting,
    unit,
    source
FROM pg_settings
WHERE source <> 'default'
ORDER BY name;
```

Also back up:

```bash
sudo cp -a \
/etc/postgresql/16/main \
/DBBackups2026/postgresql16_config_backup
```

Do not simply overwrite PostgreSQL 18's `postgresql.conf` with PostgreSQL 16's file.

Parameters can:

* change defaults;
* become obsolete;
* acquire new behavior;
* be removed;
* have new recommended values.

Review and migrate settings deliberately.

---

# 22. Check PostgreSQL 17 and 18 Release Notes

Because the upgrade skips PostgreSQL 17, review compatibility changes from:

```text
PostgreSQL 16
     ↓
PostgreSQL 17 changes
     ↓
PostgreSQL 18 changes
```

Pay special attention to:

* changed defaults;
* removed configuration parameters;
* authentication behavior;
* extensions;
* SQL behavior;
* planner behavior;
* replication changes;
* backup/recovery behavior;
* monitoring changes.

---

# 23. Phase 2 — Backup Preparation

A successful `pg_upgrade` does not replace having a valid backup.

Before the upgrade, take a verified production backup.

For small environments this might include:

```bash
sudo -u postgres pg_dumpall \
> /DBBackups2026/pre_pg18_upgrade.sql
```

For large production systems, use the organization's normal enterprise backup method such as:

```text
pgBackRest
Barman
Storage snapshot
Filesystem snapshot
Enterprise backup platform
```

The backup must have a known restore procedure.

A backup that has never been tested is not sufficient evidence of recoverability.

---

# 24. Backup Configuration Files

Run:

```bash
sudo mkdir -p /DBBackups2026/pg16_upgrade_backup
```

Copy:

```bash
sudo cp -a \
/etc/postgresql/16/main \
/DBBackups2026/pg16_upgrade_backup/
```

Save relevant service and OS configuration as well.

---

# 25. Check Free Disk Space

Run:

```bash
df -h
```

Check PostgreSQL storage:

```bash
du -sh /var/lib/postgresql/16/main
```

Check tablespaces separately.

Example:

```bash
du -sh /PGDATA
du -sh /PGINDEX
```

The required free space depends heavily on the chosen `pg_upgrade` transfer method.

Default copy mode requires significantly more storage than link/clone approaches.

---

# 26. Phase 3 — Install PostgreSQL 18 Side-by-Side

Install the PostgreSQL 18 server and required packages using your approved Ubuntu/PostgreSQL repository.

After installation verify:

```bash
/usr/lib/postgresql/16/bin/postgres --version
```

and:

```bash
/usr/lib/postgresql/18/bin/postgres --version
```

Both versions should exist.

Architecture:

```text
/usr/lib/postgresql/16/bin
/usr/lib/postgresql/18/bin
```

Do not uninstall PostgreSQL 16.

---

# 27. Verify Cluster State

Run:

```bash
pg_lsclusters
```

Depending on how PostgreSQL 18 was installed, a PostgreSQL 18 `main` cluster might have been created automatically.

Example:

```text
Ver Cluster Port Status Owner
16  main    5432 online postgres
18  main    5433 online postgres
```

Do not proceed blindly if a target cluster already contains data.

It should be a fresh target cluster intended specifically for the upgrade.

---

# 28. Install PostgreSQL 18 Extension Packages

For every third-party extension present in PostgreSQL 16, install the corresponding PostgreSQL 18 package/library.

Example conceptually:

```text
PG16 extension binaries
       ↓
Install PG18-compatible binaries
       ↓
pg_upgrade migrates database metadata
```

Do not manually recreate all extension objects in the PostgreSQL 18 database before the upgrade.

For example, do not blindly run:

```sql
CREATE EXTENSION postgis;
```

on the target merely because the source has PostGIS.

The required PostgreSQL 18 binary/library should be installed instead.

---

# 29. Check Custom Full-Text Search Files

PostgreSQL specifically identifies custom full-text-search files as upgrade items.

Check for custom:

```text
Dictionaries
Thesaurus files
Stop-word files
Synonym files
Custom text-search files
```

Install or copy compatible versions into the PostgreSQL 18 installation before production cutover.

---

# 30. Phase 4 — Compatibility Test

Before the production downtime, run compatibility checks.

For a manual upstream `pg_upgrade`, PostgreSQL recommends:

```bash
pg_upgrade --check
```

On Ubuntu, the production upgrade is better integrated through:

```bash
pg_upgradecluster
```

using:

```text
--method=upgrade
```

which selects `pg_upgrade` rather than the slower dump-based upgrade method.

For complex or highly critical systems, perform a full upgrade rehearsal on a cloned/restored copy of production before the maintenance window.

---

# 31. Rehearsal Recommendation

A production-grade workflow should ideally be:

```text
Production PG16
      |
      +---- Backup / Snapshot
               |
               v
          Test Server
               |
               v
       PG16 → PG18 rehearsal
               |
               +---- Record duration
               +---- Record errors
               +---- Record extension issues
               +---- Validate applications
```

Record:

```text
Upgrade duration
Database startup duration
Statistics refresh duration
Extension update duration
Application validation duration
Rollback duration
```

The rehearsal gives a realistic maintenance-window estimate.

---

# 32. Phase 5 — Maintenance Window

Before stopping PostgreSQL, stop or drain all writers.

Examples:

```text
Application services
ETL
Batch jobs
Cron jobs
Schedulers
Monitoring systems that execute writes
Reporting applications that create temp/persistent objects
PgBouncer pools
Background integration jobs
```

---

# 33. Check Active Connections

Run:

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    state,
    backend_start,
    xact_start
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
ORDER BY datname, usename;
```

Make sure application traffic has been stopped.

---

# 34. Take Final Recovery Point

Immediately before the major upgrade, create the final approved recovery point.

Depending on your environment:

```text
pgBackRest full/differential backup
Storage snapshot
VM/storage snapshot
Enterprise backup
```

Record the backup ID/snapshot ID in the change ticket.

---

# 35. Stop PostgreSQL 16 Cleanly

On Ubuntu:

```bash
sudo pg_ctlcluster 16 main stop
```

Verify:

```bash
pg_lsclusters
```

Expected:

```text
16  main  5432  down
```

Also check:

```bash
ps -ef | grep '[p]ostgres'
```

Verify there is no unexpected PostgreSQL 16 server process.

---

# 36. Recommended Ubuntu Upgrade Method

For Ubuntu package-managed PostgreSQL clusters, use:

```bash
sudo pg_upgradecluster \
-v 18 \
--method=upgrade \
16 main
```

This is preferred over manually mixing Ubuntu configuration paths and upstream `pg_upgrade` paths.

`pg_upgradecluster`:

* creates/manages the new cluster;
* integrates with `/etc/postgresql`;
* handles Ubuntu's cluster layout;
* adjusts configuration;
* manages cluster ports;
* keeps the old cluster after upgrade;
* can use PostgreSQL `pg_upgrade` internally with `--method=upgrade`.

The old cluster is not automatically destroyed.

That provides a safer validation period.

---

# 37. Important — Do Not Omit `--method=upgrade`

Use:

```bash
sudo pg_upgradecluster \
-v 18 \
--method=upgrade \
16 main
```

Do not assume:

```bash
pg_upgradecluster 16 main
```

will automatically use `pg_upgrade`.

The explicit option:

```text
--method=upgrade
```

makes the intended strategy clear.

---

# 38. Transfer Method Considerations

PostgreSQL `pg_upgrade` supports multiple mechanisms depending on platform/filesystem and tooling.

Typical choices include:

```text
copy
link
clone
copy-file-range
swap
```

## Copy

Advantages:

```text
Old cluster remains independent
Simpler rollback
Lowest risk of file sharing
```

Disadvantages:

```text
Requires more disk
Can take longer
```

## Link

Advantages:

```text
Very fast
Very little additional disk
Good for multi-terabyte clusters
```

Disadvantages:

```text
Strong rollback restrictions after PG18 starts
Old and new clusters can share underlying files
```

## Clone

Where supported by the filesystem:

```text
Very fast
Low additional initial disk usage
Old cluster remains independent
```

This can be attractive on filesystems supporting reflinks.

## Swap

PostgreSQL 18 adds `--swap`.

It can be extremely fast in suitable environments, but has stricter operational and rollback implications.

For a first production 16 → 18 upgrade, default/copy or a carefully tested clone strategy is generally easier to govern.

Use link/swap only after rehearsing the exact rollback process.

---

# 39. Never Use `--no-sync` Casually in Production

PostgreSQL provides:

```text
--no-sync
```

primarily to accelerate testing.

Do not use it for a normal production upgrade.

An OS crash before data is safely synchronized to disk can leave the new cluster corrupted.

Production SOP:

```text
DO NOT USE --no-sync
```

unless there is a specifically tested and approved engineering reason.

---

# 40. PostgreSQL 18 Checksum Change

PostgreSQL 18 enables data checksums by default for clusters initialized with `initdb`.

This is a major upgrade consideration.

PostgreSQL documentation specifically states that `pg_upgrade` requires compatible checksum settings between clusters. ([PostgreSQL][2])

On Ubuntu, current PostgreSQL cluster-management tooling understands checksum handling, but still validate the final state.

After upgrade:

```bash
sudo -u postgres psql -c "SHOW data_checksums;"
```

Compare it with the pre-upgrade result.

---

# 41. Phase 6 — Check Upgrade Result

Run:

```bash
pg_lsclusters
```

The target PostgreSQL 18 cluster should now be present.

Verify:

```bash
sudo -u postgres psql -c "SELECT version();"
```

Expected:

```text
PostgreSQL 18.x
```

Also:

```bash
sudo -u postgres psql -c "SHOW server_version;"
```

---

# 42. Verify Data Directory

Run:

```bash
sudo -u postgres psql -c "SHOW data_directory;"
```

Expected Ubuntu-style path:

```text
/var/lib/postgresql/18/main
```

---

# 43. Verify Configuration

Run:

```bash
sudo -u postgres psql -c "SHOW config_file;"
sudo -u postgres psql -c "SHOW hba_file;"
```

Expected:

```text
/etc/postgresql/18/main/postgresql.conf
/etc/postgresql/18/main/pg_hba.conf
```

---

# 44. Review PostgreSQL 18 Configuration

Do not assume every PostgreSQL 16 setting should remain unchanged.

Compare:

```bash
diff -u \
/etc/postgresql/16/main/postgresql.conf \
/etc/postgresql/18/main/postgresql.conf
```

Review at least:

```text
listen_addresses
port
max_connections

shared_buffers
work_mem
maintenance_work_mem
effective_cache_size

wal_level
max_wal_size
min_wal_size
checkpoint_timeout

max_wal_senders
max_replication_slots

archive_mode
archive_command

shared_preload_libraries

ssl
ssl_cert_file
ssl_key_file

logging_collector
log_directory
log_filename

autovacuum
autovacuum_max_workers
```

Do not overwrite PostgreSQL 18's file with PostgreSQL 16's configuration file.

---

# 45. Review `pg_hba.conf`

Compare:

```bash
diff -u \
/etc/postgresql/16/main/pg_hba.conf \
/etc/postgresql/18/main/pg_hba.conf
```

Verify:

```text
Local authentication
Application subnets
DBA access
Replication access
Monitoring access
SSL requirements
Authentication method
```

Test authentication rather than assuming the copied configuration is correct.

---

# 46. Review `pg_ident.conf`

If identity mapping is used:

```bash
diff -u \
/etc/postgresql/16/main/pg_ident.conf \
/etc/postgresql/18/main/pg_ident.conf
```

Confirm mappings still behave as intended.

---

# 47. Validate Database Inventory

Run:

```bash
sudo -u postgres psql -c "\l+"
```

Compare against the pre-upgrade inventory.

Check database sizes:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS database_size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Small size differences can be normal, but unexpected large differences require investigation.

---

# 48. Validate Roles

Run:

```bash
sudo -u postgres psql -c "\du+"
```

Compare:

```text
Role names
SUPERUSER
CREATEDB
CREATEROLE
LOGIN
REPLICATION
BYPASSRLS
Role memberships
```

---

# 49. Validate Extensions

Run per database:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY extname;
```

Check available versions:

```sql
SELECT
    name,
    default_version,
    installed_version
FROM pg_available_extensions
WHERE installed_version IS NOT NULL
ORDER BY name;
```

Upgrade extensions where the extension vendor/documentation requires it.

Example:

```sql
ALTER EXTENSION extension_name UPDATE;
```

Do not issue this blindly.

Review extension-specific upgrade instructions first.

---

# 50. Run `pg_upgrade` Generated Follow-Up Actions

Review all output from the upgrade.

`pg_upgrade` can identify objects requiring post-upgrade processing or generate scripts for corrective work.

Complete any required rebuild/update actions before allowing full application traffic.

Do not ignore generated scripts.

---

# 51. PostgreSQL 18 Statistics Handling

PostgreSQL 18 significantly improves major upgrades by preserving optimizer statistics through `pg_upgrade`. ([PostgreSQL][3])

However, not every statistic is preserved.

In particular, extended statistics are not preserved.

Therefore post-upgrade statistics work is still required.

---

# 52. Generate Missing Statistics First

Recommended first step:

```bash
sudo -u postgres vacuumdb \
--all \
--analyze-in-stages \
--missing-stats-only \
--jobs=4
```

This prioritizes objects lacking usable statistics.

Adjust `--jobs` based on tested CPU and I/O capacity.

---

# 53. Perform Full Statistics Refresh

After the system is operational and resources permit:

```bash
sudo -u postgres vacuumdb \
--all \
--analyze-only \
--jobs=4
```

This refreshes statistics across the cluster.

For a very large production environment, schedule and tune this operation according to system capacity.

---

# 54. Check Invalid Indexes

Run in each important database:

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS index_name
FROM pg_index i
JOIN pg_class c
    ON c.oid = i.indexrelid
JOIN pg_namespace n
    ON n.oid = c.relnamespace
WHERE NOT i.indisvalid
ORDER BY 1,2;
```

Investigate every unexpected result.

---

# 55. Validate Objects

Check:

```text
Schemas
Tables
Partitions
Indexes
Sequences
Views
Materialized views
Functions
Procedures
Triggers
Extensions
Foreign tables
Publications
Subscriptions
```

Useful `psql` commands include:

```text
\dn+
\dt+
\di+
\dv+
\dm+
\df+
\des+
```

---

# 56. Validate Sequences

Application-generated numeric keys deserve special attention.

Example check:

```sql
SELECT
    schemaname,
    sequencename,
    last_value
FROM pg_sequences
ORDER BY schemaname, sequencename;
```

Compare with expected application behavior.

---

# 57. Validate Application Connectivity

Test from:

```text
Database server
Application server
Batch server
ETL server
Monitoring server
PgBouncer/proxy
```

Example:

```bash
psql \
-h database-host \
-p 5432 \
-U app_user \
-d application_database
```

Validate:

```text
DNS
Port
Firewall
TLS
pg_hba.conf
Credentials
Role privileges
Connection pooling
```

---

# 58. Application Smoke Testing

Do not declare success only because PostgreSQL starts.

Test real workflows:

```text
Login
SELECT
INSERT
UPDATE
DELETE
Transactions
Functions
Procedures
Reports
ETL
Batch processing
API requests
Scheduled jobs
Connection pooling
```

Business/application owners should confirm critical workflows.

---

# 59. Monitor PostgreSQL Logs

On Ubuntu:

```bash
journalctl -u postgresql -f
```

Also inspect PostgreSQL's configured log directory.

Look for:

```text
ERROR
FATAL
PANIC
missing library
authentication failure
collation warning
checkpoint issue
WAL issue
replication failure
extension failure
```

---

# 60. Check Database Activity

Run:

```sql
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit
FROM pg_stat_database
ORDER BY datname;
```

---

# 61. Check Active Connections

```sql
SELECT
    datname,
    usename,
    application_name,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY
    datname,
    usename,
    application_name,
    state
ORDER BY datname, count(*) DESC;
```

Compare connection patterns with the PostgreSQL 16 baseline.

---

# 62. Check Long-Running Queries

```sql
SELECT
    pid,
    usename,
    datname,
    now() - query_start AS duration,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

Investigate new performance regressions.

---

# 63. Check Blocking

```sql
SELECT
    pid,
    pg_blocking_pids(pid) AS blocking_pids,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Unexpected blocking immediately after cutover can indicate application or query-plan changes.

---

# 64. Check Autovacuum

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 30;
```

Confirm autovacuum is working normally.

---

# 65. Check WAL Archiving

Run:

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

Verify new WAL files are reaching the archive destination.

A successful database upgrade with broken WAL archiving is not a successful production change.

---

# 66. Validate Physical Replication

If physical standbys are used:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

Expected state normally includes:

```text
state = streaming
```

Validate replication lag.

---

# 67. Rebuild Standbys Where Required

A common production strategy is:

```text
PG16 Primary
     |
     | Major upgrade
     v
PG18 Primary
     |
     | pg_basebackup / backup restore
     v
PG18 Standby
```

Do not attempt to attach an old PostgreSQL 16 physical standby directly to a PostgreSQL 18 primary.

Physical replication requires matching PostgreSQL major-version architecture.

---

# 68. Validate Logical Replication Separately

If logical replication exists, validate:

```sql
SELECT *
FROM pg_stat_subscription;
```

Also:

```sql
SELECT *
FROM pg_replication_slots;
```

Logical replication major upgrades have additional considerations and should follow PostgreSQL's dedicated logical-replication upgrade guidance.

---

# 69. Validate Backup Platform

Immediately test:

```text
Backup connection
Repository access
Backup job
WAL archive
Retention
Monitoring
Encryption
Restore metadata
```

Update any scripts containing PostgreSQL 16 paths.

Example old path:

```text
/usr/lib/postgresql/16/bin
```

New:

```text
/usr/lib/postgresql/18/bin
```

---

# 70. Take New PostgreSQL 18 Baseline Backup

Once PostgreSQL 18 has passed initial validation, create a new full/baseline backup.

Architecture:

```text
PostgreSQL 18
     |
     +---- Full Backup
     |
     +---- WAL Archive
     |
     +---- Retention Policy
```

This should be completed before the old cluster is finally removed.

---

# 71. Validate Monitoring

Update:

```text
PostgreSQL version
Binary paths
Data directory
Config path
Service/cluster
Log path
Metrics queries
Replication checks
Backup checks
Extension monitoring
```

Test alerts.

Examples:

```text
Server down
High connections
Replication lag
Archive failure
Disk space
Deadlocks
Long queries
Backup failure
Autovacuum issues
```

---

# 72. Performance Baseline Comparison

Compare PostgreSQL 18 against the captured PostgreSQL 16 baseline.

Review:

```text
CPU
Memory
Disk latency
Disk throughput
Connections
TPS
Query latency
Cache hit ratio
WAL generation
Checkpoint activity
Autovacuum
Top SQL
Locks
```

Do not expect identical query plans across major releases.

Planner improvements and behavior changes are one reason application testing is important.

---

# 73. Rollback Strategy — Copy/Independent Storage

If the PostgreSQL 18 upgrade fails before accepting production writes:

```text
1. Stop applications.
2. Stop PostgreSQL 18.
3. Confirm PG16 cluster integrity.
4. Restore PG16 configuration/port as required.
5. Start PostgreSQL 16.
6. Validate PostgreSQL 16.
7. Restore application traffic.
8. Investigate PostgreSQL 18 failure.
```

The rollback is simplest before significant PostgreSQL 18 transactions occur.

---

# 74. Important Data-Divergence Warning

Once applications write to PostgreSQL 18:

```text
PG16 data
    !=
PG18 data
```

Those PostgreSQL 18 transactions are not automatically applied back to PostgreSQL 16.

Therefore:

```text
Initial PG18 validation
       ↓
Go/No-Go decision
       ↓
Open application writes
```

is safer than immediately opening all production traffic.

---

# 75. Link-Mode Rollback Warning

If link mode is used, rollback must be treated differently.

After the PostgreSQL 18 cluster starts, linked data files may have been modified by PostgreSQL 18.

Therefore:

```text
DO NOT simply restart PostgreSQL 16
```

after PostgreSQL 18 has been started using link mode.

Use the documented PostgreSQL rollback/recovery procedure or restore the PostgreSQL 16 cluster from the protected backup/snapshot.

This is why link mode requires much more disciplined rollback planning.

---

# 76. Keep PostgreSQL 16 Temporarily

Do not immediately run:

```bash
pg_dropcluster 16 main
```

Keep PostgreSQL 16 until:

```text
[ ] Database validation passed
[ ] Application validation passed
[ ] DBA validation passed
[ ] Performance validation passed
[ ] Extensions validated
[ ] Monitoring validated
[ ] Backup validated
[ ] Replication validated
[ ] Business acceptance completed
[ ] Rollback window expired
```

---

# 77. Final PostgreSQL 16 Removal

Only after formal approval:

```bash
sudo pg_dropcluster 16 main
```

Use the appropriate option according to your environment and verify exactly what will be deleted before executing removal commands.

This is destructive.

Confirm backups and the rollback-window closure first.

---

# 78. Package Cleanup

Only after the old cluster is no longer required should PostgreSQL 16 packages be considered for removal.

Do not combine:

```text
Major upgrade
+
Old-cluster deletion
+
Old-package deletion
```

into one change step.

Separating these activities provides a safer rollback window.

---

# 79. Final Production Validation Checklist

## PostgreSQL

```text
[ ] PostgreSQL 18 starts successfully
[ ] server_version reports PostgreSQL 18
[ ] Data directory is correct
[ ] Configuration path is correct
[ ] PostgreSQL logs are clean
```

## Databases

```text
[ ] All databases present
[ ] Database owners correct
[ ] Database sizes reasonable
[ ] Encoding correct
[ ] Locale/collation reviewed
```

## Security

```text
[ ] Roles present
[ ] Memberships correct
[ ] pg_hba.conf validated
[ ] TLS validated
[ ] Application authentication works
[ ] Replication authentication works
```

## Objects

```text
[ ] Schemas present
[ ] Tables present
[ ] Indexes present
[ ] Invalid indexes checked
[ ] Views present
[ ] Functions present
[ ] Procedures present
[ ] Sequences validated
[ ] Extensions validated
```

## Performance

```text
[ ] Missing statistics generated
[ ] Full ANALYZE completed/planned
[ ] Query performance compared
[ ] CPU normal
[ ] Memory normal
[ ] Disk I/O normal
[ ] Blocking normal
[ ] Autovacuum operational
```

## Backup

```text
[ ] PostgreSQL 18 full backup successful
[ ] WAL archiving successful
[ ] Retention validated
[ ] Backup monitoring successful
[ ] Restore procedure updated
```

## HA / DR

```text
[ ] Standby rebuilt/upgraded
[ ] Replication streaming
[ ] Replication lag acceptable
[ ] Replication slots validated
[ ] DR documentation updated
```

## Application

```text
[ ] Application login successful
[ ] Read operations successful
[ ] Write operations successful
[ ] Batch jobs successful
[ ] ETL successful
[ ] Reports successful
[ ] APIs successful
[ ] Connection pool validated
```

---

# 80. Recommended Production Flow

```text
PostgreSQL 16 Production
          |
          v
Inventory + Health Checks
          |
          v
Extension Compatibility
          |
          v
Backup + Restore Validation
          |
          v
Install PostgreSQL 18
          |
          v
Upgrade Rehearsal
          |
          v
Maintenance Window
          |
          v
Stop Application Writes
          |
          v
Final Recovery Point
          |
          v
Stop PostgreSQL 16
          |
          v
pg_upgradecluster
--method=upgrade
          |
          v
PostgreSQL 18
          |
          v
Generated Upgrade Actions
          |
          v
Statistics
          |
          v
DBA Validation
          |
          v
Application Smoke Test
          |
          v
Open Production Traffic
          |
          v
Performance Monitoring
          |
          v
Backup + Replication Validation
          |
          v
Acceptance
          |
          v
Rollback Window
          |
          v
Remove PostgreSQL 16
```

---

# 81. Core Ubuntu Commands — Quick Reference

## Check clusters

```bash
pg_lsclusters
```

## Check PostgreSQL 16

```bash
sudo -u postgres psql -c "SELECT version();"
```

## Check checksums

```bash
sudo -u postgres psql -c "SHOW data_checksums;"
```

## Check source cluster

```bash
sudo -u postgres \
/usr/lib/postgresql/16/bin/pg_controldata \
/var/lib/postgresql/16/main
```

## Check PostgreSQL binaries

```bash
/usr/lib/postgresql/16/bin/postgres --version
/usr/lib/postgresql/18/bin/postgres --version
```

## Stop PostgreSQL 16

```bash
sudo pg_ctlcluster 16 main stop
```

## Execute Ubuntu side-by-side major upgrade

```bash
sudo pg_upgradecluster \
-v 18 \
--method=upgrade \
16 main
```

## Check clusters after upgrade

```bash
pg_lsclusters
```

## Verify PostgreSQL 18

```bash
sudo -u postgres psql \
-c "SELECT version();"
```

## Generate missing statistics

```bash
sudo -u postgres vacuumdb \
--all \
--analyze-in-stages \
--missing-stats-only \
--jobs=4
```

## Refresh statistics

```bash
sudo -u postgres vacuumdb \
--all \
--analyze-only \
--jobs=4
```

## Check databases

```bash
sudo -u postgres psql -c "\l+"
```

## Check extensions

```bash
sudo -u postgres psql -c "
SELECT extname, extversion
FROM pg_extension
ORDER BY extname;
"
```

## Check replication

```bash
sudo -u postgres psql -c "
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
"
```

---

# 82. Major Do's and Don'ts

## Do

```text
DO upgrade PG16 to a current 16.x maintenance release first.

DO review PG17 and PG18 release notes.

DO perform an upgrade rehearsal.

DO verify extension compatibility.

DO take a verified backup.

DO preserve PG16 during the rollback window.

DO use PostgreSQL 18-compatible extension libraries.

DO validate checksums.

DO validate collation behavior.

DO regenerate missing statistics.

DO take a fresh PostgreSQL 18 backup.

DO validate HA and monitoring.
```

## Don't

```text
DON'T uninstall PostgreSQL 16 before the upgrade.

DON'T blindly copy postgresql.conf.

DON'T blindly copy old extension binaries.

DON'T ignore pg_upgrade warnings.

DON'T use --no-sync casually in production.

DON'T assume PG16 can simply restart after link-mode PG18 startup.

DON'T open all application traffic before validation.

DON'T forget replication/standbys.

DON'T delete PG16 immediately after upgrade.

DON'T consider PostgreSQL startup alone proof of success.
```

---

# 83. Final Recommended Strategy

For a standard Ubuntu 26.04 package-managed PostgreSQL environment:

```text
PostgreSQL 16
     ↓
Install PostgreSQL 18 side-by-side
     ↓
Validate extensions
     ↓
Backup
     ↓
Rehearse
     ↓
Stop application
     ↓
Final recovery point
     ↓
pg_upgradecluster
-v 18
--method=upgrade
16 main
     ↓
PostgreSQL 18
     ↓
Statistics + Extension Validation
     ↓
Application Validation
     ↓
Backup + Replication Validation
     ↓
Monitor
     ↓
Retain PG16 temporarily
     ↓
Remove PG16 after acceptance
```

This gives a much cleaner Ubuntu-specific procedure than manually mixing upstream PostgreSQL paths with Ubuntu's PostgreSQL cluster-management framework.

---

# 84. Official Technical Basis

This SOP follows the PostgreSQL 18 major-upgrade and `pg_upgrade` guidance.

PostgreSQL documents `pg_upgrade` as the supported method for rapidly upgrading an existing PostgreSQL cluster without performing a full logical dump and restore.

PostgreSQL 18 also introduces several upgrade-specific considerations, including:

* optimizer-statistics preservation;
* checksums enabled by default for newly initialized clusters;
* additional `pg_upgrade` transfer capabilities;
* improved parallel processing;
* new upgrade-related options.

Ubuntu 26.04 provides PostgreSQL 18 and the Debian/Ubuntu PostgreSQL cluster-management framework, which includes tools such as:

```text
pg_lsclusters
pg_ctlcluster
pg_createcluster
pg_upgradecluster
pg_dropcluster
```

For Ubuntu package-managed servers, these tools should be considered when designing the operational runbook because Ubuntu separates PostgreSQL configuration and data directories.

---

# 85. References

**PostgreSQL 18 — pg_upgrade**

[https://www.postgresql.org/docs/18/pgupgrade.html](https://www.postgresql.org/docs/18/pgupgrade.html)

**PostgreSQL 18 — Major Version Upgrade**

[https://www.postgresql.org/docs/18/upgrading.html](https://www.postgresql.org/docs/18/upgrading.html)

**PostgreSQL 18 — Release Notes**

[https://www.postgresql.org/docs/18/release-18.html](https://www.postgresql.org/docs/18/release-18.html)

**PostgreSQL 18 — Data Checksums**

[https://www.postgresql.org/docs/18/checksums.html](https://www.postgresql.org/docs/18/checksums.html)

**PostgreSQL 18 — initdb**

[https://www.postgresql.org/docs/18/app-initdb.html](https://www.postgresql.org/docs/18/app-initdb.html)

**Ubuntu — pg_upgrade**

[https://manpages.ubuntu.com/manpages/resolute/man1/pg_upgrade.1.html](https://manpages.ubuntu.com/manpages/resolute/man1/pg_upgrade.1.html)

**Ubuntu — PostgreSQL Cluster Management**

Use the locally installed documentation for your exact Ubuntu 26.04 `postgresql-common` package:

```bash
man pg_upgradecluster
man pg_ctlcluster
man pg_lsclusters
man pg_dropcluster
```

---

# Conclusion

A PostgreSQL 16 → PostgreSQL 18 upgrade is a supported direct major-version upgrade.

On Ubuntu 26.04, a production implementation should account for Ubuntu's PostgreSQL cluster-management framework rather than treating the host like a generic source-code installation.

The safest operational pattern is:

```text
Precheck
   ↓
Backup
   ↓
Install PG18
   ↓
Compatibility Validation
   ↓
Rehearsal
   ↓
Maintenance Window
   ↓
pg_upgrade / pg_upgradecluster
   ↓
Post-Upgrade Processing
   ↓
Application Validation
   ↓
Backup + HA Validation
   ↓
Monitoring
   ↓
Rollback Window
   ↓
PG16 Decommission
```

The PostgreSQL 16 cluster should remain protected and untouched as long as practical until the PostgreSQL 18 production environment has been fully accepted.

One important improvement over the earlier notes is that this version **does not blindly use raw `pg_upgrade` with `/var/lib/postgresql/...` paths on Ubuntu**. Ubuntu 26.04 has its own PostgreSQL cluster-management layer, and the current Ubuntu PostgreSQL 18 package exposes the expected `pg_upgrade` architecture while `pg_upgradecluster` provides the Ubuntu-aware wrapper. ([Ubuntu Manpages][4])

Also, the PostgreSQL 18 checksum and statistics changes are explicitly reflected above: PostgreSQL 18 enables checksums by default for new clusters, and `pg_upgrade` can now preserve optimizer statistics, although extended statistics are not preserved. ([PostgreSQL][2])

If you want, I can next turn **this exact SOP into a professional Word document with cover page, document control, index/TOC, Calibri headings 14 and body 11**, without changing the technical content.

[1]: https://manpages.ubuntu.com/manpages/questing/man1/pg_upgradecluster.1.html "Ubuntu Manpage: pg_upgradecluster - upgrade an existing PostgreSQL cluster to a new major version."
[2]: https://www.postgresql.org/docs/current/release-18.html "PostgreSQL: Documentation: 18: E.6. Release 18"
[3]: https://www.postgresql.org/docs/18/release-18.html "PostgreSQL: Documentation: 18: E.6. Release 18"
[4]: https://manpages.ubuntu.com/manpages/resolute/man1/pg_upgrade.1.html "Ubuntu Manpage: pg_upgrade - upgrade a PostgreSQL server instance"

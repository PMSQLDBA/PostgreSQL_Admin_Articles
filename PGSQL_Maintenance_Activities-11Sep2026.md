**60-topic PostgreSQL DBA Maintenance Activities**. \
**DVD Rental (`dvdrental`)** : Table/database example, and PostgreSQL **17/18** for administration, upgrade, monitoring, backup, WAL, replication, and version-specific topics.

PostgreSQL officially treats vacuuming, statistics, reindexing, log maintenance, backup/recovery, monitoring, WAL, replication, and reliability as core administration areas. 

PostgreSQL recommends autovacuum for most installations and notes that routine maintenance should be monitored and automated appropriately. ([PostgreSQL][1])

---

# PostgreSQL DBA Maintenance Activities

## 1. VACUUM

### What it is

`VACUUM` removes/reclaims space from obsolete tuple versions created by `UPDATE` and `DELETE`.

PostgreSQL MVCC does not immediately physically remove an old row version.

### DVD Rental example

```sql
\c dvdrental

UPDATE customer
SET active = 0
WHERE customer_id BETWEEN 1 AND 100;
```

Check statistics:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'customer';
```

Run:

```sql
VACUUM (VERBOSE) customer;
```

Or:

```sql
VACUUM (ANALYZE) customer;
```

### DBA point

Regular `VACUUM` generally makes dead-tuple space **reusable inside the relation**. It normally does not return that space directly to the operating system.

PostgreSQL officially recommends routine vacuuming and normally expects autovacuum to perform it. ([PostgreSQL][1])

---

# 2. Autovacuum

Autovacuum automatically executes `VACUUM` and `ANALYZE` when tables qualify.

Check:

```sql
SHOW autovacuum;
SHOW autovacuum_naptime;
SHOW autovacuum_max_workers;
```

Monitor DVD Rental:

```sql
SELECT
    relname,
    n_dead_tup,
    last_autovacuum,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Important:

```text
autovacuum_naptime = 1min
```

does **not** mean every table is vacuumed every minute.

Workers evaluate tables and perform maintenance when necessary. ([PostgreSQL][1])

---

# 3. VACUUM FULL

`VACUUM FULL` rewrites a table and can return unused space to the operating system.

Example:

```sql
VACUUM FULL customer;
```

Check before/after:

```sql
SELECT
    pg_size_pretty(pg_total_relation_size('customer'));
```

### Important

`VACUUM FULL`:

* rewrites the relation
* needs additional temporary disk space
* requires an `ACCESS EXCLUSIVE` lock
* blocks normal access to the table
* should not be routine maintenance

PostgreSQL recommends standard `VACUUM` instead wherever possible. ([PostgreSQL][1])

---

# 4. ANALYZE

`ANALYZE` gathers statistics used by the optimizer.

Example:

```sql
ANALYZE customer;
```

Detailed:

```sql
ANALYZE VERBOSE customer;
```

Inspect statistics:

```sql
SELECT
    tablename,
    attname,
    n_distinct,
    null_frac
FROM pg_stats
WHERE tablename = 'customer';
```

After a major data load, statistics may need refreshing so PostgreSQL can choose better execution plans.

---

# 5. Autoanalyze

Autovacuum workers also automatically perform `ANALYZE`.

Monitor:

```sql
SELECT
    relname,
    last_analyze,
    last_autoanalyze,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables;
```

Relevant settings:

```sql
SHOW autovacuum_analyze_threshold;
SHOW autovacuum_analyze_scale_factor;
```

---

# 6. REINDEX

`REINDEX` rebuilds an index.

Example:

```sql
REINDEX INDEX idx_customer_last_name;
```

Table:

```sql
REINDEX TABLE customer;
```

Database:

```sql
REINDEX DATABASE dvdrental;
```

Typical reasons include:

* index corruption
* unusual index bloat
* index storage parameter changes
* certain collation-related situations

PostgreSQL officially documents corruption and excessive index bloat among reasons to use `REINDEX`. ([PostgreSQL][2])

---

# 7. REINDEX CONCURRENTLY

This allows PostgreSQL to rebuild indexes while minimizing blocking of normal reads/writes.

Example:

```sql
REINDEX INDEX CONCURRENTLY idx_customer_last_name;
```

Or:

```sql
REINDEX TABLE CONCURRENTLY customer;
```

Use it when availability matters.

It requires more work and takes longer than normal `REINDEX`.

---

# 8. CLUSTER

`CLUSTER` physically reorganizes a table according to an index.

Example:

```sql
CREATE INDEX idx_payment_customer
ON payment(customer_id);

CLUSTER payment USING idx_payment_customer;
```

Check:

```sql
SELECT *
FROM pg_stat_progress_cluster;
```

Important: PostgreSQL does **not continuously maintain** this physical ordering.

---

# 9. Bloat Management

Bloat is excess table/index space caused by workload patterns where reusable space is not efficiently reused.

Basic observation:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Also inspect sizes:

```sql
SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(relid))
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

Do not automatically equate `n_dead_tup > 0` with serious bloat.

---

# 10. Dead Tuple Cleanup

Check:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Create some dead tuples:

```sql
UPDATE rental
SET return_date = return_date
WHERE rental_id BETWEEN 1 AND 500;
```

Then:

```sql
VACUUM rental;
```

Dead tuples are a normal result of PostgreSQL MVCC.

---

# 11. Tuple Freezing

PostgreSQL must eventually freeze sufficiently old tuple transaction information.

Inspect:

```sql
SELECT
    relname,
    age(relfrozenxid)
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC;
```

Manual maintenance example:

```sql
VACUUM (FREEZE) customer;
```

Do not routinely use `FREEZE` without understanding why you need it.

---

# 12. Transaction ID Wraparound Prevention

Monitor database XID age:

```sql
SELECT
    datname,
    age(datfrozenxid)
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

Check configuration:

```sql
SHOW autovacuum_freeze_max_age;
```

PostgreSQL can force autovacuum even if normal activity thresholds have not been reached because preventing XID wraparound is a database-safety requirement. ([PostgreSQL][1])

---

# 13. Multixact ID Wraparound Prevention

Multixacts are used when multiple transactions share certain row-level locks.

Inspect configuration:

```sql
SHOW autovacuum_multixact_freeze_max_age;
```

Inspect table age:

```sql
SELECT
    relname,
    mxid_age(relminmxid)
FROM pg_class
WHERE relkind = 'r'
ORDER BY mxid_age(relminmxid) DESC;
```

Autovacuum also protects against multixact wraparound. ([PostgreSQL][1])

---

# 14. Visibility Map Maintenance

The visibility map tracks heap pages whose tuples are:

* all-visible
* potentially all-frozen

This is particularly important for index-only scans.

Example:

```sql
VACUUM customer;
```

Then inspect a possible index-only plan:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id
FROM customer
WHERE customer_id BETWEEN 1 AND 100;
```

VACUUM updating the visibility map is one of the reasons PostgreSQL requires routine vacuuming. ([PostgreSQL][1])

---

# 15. Free Space Map Maintenance

The Free Space Map tracks reusable space in heap/index pages.

Normal VACUUM helps make dead-tuple space reusable.

Example:

```sql
DELETE FROM payment
WHERE payment_id > 9999999;

VACUUM payment;
```

Future inserts can reuse suitable available space rather than always extending the table.

---

# 16. Statistics Collection

PostgreSQL maintains cumulative statistics that DBAs use for maintenance.

Example:

```sql
SELECT *
FROM pg_stat_database
WHERE datname = 'dvdrental';
```

Useful values include:

```text
xact_commit
xact_rollback
blks_read
blks_hit
tup_returned
tup_fetched
tup_inserted
tup_updated
tup_deleted
temp_files
temp_bytes
```

PostgreSQL 18 exposes a broad set of `pg_stat_*` views for database, table, index, WAL, replication, checkpointer and I/O monitoring. ([PostgreSQL][3])

---

# 17. Extended Statistics

Normal column statistics can miss relationships between columns.

Create statistics:

```sql
CREATE STATISTICS customer_stats
    (dependencies, ndistinct)
ON store_id, active
FROM customer;
```

Refresh:

```sql
ANALYZE customer;
```

Inspect:

```sql
SELECT *
FROM pg_statistic_ext;
```

Use extended statistics when correlated predicates lead to poor cardinality estimates.

---

# 18. Checkpoints

A checkpoint ensures modified buffers reach persistent storage and creates a recovery reference point.

Monitor:

```sql
SELECT *
FROM pg_stat_checkpointer;
```

Configuration:

```sql
SHOW checkpoint_timeout;
SHOW checkpoint_completion_target;
SHOW max_wal_size;
```

Manual checkpoint:

```sql
CHECKPOINT;
```

Do not run manual checkpoints repeatedly without a specific operational reason.

---

# 19. WAL Management

WAL is PostgreSQL's Write-Ahead Log.

Inspect:

```sql
SELECT *
FROM pg_stat_wal;
```

Current LSN:

```sql
SELECT pg_current_wal_lsn();
```

WAL directory:

```text
$PGDATA/pg_wal
```

WAL supports crash recovery, physical replication, PITR and other durability mechanisms. ([PostgreSQL][4])

---

# 20. WAL Archiving

Enable:

```text
wal_level = replica
archive_mode = on
archive_command = '...'
```

Check:

```sql
SHOW archive_mode;
SHOW archive_command;
```

Monitor:

```sql
SELECT *
FROM pg_stat_archiver;
```

Important values:

```text
archived_count
failed_count
last_archived_wal
last_failed_wal
```

WAL archiving is a central part of continuous archiving and PITR. ([PostgreSQL][5])

---

# 21. Replication Slot Maintenance

List slots:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    confirmed_flush_lsn
FROM pg_replication_slots;
```

A forgotten slot can retain WAL indefinitely.

Estimate retained WAL:

```sql
SELECT
    slot_name,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal
FROM pg_replication_slots
WHERE restart_lsn IS NOT NULL;
```

---

# 22. WAL Retention Monitoring

Check configuration:

```sql
SHOW wal_keep_size;
SHOW max_slot_wal_keep_size;
SHOW max_wal_size;
```

Check WAL size:

```sql
SELECT
    pg_size_pretty(
        SUM(size)
    )
FROM pg_ls_waldir();
```

Monitor retention carefully because uncontrolled WAL growth can fill storage.

---

# 23. Backup

PostgreSQL documents three broad backup approaches:

* SQL dump
* file-system level backup
* continuous archiving

([PostgreSQL][5])

For production systems, backup design should start from required:

```text
RPO
RTO
Database size
Recovery method
Retention
Compliance
```

---

# 24. Base Backup

Physical backup:

```bash
pg_basebackup \
  -h localhost \
  -U replicator \
  -D /backup/base \
  -Fp \
  -Xs \
  -P
```

Tar format:

```bash
pg_basebackup \
  -D /backup/base \
  -Ft \
  -z \
  -P
```

PostgreSQL 18 backup documentation also covers incremental backup functionality. ([PostgreSQL][5])

---

# 25. Logical Backup

DVD Rental backup:

```bash
pg_dump -Fc \
  -d dvdrental \
  -f dvdrental.dump
```

Plain SQL:

```bash
pg_dump \
  -d dvdrental \
  -f dvdrental.sql
```

Cluster-wide globals:

```bash
pg_dumpall --globals-only > globals.sql
```

---

# 26. Restore

Custom format:

```bash
createdb dvdrental_restore

pg_restore \
  -d dvdrental_restore \
  dvdrental.dump
```

List backup contents:

```bash
pg_restore -l dvdrental.dump
```

Parallel restore:

```bash
pg_restore \
  -j 4 \
  -d dvdrental_restore \
  dvdrental.dump
```

---

# 27. Point-in-Time Recovery

PITR combines:

```text
Base backup
+
Archived WAL
+
Recovery target
```

Example target:

```text
recovery_target_time =
'2026-09-11 09:30:00-05'
```

Conceptually:

```text
Base Backup
     |
     v
Replay WAL
     |
     v
Stop before accidental event
```

PostgreSQL documents PITR as part of continuous archiving. ([PostgreSQL][5])

---

# 28. Backup Validation / Restore Testing

A backup is not fully trusted until you prove it can restore.

Example:

```bash
pg_restore --list dvdrental.dump
```

More importantly:

```bash
createdb backup_validation

pg_restore \
  -d backup_validation \
  dvdrental.dump
```

Then:

```sql
SELECT COUNT(*) FROM customer;
SELECT COUNT(*) FROM rental;
SELECT COUNT(*) FROM payment;
```

For physical backups, also test startup and application-level validation.

---

# 29. Index Maintenance

List indexes:

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'public';
```

Usage:

```sql
SELECT
    relname,
    indexrelname,
    idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan;
```

Maintenance does **not** mean rebuilding every index every weekend.

---

# 30. Index Bloat Monitoring

Start with index size and usage:

```sql
SELECT
    indexrelname,
    idx_scan,
    pg_size_pretty(
        pg_relation_size(indexrelid)
    ) AS index_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;
```

For confirmed serious bloat:

```sql
REINDEX INDEX CONCURRENTLY index_name;
```

Use evidence, not a fixed schedule.

---

# 31. Table Bloat Monitoring

Check tuple activity:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    n_tup_upd,
    n_tup_del
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Check size:

```sql
SELECT
    relname,
    pg_size_pretty(
        pg_total_relation_size(relid)
    )
FROM pg_stat_user_tables;
```

Dead tuples alone do not prove serious table bloat.

---

# 32. Partition Maintenance

For large production tables, partitions may simplify:

* retention
* archival
* loading
* index maintenance
* deletion of old data

Example parent:

```sql
CREATE TABLE rental_history
(
    rental_id bigint,
    rental_date timestamp,
    customer_id integer
)
PARTITION BY RANGE (rental_date);
```

---

# 33. Partition Creation

Example:

```sql
CREATE TABLE rental_2026_01
PARTITION OF rental_history
FOR VALUES FROM ('2026-01-01')
TO ('2026-02-01');
```

Next:

```sql
CREATE TABLE rental_2026_02
PARTITION OF rental_history
FOR VALUES FROM ('2026-02-01')
TO ('2026-03-01');
```

Pre-create future partitions before incoming rows require them.

---

# 34. Partition Detach / Drop

Detach:

```sql
ALTER TABLE rental_history
DETACH PARTITION rental_2025_01;
```

Then archive or drop:

```sql
DROP TABLE rental_2025_01;
```

This can be much faster than:

```sql
DELETE FROM huge_table
WHERE rental_date < ...;
```

for suitable retention designs.

---

# 35. Tablespace Maintenance

List:

```sql
SELECT
    spcname,
    pg_tablespace_location(oid)
FROM pg_tablespace;
```

Create:

```sql
CREATE TABLESPACE fast_storage
LOCATION '/pgdata/fast';
```

Move:

```sql
ALTER TABLE payment
SET TABLESPACE fast_storage;
```

Production storage changes should be carefully planned.

---

# 36. Database Statistics Monitoring

Example:

```sql
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    tup_inserted,
    tup_updated,
    tup_deleted,
    temp_files,
    temp_bytes
FROM pg_stat_database
WHERE datname = 'dvdrental';
```

This is one of the main DBA health views. ([PostgreSQL][3])

---

# 37. Long-Running Transaction Monitoring

```sql
SELECT
    pid,
    usename,
    state,
    xact_start,
    now() - xact_start AS duration,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Long transactions can:

* retain old snapshots
* delay VACUUM cleanup
* increase table bloat
* contribute to XID age problems

---

# 38. Idle-in-Transaction Monitoring

```sql
SELECT
    pid,
    usename,
    state,
    xact_start,
    query_start,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction';
```

Protect with configuration where appropriate:

```sql
SHOW idle_in_transaction_session_timeout;
```

Application code should commit or roll back promptly.

---

# 39. Lock Monitoring

```sql
SELECT
    pid,
    locktype,
    relation::regclass,
    mode,
    granted
FROM pg_locks
ORDER BY pid;
```

Join with activity:

```sql
SELECT
    a.pid,
    a.usename,
    l.mode,
    l.granted,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a
ON a.pid = l.pid;
```

PostgreSQL provides `pg_locks` specifically for lock monitoring. ([PostgreSQL][3])

---

# 40. Blocking Session Monitoring

Simple blocker detection:

```sql
SELECT
    pid,
    pg_blocking_pids(pid) AS blocking_pids,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Then inspect blocker:

```sql
SELECT *
FROM pg_stat_activity
WHERE pid = <blocking_pid>;
```

Terminate only when operationally justified:

```sql
SELECT pg_terminate_backend(<pid>);
```

---

# 41. Temporary File Monitoring

Database-level:

```sql
SELECT
    datname,
    temp_files,
    pg_size_pretty(temp_bytes)
FROM pg_stat_database;
```

Configuration:

```sql
SHOW work_mem;
SHOW temp_file_limit;
```

Excess temp files may indicate large sorts, hashes, aggregation or poorly sized memory settings.

---

# 42. Disk Space Monitoring

Database side:

```sql
SELECT
    pg_size_pretty(
        pg_database_size('dvdrental')
    );
```

Linux:

```bash
df -h
du -sh $PGDATA
```

PostgreSQL explicitly discusses disk-usage monitoring and disk-full failure behavior. ([PostgreSQL][3])

---

# 43. Database Size Monitoring

```sql
SELECT
    datname,
    pg_size_pretty(
        pg_database_size(datname)
    ) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

For DVD Rental:

```sql
SELECT
    pg_size_pretty(
        pg_database_size('dvdrental')
    );
```

Capture history daily to determine growth rate.

---

# 44. Table and Index Size Monitoring

```sql
SELECT
    relname,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_indexes_size(relid)) AS indexes,
    pg_size_pretty(pg_total_relation_size(relid)) AS total
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

This quickly identifies your biggest objects.

---

# 45. Connection Maintenance

Current connections:

```sql
SELECT
    datname,
    state,
    COUNT(*)
FROM pg_stat_activity
GROUP BY datname, state
ORDER BY datname;
```

Limits:

```sql
SHOW max_connections;
SHOW superuser_reserved_connections;
```

Connection storms may require an external connection pooler rather than simply increasing `max_connections`.

---

# 46. Role and Privilege Review

Roles:

```sql
\du
```

SQL:

```sql
SELECT
    rolname,
    rolsuper,
    rolcreaterole,
    rolcreatedb,
    rolcanlogin
FROM pg_roles;
```

Object privileges:

```sql
SELECT
    grantee,
    table_schema,
    table_name,
    privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'public';
```

Follow least privilege.

---

# 47. Log Maintenance / Rotation

Check:

```sql
SHOW logging_collector;
SHOW log_directory;
SHOW log_filename;
SHOW log_rotation_age;
SHOW log_rotation_size;
```

PostgreSQL includes log file management as one of its routine database maintenance areas. ([PostgreSQL][6])

Logs should be retained according to operational and compliance requirements without exhausting disk.

---

# 48. Configuration Maintenance

Inspect configuration:

```sql
SELECT
    name,
    setting,
    unit,
    source,
    pending_restart
FROM pg_settings
ORDER BY name;
```

Check file:

```sql
SHOW config_file;
```

Reload:

```sql
SELECT pg_reload_conf();
```

Remember:

```text
Some parameters = reloadable
Some parameters = restart required
```

PostgreSQL 18 documents server configuration parameters centrally in Chapter 19. ([PostgreSQL][7])

---

# 49. Extension Maintenance

Installed:

```sql
SELECT
    extname,
    extversion
FROM pg_extension;
```

Available updates:

```sql
SELECT
    name,
    default_version,
    installed_version
FROM pg_available_extensions
WHERE installed_version IS NOT NULL;
```

Example update:

```sql
ALTER EXTENSION pg_stat_statements UPDATE;
```

Test extension compatibility before a major PostgreSQL upgrade.

---

# 50. Minor Version Patching

Examples:

```text
17.x → newer 17.x
18.x → newer 18.x
```

Check:

```sql
SELECT version();
```

Minor releases generally contain bug, security and reliability fixes.

Operational workflow:

```text
Review release notes
      ↓
Test
      ↓
Backup / recovery readiness
      ↓
Patch binaries/packages
      ↓
Restart if needed
      ↓
Validate database/application
```

---

# 51. Major Version Upgrade

Example:

```text
PostgreSQL 17
     ↓
PostgreSQL 18
```

Common methods:

```text
pg_upgrade
pg_dump / pg_restore
Logical replication
```

Check old/new binaries:

```bash
/usr/lib/postgresql/17/bin/postgres --version
/usr/lib/postgresql/18/bin/postgres --version
```

Precheck:

```bash
pg_upgrade --check ...
```

Major-version upgrades require compatibility planning for extensions and application behavior.

---

# 52. Database Integrity / Consistency Checks

PostgreSQL does not have a direct SQL Server `DBCC CHECKDB` equivalent.

Important native mechanisms include:

### Checksums

```bash
pg_checksums --check -D /var/lib/postgresql/18/main
```

Inside PostgreSQL:

```sql
SHOW data_checksums;
```

PostgreSQL documents data checksums as part of reliability and WAL administration. ([PostgreSQL][4])

Logical validation and backup restore testing should supplement this.

---

# 53. Sequence Maintenance

DVD Rental sequences:

```sql
SELECT
    schemaname,
    sequencename,
    last_value
FROM pg_sequences
WHERE schemaname = 'public';
```

Compare sequence:

```sql
SELECT MAX(customer_id)
FROM customer;
```

Then sequence state if required:

```sql
SELECT setval(
    'customer_customer_id_seq',
    (SELECT MAX(customer_id) FROM customer)
);
```

Sequence correction can be required after certain bulk loads or manually supplied IDs.

---

# 54. Orphaned Object Review

This is an operational review rather than a single built-in PostgreSQL maintenance command.

Examples:

* abandoned schemas
* unused tables
* stale replication slots
* unused roles
* old partitions
* obsolete extensions
* abandoned sequences

Find non-system schemas:

```sql
SELECT schema_name
FROM information_schema.schemata
WHERE schema_name NOT LIKE 'pg_%'
  AND schema_name <> 'information_schema';
```

Never drop objects based only on apparent inactivity.

---

# 55. Unused Index Monitoring

```sql
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
ORDER BY idx_scan;
```

An index showing:

```text
idx_scan = 0
```

is a **candidate for investigation**, not automatically a candidate for deletion.

Statistics may have been reset recently, and indexes can support constraints or rare critical workloads.

---

# 56. Invalid Index Monitoring

```sql
SELECT
    n.nspname,
    c.relname AS index_name,
    i.indisvalid,
    i.indisready
FROM pg_index i
JOIN pg_class c
  ON c.oid = i.indexrelid
JOIN pg_namespace n
  ON n.oid = c.relnamespace
WHERE NOT i.indisvalid
   OR NOT i.indisready;
```

Invalid indexes may result from failed/concurrent index operations.

Possible remediation:

```sql
REINDEX INDEX CONCURRENTLY index_name;
```

depending on the situation.

---

# 57. Statistics Staleness Monitoring

Review:

```sql
SELECT
    relname,
    n_mod_since_analyze,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_mod_since_analyze DESC;
```

For a heavily changed table:

```sql
ANALYZE payment;
```

Poor statistics can produce poor cardinality estimates and suboptimal plans.

---

# 58. Autovacuum Tuning

Global settings:

```sql
SHOW autovacuum_vacuum_threshold;
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_vacuum_max_threshold;
SHOW autovacuum_analyze_threshold;
SHOW autovacuum_analyze_scale_factor;
SHOW autovacuum_max_workers;
SHOW autovacuum_naptime;
```

Large hot table example:

```sql
ALTER TABLE payment SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_vacuum_threshold = 1000
);
```

Check:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'payment';
```

PostgreSQL specifically recommends adjusting autovacuum parameters where needed rather than assuming one configuration fits every workload. ([PostgreSQL][1])

---

# 59. Checkpoint Tuning

Important:

```sql
SHOW checkpoint_timeout;
SHOW checkpoint_completion_target;
SHOW max_wal_size;
SHOW min_wal_size;
```

Monitor PostgreSQL 18:

```sql
SELECT *
FROM pg_stat_checkpointer;
```

Frequent requested checkpoints can indicate insufficient WAL sizing or workload/configuration mismatch.

Conceptually:

```text
Too frequent checkpoints
       ↓
more write activity
       ↓
possible I/O pressure
```

Avoid tuning `max_wal_size` independently of recovery, storage and workload considerations.

---

# 60. Capacity Planning

Capacity planning combines historical growth and workload information.

Monitor database size:

```sql
SELECT
    current_date AS capture_date,
    pg_database_size('dvdrental') AS bytes;
```

Largest tables:

```sql
SELECT
    relname,
    pg_total_relation_size(relid) AS bytes
FROM pg_stat_user_tables
ORDER BY bytes DESC;
```

Connections:

```sql
SELECT COUNT(*)
FROM pg_stat_activity;
```

WAL:

```sql
SELECT *
FROM pg_stat_wal;
```

Temporary workload:

```sql
SELECT
    temp_files,
    temp_bytes
FROM pg_stat_database
WHERE datname = 'dvdrental';
```

A production capacity forecast should cover:

```text
CPU
RAM
Data storage
Index storage
WAL
Temporary space
Backup storage
Archive storage
Replication
Network
Connections
Table growth
Transaction growth
Retention
RPO/RTO
```

A simple growth example:

```text
Current database = 2 TB
Growth = 100 GB/month
Forecast = 12 months

2 TB + 1.2 TB
≈ 3.2 TB

Then add operational headroom.
```

Do not provision exactly 3.2 TB; VACUUM, index builds, `REINDEX`, WAL, upgrades, restores and maintenance operations can require additional capacity.

---

# Recommended DBA Maintenance Schedule

| Activity                | Daily | Weekly | Monthly | Quarterly | On Demand |
| ----------------------- | :---: | :----: | :-----: | :-------: | :-------: |
| Autovacuum health       |   ✓   |        |         |           |           |
| Dead tuples             |   ✓   |        |         |           |           |
| XID age                 |   ✓   |        |         |           |           |
| Replication/WAL         |   ✓   |        |         |           |           |
| Backup status           |   ✓   |        |         |           |           |
| Disk capacity           |   ✓   |        |         |           |           |
| Locks/long transactions |   ✓   |        |         |           |           |
| Bloat trend             |       |    ✓   |         |           |           |
| Index usage             |       |    ✓   |         |           |           |
| Statistics health       |       |    ✓   |         |           |           |
| Growth trend            |       |    ✓   |    ✓    |           |           |
| Restore validation      |       |        |    ✓    |           |           |
| Configuration review    |       |        |    ✓    |           |           |
| Security review         |       |        |         |     ✓     |           |
| Full DR test            |       |        |         |     ✓     |           |
| RPO/RTO validation      |       |        |         |     ✓     |           |
| Major upgrade planning  |       |        |         |     ✓     |           |
| VACUUM FULL             |       |        |         |           |     ✓     |
| REINDEX                 |       |        |         |           |     ✓     |
| CLUSTER                 |       |        |         |           |     ✓     |
| Manual PITR             |       |        |         |           |     ✓     |

## The most important rule

Do **not** convert these 60 subjects into 60 scheduled jobs.

PostgreSQL already performs many maintenance operations dynamically. In particular, autovacuum is designed to react to actual table activity. 

PostgreSQL's own documentation warns that purely fixed vacuum schedules can fail when workload suddenly changes, which is one of the main reasons autovacuum is recommended. ([PostgreSQL][1])

The practical model should be:

```text
                 PostgreSQL DBA Maintenance
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
     Automatic         Monitoring        On-Demand
          |                |                |
     Autovacuum       XID / Bloat       REINDEX
     Autoanalyze      WAL / Disk        VACUUM FULL
     Checkpoints      Locks             CLUSTER
     WAL              Backup            PITR
          |
          v
  DBA tunes and verifies,
  rather than blindly
  scheduling everything
```

This entire roadmap is consistent with PostgreSQL's official administration structure: routine vacuuming/reindexing/log maintenance, backup and PITR, replication, monitoring, disk usage, reliability, checksums, WAL, and server configuration. ([PostgreSQL][5])

[1]: https://www.postgresql.org/docs/18/routine-vacuuming.html "PostgreSQL: Documentation: 18: 24.1. Routine Vacuuming"
[2]: https://www.postgresql.org/docs/18/sql-reindex.html "PostgreSQL: Documentation: 18: REINDEX"
[3]: https://www.postgresql.org/docs/18/monitoring.html "PostgreSQL: Documentation: 18: Chapter 27. Monitoring Database Activity"
[4]: https://www.postgresql.org/docs/18/admin.html "PostgreSQL: Documentation: 18: Part III. Server Administration"
[5]: https://www.postgresql.org/docs/18/backup.html "PostgreSQL: Documentation: 18: Chapter 25. Backup and Restore"
[6]: https://www.postgresql.org/docs/18/maintenance.html? "PostgreSQL: Documentation: 18: Chapter 24. Routine Database Maintenance Tasks"
[7]: https://www.postgresql.org/docs/18/runtime-config.html "PostgreSQL: Documentation: 18: Chapter 19. Server Configuration"

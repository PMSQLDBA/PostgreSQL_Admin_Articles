Below is a **PostgreSQL 18 production-oriented backup/restore matrix**, using the **DVDRental** sample database for practical examples. 

Separating **logical backups**, **physical backups**, **WAL/PITR**, **incremental backups**, and **cluster/global backups**, because they solve different recovery scenarios.

PostgreSQL 18 officially groups backup into three fundamental approaches: **SQL dump, file-system-level backup, and continuous archiving/PITR**. 

PostgreSQL 18 also provides `pg_basebackup`, `pg_combinebackup`, and `pg_verifybackup` for physical/incremental backup workflows. ([PostgreSQL][1])

---

# PostgreSQL 18 Backup & Restore — Complete DBA Guide

## 1. Backup types at a glance

| Backup type           | PostgreSQL tool                       | Scope                 | Restore tool          | PITR?                   | Typical production use     |
| --------------------- | ------------------------------------- | --------------------- | --------------------- | ----------------------- | -------------------------- |
| Plain SQL dump        | `pg_dump -Fp`                         | Database              | `psql`                | ❌                       | Small DB / migration       |
| Custom dump           | `pg_dump -Fc`                         | Database              | `pg_restore`          | ❌                       | Flexible database backup   |
| Directory dump        | `pg_dump -Fd`                         | Database              | `pg_restore`          | ❌                       | Large DB / parallel backup |
| Tar dump              | `pg_dump -Ft`                         | Database              | `pg_restore`          | ❌                       | Portable archive           |
| Schema-only           | `pg_dump --schema-only`               | Schema objects        | `psql` / `pg_restore` | ❌                       | DDL recovery               |
| Data-only             | `pg_dump --data-only`                 | Data                  | `psql` / `pg_restore` | ❌                       | Data refresh               |
| Table backup          | `pg_dump -t`                          | Table                 | `psql` / `pg_restore` | ❌                       | Object-level recovery      |
| Cluster/global backup | `pg_dumpall`                          | Roles/DBs/tablespaces | `psql`                | ❌                       | Cluster metadata           |
| Globals-only          | `pg_dumpall --globals-only`           | Roles/tablespaces     | `psql`                | ❌                       | DR prerequisite            |
| Physical base backup  | `pg_basebackup`                       | Entire cluster        | PostgreSQL startup    | ✅*                      | DR / standby               |
| File-system backup    | Filesystem/storage                    | Entire cluster        | Filesystem restore    | Depends                 | Enterprise backup          |
| WAL archive           | `archive_command` / `archive_library` | WAL                   | Recovery process      | ✅                       | PITR                       |
| PITR                  | Base + WAL                            | Entire cluster        | PostgreSQL recovery   | ✅                       | Disaster recovery          |
| Incremental physical  | `pg_basebackup --incremental`         | Changed blocks        | `pg_combinebackup`    | With WAL                | Large databases            |
| Logical replication   | Publication/subscription              | Selected data         | Replication           | Not conventional backup | Migration/DR/HA            |

* A base backup alone is not equivalent to a complete PITR solution; PITR requires the necessary WAL archive chain. PostgreSQL explicitly distinguishes `pg_dump` logical backups from continuous WAL archiving. ([PostgreSQL][2])

---

# 2. Lab assumptions

I'll use:

```text
Database       : dvdrental
PostgreSQL     : 18
OS             : RHEL/Ubuntu Linux
DB user        : postgres
Port           : 5432
Backup root    : /backup/postgresql
```

Check:

```bash
psql --version
```

Expected:

```text
psql (PostgreSQL) 18.x
```

Connect:

```bash
psql -U postgres -d dvdrental
```

Verify database:

```sql
SELECT current_database(),
       current_user,
       version();
```

---

# 3. Backup #1 — Plain SQL backup

This is the simplest logical backup.

## Backup

```bash
pg_dump -U postgres -d dvdrental \
  -Fp \
  -f /backup/postgresql/dvdrental.sql
```

or:

```bash
pg_dump -U postgres dvdrental \
  > /backup/postgresql/dvdrental.sql
```

Check:

```bash
ls -lh /backup/postgresql/dvdrental.sql
```

---

## Restore

Create a new database:

```bash
createdb -U postgres -T template0 dvdrental_restore
```

Restore:

```bash
psql -U postgres \
     -d dvdrental_restore \
     -f /backup/postgresql/dvdrental.sql
```

Validate:

```bash
psql -U postgres -d dvdrental_restore
```

```sql
SELECT count(*) FROM customer;
SELECT count(*) FROM film;
SELECT count(*) FROM rental;
```

PostgreSQL recommends restoring plain SQL dumps using `psql`; the target database must already exist. ([PostgreSQL][3])

---

# 4. Backup #2 — Custom-format backup

This is one of the most important DBA backup formats.

```bash
pg_dump -U postgres \
  -d dvdrental \
  -Fc \
  -f /backup/postgresql/dvdrental.dump
```

Check:

```bash
file /backup/postgresql/dvdrental.dump
```

---

## View backup contents

Very useful in production:

```bash
pg_restore -l /backup/postgresql/dvdrental.dump
```

You can save the TOC:

```bash
pg_restore -l /backup/postgresql/dvdrental.dump \
  > /backup/postgresql/dvdrental_toc.txt
```

---

## Restore

Create database:

```bash
createdb -U postgres -T template0 dvdrental_restore
```

Restore:

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  /backup/postgresql/dvdrental.dump
```

Validate:

```sql
SELECT count(*) FROM film;
SELECT count(*) FROM customer;
```

Custom archives are restored with `pg_restore`, not `psql`. They allow selective and reordered restoration. ([PostgreSQL][3])

---

# 5. Custom backup with DROP/CREATE during restore

Production-style example:

```bash
pg_restore \
  -U postgres \
  -d postgres \
  --clean \
  --create \
  /backup/postgresql/dvdrental.dump
```

Meaning:

```text
--clean    DROP existing objects
--create   CREATE database
```

This is useful when you want to rebuild `dvdrental` from the archive.

---

# 6. Backup #3 — Directory format

Directory format is extremely important for larger databases because it supports **parallel dump and restore**.

Backup:

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -Fd \
  -f /backup/postgresql/dvdrental_dir
```

Directory:

```bash
ls -lh /backup/postgresql/dvdrental_dir
```

---

## Parallel directory backup

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -Fd \
  -j 4 \
  -f /backup/postgresql/dvdrental_dir
```

`-j 4` = four worker processes.

PostgreSQL 18 documentation specifically notes that **directory format is the only `pg_dump` format supporting parallel dumps**. ([PostgreSQL][4])

---

## Restore

```bash
createdb -U postgres dvdrental_restore
```

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  /backup/postgresql/dvdrental_dir
```

Parallel restore:

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  -j 4 \
  /backup/postgresql/dvdrental_dir
```

---

# 7. Backup #4 — Tar format

Backup:

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -Ft \
  -f /backup/postgresql/dvdrental.tar
```

Restore:

```bash
createdb -U postgres dvdrental_restore
```

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  /backup/postgresql/dvdrental.tar
```

---

# 8. Backup #5 — Schema-only backup

Sometimes you only need the database structure.

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  --schema-only \
  -f /backup/postgresql/dvdrental_schema.sql
```

This contains objects such as:

```text
CREATE TABLE
CREATE SEQUENCE
CREATE INDEX
CREATE VIEW
CREATE FUNCTION
...
```

Restore:

```bash
createdb -U postgres dvdrental_schema_restore
```

```bash
psql -U postgres \
  -d dvdrental_schema_restore \
  -f /backup/postgresql/dvdrental_schema.sql
```

### Real-world use

Useful for:

* Development environment creation
* Schema comparison
* DDL recovery
* Migration preparation
* Pre-production testing

---

# 9. Backup #6 — Data-only backup

Backup only data:

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  --data-only \
  -f /backup/postgresql/dvdrental_data.sql
```

Restore:

```bash
psql \
  -U postgres \
  -d dvdrental_restore \
  -f /backup/postgresql/dvdrental_data.sql
```

Useful when:

```text
Schema already exists
       +
Need only application data
```

---

# 10. Backup #7 — Specific table

For example:

```text
customer
```

Backup:

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -t public.customer \
  -f /backup/postgresql/customer.sql
```

Restore:

```bash
psql \
  -U postgres \
  -d dvdrental_restore \
  -f /backup/postgresql/customer.sql
```

---

# 11. Multiple tables

Example:

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -t public.customer \
  -t public.rental \
  -t public.payment \
  -Fc \
  -f /backup/postgresql/customer_rental_payment.dump
```

Restore:

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  /backup/postgresql/customer_rental_payment.dump
```

---

# 12. Schema-specific backup

For example:

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -n public \
  -Fc \
  -f /backup/postgresql/dvdrental_public.dump
```

Restore:

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  /backup/postgresql/dvdrental_public.dump
```

---

# 13. Backup #8 — Cluster/global objects

`pg_dump` backs up a single database. It does **not** back up cluster-wide objects such as roles and tablespaces.

For that, PostgreSQL provides `pg_dumpall`. ([PostgreSQL][3])

## Full cluster logical dump

```bash
pg_dumpall \
  -U postgres \
  > /backup/postgresql/postgresql_cluster.sql
```

Restore:

```bash
psql \
  -U postgres \
  -f /backup/postgresql/postgresql_cluster.sql \
  postgres
```

---

# 14. Globals-only backup

This is especially important for production DR.

```bash
pg_dumpall \
  -U postgres \
  --globals-only \
  > /backup/postgresql/postgresql_globals.sql
```

It includes things such as:

```text
Roles
Role memberships
Tablespaces
Other cluster-level definitions
```

Restore:

```bash
psql \
  -U postgres \
  -f /backup/postgresql/postgresql_globals.sql \
  postgres
```

### DBA recommendation

For every production PostgreSQL server, maintain:

```text
FULL LOGICAL DATABASE BACKUP
+
GLOBALS BACKUP
+
PHYSICAL BASE BACKUP
+
WAL ARCHIVE
```

---

# 15. Backup #9 — Physical base backup

Now we move from **logical backup** to **physical backup**.

Use:

```bash
pg_basebackup
```

Example:

```bash
mkdir -p /backup/postgresql/base
```

```bash
pg_basebackup \
  -U postgres \
  -D /backup/postgresql/base/dvdrental_base \
  -Fp \
  -X stream \
  -P
```

Important:

`pg_basebackup` backs up the **entire PostgreSQL cluster**, not just `dvdrental`.

Therefore, if the cluster contains:

```text
postgres
dvdrental
template0
template1
```

all are included.

PostgreSQL documents `pg_basebackup` as the standard utility for creating a base backup for continuous archiving/PITR. ([PostgreSQL][1])

---

# 16. Physical base backup using tar

```bash
pg_basebackup \
  -U postgres \
  -D /backup/postgresql/base_tar \
  -Ft \
  -X stream \
  -P
```

You may get files such as:

```text
base.tar
pg_wal.tar
```

---

# 17. Verify physical backup

PostgreSQL 18 provides:

```bash
pg_verifybackup
```

Example:

```bash
pg_verifybackup \
  /backup/postgresql/base/dvdrental_base
```

This is an important production validation step.

PostgreSQL 18 includes `pg_verifybackup` specifically for verifying base-backup integrity. ([PostgreSQL][5])

---

# 18. Restore physical base backup

Suppose PostgreSQL's data directory is:

```bash
/var/lib/pgsql/18/data
```

Stop PostgreSQL:

```bash
sudo systemctl stop postgresql-18
```

Move existing directory:

```bash
sudo mv /var/lib/pgsql/18/data \
        /var/lib/pgsql/18/data_old
```

Create new:

```bash
sudo mkdir -p /var/lib/pgsql/18/data
```

Restore:

```bash
sudo cp -a \
  /backup/postgresql/base/dvdrental_base/. \
  /var/lib/pgsql/18/data/
```

Correct ownership:

```bash
sudo chown -R postgres:postgres \
  /var/lib/pgsql/18/data
```

Start:

```bash
sudo systemctl start postgresql-18
```

Validate:

```bash
pg_lsclusters
```

or:

```bash
psql -U postgres -c "SELECT version();"
```

---

# 19. Backup #10 — WAL archiving

This is the foundation for **PITR**.

Check:

```sql
SHOW wal_level;
SHOW archive_mode;
SHOW archive_command;
```

For PITR, configure:

```conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/postgresql/wal/%f'
```

Create directory:

```bash
sudo mkdir -p /backup/postgresql/wal
sudo chown postgres:postgres /backup/postgresql/wal
```

Restart PostgreSQL because `archive_mode` requires a restart:

```bash
sudo systemctl restart postgresql-18
```

Force WAL switch:

```sql
SELECT pg_switch_wal();
```

Check:

```bash
ls -lh /backup/postgresql/wal
```

You should see WAL segment files.

---

# 20. Backup #11 — Base backup + WAL = PITR

This is the most important production DR architecture.

Conceptually:

```text
             BASE BACKUP
                  |
                  v
        +-------------------+
        | PostgreSQL Cluster|
        +-------------------+
                  |
                  v
             WAL 000001
                  |
                  v
             WAL 000002
                  |
                  v
             WAL 000003
                  |
                  v
             WAL 000004
```

The combination allows recovery to a point in time after the base backup.

PostgreSQL explicitly states that continuous archiving allows recovery to the database state at a selected time after the base backup, provided the required WAL sequence is available. ([PostgreSQL][2])

---

# 21. Practical PITR scenario

Let's create a realistic incident.

At:

```text
10:00 AM
```

you take a base backup.

At:

```text
10:30 AM
```

someone accidentally deletes data.

At:

```text
10:45 AM
```

you discover the problem.

You want:

```text
Restore database to 10:29:59
```

---

## Step 1 — Base backup

```bash
pg_basebackup \
  -U postgres \
  -D /backup/postgresql/base/2026-09-25 \
  -Fp \
  -X stream \
  -P
```

---

## Step 2 — WAL continuously archived

Example:

```text
/backup/postgresql/wal/
```

contains:

```text
000000010000000000000001
000000010000000000000002
000000010000000000000003
...
```

---

# 22. PITR restore configuration

Stop PostgreSQL:

```bash
sudo systemctl stop postgresql-18
```

Replace data directory with base backup.

Then configure recovery target.

PostgreSQL 18 uses:

```conf
restore_command = 'cp /backup/postgresql/wal/%f %p'
```

For example:

```conf
restore_command = 'cp /backup/postgresql/wal/%f %p'
recovery_target_time = '2026-09-25 10:29:59-05'
```

Then start PostgreSQL:

```bash
sudo systemctl start postgresql-18
```

PostgreSQL replays:

```text
Base Backup
     ↓
WAL
     ↓
WAL
     ↓
WAL
     ↓
10:29:59
     ↓
STOP
```

---

# 23. PITR with recovery_target_name

Instead of time, you can define a recovery target name.

Example:

```sql
SELECT pg_create_restore_point('before_bad_deployment');
```

Later:

```conf
recovery_target_name = 'before_bad_deployment'
```

This is extremely useful for planned deployment rollback points.

---

# 24. PITR with LSN

You can also target an LSN.

Find LSN:

```sql
SELECT pg_current_wal_lsn();
```

Example:

```text
0/5A3F120
```

Then:

```conf
recovery_target_lsn = '0/5A3F120'
```

This provides an exact WAL position as the recovery target.

---

# 25. Backup #12 — Incremental physical backup

PostgreSQL 18 adds/contains support for incremental physical backup workflows using:

```text
pg_basebackup --incremental
pg_combinebackup
```

The PostgreSQL 18 client tools explicitly include `pg_combinebackup`, which reconstructs a full backup from an incremental backup and its required dependent backups. ([PostgreSQL][5])

Conceptually:

```text
FULL BASE
   |
   +------ Incremental 1
   |
   +------ Incremental 2
   |
   +------ Incremental 3
```

Instead of repeatedly transferring the entire cluster.

---

## Example

First full backup:

```bash
pg_basebackup \
  -U postgres \
  -D /backup/postgresql/base/full1 \
  -Fp \
  -X stream \
  -P
```

Later an incremental backup can be taken using the appropriate manifest from the previous backup.

Example structure:

```bash
pg_basebackup \
  -U postgres \
  -D /backup/postgresql/inc1 \
  --incremental=/backup/postgresql/base/full1/backup_manifest \
  -Fp \
  -X stream \
  -P
```

Then combine the dependent backups with:

```bash
pg_combinebackup \
  /backup/postgresql/base/full1 \
  /backup/postgresql/inc1 \
  -o /backup/postgresql/reconstructed
```

**Important:** incremental physical backup chains should be tested carefully in your PostgreSQL 18 lab before being adopted for production. The backup manifest and dependency chain are fundamental to reconstruction.

---

# 26. Backup #13 — Table-level backup

Example:

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -t public.customer \
  -Fc \
  -f /backup/postgresql/customer.dump
```

Restore:

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  /backup/postgresql/customer.dump
```

Useful for:

```text
Accidental table drop
Object migration
Development refresh
Individual object recovery
```

---

# 27. Selective restore

Suppose the backup contains:

```text
customer
film
payment
rental
inventory
```

List:

```bash
pg_restore -l dvdrental.dump
```

You can extract/edit the TOC:

```bash
pg_restore -l dvdrental.dump > restore.list
```

Edit:

```bash
vi restore.list
```

Then:

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  -L restore.list \
  dvdrental.dump
```

This is a major advantage of custom/directory archives.

---

# 28. Backup only schema + selected data

Example:

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  --schema-only \
  -f dvdrental_schema.sql
```

Then:

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -t public.customer \
  -t public.film \
  --data-only \
  -f selected_data.sql
```

Restore:

```bash
psql -U postgres -d dvdrental_restore \
     -f dvdrental_schema.sql
```

Then:

```bash
psql -U postgres -d dvdrental_restore \
     -f selected_data.sql
```

---

# 29. Compression

Custom format already provides compression behavior.

Example:

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -Fc \
  -f /backup/postgresql/dvdrental.dump
```

You can also control compression depending on the PostgreSQL 18 options.

Check:

```bash
pg_dump --help
```

---

# 30. Remote backup

Very common production scenario.

Suppose:

```text
DB Server:
192.168.0.145

Backup server:
192.168.0.200
```

Run from backup server:

```bash
pg_dump \
  -h 192.168.0.145 \
  -p 5432 \
  -U postgres \
  -d dvdrental \
  -Fc \
  -f /backup/dvdrental.dump
```

This is one advantage of logical backup: `pg_dump` can connect remotely like a normal PostgreSQL client. ([PostgreSQL][3])

---

# 31. Remote physical backup

```bash
pg_basebackup \
  -h 192.168.0.145 \
  -p 5432 \
  -U postgres \
  -D /backup/base/dvdrental \
  -Fp \
  -X stream \
  -P
```

This requires the appropriate replication privileges and authentication configuration.

---

# 32. Backup using a replication user

Create:

```sql
CREATE ROLE backup_user
WITH LOGIN
REPLICATION
PASSWORD 'StrongPassword';
```

Then:

```bash
pg_basebackup \
  -h 192.168.0.145 \
  -U backup_user \
  -D /backup/base/full \
  -Fp \
  -X stream \
  -P
```

In production, use a dedicated backup identity and secure authentication rather than embedding passwords in scripts.

---

# 33. `pg_dump` vs `pg_basebackup`

This distinction is critical for interviews.

| Feature                 | `pg_dump`                  | `pg_basebackup`             |
| ----------------------- | -------------------------- | --------------------------- |
| Type                    | Logical                    | Physical                    |
| Scope                   | Single DB                  | Entire cluster              |
| Restore                 | `psql`/`pg_restore`        | Filesystem/startup recovery |
| PITR                    | ❌                          | ✅ with WAL                  |
| Cross-platform          | Yes                        | No                          |
| Major-version migration | Yes                        | No                          |
| Selective restore       | Yes                        | No                          |
| Table restore           | Yes                        | No                          |
| HA standby creation     | No                         | Yes                         |
| Large production DR     | Usually not primary method | Yes                         |

PostgreSQL notes that logical dumps are generally portable to newer PostgreSQL versions, whereas physical/file-level backups are server-version-specific. ([PostgreSQL][3])

---

# 34. `pg_dump` vs `pg_dumpall`

| Requirement          | pg_dump | pg_dumpall |
| -------------------- | ------: | ---------: |
| One database         |       ✅ |          ✅ |
| Multiple databases   |       ❌ |          ✅ |
| Roles                |       ❌ |          ✅ |
| Tablespaces          |       ❌ |          ✅ |
| Cluster-wide objects |       ❌ |          ✅ |
| Selective table      |       ✅ |    Limited |
| Custom format        |       ✅ |          ❌ |
| `pg_restore`         |       ✅ |          ❌ |

---

# 35. Logical backup vs PITR

This is one of the most important DBA concepts.

### Logical

```text
pg_dump
   ↓
dvdrental.dump
   ↓
pg_restore
```

You recover the state captured by the dump.

### PITR

```text
Base Backup
     +
WAL archives
     ↓
Recovery
     ↓
Specific point in time
```

Therefore:

> **`pg_dump` cannot replace WAL-based PITR.**

PostgreSQL explicitly states that logical dumps do not contain enough information for WAL replay and therefore cannot be used as part of a continuous-archiving solution. ([PostgreSQL][2])

---

# 36. Real production backup architecture

For a serious PostgreSQL production environment, think in layers:

```text
                 PostgreSQL 18
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
    Logical         Physical          WAL
    Backup          Backup          Archive
        |              |              |
        v              v              v
    pg_dump       pg_basebackup    archive_command
        |              |              |
        +--------------+--------------+
                       |
                       v
                 Backup Storage
                       |
              +--------+--------+
              |                 |
              v                 v
           Local             Remote
          Storage           Object Store
```

---

# 37. Recommended production schedule

A practical example:

| Frequency           | Backup                |
| ------------------- | --------------------- |
| Continuous          | WAL archive           |
| Daily               | Physical base backup  |
| Weekly              | Full logical backup   |
| Daily               | Globals backup        |
| Before deployment   | Restore point         |
| Before major change | Logical/custom backup |
| Monthly             | DR restore test       |

Example:

```text
00:00      Base backup
Continuous WAL archiving
06:00      Logical backup
12:00      Globals backup
18:00      Logical backup
23:00      Backup validation
```

Actual frequency should be determined by **RPO/RTO, database size, WAL generation rate, retention, and storage cost**.

---

# 38. Pre-change restore point

Before a major deployment:

```sql
SELECT pg_create_restore_point(
    'before_dvdrental_application_upgrade'
);
```

Record:

```sql
SELECT pg_current_wal_lsn();
```

Now you have a named point in the WAL history.

---

# 39. Backup validation

Never assume:

```text
backup completed = backup is usable
```

For logical:

```bash
pg_restore -l dvdrental.dump
```

Test restore:

```bash
createdb dvdrental_test_restore
```

```bash
pg_restore \
  -U postgres \
  -d dvdrental_test_restore \
  dvdrental.dump
```

Then:

```sql
SELECT count(*) FROM customer;
SELECT count(*) FROM film;
SELECT count(*) FROM rental;
SELECT count(*) FROM payment;
```

For physical:

```bash
pg_verifybackup /backup/postgresql/base/full
```

---

# 40. DVDRental validation queries

After every restore, run:

```sql
SELECT current_database();

SELECT count(*) AS customers
FROM customer;

SELECT count(*) AS films
FROM film;

SELECT count(*) AS rentals
FROM rental;

SELECT count(*) AS payments
FROM payment;

SELECT count(*) AS inventory
FROM inventory;

SELECT count(*) AS actors
FROM actor;
```

Also:

```sql
SELECT pg_size_pretty(pg_database_size('dvdrental'));
```

And:

```sql
SELECT schemaname,
       count(*) AS tables
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
GROUP BY schemaname
ORDER BY schemaname;
```

---

# 41. Complete backup/restore cheat sheet

### Plain SQL

```bash
pg_dump -d dvdrental > dvdrental.sql
```

Restore:

```bash
psql -d dvdrental_restore -f dvdrental.sql
```

### Custom

```bash
pg_dump -Fc -d dvdrental -f dvdrental.dump
```

Restore:

```bash
pg_restore -d dvdrental_restore dvdrental.dump
```

### Directory

```bash
pg_dump -Fd -j 4 -d dvdrental -f dvdrental_dir
```

Restore:

```bash
pg_restore -j 4 -d dvdrental_restore dvdrental_dir
```

### Tar

```bash
pg_dump -Ft -d dvdrental -f dvdrental.tar
```

Restore:

```bash
pg_restore -d dvdrental_restore dvdrental.tar
```

### Schema only

```bash
pg_dump --schema-only -d dvdrental > schema.sql
```

### Data only

```bash
pg_dump --data-only -d dvdrental > data.sql
```

### Table

```bash
pg_dump -t customer -d dvdrental > customer.sql
```

### Globals

```bash
pg_dumpall --globals-only > globals.sql
```

### Full cluster logical

```bash
pg_dumpall > cluster.sql
```

### Physical

```bash
pg_basebackup \
  -D /backup/base \
  -Fp \
  -X stream \
  -P
```

### Verify

```bash
pg_verifybackup /backup/base
```

### WAL

```sql
SELECT pg_switch_wal();
```

### Restore point

```sql
SELECT pg_create_restore_point('before_change');
```

### PITR

```conf
restore_command = 'cp /backup/wal/%f %p'
recovery_target_time = '2026-09-25 10:29:59-05'
```

### Incremental

```bash
pg_basebackup \
  --incremental=/backup/base/backup_manifest \
  -D /backup/incremental
```

### Combine

```bash
pg_combinebackup \
  /backup/base \
  /backup/incremental \
  -o /backup/reconstructed
```

---

# 42. The DBA decision tree

When a production incident occurs:

```text
                 Need recovery?
                       |
             +---------+---------+
             |                   |
        Whole cluster?       Specific object?
             |                   |
             v                   v
        Physical/PITR        Logical backup
             |                   |
             |            +------+------+
             |            |             |
             |          Table         DB
             |            |             |
             |         pg_dump        pg_dump
             |            |             |
             |         pg_restore/psql
             |
             v
        Need exact time?
             |
            YES
             |
             v
      Base + WAL + PITR
```

---

# 43. Most important interview distinction

If an interviewer asks:

> **"What backup strategy would you implement for a 5 TB PostgreSQL production database?"**

Don't answer:

```text
I will run pg_dump every night.
```

A more technically complete architecture is:

```text
Continuous WAL archiving
          +
Regular physical base backups
          +
Backup verification
          +
Off-host/off-site backup storage
          +
Periodic logical backups where required
          +
Regular restore/DR testing
```

For PostgreSQL 18 specifically, the native toolset now includes the important physical-backup components `pg_basebackup`, `pg_verifybackup`, `pg_combinebackup`, and WAL tooling alongside `pg_dump`/`pg_restore`. ([PostgreSQL][5])

### One important PostgreSQL 18 point

For **major-version migration**, don't try to restore a physical PostgreSQL 17 data directory directly into PostgreSQL 18. PostgreSQL 18's release documentation identifies **dump/restore, `pg_upgrade`, or logical replication** as migration approaches from previous major releases. ([PostgreSQL][6])

---

## Official PostgreSQL 18 references

* [PostgreSQL 18 — Backup and Restore](https://www.postgresql.org/docs/18/backup.html)
* [PostgreSQL 18 — SQL Dump / pg_dump / pg_dumpall](https://www.postgresql.org/docs/18/backup-dump.html)
* [PostgreSQL 18 — pg_dump](https://www.postgresql.org/docs/18/app-pgdump.html)
* [PostgreSQL 18 Documentation](https://www.postgresql.org/docs/18/)

This gives you the **native PostgreSQL 18 backup/restore toolbox**. For your DBA lab, the next useful exercise is to actually execute **10 DVDRental recovery scenarios**: accidental table drop → table restore, database corruption simulation → full restore, `pg_dump` restore, custom selective restore, physical restore, WAL archiving, PITR to timestamp, PITR to named restore point, incremental backup reconstruction, and a complete DR rebuild.

[1]: https://www.postgresql.org/docs/18/backup.html "PostgreSQL: Documentation: 18: Chapter 25. Backup and Restore"
[2]: https://www.postgresql.org/docs/17/continuous-archiving.html "PostgreSQL: Documentation: 17: 25.3. Continuous Archiving and Point-in-Time Recovery (PITR)"
[3]: https://www.postgresql.org/docs/18/backup-dump.html "PostgreSQL: Documentation: 18: 25.1. SQL Dump"
[4]: https://www.postgresql.org/docs/18/app-pgdump.html "PostgreSQL: Documentation: 18: pg_dump"
[5]: https://www.postgresql.org/docs/18/reference-client.html "PostgreSQL: Documentation: 18: PostgreSQL Client Applications"
[6]: https://www.postgresql.org/docs/18/release-18.html "PostgreSQL: Documentation: 18: E.6. Release 18"




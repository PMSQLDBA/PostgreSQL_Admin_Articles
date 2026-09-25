# STRICT EXECUTION RULE — READ THIS FIRST

**LINUX SHELL** = Run at the Rocky Linux prompt, e.g. `[rockylinux@DBLabs ~]$`.

**PGSQL / psql** = Run after entering PostgreSQL, e.g. `postgres=#` or `dvdrental=#`.

**LINUX CONFIG** = Edit a PostgreSQL configuration file from Linux. These are **not SQL commands** and must never be pasted into `psql`.

**Rule:** 

`pg_dump`, `pg_dumpall`, `pg_restore`, `pg_basebackup`, `pg_verifybackup`, `pg_combinebackup`, `createdb`, `psql`, `systemctl`, `mkdir`, `cp`, `mv`, `chown`, `ls`, `scp`, `vi` and similar commands are **Linux shell commands**. 

SQL such as `SELECT`, `CREATE ROLE`, `SHOW`, `ALTER SYSTEM`, and `ANALYZE` is **PGSQL**. 

Configuration parameters such as `archive_mode`, `archive_command`, `restore_command`, and `recovery_target_time` belong in **PostgreSQL configuration**, not in `psql`.

**Your current lab:** 
Rocky Linux / PostgreSQL 18.6 / data directory `/var/lib/pgsql/18/data` / database `dvdrental`. 
Your supplied lab output confirms PostgreSQL 18.6 is running from that data directory.  

---

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
| Physical base backup  | `pg_basebackup`                       | Entire cluster        | Filesystem/startup recovery | Base backup; PITR requires WAL | DR / standby               |
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
OS             : Rocky Linux (user lab)
PostgreSQL     : 18.6
DB user        : postgres
Port           : 5432
Data directory : /var/lib/pgsql/18/data
Backup root    : /backup/postgresql (examples)
```

Check:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
psql --version
```

Expected:

```text
psql (PostgreSQL) 18.x
```

Connect:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
psql -U postgres -d dvdrental
```

Verify database:

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT current_database(),
       current_user,
       version();
```

---

# 3. Backup #1 — Plain SQL backup

This is the simplest logical backup.

## Backup

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump -U postgres -d dvdrental \
  -Fp \
  -f /backup/postgresql/dvdrental.sql
```

or:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump -U postgres dvdrental \
  > /backup/postgresql/dvdrental.sql
```

Check:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
ls -lh /backup/postgresql/dvdrental.sql
```

---

## Restore

Create a new database:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
createdb -U postgres -T template0 dvdrental_restore
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
psql -U postgres \
     -d dvdrental_restore \
     -f /backup/postgresql/dvdrental.sql
```

Validate:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
psql -U postgres -d dvdrental_restore
```

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT count(*) FROM customer;
SELECT count(*) FROM film;
SELECT count(*) FROM rental;
```

PostgreSQL recommends restoring plain SQL dumps using `psql`; the target database must already exist. ([PostgreSQL][3])

---

# 4. Backup #2 — Custom-format backup

This is one of the most important DBA backup formats.

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump -U postgres \
  -d dvdrental \
  -Fc \
  -f /backup/postgresql/dvdrental.dump
```

Check:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
file /backup/postgresql/dvdrental.dump
```

---

## View backup contents

Very useful in production:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore -l /backup/postgresql/dvdrental.dump
```

You can save the TOC:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore -l /backup/postgresql/dvdrental.dump \
  > /backup/postgresql/dvdrental_toc.txt
```

---

## Restore

Create database:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
createdb -U postgres -T template0 dvdrental_restore
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  /backup/postgresql/dvdrental.dump
```

Validate:

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT count(*) FROM film;
SELECT count(*) FROM customer;
```

Custom archives are restored with `pg_restore`, not `psql`. They allow selective and reordered restoration. ([PostgreSQL][3])

---

# 5. Custom backup with DROP/CREATE during restore

Production-style example:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -Fd \
  -f /backup/postgresql/dvdrental_dir
```

Directory:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
ls -lh /backup/postgresql/dvdrental_dir
```

---

## Parallel directory backup

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
createdb -U postgres dvdrental_restore
```

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  /backup/postgresql/dvdrental_dir
```

Parallel restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -Ft \
  -f /backup/postgresql/dvdrental.tar
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
createdb -U postgres dvdrental_restore
```

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  /backup/postgresql/dvdrental.tar
```

---

# 8. Backup #5 — Schema-only backup

Sometimes you only need the database structure.

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
createdb -U postgres dvdrental_schema_restore
```

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  --data-only \
  -f /backup/postgresql/dvdrental_data.sql
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -t public.customer \
  -f /backup/postgresql/customer.sql
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
psql \
  -U postgres \
  -d dvdrental_restore \
  -f /backup/postgresql/customer.sql
```

---

# 11. Multiple tables

Example:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  /backup/postgresql/customer_rental_payment.dump
```

---

# 12. Schema-specific backup

For example:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -n public \
  -Fc \
  -f /backup/postgresql/dvdrental_public.dump
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dumpall \
  -U postgres \
  > /backup/postgresql/postgresql_cluster.sql
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
psql \
  -U postgres \
  -f /backup/postgresql/postgresql_cluster.sql \
  postgres
```

---

# 14. Globals-only backup

This is especially important for production DR.

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_basebackup
```

Example:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
mkdir -p /backup/postgresql/base
```

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_verifybackup
```

Example:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_verifybackup \
  /backup/postgresql/base/dvdrental_base
```

This is an important production validation step.

PostgreSQL 18 includes `pg_verifybackup` specifically for verifying base-backup integrity. ([PostgreSQL][5])

---

# 18. Restore physical base backup

Suppose PostgreSQL's data directory is:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
/var/lib/pgsql/18/data
```

Stop PostgreSQL:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
sudo systemctl stop postgresql-18
```

Move existing directory:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
sudo mv /var/lib/pgsql/18/data \
        /var/lib/pgsql/18/data_old
```

Create new:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
sudo mkdir -p /var/lib/pgsql/18/data
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
sudo cp -a \
  /backup/postgresql/base/dvdrental_base/. \
  /var/lib/pgsql/18/data/
```

Correct ownership:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
sudo chown -R postgres:postgres \
  /var/lib/pgsql/18/data
```

Start:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
sudo systemctl start postgresql-18
```

Validate:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_lsclusters
```

or:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
psql -U postgres -c "SELECT version();"
```

---

# 19. Backup #10 — WAL archiving

This is the foundation for **PITR**.

Check:

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SHOW wal_level;
SHOW archive_mode;
SHOW archive_command;
```

For PITR, configure:

**RUN ON: LINUX — edit PostgreSQL configuration (`postgresql.conf` / recovery configuration)**

```conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/postgresql/wal/%f'
```

Create directory:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
sudo mkdir -p /backup/postgresql/wal
sudo chown postgres:postgres /backup/postgresql/wal
```

Restart PostgreSQL because `archive_mode` requires a restart:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
sudo systemctl restart postgresql-18
```

Force WAL switch:

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT pg_switch_wal();
```

Check:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
sudo systemctl stop postgresql-18
```

Replace data directory with base backup.

Then configure recovery target.

PostgreSQL 18 uses:

**RUN ON: LINUX — edit PostgreSQL configuration (`postgresql.conf` / recovery configuration)**

```conf
restore_command = 'cp /backup/postgresql/wal/%f %p'
```

For example:

**RUN ON: LINUX — edit PostgreSQL configuration (`postgresql.conf` / recovery configuration)**

```conf
restore_command = 'cp /backup/postgresql/wal/%f %p'
recovery_target_time = '2026-09-25 10:29:59-05'
```

Then start PostgreSQL:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT pg_create_restore_point('before_bad_deployment');
```

Later:

**RUN ON: LINUX — edit PostgreSQL configuration (`postgresql.conf` / recovery configuration)**

```conf
recovery_target_name = 'before_bad_deployment'
```

This is extremely useful for planned deployment rollback points.

---

# 24. PITR with LSN

You can also target an LSN.

Find LSN:

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT pg_current_wal_lsn();
```

Example:

```text
0/5A3F120
```

Then:

**RUN ON: LINUX — edit PostgreSQL configuration (`postgresql.conf` / recovery configuration)**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -t public.customer \
  -Fc \
  -f /backup/postgresql/customer.dump
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore -l /backup/postgresql/dvdrental.dump
```

You can extract/edit the TOC:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore -l /backup/postgresql/dvdrental.dump > restore.list
```

Edit:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
vi /backup/postgresql/restore.list
```

Then:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore \
  -U postgres \
  -d dvdrental_restore \
  -L /backup/postgresql/restore.list \
  dvdrental.dump
```

This is a major advantage of custom/directory archives.

---

# 28. Backup only schema + selected data

Example:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  --schema-only \
  -f dvdrental_schema.sql
```

Then:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
psql -U postgres -d dvdrental_restore \
     -f dvdrental_schema.sql
```

Then:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
psql -U postgres -d dvdrental_restore \
     -f selected_data.sql
```

---

# 29. Compression

Custom format already provides compression behavior.

Example:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump \
  -U postgres \
  -d dvdrental \
  -Fc \
  -f /backup/postgresql/dvdrental.dump
```

You can also control compression depending on the PostgreSQL 18 options.

Check:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
CREATE ROLE backup_user
WITH LOGIN
REPLICATION
PASSWORD 'StrongPassword';
```

Then:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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
| One database         |       ✅ |           |
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

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT pg_create_restore_point(
    'before_dvdrental_application_upgrade'
);
```

Record:

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore -l /backup/postgresql/dvdrental.dump
```

Test restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
createdb dvdrental_test_restore
```

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore \
  -U postgres \
  -d dvdrental_test_restore \
  dvdrental.dump
```

Then:

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT count(*) FROM customer;
SELECT count(*) FROM film;
SELECT count(*) FROM rental;
SELECT count(*) FROM payment;
```

For physical:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_verifybackup /backup/postgresql/base/full
```

---

# 40. DVDRental validation queries

After every restore, run:

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

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

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT pg_size_pretty(pg_database_size('dvdrental'));
```

And:

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

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

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump -d dvdrental > dvdrental.sql
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
psql -d dvdrental_restore -f dvdrental.sql
```

### Custom

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump -Fc -d dvdrental -f dvdrental.dump
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore -d dvdrental_restore dvdrental.dump
```

### Directory

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump -Fd -j 4 -d dvdrental -f dvdrental_dir
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore -j 4 -d dvdrental_restore dvdrental_dir
```

### Tar

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump -Ft -d dvdrental -f dvdrental.tar
```

Restore:

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_restore -d dvdrental_restore dvdrental.tar
```

### Schema only

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump --schema-only -d dvdrental > schema.sql
```

### Data only

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump --data-only -d dvdrental > data.sql
```

### Table

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dump -t customer -d dvdrental > customer.sql
```

### Globals

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dumpall --globals-only > globals.sql
```

### Full cluster logical

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_dumpall > cluster.sql
```

### Physical

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_basebackup \
  -D /backup/base \
  -Fp \
  -X stream \
  -P
```

### Verify

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_verifybackup /backup/base
```

### WAL

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT pg_switch_wal();
```

### Restore point

**RUN ON: PGSQL / `psql` — NOT at the Linux shell**

```sql
SELECT pg_create_restore_point('before_change');
```

### PITR

**RUN ON: LINUX — edit PostgreSQL configuration (`postgresql.conf` / recovery configuration)**

```conf
restore_command = 'cp /backup/wal/%f %p'
recovery_target_time = '2026-09-25 10:29:59-05'
```

### Incremental

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

```bash
pg_basebackup \
  --incremental=/backup/base/backup_manifest \
  -D /backup/incremental
```

### Combine

**RUN ON: LINUX SHELL (Rocky Linux) — NOT inside `psql`**

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

---

# 44. Lab correction — Windows → Rocky Linux file transfer

Your lab transcript shows this failed:

**RUN ON: LINUX SHELL**

```bash
scp "T:\4_PGSQL_DBA\PGSQLLABS\dvdrental\dvdrental.tar" rockylinux@192.168.223.129:/backup/dvdrental/
```

The reason is important: that command was being executed **on Rocky Linux**, so Linux interpreted `T:` as a hostname/path component. The error `Could not resolve hostname t` confirms this. 

If `dvdrental.tar` is on the Windows host, run `scp` **from Windows PowerShell**, not from Rocky Linux:

**RUN ON: WINDOWS POWERSHELL**

```powershell
scp "T:\4_PGSQL_DBA\PGSQLLABS\dvdrental\dvdrental.tar" rockylinux@192.168.223.129:/backup/dvdrental/
```

Then verify the file on Rocky Linux:

**RUN ON: LINUX SHELL**

```bash
ls -lh /backup/dvdrental/
```

Your supplied lab output shows the archive ultimately existed on Rocky Linux at `/backup/dvdrental/dvdrental.tar` and was 2.8 MB. 

---

# 45. Lab correction — `\l`, `\dt` and SQL

These are **psql meta-commands**, so they are run **inside `psql`**, not at the Linux shell.

**RUN ON: PGSQL / `psql`**

```sql
\l
\c dvdrental
\dt
```

Your transcript shows `\dt` working when executed through `psql`, while typing `\l` at the Linux prompt caused `bash: l: command not found`. 

Correct Linux entry into psql:

**RUN ON: LINUX SHELL**

```bash
sudo -u postgres psql
```

Then the prompt changes to:

**PGSQL / `psql`**

```text
postgres=#
```

Exit psql:

**RUN ON: PGSQL / `psql`**

```sql
\q
```

---

# 46. Lab correction — `sudo` syntax

These are Linux commands.

**RUN ON: LINUX SHELL**

```bash
sudo -u postgres psql
sudo -u postgres createdb -T template0 dvdrental
sudo -u postgres pg_restore --dbname=dvdrental --no-owner --no-acl --verbose /backup/dvdrental/dvdrental.tar
```

Do **not** type `sudo -u postgres` by itself expecting a shell. Do not type `sudo - u postgres`; the space between `-` and `u` is invalid. Your transcript shows both forms producing errors. 

---

# 47. Strict command classification — quick reference

| Command / syntax | Run where |
|---|---|
| `psql --version` | **LINUX SHELL** |
| `sudo systemctl status postgresql-18` | **LINUX SHELL** |
| `sudo systemctl start/stop/restart postgresql-18` | **LINUX SHELL** |
| `createdb` | **LINUX SHELL** |
| `pg_dump` | **LINUX SHELL** |
| `pg_dumpall` | **LINUX SHELL** |
| `pg_restore` | **LINUX SHELL** |
| `pg_basebackup` | **LINUX SHELL** |
| `pg_verifybackup` | **LINUX SHELL** |
| `pg_combinebackup` | **LINUX SHELL** |
| `mkdir`, `cp`, `mv`, `chown`, `ls`, `vi` | **LINUX SHELL** |
| `scp` | **LINUX SHELL** or **Windows PowerShell**, depending on where the source file exists |
| `SHOW ...;` | **PGSQL / `psql`** |
| `SELECT ...;` | **PGSQL / `psql`** |
| `CREATE ROLE ...;` | **PGSQL / `psql`** |
| `ANALYZE;` | **PGSQL / `psql`** |
| `SELECT pg_switch_wal();` | **PGSQL / `psql`** |
| `SELECT pg_create_restore_point(...);` | **PGSQL / `psql`** |
| `\l`, `\dt`, `\c`, `\q` | **PGSQL / `psql` meta-commands** |
| `archive_mode = on` | **LINUX CONFIG** |
| `archive_command = ...` | **LINUX CONFIG** |
| `restore_command = ...` | **LINUX CONFIG** |
| `recovery_target_time = ...` | **LINUX CONFIG** |

---

# 48. Important corrections to the original notes

1. **`pg_dumpall` does not produce a custom/tar/directory archive.** It produces SQL text and is restored with `psql`.

2. **`pg_basebackup` backs up the whole PostgreSQL cluster**, not only `dvdrental`. Your original note correctly states this; keep it.

3. **A base backup by itself is not PITR.** PITR requires the base backup plus the required continuous WAL archive chain.

4. **`pg_dump` is logical; `pg_basebackup` is physical.** Do not describe them as interchangeable.

5. **PostgreSQL 12+ recovery uses `recovery.signal` / `standby.signal` mechanics.** Do not teach old `recovery.conf`-style instructions as if they were PostgreSQL 18 instructions.

6. **`archive_mode` requires a server restart when enabling it.** `archive_command` is a normal configuration parameter and does not by itself require a restart.

7. **`pg_verifybackup` verifies a `pg_basebackup` backup**, not a logical `pg_dump` archive.

8. **`pg_restore` does not restore plain `.sql` files.** Use `psql` for plain SQL dumps.

9. **`pg_restore` can restore TAR, custom, and directory archives.**

10. **`pg_dump -Fd` is the format that supports parallel dumping with `-j`.**

11. **`psql`, `pg_dump`, and `pg_restore` are client programs launched from Linux/Windows shells.** The fact that they connect to PostgreSQL does not make them SQL commands.

12. **`\l`, `\dt`, `\c`, and `\q` are psql meta-commands**, not SQL and not Linux commands.

---

# 49. Recommended notation for all future PGSQL lab notes

Use this exact pattern so there is never ambiguity:

### STEP X — Description

**RUN ON: LINUX SHELL**

```bash
sudo -u postgres psql -d dvdrental
```

**RUN ON: PGSQL / `psql`**

```sql
SELECT current_database();
```

**RUN ON: LINUX CONFIG**

```conf
archive_mode = on
archive_command = 'cp %p /backup/postgresql/wal/%f'
```

**RUN ON: LINUX SHELL**

```bash
sudo systemctl restart postgresql-18
```

This is the convention to use throughout the DBA notes.

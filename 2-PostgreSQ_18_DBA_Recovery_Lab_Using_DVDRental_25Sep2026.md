**Hands-on PostgreSQL 18 DBA recovery lab** using `dvdrental`.

**current PostgreSQL 18 behavior**, including the PostgreSQL 18 incremental-backup workflow with `pg_basebackup --incremental` + `pg_combinebackup`. 

PostgreSQL 18 requires the earlier backups in the incremental chain to remain available, and `pg_verifybackup` validates the backup against its manifest but does **not** replace an actual test restore. ([PostgreSQL][1])

# PostgreSQL 18 — DVDRental Backup & Restore Practical Lab

## Lab objectives

You will perform these **10 real-world DBA scenarios**:

| #  | Scenario                           | Backup             | Recovery               |
| -- | ---------------------------------- | ------------------ | ---------------------- |
| 1  | Accidental table deletion          | Custom dump        | Table/object restore   |
| 2  | Complete database logical recovery | Custom dump        | `pg_restore`           |
| 3  | Schema-only recovery               | Schema dump        | `psql`                 |
| 4  | Data-only recovery                 | Data dump          | `psql`                 |
| 5  | Selective object recovery          | Custom dump        | `pg_restore -t`        |
| 6  | Complete cluster physical recovery | `pg_basebackup`    | Physical restore       |
| 7  | WAL archiving                      | WAL                | WAL replay             |
| 8  | Point-in-time recovery             | Base + WAL         | PITR                   |
| 9  | Named restore point                | Base + WAL         | Restore-point recovery |
| 10 | PostgreSQL 18 incremental backup   | Full + incremental | `pg_combinebackup`     |

---

# PART 1 — Prepare the DVDRental Lab

## 1. Check PostgreSQL

```bash
psql --version
```

Expected:

```text
psql (PostgreSQL) 18.x
```

Check server:

```bash
sudo -u postgres psql -c "SELECT version();"
```

---

# 2. Confirm DVDRental

```bash
sudo -u postgres psql -l
```

Connect:

```bash
sudo -u postgres psql -d dvdrental
```

Run:

```sql
SELECT current_database();

SELECT count(*) FROM customer;
SELECT count(*) FROM film;
SELECT count(*) FROM rental;
SELECT count(*) FROM payment;
```

---

# 3. Create lab backup directories

```bash
sudo mkdir -p /backup/pg18/{logical,physical,wal,pitr,incremental,restore}
```

Set ownership:

```bash
sudo chown -R postgres:postgres /backup/pg18
```

Verify:

```bash
ls -ld /backup/pg18/*
```

You should have:

```text
logical
physical
wal
pitr
incremental
restore
```

---

# SCENARIO 1

# Accidental Table Deletion

This is one of the most common DBA recovery scenarios.

Suppose someone accidentally drops:

```text
customer
```

---

## Step 1 — Take a backup

Create a custom-format backup:

```bash
sudo -u postgres pg_dump \
  -d dvdrental \
  -Fc \
  -f /backup/pg18/logical/dvdrental_full.dump
```

Verify:

```bash
sudo -u postgres pg_restore \
  -l /backup/pg18/logical/dvdrental_full.dump
```

---

## Step 2 — Simulate disaster

Connect:

```bash
sudo -u postgres psql -d dvdrental
```

Check:

```sql
SELECT count(*) FROM customer;
```

Now intentionally drop the table:

```sql
DROP TABLE customer CASCADE;
```

Exit:

```sql
\q
```

---

## Step 3 — Confirm disaster

```bash
sudo -u postgres psql -d dvdrental -c \
"SELECT count(*) FROM customer;"
```

Expected:

```text
ERROR: relation "customer" does not exist
```

---

## Step 4 — Restore only customer

First extract the table from the archive.

```bash
sudo -u postgres pg_restore \
  -d dvdrental \
  -t customer \
  /backup/pg18/logical/dvdrental_full.dump
```

However, because `customer` has dependencies, a **production DBA should first inspect the archive TOC** rather than blindly restoring a single table.

```bash
sudo -u postgres pg_restore \
  -l /backup/pg18/logical/dvdrental_full.dump \
  | grep customer
```

For a clean lab, recreate the required object through a selective restore or restore into a separate database first.

---

# SCENARIO 2

# Complete Logical Database Restore

Now simulate complete database loss.

## Step 1 — Drop database

From another database:

```bash
sudo -u postgres psql -d postgres
```

```sql
DROP DATABASE dvdrental;
```

---

## Step 2 — Create empty database

```bash
sudo -u postgres createdb dvdrental
```

---

## Step 3 — Restore

```bash
sudo -u postgres pg_restore \
  -d dvdrental \
  /backup/pg18/logical/dvdrental_full.dump
```

---

## Step 4 — Validate

```bash
sudo -u postgres psql -d dvdrental
```

```sql
SELECT count(*) FROM customer;

SELECT count(*) FROM film;

SELECT count(*) FROM rental;

SELECT count(*) FROM payment;
```

Also:

```sql
SELECT pg_size_pretty(pg_database_size('dvdrental'));
```

---

# SCENARIO 3

# Schema-Only Backup & Restore

This simulates:

> "I need the database structure but not the data."

---

## Backup

```bash
sudo -u postgres pg_dump \
  -d dvdrental \
  --schema-only \
  -f /backup/pg18/logical/dvdrental_schema.sql
```

Check:

```bash
head -50 /backup/pg18/logical/dvdrental_schema.sql
```

---

## Create test database

```bash
sudo -u postgres createdb dvdrental_schema_test
```

---

## Restore

```bash
sudo -u postgres psql \
  -d dvdrental_schema_test \
  -f /backup/pg18/logical/dvdrental_schema.sql
```

Check tables:

```bash
sudo -u postgres psql -d dvdrental_schema_test -c \
"\dt"
```

You should see the tables but no application data.

---

# SCENARIO 4

# Data-Only Backup & Restore

This is useful when the schema already exists.

## Backup

```bash
sudo -u postgres pg_dump \
  -d dvdrental \
  --data-only \
  -f /backup/pg18/logical/dvdrental_data.sql
```

Create test database:

```bash
sudo -u postgres createdb dvdrental_data_test
```

Create schema first:

```bash
sudo -u postgres pg_dump \
  -d dvdrental \
  --schema-only \
  | sudo -u postgres psql -d dvdrental_data_test
```

Then load data:

```bash
sudo -u postgres psql \
  -d dvdrental_data_test \
  -f /backup/pg18/logical/dvdrental_data.sql
```

Validate:

```bash
sudo -u postgres psql -d dvdrental_data_test -c \
"SELECT count(*) FROM customer;"
```

---

# SCENARIO 5

# Selective Object Recovery

Suppose:

```text
customer
film
rental
payment
```

exist in the backup, but you only need `payment`.

List archive:

```bash
sudo -u postgres pg_restore \
  -l /backup/pg18/logical/dvdrental_full.dump
```

Search:

```bash
sudo -u postgres pg_restore \
  -l /backup/pg18/logical/dvdrental_full.dump \
  | grep payment
```

Restore selectively:

```bash
sudo -u postgres pg_restore \
  -d dvdrental_restore \
  -t payment \
  /backup/pg18/logical/dvdrental_full.dump
```

This is one of the major advantages of **custom/directory archives** over plain SQL dumps.

---

# SCENARIO 6

# Physical Full Backup

Now move to **physical backup**.

Important:

> `pg_basebackup` backs up the **entire PostgreSQL cluster**, not only `dvdrental`.

PostgreSQL 18 provides `pg_basebackup` for physical cluster backups and `pg_verifybackup` for manifest-based validation. ([PostgreSQL][2])

---

## Step 1 — Check cluster

```bash
sudo -u postgres psql -c \
"SELECT current_database(), version();"
```

---

## Step 2 — Create physical backup

```bash
sudo -u postgres pg_basebackup \
  -D /backup/pg18/physical/full_01 \
  -Fp \
  -X stream \
  -P
```

You should see progress similar to:

```text
waiting for checkpoint
...
XXX/XXX kB (100%), 1/1 tablespace
```

---

# Step 3 — Check manifest

```bash
ls -lh /backup/pg18/physical/full_01/
```

You should see:

```text
PG_VERSION
backup_label
backup_manifest
base/
global/
pg_wal/
...
```

---

# Step 4 — Verify

```bash
sudo -u postgres pg_verifybackup \
  /backup/pg18/physical/full_01
```

Expected:

```text
backup successfully verified
```

`pg_verifybackup` verifies the files against `backup_manifest`, but PostgreSQL explicitly recommends performing actual test restores as an additional validation step. ([PostgreSQL][3])

---

# SCENARIO 7

# Configure WAL Archiving

Now we build the foundation for PITR.

## Step 1 — Check current configuration

```bash
sudo -u postgres psql -c \
"SHOW archive_mode;"
```

```bash
sudo -u postgres psql -c \
"SHOW archive_command;"
```

```bash
sudo -u postgres psql -c \
"SHOW wal_level;"
```

---

# Step 2 — Configure WAL archive

Find configuration:

```sql
SHOW config_file;
```

Example:

```text
/etc/postgresql/18/main/postgresql.conf
```

or on RHEL:

```text
/var/lib/pgsql/18/data/postgresql.conf
```

Add:

```conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/pg18/wal/%f'
```

---

# Step 3 — Create archive directory

```bash
sudo mkdir -p /backup/pg18/wal
sudo chown postgres:postgres /backup/pg18/wal
```

---

# Step 4 — Restart

Because `archive_mode` requires a server restart:

```bash
sudo systemctl restart postgresql
```

Verify:

```bash
sudo -u postgres psql -c \
"SHOW archive_mode;"
```

Expected:

```text
on
```

---

# Step 5 — Force WAL switch

```sql
SELECT pg_switch_wal();
```

Check:

```bash
ls -lh /backup/pg18/wal
```

You should now see WAL segment files.

---

# SCENARIO 8

# Point-in-Time Recovery

This is the most important production exercise.

We'll deliberately create:

```text
10:00  Base backup
10:10  Good data
10:20  Bad change
10:30  Disaster discovered
```

We want:

```text
Restore to 10:19:59
```

---

# Step 1 — Take base backup

```bash
sudo -u postgres pg_basebackup \
  -D /backup/pg18/pitr/base_01 \
  -Fp \
  -X stream \
  -P
```

Verify:

```bash
sudo -u postgres pg_verifybackup \
  /backup/pg18/pitr/base_01
```

---

# Step 2 — Insert a test record

Connect:

```bash
sudo -u postgres psql -d dvdrental
```

Create a dedicated lab table:

```sql
CREATE TABLE backup_lab_test
(
    id          integer PRIMARY KEY,
    description text,
    created_at  timestamptz DEFAULT clock_timestamp()
);
```

Insert:

```sql
INSERT INTO backup_lab_test(id, description)
VALUES
(1, 'GOOD DATA'),
(2, 'GOOD DATA');
```

Force WAL:

```sql
SELECT pg_switch_wal();
```

---

# Step 3 — Create a restore point

```sql
SELECT pg_create_restore_point('GOOD_STATE');
```

Record the result.

Example:

```text
0/7000120
```

This gives us a known recovery target.

---

# Step 4 — Simulate bad transaction

```sql
INSERT INTO backup_lab_test(id, description)
VALUES
(3, 'BAD DATA'),
(4, 'BAD DATA'),
(5, 'BAD DATA');
```

Force WAL:

```sql
SELECT pg_switch_wal();
```

Now:

```sql
SELECT *
FROM backup_lab_test
ORDER BY id;
```

You should see:

```text
1 GOOD DATA
2 GOOD DATA
3 BAD DATA
4 BAD DATA
5 BAD DATA
```

---

# Step 5 — Simulate disaster

```sql
DROP TABLE backup_lab_test;
```

Now the table is gone.

---

# Step 6 — Stop PostgreSQL

```bash
sudo systemctl stop postgresql
```

---

# Step 7 — Preserve failed cluster

Never immediately destroy the original data directory in a real incident.

For the lab:

```bash
sudo mv /var/lib/postgresql/18/main \
        /var/lib/postgresql/18/main_failed
```

Use the actual `PGDATA` path from:

```bash
sudo -u postgres psql -c "SHOW data_directory;"
```

before shutdown.

---

# Step 8 — Restore base backup

```bash
sudo mkdir -p /var/lib/postgresql/18/main
```

```bash
sudo cp -a \
  /backup/pg18/pitr/base_01/. \
  /var/lib/postgresql/18/main/
```

Fix ownership:

```bash
sudo chown -R postgres:postgres \
  /var/lib/postgresql/18/main
```

---

# Step 9 — Configure recovery

In `postgresql.conf`:

```conf
restore_command = 'cp /backup/pg18/wal/%f %p'
recovery_target_name = 'GOOD_STATE'
```

Create:

```bash
sudo -u postgres touch \
  /var/lib/postgresql/18/main/recovery.signal
```

---

# Step 10 — Start PostgreSQL

```bash
sudo systemctl start postgresql
```

PostgreSQL will:

```text
Read base backup
      ↓
Read WAL
      ↓
Replay transactions
      ↓
Find GOOD_STATE
      ↓
Stop recovery
```

---

# Step 11 — Validate

```bash
sudo -u postgres psql -d dvdrental
```

```sql
SELECT *
FROM backup_lab_test
ORDER BY id;
```

Expected:

```text
1 GOOD DATA
2 GOOD DATA
```

The bad rows:

```text
3
4
5
```

should not exist because recovery stopped at the named restore point.

---

# SCENARIO 9

# PITR Using Timestamp

Instead of:

```conf
recovery_target_name
```

you can use:

```conf
recovery_target_time
```

Example:

```conf
restore_command = 'cp /backup/pg18/wal/%f %p'

recovery_target_time = '2026-09-25 10:19:59-05'
```

Then:

```bash
sudo -u postgres touch \
  /var/lib/postgresql/18/main/recovery.signal
```

Start:

```bash
sudo systemctl start postgresql
```

This is the classic:

> "Recover the database to 10:19:59, immediately before the bad transaction."

---

# SCENARIO 10

# PostgreSQL 18 Incremental Backup

This is particularly important because PostgreSQL 18 introduced the native incremental physical backup workflow.

The workflow is:

```text
FULL BACKUP
     |
     v
INCREMENTAL BACKUP 1
     |
     v
INCREMENTAL BACKUP 2
     |
     v
pg_combinebackup
     |
     v
SYNTHETIC FULL BACKUP
```

PostgreSQL 18's `pg_basebackup --incremental` uses the previous backup's manifest, while `pg_combinebackup` reconstructs a full backup from the full + dependent incremental backups. ([PostgreSQL][1])

---

# Step 1 — Take full backup

```bash
sudo -u postgres pg_basebackup \
  -D /backup/pg18/incremental/full_01 \
  -Fp \
  -X stream \
  -P
```

---

# Step 2 — Verify full backup

```bash
sudo -u postgres pg_verifybackup \
  /backup/pg18/incremental/full_01
```

---

# Step 3 — Take incremental backup

Use the full backup's manifest:

```bash
sudo -u postgres pg_basebackup \
  -D /backup/pg18/incremental/inc_01 \
  --incremental=/backup/pg18/incremental/full_01/backup_manifest \
  -Fp \
  -X stream \
  -P
```

The option is `--incremental`, and PostgreSQL 18 requires the backup manifest of the reference backup. ([PostgreSQL][4])

---

# Step 4 — Make more changes

For example:

```bash
sudo -u postgres psql -d dvdrental
```

```sql
INSERT INTO backup_lab_test
VALUES
(10, 'Incremental backup test 1'),
(11, 'Incremental backup test 2');
```

---

# Step 5 — Take another incremental backup

The second incremental backup must use the appropriate reference manifest from the backup chain.

For example:

```bash
sudo -u postgres pg_basebackup \
  -D /backup/pg18/incremental/inc_02 \
  --incremental=/backup/pg18/incremental/inc_01/backup_manifest \
  -Fp \
  -X stream \
  -P
```

---

# Step 6 — Reconstruct full backup

```bash
sudo -u postgres pg_combinebackup \
  /backup/pg18/incremental/full_01 \
  /backup/pg18/incremental/inc_01 \
  /backup/pg18/incremental/inc_02 \
  -o /backup/pg18/incremental/reconstructed_full
```

This is the critical PostgreSQL 18 concept:

> **An incremental backup cannot simply be pointed at PostgreSQL as if it were a standalone full backup.**

You reconstruct the required full backup using `pg_combinebackup`. PostgreSQL also requires you to retain all earlier backups on which the incremental backup depends. ([PostgreSQL][1])

---

# Step 7 — Verify reconstructed backup

```bash
sudo -u postgres pg_verifybackup \
  /backup/pg18/incremental/reconstructed_full
```

---

# Step 8 — Test restore

For the actual DR test:

```text
Stop PostgreSQL
       ↓
Preserve failed PGDATA
       ↓
Restore reconstructed_full
       ↓
Configure WAL recovery
       ↓
Start PostgreSQL
       ↓
Replay WAL
       ↓
Validate
```

---

# BONUS — Backup Globals

This should be part of your production backup strategy.

```bash
sudo -u postgres pg_dumpall \
  --globals-only \
  > /backup/pg18/logical/globals.sql
```

Restore:

```bash
sudo -u postgres psql \
  -d postgres \
  -f /backup/pg18/logical/globals.sql
```

This protects cluster-level objects such as roles and tablespaces that aren't included in an individual `pg_dump`.

---

# BONUS — Complete logical cluster dump

```bash
sudo -u postgres pg_dumpall \
  > /backup/pg18/logical/cluster_full.sql
```

Restore:

```bash
sudo -u postgres psql \
  -d postgres \
  -f /backup/pg18/logical/cluster_full.sql
```

---

# Production DBA Backup Matrix

This is the matrix I would use in your PostgreSQL DBA SOP.

| Requirement                | Recommended mechanism                           |
| -------------------------- | ----------------------------------------------- |
| Single DB backup           | `pg_dump -Fc`                                   |
| Table/object recovery      | `pg_dump -Fc` + `pg_restore`                    |
| Schema-only                | `pg_dump --schema-only`                         |
| Data-only                  | `pg_dump --data-only`                           |
| Multiple DB logical backup | `pg_dumpall`                                    |
| Roles/tablespaces          | `pg_dumpall --globals-only`                     |
| Full physical backup       | `pg_basebackup`                                 |
| Standby creation           | `pg_basebackup`                                 |
| PITR                       | Base backup + WAL                               |
| Continuous DR              | Base backup + continuous WAL archive            |
| PostgreSQL 18 incremental  | `pg_basebackup --incremental`                   |
| Incremental reconstruction | `pg_combinebackup`                              |
| Backup validation          | `pg_verifybackup`                               |
| Actual recovery validation | Test restore                                    |
| Exact transaction recovery | PITR/LSN/restore point                          |
| Major-version migration    | `pg_upgrade` / logical migration / dump-restore |

The PostgreSQL 18 client toolset officially includes `pg_dump`, `pg_dumpall`, `pg_restore`, `pg_basebackup`, `pg_combinebackup`, `pg_verifybackup`, `pg_receivewal`, and related recovery utilities. ([PostgreSQL][2])

---

# Production Backup Architecture

For a real production PostgreSQL 18 environment, think of it as:

```text
                         PostgreSQL 18
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
       Logical Backup    Physical Backup   WAL Archive
             |                |                |
          pg_dump        pg_basebackup    Continuous
             |                |                |
             +----------------+----------------+
                              |
                              v
                     Backup Repository
                              |
                  +-----------+-----------+
                  |                       |
                  v                       v
             Local storage          Remote storage
                                          |
                                          v
                                   Off-site / Object
                                      Storage
```

And recovery:

```text
                    FAILURE
                       |
          +------------+------------+
          |                         |
          v                         v
    Object/database           Entire cluster
       problem                    problem
          |                         |
          v                         v
    pg_restore                Base backup
                                  +
                                WAL
                                  |
                                  v
                                PITR
```

## The 10 exercises you should actually perform

I recommend you execute them in this exact order:

```text
LAB 01  Custom pg_dump
        ↓
LAB 02  Full pg_restore
        ↓
LAB 03  Schema-only
        ↓
LAB 04  Data-only
        ↓
LAB 05  Selective table restore
        ↓
LAB 06  pg_basebackup
        ↓
LAB 07  WAL archiving
        ↓
LAB 08  PITR by timestamp
        ↓
LAB 09  PITR by named restore point
        ↓
LAB 10  PostgreSQL 18 incremental backup
```

**Do not skip LAB 07–10.** Those are the exercises that move you from knowing `pg_dump` syntax to understanding **production PostgreSQL DR/PITR**, which is the substantially more important DBA skill.

One PostgreSQL 18-specific correction to the earlier material: incremental backup support is **not merely a conceptual/proposed feature** in v18; `pg_basebackup --incremental` and `pg_combinebackup` are documented PostgreSQL 18 functionality. ([PostgreSQL][1])

[1]: https://www.postgresql.org/docs/18/app-pgcombinebackup.html "PostgreSQL: Documentation: 18: pg_combinebackup"
[2]: https://www.postgresql.org/docs/18/reference-client.html "PostgreSQL: Documentation: 18: PostgreSQL Client Applications"
[3]: https://www.postgresql.org/docs/18/app-pgverifybackup.html "PostgreSQL: Documentation: 18: pg_verifybackup"
[4]: https://www.postgresql.org/docs/17/app-pgbasebackup.html "PostgreSQL: Documentation: 17: pg_basebackup"

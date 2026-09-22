**PostgreSQL 18 PITR end-to-end note**, using your Rocky Linux/PostgreSQL 18.6 lab paths and commands. 

# PostgreSQL 18 — PITR Notes: Beginning to End

## 1. PITR Overview

**PITR = Point-In-Time Recovery.**

PITR allows you to restore a PostgreSQL database to a specific point in time by combining:

```text
Base Backup
     +
Archived WAL Files
     +
Recovery Target
     =
Point-In-Time Recovery
```

In your lab:

```text
PostgreSQL Version : 18.6
OS                 : Rocky Linux
PGDATA             : /var/lib/pgsql/18/data
Base Backups       : /var/lib/pgsql/18/database_backups
WAL Archives       : /var/lib/pgsql/18/wal_archives
Service            : postgresql-18
```

Your WAL archive directory contained sequential WAL files such as `000000010000000000000007`, `...008`, and `...009`, demonstrating the archived WAL stream used for recovery. 

---

# 2. Create Backup Directories

Create the directories:

```bash
mkdir -p /var/lib/pgsql/18/database_backups
mkdir -p /var/lib/pgsql/18/wal_archives
```

Set ownership:

```bash
chown -R postgres:postgres /var/lib/pgsql/18/database_backups
chown -R postgres:postgres /var/lib/pgsql/18/wal_archives
```

Set permissions:

```bash
chmod 700 /var/lib/pgsql/18/database_backups
chmod 700 /var/lib/pgsql/18/wal_archives
```

Verify:

```bash
ls -ld /var/lib/pgsql/18/database_backups
ls -ld /var/lib/pgsql/18/wal_archives
```

---

# 3. Configure WAL Archiving

Edit:

```bash
vi /var/lib/pgsql/18/data/postgresql.conf
```

Configure:

```conf
archive_mode = on

archive_command =
'test ! -f /var/lib/pgsql/18/wal_archives/%f && cp %p /var/lib/pgsql/18/wal_archives/%f'
```

Your lab configuration used this archive command. 

Restart PostgreSQL:

```bash
systemctl restart postgresql-18
```

Verify:

```bash
sudo -u postgres psql -c "SHOW archive_mode;"
sudo -u postgres psql -c "SHOW archive_command;"
```

Expected:

```text
archive_mode
------------
on
```

---

# 4. Verify WAL Archiving

Run:

```bash
ls -lh /var/lib/pgsql/18/wal_archives/
```

You should see WAL files:

```text
000000010000000000000001
000000010000000000000002
000000010000000000000003
...
```

You may also see `.backup` history files generated around base-backup operations. Your lab produced such files alongside archived WAL segments. 

---

# 5. Create the Base Backup

Create a timestamped backup directory:

```bash
BACKUP_DIR="/var/lib/pgsql/18/database_backups/base_$(date +%Y%m%d_%H%M%S)"
```

Create it:

```bash
mkdir -p "$BACKUP_DIR"
chown postgres:postgres "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"
```

Create the physical base backup:

```bash
sudo -u postgres pg_basebackup \
-D "$BACKUP_DIR" \
-F p \
-P \
-X stream
```

Parameters:

| Option      | Meaning                  |
| ----------- | ------------------------ |
| `-D`        | Backup destination       |
| `-F p`      | Plain directory format   |
| `-P`        | Show progress            |
| `-X stream` | Stream WAL during backup |

---

# 6. Verify the Base Backup

Check:

```bash
du -sh "$BACKUP_DIR"
```

Check contents:

```bash
ls -lah "$BACKUP_DIR"
```

You should have a PostgreSQL data-directory structure containing items such as:

```text
PG_VERSION
base/
global/
pg_wal/
postgresql.conf
...
```

Your lab base-backup directory was approximately 40 MB and contained the PostgreSQL data-directory structure.

---

# 7. Create Test Data

Connect:

```bash
sudo -u postgres psql
```

Create/use your PITR test database:

```sql
\c pitr_lab
```

Example table:

```sql
CREATE TABLE customer_orders
(
    order_id      SERIAL PRIMARY KEY,
    customer_name VARCHAR(100),
    product_name  VARCHAR(200),
    amount        NUMERIC(12,2),
    order_time    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Insert initial data:

```sql
INSERT INTO customer_orders
(customer_name, product_name, amount)
VALUES
('Praveen', 'PostgreSQL Training', 5000.00),
('John', 'Linux Training', 3000.00),
('David', 'DBA Training', 4500.00);
```

Verify:

```sql
SELECT * FROM customer_orders
ORDER BY order_id;
```

---

# 8. Establish the PITR Recovery Point

At the desired recovery point, record the current timestamp:

```sql
SELECT CURRENT_TIMESTAMP;
```

In your lab, the target was:

```text
2026-09-22 18:40:17
```

The test rows were inserted shortly before that timestamp. 

---

# 9. Generate Data After the Recovery Point

After recording the target time, insert additional data that you intentionally want PITR to remove.

Example:

```sql
INSERT INTO customer_orders
(customer_name, product_name, amount)
VALUES
('BAD-DATA', 'Transaction after recovery point', 99999.99);
```

Your lab used exactly this type of post-recovery-point test transaction. 

Verify:

```sql
SELECT *
FROM customer_orders
ORDER BY order_id;
```

At this point:

```text
Rows 1-6 = desired data
Row 7     = unwanted post-recovery-point data
```

---

# 10. Stop PostgreSQL

```bash
systemctl stop postgresql-18
```

Your lab uses the PostgreSQL 18 service name:

```text
postgresql-18.service
```

rather than:

```text
postgresql.service
```

The service is configured/enabled as `postgresql-18`. 

---

# 11. Preserve the Existing Data Directory

For a physical PITR, do **not** overwrite your original data blindly.

A common lab approach is:

```bash
mv /var/lib/pgsql/18/data \
   /var/lib/pgsql/18/data_before_pitr
```

Then recreate:

```bash
mkdir -p /var/lib/pgsql/18/data
chown postgres:postgres /var/lib/pgsql/18/data
chmod 700 /var/lib/pgsql/18/data
```

---

# 12. Restore the Base Backup

Copy the base backup into `PGDATA`:

```bash
cp -a /var/lib/pgsql/18/database_backups/base_YYYYMMDD_HHMMSS/. \
/var/lib/pgsql/18/data/
```

Set ownership:

```bash
chown -R postgres:postgres /var/lib/pgsql/18/data
```

Set permissions:

```bash
chmod 700 /var/lib/pgsql/18/data
```

---

# 13. Configure WAL Restore

Edit:

```bash
vi /var/lib/pgsql/18/data/postgresql.conf
```

Add:

```conf
restore_command = 'cp /var/lib/pgsql/18/wal_archives/%f %p'
```

This tells PostgreSQL how to retrieve archived WAL files during recovery.

The meaning is:

```text
%f = requested WAL filename
%p = destination path where PostgreSQL wants the WAL
```

Therefore:

```text
PostgreSQL requests WAL
        ↓
restore_command
        ↓
/var/lib/pgsql/18/wal_archives/%f
        ↓
copy to %p
```

---

# 14. Configure the PITR Target

For your specific lab:

```conf
recovery_target_time = '2026-09-22 18:40:17'
```

So the relevant recovery configuration is:

```conf
restore_command = 'cp /var/lib/pgsql/18/wal_archives/%f %p'

recovery_target_time = '2026-09-22 18:40:17'
```

---

# 15. Create `recovery.signal`

Create the recovery trigger file:

```bash
touch /var/lib/pgsql/18/data/recovery.signal
```

Set ownership:

```bash
chown postgres:postgres /var/lib/pgsql/18/data/recovery.signal
chmod 600 /var/lib/pgsql/18/data/recovery.signal
```

This tells PostgreSQL to enter archive recovery when it starts.

Verify:

```bash
ls -l /var/lib/pgsql/18/data/recovery.signal
```

---

# 16. Start PostgreSQL and Perform PITR

Start:

```bash
systemctl start postgresql-18
```

Monitor:

```bash
tail -f /var/lib/pgsql/18/data/log/*.log
```

PostgreSQL will:

```text
Start with restored base backup
          ↓
Read WAL
          ↓
Execute WAL records
          ↓
Reach recovery_target_time
          ↓
Stop recovery
```

---

# 17. Verify PITR Completion

Check:

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

When recovery has completed:

```text
pg_is_in_recovery
------------------
f
```

`f` means:

```text
PostgreSQL is NOT in recovery
```

You can also check:

```bash
sudo -u postgres psql -c "SELECT pg_current_wal_lsn();"
```

---

# 18. Validate the Recovered Database

Connect:

```bash
sudo -u postgres psql -d pitr_lab
```

Run:

```sql
SELECT *
FROM customer_orders
ORDER BY order_id;
```

Expected result:

```text
Praveen
John
David
Alice
Robert
Michael
```

The intentionally inserted:

```text
BAD-DATA
```

should no longer exist.

Your lab validation showed the six intended rows remained after PITR. 

---

# POST-PITR COMPLETION — IMPORTANT STEPS

This is the section you specifically asked for.

Once PITR has successfully reached the desired recovery point, you need to transition the recovered instance from **recovery configuration** to **normal PostgreSQL operation**.

## Step 1 — Confirm recovery is complete

Run:

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

Expected:

```text
f
```

**Do this before changing recovery configuration.**

---

## Step 2 — Verify the recovered data

```bash
sudo -u postgres psql -d pitr_lab -c \
"SELECT * FROM customer_orders ORDER BY order_id;"
```

Confirm the unwanted post-target transaction is gone.

---

## Step 3 — Remove the recovery trigger

After recovery is complete:

```bash
rm -f /var/lib/pgsql/18/data/recovery.signal
```

Verify:

```bash
ls -l /var/lib/pgsql/18/data/recovery.signal
```

Expected:

```text
No such file or directory
```

---

## Step 4 — Remove the PITR target

In `postgresql.conf`, remove/comment:

```conf
recovery_target_time = '2026-09-22 18:40:17'
```

For example:

```conf
#recovery_target_time = '2026-09-22 18:40:17'
```

You do not want the old PITR target configured for a future normal restart.

---

## Step 5 — Decide about `restore_command`

For a normal primary database that is **no longer performing archive recovery**, you do not need `restore_command` for normal WAL archiving.

Your normal WAL archiving is controlled separately by:

```conf
archive_mode = on

archive_command =
'test ! -f /var/lib/pgsql/18/wal_archives/%f && cp %p /var/lib/pgsql/18/wal_archives/%f'
```

Therefore, distinguish:

```text
archive_command
     ↓
Normal operation
     ↓
Archives WAL
```

from:

```text
restore_command
     ↓
Recovery
     ↓
Retrieves archived WAL
```

For your lab, after confirming `pg_is_in_recovery() = f`, you can comment out the PITR-specific `restore_command`:

```conf
#restore_command = 'cp /var/lib/pgsql/18/wal_archives/%f %p'
```

---

# Step 6 — Keep WAL Archiving Enabled

Do **not** confuse `restore_command` with `archive_command`.

Keep:

```conf
archive_mode = on
```

and your normal:

```conf
archive_command = 'test ! -f /var/lib/pgsql/18/wal_archives/%f && cp %p /var/lib/pgsql/18/wal_archives/%f'
```

This allows future WAL files to continue being archived for another PITR.

Your lab already had `archive_mode=on` and an `archive_command` configured. 

---

# Step 7 — Restart PostgreSQL

After removing the completed PITR configuration:

```bash
systemctl restart postgresql-18
```

Check:

```bash
systemctl status postgresql-18 --no-pager
```

Expected:

```text
Active: active (running)
```

---

# Step 8 — Verify Normal Mode

Run:

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

Expected:

```text
f
```

Then:

```bash
sudo -u postgres psql -c "SHOW archive_mode;"
```

Expected:

```text
on
```

And:

```bash
sudo -u postgres psql -c "SHOW archive_command;"
```

Verify your WAL archiving command is still present.

---

# Step 9 — Verify WAL Archiving Continues

Run:

```bash
ls -lh /var/lib/pgsql/18/wal_archives/
```

Generate some WAL:

```bash
sudo -u postgres psql -d pitr_lab -c \
"INSERT INTO customer_orders(customer_name, product_name, amount)
 VALUES ('PITR-Test', 'Post Recovery Test', 1000);"
```

Then:

```bash
sudo -u postgres psql -c "SELECT pg_switch_wal();"
```

Check:

```bash
ls -ltrh /var/lib/pgsql/18/wal_archives/ | tail
```

You should see newly archived WAL as archiving continues.

---

# Step 10 — Take a New Base Backup

This is an important operational step.

After PITR, the recovered database is now your new database state.

Take a fresh base backup:

```bash
BACKUP_DIR="/var/lib/pgsql/18/database_backups/post_pitr_$(date +%Y%m%d_%H%M%S)"

mkdir -p "$BACKUP_DIR"

chown postgres:postgres "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

sudo -u postgres pg_basebackup \
-D "$BACKUP_DIR" \
-F p \
-P \
-X stream
```

Verify:

```bash
du -sh "$BACKUP_DIR"
```

---

# Step 11 — Document the Recovery

Record:

```text
PITR Date:
PITR Target Time:
Base Backup Used:
First WAL Required:
Last WAL Replayed:
Recovery Completion Time:
Recovered Database:
Validation Result:
New Base Backup:
```

For your lab:

```text
PITR Target:
2026-09-22 18:40:17

Database:
pitr_lab

Expected:
BAD-DATA transaction removed
```

---

# Final Post-PITR Checklist

Use this as your DBA checklist:

```text
[ ] PITR completed successfully
[ ] pg_is_in_recovery() = false
[ ] Recovered data validated
[ ] recovery.signal removed
[ ] recovery_target_time removed/commented
[ ] restore_command removed/commented if no longer required for recovery
[ ] archive_mode remains ON
[ ] archive_command remains configured
[ ] PostgreSQL restarted successfully
[ ] pg_is_in_recovery() still false
[ ] WAL archiving verified
[ ] New WAL generated successfully
[ ] New base backup taken
[ ] Recovery details documented
```

### The key concept

After successful PITR, your configuration should conceptually transition from:

```text
              PITR
               │
               ▼
restore_command
recovery_target_time
recovery.signal
               │
               ▼
       Recovery completes
               │
               ▼
       pg_is_in_recovery()
               │
               ▼
              false
               │
               ▼
      Normal PostgreSQL
               │
       ┌───────┴────────┐
       ▼                ▼
archive_command     New base backup
       │
       ▼
Future PITR capability
```

Your lab's base backup/WAL architecture is therefore **not discarded after PITR**. 
The recovered database becomes the current database state, while the normal WAL-archiving process should continue so that you can perform another PITR in the future.

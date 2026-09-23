# PostgreSQL DBA — Complete Job-Ready & Production Lab Notes

## SQL Server DBA → Senior PostgreSQL DBA

### PostgreSQL 18 | Linux | Backup/PITR | HA | Patroni | Performance | Security | PgBouncer | Monitoring

---

# TABLE OF CONTENTS

1. PostgreSQL DBA Learning Strategy
2. SQL Server → PostgreSQL Mental Map
3. 30-Day PostgreSQL DBA Roadmap
4. Week 1 — Linux + PostgreSQL Architecture & Internals
5. Week 2 — Backup, Restore & PITR
6. Week 3 — Streaming Replication + HA + Patroni
7. Week 4 — Performance + Security + PgBouncer + Monitoring
8. Advanced Production DBA Scenarios
9. PostgreSQL 18 HA Lab Architecture
10. PostgreSQL 18 Streaming Replication Lab
11. Patroni + etcd HA Lab
12. HAProxy Application Routing
13. pgBackRest Backup/PITR Layer
14. Production Troubleshooting Methodology
15. Senior DBA Interview Topics
16. 30-Day Completion Checklist
17. Final Enterprise DBA Capstone

---

# 1. POSTGRESQL DBA LEARNING STRATEGY

For someone coming from a strong SQL Server DBA background, the objective is **not** to relearn generic DBA concepts.

Focus on PostgreSQL-specific implementation.

### Highest-value areas

| Priority | Area                     | Hands-on requirement                           |
| -------: | ------------------------ | ---------------------------------------------- |
|        1 | Architecture & Internals | Processes, memory, WAL, MVCC, locks, storage   |
|        2 | Backup & PITR            | `pg_basebackup`, WAL archive, pgBackRest, PITR |
|        3 | HA & Replication         | Streaming replication, slots, sync/async       |
|        4 | Patroni                  | DCS, leader election, failover, switchover     |
|        5 | Performance              | `EXPLAIN`, indexes, statistics, VACUUM         |
|        6 | Linux                    | RHEL/Ubuntu, systemd, storage, networking      |
|        7 | Cloud                    | AWS RDS/Aurora, Azure PostgreSQL               |
|        8 | Security                 | Roles, `pg_hba.conf`, SSL, auditing            |
|        9 | Automation               | Bash, Python, Ansible, Terraform               |
|       10 | Migration                | SQL Server/Oracle → PostgreSQL                 |

PostgreSQL's own documentation separates backup/recovery, WAL, replication, HA, and administration into dedicated operational areas, which is a useful structure for your DBA training. ([PostgreSQL][1])

---

# 2. SQL SERVER → POSTGRESQL MENTAL MAP

| SQL Server             | PostgreSQL                                                      |
| ---------------------- | --------------------------------------------------------------- |
| SQL Server instance    | PostgreSQL server/cluster                                       |
| Database               | Database                                                        |
| Schema                 | Schema                                                          |
| Login                  | Role with `LOGIN`                                               |
| User                   | Role/user identity within database                              |
| SQL Agent              | cron/systemd/external scheduler                                 |
| Transaction log        | WAL                                                             |
| LSN                    | LSN                                                             |
| SPID                   | Backend PID                                                     |
| DMVs                   | `pg_stat_*` + system catalogs                                   |
| Blocking               | Locks/waiting sessions                                          |
| Always On AG           | Streaming replication + HA orchestration                        |
| AG automatic failover  | Patroni/other HA tooling                                        |
| AG listener            | HAProxy/load-balancing/routing layer                            |
| Log shipping           | WAL shipping/archive/recovery                                   |
| Full backup            | Logical or physical backup depending on requirement             |
| Differential backup    | pgBackRest differential                                         |
| Incremental backup     | pgBackRest incremental                                          |
| `RESTORE DATABASE`     | PostgreSQL restore/recovery process                             |
| Transaction-log backup | WAL archiving                                                   |
| PIT restore            | PITR                                                            |
| `DBCC CHECKDB`         | No single direct equivalent                                     |
| TempDB                 | Temporary relations/files; no direct equivalent                 |
| Filegroup              | Tablespace/physical storage concepts                            |
| Index rebuild          | `REINDEX`, maintenance strategies                               |
| Statistics             | PostgreSQL planner statistics                                   |
| Query Store            | No exact one-to-one equivalent; `pg_stat_statements` is central |
| Extended Events        | Logging/extensions/observability tooling                        |
| Windows Event Log      | systemd journal / PostgreSQL logs                               |
| Registry/configuration | PostgreSQL configuration files/settings                         |

### Critical distinction

Do **not** say:

> PostgreSQL cluster = SQL Server Failover Cluster.

They are different concepts.

A PostgreSQL cluster is a PostgreSQL data directory/server instance containing databases and cluster-wide objects.

---

# 3. 30-DAY POSTGRESQL DBA ROADMAP

## Week 1

```text
Linux
 ↓
Architecture
 ↓
Configuration
 ↓
Processes
 ↓
Memory
 ↓
WAL
 ↓
MVCC
 ↓
Locks
```

## Week 2

```text
Logical backup
 ↓
Physical backup
 ↓
WAL archiving
 ↓
pgBackRest
 ↓
PITR
 ↓
Recovery validation
```

## Week 3

```text
Streaming replication
 ↓
Replication monitoring
 ↓
Slots
 ↓
Sync/Async
 ↓
Patroni
 ↓
etcd
 ↓
Switchover
 ↓
Failover
```

## Week 4

```text
EXPLAIN
 ↓
Indexes
 ↓
Statistics
 ↓
VACUUM
 ↓
Bloat
 ↓
Security
 ↓
PgBouncer
 ↓
Monitoring
```

---

# WEEK 1

# LINUX + POSTGRESQL CORE & INTERNALS

---

# DAY 1 — LINUX FOR POSTGRESQL DBA

## 1. Linux filesystem

Know:

```text
/
├── etc
├── var
│   ├── log
│   └── lib
├── tmp
├── home
├── usr
├── opt
├── dev
├── proc
└── sys
```

Important DBA locations:

| Directory  | Purpose                    |
| ---------- | -------------------------- |
| `/etc`     | Configuration              |
| `/var/log` | Logs                       |
| `/var/lib` | Application/database data  |
| `/tmp`     | Temporary files            |
| `/dev`     | Devices                    |
| `/proc`    | Process/kernel information |
| `/sys`     | Kernel/device information  |

Commands:

```bash
pwd
ls
ls -lah
cd
mkdir
touch
cp
mv
rm
find
```

---

# 2. Linux ownership

PostgreSQL normally runs under:

```text
postgres
```

Check:

```bash
id postgres
```

Check ownership:

```bash
ls -ld /var/lib/pgsql
```

Permissions:

```text
r = 4
w = 2
x = 1
```

Examples:

```bash
chmod 750 directory
chmod 640 file
```

Ownership:

```bash
chown postgres:postgres file
```

Never casually change permissions on `$PGDATA`.

---

# 3. Linux users

```bash
whoami
id
sudo -l
```

Switch to PostgreSQL OS account:

```bash
sudo -iu postgres
```

Important distinction:

```text
Linux root
Linux postgres
PostgreSQL superuser
```

These are different security identities.

---

# 4. Processes

```bash
ps -ef
ps aux
pgrep -a postgres
top
```

Typical PostgreSQL processes include:

```text
postgres
checkpointer
background writer
walwriter
autovacuum launcher
logical replication launcher
client backend
```

Mental model:

```text
Linux process
      ↓
PostgreSQL backend
      ↓
Database session
```

---

# 5. CPU

```bash
top
uptime
lscpu
```

Check process CPU:

```bash
ps -eo pid,ppid,%cpu,%mem,cmd --sort=-%cpu | head
```

Interpret load average relative to CPU count.

---

# 6. Memory

```bash
free -h
vmstat 1
```

Important `vmstat` values:

```text
si = swap in
so = swap out
```

Persistent heavy swapping is a performance warning.

---

# 7. Disk

```bash
df -h
df -i
du -sh *
lsblk
mount
```

I/O:

```bash
iostat -xz 1
```

Install tools if required.

RHEL:

```bash
sudo dnf install sysstat
```

Ubuntu:

```bash
sudo apt install sysstat
```

---

# 8. Networking

```bash
ip addr
ip route
hostname
hostname -I
ss -lntp
```

Check PostgreSQL:

```bash
ss -lntp | grep 5432
```

Test:

```bash
nc -zv <server-ip> 5432
```

---

# 9. systemd

Depending on the installation, PostgreSQL may expose a service such as:

```bash
systemctl status postgresql
```

Commands:

```bash
systemctl start postgresql
systemctl stop postgresql
systemctl restart postgresql
systemctl reload postgresql
systemctl enable postgresql
```

On Debian/Ubuntu packaged installations, cluster-specific utilities may also be available:

```bash
pg_lsclusters
pg_ctlcluster
```

Your existing Ubuntu PostgreSQL 18 lab uses this cluster-oriented model.

---

# 10. Logs

```bash
journalctl
journalctl -u postgresql
journalctl -u postgresql -n 100
journalctl -u postgresql -f
```

Application/PostgreSQL log location depends on configuration.

---

# 11. Text processing

```bash
cat
less
more
head
tail
grep
awk
sed
cut
sort
uniq
```

Examples:

```bash
grep -i error postgresql.log
tail -f postgresql.log
grep -i listen_addresses postgresql.conf
```

---

# DAY 1 TROUBLESHOOTING FLOW

If PostgreSQL is unavailable:

```text
Server reachable?
      ↓
OS healthy?
      ↓
PostgreSQL process running?
      ↓
Service/cluster online?
      ↓
Listening on 5432?
      ↓
Firewall/network?
      ↓
Authentication?
      ↓
Configuration?
      ↓
Logs?
      ↓
Filesystem?
```

Commands:

```bash
hostname
hostname -I
uptime
df -h
free -h
ps -ef | grep postgres
ss -lntp | grep 5432
systemctl status postgresql
journalctl -u postgresql -n 50
```

---

# DAY 2 — POSTGRESQL ARCHITECTURE

Architecture:

```text
PostgreSQL Server
       |
       +-- Cluster
             |
             +-- Database
             |     |
             |     +-- Schema
             |           |
             |           +-- Table
             |           +-- Index
             |           +-- View
             |           +-- Function
             |
             +-- Database
```

PostgreSQL documentation describes streaming replication in terms of **primary and standby servers**, with the primary sending WAL and the standby receiving/replaying it. ([PostgreSQL][2])

---

# Cluster

On Ubuntu:

```bash
pg_lsclusters
```

Example:

```text
Ver Cluster Port Status Owner    Data directory
18  main    5432 online postgres /var/lib/postgresql/18/main
```

Understand:

```text
18      = PostgreSQL version
main    = cluster name
5432    = port
online  = state
postgres = OS owner
data directory = PGDATA
```

---

# psql

Connect:

```bash
sudo -iu postgres
psql
```

Useful commands:

```sql
\l
\c postgres
\dn
\du
\dt
\d table_name
\q
```

---

# DAY 3 — CONFIGURATION

Main configuration:

```text
postgresql.conf
pg_hba.conf
```

## postgresql.conf

Examples:

```text
port
listen_addresses
max_connections
shared_buffers
work_mem
maintenance_work_mem
effective_cache_size
wal_level
max_wal_size
checkpoint_timeout
archive_mode
archive_command
autovacuum
logging
```

Check:

```sql
SHOW port;
SHOW listen_addresses;
SHOW max_connections;
SHOW shared_buffers;
SHOW wal_level;
```

---

# pg_hba.conf

Structure:

```text
TYPE DATABASE USER ADDRESS METHOD
```

Example:

```text
host all all 192.168.1.0/24 scram-sha-256
```

Connection types:

```text
local
host
hostssl
hostnossl
```

Authentication methods include:

```text
peer
scram-sha-256
trust
reject
```

### Critical rule

`pg_hba.conf` is processed in order and the **first matching rule** controls the connection.

---

# DAY 4 — POSTGRESQL PROCESSES & MEMORY

Architecture:

```text
                  PostgreSQL
                      |
          +-----------+-----------+
          |                       |
     Shared Memory             Processes
          |                       |
    shared_buffers          Backend processes
    WAL buffers              Checkpointer
                             WAL writer
                             Autovacuum
                             Background writer
```

Check:

```sql
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW effective_cache_size;
SHOW max_connections;
```

Don't memorize arbitrary tuning percentages.

Learn:

```text
parameter
   ↓
purpose
   ↓
workload impact
   ↓
symptoms
   ↓
safe tuning approach
```

---

# DAY 5 — WAL

**WAL = Write-Ahead Logging**

Core principle:

```text
Change
  ↓
WAL record
  ↓
WAL flushed
  ↓
Data pages eventually written
```

PostgreSQL can recover committed changes from WAL because WAL records describing changes are flushed before the corresponding data changes need to be persisted. WAL is also fundamental to online backup, PITR and replication. ([PostgreSQL][3])

Check:

```sql
SELECT pg_current_wal_lsn();
```

WAL directory:

```text
$PGDATA/pg_wal
```

Normally WAL segments are 16 MB, although the segment size can be changed at cluster initialization. ([PostgreSQL][4])

---

# LSN

**LSN = Log Sequence Number**

It represents a position in WAL.

```sql
SELECT pg_current_wal_lsn();
```

LSNs can be compared to determine WAL distance and are used to measure replication/recovery progress. ([PostgreSQL][4])

---

# DAY 6 — MVCC

**MVCC = Multi-Version Concurrency Control**

Conceptually:

```text
UPDATE
  ↓
new row version
  ↓
old version remains
  ↓
visibility rules
  ↓
VACUUM eventually cleans/reclaims reusable space
```

Learn:

```text
Transaction ID
Snapshot
Tuple
Visibility
Dead tuples
VACUUM
Autovacuum
Isolation
```

Production chain:

```text
UPDATE/DELETE
      ↓
dead tuples
      ↓
VACUUM/autovacuum
      ↓
space reuse
```

---

# DAY 7 — LOCKS & BLOCKING

Active sessions:

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity;
```

Locks:

```sql
SELECT
    pid,
    locktype,
    mode,
    granted
FROM pg_locks;
```

Create a blocking test.

### Session 1

```sql
BEGIN;

UPDATE performance_lab
SET amount = amount + 10
WHERE id = 100;
```

Do not commit.

### Session 2

```sql
UPDATE performance_lab
SET amount = amount + 20
WHERE id = 100;
```

Session 2 waits.

Investigate:

```text
Blocked PID
     ↓
Lock wait
     ↓
Blocking PID
     ↓
Blocking transaction
     ↓
Root cause
```

Terminate only when appropriate:

```sql
SELECT pg_terminate_backend(<pid>);
```

---

# WEEK 1 COMPLETION GATE

You should explain:

```text
Client
 ↓
Connection
 ↓
Backend process
 ↓
Transaction
 ↓
MVCC snapshot
 ↓
SQL execution
 ↓
WAL
 ↓
Locks
 ↓
Commit
 ↓
Checkpoint/data pages
 ↓
VACUUM
```

---

# WEEK 2

# BACKUP, RESTORE & PITR

PostgreSQL 18 documentation identifies three fundamental backup approaches:

```text
SQL dump
File-system-level backup
Continuous archiving
```

([PostgreSQL][1])

---

# DAY 8 — LOGICAL BACKUP

## pg_dump

Create:

```sql
CREATE DATABASE backup_lab;
```

Example:

```bash
pg_dump -d backup_lab -Fc -f /tmp/backup_lab.dump
```

`-Fc`:

```text
Custom format
```

Restore with:

```bash
pg_restore -d backup_restore_lab /tmp/backup_lab.dump
```

---

# Backup formats

| Format    | Option | Restore      |
| --------- | ------ | ------------ |
| Plain SQL | `-Fp`  | `psql`       |
| Custom    | `-Fc`  | `pg_restore` |
| Directory | `-Fd`  | `pg_restore` |
| Tar       | `-Ft`  | `pg_restore` |

---

# pg_dumpall

```bash
pg_dumpall > /backup/cluster.sql
```

Useful for cluster-wide SQL representation including roles and database definitions.

---

# DAY 9 — RESTORE

Create:

```sql
CREATE DATABASE backup_restore_lab;
```

Restore:

```bash
pg_restore \
  -d backup_restore_lab \
  /tmp/backup_lab.dump
```

Validate:

```sql
SELECT * FROM customers;
```

---

# Logical vs Physical Backup

| Logical                     | Physical         |
| --------------------------- | ---------------- |
| Object/data representation  | Cluster files    |
| `pg_dump`                   | `pg_basebackup`  |
| `pg_dumpall`                | pgBackRest       |
| Selective restore           | Cluster recovery |
| Migration                   | DR/PITR          |
| Version migration scenarios | Standby creation |

---

# DAY 10 — pg_basebackup

`pg_basebackup` creates a physical base backup of the PostgreSQL cluster.

Example:

```bash
pg_basebackup \
  -h <PRIMARY_IP> \
  -U replicator \
  -D /backup/basebackup \
  -Fp \
  -Xs \
  -P
```

Important:

```text
-D = destination
-Fp = plain format
-Xs = stream WAL
-P = progress
```

PostgreSQL 18 also provides backup manifests and `pg_verifybackup` for verification of `pg_basebackup` backups. ([PostgreSQL][5])

---

# Verify base backup

If a manifest was generated:

```bash
pg_verifybackup /backup/basebackup
```

But:

> Verification is not the same as a test restore.

PostgreSQL explicitly recommends test restores because `pg_verifybackup` cannot reproduce every check a running PostgreSQL server will perform. ([PostgreSQL][5])

---

# DAY 11 — WAL ARCHIVING

Check:

```sql
SHOW wal_level;
SHOW archive_mode;
SHOW archive_command;
```

Concept:

```text
PostgreSQL
    |
    v
pg_wal
    |
    v
archive_command
    |
    v
WAL Repository
```

Example lab configuration:

```text
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
```

`archive_mode` requires a restart.

---

# WAL ARCHIVING FAILURE

Monitor:

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

If:

```text
failed_count ↑
```

investigate:

```text
Permissions
Disk
archive_command
Destination
Network
Storage
```

Never simply delete WAL from `pg_wal` to solve disk pressure.

---

# DAY 12 — PITR

**PITR = Point-In-Time Recovery**

Required:

```text
Base backup
+
Required WAL
+
Recovery target
```

Example:

```text
02:00 base backup
10:20 bad transaction
10:30 incident discovered

Goal:
Recover to immediately before 10:20
```

Architecture:

```text
Base Backup
     +
WAL Archive
     |
     v
Recovery Server
     |
     v
WAL Replay
     |
     v
Recovery Target
```

PostgreSQL documents continuous archiving/PITR as restoring a prior physical backup and replaying WAL until the desired point. ([PostgreSQL][1])

---

# Recovery targets

Know:

```text
recovery_target_time
recovery_target_lsn
recovery_target_name
recovery_target_xid
recovery_target_immediate
```

### PostgreSQL 18 correction

Do **not** use old:

```text
recovery.conf
standby_mode
trigger_file
```

`recovery.conf` was removed in PostgreSQL 12. PostgreSQL 12+ uses normal configuration parameters plus signal files such as `standby.signal` and `recovery.signal`. Promotion is performed with `pg_ctl promote` or `pg_promote()`. ([PostgreSQL][6])

---

# DAY 13 — pgBackRest

pgBackRest is the production-oriented backup/restore tool in this roadmap.

Architecture:

```text
PostgreSQL
     |
     +---- Full
     +---- Differential
     +---- Incremental
     |
     +---- WAL
            |
            v
        pgBackRest
            |
            v
       Repository
```

Important concepts:

```text
Stanza
Repository
Backup
Archive
Retention
Restore
Check
Info
```

pgBackRest documentation defines:

### Full

Entire cluster backup.

### Differential

Changes since the latest full backup.

### Incremental

Changes since the previous backup.

The incremental chain depends on its preceding backups; the required backup chain must remain valid for restore. ([pgBackRest][7])

---

# pgBackRest commands

Check:

```bash
pgbackrest --stanza=main check
```

Info:

```bash
pgbackrest --stanza=main info
```

Full:

```bash
pgbackrest --stanza=main --type=full backup
```

Differential:

```bash
pgbackrest --stanza=main --type=diff backup
```

Incremental:

```bash
pgbackrest --stanza=main --type=incr backup
```

---

# DAY 14 — DR LAB

Scenario:

```text
02:00 Full backup
10:00 Normal operation
10:20 Important data
10:30 DELETE
10:35 Incident discovered
```

Process:

```text
Preserve source
      ↓
Identify target
      ↓
Verify backup
      ↓
Verify WAL
      ↓
Prepare recovery environment
      ↓
Restore base backup
      ↓
Replay WAL
      ↓
Stop at recovery target
      ↓
Validate
      ↓
Application validation
```

---

# Recovery validation

Database:

```sql
SELECT datname
FROM pg_database;
```

Recovery status:

```sql
SELECT pg_is_in_recovery();
```

Data:

```sql
SELECT COUNT(*)
FROM orders;
```

A successful PostgreSQL startup does **not** automatically prove a successful recovery.

Validate:

```text
Database
Schema
Tables
Rows
Constraints
Sequences
Application consistency
WAL/recovery state
```

---

# WEEK 2 COMPLETION GATE

You should independently demonstrate:

```text
pg_dump
   ↓
pg_restore
   ↓
pg_basebackup
   ↓
WAL archive
   ↓
pgBackRest
   ↓
PITR
   ↓
Recovery validation
```

---

# WEEK 3

# STREAMING REPLICATION + HA + PATRONI

PostgreSQL's native architecture supports:

```text
Primary
   |
   | WAL
   v
Standby
```

Streaming replication can be asynchronous or synchronous. PostgreSQL itself does not provide the complete failure-detection/failover orchestration layer; HA tooling can provide that. ([PostgreSQL][8])

---

# DAY 15 — STREAMING REPLICATION

Architecture:

```text
PRIMARY
   |
WAL Sender
   |
Streaming
   |
WAL Receiver
   |
STANDBY
   |
WAL Replay
```

Primary generates WAL.

Standby:

```text
receives WAL
     ↓
flushes WAL
     ↓
replays WAL
```

---

# Primary settings

```sql
SHOW wal_level;
SHOW max_wal_senders;
SHOW max_replication_slots;
SHOW wal_keep_size;
```

PostgreSQL documents `max_wal_senders`, `wal_keep_size`, replication slots and related settings as part of streaming replication configuration. ([PostgreSQL][2])

---

# Replication user

```sql
CREATE ROLE replicator
WITH REPLICATION
LOGIN
PASSWORD 'StrongPassword';
```

`pg_hba.conf`:

```text
host replication replicator <standby-ip>/32 scram-sha-256
```

Reload:

```bash
systemctl reload postgresql
```

---

# DAY 16 — CREATE PHYSICAL STANDBY

Use:

```bash
pg_basebackup \
-h <PRIMARY_IP> \
-U replicator \
-D <STANDBY_DATA_DIRECTORY> \
-Fp \
-X stream \
-P
```

PostgreSQL's replication protocol explicitly supports streaming base backups. ([PostgreSQL][9])

Create:

```text
standby.signal
```

Configure:

```text
primary_conninfo
```

PostgreSQL 18 documents `primary_conninfo` as the standby connection string to the sending server. ([PostgreSQL][2])

---

# Verify standby

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
true
```

Primary:

```sql
SELECT
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

Expected:

```text
state = streaming
```

---

# Replication test

Primary:

```sql
CREATE TABLE test_data
(
    id integer PRIMARY KEY,
    description text
);

INSERT INTO test_data
VALUES
(1,'Primary'),
(2,'Replication'),
(3,'Test');
```

Standby:

```sql
SELECT * FROM test_data;
```

Flow:

```text
INSERT
 ↓
WAL
 ↓
WAL Sender
 ↓
Network
 ↓
WAL Receiver
 ↓
WAL
 ↓
Replay
 ↓
Visible data
```

---

# DAY 17 — REPLICATION INTERNALS

Primary:

```text
Backend
WAL Writer
Checkpointer
WAL Sender
```

Standby:

```text
WAL Receiver
Startup/recovery process
Replay
```

Important terms:

| Term             | Meaning                       |
| ---------------- | ----------------------------- |
| WAL Sender       | Sends WAL                     |
| WAL Receiver     | Receives WAL                  |
| Replay           | Applies WAL                   |
| LSN              | WAL position                  |
| Replication slot | Retains WAL for consumer      |
| Replay lag       | Standby replay behind primary |

---

# LSN PIPELINE

Primary:

```sql
SELECT pg_current_wal_lsn();
```

Standby:

```sql
SELECT pg_last_wal_receive_lsn();
```

```sql
SELECT pg_last_wal_replay_lsn();
```

Mental model:

```text
Primary WAL
     ↓
Sent
     ↓
Received
     ↓
Flushed
     ↓
Replayed
```

---

# DAY 18 — REPLICATION LAG

Primary:

```sql
SELECT
    client_addr,
    state,
    sync_state,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

Investigate:

```text
Network
   ↓
WAL generation
   ↓
Transmission
   ↓
Receive
   ↓
Flush
   ↓
Replay
```

Do not automatically label every lag problem as a network issue.

---

# REPLICATION SLOTS

Check:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn
FROM pg_replication_slots;
```

Core concept:

```text
Consumer stops
     ↓
WAL retained
     ↓
pg_wal grows
     ↓
Filesystem pressure
```

PostgreSQL documentation explicitly warns that replication slots can cause WAL accumulation and eventually consume disk space if the consumer does not keep up. ([PostgreSQL][10])

---

# DAY 19 — ASYNC VS SYNC

## Asynchronous

```text
Primary
   |
   +------> Standby
```

Primary does not normally wait for standby confirmation for every commit.

Potential:

```text
Lower latency
+
Potential data-loss window
```

---

## Synchronous

```text
Application
     ↓
Primary
     ↓
Standby confirmation
     ↓
Commit proceeds
```

Important settings:

```sql
SHOW synchronous_standby_names;
SHOW synchronous_commit;
```

`sync_state` can be inspected through:

```sql
pg_stat_replication
```

PostgreSQL's synchronous replication configuration controls which standby confirmations transactions wait for. ([PostgreSQL][2])

---

# SQL SERVER AG COMPARISON

| SQL Server AG         | PostgreSQL                           |
| --------------------- | ------------------------------------ |
| Primary replica       | Primary                              |
| Secondary replica     | Standby                              |
| Transaction log       | WAL                                  |
| Log transport         | WAL streaming                        |
| Synchronous commit    | Synchronous replication              |
| Asynchronous commit   | Asynchronous replication             |
| AG synchronization    | Replication state                    |
| AG automatic failover | HA framework such as Patroni         |
| Listener              | Routing layer such as HAProxy        |
| Seeding               | `pg_basebackup`/bootstrap mechanisms |

### Critical interview statement

> PostgreSQL streaming replication provides data replication, but automatic HA requires additional failure-detection, leader-election and routing mechanisms.

PostgreSQL itself documents that it does not provide the system software needed to automatically detect primary failure and notify/promote a standby. ([PostgreSQL][11])

---

# DAY 20 — PATRONI

Patroni is an HA orchestration framework.

Architecture:

```text
                 Application
                      |
                    HAProxy
                  /       \
                 /         \
              PG-01       PG-02
             PRIMARY      STANDBY
                 \          /
                  \        /
                   Patroni
                      |
                      v
                     DCS
                      |
               etcd / Consul
```

Think:

```text
PostgreSQL
    =
Database engine

Streaming replication
    =
Data replication

Patroni
    =
HA orchestration

etcd
    =
Distributed cluster state
```

Patroni documentation describes the DCS as the location where cluster state and dynamic configuration are stored. ([Patroni][12])

---

# Patroni responsibilities

Understand:

```text
Leader election
Health checking
PostgreSQL lifecycle
Failover
Switchover
Replication management
Cluster state
```

Important parameters:

```text
ttl
loop_wait
retry_timeout
maximum_lag_on_failover
```

Patroni's documented relationship is:

```text
loop_wait + 2 * retry_timeout <= ttl
```

and `maximum_lag_on_failover` controls how far a candidate may lag and still participate in leader election. ([Patroni][12])

---

# DAY 21 — SWITCHOVER

Planned operation:

```text
PG-01 PRIMARY
PG-02 STANDBY

       ↓

Switchover

       ↓

PG-01 STANDBY
PG-02 PRIMARY
```

Use cases:

```text
OS patching
PostgreSQL maintenance
Infrastructure maintenance
Planned testing
```

Patroni supports `patronictl switchover`. ([Patroni][13])

---

# FAILOVER

Unplanned:

```text
PRIMARY
   X
   |
Failure detected
   |
Standby candidate
   |
Promotion
   |
NEW PRIMARY
```

Patroni supports failover operations, but failover can result in data loss depending on replication state. Patroni's own documentation warns about this. ([Patroni][14])

---

# Critical split-brain protection

Never simply bring the old primary back online after another node has been promoted.

Potential:

```text
OLD PRIMARY
      |
      +---- thinks PRIMARY

NEW PRIMARY
      |
      +---- thinks PRIMARY
```

That is a dangerous dual-primary condition.

PostgreSQL explicitly discusses the need to prevent the old primary from operating as primary after failover; mechanisms such as fencing/STONITH are important in HA designs. ([PostgreSQL][11])

---

# Rebuilding old primary

After failover:

```text
NEW PRIMARY
     |
     |
OLD PRIMARY
```

The old primary generally needs to be:

```text
fenced
 ↓
reinitialized/rejoined
 ↓
new standby
```

For suitable situations, `pg_rewind` can accelerate rejoining a diverged former primary. PostgreSQL documents this as a way to reduce the work required to recreate a standby after failover. ([PostgreSQL][11])

---

# WEEK 3 COMPLETION GATE

You should be able to demonstrate:

```text
Primary
 ↓
WAL
 ↓
Streaming
 ↓
Standby
 ↓
Replication monitoring
 ↓
Lag detection
 ↓
Slot monitoring
 ↓
Switchover
 ↓
Failover
 ↓
Rebuild standby
```

---

# WEEK 4

# PERFORMANCE + SECURITY + PGBOUNCER + MONITORING

---

# DAY 22 — EXPLAIN

Basic:

```sql
EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 100;
```

Actual:

```sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE customer_id = 100;
```

Buffers:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 100;
```

---

# Execution operators

| Operator          | Meaning                     |
| ----------------- | --------------------------- |
| Seq Scan          | Sequential table scan       |
| Index Scan        | Index lookup                |
| Index Only Scan   | Index can satisfy query     |
| Bitmap Index Scan | Bitmap generated from index |
| Bitmap Heap Scan  | Heap access through bitmap  |
| Nested Loop       | Repeated inner lookup       |
| Hash Join         | Hash-based join             |
| Merge Join        | Merge-based join            |
| Sort              | Sort operation              |
| Aggregate         | Aggregation                 |
| HashAggregate     | Hash-based aggregation      |

---

# Estimated vs Actual

Example:

```text
Estimated rows = 10
Actual rows    = 500,000
```

Investigate:

```text
Statistics
Selectivity
Data distribution
Predicate
Indexes
Join conditions
Parameter values
```

Important:

```text
cost
rows
actual rows
loops
buffers
```

---

# DAY 23 — INDEXING

Know:

```text
B-tree
Hash
GIN
GiST
BRIN
```

General:

```sql
CREATE INDEX idx_orders_customer
ON orders(customer_id);
```

Composite:

```sql
CREATE INDEX idx_orders_customer_date
ON orders(customer_id, created_at);
```

Partial:

```sql
CREATE INDEX idx_active_orders
ON orders(customer_id)
WHERE status = 'ACTIVE';
```

Expression:

```sql
CREATE INDEX idx_customer_lower_name
ON customers(lower(customer_name));
```

BRIN:

```sql
CREATE INDEX idx_orders_created_brin
ON orders USING BRIN(created_at);
```

---

# Index Lab

```sql
CREATE TABLE performance_lab
(
    id bigint,
    customer_id integer,
    order_date timestamp,
    amount numeric,
    status text
);
```

Generate data:

```sql
INSERT INTO performance_lab
SELECT
    g,
    (random() * 10000)::integer,
    timestamp '2025-01-01'
       + random() * interval '365 days',
    random() * 1000,
    CASE
       WHEN random() < 0.8 THEN 'ACTIVE'
       ELSE 'CLOSED'
    END
FROM generate_series(1,1000000) g;
```

Analyze:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM performance_lab
WHERE customer_id = 5000;
```

Create:

```sql
CREATE INDEX idx_perf_customer
ON performance_lab(customer_id);
```

Analyze again.

---

# DAY 24 — pg_stat_statements

Check:

```sql
SELECT *
FROM pg_extension
WHERE extname = 'pg_stat_statements';
```

Example:

```sql
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Analyze:

```text
Total execution time
Average execution time
Calls
Rows
```

Important DBA principle:

```text
1,000,000 executions × 10 ms
```

may create more cumulative workload than:

```text
1 execution × 5 seconds
```

---

# STATISTICS

```sql
ANALYZE performance_lab;
```

Inspect:

```sql
SELECT
    schemaname,
    tablename,
    attname,
    n_distinct
FROM pg_stats
WHERE tablename = 'performance_lab';
```

Understand:

```text
Statistics
Selectivity
Cardinality
Histograms
Most common values
Planner estimates
```

---

# DAY 25 — VACUUM

PostgreSQL MVCC creates obsolete row versions.

```text
UPDATE/DELETE
      ↓
Dead tuple
      ↓
VACUUM
      ↓
Reusable space
```

Run:

```sql
VACUUM performance_lab;
```

Verbose:

```sql
VACUUM (VERBOSE)
performance_lab;
```

Both:

```sql
VACUUM (ANALYZE)
performance_lab;
```

---

# AUTOVACUUM

Check:

```sql
SHOW autovacuum;
SHOW autovacuum_max_workers;
SHOW autovacuum_naptime;
SHOW autovacuum_vacuum_threshold;
SHOW autovacuum_analyze_threshold;
```

Monitor:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Investigate:

```text
Dead tuples
Autovacuum workers
Autovacuum thresholds
Long transactions
I/O
Table-specific settings
Transaction age
```

---

# TRANSACTION ID AGE

```sql
SELECT
    datname,
    age(datfrozenxid)
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

Understand:

```text
MVCC
XID
freezing
VACUUM
wraparound protection
```

---

# DAY 26 — BLOCKING

Active sessions:

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity;
```

Waiting locks:

```sql
SELECT
    pid,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

Target:

```text
Blocked PID
     ↓
Lock
     ↓
Blocking PID
     ↓
Blocking transaction
     ↓
Root cause
```

---

# DEADLOCK

Concept:

```text
Session A
  |
Lock 1
  |
wait Lock 2


Session B
  |
Lock 2
  |
wait Lock 1
```

PostgreSQL detects deadlocks and aborts one transaction.

Investigate:

```text
PostgreSQL log
SQL statements
Transaction order
Application behavior
Lock ordering
```

---

# DAY 27 — SECURITY

Create role:

```sql
CREATE ROLE app_user
LOGIN
PASSWORD 'StrongPassword';
```

Create database:

```sql
CREATE DATABASE production_lab;
```

Grant:

```sql
GRANT CONNECT
ON DATABASE production_lab
TO app_user;
```

Schema:

```sql
GRANT USAGE
ON SCHEMA public
TO app_user;
```

Table:

```sql
GRANT SELECT, INSERT, UPDATE
ON orders
TO app_user;
```

---

# Role attributes

Understand:

```text
LOGIN
NOLOGIN
SUPERUSER
CREATEDB
CREATEROLE
REPLICATION
```

Use least privilege.

---

# pg_hba.conf

Example:

```text
host all app_user 192.168.1.0/24 scram-sha-256
```

Flow:

```text
Client
 ↓
PostgreSQL
 ↓
pg_hba.conf
 ↓
First matching rule
 ↓
Authentication
```

---

# Security checklist

```text
SCRAM
SSL/TLS
Least privilege
Role hierarchy
pg_hba.conf
Superuser control
Audit logging
pgaudit
Row-Level Security
Password management
```

---

# DAY 28 — PGBOUNCER

Architecture:

```text
Applications
     |
     v
 PgBouncer
     |
     +---- PostgreSQL
     +---- PostgreSQL
     +---- PostgreSQL
```

Purpose:

> Connection pooling.

Pooling modes:

```text
session
transaction
statement
```

### Session

Connection remains associated with client session.

### Transaction

Connection can be reused after transaction completion.

### Statement

Connection can be reused after statement completion.

Important:

Transaction/statement pooling can affect session-dependent PostgreSQL features.

---

# CONNECTION EXHAUSTION

Check:

```sql
SHOW max_connections;
```

Current:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

By database:

```sql
SELECT
    datname,
    count(*)
FROM pg_stat_activity
GROUP BY datname
ORDER BY count(*) DESC;
```

By state:

```sql
SELECT
    state,
    count(*)
FROM pg_stat_activity
GROUP BY state;
```

Look for:

```text
active
idle
idle in transaction
```

Don't automatically fix:

```text
Too many connections
```

by increasing `max_connections`.

Investigate the application connection pool and consider PgBouncer.

---

# DAY 29 — MONITORING

Important PostgreSQL views:

| View                      | Purpose                          |
| ------------------------- | -------------------------------- |
| `pg_stat_activity`        | Sessions/queries                 |
| `pg_stat_replication`     | Replication                      |
| `pg_stat_wal`             | WAL                              |
| `pg_stat_database`        | Database statistics              |
| `pg_stat_user_tables`     | Table activity                   |
| `pg_stat_user_indexes`    | Index usage                      |
| `pg_stat_bgwriter`        | Background/checkpoint statistics |
| `pg_locks`                | Locks                            |
| `pg_replication_slots`    | Slots                            |
| `pg_stat_progress_vacuum` | VACUUM progress                  |
| `pg_stat_archiver`        | WAL archiving                    |

---

# Production monitoring model

```text
                     PostgreSQL
                          |
          +---------------+---------------+
          |               |               |
        System          Database           HA
          |               |               |
         CPU             Queries           WAL
         RAM             Locks             Lag
         Disk            Sessions          Slots
         I/O             Bloat             Standby
```

OS:

```bash
top
free -h
vmstat 1
iostat -xz 1
df -h
df -i
sar
ss
```

PostgreSQL:

```sql
pg_stat_activity
pg_stat_database
pg_stat_replication
pg_stat_wal
pg_locks
pg_stat_user_tables
pg_replication_slots
pg_stat_archiver
```

---

# DAY 30 — PRODUCTION CAPSTONE

Build:

```text
                  APPLICATION
                       |
                   PgBouncer
                       |
                    HAProxy
                       |
               +-------+-------+
               |               |
            PG-01           PG-02
           PRIMARY          STANDBY
               |               |
               +----- WAL -----+
                       |
                    Patroni
                       |
                      etcd
                       |
                  pgBackRest
                       |
                   Repository
```

---

# CAPSTONE SCENARIO

```text
10:00 PostgreSQL healthy
10:15 Backup successful
10:20 Application workload increases
10:25 CPU increases
10:30 Query latency increases
10:35 Replication lag increases
10:40 Disk = 90%
10:45 Connection failures
10:50 Primary failure
```

Your investigation:

---

## STEP 1 — OS

```bash
df -h
free -h
top
iostat -xz 1
vmstat 1
```

---

## STEP 2 — SERVICE

```bash
systemctl status postgresql
```

If Patroni controls PostgreSQL, investigate Patroni status rather than independently manipulating the PostgreSQL service.

---

## STEP 3 — LOGS

```bash
journalctl -u postgresql
```

and PostgreSQL/Patroni logs as configured.

---

## STEP 4 — CONNECTIONS

```sql
SELECT *
FROM pg_stat_activity;
```

---

## STEP 5 — BLOCKING

```sql
SELECT *
FROM pg_locks;
```

---

## STEP 6 — EXPENSIVE SQL

```sql
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

---

## STEP 7 — REPLICATION

```sql
SELECT *
FROM pg_stat_replication;
```

---

## STEP 8 — SLOTS

```sql
SELECT *
FROM pg_replication_slots;
```

---

## STEP 9 — BLOAT

```sql
SELECT
    relname,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

---

## STEP 10 — HA

If primary genuinely failed:

```text
Confirm failure
      ↓
Ensure old primary is fenced
      ↓
Evaluate standby health
      ↓
Evaluate replication lag
      ↓
Patroni failover
      ↓
Validate new primary
      ↓
Route application
      ↓
Rebuild old primary
```

---

# ADVANCED PRODUCTION DBA SCENARIOS

## Scenario 1 — CPU 100%

```text
CPU
 ↓
Top PostgreSQL process
 ↓
PID
 ↓
Session
 ↓
Query
 ↓
EXPLAIN ANALYZE
 ↓
Root cause
```

Investigate:

```text
Bad plan
Missing index
Cardinality estimation
Large sort
Hash operation
Excessive concurrency
High execution frequency
```

---

# Scenario 2 — Disk 95%

Start:

```bash
df -h
```

Then:

```bash
du -sh /var/lib/postgresql/*
```

Check:

```bash
du -sh $PGDATA/pg_wal
```

Slots:

```sql
SELECT
    slot_name,
    active,
    restart_lsn
FROM pg_replication_slots;
```

Possible causes:

```text
Replication slot
Failed archive
Disconnected standby
High WAL generation
Backup/archive destination
```

Never manually delete active PostgreSQL WAL files.

---

# Scenario 3 — Replication lag

Check primary:

```sql
SELECT
    client_addr,
    state,
    sync_state,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

Standby:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

Investigate:

```text
Network
WAL generation
Receive
Flush
Replay
I/O
Standby workload
```

---

# Scenario 4 — Standby disconnected

Primary:

```sql
SELECT *
FROM pg_stat_replication;
```

Standby:

```bash
systemctl status postgresql
journalctl -u postgresql -n 100
```

Network:

```bash
ss -ntp
nc -zv <PRIMARY_IP> 5432
```

Investigate:

```text
PostgreSQL down
Network
pg_hba.conf
Authentication
Replication user
WAL availability
Replication slot
Configuration
```

---

# Scenario 5 — Long transaction

```sql
SELECT
    pid,
    usename,
    datname,
    xact_start,
    now() - xact_start AS transaction_age,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Problem:

```text
Long transaction
      ↓
Old snapshot
      ↓
Cleanup delayed
      ↓
Dead tuples
      ↓
Bloat
```

---

# Scenario 6 — Autovacuum falling behind

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Investigate:

```text
Dead tuples
Autovacuum settings
Long transactions
I/O
Worker availability
Table-specific configuration
XID age
```

---

# Scenario 7 — Connection exhaustion

Error:

```text
remaining connection slots are reserved
```

Investigate:

```sql
SHOW max_connections;
```

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

Then:

```text
Application pool
 ↓
PgBouncer
 ↓
PostgreSQL
```

---

# Scenario 8 — Blocking

Identify:

```text
Blocked PID
 ↓
Lock
 ↓
Blocking PID
 ↓
Transaction
 ↓
Root cause
```

Don't terminate sessions before understanding the transaction and business impact.

---

# Scenario 9 — Deadlock

```text
A locks X
A waits Y

B locks Y
B waits X
```

Investigate PostgreSQL logs and application transaction ordering.

---

# Scenario 10 — WAL archiving failure

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

Investigate:

```text
archive_command
Permissions
Destination
Disk
Network
Storage
```

---

# Scenario 11 — Backup failure

pgBackRest:

```bash
pgbackrest --stanza=main check
```

```bash
pgbackrest --stanza=main info
```

Investigate:

```text
Repository
Permissions
Disk
WAL archive
Stanza
Configuration
Retention
Connectivity
```

---

# Scenario 12 — PITR incident

```text
02:00 Full backup
10:20 Corruption
10:30 Discovery
```

Requirement:

```text
Recover to 10:19:59
```

Process:

```text
Identify backup
 ↓
Verify WAL
 ↓
Restore
 ↓
Replay WAL
 ↓
Recovery target
 ↓
Validate
```

---

# Scenario 13 — Patroni primary failure

```text
PG-01
PRIMARY
   X

PG-02
STANDBY
   |
   v
Promotion
   |
   v
PRIMARY
```

Investigate:

```text
Patroni
DCS
Leader
Replication
Lag
Application routing
```

Then:

```text
Failover
 ↓
Validate
 ↓
Route application
 ↓
Rebuild old primary
```

---

# Scenario 14 — PostgreSQL 15 → 18 upgrade

Understand:

```text
Major upgrade
    ≠
Minor update
```

Major upgrade methods include:

```text
pg_upgrade
Logical replication
Dump/restore
Migration tooling
```

Do not treat a major-version upgrade as merely replacing PostgreSQL binaries.

---

# Scenario 15 — SQL Server → PostgreSQL migration

Architecture:

```text
SQL Server
    |
Schema assessment
    |
Schema conversion
    |
Data migration
    |
Application conversion
    |
Validation
    |
Performance testing
    |
Cutover
```

Study:

```text
AWS DMS
pgloader
Logical replication
COPY
Custom ETL
```

---

# SQL SERVER DATA-TYPE MAPPING

| SQL Server         | PostgreSQL                                               |
| ------------------ | -------------------------------------------------------- |
| `int`              | `integer`                                                |
| `bigint`           | `bigint`                                                 |
| `decimal`          | `numeric`                                                |
| `datetime`         | `timestamp`                                              |
| `datetime2`        | `timestamp`                                              |
| `bit`              | `boolean`                                                |
| `uniqueidentifier` | `uuid`                                                   |
| `nvarchar`         | `varchar`/`text`                                         |
| `varbinary`        | `bytea`                                                  |
| Identity           | Identity/generated column                                |
| `GETDATE()`        | `CURRENT_TIMESTAMP` / related functions                  |
| `NEWID()`          | `gen_random_uuid()` depending on extension/version setup |

Migration areas:

```text
Data types
Identity
Sequences
T-SQL
Stored procedures
Functions
Triggers
Security
SQL Agent
Linked Servers
SSIS
Application drivers
Collation
Time zones
```

---

# Scenario 16 — AWS RDS PostgreSQL

Learn:

```text
RDS PostgreSQL
Aurora PostgreSQL
```

Topics:

```text
Parameter groups
Automated backups
Snapshots
PITR
Multi-AZ
Read replicas
CloudWatch
Performance monitoring
Extensions
Storage scaling
Major upgrades
Security groups
IAM integration where applicable
```

Senior interview question:

> What changes when PostgreSQL moves from self-managed infrastructure to RDS?

Think:

```text
OS access
Storage management
Backups
Patching
HA implementation
Monitoring
Extensions
Configuration privileges
```

---

# Scenario 17 — Monitoring Architecture

```text
PostgreSQL
     |
Exporter/Agent
     |
Prometheus
     |
Grafana
```

Monitor:

```text
CPU
Memory
Disk
IOPS
Connections
Queries
Locks
WAL
Replication
Autovacuum
Bloat
Backups
```

---

# Scenario 18 — Automation

Start:

```text
Bash
 ↓
SQL
 ↓
Python
 ↓
Ansible
 ↓
Terraform
```

Automate:

```text
Health checks
Backup checks
Replication checks
Disk checks
Long-running queries
Blocking
Replication lag
Backup status
Configuration validation
```

Example:

```bash
#!/bin/bash

df -h
free -h
uptime

psql -d postgres -c "SELECT now();"
```

---

# POSTGRESQL HA LAB

# FINAL VERIFIED ARCHITECTURE

For the lab:

```text
                         Application
                              |
                              v
                          HAProxy
                              |
                     +--------+--------+
                     |                 |
                     v                 v
                   PG-01             PG-02
                  PRIMARY           STANDBY
                     |                 |
                     +------ WAL ------+
                              |
                           Patroni
                              |
                              v
                         DCS / etcd
                              |
                     +----------------+
                     |                |
                 pgBackRest       Monitoring
                     |
                  Repository
```

### Lab IP plan

| VM      | IP               | Role                    |
| ------- | ---------------- | ----------------------- |
| PG-01   | `192.168.0.151`  | PostgreSQL 18 + Patroni |
| PG-02   | `192.168.0.152`  | PostgreSQL 18 + Patroni |
| ETCD-01 | `192.168.0.153`  | etcd                    |
| HAProxy | `192.168.0.154`  | Routing                 |
| Client  | Existing Windows | Testing                 |

---

# IMPORTANT DCS CORRECTION

A single etcd VM is acceptable for a **learning lab**, but not a fault-tolerant production DCS.

Patroni's current documentation recommends **3 or 5 DCS nodes** for consensus/fault tolerance. ([Patroni][15])

Production:

```text
ETCD-01
ETCD-02
ETCD-03
```

Lab:

```text
ETCD-01
```

is acceptable for learning the mechanics, but it is a single point of failure.

---

# PG-01 / PG-02

Install PostgreSQL 18 using the appropriate repository/package method for your RHEL release.

Verify:

```bash
psql --version
```

Initialize the cluster according to the distribution's PostgreSQL packaging.

---

# PRIMARY REPLICATION CONFIGURATION

Conceptually:

```text
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
```

Then restart if required by the changed parameter.

Create:

```sql
CREATE ROLE replicator
WITH REPLICATION
LOGIN
PASSWORD 'StrongPassword';
```

`pg_hba.conf`:

```text
host replication replicator 192.168.0.152/32 scram-sha-256
```

Reload.

---

# STANDBY CREATION

Stop the disposable lab standby.

Initialize it from primary:

```bash
pg_basebackup \
    -h 192.168.0.151 \
    -U replicator \
    -D <PGDATA> \
    -Fp \
    -Xs \
    -P
```

Create:

```text
standby.signal
```

Configure:

```text
primary_conninfo
```

PostgreSQL 18 uses the `standby.signal` mechanism and normal configuration parameters rather than the obsolete `recovery.conf`/`standby_mode` approach. ([PostgreSQL][6])

---

# VERIFY

PG-02:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
true
```

PG-01:

```sql
SELECT
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

---

# PATRONI LAYER

Patroni:

```text
PG-01
 |
Patroni
 |
etcd

PG-02
 |
Patroni
 |
etcd
```

Use a Patroni configuration appropriate to your actual installed Patroni version and PostgreSQL packaging.

Do not mix:

```text
systemd-managed PostgreSQL
```

with:

```text
Patroni-managed PostgreSQL
```

without understanding ownership of the PostgreSQL lifecycle.

Once Patroni is managing the cluster, Patroni should normally control PostgreSQL startup/stop/restart operations.

---

# PATRONI STATUS

Typical:

```bash
patronictl -c /etc/patroni.yml list
```

Conceptually:

```text
+--------+---------+---------+
| Member | Role    | State   |
+--------+---------+---------+
| PG-01  | Leader  | running |
| PG-02  | Replica | running |
+--------+---------+---------+
```

---

# PLANNED SWITCHOVER

Use:

```bash
patronictl -c /etc/patroni.yml switchover
```

Expected:

```text
PG-01
Leader
  ↓
Standby

PG-02
Replica
  ↓
Leader
```

Patroni's current documentation distinguishes switchover from failover and exposes both through `patronictl`/REST API. ([Patroni][13])

---

# FAILOVER

Use only after confirming the primary is actually unavailable/fenced.

Conceptually:

```text
PRIMARY
   X
   |
Patroni
   |
DCS
   |
Eligible standby
   |
Promotion
```

After promotion:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
false
```

---

# HAProxy

Architecture:

```text
Application
     |
  HAProxy
   /    \
 PG-01  PG-02
```

Do not blindly round-robin writes between primary and standby.

Use Patroni health/API information to determine which node is the current primary.

---

# PGBOUNCER

Final application path:

```text
Application
     |
 PgBouncer
     |
 HAProxy
     |
PostgreSQL HA
```

Depending on your architecture, PgBouncer and HAProxy can be arranged differently; the important requirement is that connection routing follows the current writable primary.

---

# PGBACKREST

Final DR path:

```text
Primary
   |
pgBackRest
   |
Repository
   |
WAL
   |
PITR
```

pgBackRest's current documentation describes full, differential and incremental backups and restore dependencies accordingly. ([pgBackRest][7])

---

# PRODUCTION DBA TROUBLESHOOTING MODEL

For **every incident**, use:

```text
DETECT
   ↓
IDENTIFY
   ↓
MEASURE
   ↓
ISOLATE
   ↓
MITIGATE
   ↓
RECOVER
   ↓
VALIDATE
   ↓
RCA
   ↓
PREVENT
```

Example:

## Replication lag

```text
Detect
 ↓
pg_stat_replication
 ↓
Identify send/receive/replay point
 ↓
Measure WAL + I/O + network
 ↓
Mitigate
 ↓
Restore synchronization
 ↓
Validate
 ↓
RCA
 ↓
Improve monitoring
```

---

# SENIOR POSTGRESQL DBA INTERVIEW AREAS

## Architecture

1. PostgreSQL architecture?
2. Cluster vs database?
3. Backend process?
4. Shared buffers?
5. WAL writer?
6. Checkpointer?
7. Background writer?
8. Autovacuum launcher?
9. Catalogs?
10. Page/tuple architecture?

## WAL

11. What is WAL?
12. Why WAL?
13. What is LSN?
14. What is checkpoint?
15. What causes WAL growth?
16. WAL archiving?
17. WAL retention?
18. Replication slot?
19. Slot-induced disk growth?
20. How do you monitor WAL?

## MVCC

21. What is MVCC?
22. Dead tuple?
23. VACUUM?
24. Autovacuum?
25. Long-running transaction impact?
26. XID wraparound?
27. Freeze?
28. Snapshot?

## Backup

29. `pg_dump` vs `pg_basebackup`?
30. `pg_dumpall`?
31. Physical vs logical?
32. pgBackRest?
33. Full/diff/incr?
34. Backup verification?
35. Restore testing?

## PITR

36. PITR?
37. Required components?
38. Recovery target?
39. WAL continuity?
40. Recovery validation?
41. Timeline?
42. What happens after target reached?

## Replication

43. Streaming replication?
44. WAL sender?
45. WAL receiver?
46. Replay?
47. Sync vs async?
48. Replication lag?
49. Replication slot?
50. Cascading replication?

## Patroni

51. What does Patroni solve?
52. What is DCS?
53. Why etcd?
54. Leader election?
55. TTL?
56. Failover?
57. Switchover?
58. Split brain?
59. Fencing?
60. Rebuild old primary?

## Performance

61. `EXPLAIN`?
62. `EXPLAIN ANALYZE`?
63. BUFFERS?
64. Seq Scan?
65. Nested Loop?
66. Hash Join?
67. Bitmap scan?
68. Statistics?
69. Index selectivity?
70. `pg_stat_statements`?

## Vacuum

71. Dead tuples?
72. Autovacuum?
73. Bloat?
74. Long transaction?
75. XID age?
76. Wraparound?
77. VACUUM vs VACUUM FULL?

## Security

78. Role?
79. Login role?
80. `pg_hba.conf`?
81. SCRAM?
82. SSL?
83. Least privilege?
84. Superuser?
85. RLS?
86. pgaudit?

## Connections

87. `max_connections`?
88. Connection exhaustion?
89. PgBouncer?
90. Session vs transaction pooling?

## Cloud

91. RDS PostgreSQL?
92. Aurora PostgreSQL?
93. Multi-AZ?
94. Read replica?
95. PITR?
96. Parameter groups?
97. Cloud monitoring?
98. RDS vs self-managed PostgreSQL?

## Migration

99. SQL Server → PostgreSQL?
100. Major-version migration?
101. `pg_upgrade`?
102. Logical replication migration?
103. AWS DMS?
104. Data validation?
105. Cutover?
106. Rollback?

---

# FINAL 30-DAY CHECKLIST

## Linux

```text
[ ] Filesystem
[ ] chmod
[ ] chown
[ ] Users/groups
[ ] sudo
[ ] Processes
[ ] CPU
[ ] Memory
[ ] I/O
[ ] Filesystem
[ ] Networking
[ ] systemd
[ ] journalctl
[ ] grep
[ ] tail
```

## PostgreSQL Core

```text
[ ] Cluster
[ ] Database
[ ] Schema
[ ] Configuration
[ ] pg_hba.conf
[ ] Processes
[ ] Memory
[ ] MVCC
[ ] WAL
[ ] LSN
[ ] Checkpoints
[ ] Locks
[ ] Blocking
```

## Backup/DR

```text
[ ] pg_dump
[ ] pg_restore
[ ] pg_dumpall
[ ] pg_basebackup
[ ] WAL archiving
[ ] pg_verifybackup
[ ] pgBackRest
[ ] Full
[ ] Differential
[ ] Incremental
[ ] PITR
[ ] Recovery validation
```

## HA

```text
[ ] Streaming replication
[ ] Physical standby
[ ] Replication slots
[ ] Replication lag
[ ] Sync replication
[ ] Async replication
[ ] Patroni
[ ] DCS
[ ] etcd
[ ] Switchover
[ ] Failover
[ ] Fencing
[ ] pg_rewind
```

## Performance

```text
[ ] EXPLAIN
[ ] EXPLAIN ANALYZE
[ ] BUFFERS
[ ] Indexes
[ ] Statistics
[ ] pg_stat_statements
[ ] VACUUM
[ ] Autovacuum
[ ] Bloat
[ ] Blocking
[ ] Deadlocks
```

## Security

```text
[ ] Roles
[ ] Privileges
[ ] pg_hba.conf
[ ] SCRAM
[ ] SSL
[ ] Least privilege
[ ] Auditing
[ ] RLS
```

## Connectivity

```text
[ ] max_connections
[ ] pg_stat_activity
[ ] PgBouncer
[ ] Session pooling
[ ] Transaction pooling
[ ] HAProxy
```

## Cloud

```text
[ ] AWS RDS PostgreSQL
[ ] Aurora PostgreSQL
[ ] Azure Database for PostgreSQL
[ ] Cloud monitoring
[ ] Cloud backup/PITR
[ ] Cloud HA
```

## Automation

```text
[ ] Bash
[ ] SQL
[ ] Python
[ ] Ansible
[ ] Terraform
[ ] CI/CD
```

---

# FINAL SENIOR DBA CAPSTONE

Your final lab should look like:

```text
                         USERS
                           |
                           v
                      APPLICATION
                           |
                           v
                       PgBouncer
                           |
                           v
                        HAProxy
                           |
                +----------+----------+
                |                     |
                v                     v
             PG-01                  PG-02
            PRIMARY                STANDBY
                |                     |
                |------ WAL -------->|
                |                     |
             Patroni              Patroni
                |                     |
                +----------+----------+
                           |
                           v
                          etcd
                     DCS / cluster state

                PG-01
                  |
                  v
              pgBackRest
                  |
                  v
          Backup Repository
                  |
                  v
              WAL Archive
                  |
                  v
                 PITR

PostgreSQL metrics
        |
        v
Prometheus
        |
        v
Grafana
```

Then deliberately break it:

```text
1. Stop standby
2. Restart standby
3. Introduce replication lag
4. Stop WAL consumer
5. Create slot retention
6. Fill filesystem in controlled lab
7. Create blocking
8. Create deadlock
9. Create long transaction
10. Stress CPU
11. Stress connections
12. Break WAL archiving
13. Test backup failure
14. Perform PITR
15. Perform planned switchover
16. Perform primary failure
17. Promote standby
18. Fence old primary
19. Rebuild old primary
20. Restore normal HA
```

The important outcome is not merely:

> **"I installed PostgreSQL, Patroni and etcd."**

The senior DBA outcome is:

```text
I can design it
     ↓
I can configure it
     ↓
I can monitor it
     ↓
I can break it
     ↓
I can diagnose it
     ↓
I can recover it
     ↓
I can validate it
     ↓
I can explain the RCA
     ↓
I can prevent recurrence
```

That is the complete **SQL Server DBA → PostgreSQL 18 Senior DBA** learning path and production lab from **Linux fundamentals through architecture, WAL, MVCC, backup/PITR, pgBackRest, streaming replication, Patroni, etcd, HAProxy, PgBouncer, performance, security, monitoring, cloud, migration, automation and production incident scenarios**.

### Current technical references

* [PostgreSQL 18 — Backup and Restore](https://www.postgresql.org/docs/18/backup.html) ([PostgreSQL][1])
* [PostgreSQL 18 — High Availability, Load Balancing and Replication](https://www.postgresql.org/docs/18/high-availability.html) ([PostgreSQL][8])
* [PostgreSQL 18 — Replication Configuration](https://www.postgresql.org/docs/18/runtime-config-replication.html) ([PostgreSQL][2])
* [PostgreSQL 18 — WAL Internals](https://www.postgresql.org/docs/18/wal-internals.html) ([PostgreSQL][4])
* [PostgreSQL 18 — Failover](https://www.postgresql.org/docs/18/warm-standby-failover.html) ([PostgreSQL][11])
* [PostgreSQL 18 — pg_verifybackup](https://www.postgresql.org/docs/18/app-pgverifybackup.html) ([PostgreSQL][5])
* [Patroni Documentation](https://patroni.readthedocs.io/en/latest/) ([Patroni][15])
* [pgBackRest User Guide](https://pgbackrest.org/user-guide.html) ([pgBackRest][7])

[1]: https://www.postgresql.org/docs/18/backup.html "PostgreSQL: Documentation: 18: Chapter 25. Backup and Restore"
[2]: https://www.postgresql.org/docs/18/runtime-config-replication.html "PostgreSQL: Documentation: 18: 19.6. Replication"
[3]: https://www.postgresql.org/docs/18/wal-intro.html "PostgreSQL: Documentation: 18: 28.3. Write-Ahead Logging (WAL)"
[4]: https://www.postgresql.org/docs/18/wal-internals.html "PostgreSQL: Documentation: 18: 28.6. WAL Internals"
[5]: https://www.postgresql.org/docs/18/app-pgverifybackup.html "PostgreSQL: Documentation: 18: pg_verifybackup"
[6]: https://www.postgresql.org/docs/18/recovery-config.html "PostgreSQL: Documentation: 18: O.1. recovery.conf file merged into postgresql.conf"
[7]: https://pgbackrest.org/user-guide.html "pgBackRest User Guide - Debian & Ubuntu"
[8]: https://www.postgresql.org/docs/18/high-availability.html "PostgreSQL: Documentation: 18: Chapter 26. High Availability, Load Balancing, and Replication"
[9]: https://www.postgresql.org/docs/18/protocol-replication.html "PostgreSQL: Documentation: 18: 54.4. Streaming Replication Protocol"
[10]: https://www.postgresql.org/docs/18/app-pgreceivewal.html "PostgreSQL: Documentation: 18: pg_receivewal"
[11]: https://www.postgresql.org/docs/18/warm-standby-failover.html "PostgreSQL: Documentation: 18: 26.3. Failover"
[12]: https://patroni.readthedocs.io/en/latest/dynamic_configuration.html "Dynamic Configuration Settings — Patroni 4.1.5 documentation"
[13]: https://patroni.readthedocs.io/en/latest/modules/patroni.ctl.html "patroni.ctl module — Patroni 4.1.5 documentation"
[14]: https://patroni.readthedocs.io/en/latest/rest_api.html "Patroni REST API — Patroni 4.1.5 documentation"
[15]: https://patroni.readthedocs.io/en/latest/README.html "Introduction — Patroni 4.1.5 documentation"

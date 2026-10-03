### 1. PostgreSQL 18 Production Directory & Filesystem Standard

A production layout covering:

* `/usr/pgsql-18` — PostgreSQL binaries
* `/pgdata/18/<cluster>/data` — primary database cluster
* `/pgwal/18/<cluster>` — separate WAL location
* `/pgtblspc/18/<cluster>/<tablespace>` — tablespaces
* `/pgarchive/18/<cluster>` — WAL archive staging
* `/pgbackup` — backup area
* `/var/log/postgresql` — PostgreSQL logs
* LVM/XFS/mount-point strategy
* Storage separation and failure-domain considerations
* SELinux contexts and permissions

### 2. PostgreSQL 18 on RHEL 9/10 — End-to-End Production Installation Runbook

The deliverable covered:

* OS and hardware prechecks
* LVM/XFS design
* Mounts and `/etc/fstab`
* PGDG repository/package installation
* PostgreSQL user/group and permissions
* SELinux configuration
* `initdb`
* Separate WAL initialization using `--waldir`
* Data checksums
* systemd configuration
* SCRAM authentication
* TLS
* Firewall/network configuration
* `pg_hba.conf`
* pgBackRest
* WAL archiving
* Streaming replication
* Monitoring
* Production validation
* Backup/restore testing
* PITR testing
* Failover testing
* Go-live checklist
* Rollback gates

### 3. PostgreSQL 18 Production Parameter/Tuning Matrix

This was the next promised deliverable, and I just provided it.

It covered six RAM sizes:

|    RAM | Server class     |
| -----: | ---------------- |
|  32 GB | Small production |
|  64 GB | Medium           |
| 128 GB | Large            |
| 256 GB | Large            |
| 512 GB | Very large       |
|   1 TB | Enterprise-scale |

Including:

* `shared_buffers`
* `effective_cache_size`
* `work_mem`
* `maintenance_work_mem`
* `autovacuum_work_mem`
* `max_connections`
* WAL/checkpoint parameters
* Autovacuum
* Parallel workers
* PostgreSQL 18 AIO
* Huge Pages
* Planner parameters
* JIT
* Logging
* Monitoring
* Production thresholds

### 4. Earlier PostgreSQL 15 → 18 RHEL Upgrade Deliverables

From the earlier discussion, there were also two production upgrade runbooks:

**A. In-place upgrade**

* PostgreSQL 15 → 18
* `pg_upgrade --check`
* `pg_upgrade`
* Package management
* Configuration review
* Validation
* Replication
* Rollback
* Production execution checklist

**B. Side-by-side upgrade**

* PG15 and PG18 installed independently
* Separate versioned binaries/data
* Migration using `pg_upgrade`
* Copy/link/clone approaches
* PostgreSQL 18 `--swap`
* Startup and validation
* Replication
* Rollback
* Production cutover checklist

The key directory principle from that earlier upgrade work was **version isolation**, e.g.:

```text
/usr/pgsql-15/
 /usr/pgsql-18/

and version-specific clusters/data.
```

So the overall deliverable set we have built is essentially:

```text
PostgreSQL 18 on RHEL
│
├── 1. Directory & Filesystem Standard
│
├── 2. Production Installation Runbook
│
├── 3. Security Configuration
│
├── 4. Backup / WAL / PITR
│
├── 5. Streaming Replication & Failover
│
├── 6. Production Validation / Go-Live
│
├── 7. PostgreSQL 18 Parameter & Tuning Matrix
│
└── 8. PostgreSQL 15 → 18 Upgrade Runbooks
      ├── In-place
      └── Side-by-side
```


For a **production PostgreSQL deployment on RHEL**, it is recommended the filesystem layout as part of the database architecture—not merely an installation detail.

The key principles are:

* Keep **PostgreSQL binaries**, **database data**, **WAL**, **backups**, and **operational logs** logically separated.
* Use dedicated LVM/filesystems where the infrastructure permits it.
* Never manually manipulate files inside `PGDATA`.
* Keep backup storage independent from the primary database filesystem.
* Use PostgreSQL **tablespaces** deliberately rather than creating arbitrary directories under `PGDATA`.
* Make ownership, permissions, SELinux contexts, mount options, monitoring, and capacity planning part of the design.

PostgreSQL itself traditionally keeps the cluster under a single `PGDATA`, commonly `/var/lib/pgsql/data`, and `PGDATA` contains `base`, `pg_wal`, `pg_tblspc`, configuration files, control files, etc. ([PostgreSQL][1])

## 1. Recommended production filesystem layout

For a serious RHEL production server, I would use something like:

```text
/
├── boot
├── etc
│   ├── postgresql/
│   └── ...
│
├── opt/
│   └── ...
│
├── usr/
│   ├── bin/
│   ├── lib/
│   └── ...
│
├── var/
│   ├── lib/
│   │   └── pgsql/
│   │       └── 18/
│   │           └── data/             <-- PGDATA
│   │
│   ├── log/
│   │   └── postgresql/               <-- PostgreSQL logs
│   │
│   └── lib/pgsql/
│
├── pgdata/
│   └── 18/
│       └── data/                     <-- Alternative PGDATA
│
├── pgwal/
│   └── 18/                           <-- Optional dedicated WAL filesystem
│
├── pgtblspc/
│   ├── 18/
│   │   ├── ts_fast/
│   │   ├── ts_data/
│   │   └── ts_archive/
│   │
├── pgbackup/
│   └── ...
│
└── pgarchive/
    └── ...                            <-- WAL archive
```

However, **do not create all of these directories blindly**. 

The final design should depend on storage architecture, workload, HA/DR requirements, backup strategy, and whether you're using SAN, local NVMe, cloud block storage, etc.

---

# 2. My preferred production layout

For a dedicated PostgreSQL production server, I would generally design the storage like this:

| Filesystem            | Purpose                      | Example              |
| --------------------- | ---------------------------- | -------------------- |
| `/`                   | OS                           | 30–50 GB             |
| `/boot`               | Boot                         | OS standard          |
| `/var`                | OS/application files         | As per RHEL standard |
| `/var/lib/pgsql/18`   | PostgreSQL installation/data | Small/medium         |
| `/pgdata`             | PostgreSQL data              | Large                |
| `/pgwal`              | WAL                          | Separate storage     |
| `/pgbackup`           | Local backup staging         | Separate             |
| `/pgarchive`          | WAL archive                  | Separate             |
| `/var/log/postgresql` | PostgreSQL logs              | Separate/log-managed |

A more explicit design:

```text
/pgdata
└── 18
    └── data
        ├── base
        ├── global
        ├── pg_wal
        ├── pg_xact
        ├── pg_multixact
        ├── pg_tblspc
        ├── pg_stat
        ├── pg_stat_tmp
        ├── pg_replslot
        ├── pg_logical
        ├── pg_notify
        ├── pg_serial
        ├── pg_snapshots
        ├── pg_subtrans
        ├── pg_twophase
        ├── postgresql.conf
        ├── pg_hba.conf
        └── PG_VERSION
```

PostgreSQL documents these directories as part of the normal cluster storage structure. ([PostgreSQL][1])

---

# 3. PostgreSQL binaries

Do **not** put PostgreSQL binaries inside your database data filesystem.

For example:

```text
/usr/pgsql-18/
├── bin/
├── lib/
├── share/
└── ...
```

For PGDG packages, a versioned installation path such as:

```text
/usr/pgsql-18/
```

is preferable because it makes parallel major-version installation and upgrades much cleaner.

For example:

```text
/usr/pgsql-15/
 /usr/pgsql-16/
 /usr/pgsql-17/
 /usr/pgsql-18/
```

This becomes particularly useful during:

```text
PostgreSQL 15
      |
      | upgrade
      v
PostgreSQL 18
```

You can retain the old binaries while the new version is being validated.

---

# 4. PGDATA

`PGDATA` is the most important directory.

Example:

```bash
/pgdata/18/data
```

Set:

```bash
export PGDATA=/pgdata/18/data
```

or configure the service appropriately.

The directory should belong exclusively to the PostgreSQL operating-system account:

```bash
chown -R postgres:postgres /pgdata/18/data
chmod 700 /pgdata/18/data
```

A production system should **not** have:

```text
root:root
755
```

for the database cluster.

You want:

```text
postgres:postgres
700
```

for the cluster root.

---

# 5. Do not manually create PGDATA subdirectories

This is an important operational rule.

Do **not** do things such as:

```bash
mkdir $PGDATA/base
mkdir $PGDATA/pg_wal
mkdir $PGDATA/global
```

PostgreSQL creates and manages these itself during:

```bash
initdb
```

For example:

```bash
initdb -D /pgdata/18/data
```

PostgreSQL's documented cluster layout includes directories such as `base`, `pg_wal`, `pg_xact`, `pg_tblspc`, etc. ([PostgreSQL][1])

**DBA rule:**

> PostgreSQL owns the internal structure of PGDATA. The OS/storage administrator owns the filesystem on which PGDATA resides.

---

# 6. Should WAL be on a separate filesystem?

Potentially, yes—but **not automatically**.

A common architecture is:

```text
/pgdata
      |
      +-- database data

/pgwal
      |
      +-- pg_wal
```

The motivation is primarily **I/O isolation and capacity management**, not simply "WAL must always be separate."

PostgreSQL's WAL directory is:

```text
$PGDATA/pg_wal
```

by default. ([PostgreSQL][1])

You can relocate WAL using a symbolic link created while PostgreSQL is stopped, but this should be done only with a controlled operational procedure.

Example architecture:

```text
/pgdata/18/data
       |
       +-- pg_wal -> /pgwal/18
```

The underlying WAL storage should ideally have:

* low latency
* high IOPS
* predictable write performance
* appropriate redundancy
* independent capacity monitoring

### Important

Do **not** separate WAL merely because it sounds like a best practice.

If your storage subsystem already provides excellent performance and redundancy, splitting WAL can add complexity without meaningful benefit.

---

# 7. Tablespaces

Tablespaces are where many production DBA designs become unnecessarily complicated.

PostgreSQL explicitly supports tablespaces for controlling the filesystem location of database objects. They can be used to place objects on different storage classes or to extend storage when the original filesystem cannot easily be expanded. ([PostgreSQL][2])

For example:

```text
/pgdata
       |
       +-- primary database storage

/pgtblspc
       |
       +-- ts_fast
       +-- ts_data
       +-- ts_archive
```

Then:

```sql
CREATE TABLESPACE ts_fast
LOCATION '/pgtblspc/18/ts_fast';
```

But don't create tablespaces for every database/table.

Use them when there is a **specific storage-management requirement**.

---

# 8. A good tablespace strategy

For example:

```text
/pgdata
   |
   +-- PostgreSQL cluster
       |
       +-- normal tables
       +-- system catalogs

/pgtblspc
   |
   +-- ts_fast
   |      +-- high-I/O indexes
   |
   +-- ts_data
   |      +-- large tables
   |
   +-- ts_archive
          +-- low-access historical data
```

The actual design should be workload-driven.

PostgreSQL documentation specifically identifies tablespaces as useful for separating objects according to storage characteristics—for example, putting heavily used indexes on faster storage and less performance-sensitive archived data on cheaper storage. ([PostgreSQL][2])

---

# 9. Never put tablespaces inside PGDATA

Avoid:

```text
/pgdata/18/data/tablespace1
```

Instead:

```text
/pgdata/18/data
/pgtblspc/18/ts_fast
```

This makes the filesystem boundary explicit.

It also reduces the possibility of accidentally:

```text
rm -rf $PGDATA
```

destroying both the cluster and tablespace data.

---

# 10. Backup directory

Do **not** consider:

```text
/pgdata/backups
```

a proper production backup architecture.

The database and its backup should not depend on the same failure domain.

Bad:

```text
Disk 1
└── /pgdata
     ├── database
     └── backups
```

If Disk 1 fails:

```text
Database -> LOST
Backup   -> LOST
```

Better:

```text
Primary storage
    |
    +-- /pgdata

Backup storage
    |
    +-- /pgbackup
```

Better still:

```text
PostgreSQL
     |
     +---- pgBackRest
              |
              +---- Repository 1
              |
              +---- Repository 2 / Object Storage
```

For production, I would strongly prefer an independent backup repository rather than relying on local backup files.

---

# 11. WAL archive directory

If you use continuous archiving/PITR:

```text
/pgarchive
```

can be used as an archive destination or staging location.

Conceptually:

```text
PostgreSQL
   |
   | WAL
   v
pg_wal
   |
   | archive_command / archive library
   v
/pgarchive
   |
   v
Backup repository
```

However, `/pgarchive` on the **same physical storage failure domain** as `/pgdata` should not be considered sufficient DR protection.

For example:

```text
/pgdata
/pgarchive
```

on the same SAN/LUN does not provide meaningful protection against that storage failure.

---

# 12. PostgreSQL logs

Don't unnecessarily fill `PGDATA` with operational log files.

A clean arrangement is:

```text
/var/log/postgresql/
```

or a dedicated filesystem such as:

```text
/pglog/
```

Depending on your organization's logging architecture.

Typical production setup:

```text
/var/log/postgresql/
    postgresql-Mon.log
    postgresql-Tue.log
    ...
```

and then forward them to your centralized logging/SIEM platform.

For example:

```text
PostgreSQL
     |
     v
/var/log/postgresql
     |
     v
rsyslog / Fluent Bit / Filebeat
     |
     v
SIEM / Log Analytics
```

Log rotation is critical.

Never allow PostgreSQL logs to consume the filesystem containing the database.

---

# 13. Configuration files

PostgreSQL traditionally stores:

```text
postgresql.conf
pg_hba.conf
pg_ident.conf
```

inside `PGDATA`, although PostgreSQL supports placing configuration files elsewhere. ([PostgreSQL][1])

For a production deployment, I generally prefer:

```text
/pgdata/18/data/postgresql.conf
/pgdata/18/data/pg_hba.conf
```

unless there is a strong configuration-management reason to externalize them.

For example:

```text
/etc/postgresql/18/
    postgresql.conf
    pg_hba.conf
    pg_ident.conf
```

can work, but then your service configuration and operational procedures must clearly document the relationship between:

```text
PGDATA
```

and:

```text
configuration_directory
```

Don't introduce complexity merely to make the directory tree look cleaner.

---

# 14. `postgresql.auto.conf`

Be aware of:

```text
postgresql.auto.conf
```

PostgreSQL uses this file for parameters set through:

```sql
ALTER SYSTEM
```

It is part of the cluster configuration structure. ([PostgreSQL][1])

Therefore, don't casually edit it with:

```bash
vi postgresql.auto.conf
```

Use:

```sql
ALTER SYSTEM SET ...
```

when `ALTER SYSTEM` is intentionally part of your configuration-management strategy.

In highly controlled environments, you may instead standardize configuration through configuration management and minimize `ALTER SYSTEM`.

---

# 15. Recommended LVM architecture

On RHEL production servers, LVM is commonly useful.

For example:

```text
VG: vg_pgsql
│
├── LV: lv_pgdata
│       └── /pgdata
│
├── LV: lv_pgwal
│       └── /pgwal
│
├── LV: lv_pglog
│       └── /pglog
│
└── LV: lv_pgbackup
        └── /pgbackup
```

But the underlying physical storage design matters more than merely creating separate LVs.

For example:

```text
lv_pgdata
lv_pgwal
```

on the **same physical disk** do not provide actual I/O isolation.

---

# 16. Better storage architecture

For a high-end production environment:

```text
                 PostgreSQL
                     |
        +------------+-------------+
        |            |             |
        v            v             v
     PGDATA         WAL          Backup
        |            |             |
        v            v             v
  Storage Pool A  Storage B    Storage C
        |            |             |
     SSD/NVMe       SSD/NVMe    Object/SAN
```

This allows you to independently monitor:

* latency
* IOPS
* throughput
* capacity
* queue depth
* filesystem utilization

---

# 17. Filesystem capacity planning

Never size:

```text
/pgdata
```

to exactly match today's database size.

If the database is:

```text
5 TB
```

don't create:

```text
5.1 TB filesystem
```

You need headroom for:

* table growth
* indexes
* WAL
* temporary files
* maintenance operations
* autovacuum
* CREATE INDEX
* VACUUM
* batch workloads
* replication slots
* operational overhead

Also remember that PostgreSQL can generate significant temporary disk usage depending on queries and maintenance operations.

---

# 18. Monitor more than filesystem %

A common mistake is monitoring only:

```text
df -h
```

Production PostgreSQL monitoring should include:

```text
Filesystem capacity
Filesystem inodes
WAL generation
WAL retention
Replication slots
Archive failures
Archive backlog
Tablespace usage
Database growth
Temp file generation
Autovacuum activity
Disk latency
IOPS
Throughput
I/O queue depth
```

A filesystem can have:

```text
30% free space
```

and PostgreSQL can still experience an operational problem because WAL/archive/replication retention is growing unexpectedly.

---

# 19. Permissions

At minimum:

```text
postgres:postgres
```

should own PostgreSQL data.

Example:

```bash
chown -R postgres:postgres /pgdata
chmod 700 /pgdata
```

Tablespace directories should similarly be owned by PostgreSQL and appropriately restricted.

Avoid:

```bash
chmod 777
```

for PostgreSQL directories.

There is virtually never a legitimate production requirement for:

```text
777
```

on PostgreSQL data.

---

# 20. SELinux

On RHEL production systems, **do not disable SELinux simply because PostgreSQL is using a non-default directory**.

If you move PostgreSQL data from the default location to something such as:

```text
/pgdata
```

you need to ensure the appropriate SELinux labeling is applied for the PostgreSQL service.

Conceptually:

```text
Default:
    /var/lib/pgsql/...

Custom:
    /pgdata/...
```

The custom path must have the appropriate PostgreSQL SELinux context.

This is one of the reasons I recommend deciding the final filesystem architecture **before** initializing the cluster.

---

# 21. Mount options

Don't blindly copy generic Linux mount options from the Internet.

Evaluate:

```text
noatime
nodiratime
barrier
discard
```

according to:

* RHEL version
* filesystem type
* storage platform
* SAN/NVMe
* cloud block storage
* vendor recommendations

For SSD/cloud storage, continuous:

```text
discard
```

may not necessarily be the best choice; scheduled TRIM may be preferable depending on the storage layer.

The PostgreSQL application should not be exposed to storage behavior that hasn't been validated.

---

# 22. XFS vs ext4

For large RHEL PostgreSQL installations, **XFS is a common choice**, particularly in enterprise environments.

Example:

```text
/dev/mapper/vg_pgsql-lv_pgdata
        |
        v
       XFS
        |
        v
      /pgdata
```

But filesystem selection should follow your organization's RHEL/storage standards and workload testing rather than treating one filesystem as universally superior.

---

# 23. Avoid NFS for PGDATA

Do not casually place:

```text
PGDATA
```

on NFS.

For primary PostgreSQL database storage, use storage designed and validated for PostgreSQL's durability and locking requirements.

Network storage can be appropriate in certain enterprise architectures, but it requires explicit validation of:

* synchronous write semantics
* fsync behavior
* locking
* latency
* failure behavior
* fencing
* recovery behavior

---

# 24. Do not use NFS as a shortcut for WAL

Especially avoid architectures like:

```text
/pgdata  -> local
/pgwal   -> NFS
```

without thorough validation.

WAL is critical to PostgreSQL durability and recovery.

The storage path used for WAL deserves particularly careful performance and failure-domain analysis.

---

# 25. Separate installation from data

I strongly recommend:

```text
PostgreSQL binaries
        |
        v
/usr/pgsql-18/

PostgreSQL data
        |
        v
/pgdata/18/data

PostgreSQL WAL
        |
        v
/pgwal/18

PostgreSQL logs
        |
        v
/var/log/postgresql

Backup
        |
        v
/pgbackup

WAL archive
        |
        v
/pgarchive
```

This gives you clean lifecycle management.

For example:

```text
Upgrade PostgreSQL
        |
        +---- binaries change
        |
        +---- PGDATA remains controlled
        |
        +---- backup remains independent
        |
        +---- logs remain independent
```

---

# 26. Multiple PostgreSQL instances

If you run multiple clusters on the same RHEL server:

```text
/pgdata/
├── 18/
│   └── prod/
│       └── data/
│
├── 18/
│   └── reporting/
│       └── data/
│
└── 17/
    └── legacy/
        └── data/
```

I prefer an explicit hierarchy:

```text
/pgdata/<major>/<cluster>/data
```

rather than:

```text
/pgdata1
/pgdata2
/pgdata3
```

This immediately tells an administrator:

```text
18/prod
18/reporting
17/legacy
```

which is much safer operationally.

---

# 27. Recommended enterprise directory standard

If I were defining a PostgreSQL RHEL production standard, I'd use:

```text
/usr/pgsql-18/
```

for binaries.

```text
/pgdata/
└── 18/
    └── prod/
        └── data/
```

for database cluster data.

```text
/pgwal/
└── 18/
    └── prod/
```

for optionally isolated WAL.

```text
/pgtblspc/
└── 18/
    └── prod/
        ├── ts_fast/
        ├── ts_data/
        └── ts_archive/
```

for deliberately designed tablespaces.

```text
/pgbackup/
└── 18/
    └── prod/
```

for local backup staging if required.

```text
/pgarchive/
└── 18/
    └── prod/
```

for WAL archive staging if required.

```text
/var/log/postgresql/
```

for logs.

That gives you:

```text
                    PostgreSQL 18
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      /usr/pgsql-18   /pgdata        /pgwal
       binaries         |              |
                        |              |
                        +-------+------+
                                |
                         PostgreSQL cluster
                                |
                    +-----------+-----------+
                    |                       |
                    v                       v
              /pgtblspc                 /pgarchive
              tablespaces               WAL archive
                    |
                    v
              /pgbackup
              backups
```

---

# 28. What I would NOT recommend

### ❌ Everything under `/`

```text
/postgres/
```

### ❌ Binaries inside PGDATA

```text
/pgdata/bin
```

### ❌ Backups inside PGDATA

```text
/pgdata/backups
```

### ❌ Logs inside the database filesystem without capacity controls

```text
/pgdata/logs
```

### ❌ Tablespaces randomly scattered across the OS

```text
/home/user/db
/tmp/db
/opt/random/db
```

### ❌ `777` permissions

```bash
chmod -R 777 /pgdata
```

### ❌ Disabling SELinux

```bash
setenforce 0
```

just because a custom PostgreSQL path was introduced.

### ❌ Putting everything on one filesystem

```text
/pgdata
/pgwal
/backup
/log
```

all backed by the same underlying storage failure domain.

---

# 29. One important PostgreSQL consideration: WAL is not a backup

This distinction is fundamental:

```text
PGDATA
  ≠
Backup
```

and:

```text
pg_wal
  ≠
Backup
```

WAL provides the write-ahead logging required for PostgreSQL durability and recovery. It becomes part of a backup/PITR architecture when appropriately archived and retained.

A production design should therefore look more like:

```text
                PostgreSQL
                    |
             +------+------+
             |             |
             v             v
          PGDATA          WAL
             |             |
             |       archive/backup
             |             |
             +------+------+
                    |
                    v
              Backup Repository
                    |
              +-----+-----+
              |           |
           Local       Remote/Object
           copy           storage
```

---

# 30. Don't forget `pg_tblspc`

One subtle but important point:

Inside `PGDATA` you'll see:

```text
pg_tblspc/
```

This directory contains symbolic links associated with PostgreSQL tablespaces. PostgreSQL documents `pg_tblspc` as part of the cluster's internal file layout. ([PostgreSQL][1])

Therefore:

**Do not manually delete or manipulate files under `pg_tblspc`.**

Manage tablespaces through PostgreSQL:

```sql
CREATE TABLESPACE ...
DROP TABLESPACE ...
ALTER TABLE ... SET TABLESPACE ...
```

rather than manually moving files around.

---

# 31. Production gold-standard layout

For a large enterprise RHEL deployment, my baseline would be:

```text
                    RHEL
                     |
      +--------------+----------------+
      |              |                |
      v              v                v
 PostgreSQL       PostgreSQL        PostgreSQL
 binaries           data              logs
      |              |                |
      v              v                v
/usr/pgsql-18     /pgdata          /var/log/postgresql
                     |
                     |
              +------+------+
              |             |
              v             v
           PGDATA          WAL
                            |
                         /pgwal
                            |
                            v
                       WAL archive
                            |
                       /pgarchive
                            |
                            v
                     Backup Repository
                            |
                     Remote/Object Storage
```

### The hierarchy I would standardize

```text
/usr/pgsql-18
/pgdata/18/<cluster>/data
/pgwal/18/<cluster>
/pgarchive/18/<cluster>
/pgtblspc/18/<cluster>/<tablespace>
/var/log/postgresql
/pgbackup
```

This is clean, auditable, upgrade-friendly, and scales well to multiple PostgreSQL clusters.

One caveat: **separate WAL, tablespaces, logs, and backup files only when there is a concrete storage/performance/operational reason**. More mount points do not automatically make PostgreSQL more reliable. PostgreSQL's own documentation supports tablespaces as a mechanism for controlling object storage locations, but they should be used deliberately. ([PostgreSQL][2])

Also, filesystem design is only one part of production PostgreSQL. Memory, huge pages, Linux resource limits, and storage behavior need to be designed together; PostgreSQL's current documentation specifically discusses these Linux resource considerations. ([PostgreSQL][3])

If you're building a **PostgreSQL 18 on RHEL 9/10 production standard**, the next logical step is to turn this into a **complete installation blueprint** covering **LVM → XFS → mount points → SELinux → `postgres` user → PGDG packages → `initdb` → systemd → WAL → tablespaces → pgBackRest → monitoring → security → pre-production validation**, with exact commands and a production-ready directory/partition sizing matrix.

[1]: https://www.postgresql.org/docs/18/storage-file-layout.html "PostgreSQL: Documentation: 18: 66.1. Database File Layout"
[2]: https://www.postgresql.org/docs/current/manage-ag-tablespaces.html "PostgreSQL: Documentation: 18: 22.6. Tablespaces"
[3]: https://www.postgresql.org/docs/18/kernel-resources.html "PostgreSQL: Documentation: 18: 18.4. Managing Kernel Resources"

===================================================================================

Below is a **production-oriented PostgreSQL 18 on RHEL 9/10 installation blueprint**. I am deliberately separating **what PostgreSQL officially supports** from infrastructure choices that should be validated against your storage platform.

PostgreSQL 18 is current, and the PostgreSQL project provides PGDG packages for Red Hat-family distributions. The PGDG repository is preferable when you need a specific supported PostgreSQL major version rather than the version supplied by the RHEL module stream. ([PostgreSQL][1])

---

# PostgreSQL 18 on RHEL — Production Installation & Filesystem Standard

## 1. Target architecture

For a production primary:

```text
                         RHEL 9/10
                            |
            +---------------+----------------+
            |               |                |
            v               v                v
       PostgreSQL        PostgreSQL       PostgreSQL
        binaries           data             logs
            |               |                |
            v               v                v
      /usr/pgsql-18      /pgdata        /var/log/postgresql
                            |
                 +----------+----------+
                 |                     |
                 v                     v
              PGDATA                  WAL
                 |                     |
                 |                  /pgwal
                 |
          +------+------+
          |             |
          v             v
      Tablespaces    PostgreSQL
      /pgtblspc       cluster
                            |
                            v
                      WAL Archive
                            |
                       /pgarchive
                            |
                            v
                    Backup Repository
                            |
                            v
                  Remote/Object Storage
```

### Recommended standard

```text
/usr/pgsql-18/
```

PostgreSQL binaries.

```text
/pgdata/18/prod/data/
```

PostgreSQL cluster.

```text
/pgwal/18/prod/
```

Optional dedicated WAL filesystem.

```text
/pgtblspc/18/prod/
```

Deliberately configured tablespaces.

```text
/pgarchive/18/prod/
```

Local WAL archive staging, if required.

```text
/pgbackup/
```

Local backup staging, if required.

```text
/var/log/postgresql/
```

PostgreSQL logs.

**Important:** separate filesystems should represent real storage/operational boundaries. Five mount points backed by the same physical LUN do not create five independent failure domains.

---

# 2. Storage design

A reasonable enterprise starting point:

| Mount                 | Purpose              | Initial sizing principle                |
| --------------------- | -------------------- | --------------------------------------- |
| `/`                   | RHEL OS              | 30–50 GB+                               |
| `/boot`               | OS boot              | RHEL standard                           |
| `/pgdata`             | PostgreSQL data      | DB size + growth + maintenance headroom |
| `/pgwal`              | WAL                  | Workload/WAL-rate driven                |
| `/pgarchive`          | WAL archive staging  | Retention/WAL-rate driven               |
| `/pgbackup`           | Local backup staging | Backup-set size driven                  |
| `/var/log/postgresql` | DB logs              | Log volume + retention                  |

Do **not** prescribe a universal `/pgwal = X GB` number.

Measure:

```sql
SELECT
    pg_size_pretty(wal_bytes) AS wal_generated
FROM pg_stat_wal;
```

PostgreSQL 18 exposes WAL statistics through `pg_stat_wal`, including `wal_bytes`. ([PostgreSQL][2])

Measure this over representative workload periods and size WAL storage from actual WAL generation, replication requirements, archiving delays, backup operations, and failure scenarios.

---

# 3. LVM design

For a dedicated server:

```text
VG: vg_pgsql

├── lv_pgdata
│      └── /pgdata
│
├── lv_pgwal
│      └── /pgwal
│
└── lv_pglog
       └── /var/log/postgresql
```

Backup storage should preferably be outside the primary database storage failure domain.

### Important distinction

This:

```text
/dev/mapper/vg_pgsql-lv_pgdata
/dev/mapper/vg_pgsql-lv_pgwal
```

does **not** automatically provide I/O isolation if both LVs reside on the same underlying physical storage.

For high-performance workloads:

```text
PGDATA -> Storage A
WAL    -> Storage B
```

can be materially different from:

```text
PGDATA -> LUN A
WAL    -> LUN B
```

where both LUNs ultimately use the same storage pool.

---

# 4. OS preparation

First validate the operating system:

```bash
cat /etc/redhat-release
uname -r
hostnamectl
getenforce
timedatectl
```

Check CPU and memory:

```bash
lscpu
free -h
```

Check storage:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
df -hT
```

Check LVM:

```bash
pvs
vgs
lvs
```

Check available repositories:

```bash
dnf repolist
```

---

# 5. Create the PostgreSQL storage

Example:

```bash
mkdir -p /pgdata
mkdir -p /pgwal
mkdir -p /pgarchive
mkdir -p /pgtblspc
mkdir -p /pgbackup
```

But in a real production deployment these should normally be **mount points**, not merely directories on `/`.

For example:

```text
/dev/mapper/vg_pgsql-lv_pgdata  -> /pgdata
/dev/mapper/vg_pgsql-lv_pgwal   -> /pgwal
```

Format according to your organization's approved filesystem standard.

For example, if XFS has been selected:

```bash
mkfs.xfs /dev/mapper/vg_pgsql-lv_pgdata
mkfs.xfs /dev/mapper/vg_pgsql-lv_pgwal
```

**Never run `mkfs` against a production volume containing data.**

---

# 6. `/etc/fstab`

Use UUIDs:

```bash
blkid
```

Example:

```text
UUID=<PGDATA_UUID>  /pgdata  xfs  defaults  0 0
UUID=<PGWAL_UUID>   /pgwal   xfs  defaults  0 0
```

Then:

```bash
mount -a
```

Validate:

```bash
findmnt /pgdata
findmnt /pgwal
df -hT /pgdata /pgwal
```

I would **not** blindly add `noatime`, `discard`, or other mount options. Validate them against the RHEL version, filesystem, storage array/cloud block device, and your organization's storage standards.

---

# 7. Install PostgreSQL 18

For RHEL, use the official PGDG repository when PostgreSQL 18 is required. The PostgreSQL project explicitly provides packages for supported Red Hat-family distributions. ([PostgreSQL][1])

The exact repository RPM should be taken from the current PostgreSQL Red Hat download page rather than hard-coding an outdated repository URL. ([PostgreSQL][1])

After enabling the repository:

```bash
dnf repolist | grep -i pgdg
```

Then:

```bash
dnf install -y \
    postgresql18-server \
    postgresql18 \
    postgresql18-contrib
```

Validate:

```bash
/usr/pgsql-18/bin/postgres --version
/usr/pgsql-18/bin/psql --version
```

Expected:

```text
postgres (PostgreSQL) 18.x
```

---

# 8. Do NOT initialize yet

This is important.

Before:

```bash
initdb
```

finalize:

* filesystem layout
* ownership
* SELinux
* mount points
* storage performance
* hostname
* DNS
* time synchronization
* firewall
* backup architecture
* cluster naming
* authentication design

`initdb` creates the complete PostgreSQL cluster structure and must be executed as the PostgreSQL operating-system user, not root. ([PostgreSQL][3])

---

# 9. PostgreSQL OS account

Check:

```bash
id postgres
```

Normally the PGDG installation creates the account.

Verify:

```bash
getent passwd postgres
getent group postgres
```

Do not run PostgreSQL as:

```text
root
```

PostgreSQL explicitly refuses to run `initdb` as root. ([PostgreSQL][3])

---

# 10. Directory ownership

Create the production hierarchy:

```bash
mkdir -p /pgdata/18/prod/data
mkdir -p /pgwal/18/prod
mkdir -p /pgarchive/18/prod
mkdir -p /pgtblspc/18/prod
```

Set ownership:

```bash
chown -R postgres:postgres /pgdata
chown -R postgres:postgres /pgwal
chown -R postgres:postgres /pgarchive
chown -R postgres:postgres /pgtblspc
```

Protect the directories:

```bash
chmod 700 /pgdata
chmod 700 /pgwal
chmod 700 /pgarchive
chmod 700 /pgtblspc
```

The actual cluster directory should ultimately have PostgreSQL's expected restrictive permissions. PostgreSQL documents `0700` directories and `0600` files for owner-only cluster access; `0750/0640` is the documented alternative when group read access is intentionally enabled. ([PostgreSQL][4])

---

# 11. SELinux — DO NOT disable it

Check:

```bash
getenforce
```

Production expectation:

```text
Enforcing
```

Do **not** solve custom PostgreSQL directory problems by doing:

```bash
setenforce 0
```

or disabling SELinux permanently.

For custom paths, establish the appropriate SELinux file context and verify it with:

```bash
ls -Zd /pgdata
ls -Zd /pgwal
```

and:

```bash
matchpathcon /pgdata
matchpathcon /pgwal
```

Red Hat documents `semanage fcontext` and `restorecon` as the mechanisms for persistently assigning and applying SELinux contexts to custom paths. ([Red Hat Documentation][5])

**Do not invent a context type.** Verify the PostgreSQL policy installed on your exact RHEL build:

```bash
semanage fcontext -l | grep -i postgres
```

Then apply the appropriate policy-supported context.

For example, after establishing the correct policy mapping:

```bash
semanage fcontext -a -e <approved_postgresql_default_path> /pgdata
restorecon -Rv /pgdata
```

The exact source path/type should be confirmed against the installed RHEL PostgreSQL SELinux policy rather than copied blindly between RHEL releases.

---

# 12. Initialize the cluster

Now initialize:

```bash
su - postgres
```

Set:

```bash
export PGDATA=/pgdata/18/prod/data
```

Then:

```bash
/usr/pgsql-18/bin/initdb \
    -D "$PGDATA" \
    --waldir=/pgwal/18/prod
```

This is an excellent approach because PostgreSQL itself supports `initdb --waldir` for placing WAL separately. ([PostgreSQL][3])

This is preferable to manually moving `pg_wal` after initialization.

### Why?

You get:

```text
/pgdata/18/prod/data
```

for the main cluster and:

```text
/pgwal/18/prod
```

for WAL from the beginning.

---

# 13. PostgreSQL 18 checksums

PostgreSQL 18 changes the default behavior: `initdb` now enables data checksums by default. PostgreSQL documents `--no-data-checksums` for explicitly disabling them. ([PostgreSQL][6])

For a new production cluster:

```bash
/usr/pgsql-18/bin/pg_checksums \
    --check \
    -D /pgdata/18/prod/data
```

after the cluster is stopped, when applicable.

Verify from SQL:

```sql
SHOW data_checksums;
```

Expected:

```text
on
```

---

# 14. Authentication during initialization

Be careful here.

PostgreSQL documentation notes that `trust` is the default for ease of installation, but explicitly warns not to use it unless you trust all local users. ([PostgreSQL][7])

For production, don't blindly accept:

```text
trust
```

in `pg_hba.conf`.

A controlled initial configuration should move toward:

```text
scram-sha-256
```

for password authentication where appropriate.

Example:

```text
local   all             all                         scram-sha-256
host    all             all    10.10.0.0/16       scram-sha-256
```

The actual CIDRs should be your application/network ranges.

---

# 15. Configure PostgreSQL

Your primary configuration:

```text
/pgdata/18/prod/data/postgresql.conf
```

Security:

```text
/pgdata/18/prod/data/pg_hba.conf
```

Check:

```bash
ls -l /pgdata/18/prod/data/
```

---

# 16. Core production configuration

Do **not** copy random tuning values from the Internet.

At minimum, explicitly review:

```text
listen_addresses
port
max_connections
shared_buffers
effective_cache_size
work_mem
maintenance_work_mem
huge_pages
shared_preload_libraries
wal_level
max_wal_size
min_wal_size
checkpoint_timeout
checkpoint_completion_target
max_wal_senders
max_replication_slots
wal_keep_size
archive_mode
archive_command / archive_library
random_page_cost
effective_io_concurrency
log_destination
logging_collector
log_directory
log_filename
log_rotation_age
log_rotation_size
log_min_duration_statement
log_checkpoints
log_connections
log_disconnections
log_lock_waits
password_encryption
ssl
```

These should be derived from:

* RAM
* CPU
* connection model
* workload
* storage latency
* WAL generation
* HA architecture
* backup requirements

rather than copied as fixed numbers.

---

# 17. WAL architecture

A production environment should normally use:

```text
wal_level = replica
```

when physical replication and/or continuous archiving is required.

PostgreSQL documents that `replica` generates enough WAL for WAL archiving and physical replication. ([PostgreSQL][8])

Then:

```text
archive_mode = on
```

and either:

```text
archive_command = ...
```

or:

```text
archive_library = ...
```

PostgreSQL 18 supports both mechanisms. ([PostgreSQL][8])

---

# 18. Don't treat `/pgarchive` as the final DR repository

This:

```text
Primary
 |
 +-- /pgarchive
```

is only useful if that storage itself is appropriately protected.

For real DR:

```text
Primary
   |
   +---- WAL
          |
          v
      Backup system
          |
       +--+--+
       |     |
       v     v
    Local  Remote
           Object/SAN
```

PostgreSQL's PITR architecture combines a base backup with the required WAL archive. ([PostgreSQL][9])

---

# 19. pgBackRest

For production environments, I would normally introduce **pgBackRest** rather than building a collection of ad-hoc `cp` scripts.

Architecture:

```text
PostgreSQL
     |
     | base backups
     | WAL
     v
 pgBackRest
     |
     +------------------+
     |                  |
     v                  v
Local Repository     Remote Repository
                         |
                         v
                   Object Storage
```

Your actual repository design should follow your RPO/RTO and organization's backup standards.

The critical principle is:

```text
Database storage
       ≠
Backup storage
```

---

# 20. PostgreSQL-native backup validation

Even if pgBackRest is your enterprise backup tool, PostgreSQL's native tooling is valuable for understanding backup integrity.

A PostgreSQL 18 base backup can contain a backup manifest, and:

```bash
pg_verifybackup
```

can validate it. PostgreSQL explicitly states that `pg_verifybackup` checks the backup against the generated manifest, including files and required WAL, but also recommends actual test restores because verification cannot detect every possible restore problem. ([PostgreSQL][10])

Therefore:

```text
Backup successful
        ≠
Restore proven
```

Your operational standard should be:

```text
Backup
  ↓
Verify
  ↓
Restore
  ↓
Validate database
  ↓
Measure RTO
```

---

# 21. Log architecture

Configure:

```text
/var/log/postgresql/
```

rather than allowing unlimited logs to consume `/pgdata`.

Example:

```text
/var/log/postgresql/
├── postgresql-Mon.log
├── postgresql-Tue.log
├── postgresql-Wed.log
└── ...
```

Use controlled rotation and centralized collection.

Monitor:

```text
FATAL
ERROR
PANIC
archive failures
connection failures
authentication failures
deadlocks
lock waits
checkpoint warnings
replication failures
```

---

# 22. systemd

After initialization:

```bash
systemctl status postgresql-18
```

Depending on the PGDG packaging/service naming in your installed release, verify the exact unit:

```bash
systemctl list-unit-files | grep -i postgres
```

Then enable the validated service unit:

```bash
systemctl enable postgresql-18
```

and:

```bash
systemctl start postgresql-18
```

The PostgreSQL project's Red Hat-family packaging documentation notes that the PGDG installation does not automatically initialize/start PostgreSQL; initialization and service enablement/start are explicit post-installation steps. ([PostgreSQL][1])

---

# 23. Validate the cluster

```bash
systemctl status postgresql-18
```

Then:

```bash
su - postgres
```

```bash
psql
```

Run:

```sql
SELECT version();

SHOW data_directory;

SHOW config_file;

SHOW hba_file;

SHOW port;

SHOW listen_addresses;

SHOW data_checksums;

SHOW wal_level;

SHOW archive_mode;
```

---

# 24. Validate WAL location

Run:

```sql
SELECT pg_ls_dir('pg_wal');
```

Then from Linux:

```bash
ls -ld /pgwal/18/prod
```

Also:

```bash
psql -c "SHOW data_directory;"
```

and:

```bash
readlink -f /pgdata/18/prod/data/pg_wal
```

With `initdb --waldir`, the WAL directory is deliberately located separately. PostgreSQL supports this directly through `initdb`. ([PostgreSQL][3])

---

# 25. Validate storage

```bash
df -hT
df -ih
```

Then:

```bash
findmnt /pgdata
findmnt /pgwal
```

Check:

```bash
lsblk -o NAME,KNAME,SIZE,FSTYPE,MOUNTPOINTS
```

Check I/O:

```bash
iostat -xz 5 10
```

Look for:

```text
await
util
r/s
w/s
rkB/s
wkB/s
```

For serious performance validation, use workload-specific benchmarking rather than relying solely on synthetic tests.

---

# 26. PostgreSQL storage validation

```sql
SELECT
    pg_size_pretty(pg_database_size(datname)) AS size,
    datname
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Tablespace:

```sql
SELECT
    spcname,
    pg_size_pretty(pg_tablespace_size(spcname))
FROM pg_tablespace;
```

WAL:

```sql
SELECT
    wal_records,
    wal_fpi,
    pg_size_pretty(wal_bytes) AS wal_generated,
    wal_buffers_full
FROM pg_stat_wal;
```

PostgreSQL 18 provides extensive statistics through `pg_stat_wal`, `pg_stat_io`, `pg_stat_archiver`, and related views. ([PostgreSQL][11])

---

# 27. WAL archive validation

Run:

```sql
SELECT *
FROM pg_stat_archiver;
```

Pay particular attention to:

```text
archived_count
last_archived_wal
last_archived_time
failed_count
last_failed_wal
last_failed_time
```

These are explicitly exposed by PostgreSQL's `pg_stat_archiver` view. ([PostgreSQL][2])

### Critical alert

```text
failed_count increasing
```

or:

```text
last_archived_time becoming stale
```

should be treated as a backup/PITR health problem—not merely a logging problem.

---

# 28. Replication validation

If you have a standby:

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

Also monitor:

```text
replication lag
replay lag
WAL retention
replication slots
archive lag
```

---

# 29. Replication slots — storage danger

Check:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn
FROM pg_replication_slots;
```

An inactive replication slot can cause WAL retention.

That means:

```text
replication slot problem
        ↓
WAL retention
        ↓
pg_wal growth
        ↓
/pgwal fills
        ↓
PostgreSQL outage
```

This is one of the most important filesystem-related PostgreSQL alerts.

---

# 30. Critical filesystem alerts

I would establish at least:

### `/pgdata`

```text
Warning: 70%
Critical: 80–85%
```

### `/pgwal`

Use both:

```text
filesystem utilization
+
WAL retention
```

Don't rely solely on filesystem percentage.

### `/pgarchive`

Alert on:

```text
archive backlog
archive failure
stale last archive time
```

### `/var/log/postgresql`

Alert on:

```text
filesystem capacity
```

and ensure rotation works.

Exact thresholds should be adjusted to your capacity, growth rate, and incident response time.

---

# 31. Security baseline

### PostgreSQL OS user

```text
postgres
```

No interactive operational use beyond what your organization permits.

### Cluster

```text
700
```

### Cluster files

```text
600
```

### Configuration

Restrict:

```text
postgresql.conf
pg_hba.conf
pg_ident.conf
```

### Authentication

Prefer:

```text
scram-sha-256
```

rather than:

```text
trust
```

### Network

Don't expose:

```text
5432
```

to the entire network.

Firewall:

```text
Application subnet
       |
       v
     5432
       |
       v
PostgreSQL
```

not:

```text
Internet
   |
   v
  5432
```

---

# 32. SSL/TLS

Production application connectivity should normally be evaluated for TLS requirements.

Configuration:

```text
ssl = on
```

Then provide properly secured:

```text
server certificate
private key
CA chain
```

Private key permissions are especially important.

Do not place private keys in:

```text
777
```

or application-readable locations unnecessarily.

---

# 33. Backup architecture

My preferred production architecture:

```text
                     PRIMARY
                        |
                +-------+-------+
                |               |
                v               v
              PGDATA           WAL
                |               |
                +-------+-------+
                        |
                        v
                    pgBackRest
                        |
              +---------+---------+
              |                   |
              v                   v
        Local Repository      Remote Repository
                                  |
                                  v
                           Object Storage
```

Then:

```text
                 BACKUP
                   |
       +-----------+-----------+
       |                       |
       v                       v
    Verify                  Restore
                               |
                               v
                         DR Validation
```

PostgreSQL officially recognizes SQL dumps, filesystem-level backups, and continuous archiving as distinct backup approaches. ([PostgreSQL][12])

---

# 34. RPO/RTO must determine the design

Before finalizing:

```text
RPO = ?
RTO = ?
```

Example:

```text
RPO: 5 minutes
RTO: 30 minutes
```

could lead to:

```text
Streaming replication
+
WAL archiving
+
frequent backups
+
automated monitoring
+
tested failover
```

Whereas:

```text
RPO: 24 hours
RTO: 8 hours
```

could support a substantially different architecture.

Don't choose storage topology before defining these requirements.

---

# 35. Production pre-go-live checklist

## OS

```text
[ ] RHEL supported version
[ ] CPU validated
[ ] RAM validated
[ ] Time synchronization
[ ] DNS
[ ] Hostname
[ ] SELinux enforcing
[ ] Firewall
[ ] Security baseline
[ ] OS patch baseline
```

## Storage

```text
[ ] /pgdata created
[ ] /pgwal created if required
[ ] XFS/ext4 approved
[ ] LVM validated
[ ] Mounts persistent
[ ] Storage redundancy validated
[ ] IOPS tested
[ ] Latency tested
[ ] Capacity headroom
[ ] inode capacity
```

## PostgreSQL

```text
[ ] PostgreSQL 18 installed
[ ] PGDATA documented
[ ] WAL location documented
[ ] Ownership verified
[ ] Permissions verified
[ ] SELinux context verified
[ ] data checksums enabled
[ ] authentication configured
[ ] SSL configured
[ ] logging configured
[ ] parameters reviewed
```

## Backup

```text
[ ] Base backup working
[ ] WAL archive working
[ ] Backup repository independent
[ ] Backup retention configured
[ ] Backup verification working
[ ] Restore tested
[ ] PITR tested
[ ] RPO measured
[ ] RTO measured
```

## HA

```text
[ ] Standby built
[ ] Streaming replication healthy
[ ] Replication slots monitored
[ ] WAL retention monitored
[ ] Failover tested
[ ] Rejoin procedure tested
[ ] Split-brain prevention
```

## Monitoring

```text
[ ] CPU
[ ] RAM
[ ] filesystem
[ ] IOPS
[ ] latency
[ ] WAL generation
[ ] WAL retention
[ ] archive failures
[ ] replication lag
[ ] connection count
[ ] locks
[ ] deadlocks
[ ] long transactions
[ ] autovacuum
[ ] bloat
[ ] checkpoints
[ ] backup status
```

---

# 36. Final production directory standard

If I were writing the organization's **PostgreSQL RHEL Standard Operating Procedure**, I'd make this the baseline:

```text
/
├── usr/
│   └── pgsql-18/
│       ├── bin/
│       ├── lib/
│       └── share/
│
├── pgdata/
│   └── 18/
│       └── prod/
│           └── data/
│               ├── base/
│               ├── global/
│               ├── pg_commit_ts/
│               ├── pg_dynshmem/
│               ├── pg_logical/
│               ├── pg_multixact/
│               ├── pg_notify/
│               ├── pg_replslot/
│               ├── pg_serial/
│               ├── pg_snapshots/
│               ├── pg_stat/
│               ├── pg_stat_tmp/
│               ├── pg_subtrans/
│               ├── pg_tblspc/
│               ├── pg_twophase/
│               ├── pg_wal/  -> /pgwal/18/prod/
│               ├── postgresql.conf
│               ├── pg_hba.conf
│               ├── pg_ident.conf
│               └── ...
│
├── pgwal/
│   └── 18/
│       └── prod/
│
├── pgtblspc/
│   └── 18/
│       └── prod/
│           ├── ts_fast/
│           ├── ts_data/
│           └── ts_archive/
│
├── pgarchive/
│   └── 18/
│       └── prod/
│
├── pgbackup/
│
└── var/
    └── log/
        └── postgresql/
```

The important part is that PostgreSQL itself manages the internal cluster structure. `initdb` creates that structure, and PostgreSQL's documented storage layout includes directories such as `base`, `global`, `pg_wal`, and `pg_tblspc`. ([PostgreSQL][3])

---

# 37. The production architecture in one picture

```text
                         ┌───────────────────────┐
                         │       RHEL 9/10       │
                         └───────────┬───────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
       /usr/pgsql-18             /pgdata                 /pgwal
          Binaries                PGDATA                   WAL
              │                      │                      │
              │                      └──────────┬───────────┘
              │                                 │
              │                                 ▼
              │                         PostgreSQL 18
              │                                 │
              │                  ┌──────────────┼──────────────┐
              │                  │              │              │
              │                  ▼              ▼              ▼
              │             Tablespaces       WAL          Statistics
              │                  │              │
              │                  ▼              ▼
              │             /pgtblspc      /pgarchive
              │                                 │
              │                                 ▼
              │                          Backup Platform
              │                                 │
              │                       ┌─────────┴─────────┐
              │                       │                   │
              │                       ▼                   ▼
              │                  Local Repository    Remote/Object
              │                                        Storage
              │
              └───────────────────────┐
                                      ▼
                              /var/log/postgresql
                                      │
                                      ▼
                                Central SIEM
```

## The five rules I would make mandatory

**1.** `/pgdata` is for PostgreSQL data—not application files, scripts, backups, or random logs.

**2.** `/pgwal` is separated only when storage architecture/workload justifies it; use `initdb --waldir` rather than manually rearranging `pg_wal`. ([PostgreSQL][3])

**3.** `/pgbackup` or a backup repository must not be treated as a substitute for independent/remote backup storage.

**4.** SELinux remains **Enforcing**; custom PostgreSQL paths receive the appropriate persistent SELinux labeling rather than disabling security controls. ([Red Hat Documentation][5])

**5.** A backup is not considered production-ready until **restore/PITR has actually been tested**. PostgreSQL's own documentation makes the distinction between backup verification and a real restore test. ([PostgreSQL][10])

### Official references

* [PostgreSQL 18 — Red Hat-family installation](https://www.postgresql.org/download/linux/redhat/)
* [PostgreSQL 18 — `initdb`](https://www.postgresql.org/docs/18/app-initdb.html)
* [PostgreSQL 18 — Backup and Restore](https://www.postgresql.org/docs/18/backup.html)
* [PostgreSQL 18 — Continuous Archiving/PITR](https://www.postgresql.org/docs/18/continuous-archiving.html)
* [RHEL 9 — SELinux documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/using_selinux/index)

[1]: https://www.postgresql.org/download/linux/redhat/ "PostgreSQL: Linux downloads (Red Hat family)"
[2]: https://www.postgresql.org/docs/current/monitoring-stats.html "PostgreSQL: Documentation: 18: 27.2. The Cumulative Statistics System"
[3]: https://www.postgresql.org/docs/18/app-initdb.html "PostgreSQL: Documentation: 18: initdb"
[4]: https://www.postgresql.org/docs/current/creating-cluster.html "PostgreSQL: Documentation: 18: 18.2. Creating a Database Cluster"
[5]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/using_selinux/index "Using SELinux | Red Hat Enterprise Linux | 9 | Red Hat Documentation"
[6]: https://www.postgresql.org/docs/18/release-18.html "PostgreSQL: Documentation: 18: E.6. Release 18"
[7]: https://www.postgresql.org/docs/current/app-initdb.html "PostgreSQL: Documentation: 18: initdb"
[8]: https://www.postgresql.org/docs/current/runtime-config-wal.html "PostgreSQL: Documentation: 18: 19.5. Write Ahead Log"
[9]: https://www.postgresql.org/docs/18/continuous-archiving.html "PostgreSQL: Documentation: 18: 25.3. Continuous Archiving and Point-in-Time Recovery (PITR)"
[10]: https://www.postgresql.org/docs/18/app-pgverifybackup.html "PostgreSQL: Documentation: 18: pg_verifybackup"
[11]: https://www.postgresql.org/docs/18/monitoring.html "PostgreSQL: Documentation: 18: Chapter 27. Monitoring Database Activity"
[12]: https://www.postgresql.org/docs/18/backup.html "PostgreSQL: Documentation: 18: Chapter 25. Backup and Restore"

==============================================================================================================================

Absolutely. Below is the **PostgreSQL 18 production tuning matrix for RHEL**, designed as a **safe starting baseline**, not as a substitute for workload-specific benchmarking.

PostgreSQL 18 introduces the new AIO subsystem, so I’ve included the PG18-specific I/O parameters as well. PostgreSQL 18 documents `worker`, `io_uring`, and `sync` for `io_method`; the default is `worker`. ([PostgreSQL][1])

# PostgreSQL 18 Production Parameter Matrix — RHEL

## 1. Server sizing assumptions

The following matrix assumes:

* Dedicated PostgreSQL server
* RHEL 9/10
* XFS
* SSD/NVMe storage
* Production OLTP/mixed workload
* Streaming replication may be used
* pgBackRest/PITR
* Application connection pooling preferred
* `shared_buffers` approximately 25% of RAM as the initial baseline
* No other major applications consuming RAM

**Important:** RAM alone does not determine every PostgreSQL parameter. CPU count, storage latency/IOPS, database size, transaction rate, concurrency and query profile matter just as much.

---

# 2. Core memory parameters

| Parameter              |  32 GB |  64 GB | 128 GB | 256 GB | 512 GB |      1 TB |
| ---------------------- | -----: | -----: | -----: | -----: | -----: | --------: |
| `shared_buffers`       |   8 GB |  16 GB |  32 GB |  64 GB | 128 GB |    256 GB |
| `effective_cache_size` |  24 GB |  48 GB |  96 GB | 192 GB | 384 GB |    768 GB |
| `work_mem`             |  16 MB |  32 MB |  32 MB |  64 MB |  64 MB | 64–128 MB |
| `maintenance_work_mem` |   1 GB |   2 GB |   4 GB |   8 GB |  16 GB |     32 GB |
| `autovacuum_work_mem`  | 256 MB | 512 MB | 512 MB |   1 GB |   2 GB |    2–4 GB |
| `temp_buffers`         |   8 MB |   8 MB |   8 MB |  16 MB |  16 MB |     16 MB |

### Critical warning about `work_mem`

Do **not** calculate:

```text
work_mem = RAM / max_connections
```

That is unsafe.

`work_mem` can be consumed **multiple times within a single query**, and parallel queries can multiply memory consumption further. PostgreSQL's own documentation specifically points to `work_mem`, `shared_buffers`, and excessive connections as potential contributors to memory exhaustion. ([PostgreSQL][2])

For example:

```text
1 query
  ├── Sort
  ├── Hash Join
  ├── Hash Aggregate
  └── another Sort
```

can consume considerably more than one `work_mem`.

Therefore:

> **Start conservative. Increase only after `EXPLAIN (ANALYZE, BUFFERS)` and workload measurements demonstrate a benefit.**

---

# 3. `shared_buffers`

A good production starting point is approximately:

```text
25% of physical RAM
```

So:

```text
32 GB   → 8 GB
64 GB   → 16 GB
128 GB  → 32 GB
256 GB  → 64 GB
512 GB  → 128 GB
1 TB    → 256 GB
```

This is a **starting point**, not a mandatory formula.

For very large systems, don't automatically increase `shared_buffers` to 50–70% of RAM.

PostgreSQL also relies heavily on the operating-system page cache.

---

# 4. `effective_cache_size`

This parameter does **not allocate memory**.

It tells the PostgreSQL planner approximately how much memory is available for filesystem caching.

Example starting point:

```text
32 GB   → 24 GB
64 GB   → 48 GB
128 GB  → 96 GB
256 GB  → 192 GB
512 GB  → 384 GB
1 TB    → 768 GB
```

If other applications consume memory, reduce this accordingly.

---

# 5. Connection management

This is one of the most important areas.

### Starting baseline

|    RAM | Suggested initial `max_connections` |
| -----: | ----------------------------------: |
|  32 GB |                                 150 |
|  64 GB |                                 200 |
| 128 GB |                                 300 |
| 256 GB |                                 400 |
| 512 GB |                                 500 |
|   1 TB |                                 500 |

These numbers are **not RAM-derived limits**. They are conservative starting caps for a dedicated database server.

Prefer:

```text
Applications
     ↓
PgBouncer / connection pool
     ↓
PostgreSQL
```

rather than:

```text
5,000 application connections
          ↓
5,000 PostgreSQL backends
```

PostgreSQL documentation explicitly notes that reducing `max_connections` and using external connection pooling can be preferable when memory exhaustion is a concern. ([PostgreSQL][2])

### Also consider

```conf
superuser_reserved_connections = 5
```

Keep administrative capacity available during connection storms.

---

# 6. WAL and checkpoint configuration

For production:

```conf
wal_level = replica
fsync = on
full_page_writes = on
synchronous_commit = on
checkpoint_completion_target = 0.9
checkpoint_timeout = 15min
```

`checkpoint_completion_target=0.9` is already PostgreSQL's documented default and is designed to spread checkpoint I/O across the checkpoint interval. ([PostgreSQL][3])

### Initial WAL sizing

|    RAM | `min_wal_size` | `max_wal_size` |
| -----: | -------------: | -------------: |
|  32 GB |           2 GB |           8 GB |
|  64 GB |           4 GB |          16 GB |
| 128 GB |           8 GB |          32 GB |
| 256 GB |          16 GB |          64 GB |
| 512 GB |          32 GB |         128 GB |
|   1 TB |          64 GB |         256 GB |

But **WAL rate is more important than RAM**.

For example, if your system generates:

```text
500 MB WAL/minute
```

then:

```text
30 minutes ≈ 15 GB WAL
```

So `max_wal_size` must be evaluated against actual WAL generation.

PostgreSQL describes `max_wal_size` as a **soft limit**, meaning WAL can exceed it under certain conditions. ([PostgreSQL][3])

Monitor:

```sql
SELECT
    wal_records,
    wal_fpi,
    wal_bytes,
    wal_buffers_full,
    stats_reset
FROM pg_stat_wal;
```

---

# 7. Do NOT disable these in normal production

Keep:

```conf
fsync = on
full_page_writes = on
synchronous_commit = on
```

Do not use:

```conf
fsync = off
full_page_writes = off
```

just to obtain benchmark performance.

PostgreSQL explicitly documents these as non-durable settings that can compromise durability. ([PostgreSQL][4])

---

# 8. WAL buffers

Use:

```conf
wal_buffers = -1
```

Let PostgreSQL calculate the value unless workload testing proves otherwise.

Don't blindly configure:

```conf
wal_buffers = 1GB
```

just because the server has large RAM.

---

# 9. Autovacuum — production baseline

This is where many PostgreSQL production environments require significant attention.

### Recommended starting point

```conf
autovacuum = on
track_counts = on

autovacuum_naptime = 10s

autovacuum_vacuum_scale_factor = 0.02
autovacuum_analyze_scale_factor = 0.01

autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50
```

PostgreSQL's defaults are considerably less aggressive: `autovacuum_vacuum_scale_factor=0.2` and `autovacuum_analyze_scale_factor=0.1`. PostgreSQL allows these parameters to be overridden at table level, which is extremely important for large/high-churn tables. ([PostgreSQL][5])

For a table containing:

```text
1,000,000,000 rows
```

20% means:

```text
200,000,000 changed rows
```

before the scale-factor component triggers vacuum.

That is usually unacceptable for a heavily updated production table.

---

# 10. Autovacuum workers

Starting point:

|    RAM | `autovacuum_max_workers` |
| -----: | -----------------------: |
|  32 GB |                        5 |
|  64 GB |                        6 |
| 128 GB |                        8 |
| 256 GB |                       10 |
| 512 GB |                       12 |
|   1 TB |                       16 |

But CPU, number of databases and number of high-churn tables matter.

PostgreSQL 18 also has:

```conf
autovacuum_worker_slots
```

The documentation notes that `autovacuum_max_workers` cannot effectively exceed the available worker slots. ([PostgreSQL][5])

Don't simply increase both to huge numbers.

---

# 11. Per-table autovacuum tuning

For very large/high-churn tables, use table-level settings.

Example:

```sql
ALTER TABLE public.orders
SET (
    autovacuum_vacuum_scale_factor = 0.005,
    autovacuum_analyze_scale_factor = 0.002
);
```

For a billion-row table:

```text
0.5% = 5 million rows
0.2% = 2 million rows
```

This is far more controllable than waiting for 200 million changes.

---

# 12. CPU / parallelism

Parallelism should be based primarily on **CPU**, not RAM.

A useful starting model:

| vCPU | `max_worker_processes` | `max_parallel_workers` | `max_parallel_workers_per_gather` |
| ---: | ---------------------: | ---------------------: | --------------------------------: |
|    8 |                     12 |                      4 |                                 2 |
|   16 |                     20 |                      8 |                                 2 |
|   32 |                     40 |                     16 |                               2–4 |
|   64 |                     72 |                     32 |                                 4 |
|   96 |                    104 |                     48 |                                 4 |
|  128 |                    136 |                     64 |                               4–8 |

These are **starting values**, not universal recommendations.

PostgreSQL limits parallel query workers through `max_parallel_workers_per_gather`, while the total available parallel workers are constrained by `max_worker_processes` and `max_parallel_workers`. ([PostgreSQL][6])

For a latency-sensitive OLTP workload, don't assume:

```conf
max_parallel_workers_per_gather = 8
```

is better than:

```conf
max_parallel_workers_per_gather = 2
```

Too much parallelism can create CPU contention.

---

# 13. PostgreSQL 18 AIO — important new area

PostgreSQL 18 introduces asynchronous I/O.

The relevant parameters include:

```conf
io_method
io_workers
io_max_concurrency
io_combine_limit
effective_io_concurrency
maintenance_io_concurrency
```

PostgreSQL 18 supports:

```text
io_method = worker
io_method = io_uring
io_method = sync
```

The documented default is:

```conf
io_method = worker
```

`io_uring` requires a PostgreSQL build with liburing support. ([PostgreSQL][1])

### Production baseline

Start with:

```conf
io_method = worker
io_workers = 3
```

Don't immediately change to:

```conf
io_method = io_uring
```

just because the server is running Linux.

Benchmark it against your actual RHEL kernel, PostgreSQL build and storage stack.

PostgreSQL 18 also raised the defaults for:

```text
effective_io_concurrency
maintenance_io_concurrency
```

to 16 to better reflect modern hardware. ([PostgreSQL][7])

---

# 14. Huge Pages

For large-memory PostgreSQL servers, investigate explicit huge pages.

Especially:

```text
256 GB
512 GB
1 TB
```

PostgreSQL provides:

```sql
SHOW shared_memory_size;
SHOW shared_memory_size_in_huge_pages;
SHOW huge_pages;
SHOW huge_page_size;
```

And Linux:

```bash
grep -i Huge /proc/meminfo
```

PostgreSQL documentation specifically explains that huge pages can reduce overhead when using large shared-memory allocations and provides `shared_memory_size_in_huge_pages` to calculate the requirement. ([PostgreSQL][8])

### Recommended approach

Initially:

```conf
huge_pages = try
```

Validate:

```sql
SHOW huge_pages_status;
```

Then, once the OS configuration is proven stable, consider:

```conf
huge_pages = on
```

if your operational policy prefers PostgreSQL to fail startup rather than silently fall back when huge pages aren't available.

---

# 15. Query planner baseline

Start with:

```conf
seq_page_cost = 1.0
random_page_cost = 1.1–1.5
```

**only benchmark the lower `random_page_cost` values on SSD/NVMe.**

Don't blindly use:

```conf
random_page_cost = 1.0
```

on every system.

Also:

```conf
default_statistics_target = 100
```

Then increase statistics selectively:

```sql
ALTER TABLE orders
ALTER COLUMN customer_id
SET STATISTICS 500;
```

for columns with significant data skew or complex predicates.

---

# 16. JIT

For traditional OLTP:

```conf
jit = off
```

is a reasonable **baseline to benchmark**.

For analytical workloads, leave JIT enabled and test it.

PostgreSQL documentation notes that JIT is primarily beneficial for longer, CPU-bound queries; for short queries, compilation overhead can exceed the benefit. ([PostgreSQL][9])

Therefore:

```text
OLTP       → benchmark JIT OFF
Analytics  → benchmark JIT ON
Mixed      → workload dependent
```

---

# 17. Logging baseline

Production:

```conf
logging_collector = on

log_checkpoints = on
log_lock_waits = on

log_autovacuum_min_duration = 1s

log_line_prefix = '%m [%p] %u@%d %r '

log_rotation_age = 1d
log_rotation_size = 100MB
```

For troubleshooting slow SQL:

```conf
log_min_duration_statement = 1000
```

can be temporarily useful.

But don't permanently log every statement on a high-throughput production system unless you have explicitly evaluated the volume and overhead.

---

# 18. Monitoring parameters

Enable/use:

```conf
track_io_timing = on
```

and monitor:

```sql
pg_stat_activity
pg_stat_database
pg_stat_bgwriter
pg_stat_wal
pg_stat_io
pg_stat_progress_vacuum
```

PostgreSQL 18's AIO subsystem also introduces:

```text
pg_aios
```

for observing asynchronous I/O activity. ([PostgreSQL][7])

---

# 19. Critical production thresholds

These should become monitoring/alerting rules.

| Metric                         |           Warning |              Critical |
| ------------------------------ | ----------------: | --------------------: |
| CPU                            |    >70% sustained |                  >90% |
| Memory                         |              >75% |                  >90% |
| Swap activity                  |     Any sustained |                 Heavy |
| Disk utilization               |              >70% |               >85–90% |
| Disk latency                   | workload-specific | sustained degradation |
| WAL growth                     |          abnormal |          uncontrolled |
| Replication lag                |         SLA-based |            SLA breach |
| Replication slot WAL retention |          abnormal |             disk-risk |
| Long transaction               |            >5 min |               >30 min |
| Idle-in-transaction            |            >5 min |               >15 min |
| Autovacuum duration            |       investigate |      blocking/lagging |
| Dead tuples                    |        increasing |      sustained growth |
| Checkpoint frequency           |         excessive |     checkpoint storms |
| Connection utilization         |              >70% |                  >90% |
| Temp file growth               |          abnormal |             disk-risk |

These should ultimately be based on your **normal production baseline and SLA**, rather than universal numbers.

---

# 20. My recommended starting `postgresql.conf`

For a **128 GB / 32-vCPU OLTP server**, for example:

```conf
#------------------------------------------------------------------------------
# CONNECTIONS
#------------------------------------------------------------------------------

max_connections = 300
superuser_reserved_connections = 5


#------------------------------------------------------------------------------
# MEMORY
#------------------------------------------------------------------------------

shared_buffers = 32GB
effective_cache_size = 96GB

work_mem = 32MB
maintenance_work_mem = 4GB
autovacuum_work_mem = 512MB

temp_buffers = 8MB


#------------------------------------------------------------------------------
# WAL
#------------------------------------------------------------------------------

wal_level = replica

fsync = on
full_page_writes = on
synchronous_commit = on

wal_buffers = -1

min_wal_size = 8GB
max_wal_size = 32GB

checkpoint_timeout = 15min
checkpoint_completion_target = 0.9


#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------

autovacuum = on
track_counts = on

autovacuum_naptime = 10s

autovacuum_max_workers = 8

autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50

autovacuum_vacuum_scale_factor = 0.02
autovacuum_analyze_scale_factor = 0.01


#------------------------------------------------------------------------------
# PARALLELISM
#------------------------------------------------------------------------------

max_worker_processes = 40
max_parallel_workers = 16
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2


#------------------------------------------------------------------------------
# PG18 AIO
#------------------------------------------------------------------------------

io_method = worker
io_workers = 3


#------------------------------------------------------------------------------
# PLANNER
#------------------------------------------------------------------------------

seq_page_cost = 1.0
random_page_cost = 1.2

default_statistics_target = 100


#------------------------------------------------------------------------------
# JIT
#------------------------------------------------------------------------------

jit = off


#------------------------------------------------------------------------------
# LOGGING
#------------------------------------------------------------------------------

logging_collector = on

log_checkpoints = on
log_lock_waits = on

log_autovacuum_min_duration = 1s

log_line_prefix = '%m [%p] %u@%d %r '

log_rotation_age = 1d
log_rotation_size = 100MB


#------------------------------------------------------------------------------
# OBSERVABILITY
#------------------------------------------------------------------------------

track_io_timing = on
```

**I would not deploy this configuration unchanged.** Treat it as the initial production baseline, then tune using measured workload data.

---

# 21. Parameters I would NOT blindly tune

Avoid copying Internet tuning scripts containing things like:

```conf
shared_buffers = 50GB
work_mem = 256MB
maintenance_work_mem = 16GB
max_connections = 2000
random_page_cost = 1
effective_io_concurrency = 1000
autovacuum_max_workers = 20
```

without workload evidence.

Those values can look impressive while making a production system less stable.

The correct DBA approach is:

```text
Hardware
   ↓
OS configuration
   ↓
PostgreSQL baseline
   ↓
Workload measurement
   ↓
Identify bottleneck
   ↓
Change ONE parameter/group
   ↓
Benchmark
   ↓
Observe production
   ↓
Retain / revert
```

---

# 22. The most important production DBA rule

Don't tune PostgreSQL based on:

```text
"How much RAM do I have?"
```

Tune based on:

```text
RAM
+
CPU
+
Storage latency
+
IOPS
+
WAL generation
+
TPS
+
Query concurrency
+
Database size
+
Working-set size
+
Autovacuum workload
+
Replication
+
Application connection behavior
```

That distinction is particularly important on **256 GB–1 TB servers**.

PostgreSQL 18's AIO changes also make it worthwhile to benchmark the I/O subsystem rather than carrying forward PostgreSQL 15/16/17 tuning assumptions unchanged. ([PostgreSQL][7])

### Production architecture I would target

```text
                         ┌──────────────────────┐
                         │    Application Tier   │
                         └──────────┬───────────┘
                                    │
                             Connection Pool
                                    │
                         ┌──────────▼───────────┐
                         │     PostgreSQL 18     │
                         │                       │
                         │ shared_buffers        │
                         │ work_mem              │
                         │ WAL / checkpoints     │
                         │ Autovacuum             │
                         │ PG18 AIO               │
                         └──────┬────────┬───────┘
                                │        │
                    ┌───────────┘        └───────────┐
                    ▼                                ▼
              Fast Data Storage                 WAL Storage
                    │                                │
                    ▼                                ▼
                XFS / LVM                         XFS / LVM
                    │                                │
                    └──────────────┬─────────────────┘
                                   ▼
                              pgBackRest
                                   │
                                   ▼
                            Object Storage
```

This gives you a **solid PostgreSQL 18/RHEL production starting point** while keeping the values deliberately conservative and measurable.

[1]: https://www.postgresql.org/docs/18/runtime-config-resource.html "PostgreSQL: Documentation: 18: 19.4. Resource Consumption"
[2]: https://www.postgresql.org/docs/18/kernel-resources.html "PostgreSQL: Documentation: 18: 18.4. Managing Kernel Resources"
[3]: https://www.postgresql.org/docs/current/runtime-config-wal.html "PostgreSQL: Documentation: 18: 19.5. Write Ahead Log"
[4]: https://www.postgresql.org/docs/18/non-durability.html "PostgreSQL: Documentation: 18: 14.5. Non-Durable Settings"
[5]: https://www.postgresql.org/docs/18/runtime-config-vacuum.html "PostgreSQL: Documentation: 18: 19.10. Vacuuming"
[6]: https://www.postgresql.org/docs/18/how-parallel-query-works.html "PostgreSQL: Documentation: 18: 15.1. How Parallel Query Works"
[7]: https://www.postgresql.org/docs/18/release-18.html "PostgreSQL: Documentation: 18: E.6. Release 18"
[8]: https://www.postgresql.org/files/documentation/pdf/18/postgresql-18-US.pdf "PostgreSQL 18.1 Documentation"
[9]: https://www.postgresql.org/docs/18/jit-decision.html "PostgreSQL: Documentation: 18: 30.2. When to JIT?"

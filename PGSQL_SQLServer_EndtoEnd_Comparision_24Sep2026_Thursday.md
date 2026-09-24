The fastest way to learn PostgreSQL is to treat **SQL Server as the reference point** and map PostgreSQL concepts to the closest SQL Server equivalent.

Below is an **end-to-end PostgreSQL ↔ SQL Server DBA correlation map**, organized in the order recommended for learning/administering them.

> **Important:** These are conceptual mappings, not statements that the two products behave identically. Where there is no true equivalent, marked it as **No direct equivalent**.

# PostgreSQL ↔ SQL Server — End-to-End DBA Comparison

## 1. Core Architecture

| #  | PostgreSQL                              | SQL Server                                 | Easy DBA Understanding                                                                                                                                                  |
| -- | --------------------------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | PostgreSQL Server                       | SQL Server Database Engine                 | Main database engine                                                                                                                                                    |
| 2  | PostgreSQL instance/process environment | SQL Server Instance                        | PostgreSQL architecture is process-based; SQL Server is primarily service/process based                                                                                 |
| 3  | Cluster                                 | SQL Server Instance                        | **Important:** PostgreSQL "cluster" is a collection of databases managed by one server installation/data directory; it is not equivalent to SQL Server Failover Cluster |
| 4  | Database                                | Database                                   | Similar logical container                                                                                                                                               |
| 5  | Schema                                  | Schema                                     | Very similar                                                                                                                                                            |
| 6  | Table                                   | Table                                      | Same basic concept                                                                                                                                                      |
| 7  | View                                    | View                                       | Same                                                                                                                                                                    |
| 8  | Materialized View                       | Indexed View                               | Similar purpose, but implementation differs significantly                                                                                                               |
| 9  | Function                                | Function / Stored Procedure                | PostgreSQL functions are heavily used; SQL Server separates functions and procedures more distinctly                                                                    |
| 10 | Procedure                               | Stored Procedure                           | Similar                                                                                                                                                                 |
| 11 | Extension                               | SQL Server component/feature               | PostgreSQL extensions such as `pg_stat_statements` add functionality                                                                                                    |
| 12 | PostgreSQL backend process              | SQL Server worker/session architecture     | PostgreSQL creates backend processes for client sessions                                                                                                                |
| 13 | Postmaster terminology                  | SQL Server Database Engine service         | Historical PostgreSQL term; modern documentation generally refers to the PostgreSQL server                                                                              |
| 14 | WAL                                     | Transaction Log                            | One of the most important mappings                                                                                                                                      |
| 15 | Shared buffers                          | Buffer Pool                                | PostgreSQL shared_buffers ≈ SQL Server buffer pool concept                                                                                                              |
| 16 | Work memory                             | Query workspace memory                     | Similar purpose to SQL Server query memory grants, but not equivalent                                                                                                   |
| 17 | Maintenance_work_mem                    | Maintenance memory                         | Used by VACUUM, CREATE INDEX, etc.                                                                                                                                      |
| 18 | Temp buffers                            | TempDB buffer usage                        | Related concept, but PostgreSQL temporary storage architecture differs                                                                                                  |
| 19 | Background writer                       | SQL Server background/checkpoint processes | Writes dirty buffers                                                                                                                                                    |
| 20 | Checkpointer                            | SQL Server Checkpoint                      | Similar responsibility                                                                                                                                                  |
| 21 | WAL writer                              | SQL Server Log Writer                      | Writes WAL records                                                                                                                                                      |
| 22 | Autovacuum                              | No direct SQL Server equivalent            | PostgreSQL-specific critical subsystem                                                                                                                                  |
| 23 | Vacuum                                  | No direct equivalent                       | Some objectives overlap with SQL Server maintenance, but mechanism is different                                                                                         |
| 24 | MVCC                                    | SQL Server row-versioning/MVCC             | Both support MVCC concepts, but implementation differs                                                                                                                  |
| 25 | Visibility Map                          | No direct equivalent                       | PostgreSQL storage optimization                                                                                                                                         |
| 26 | Free Space Map                          | No direct equivalent                       | Tracks reusable space inside PostgreSQL relations                                                                                                                       |

---

# 2. Installation & Services

| PostgreSQL                    | SQL Server                                            | DBA Mapping                                 |
| ----------------------------- | ----------------------------------------------------- | ------------------------------------------- |
| PostgreSQL package            | SQL Server installation media/package                 | Software installation                       |
| `postgres` OS user            | SQL Server service account                            | Database service identity                   |
| `postgresql.service`          | SQL Server service                                    | Start/stop database engine                  |
| `systemctl start postgresql`  | `Start-Service MSSQLSERVER`                           | Start service                               |
| `systemctl stop postgresql`   | `Stop-Service MSSQLSERVER`                            | Stop service                                |
| `systemctl status postgresql` | `Get-Service MSSQLSERVER`                             | Check service                               |
| `pg_lsclusters`               | SQL Server Configuration Manager / instance discovery | PostgreSQL-specific cluster management      |
| `pg_ctl`                      | SQL Server service/control utilities                  | Server control                              |
| `psql`                        | SSMS/sqlcmd                                           | Primary command-line client                 |
| pgAdmin                       | SSMS                                                  | GUI administration                          |
| PostgreSQL port 5432          | SQL Server port 1433                                  | Default TCP port                            |
| Multiple PostgreSQL clusters  | Multiple SQL Server instances                         | Similar concept, but implementation differs |

---

# 3. PostgreSQL Cluster vs SQL Server Instance

This causes significant confusion for SQL Server DBAs.

| PostgreSQL                                                     | SQL Server                                                          |
| -------------------------------------------------------------- | ------------------------------------------------------------------- |
| PostgreSQL Cluster                                             | Closest conceptual equivalent: SQL Server Instance                  |
| Cluster contains multiple databases                            | Instance contains multiple databases                                |
| One PostgreSQL server process architecture manages the cluster | SQL Server instance manages databases                               |
| Each cluster normally has its own port/data directory          | Each SQL Server instance has its own system databases/configuration |
| PostgreSQL cluster ≠ Windows Failover Cluster                  | SQL Server FCI is an HA technology                                  |

### Example

```text
PostgreSQL
   Cluster
      ├── postgres
      ├── template0
      ├── template1
      ├── appdb
      └── salesdb
```

SQL Server:

```text
SQL Server Instance
      ├── master
      ├── model
      ├── msdb
      ├── tempdb
      ├── AppDB
      └── SalesDB
```

---

# 4. Configuration

| PostgreSQL                   | SQL Server                                      | Purpose                                  |
| ---------------------------- | ----------------------------------------------- | ---------------------------------------- |
| `postgresql.conf`            | SQL Server server configuration                 | Main PostgreSQL configuration            |
| `pg_hba.conf`                | SQL Server security/login/network configuration | Client authentication/access             |
| `listen_addresses`           | SQL Server TCP/IP configuration                 | Listening interfaces                     |
| `port = 5432`                | TCP 1433                                        | Network port                             |
| `max_connections`            | `user connections` / connection limits          | Concurrent connections                   |
| `shared_buffers`             | Buffer Pool configuration                       | Memory for data pages                    |
| `work_mem`                   | Query memory grant concepts                     | Query operation memory                   |
| `maintenance_work_mem`       | Index/maintenance memory concepts               | Maintenance operations                   |
| `effective_cache_size`       | No direct equivalent                            | Planner assumption about available cache |
| `random_page_cost`           | No direct equivalent                            | Planner cost model                       |
| `effective_io_concurrency`   | No direct equivalent                            | I/O behavior                             |
| `wal_level`                  | Recovery/replication configuration              | WAL functionality                        |
| `max_wal_size`               | Transaction log growth/checkpoint behavior      | WAL/checkpoint control                   |
| `checkpoint_timeout`         | Checkpoint-related configuration                | Checkpoint frequency                     |
| `autovacuum`                 | No direct equivalent                            | Automatic vacuum/analyze                 |
| `log_min_duration_statement` | Query duration logging / Extended Events        | Slow query detection                     |
| `log_statement`              | SQL Audit / Extended Events                     | Statement logging                        |

---

# 5. Authentication & Security

| PostgreSQL                  | SQL Server                                         |
| --------------------------- | -------------------------------------------------- |
| Role                        | Login/User depending on context                    |
| Login role                  | SQL Server Login                                   |
| Database role               | Database Role                                      |
| `pg_hba.conf`               | SQL Server authentication/access configuration     |
| SCRAM-SHA-256               | SQL authentication/password authentication         |
| LDAP authentication         | AD/LDAP integration                                |
| Kerberos                    | Kerberos/Windows Authentication                    |
| SSL/TLS                     | TLS encryption                                     |
| `GRANT`                     | `GRANT`                                            |
| `REVOKE`                    | `REVOKE`                                           |
| `pg_roles`                  | `sys.server_principals`                            |
| `pg_authid`                 | Security catalog concepts                          |
| `information_schema.role_*` | Security metadata                                  |
| `ALTER ROLE`                | `ALTER LOGIN` / `ALTER USER`                       |
| Role membership             | Role membership                                    |
| Row-Level Security          | Row-Level Security                                 |
| `SECURITY DEFINER`          | `EXECUTE AS` concepts                              |
| `search_path`               | Default schema/database object resolution concepts |

---

# 6. Database & Schema

| PostgreSQL         | SQL Server                                    |
| ------------------ | --------------------------------------------- |
| Database           | Database                                      |
| Schema             | Schema                                        |
| `public` schema    | `dbo` is the closest commonly used schema     |
| `CREATE DATABASE`  | `CREATE DATABASE`                             |
| `CREATE SCHEMA`    | `CREATE SCHEMA`                               |
| Database owner     | Database owner                                |
| Schema owner       | Schema owner                                  |
| Tablespaces        | Filegroups                                    |
| Database encoding  | Database/column collation & encoding concepts |
| `LC_COLLATE`       | Collation                                     |
| `LC_CTYPE`         | Character classification/locale               |
| `CREATE EXTENSION` | Install/enable SQL Server feature/component   |

---

# 7. Storage Architecture

| PostgreSQL                          | SQL Server                                           |
| ----------------------------------- | ---------------------------------------------------- |
| Data directory                      | SQL Server data directory                            |
| Relation                            | Database object/storage structure                    |
| Heap table                          | SQL Server heap                                      |
| PostgreSQL page                     | SQL Server 8-KB page                                 |
| PostgreSQL page size generally 8 KB | SQL Server page = 8 KB                               |
| Tuple                               | Row                                                  |
| TOAST                               | No direct equivalent                                 |
| FSM                                 | No direct equivalent                                 |
| Visibility Map                      | No direct equivalent                                 |
| WAL                                 | Transaction log                                      |
| Tablespace                          | Filegroup                                            |
| Relation fork                       | No direct equivalent                                 |
| `pg_relation_size()`                | `sys.dm_db_partition_stats`, `sp_spaceused` concepts |
| `pg_total_relation_size()`          | Object/database space usage                          |
| `pg_database_size()`                | Database size                                        |

---

# 8. Tables

| PostgreSQL                     | SQL Server                                                             |
| ------------------------------ | ---------------------------------------------------------------------- |
| `CREATE TABLE`                 | `CREATE TABLE`                                                         |
| Heap table                     | Heap                                                                   |
| Primary key                    | Primary key                                                            |
| Foreign key                    | Foreign key                                                            |
| Unique constraint              | Unique constraint                                                      |
| Check constraint               | Check constraint                                                       |
| Default                        | Default constraint                                                     |
| Generated column               | Computed column                                                        |
| Identity column                | Identity column                                                        |
| Serial                         | Older PostgreSQL auto-number mechanism; SQL Server closest is IDENTITY |
| `GENERATED ALWAYS AS IDENTITY` | `IDENTITY`                                                             |
| Temporary table                | `#temp` / temporary tables                                             |
| Unlogged table                 | No direct equivalent                                                   |
| Partitioned table              | Partitioned table                                                      |
| Inheritance                    | No direct equivalent                                                   |
| `RETURNING`                    | `OUTPUT`                                                               |

---

# 9. Data Types

| PostgreSQL                 | SQL Server                       |
| -------------------------- | -------------------------------- |
| `smallint`                 | `smallint`                       |
| `integer`                  | `int`                            |
| `bigint`                   | `bigint`                         |
| `numeric`                  | `decimal/numeric`                |
| `real`                     | `real`                           |
| `double precision`         | `float`                          |
| `money`                    | `money`                          |
| `boolean`                  | `bit`                            |
| `char`                     | `char`                           |
| `varchar`                  | `varchar`                        |
| `text`                     | `varchar(max)` conceptually      |
| `date`                     | `date`                           |
| `time`                     | `time`                           |
| `timestamp`                | `datetime2` conceptually         |
| `timestamp with time zone` | `datetimeoffset` conceptually    |
| `uuid`                     | `uniqueidentifier`               |
| `json`                     | JSON stored in character columns |
| `jsonb`                    | No exact equivalent              |
| `bytea`                    | `varbinary`                      |
| Array                      | No direct equivalent             |
| Range types                | No direct equivalent             |
| `inet`                     | No direct equivalent             |
| `hstore`                   | No direct equivalent             |

---

# 10. SQL Language

| PostgreSQL          | SQL Server                                      |
| ------------------- | ----------------------------------------------- |
| PostgreSQL SQL      | T-SQL                                           |
| `LIMIT`             | `TOP`                                           |
| `OFFSET ... LIMIT`  | `OFFSET ... FETCH`                              |
| `RETURNING`         | `OUTPUT`                                        |
| `ILIKE`             | `LIKE` with appropriate collation/case behavior |
| `COALESCE`          | `COALESCE`                                      |
| `NULLIF`            | `NULLIF`                                        |
| `CASE`              | `CASE`                                          |
| `STRING_AGG`        | `STRING_AGG`                                    |
| `GENERATE_SERIES()` | Recursive CTE / tally-table approaches          |
| `FILTER`            | Conditional aggregation                         |
| `LATERAL`           | `CROSS APPLY` / `OUTER APPLY` conceptually      |
| `WITH`              | CTE                                             |
| Window functions    | Window functions                                |
| `EXPLAIN`           | Execution plan                                  |
| `EXPLAIN ANALYZE`   | Actual execution plan                           |
| `VACUUM`            | No direct equivalent                            |

---

# 11. Stored Procedures & Functions

| PostgreSQL          | SQL Server                      |
| ------------------- | ------------------------------- |
| Function            | Function                        |
| Procedure           | Stored procedure                |
| PL/pgSQL            | T-SQL                           |
| `CREATE FUNCTION`   | `CREATE FUNCTION`               |
| `CREATE PROCEDURE`  | `CREATE PROCEDURE`              |
| `CALL`              | `EXEC`                          |
| `RAISE NOTICE`      | `PRINT` / logging mechanisms    |
| `RAISE EXCEPTION`   | `THROW`                         |
| `EXCEPTION` block   | `TRY...CATCH`                   |
| Function volatility | No direct SQL Server equivalent |
| `IMMUTABLE`         | No direct equivalent            |
| `STABLE`            | No direct equivalent            |
| `VOLATILE`          | No direct equivalent            |
| `SECURITY DEFINER`  | `EXECUTE AS` concept            |

---

# 12. Indexes

| PostgreSQL             | SQL Server                                 |
| ---------------------- | ------------------------------------------ |
| B-tree                 | B-tree                                     |
| Hash                   | Hash index concepts                        |
| GiST                   | No direct equivalent                       |
| SP-GiST                | No direct equivalent                       |
| GIN                    | No direct equivalent                       |
| BRIN                   | No direct equivalent                       |
| Partial index          | Filtered index                             |
| Expression index       | Function-based/indexed expression concepts |
| Unique index           | Unique index                               |
| Multicolumn index      | Composite index                            |
| `INCLUDE` columns      | Included columns                           |
| `REINDEX`              | `ALTER INDEX ... REBUILD`                  |
| `REINDEX CONCURRENTLY` | Online index rebuild concept               |
| `pg_stat_user_indexes` | `sys.dm_db_index_usage_stats`              |
| `pg_stat_all_indexes`  | Index DMVs                                 |

### Important

A SQL Server DBA should remember:

**PostgreSQL has significantly more index access-method flexibility than SQL Server.**

Especially:

```text
B-tree
GIN
GiST
SP-GiST
BRIN
Hash
```

---

# 13. Partitioning

| PostgreSQL               | SQL Server                               |
| ------------------------ | ---------------------------------------- |
| Declarative partitioning | Table partitioning                       |
| Range partition          | RANGE partition                          |
| List partition           | LIST partition                           |
| Hash partition           | HASH partition                           |
| Partition key            | Partitioning column                      |
| Partitioned table        | Partitioned table                        |
| Partitioned index        | Partitioned index                        |
| Partition pruning        | Partition elimination                    |
| Default partition        | No exact equivalent                      |
| Attach partition         | Add/switch-like administrative operation |
| Detach partition         | Partition switching/removal concepts     |
| Partition constraint     | Partition boundary                       |
| `pg_partition_tree()`    | Partition metadata DMVs                  |
| `pg_partman`             | No native exact equivalent               |

### DBA translation

```text
PostgreSQL partition pruning
        ↓
SQL Server partition elimination
```

---

# 14. Transactions

| PostgreSQL         | SQL Server                                                  |
| ------------------ | ----------------------------------------------------------- |
| `BEGIN`            | `BEGIN TRANSACTION`                                         |
| `COMMIT`           | `COMMIT`                                                    |
| `ROLLBACK`         | `ROLLBACK`                                                  |
| Savepoint          | Savepoint                                                   |
| `SAVEPOINT`        | `SAVE TRANSACTION`                                          |
| MVCC               | Row-versioning/MVCC                                         |
| Transaction ID/XID | Transaction metadata                                        |
| Snapshot           | Transaction isolation snapshot                              |
| `pg_stat_activity` | DMVs such as `sys.dm_exec_sessions`, `sys.dm_exec_requests` |

---

# 15. Isolation Levels

| PostgreSQL                 | SQL Server                                                                    |
| -------------------------- | ----------------------------------------------------------------------------- |
| Read Uncommitted behavior  | READ UNCOMMITTED                                                              |
| Read Committed             | READ COMMITTED                                                                |
| Repeatable Read            | REPEATABLE READ                                                               |
| Serializable               | SERIALIZABLE                                                                  |
| Snapshot-style behavior    | SNAPSHOT                                                                      |
| PostgreSQL Repeatable Read | Stronger semantics than SQL Server's traditional Repeatable Read              |
| MVCC-based reads           | Lock/version-based mechanisms depending on SQL Server isolation configuration |

### Critical difference

Do **not** assume:

```text
PostgreSQL READ COMMITTED = SQL Server READ COMMITTED
```

The names match, but the underlying behavior and concurrency implementation differ.

---

# 16. Locking & Blocking

| PostgreSQL               | SQL Server                                       |
| ------------------------ | ------------------------------------------------ |
| Lock manager             | Lock Manager                                     |
| `pg_locks`               | `sys.dm_tran_locks`                              |
| `pg_stat_activity`       | `sys.dm_exec_requests`                           |
| Blocking PID             | Blocking session ID                              |
| `pg_blocking_pids()`     | Blocking session/DMV queries                     |
| `pg_terminate_backend()` | `KILL`                                           |
| Row locks                | Row/key locks                                    |
| Table locks              | Table locks                                      |
| Advisory locks           | Application locks (`sp_getapplock`) conceptually |
| Deadlock detection       | Deadlock detection                               |
| Deadlock log             | Extended Events/system diagnostics               |

---

# 17. Query Performance

| PostgreSQL            | SQL Server                             |
| --------------------- | -------------------------------------- |
| `EXPLAIN`             | Estimated execution plan               |
| `EXPLAIN ANALYZE`     | Actual execution plan                  |
| `pg_stat_statements`  | Query Store / DMVs                     |
| `pg_stat_activity`    | `sys.dm_exec_requests`                 |
| `pg_stat_database`    | Database-level DMVs                    |
| `pg_stat_user_tables` | Table/index DMVs                       |
| `auto_explain`        | Extended Events / Query Store concepts |
| Planner               | Query Optimizer                        |
| Cost-based optimizer  | Cost-based optimizer                   |
| Sequential Scan       | Table Scan                             |
| Index Scan            | Index Seek/Scan depending on plan      |
| Bitmap Heap Scan      | No exact SQL Server equivalent         |
| Bitmap Index Scan     | Bitmap operations in SQL Server plans  |
| Nested Loop           | Nested Loops                           |
| Hash Join             | Hash Match                             |
| Merge Join            | Merge Join                             |
| Sort                  | Sort                                   |
| Aggregate             | Stream/Hash Aggregate                  |

---

# 18. Execution Plan Translation

| PostgreSQL          | SQL Server                     |
| ------------------- | ------------------------------ |
| Seq Scan            | Table Scan                     |
| Index Scan          | Index Scan                     |
| Index Only Scan     | Covering/index-only concept    |
| Bitmap Heap Scan    | No direct operator equivalent  |
| Nested Loop         | Nested Loops                   |
| Hash Join           | Hash Match                     |
| Merge Join          | Merge Join                     |
| Sort                | Sort                           |
| Aggregate           | Aggregate                      |
| Materialize         | Spool/materialization concepts |
| Memoize             | No direct common equivalent    |
| Gather              | Parallelism                    |
| Gather Merge        | Parallelism + ordered merge    |
| Parallel Seq Scan   | Parallel Table Scan            |
| Parallel Index Scan | Parallel Index Scan            |
| Planning Time       | Compile/optimization time      |
| Execution Time      | Query execution duration       |

---

# 19. Statistics

| PostgreSQL                  | SQL Server                                 |
| --------------------------- | ------------------------------------------ |
| `ANALYZE`                   | `UPDATE STATISTICS`                        |
| Autovacuum Analyze          | Auto statistics maintenance                |
| `pg_stats`                  | `sys.stats`, statistics DMVs               |
| Extended statistics         | Multi-column/extended statistics concepts  |
| Histogram                   | Histogram                                  |
| Most common values          | Frequency information                      |
| `default_statistics_target` | Statistics sampling/configuration concepts |
| Planner statistics          | Query optimizer statistics                 |

---

# 20. Vacuum & Bloat

This is one of the **largest conceptual differences** for a SQL Server DBA.

| PostgreSQL     | SQL Server                                                           |
| -------------- | -------------------------------------------------------------------- |
| VACUUM         | No direct equivalent                                                 |
| VACUUM FULL    | No direct exact equivalent; resembles table rebuild/space compaction |
| Autovacuum     | No direct equivalent                                                 |
| Table bloat    | Fragmentation/unused space, but not identical                        |
| Dead tuples    | Row versions/obsolete row versions conceptually                      |
| Freeze         | No SQL Server equivalent                                             |
| XID wraparound | No SQL Server equivalent                                             |
| Visibility Map | No SQL Server equivalent                                             |
| `pg_repack`    | Rebuild/online maintenance concepts                                  |

### SQL Server DBA mindset

In SQL Server:

```text
Index fragmentation
→ REORGANIZE / REBUILD
```

PostgreSQL:

```text
Dead tuples
→ VACUUM
→ AUTOVACUUM
→ VACUUM FULL / pg_repack when appropriate
```

These should **not** be treated as identical operations.

---

# 21. Backup

| PostgreSQL           | SQL Server                            |
| -------------------- | ------------------------------------- |
| `pg_dump`            | Logical backup / scripting equivalent |
| `pg_dumpall`         | Instance-level logical scripting      |
| `pg_basebackup`      | Physical full/base backup concept     |
| Filesystem backup    | File/database backup                  |
| WAL archive          | Transaction log backup/archive stream |
| WAL archiving        | Log backup/archive mechanism concept  |
| PITR                 | Point-in-time restore                 |
| Base backup + WAL    | Full + transaction logs conceptually  |
| Continuous Archiving | Log-based recovery architecture       |
| `pg_restore`         | Restore logical backup                |
| `psql` restore       | Script execution                      |
| `restore_command`    | Recovery log retrieval mechanism      |

---

# 22. PITR

PostgreSQL:

```text
Base Backup
      +
Archived WAL
      +
Recovery Target
      ↓
Point-in-Time Recovery
```

SQL Server:

```text
Full Backup
      +
Differential Backup
      +
Transaction Log Backups
      ↓
Point-in-Time Restore
```

| PostgreSQL             | SQL Server                  |
| ---------------------- | --------------------------- |
| Base backup            | Full backup                 |
| WAL archive            | Transaction log backups     |
| Recovery target time   | STOPAT                      |
| Recovery target LSN    | LSN-based recovery concepts |
| `restore_command`      | Backup restore chain        |
| Recovery signal        | Recovery configuration      |
| `recovery_target_time` | `STOPAT`                    |
| `recovery_target_lsn`  | LSN-oriented recovery       |
| Timeline               | No direct equivalent        |

---

# 23. WAL vs SQL Server Transaction Log

| PostgreSQL WAL      | SQL Server Transaction Log            |
| ------------------- | ------------------------------------- |
| Write-Ahead Logging | Write-Ahead Logging                   |
| WAL records         | Log records                           |
| WAL segments        | Transaction-log virtual structures    |
| WAL archive         | Log backup/archive                    |
| LSN                 | LSN                                   |
| WAL sender          | Log/replication sender concepts       |
| WAL receiver        | Secondary/log receiver concepts       |
| `pg_wal`            | Transaction log files                 |
| `wal_level`         | Recovery/replication configuration    |
| `archive_mode`      | Log backup/archive strategy           |
| `archive_command`   | Backup/archive mechanism              |
| Replication slot    | Replication-state retention mechanism |

---

# 24. High Availability

| PostgreSQL               | SQL Server                                     |
| ------------------------ | ---------------------------------------------- |
| Streaming Replication    | Availability Group/log-based secondary concept |
| Physical Standby         | AG secondary / DR secondary concept            |
| Synchronous replication  | Synchronous AG commit                          |
| Asynchronous replication | Asynchronous AG commit                         |
| Hot Standby              | Readable secondary                             |
| Primary                  | Primary replica                                |
| Standby                  | Secondary replica                              |
| WAL sender               | Primary replication sender                     |
| WAL receiver             | Secondary receiver                             |
| Replication slot         | No exact SQL Server equivalent                 |
| Patroni                  | No direct SQL Server equivalent                |
| EFM                      | No direct SQL Server equivalent                |
| pg_auto_failover         | No direct equivalent                           |
| repmgr                   | No direct equivalent                           |

---

# 25. PostgreSQL HA Architecture

Typical:

```text
                Application
                     |
                  HAProxy
                     |
              +------+------+
              |             |
          PostgreSQL     PostgreSQL
           Primary        Standby
              |             |
              +---- WAL -----+
```

SQL Server equivalent concept:

```text
                Application
                     |
                  Listener
                     |
              +------+------+
              |             |
          SQL Primary    SQL Secondary
              |
          Transaction Log
```

But PostgreSQL requires an HA manager such as **Patroni** in many production architectures because PostgreSQL itself provides replication primitives but not a complete automatic cluster-management layer equivalent to SQL Server AG's integrated orchestration.

---

# 26. Replication

| PostgreSQL           | SQL Server                                          |
| -------------------- | --------------------------------------------------- |
| Physical replication | AG/log shipping-style physical replication concepts |
| Logical replication  | Transactional replication / CDC-like use cases      |
| Publication          | Replication publication                             |
| Subscription         | Replication subscriber                              |
| Publisher            | Publisher                                           |
| Subscriber           | Subscriber                                          |
| Replication slot     | No exact equivalent                                 |
| WAL sender           | Replication sender                                  |
| WAL receiver         | Replication receiver                                |
| Logical decoding     | CDC/change-stream concepts                          |
| Replication origin   | No direct equivalent                                |

---

# 27. SQL Server Always On AG vs PostgreSQL Streaming Replication

| Area                    | PostgreSQL                                        | SQL Server                             |
| ----------------------- | ------------------------------------------------- | -------------------------------------- |
| Primary                 | Primary                                           | Primary replica                        |
| Secondary               | Standby                                           | Secondary replica                      |
| Sync replication        | Synchronous                                       | Synchronous commit                     |
| Async replication       | Asynchronous                                      | Asynchronous commit                    |
| Read-only secondary     | Hot Standby                                       | Readable secondary                     |
| Automatic failover      | Usually HA manager required                       | Built-in AG functionality              |
| Listener                | External HA/load-balancing solution commonly used | AG Listener                            |
| Quorum                  | External/HA manager dependent                     | WSFC/quorum for traditional Windows AG |
| Replication unit        | Cluster/physical WAL stream                       | Database replica                       |
| Database-level failover | Different architecture                            | AG database-level                      |

---

# 28. Log Shipping

| PostgreSQL              | SQL Server               |
| ----------------------- | ------------------------ |
| WAL archiving + restore | Log shipping             |
| Archive WAL             | Copy transaction log     |
| Restore WAL             | Restore transaction log  |
| Standby recovery        | Secondary database       |
| `restore_command`       | Log-shipping restore job |
| Recovery target         | Standby recovery         |

PostgreSQL does not have a built-in feature literally named **Log Shipping**, but WAL archiving plus continuous recovery can provide a comparable DR architecture.

---

# 29. Disaster Recovery

| PostgreSQL                 | SQL Server                       |
| -------------------------- | -------------------------------- |
| WAL archiving              | Transaction log backups          |
| PITR                       | Point-in-time restore            |
| Streaming replica          | AG secondary                     |
| Delayed standby            | Delayed log shipping/DR concepts |
| Cross-region replica       | Cross-region AG/DR secondary     |
| Object-storage WAL archive | Cloud backup storage             |
| Base backup                | Full backup                      |
| Timeline                   | No direct equivalent             |

---

# 30. Jobs & Scheduling

| PostgreSQL                     | SQL Server                          |
| ------------------------------ | ----------------------------------- |
| pgAgent                        | SQL Server Agent                    |
| cron                           | SQL Agent scheduling / OS scheduler |
| systemd timer                  | SQL Agent schedule                  |
| `pg_cron`                      | SQL Server Agent                    |
| pgBackRest scheduled jobs      | SQL Agent backup jobs               |
| Shell scripts                  | PowerShell/Cmd                      |
| PL/pgSQL maintenance procedure | T-SQL maintenance procedure         |

### Very important

PostgreSQL **does not have a native SQL Server Agent equivalent built into the core database engine**.

Common choices:

```text
pg_cron
pgAgent
Linux cron
systemd timers
Kubernetes CronJobs
External automation
```

---

# 31. Monitoring

| PostgreSQL                      | SQL Server                                 |
| ------------------------------- | ------------------------------------------ |
| `pg_stat_activity`              | `sys.dm_exec_requests` / sessions          |
| `pg_stat_database`              | Database DMVs                              |
| `pg_stat_user_tables`           | Table/index DMVs                           |
| `pg_stat_user_indexes`          | `sys.dm_db_index_usage_stats`              |
| `pg_locks`                      | `sys.dm_tran_locks`                        |
| `pg_stat_bgwriter`              | Database engine I/O/checkpoint diagnostics |
| `pg_stat_wal`                   | Transaction log/WAL diagnostics            |
| `pg_stat_replication`           | AG/replication DMVs                        |
| `pg_replication_slots`          | Replication metadata                       |
| `pg_stat_progress_vacuum`       | No direct equivalent                       |
| `pg_stat_progress_create_index` | Index operation monitoring                 |
| `pg_stat_statements`            | Query Store                                |
| PostgreSQL logs                 | SQL Server error log                       |
| `log_min_duration_statement`    | Query performance diagnostics              |
| Prometheus/Grafana              | SQL monitoring tools                       |

---

# 32. SQL Server DBA Monitoring Translation

When you normally ask:

```sql
sp_who2
sp_WhoIsActive
sys.dm_exec_requests
sys.dm_exec_sessions
```

PostgreSQL's starting point is:

```sql
SELECT *
FROM pg_stat_activity;
```

Blocking:

```sql
SELECT *
FROM pg_locks;
```

Blocking PIDs:

```sql
SELECT pg_blocking_pids(pid)
FROM pg_stat_activity;
```

Query history:

```text
pg_stat_statements
```

This is one of the first PostgreSQL areas I recommend mastering.

---

# 33. Performance Tuning

| SQL Server DBA Concept | PostgreSQL                                              |
| ---------------------- | ------------------------------------------------------- |
| CPU                    | CPU                                                     |
| Memory pressure        | Memory/cache pressure                                   |
| Buffer cache           | shared_buffers + OS cache                               |
| Query Store            | pg_stat_statements                                      |
| Execution plan         | EXPLAIN                                                 |
| Actual plan            | EXPLAIN ANALYZE                                         |
| Missing index DMVs     | No direct equivalent                                    |
| Statistics             | PostgreSQL statistics                                   |
| Parameter sniffing     | Different prepared-plan/generic-vs-custom plan behavior |
| Blocking               | Blocking                                                |
| Deadlocks              | Deadlocks                                               |
| Wait statistics        | No direct `sys.dm_os_wait_stats` equivalent             |
| Query hints            | PostgreSQL planner controls/hints are different         |
| Plan guides            | No direct equivalent                                    |
| Forced plans           | Different tooling/approach                              |
| MAXDOP                 | No direct equivalent                                    |
| Cost Threshold         | No direct equivalent                                    |

---

# 34. Wait Statistics

This is another major difference.

SQL Server:

```sql
sys.dm_os_wait_stats
```

Common:

```text
PAGEIOLATCH
WRITELOG
CXPACKET
CXCONSUMER
LCK_M_*
RESOURCE_SEMAPHORE
SOS_SCHEDULER_YIELD
```

PostgreSQL doesn't have a one-to-one wait-statistics architecture.

Instead you commonly examine:

```text
pg_stat_activity
wait_event_type
wait_event
pg_stat_statements
pg_locks
I/O statistics
OS metrics
```

Example:

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

---

# 35. SQL Server TempDB vs PostgreSQL Temporary Storage

| SQL Server           | PostgreSQL                                        |
| -------------------- | ------------------------------------------------- |
| TempDB               | Temporary relations/files                         |
| `#temp`              | Temporary table                                   |
| `##temp`             | No direct equivalent                              |
| Table variables      | No direct equivalent                              |
| TempDB contention    | Different PostgreSQL temporary I/O considerations |
| TempDB version store | PostgreSQL MVCC/WAL/storage architecture          |
| TempDB configuration | PostgreSQL temporary file configuration           |

---

# 36. Encryption

| PostgreSQL                                 | SQL Server                                  |
| ------------------------------------------ | ------------------------------------------- |
| TLS/SSL                                    | TLS                                         |
| Transparent disk encryption via OS/storage | TDE                                         |
| pgcrypto                                   | SQL Server cryptographic functions          |
| Column/application encryption              | Always Encrypted / encryption functions     |
| KMS integration                            | TDE key hierarchy / external key management |
| Certificates                               | Certificates                                |
| Client certificate authentication          | Certificate-based authentication concepts   |

### Important

PostgreSQL does **not** have a native feature that is simply equivalent to SQL Server TDE in the same architecture.

Encryption-at-rest is commonly provided through:

```text
Cloud provider encryption
Disk/storage encryption
Filesystem encryption
PostgreSQL/client-level encryption
```

---

# 37. Auditing

| PostgreSQL           | SQL Server                   |
| -------------------- | ---------------------------- |
| PostgreSQL logging   | SQL Server Audit / Error Log |
| `pgaudit`            | SQL Server Audit             |
| `log_statement`      | Extended Events/Audit        |
| `log_connections`    | Login auditing               |
| `log_disconnections` | Session auditing             |
| Native logs          | SQL Server logs              |
| External SIEM        | External SIEM                |

---

# 38. Database Mail / Notifications

| SQL Server             | PostgreSQL                       |
| ---------------------- | -------------------------------- |
| Database Mail          | No native exact equivalent       |
| SQL Agent alerts       | External scheduler/monitoring    |
| Operator notifications | Monitoring platform              |
| `sp_send_dbmail`       | External SMTP/application/script |

---

# 39. SSIS Comparison

There is no direct PostgreSQL equivalent to SQL Server Integration Services.

| SQL Server       | PostgreSQL Ecosystem    |
| ---------------- | ----------------------- |
| SSIS             | ETL tools               |
| Data Flow Task   | ETL pipeline            |
| Control Flow     | Workflow/orchestration  |
| SQL Agent        | Scheduler               |
| SSIS Catalog     | No direct equivalent    |
| SSISDB           | No direct equivalent    |
| SSIS package     | ETL workflow            |
| Script Task      | Python/Bash/custom code |
| Execute SQL Task | SQL execution step      |

Common PostgreSQL ecosystem choices include:

```text
Apache Airflow
Apache NiFi
Talend
dbt
Python
Pentaho
Cloud ETL services
```

---

# 40. SSRS Comparison

PostgreSQL has no native SSRS equivalent.

| SQL Server       | PostgreSQL            |
| ---------------- | --------------------- |
| SSRS             | BI/reporting platform |
| Report Server    | BI server             |
| Report           | Dashboard/report      |
| Dataset          | SQL query/data source |
| Paginated Report | Reporting tool        |
| Subscription     | BI/report scheduler   |

---

# 41. SQL Agent vs PostgreSQL Automation

```text
SQL Server
     |
SQL Server Agent
     |
+----+-----+
|          |
Jobs     Alerts
```

PostgreSQL:

```text
PostgreSQL
     |
+----+----------------+
|         |          |
pg_cron   pgAgent    cron
                      |
                   Scripts
```

---

# 42. DBCC Commands

| SQL Server              | PostgreSQL           |
| ----------------------- | -------------------- |
| `DBCC CHECKDB`          | No exact equivalent  |
| `DBCC CHECKTABLE`       | No direct equivalent |
| `DBCC CHECKALLOC`       | No direct equivalent |
| `DBCC SHRINKDATABASE`   | No direct equivalent |
| `DBCC SHRINKFILE`       | No direct equivalent |
| `DBCC FREEPROCCACHE`    | No direct equivalent |
| `DBCC DROPCLEANBUFFERS` | No direct equivalent |

PostgreSQL relies more heavily on:

```text
VACUUM
ANALYZE
REINDEX
amcheck
system/catalog consistency checks
filesystem/storage checks
```

For index corruption checking, PostgreSQL's `amcheck` extension is particularly relevant.

---

# 43. DBCC CHECKDB Equivalent

There is **no single PostgreSQL command equivalent to `DBCC CHECKDB`**.

A SQL Server DBA should think approximately:

```text
DBCC CHECKDB
      ↓
PostgreSQL
      ├── amcheck
      ├── VACUUM
      ├── ANALYZE
      ├── catalog checks
      ├── filesystem/storage checks
      └── backup/restore validation
```

---

# 44. Database Maintenance

| SQL Server                  | PostgreSQL                                   |
| --------------------------- | -------------------------------------------- |
| Index rebuild               | REINDEX                                      |
| Index reorganize            | No exact equivalent                          |
| Update statistics           | ANALYZE                                      |
| DBCC CHECKDB                | amcheck + other checks                       |
| Backup                      | pg_basebackup/pg_dump/etc.                   |
| Shrink database             | Generally not routine PostgreSQL maintenance |
| Cleanup old data            | DELETE/partition management                  |
| Transaction log maintenance | WAL/archive management                       |
| Agent maintenance plan      | pg_cron/pgAgent/scripts                      |

---

# 45. Connection Pooling

| PostgreSQL                                | SQL Server                                         |
| ----------------------------------------- | -------------------------------------------------- |
| PgBouncer                                 | Connection pooling at application/middleware layer |
| Session pooling                           | Connection pooling                                 |
| Transaction pooling                       | No direct SQL Server equivalent                    |
| Statement pooling                         | No direct SQL Server equivalent                    |
| PostgreSQL backend process per connection | SQL Server worker/session architecture             |

**PgBouncer is extremely important in PostgreSQL environments with high connection counts.**

---

# 46. Migration

| PostgreSQL                       | SQL Server                                    |
| -------------------------------- | --------------------------------------------- |
| `pg_dump`                        | BACPAC/script/backup-based migration concepts |
| `pg_restore`                     | Restore/import                                |
| Logical replication              | Replication-based migration                   |
| AWS DMS                          | AWS DMS                                       |
| Azure Database Migration Service | Azure migration tooling                       |
| ora2pg                           | Oracle migration tool                         |
| Foreign Data Wrapper             | Linked-server-like integration concept        |
| `postgres_fdw`                   | Linked Server                                 |
| `dblink`                         | Linked-server-like remote query               |

---

# 47. Linked Server Equivalent

SQL Server:

```text
Linked Server
      ↓
SELECT *
FROM Server.Database.Schema.Table
```

PostgreSQL:

```text
postgres_fdw
```

Conceptually:

```text
PostgreSQL
     |
postgres_fdw
     |
Remote PostgreSQL
```

For heterogeneous access, PostgreSQL FDWs can also connect to other systems through appropriate extensions/drivers.

---

# 48. SQL Server Linked Server vs FDW

| Feature                 | PostgreSQL                 | SQL Server              |
| ----------------------- | -------------------------- | ----------------------- |
| Remote PostgreSQL       | postgres_fdw               | Linked Server           |
| Remote SQL Server       | tds_fdw / other solutions  | Linked Server           |
| Remote Oracle           | oracle_fdw                 | Linked Server           |
| Remote file/data source | FDWs/extensions            | Linked Server/providers |
| Pushdown                | FDW optimizer capabilities | Provider-dependent      |

---

# 49. Database Links

| Oracle  | PostgreSQL     | SQL Server    |
| ------- | -------------- | ------------- |
| DB Link | `dblink` / FDW | Linked Server |

For PostgreSQL administration, prefer **FDW** for sustained integration where practical rather than treating `dblink` as the universal solution.

---

# 50. Replication Slots

This is a PostgreSQL concept SQL Server DBAs need to learn carefully.

```text
Primary
   |
Replication Slot
   |
Standby / Logical Subscriber
```

Purpose:

**Prevent PostgreSQL from recycling WAL that a consumer still needs.**

SQL Server does not have a direct equivalent.

Operational risk:

```text
Replication slot inactive
       ↓
WAL retained
       ↓
pg_wal grows
       ↓
Disk exhaustion
```

---

# 51. LSN

| PostgreSQL                      | SQL Server                    |
| ------------------------------- | ----------------------------- |
| LSN                             | LSN                           |
| WAL LSN                         | Transaction log LSN           |
| `pg_current_wal_lsn()`          | Current log position concepts |
| `pg_wal_lsn_diff()`             | LSN difference concepts       |
| Replication lag measured by LSN | AG/log shipping lag metrics   |

This is one of the easiest areas for an experienced SQL Server DBA to understand.

---

# 52. Replication Lag

PostgreSQL:

```text
Primary WAL position
        ↓
Standby receive LSN
        ↓
Standby replay LSN
```

SQL Server:

```text
Primary Log
      ↓
Secondary received
      ↓
Secondary hardened
      ↓
Secondary redone
```

The terminology differs, but the operational question is similar:

> **How far behind is the secondary?**

---

# 53. High Availability Managers

| PostgreSQL       | SQL Server                                    |
| ---------------- | --------------------------------------------- |
| Patroni          | AG/WSFC orchestration concepts                |
| EFM              | HA manager                                    |
| repmgr           | Replication/cluster management                |
| pg_auto_failover | Automated failover management                 |
| etcd/Consul      | Distributed consensus/configuration component |
| HAProxy          | Listener/load-balancing role                  |

### PostgreSQL common production pattern

```text
              Application
                   |
                HAProxy
                   |
          +--------+--------+
          |                 |
       Patroni            Patroni
          |                 |
       Primary            Standby
          \                 /
             etcd
```

---

# 54. Patroni vs SQL Server AG

| Patroni/PostgreSQL          | SQL Server AG                                   |
| --------------------------- | ----------------------------------------------- |
| PostgreSQL replication      | SQL Server AG replication                       |
| Patroni manages failover    | AG manages failover                             |
| Distributed consensus store | WSFC provides quorum for traditional Windows AG |
| PostgreSQL primary          | AG primary                                      |
| PostgreSQL replica          | AG secondary                                    |
| HAProxy                     | AG Listener/load balancer                       |
| DCS                         | WSFC/quorum architecture                        |
| PostgreSQL service          | SQL Server service                              |

---

# 55. Cloud Mapping

## Azure

| PostgreSQL                                    | SQL Server                                   |
| --------------------------------------------- | -------------------------------------------- |
| Azure Database for PostgreSQL                 | Azure SQL Database                           |
| Azure Database for PostgreSQL Flexible Server | Azure SQL Managed Instance                   |
| PostgreSQL on Azure VM                        | SQL Server on Azure VM                       |
| PostgreSQL HA                                 | Azure PostgreSQL HA                          |
| PostgreSQL backups                            | Azure SQL backups                            |
| PostgreSQL read replicas                      | SQL read scale/replicas depending on service |

## AWS

| PostgreSQL                | SQL Server                                             |
| ------------------------- | ------------------------------------------------------ |
| Amazon RDS for PostgreSQL | Amazon RDS for SQL Server                              |
| Aurora PostgreSQL         | No exact SQL Server equivalent                         |
| PostgreSQL on EC2         | SQL Server on EC2                                      |
| RDS Multi-AZ              | SQL Server HA architecture                             |
| Read Replica              | SQL Server read/replica concepts depending on platform |

---

# 56. System Databases

| PostgreSQL           | SQL Server                  |
| -------------------- | --------------------------- |
| `postgres` database  | No exact equivalent         |
| `template0`          | Model-like template concept |
| `template1`          | Model-like template concept |
| `pg_catalog`         | System catalog              |
| `information_schema` | `INFORMATION_SCHEMA`        |
| `pg_toast`           | No equivalent               |

### SQL Server:

```text
master
model
msdb
tempdb
```

### PostgreSQL:

```text
template0
template1
postgres
```

But do **not** map them one-for-one.

---

# 57. Metadata

| PostgreSQL           | SQL Server                        |
| -------------------- | --------------------------------- |
| `pg_catalog`         | `sys` catalog                     |
| `information_schema` | `INFORMATION_SCHEMA`              |
| `pg_class`           | Object metadata concepts          |
| `pg_namespace`       | Schemas                           |
| `pg_database`        | `sys.databases`                   |
| `pg_roles`           | `sys.server_principals`           |
| `pg_tables`          | `sys.tables`                      |
| `pg_indexes`         | `sys.indexes`                     |
| `pg_attribute`       | `sys.columns`                     |
| `pg_constraint`      | `sys.objects`/constraint metadata |
| `pg_stat_activity`   | DMVs                              |

---

# 58. Common DBA Commands

| Task             | PostgreSQL                         | SQL Server                    |
| ---------------- | ---------------------------------- | ----------------------------- |
| List databases   | `\l`                               | `sp_helpdb` / `sys.databases` |
| Connect database | `\c dbname`                        | `USE dbname`                  |
| List tables      | `\dt`                              | `sp_tables` / `sys.tables`    |
| Describe table   | `\d table`                         | `sp_help table`               |
| List users       | `\du`                              | `sys.server_principals`       |
| Current database | `SELECT current_database();`       | `DB_NAME()`                   |
| Current user     | `SELECT current_user;`             | `SUSER_SNAME()`               |
| Server version   | `SELECT version();`                | `SELECT @@VERSION`            |
| Kill session     | `SELECT pg_terminate_backend(pid)` | `KILL session_id`             |
| Current activity | `pg_stat_activity`                 | `sys.dm_exec_requests`        |
| Locks            | `pg_locks`                         | `sys.dm_tran_locks`           |

---

# 59. PostgreSQL `psql` vs SQLCMD

| PostgreSQL | SQL Server         |
| ---------- | ------------------ |
| `psql`     | `sqlcmd`           |
| `\l`       | Database listing   |
| `\c db`    | `USE db`           |
| `\dt`      | Table listing      |
| `\d table` | `sp_help`          |
| `\du`      | Login/role listing |
| `\dn`      | Schema listing     |
| `\df`      | Function listing   |
| `\q`       | Exit               |
| `\timing`  | Query timing       |
| `\x`       | Expanded output    |

---

# 60. Error Handling

| PostgreSQL                    | SQL Server                            |
| ----------------------------- | ------------------------------------- |
| `BEGIN ... EXCEPTION ... END` | `BEGIN TRY ... END TRY / BEGIN CATCH` |
| `RAISE EXCEPTION`             | `THROW`                               |
| `RAISE NOTICE`                | `PRINT`                               |
| SQLSTATE                      | Error number/state concepts           |
| `GET STACKED DIAGNOSTICS`     | `ERROR_*()` functions                 |

---

# 61. Dynamic SQL

| PostgreSQL            | SQL Server                        |
| --------------------- | --------------------------------- |
| `EXECUTE` in PL/pgSQL | `EXEC()`                          |
| `format()`            | `QUOTENAME()`/string construction |
| `EXECUTE ... USING`   | `sp_executesql` parameters        |

For safe dynamic SQL, parameterization remains important in both platforms.

---

# 62. Temporary Objects

| PostgreSQL          | SQL Server                                           |
| ------------------- | ---------------------------------------------------- |
| `CREATE TEMP TABLE` | `CREATE TABLE #temp`                                 |
| Temporary table     | Local temp table                                     |
| `ON COMMIT DROP`    | No direct equivalent                                 |
| Temporary schema    | TempDB                                               |
| Temporary files     | TempDB-like temporary work, but architecture differs |

---

# 63. Identity / Sequence

| PostgreSQL                | SQL Server                      |
| ------------------------- | ------------------------------- |
| `SEQUENCE`                | `SEQUENCE`                      |
| `nextval()`               | `NEXT VALUE FOR`                |
| `currval()`               | No direct exact equivalent      |
| Identity                  | IDENTITY                        |
| Serial                    | Legacy shorthand using sequence |
| Identity generated always | IDENTITY                        |

---

# 64. Materialized Views

| PostgreSQL                  | SQL Server                                        |
| --------------------------- | ------------------------------------------------- |
| Materialized View           | Indexed View                                      |
| `REFRESH MATERIALIZED VIEW` | Indexed view maintained automatically             |
| Manual refresh              | Automatic maintenance                             |
| Can become stale            | Indexed view generally maintained transactionally |

This is a significant architectural difference.

---

# 65. JSON

| PostgreSQL         | SQL Server                              |
| ------------------ | --------------------------------------- |
| `json`             | JSON stored in text                     |
| `jsonb`            | JSON stored in text with JSON functions |
| `jsonb_path_query` | JSON path functions                     |
| `->`               | `JSON_VALUE` / `JSON_QUERY` concepts    |
| `->>`              | `JSON_VALUE`                            |
| GIN JSONB index    | No direct equivalent                    |
| JSON containment   | JSON querying                           |

PostgreSQL's `jsonb` is substantially more database-native than SQL Server's JSON implementation.

---

# 66. Full Text Search

| PostgreSQL      | SQL Server                     |
| --------------- | ------------------------------ |
| `tsvector`      | Full-Text Search               |
| `tsquery`       | Full-Text query                |
| GIN/GiST        | Full-text indexes              |
| `to_tsvector()` | Full-text indexing concepts    |
| `to_tsquery()`  | `CONTAINS`/`FREETEXT` concepts |

---

# 67. Regular Expressions

| PostgreSQL                       | SQL Server                                   |
| -------------------------------- | -------------------------------------------- |
| Native regex operators/functions | Historically limited native regex capability |
| `~`                              | No direct equivalent                         |
| `~*`                             | No direct equivalent                         |
| `regexp_replace()`               | Complex T-SQL/custom CLR approaches          |
| `regexp_matches()`               | No direct equivalent                         |

---

# 68. Extensions

PostgreSQL:

```sql
CREATE EXTENSION pg_stat_statements;
```

Common extensions:

```text
pg_stat_statements
pgcrypto
pgaudit
PostGIS
pg_trgm
uuid-ossp
```

SQL Server:

```text
Built-in features
CLR
modules
external components
```

**PostgreSQL extensions are a major architectural concept to learn.**

---

# 69. Foreign Data Wrapper

PostgreSQL:

```text
Local PostgreSQL
       |
       +-- postgres_fdw
       |
Remote Database
```

SQL Server:

```text
SQL Server
     |
Linked Server
     |
Remote Database
```

Conceptually:

**FDW ≈ Linked Server**

but the architecture and capabilities differ.

---

# 70. Connection / Session Terminology

| PostgreSQL             | SQL Server                    |
| ---------------------- | ----------------------------- |
| Backend PID            | SPID/session                  |
| Session                | Session                       |
| Connection             | Connection                    |
| Backend process        | Worker/backend                |
| `pg_stat_activity.pid` | `session_id`                  |
| `client_addr`          | Client/network information    |
| `application_name`     | Application name/program name |
| `state`                | Request/session state         |

---

# 71. PostgreSQL Process Architecture

Typical PostgreSQL architecture:

```text
                 PostgreSQL
                     |
          +----------+----------+
          |          |          |
      Postmaster  Checkpointer  WAL Writer
          |
    +-----+-----+-----+
    |           |     |
 Backend     Backend Backend
   PID         PID     PID
```

SQL Server:

```text
                 SQL Server
                     |
              Database Engine
                     |
          +----------+----------+
          |          |          |
       Scheduler  Workers    Memory
          |
       Sessions
          |
       Requests
```

The biggest mental model:

**PostgreSQL = process-oriented architecture**

**SQL Server = worker/scheduler-oriented database-engine architecture**

---

# 72. Memory Architecture

| PostgreSQL           | SQL Server                 |
| -------------------- | -------------------------- |
| shared_buffers       | Buffer Pool                |
| work_mem             | Query workspace memory     |
| maintenance_work_mem | Maintenance memory         |
| temp_buffers         | Temporary table buffers    |
| effective_cache_size | Optimizer cache assumption |
| OS page cache        | OS cache                   |
| Per-operation memory | Memory grant concepts      |

---

# 73. CPU & Parallelism

| PostgreSQL                        | SQL Server                            |
| --------------------------------- | ------------------------------------- |
| `max_worker_processes`            | Worker/thread architecture            |
| `max_parallel_workers`            | MAXDOP/parallel worker concepts       |
| `max_parallel_workers_per_gather` | Parallelism per query                 |
| `min_parallel_table_scan_size`    | Cost-based parallelism concept        |
| `parallel_setup_cost`             | Parallelism cost model                |
| `parallel_tuple_cost`             | Parallelism cost model                |
| MAXDOP                            | No direct single PostgreSQL parameter |
| Cost Threshold                    | No direct equivalent                  |

---

# 74. SQL Server Concepts Without Direct PostgreSQL Equivalents

| SQL Server Concept     | PostgreSQL Situation                           |
| ---------------------- | ---------------------------------------------- |
| SQL Server Agent       | pg_cron/pgAgent/external scheduler             |
| DBCC CHECKDB           | No single equivalent                           |
| TempDB                 | No direct equivalent                           |
| Always On AG Listener  | External HA/load-balancing layer commonly used |
| Query Store            | pg_stat_statements + other tooling             |
| Extended Events        | Logging/extensions/external observability      |
| Database Mail          | External notification system                   |
| SSIS                   | External ETL                                   |
| SSRS                   | External reporting                             |
| SSAS                   | External analytics/OLAP                        |
| Resource Governor      | No direct equivalent                           |
| Plan Guides            | No direct equivalent                           |
| TDE                    | No exact native equivalent                     |
| Filegroups             | Tablespaces are conceptually closest           |
| Differential backup    | No direct native equivalent                    |
| Transaction log backup | WAL archiving                                  |
| Indexed view           | Materialized view, but behavior differs        |
| Filtered index         | Partial index                                  |
| Included columns       | `INCLUDE`                                      |
| FCI                    | No direct PostgreSQL equivalent                |

---

# 75. PostgreSQL Concepts SQL Server DBAs Must Learn Separately

These should **not** simply be translated into SQL Server terminology:

| PostgreSQL Concept   | Why Important                            |
| -------------------- | ---------------------------------------- |
| MVCC internals       | Fundamental PostgreSQL concurrency model |
| VACUUM               | Critical for database health             |
| Autovacuum           | Critical production subsystem            |
| XID                  | Transaction ID management                |
| XID wraparound       | Major PostgreSQL operational risk        |
| Freeze               | Prevents transaction ID wraparound       |
| WAL                  | Foundation of recovery/replication       |
| Replication slots    | Can cause WAL accumulation               |
| Timeline             | Important for recovery/replication       |
| `pg_hba.conf`        | PostgreSQL authentication architecture   |
| `search_path`        | Object resolution/security               |
| TOAST                | Large-value storage                      |
| GIN                  | Powerful PostgreSQL index type           |
| GiST                 | Specialized index framework              |
| BRIN                 | Large-table/physical-order optimization  |
| Logical decoding     | Change-data extraction                   |
| FDW                  | Remote data integration                  |
| Extensions           | Major PostgreSQL extensibility model     |
| Patroni              | Common HA management architecture        |
| PgBouncer            | Critical connection-pooling technology   |
| `pg_stat_statements` | Core performance-analysis tool           |

---

# 76. DBA Daily Operations Mapping

| Daily DBA Task      | PostgreSQL                       | SQL Server                    |
| ------------------- | -------------------------------- | ----------------------------- |
| Check server        | `systemctl`, PostgreSQL tools    | SQL Server service            |
| Check connections   | `pg_stat_activity`               | DMVs                          |
| Check blocking      | `pg_locks`, `pg_blocking_pids()` | `sys.dm_tran_locks`, DMVs     |
| Check long queries  | `pg_stat_activity`               | DMVs/Query Store              |
| Query history       | `pg_stat_statements`             | Query Store                   |
| Check database size | `pg_database_size()`             | `sys.database_files`/DMVs     |
| Check table size    | `pg_total_relation_size()`       | `sp_spaceused`                |
| Check indexes       | `pg_indexes`                     | `sys.indexes`                 |
| Check index usage   | `pg_stat_user_indexes`           | `sys.dm_db_index_usage_stats` |
| Check statistics    | `pg_stats`                       | `sys.stats`                   |
| Update stats        | `ANALYZE`                        | `UPDATE STATISTICS`           |
| Clean dead tuples   | `VACUUM`                         | No direct equivalent          |
| Rebuild index       | `REINDEX`                        | `ALTER INDEX REBUILD`         |
| Backup              | `pg_basebackup`/`pg_dump`        | BACKUP DATABASE               |
| Restore             | `pg_restore`/recovery            | RESTORE DATABASE              |
| PITR                | WAL + base backup                | Full + log backups            |
| HA status           | `pg_stat_replication`/Patroni    | AG DMVs                       |
| Kill session        | `pg_terminate_backend()`         | `KILL`                        |
| Schedule job        | pg_cron/pgAgent                  | SQL Agent                     |

---

# 77. DBA Troubleshooting Translation

| Problem               | PostgreSQL First Check                         | SQL Server First Check            |
| --------------------- | ---------------------------------------------- | --------------------------------- |
| High CPU              | `pg_stat_activity`, pg_stat_statements, OS CPU | DMVs/Query Store                  |
| Blocking              | `pg_locks`, `pg_stat_activity`                 | DMVs                              |
| Deadlock              | PostgreSQL logs                                | Extended Events/error log         |
| Slow query            | `EXPLAIN ANALYZE`, pg_stat_statements          | Actual execution plan/Query Store |
| Table bloat           | `pg_stat_user_tables`, VACUUM stats            | Fragmentation/space               |
| High WAL              | `pg_stat_wal`, replication slots               | Transaction log                   |
| Replica lag           | `pg_stat_replication`                          | AG DMVs                           |
| Disk full             | WAL/data/temp                                  | Data/log/tempdb                   |
| Connection exhaustion | `pg_stat_activity`, max_connections, PgBouncer | Connection/session DMVs           |
| Corruption            | `amcheck`, logs, storage                       | DBCC CHECKDB                      |
| Statistics problem    | ANALYZE/statistics                             | UPDATE STATISTICS                 |
| Index problem         | pg_stat_user_indexes/EXPLAIN                   | DMVs/plan                         |
| Authentication        | `pg_hba.conf`, PostgreSQL logs                 | Logins/permissions/error log      |
| Recovery failure      | WAL/archive/recovery config                    | Backup chain/log restore          |

---

# 78. Backup Strategy Translation

### SQL Server

```text
FULL
  ↓
DIFF
  ↓
LOG
  ↓
PITR
```

### PostgreSQL

```text
BASE BACKUP
     ↓
WAL ARCHIVE
     ↓
RECOVERY
     ↓
PITR
```

A very important difference:

**PostgreSQL does not have SQL Server's native Full → Differential → Transaction Log backup hierarchy.**

---

# 79. HA/DR Strategy Translation

### SQL Server

```text
                 AG Listener
                     |
               Primary Replica
                     |
          +----------+----------+
          |                     |
      Sync Secondary       Async Secondary
          |                     |
      HA/Automatic           DR
       Failover
```

### PostgreSQL

```text
                 Application
                      |
                 HAProxy/LB
                      |
             +--------+--------+
             |                 |
          Primary            Standby
             |                 |
        Streaming WAL
             |
        +----+----+
        |
      Patroni
        |
   etcd/Consul
```

---

# 80. Most Important Mental Translation Map

If you remember only one table, remember this:

| SQL Server DBA        | PostgreSQL DBA                             |
| --------------------- | ------------------------------------------ |
| Instance              | Cluster                                    |
| Database              | Database                                   |
| Schema                | Schema                                     |
| Login                 | Role                                       |
| User                  | Role/user                                  |
| SQL Agent             | pg_cron/pgAgent                            |
| SSMS                  | pgAdmin                                    |
| sqlcmd                | psql                                       |
| T-SQL                 | SQL + PL/pgSQL                             |
| Transaction Log       | WAL                                        |
| Log Backup            | WAL archiving                              |
| Full Backup           | Base backup                                |
| PITR                  | PITR                                       |
| Always On AG          | Streaming Replication + HA manager         |
| AG Secondary          | Standby                                    |
| AG Listener           | HAProxy/load balancer/DNS/service endpoint |
| WSFC                  | HA manager + DCS concepts                  |
| Query Store           | pg_stat_statements + tooling               |
| DMVs                  | pg_stat_* / pg_catalog                     |
| Buffer Pool           | shared_buffers + OS cache                  |
| TempDB                | Temporary relations/files                  |
| DBCC CHECKDB          | amcheck + other validation                 |
| UPDATE STATISTICS     | ANALYZE                                    |
| Index Rebuild         | REINDEX                                    |
| Fragmentation         | Bloat/dead tuples — **not identical**      |
| Filtered Index        | Partial Index                              |
| Indexed View          | Materialized View — **not identical**      |
| Linked Server         | FDW                                        |
| `sp_who2`             | `pg_stat_activity`                         |
| `KILL`                | `pg_terminate_backend()`                   |
| `sp_WhoIsActive`      | `pg_stat_activity` + custom queries        |
| SQL Server Audit      | pgaudit                                    |
| TDE                   | Storage/filesystem/cloud encryption        |
| SSIS                  | External ETL                               |
| SSRS                  | External reporting                         |
| Database Mail         | External SMTP/monitoring                   |
| Filegroup             | Tablespace — **not identical**             |
| Partition Elimination | Partition Pruning                          |
| MAXDOP                | PostgreSQL parallel-worker settings        |
| CXPACKET/CXCONSUMER   | No direct PostgreSQL equivalent            |
| Wait Statistics       | `wait_event` + activity/statistics         |
| Extended Events       | Logging/observability ecosystem            |
| Replication Slot      | **No direct equivalent**                   |
| Autovacuum            | **No SQL Server equivalent**               |
| XID Freeze            | **No SQL Server equivalent**               |
| TOAST                 | **No SQL Server equivalent**               |

---

# 81. Recommended Learning Sequence for a SQL Server DBA

Given your SQL Server background, I would learn PostgreSQL in this order:

| Phase  | PostgreSQL Topics       | SQL Server Anchor              |
| ------ | ----------------------- | ------------------------------ |
| **1**  | Architecture            | SQL Server Engine architecture |
| **2**  | Installation / services | SQL Server installation        |
| **3**  | `postgresql.conf`       | Server configuration           |
| **4**  | `pg_hba.conf`           | SQL authentication/security    |
| **5**  | Roles/privileges        | Logins/users/roles             |
| **6**  | Database/schema/storage | Database/files/filegroups      |
| **7**  | Tables/data types       | Tables/data types              |
| **8**  | Indexes                 | B-tree/indexes                 |
| **9**  | EXPLAIN                 | Execution plans                |
| **10** | Statistics/ANALYZE      | Statistics                     |
| **11** | MVCC                    | SQL Server row-versioning      |
| **12** | Locks/blocking          | Lock manager                   |
| **13** | VACUUM/autovacuum       | Maintenance/fragmentation      |
| **14** | WAL                     | Transaction log                |
| **15** | `pg_dump`               | Logical backup                 |
| **16** | `pg_basebackup`         | Full/physical backup           |
| **17** | WAL archiving           | Transaction log backups        |
| **18** | PITR                    | SQL Server STOPAT              |
| **19** | Streaming replication   | Always On secondary            |
| **20** | Replication slots       | New PostgreSQL concept         |
| **21** | Patroni                 | AG failover/orchestration      |
| **22** | PgBouncer               | Connection management          |
| **23** | pg_stat_statements      | Query Store                    |
| **24** | Performance tuning      | SQL Server performance tuning  |
| **25** | Partitioning            | SQL Server partitioning        |
| **26** | Logical replication     | SQL Server replication         |
| **27** | FDW                     | Linked Server                  |
| **28** | pg_cron/pgAgent         | SQL Agent                      |
| **29** | pgaudit                 | SQL Audit                      |
| **30** | PostgreSQL HA/DR        | AG/FCI/Log Shipping            |
| **31** | Cloud PostgreSQL        | RDS/Azure PostgreSQL           |
| **32** | PostgreSQL upgrades     | SQL Server version upgrades    |

Concentrate on the areas where PostgreSQL's architecture genuinely differs:

1. **PostgreSQL cluster architecture**
2. **MVCC**
3. **VACUUM + Autovacuum + bloat**
4. **WAL architecture**
5. **PITR and WAL archiving**
6. **Streaming replication**
7. **Replication slots**
8. **Patroni + DCS + failover**
9. **`pg_stat_activity` + `pg_stat_statements` + PostgreSQL performance diagnostics**
10. **`pg_hba.conf` + roles + PostgreSQL security model**

That combination will let you translate most PostgreSQL DBA scenarios back into your existing **SQL Server DBA mental model**, while still understanding where PostgreSQL behaves fundamentally differently.

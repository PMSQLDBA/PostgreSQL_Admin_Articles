# **PostgreSQL 19 Released: Complete Technical Deep Dive, New Features, Real-World DBA Scenarios, and Production Upgrade Guide**

**Technical focus:** Database Administration | Performance Tuning | Storage Management | Logical Replication | HA/DR | Security | Migration

## **1. Introduction**

The PostgreSQL Global Development Group officially released **PostgreSQL 19 on November 19, 2026**.

This is now a confirmed General Availability (GA) release, not a beta or release candidate.

PostgreSQL 19 introduces several important improvements for organizations operating large, business-critical database environments.

The major themes are:

- Online table reorganization using `REPACK (CONCURRENTLY)`.
- Query-plan stabilization through `pg_plan_advice`.
- Automatic application of query-plan advice using `pg_stash_advice`.
- Parallel autovacuum and intelligent maintenance prioritization.
- Automatic scaling of asynchronous I/O workers.
- Logical replication improvements, including sequence synchronization.
- Read-your-writes consistency using `WAIT FOR LSN`.
- Better query execution and foreign-key performance.
- Improved monitoring and observability.
- New SQL capabilities for application developers.
- Enhanced TLS configuration through Server Name Indication (SNI).

One particularly important point: PostgreSQL 19 is not simply introducing additional SQL commands. Several enhancements address operational problems that DBAs encounter when managing large databases, high-transaction workloads, replication environments and performance-sensitive applications.

The following article uses the official PostgreSQL 19 release announcement and PostgreSQL 19 documentation as its primary technical references.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.postgresql.org\&sz=32)

## **2. PostgreSQL 19 release information**

| Item                   | Details                                       |
| ---------------------- | --------------------------------------------- |
| Major version          | PostgreSQL 19                                 |
| Initial release        | 19.0                                          |
| Release date           | November 19, 2026                             |
| Release status         | General Availability                          |
| Previous major version | PostgreSQL 18                                 |
| Upgrade methods        | pg_upgrade, dump/restore, logical replication |
| Official documentation | PostgreSQL 19 manual                          |

The official release announcement confirms PostgreSQL 19.0 availability on November 19, 2026.&#x20;

### **A note about AI-assisted bug discovery**

The earlier LinkedIn post attributes the release delay to AI-assisted bug discovery.

AI-assisted analysis can help developers discover defects, but identifying, reproducing, fixing and validating a database-engine defect still requires engineering review and regression testing.

The official PostgreSQL 19 release announcement confirms the GA date and delivered features. It does not establish AI-assisted bug discovery as the sole official reason for the release timing. Therefore, that explanation should not be presented as an established fact.

## **3. PostgreSQL 19 feature summary**

| Feature                            | Technical significance                                   |
| ---------------------------------- | -------------------------------------------------------- |
| `REPACK`                           | Rewrites tables and reclaims unused physical storage     |
| `REPACK (CONCURRENTLY)`            | Reorganization without blocking normal reads and writes  |
| `pg_plan_advice`                   | Controls selected query-planner decisions                |
| `pg_stash_advice`                  | Stores and automatically applies plan advice             |
| Parallel autovacuum                | Parallel index vacuuming                                 |
| Autovacuum scoring                 | Prioritizes tables for maintenance                       |
| `WAIT FOR LSN`                     | Read-your-writes consistency on standbys                 |
| Sequence replication               | Synchronizes sequence values through logical replication |
| Automatic I/O workers              | Dynamically adjusts worker count                         |
| `INSERT ... ON CONFLICT DO SELECT` | Simplifies get-or-create application patterns            |
| SNI                                | Allows hostname-based TLS certificate selection          |
| JSON COPY                          | Supports JSON output through COPY                        |

Let's examine the features from a production DBA perspective.

## **4. REPACK: A major improvement in table-bloat management**

High-priority DBA feature

### **4.1 Understanding PostgreSQL table bloat**

PostgreSQL uses MVCC (Multi-Version Concurrency Control) to maintain transaction consistency.

When an application executes UPDATE or DELETE operations, PostgreSQL manages row versions rather than immediately overwriting or physically removing the old versions.

For example:

```
UPDATE customer_accounts
SET account_status = 'INACTIVE'
WHERE customer_id = 10001;
```

The previous row version becomes obsolete once it is no longer visible to relevant transactions.

VACUUM identifies dead tuples and makes their space available for reuse.

However, ordinary VACUUM does not generally shrink the physical table file and return that space to the operating system.

This can become a concern for large OLTP databases.

### **4.2 Traditional table-maintenance approaches**

Before PostgreSQL 19, DBAs commonly used:

| Command     | Purpose                                       | Operational consideration                        |
| ----------- | --------------------------------------------- | ------------------------------------------------ |
| VACUUM      | Reclaims dead-tuple space for reuse           | Does not normally shrink relation files          |
| VACUUM FULL | Rewrites table and returns unused space       | Requires an ACCESS EXCLUSIVE lock                |
| CLUSTER     | Physically reorganizes a table using an index | Requires significant locking                     |
| REINDEX     | Rebuilds indexes                              | Locking and availability depend on the operation |

PostgreSQL 19 introduces `REPACK` to unify the functionality previously associated with `VACUUM FULL` and `CLUSTER`.

The older commands remain available for compatibility.&#x20;


### **4.3 Understanding the new REPACK command**

The documented syntax is:

```
REPACK [ ( option [, ...] ) ]
       [ table_name [ USING INDEX index_name ] ];
```

Options include:

- `VERBOSE`
- `ANALYZE`
- `CONCURRENTLY`

Example:

```
REPACK customer_accounts;
```

Reorganize using an index:

```
REPACK customer_accounts
USING INDEX idx_customer_id;
```

Perform concurrent repacking:

```
REPACK (CONCURRENTLY)
customer_accounts;
```

Concurrent repacking allows other transactions to use the table during the operation.&#x20;


### **4.4 How REPACK works conceptually**

**REPACK execution lifecycle**

1. **Identify the target table**

   Evaluate the table, its indexes, privileges and maintenance requirements.
2. **Create the replacement storage**

   Rewrite table contents into a new physical relation.
3. **Rebuild associated indexes**

   Reconstruct the indexes for the replacement table.
4. **Handle concurrent modifications**

   The concurrent operation coordinates changes occurring while the replacement is being prepared.
5. **Complete the replacement**

   Perform the final catalog and relation transition.
6. **Validate and monitor**

   Verify table statistics, indexes, application performance and replica health.

The important difference is that the concurrent option avoids holding an ACCESS EXCLUSIVE lock throughout the entire operation. It does not mean the operation is completely lock-free.&#x20;


### **4.5 Real-world banking scenario**

Consider a banking application with:

| Parameter          | Value             |
| ------------------ | ----------------- |
| Database           | BankingDB         |
| Table              | customer_accounts |
| Records            | 50 million        |
| Table size         | 180 GB            |
| Index size         | 75 GB             |
| Workload           | 24/7 OLTP         |
| Maintenance window | Limited           |

The table experiences frequent UPDATE operations.

Over time, the DBA identifies significant dead-tuple accumulation and physical bloat.

The requirement is to reorganize the table without imposing a long period during which normal application queries cannot access it.

Implementation:

```
REPACK (CONCURRENTLY)
customer_accounts;
```

After completion:

```
ANALYZE customer_accounts;
```

Validate the results using:

- Relation size before and after.
- Index size before and after.
- Query execution plans.
- Application response times.
- Replication lag.
- Disk utilization.

### **4.6 Monitor REPACK progress**

PostgreSQL 19 introduces:

```
SELECT *
FROM pg_stat_progress_repack;
```

This view reports the progress of `REPACK`, `CLUSTER` and `VACUUM FULL`.

### **4.7 Production considerations**

Before executing REPACK:

1. Check available disk capacity.
2. Review active and long-running transactions.
3. Check replication-slot retention.
4. Evaluate WAL generation.
5. Review replica lag.
6. Check for invalid indexes.
7. Review concurrent DDL activity.
8. Define an acceptable maintenance window.

An important distinction: REPACK is not a replacement for VACUUM when preventing transaction ID wraparound. PostgreSQL explicitly recommends VACUUM for that purpose. Concurrent repacking also has limitations and can fail after substantial work has been performed.

**DBA recommendation:** Use regular VACUUM and autovacuum for routine maintenance. Use REPACK when physical space reclamation or table reorganization is justified.&#x20;


## **5. pg_plan_advice: Better control over query execution plans**

High-priority performance feature

### **5.1 The problem: Query-plan instability**

PostgreSQL uses a cost-based optimizer to select execution plans.

The optimizer considers:

- Table statistics.
- Available indexes.
- Estimated cardinality.
- Data distribution.
- Join methods.
- Available memory.
- Parallel execution possibilities.

Consider:

```
SELECT
    c.customer_id,
    c.customer_name,
    t.transaction_id,
    t.transaction_amount
FROM customers c
JOIN transactions t
    ON c.customer_id = t.customer_id
WHERE c.customer_id = 50001;
```

Initially, the query executes in 200 milliseconds.

After several months, the same query takes 12 seconds.

Potential causes:

- Data growth.
- Outdated statistics.
- Data skew.
- Changes in selectivity.
- Different join order.
- Different index or scan selection.

### **5.2 What is pg_plan_advice?**

PostgreSQL 19 introduces `pg_plan_advice`, which allows DBAs to describe, reproduce and influence query-planner decisions through a specialized plan-advice language.

The important distinction is that it does not replace the PostgreSQL optimizer.

It constrains selected planner choices.

### **5.3 Loading the extension**

The official documentation describes three approaches.

**Option 1: System-wide loading**

```
shared_preload_libraries = 'pg_plan_advice'
```

This requires a PostgreSQL restart.

**Option 2: Session-level loading**

```
session_preload_libraries = 'pg_plan_advice'
```

**Option 3: Individual session**

```
LOAD 'pg_plan_advice';
```

### **5.4 Generate execution-plan advice**

Example:

```
EXPLAIN (COSTS OFF, PLAN_ADVICE)
SELECT *
FROM transactions t
JOIN customers c
    ON t.customer_id = c.customer_id;
```

PostgreSQL generates a normal execution plan along with plan advice.

An illustrative advice structure is:

```
JOIN_ORDER(f d)
HASH_JOIN(d)
SEQ_SCAN(f d)
NO_GATHER(f d)
```

These expressions describe decisions such as:

- Join order.
- Join method.
- Scan method.
- Parallel execution.

You can then apply selected advice:

```
SET pg_plan_advice.advice =
    'JOIN_ORDER(f d)';
```

The official documentation provides these mechanisms and explains the generated advice and feedback.&#x20;


### **5.5 Real-world scenario**

A financial reporting database has:

- 200 million transaction records.
- 15 million customer records.
- Multiple reporting queries.
- Different query execution behaviour after data growth.

Start with:

```
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT *
FROM transactions t
JOIN customers c
    ON t.customer_id = c.customer_id
WHERE t.customer_id = 50001;
```

Investigate:

1. Estimated versus actual rows.
2. Join strategy.
3. Index usage.
4. Shared buffer reads and hits.
5. Temporary I/O.
6. Planning time.
7. Execution time.

Then generate and evaluate plan advice.

**Important:** Plan advice can backfire if the data distribution changes. Do not force a plan simply because it was faster yesterday.

## **6. pg_stash_advice: Automatically applying stored plan advice**

This is another important feature missing from the original screenshot's summary.

PostgreSQL 19 introduces the companion `pg_stash_advice` extension.

While `pg_plan_advice` helps generate and apply plan advice, `pg_stash_advice` stores advice and automatically applies it to queries using their query identifiers.

This is useful when an application cannot easily be modified to change its SQL statements.

### **Real-world scenario**

A payment application executes hundreds of SQL statements.

One query develops a performance regression after data growth.

The DBA identifies a better execution strategy and stores the corresponding advice.

The application continues executing its existing SQL.

The stored advice can be applied based on the query identifier.

This gives DBAs another mechanism for addressing plan regressions without changing application SQL.

**Operational caution:** Stored plan advice requires monitoring and lifecycle management. A plan that is appropriate today might become inefficient after major data-distribution changes.

Official reference: [PostgreSQL 19 pg_stash_advice documentation](https://www.postgresql.org/docs/19/pgstashadvice.html).&#x20;


## **7. Smarter autovacuum: Parallel index vacuuming and priority-based scheduling**

Major DBA enhancement

### **7.1 Why autovacuum is critical**

Autovacuum is one of the most important PostgreSQL background maintenance mechanisms.

It performs:

- Dead-tuple cleanup.
- Visibility-map maintenance.
- Transaction ID freezing.
- Multixact maintenance.
- Automatic statistics collection through ANALYZE.

If autovacuum cannot keep up with the workload, the database may experience:

- Table and index bloat.
- Increased disk consumption.
- Degraded query performance.
- Increased maintenance overhead.
- Transaction ID wraparound risk.

### **7.2 What's new in PostgreSQL 19?**

PostgreSQL 19 introduces two important enhancements:

1. Parallel autovacuum workers for table indexes.
2. A scoring mechanism to prioritize which tables should be processed first.

It also allows queries scanning tables to mark eligible pages as all-visible in the visibility map, reducing some of the work future vacuum operations need to perform.&#x20;


### **7.3 Parallel autovacuum configuration**

New configuration parameter:

```
autovacuum_max_parallel_workers = 4
```

Table-level configuration:

```
ALTER TABLE transactions
SET (autovacuum_parallel_workers = 4);
```

These values are illustrative.

The actual number of workers depends on global worker availability, table eligibility and the number of indexes requiring maintenance.

Parallelism is not guaranteed to use the configured maximum.

### **7.4 Real-world scenario**

Consider a SaaS production database:

| Table             | Size   | Workload            |
| ----------------- | ------ | ------------------- |
| transactions      | 500 GB | Heavy INSERT/UPDATE |
| audit_events      | 250 GB | Heavy INSERT        |
| customer_profile  | 100 GB | Frequent UPDATE     |
| reporting_summary | 80 GB  | Periodic refresh    |

Several tables require vacuuming at approximately the same time.

Parallel index vacuuming can improve maintenance throughput by distributing eligible index-maintenance work across additional workers.

### **7.5 Autovacuum scoring mechanism**

PostgreSQL 19 introduces a scoring system that helps determine which tables should receive maintenance attention first.

The configuration parameters include:

```
autovacuum_freeze_score_weight
autovacuum_multixact_freeze_score_weight
autovacuum_vacuum_score_weight
autovacuum_vacuum_insert_score_weight
autovacuum_analyze_score_weight
```

These weights influence the prioritization of:

- Transaction ID freezing.
- Multixact freezing.
- Dead-tuple vacuuming.
- Insert-related vacuuming.
- ANALYZE operations.

Monitor the scores:

```
SELECT *
FROM pg_stat_autovacuum_scores;
```

This gives DBAs greater visibility into the maintenance-prioritization process.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.postgresql.org\&sz=32)

PostgreSQL



### **7.6 Monitor active vacuum operations**

```
SELECT
    pid,
    datname,
    relid::regclass AS table_name,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    heap_blks_vacuumed,
    index_vacuum_count
FROM pg_stat_progress_vacuum;
```

Review table maintenance statistics:

```
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
ORDER BY n_dead_tup DESC;
```

### **7.7 Production tuning considerations**

Before increasing parallel vacuum workers, evaluate:

- CPU utilization.
- Storage throughput.
- `max_worker_processes`.
- `max_parallel_workers`.
- Available autovacuum workers.
- `maintenance_work_mem`.
- Vacuum cost settings.
- Replication lag.
- Concurrent application workload.

**DBA recommendation:** Never optimize autovacuum based only on the number of workers. Monitor transaction age, I/O latency, dead tuples, maintenance duration and workload impact.

## **8. Logical replication improvements**

Logical replication is widely used for:

- Database migrations.
- Reporting databases.
- Cross-environment synchronization.
- Data distribution.
- Application modernization.
- Selective table replication.

PostgreSQL 19 makes several improvements to logical replication initialization and sequence synchronization.

### **8.1 Enable logical replication without a restart**

Traditionally, DBAs configured:

```
wal_level = logical
```

Changing `wal_level` required a server restart.

PostgreSQL 19 introduces automatic logical-replication enablement when the configured `wal_level` is `replica` and logical replication is needed.

The new setting:

```
SHOW effective_wal_level;
```

reports the effective WAL level.

This is an important distinction: PostgreSQL 19 does not simply make every logical replication configuration restart-free under all circumstances. The documented behaviour depends on the existing WAL configuration and logical-slot creation.&#x20;


### **8.2 Real-world migration scenario**

**Source PostgreSQL 19**

Production OLTP database

Logical replication

**Target PostgreSQL 19**

Migration / reporting environment

The DBA needs to introduce logical replication without scheduling a traditional restart solely to change `wal_level` from `replica`.

PostgreSQL 19 simplifies this process.

The DBA must still configure:

- Publications.
- Subscriptions.
- Replication roles.
- Network connectivity.
- Initial table synchronization.
- Replication monitoring.
- Cutover and rollback procedures.

### **8.3 Sequence replication**

This is particularly important for application migration.

Consider:

```
CREATE SEQUENCE order_number_seq
START WITH 100000;
```

An application generates identifiers using:

```
SELECT nextval('order_number_seq');
```

Historically, logical replication of table rows did not automatically synchronize the corresponding sequence state.

This created a potential problem during application cutover.

For example:

| Component  | Sequence value |
| ---------- | -------------- |
| Source     | 500001         |
| Subscriber | 100001         |

If the application switches to the subscriber without appropriate sequence synchronization, the target may generate identifiers that conflict with existing records.

PostgreSQL 19 introduces sequence-value synchronization through logical replication.

### **8.4 Publish all sequences**

The new publication capability includes:

```
CREATE PUBLICATION banking_publication
FOR ALL TABLES, ALL SEQUENCES;
```

Sequence synchronization can also be refreshed through subscription management.

For example:

```
ALTER SUBSCRIPTION banking_subscription
REFRESH PUBLICATION
WITH (REFRESH SEQUENCES);
```

Inspect sequence state using:

```
SELECT *
FROM pg_get_sequence_data(
    'order_number_seq'::regclass
);
```

The actual synchronization occurs through the publication/subscription lifecycle. Review the official command syntax and permissions before implementing this in production.&#x20;


### **8.5 Other logical replication enhancements**

PostgreSQL 19 also introduces:

- `EXCEPT` support when publishing all tables.
- Subscription connection configuration through a foreign server using `SERVER`.
- `retain_dead_tuples` for detecting certain update/delete conflicts.
- `max_retention_duration` to limit dead-tuple retention.
- Additional subscription statistics for sequence synchronization errors.

These improvements are particularly useful in environments with complex replication and migration requirements.&#x20;


## **9. WAIT FOR LSN: Read-your-writes consistency on standby servers**

Important for distributed applications

### **9.1 Understanding the problem**

Consider a PostgreSQL primary/standby architecture.

An application executes a transaction on the primary.

The transaction commits successfully.

Immediately afterwards, the application sends a read request to a standby.

However, the standby may not yet have replayed the relevant WAL records.

The application could therefore read an older version of the data.

This is a replication consistency problem.

### **9.2 Understanding LSN**

LSN means Log Sequence Number.

PostgreSQL uses LSNs to identify positions within the Write-Ahead Log.

On a primary:

```
SELECT pg_current_wal_lsn();
```

On a standby:

```
SELECT pg_last_wal_replay_lsn();
```

The LSN helps identify how far WAL processing has progressed.

### **9.3 New WAIT command**

PostgreSQL 19 introduces:

```
WAIT FOR LSN
```

Documented syntax:

```
WAIT FOR LSN '0/16B6C50'
WITH (
    MODE 'standby_replay',
    TIMEOUT '10s'
);
```

The command supports four modes:

| Mode             | Purpose                                                   |
| ---------------- | --------------------------------------------------------- |
| `standby_replay` | Wait until the standby replays the specified WAL position |
| `standby_write`  | Wait until WAL is written on the standby                  |
| `standby_flush`  | Wait until WAL is flushed on the standby                  |
| `primary_flush`  | Wait until WAL is flushed on the primary                  |

The default mode is `standby_replay`.

The timeout is specified in milliseconds or with a time-unit suffix.&#x20;


### **9.4 Real-world banking scenario**

A customer updates an account balance.

The transaction commits on the primary.

The application subsequently needs to read the updated balance from a standby.

The general workflow is:

1. Execute the UPDATE on the primary.
2. Commit the transaction.
3. Obtain the relevant WAL position.
4. Pass the target LSN to the standby.
5. Execute `WAIT FOR LSN`.
6. Read the record after successful completion.

Example:

On the primary, after committing the transaction:

```
SELECT pg_current_wal_lsn();
```

On the standby:

```
WAIT FOR LSN '0/16B6C50'
WITH (
    MODE 'standby_replay',
    TIMEOUT '10s'
);
```

Then:

```
SELECT
    customer_id,
    account_balance
FROM customer_accounts
WHERE customer_id = 10001;
```

The LSN shown is illustrative. Production applications must capture and pass the correct transaction-related WAL position, rather than use a hardcoded value.

### **9.5 Handling timeout conditions**

PostgreSQL 19 also supports `NO_THROW`.

Example:

```
WAIT FOR LSN '0/16B6C50'
WITH (
    MODE 'standby_replay',
    TIMEOUT '10s',
    NO_THROW
);
```

Possible outcomes include:

- `success`
- `timeout`
- `not in recovery`

An application must inspect the returned status before assuming the target LSN has been reached.

This is especially important when a standby is promoted during the wait operation.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.postgresql.org\&sz=32)

PostgreSQL



### **9.6 Where this feature is useful**

- Banking applications.
- E-commerce order processing.
- Inventory systems.
- Financial transaction platforms.
- Multi-region database deployments.
- Read-replica architectures.

**DBA recommendation:** `WAIT FOR LSN` is a consistency-coordination mechanism. It does not replace synchronous replication, transaction commit guarantees or HA/DR design.

## **10. Automatic I/O worker scaling**

PostgreSQL 18 introduced asynchronous I/O improvements.

PostgreSQL 19 builds on this architecture by introducing automatic worker scaling when:

```
io_method = 'worker'
```

The new configuration parameters include:

```
io_min_workers = 2
io_max_workers = 16
io_worker_idle_timeout = '10s'
io_worker_launch_interval = '10ms'
```

The values above are illustrative.

The previous `io_workers` parameter is removed.

### **10.1 Why is this important?**

Consider a reporting platform:

- Low database activity overnight.
- High reporting activity in the morning.
- Large analytical queries during business hours.
- Periodic bulk-loading operations.

A fixed worker configuration can be inefficient when workload requirements fluctuate.

PostgreSQL 19 automatically adjusts the number of I/O workers within the configured minimum and maximum limits.

### **10.2 DBA validation**

Review:

```
SHOW io_method;
SHOW io_min_workers;
SHOW io_max_workers;
SHOW io_worker_idle_timeout;
SHOW io_worker_launch_interval;
```

Benchmark using:

```
EXPLAIN (ANALYZE, BUFFERS, IO)
SELECT *
FROM transactions
WHERE transaction_date >= DATE '2026-01-01';
```

The `IO` option is another PostgreSQL 19 enhancement that exposes asynchronous I/O activity in EXPLAIN ANALYZE output.

**DBA recommendation:** Compare throughput, CPU consumption, read latency and workload concurrency before changing the default I/O configuration.&#x20;


## **11. Query optimizer and execution improvements**

PostgreSQL 19 includes several optimizer and execution improvements that can benefit existing applications without requiring SQL changes.

### **11.1 Improved NOT IN optimization**

Consider:

```
SELECT *
FROM customers
WHERE customer_id NOT IN
(
    SELECT customer_id
    FROM blocked_customers
);
```

PostgreSQL 19 improves the ability to convert eligible `NOT IN` expressions into more efficient anti-join operations when NULL values are not present.

This is important because NULL semantics can affect whether such a transformation is valid.

### **11.2 Eager aggregation before joins**

Consider a reporting query involving large transaction and customer tables.

Traditional execution may process a large join result before aggregation.

PostgreSQL 19 improves the optimizer's ability to perform some aggregation work before joins.

This can reduce the number of rows processed in subsequent operations.

### **11.3 Improved foreign-key checks**

Foreign-key checks have also been optimized.

The official release announcement reports that testing demonstrated nearly 2x faster bulk inserts in relevant workloads.

This is not a universal performance guarantee. Actual improvement depends on schema design, workload characteristics and data distribution.

### **11.4 Other optimizer improvements**

The release notes also identify:

- Improved hash-join handling of NULL join keys.
- Better semijoin planning.
- More opportunities for Memoize with anti-joins.
- Incremental-sort planning improvements.
- Improved selectivity calculations.
- Extended statistics for virtual generated columns.

These are worth testing against real application workloads.&#x20;


## **12. New SQL capabilities for developers**

PostgreSQL 19 includes several SQL improvements that are especially relevant to application developers.

### **12.1 INSERT ... ON CONFLICT DO SELECT**

New SQL capability

One of the interesting additions is:

```
INSERT ... ON CONFLICT DO SELECT
```

This supports the common *get-or-create* application pattern.

Previously, applications often required multiple database round trips.

For example:

1. Attempt to insert a customer.
2. If the customer already exists, execute another SELECT.
3. Return the existing customer record.

PostgreSQL 19 allows the conflicting existing row to be returned through the new conflict-handling capability.

Illustrative pattern:

```
INSERT INTO customers
(
    customer_id,
    customer_name
)
VALUES
(
    10001,
    'John'
)
ON CONFLICT (customer_id)
DO SELECT
RETURNING *;
```

The exact conflict target and locking behaviour must be designed according to the application requirements.

This feature can reduce application complexity and network round trips.&#x20;


### **12.2 COPY TO JSON**

PostgreSQL 19 adds JSON output support to COPY.

Example:

```
COPY customers
TO STDOUT
WITH (FORMAT JSON);
```

It also supports exporting the result as a single JSON array using `FORCE_ARRAY`.

Potential applications:

- API data exports.
- Integration platforms.
- Application migration.
- Data interchange.
- Reporting pipelines.

This provides another option for exporting PostgreSQL data without requiring an external application to convert every row into JSON.

### **12.3 SQL/JSON path string methods**

PostgreSQL 19 adds additional string methods to the SQL/JSON path language.

These include:

- `lower()`
- `upper()`
- `replace()`
- `split_part()`
- `btrim()`

This is useful when applications need to perform string transformations during JSON-path evaluation.

### **12.4 IGNORE NULLS in window functions**

PostgreSQL 19 adds `IGNORE NULLS` and `RESPECT NULLS` support for:

- `lead()`
- `lag()`
- `first_value()`
- `last_value()`
- `nth_value()`

Example:

```
SELECT
    customer_id,
    transaction_date,
    transaction_amount,
    LAG(transaction_amount)
        IGNORE NULLS
        OVER
        (
            PARTITION BY customer_id
            ORDER BY transaction_date
        ) AS previous_amount
FROM transactions;
```

This is useful in analytical workloads involving missing values and historical comparisons.

### **12.5 Random dates and timestamps**

The `random()` function can now generate random dates and timestamps within a specified range.

This can be useful for generating test datasets and synthetic workload inputs.

These developer-oriented enhancements are documented in the official release notes.&#x20;


## **13. Improved monitoring and observability**

PostgreSQL 19 introduces several new monitoring views and additional instrumentation.

For DBAs, this is one of the most useful areas of the release.

### **13.1 New monitoring views**

| View                          | Purpose                                      |
| ----------------------------- | -------------------------------------------- |
| `pg_stat_lock`                | Statistics by lock type                      |
| `pg_stat_recovery`            | Recovery status                              |
| `pg_stat_progress_repack`     | REPACK, CLUSTER and VACUUM FULL progress     |
| `pg_stat_autovacuum_scores`   | Autovacuum prioritization                    |
| `pg_dsm_registry_allocations` | Dynamic shared-memory allocation information |

### **13.2 Lock monitoring**

```
SELECT *
FROM pg_stat_lock;
```

This provides additional visibility into lock activity by lock type.

It is useful when investigating database contention and lock-related performance issues.

### **13.3 Recovery monitoring**

```
SELECT *
FROM pg_stat_recovery;
```

This provides additional recovery-related visibility.

It is useful for administrators managing recovery and standby environments.

### **13.4 REPACK monitoring**

```
SELECT *
FROM pg_stat_progress_repack;
```

This helps DBAs monitor long-running maintenance operations.

### **13.5 WAL monitoring improvements**

PostgreSQL 19 adds reporting of WAL bytes generated for full-page images.

This information is available through:

- `pg_stat_wal`
- `EXPLAIN (ANALYZE, WAL)`
- VACUUM and ANALYZE logging

For example:

```
EXPLAIN (ANALYZE, BUFFERS, WAL)
UPDATE transactions
SET transaction_status = 'PROCESSED'
WHERE transaction_id = 10001;
```

This helps investigate the WAL impact of database operations.

### **13.6 Improved EXPLAIN ANALYZE**

The new `IO` option exposes asynchronous I/O activity.

Example:

```
EXPLAIN
(
    ANALYZE,
    BUFFERS,
    WAL,
    IO
)
SELECT *
FROM transactions
WHERE transaction_id = 10001;
```

This provides a richer performance-investigation picture.

**DBA recommendation:** Update monitoring scripts, dashboards and operational runbooks to take advantage of the new statistics.&#x20;


## **14. Security and authentication improvements**

PostgreSQL 19 includes several important security and configuration changes.

### **14.1 Server Name Indication (SNI)**

PostgreSQL 19 introduces server-side TLS SNI support.

SNI allows a server to select a TLS certificate based on the hostname used by the connecting client.

The new configuration file is:

```
PGDATA/pg_hosts.conf
```

This is useful in environments where one PostgreSQL server serves connections using multiple DNS names.

For example:

| Hostname              | Certificate            |
| --------------------- | ---------------------- |
| db-prod.company.com   | Production certificate |
| db-report.company.com | Reporting certificate  |
| db-api.company.com    | API certificate        |

This can simplify TLS certificate management for certain multi-hostname deployments.

### **14.2 MD5 authentication warnings**

MD5 password authentication was deprecated in PostgreSQL 18.

PostgreSQL 19 introduces a warning after successful MD5 authentication.

The warning can be controlled through:

```
md5_password_warnings
```

For password-based authentication, plan migration to SCRAM-SHA-256.

Review:

- `pg_hba.conf`.
- Application connection settings.
- JDBC/ODBC drivers.
- Connection pools.
- Password rotation procedures.
- Authentication integrations.

### **14.3 RADIUS authentication removed**

PostgreSQL 19 removes RADIUS authentication support.

The PostgreSQL project identifies the previous UDP-only implementation as insecure.

Organizations relying on RADIUS must review their authentication architecture before upgrading.

### **14.4 Password expiration warning**

A new setting is introduced:

```
password_expiration_warning_threshold
```

It warns users when their password is approaching expiration.

The default warning period is seven days.

### **14.5 JIT disabled by default**

PostgreSQL 19 changes the default JIT configuration to disabled.

Check:

```
SHOW jit;
```

If the workload benefits from JIT, evaluate:

```
SET jit = on;
```

Large analytical queries may benefit, but the result depends on query complexity and execution characteristics.

Do not enable JIT globally without benchmarking.&#x20;


## **15. TOAST compression changes**

PostgreSQL 19 changes the default TOAST compression method from `pglz` to `lz4`, when PostgreSQL has been built with LZ4 support.

TOAST manages large variable-length values that cannot efficiently fit within the ordinary row storage structure.

Typical examples:

- Large text columns.
- JSON documents.
- XML payloads.
- Large application descriptions.
- Variable-length data.

Check:

```
SHOW default_toast_compression;
```

Example table:

```
CREATE TABLE application_logs
(
    log_id BIGINT GENERATED ALWAYS AS IDENTITY,
    log_payload TEXT
);
```

### **Production scenario**

An application stores millions of JSON and text payloads.

The DBA wants to evaluate whether the new compression default improves storage efficiency and performance.

Benchmark:

- Table size.
- Index size.
- CPU utilization.
- Insert throughput.
- Update throughput.
- Read performance.

Do not assume that every workload will achieve the same compression ratio.

## **16. Features that did not make it into PostgreSQL 19**

The original LinkedIn post mentions three capabilities that were not delivered.

It is important not to advertise these as PostgreSQL 19 features.

| Feature                                  | Status in PostgreSQL 19 |
| ---------------------------------------- | ----------------------- |
| SQL/PGQ graph queries                    | Not listed as delivered |
| Temporal UPDATE / DELETE FOR PORTION OF  | Not listed as delivered |
| Native MERGE / SPLIT PARTITIONS commands | Not listed as delivered |

### **16.1 SQL/PGQ graph queries**

SQL/PGQ is intended to provide graph-query capabilities using SQL-based syntax.

Potential applications:

- Fraud investigation.
- Customer relationship analysis.
- Financial transaction networks.
- Recommendation systems.
- Entity relationship discovery.

These are interesting capabilities, but they should not be included in a PostgreSQL 19 production feature list based on the official release documentation.

### **16.2 Temporal UPDATE / DELETE FOR PORTION OF**

Temporal database capabilities allow applications to manage information over defined periods.

Potential use cases include:

- Employee salary history.
- Insurance policy validity.
- Contract changes.
- Historical pricing.
- Account ownership history.

The specific temporal `UPDATE ... FOR PORTION OF` and `DELETE ... FOR PORTION OF` commands are not listed as delivered PostgreSQL 19 features.

### **16.3 MERGE / SPLIT PARTITIONS**

This would be particularly useful for large partitioned transaction tables.

For example:

```
transactions
    |
    |-- transactions_2026_q1
    |-- transactions_2026_q2
    |-- transactions_2026_q3
    |-- transactions_2026_q4
```

An organization might want to change quarterly partitions into monthly partitions as its data volume grows.

Native MERGE/SPLIT PARTITIONS commands are not listed as PostgreSQL 19 additions.

Existing PostgreSQL partition-management functionality remains available.

## **17. PostgreSQL 19 migration and upgrade strategy**

This is where production DBAs need to pay particular attention.

A major-version upgrade is not simply a binary replacement.

PostgreSQL 19 introduces compatibility changes that can affect applications, extensions, authentication, SQL behaviour and database operations.

The official documentation identifies three principal migration approaches:

1. `pg_upgrade`
2. Dump and restore.
3. Logical replication.&#x20;


### **17.1 Important PostgreSQL 19 compatibility changes**

| Change                                                     | DBA action                                              |
| ---------------------------------------------------------- | ------------------------------------------------------- |
| `io_workers` removed                                       | Migrate to new I/O worker parameters                    |
| JIT disabled by default                                    | Benchmark analytical queries                            |
| `max_locks_per_transaction` default changed from 64 to 128 | Review lock configuration and shared-memory assumptions |
| RADIUS support removed                                     | Replace affected authentication integrations            |
| MD5 authentication warnings                                | Plan SCRAM migration                                    |
| `standard_conforming_strings` forced ON                    | Review legacy SQL and dump/restore procedures           |
| `MULE_INTERNAL` encoding removed                           | Identify and migrate affected databases                 |
| `inet`/`cidr` default index opclasses changed              | Review affected indexes                                 |
| Full-text search and `pg_trgm` locale behaviour changed    | Review and rebuild affected indexes where required      |
| Wait-event names changed                                   | Update monitoring and alerting scripts                  |
| `sync_error_count` renamed                                 | Update subscription monitoring queries                  |

These are documented compatibility changes, not merely optional tuning recommendations.&#x20;


### **17.2 Important: Full-text search and pg_trgm**

PostgreSQL 19 changes character classification behaviour for full-text search and `pg_trgm` to use the database locale provider.

For affected databases, the documentation recommends:

- Reindexing affected full-text-search indexes.
- Reindexing affected `pg_trgm` indexes.
- Recomputing stored `tsvector` values.

This is especially important when upgrading databases using ICU or the built-in locale provider.

Do not blindly rebuild every index without identifying whether the database is affected.&#x20;


### **17.3 Upgrade using pg_upgrade**

Typical process:

1. Verify the current cluster.
2. Take a complete backup.
3. Validate extensions.
4. Install PostgreSQL 19 binaries.
5. Initialize or prepare the new cluster.
6. Run compatibility checks.
7. Execute `pg_upgrade`.
8. Validate database objects.
9. Rebuild affected indexes where required.
10. Refresh statistics.
11. Execute application regression testing.

Example command structure:

```
pg_upgrade \
    --old-bindir=/usr/lib/postgresql/18/bin \
    --new-bindir=/usr/lib/postgresql/19/bin \
    --old-datadir=/var/lib/postgresql/18/main \
    --new-datadir=/var/lib/postgresql/19/main \
    --check
```

The paths are illustrative and must be adapted to the operating system and installation layout.

### **17.4 Dump and restore**

For logical migration or clean-cluster migration:

Backup:

```
pg_dumpall \
    -U postgres \
    -f complete_backup.sql
```

Restore:

```
psql \
    -U postgres \
    -f complete_backup.sql
```

For large databases, consider parallel database-level backup and restore using `pg_dump` and `pg_restore`.

Also validate roles, tablespaces, extensions, ownership, privileges and database-level configuration.

### **17.5 Logical replication migration**

A typical migration architecture:

**Source PostgreSQL 18**

Existing production workload

Logical replication

**Target PostgreSQL 19**

New production environment

**Validation → Cutover → Application testing**

This approach can reduce downtime, but it requires careful handling of:

- Initial data synchronization.
- Ongoing changes.
- DDL changes.
- Sequence synchronization.
- Replication lag.
- Long-running transactions.
- Application write cutover.
- Rollback.

**Important:** Logical replication is not a complete physical HA/DR replacement. It does not automatically migrate every database object or every piece of cluster configuration.

## **18. PostgreSQL 19 hands-on DBA laboratory**

For someone working in PostgreSQL administration, I recommend validating the new functionality through practical exercises rather than simply memorizing the new commands.

### **Recommended lab exercises**

| Lab | Practical exercise                                 | Learning outcome                    |
| --- | -------------------------------------------------- | ----------------------------------- |
| 1   | Create a large test table and generate dead tuples | Understand table bloat              |
| 2   | Execute `REPACK`                                   | Table reorganization                |
| 3   | Execute `REPACK (CONCURRENTLY)`                    | Locking and concurrency             |
| 4   | Generate query-plan advice                         | Query optimizer control             |
| 5   | Test `pg_stash_advice`                             | Automatic plan stabilization        |
| 6   | Configure parallel autovacuum                      | Maintenance parallelism             |
| 7   | Examine autovacuum scores                          | Maintenance prioritization          |
| 8   | Configure logical replication                      | Publisher/subscriber administration |
| 9   | Synchronize sequences                              | Migration consistency               |
| 10  | Execute `WAIT FOR LSN`                             | Read-your-writes consistency        |
| 11  | Benchmark automatic I/O workers                    | Asynchronous I/O                    |
| 12  | Test PostgreSQL 18 to 19 migration                 | Major-version upgrade               |

### **Suggested three-node architecture**

## **PostgreSQL 19 HA/DR laboratory**

**Node 1 – Primary**

PostgreSQL 19 · OLTP workload

**Node 2**

Physical standby

**Node 3**

Logical subscriber

**Test scenarios**

- WAL generation and replay.
- Read-after-write consistency.
- Sequence synchronization.
- Logical replication monitoring.
- Autovacuum under heavy UPDATE/DELETE workloads.
- Concurrent table repacking.
- Planned failover and recovery.

Remember that physical replication, logical replication and automatic failover orchestration are separate capabilities. A three-node topology does not automatically provide automated failover.

## **19. PostgreSQL 19 production readiness checklist**

Use this checklist before adopting PostgreSQL 19 in a production environment.

**PostgreSQL 19 readiness**

0/15

Confirm PostgreSQL 19.0 installation and patch level

Review official PostgreSQL 19 release notes

Inventory extensions and application dependencies

Validate full backup and point-in-time recovery

Test pg_upgrade or the selected migration strategy

Review MD5 and RADIUS authentication dependencies

Validate full-text search and pg_trgm compatibility

Review JIT and I/O worker configuration

Benchmark parallel autovacuum

Test REPACK disk and WAL requirements

Validate logical replication and sequence synchronization

Test WAIT FOR LSN timeout and promotion behaviour

Update monitoring dashboards and alerting scripts

Execute application regression testing

Validate HA/DR failover and rollback procedures

Reset checklist

## **20. Final DBA assessment: Which PostgreSQL 19 features deserve priority?**

My assessment of the release from an enterprise DBA perspective:

| Area                         | Priority  | Reason                                             |
| ---------------------------- | --------- | -------------------------------------------------- |
| REPACK CONCURRENTLY          | Very high | Physical storage reclamation with reduced blocking |
| pg_plan_advice               | Very high | Query-plan stability                               |
| Parallel autovacuum          | Very high | Large-table maintenance                            |
| WAIT FOR LSN                 | Very high | Read consistency across replicas                   |
| Logical sequence replication | High      | Safer migration and cutover                        |
| Automatic I/O workers        | High      | Workload-dependent I/O scalability                 |
| pg_stash_advice              | High      | Stored query-plan management                       |
| Monitoring enhancements      | High      | Better operational visibility                      |
| SNI                          | Medium    | Enterprise TLS architecture                        |
| New SQL capabilities         | Medium    | Application development improvements               |

### **My recommended learning order**

If I were preparing a PostgreSQL DBA team for PostgreSQL 19, I would prioritize these seven subjects:

1. **Storage management:** REPACK, VACUUM, table bloat and index maintenance.
2. **Performance engineering:** `pg_plan_advice`, `pg_stash_advice` and EXPLAIN.
3. **Maintenance architecture:** Parallel autovacuum and transaction ID wraparound.
4. **Replication:** Logical replication, sequence synchronization and conflict management.
5. **HA/DR:** WAIT FOR LSN, WAL replay and standby consistency.
6. **I/O architecture:** Asynchronous I/O and automatic worker scaling.
7. **Upgrade engineering:** PostgreSQL 18 to 19 compatibility and migration.

## **21. Official PostgreSQL documentation and references**

Use these official resources as the technical reference material for implementation and further learning.

**PostgreSQL Global Development Group**

**Release and upgrade documentation**

- [PostgreSQL 19 Official Release Announcement](https://www.postgresql.org/about/news/postgresql-19-released-3449/)
- [PostgreSQL 19 Complete Release Notes](https://www.postgresql.org/docs/19/release-19.html)
- [Upgrading a PostgreSQL Cluster](https://www.postgresql.org/docs/19/upgrading.html)

  **Storage and performance**
- [REPACK SQL Command](https://www.postgresql.org/docs/19/sql-repack.html)
- [pg_plan_advice](https://www.postgresql.org/docs/19/pgplanadvice.html)
- [pg_stash_advice](https://www.postgresql.org/docs/19/pgstashadvice.html)
- [Routine Vacuuming](https://www.postgresql.org/docs/19/routine-vacuuming.html)

  **Replication and HA/DR**
- [WAIT SQL Command](https://www.postgresql.org/docs/19/sql-wait.html)
- [Logical Replication](https://www.postgresql.org/docs/19/logical-replication.html)
- [Warm Standby and Streaming Replication](https://www.postgresql.org/docs/19/warm-standby.html)

  **Monitoring and configuration**
- [Monitoring Database Activity](https://www.postgresql.org/docs/19/monitoring-stats.html)
- [Resource Consumption Configuration](https://www.postgresql.org/docs/19/runtime-config-resource.html)
- [Autovacuum Configuration](https://www.postgresql.org/docs/19/runtime-config-autovacuum.html)

## **Conclusion**

PostgreSQL 19 is an important release for database administrators, developers and organizations running business-critical PostgreSQL workloads.

The biggest operational improvements are not necessarily the new SQL syntax. They are the improvements to storage maintenance, query-plan management, vacuum scheduling, asynchronous I/O, logical replication and standby consistency.

For DBAs, the real opportunity is to understand how these capabilities change existing operational procedures.

**Do not upgrade production databases simply because PostgreSQL 19 is GA. First establish compatibility, test performance, validate recovery procedures and benchmark the new features against your actual workload.**

The best way to understand PostgreSQL 19 is to build a realistic lab, reproduce production scenarios and document the results.

*Technical reference: PostgreSQL 19.0 official documentation and release announcement, November 19, 2026.*

For **Amazon Aurora PostgreSQL**, treat alerts as a combination of **Aurora/RDS CloudWatch metrics, PostgreSQL health, replication, storage, connections, performance, and operational events**.

### Critical Aurora PostgreSQL alerts

| Priority | Alert                                | Recommended threshold / condition                                                      |
| -------- | ------------------------------------ | -------------------------------------------------------------------------------------- |
| 🔴 P1    | **Database instance unavailable**    | `DBInstanceStatus != available`                                                        |
| 🔴 P1    | **Aurora cluster unavailable**       | Cluster/instance availability failure                                                  |
| 🔴 P1    | **Writer failure / failover**        | Writer changed, failover initiated/completed                                           |
| 🔴 P1    | **Replica lag**                      | `AuroraReplicaLag` > **60 sec**; critical > **300 sec**                                |
| 🔴 P1    | **Freeable memory**                  | < **10%** or sustained < **1 GB**                                                      |
| 🔴 P1    | **CPU utilization**                  | > **90% for 15 min**                                                                   |
| 🔴 P1    | **Database connections**             | > **80% of max_connections**                                                           |
| 🔴 P1    | **Deadlocks**                        | Any sustained/increasing deadlocks                                                     |
| 🔴 P1    | **Storage capacity**                 | Aurora storage is distributed automatically, but monitor storage-related events/errors |
| 🔴 P1    | **I/O latency**                      | Sustained high read/write latency                                                      |
| 🔴 P1    | **Disk queue depth**                 | Sustained abnormal increase                                                            |
| 🔴 P1    | **Replication failure**              | Aurora PostgreSQL replication/reader errors                                            |
| 🔴 P1    | **Transaction ID / wraparound risk** | Approaching PostgreSQL XID/MXID limits                                                 |
| 🔴 P1    | **Long-running transactions**        | > **30–60 min**, depending on workload                                                 |
| 🔴 P1    | **Blocked sessions**                 | Blocking > **5 min**                                                                   |
| 🔴 P1    | **Connection exhaustion**            | Connection count approaching configured limit                                          |
| 🔴 P1    | **Failover / recovery events**       | Any unexpected failover/recovery                                                       |
| 🟠 P2    | **CPU sustained high**               | > **70–80% for 15–30 min**                                                             |
| 🟠 P2    | **Memory pressure**                  | Freeable memory < **20%**                                                              |
| 🟠 P2    | **Replica lag warning**              | > **30 sec**                                                                           |
| 🟠 P2    | **Read/write IOPS abnormal**         | Significant deviation from baseline                                                    |
| 🟠 P2    | **Network throughput abnormal**      | Sudden sustained deviation                                                             |
| 🟠 P2    | **Database connections warning**     | > **70% of max_connections**                                                           |
| 🟠 P2    | **Autovacuum problems**              | Tables accumulating excessive dead tuples                                              |
| 🟠 P2    | **Replication slots**                | Inactive/stale slots or WAL retention growing                                          |
| 🟠 P2    | **WAL generation abnormal**          | Significant unexpected increase                                                        |
| 🟠 P2    | **Checkpoint pressure**              | Excessive checkpoint frequency/duration                                                |
| 🟠 P2    | **Query latency**                    | Application-defined SLA exceeded                                                       |
| 🟠 P2    | **Locks**                            | Blocking locks > **1–5 min**                                                           |
| 🟡 P3    | **Storage growth**                   | Unusual growth rate                                                                    |
| 🟡 P3    | **Connection growth**                | Sudden increase                                                                        |
| 🟡 P3    | **CPU trend**                        | Sustained > **60–70%**                                                                 |
| 🟡 P3    | **Replica lag trend**                | Increasing but < critical threshold                                                    |
| 🟡 P3    | **Vacuum/analyze health**            | Tables falling behind maintenance                                                      |
| 🟡 P3    | **Backup/PITR issues**               | Backup or retention-related events                                                     |
| 🟡 P3    | **Parameter/configuration changes**  | Production DB parameter/security changes                                               |
| 🟡 P3    | **Maintenance events**               | Upcoming/started/completed maintenance                                                 |

### Divide the monitoring into 7 alert groups

**1. Availability**

* Cluster unavailable
* Instance unavailable
* Writer failure
* Failover
* Reader failure
* Recovery/restart
* Maintenance events

**2. Capacity**

* CPU
* Freeable memory
* DB connections
* I/O
* Network
* Storage growth

**3. PostgreSQL engine health**

* Deadlocks
* Long-running transactions
* Blocking
* Autovacuum failures
* Excessive dead tuples
* XID/MXID age
* WAL generation
* Checkpoint pressure

**4. Aurora replication**

* Reader lag
* Replica failure
* Replication errors
* WAL/replication-slot retention where applicable
* Reader unable to keep up

**5. Performance**

* Database load / **DBLoad**
* DBLoad relative to vCPU
* Top SQL
* High latency
* I/O latency
* Lock waits
* CPU saturation

**6. Backup / DR**

* Backup/PITR configuration
* Backup failures/events
* Replication/DR health
* Cross-region replica health, if used
* Recovery-related events

**7. Security & operations**

* Public accessibility/configuration changes
* Parameter-group changes
* Security-group changes
* Encryption configuration changes
* IAM/authentication failures
* RDS/Aurora configuration changes
* CloudTrail API activity

### A practical production alert hierarchy

I recommend keeping the actual **PagerDuty/SNS/Teams/Email alerting** relatively disciplined:

**P1 — Immediate DBA/On-call**

* Aurora unavailable
* Writer failure/failover
* Severe replica lag
* Connection exhaustion
* Severe memory pressure
* Deadlocks/locking causing application outage
* XID wraparound danger
* Replication failure
* Severe I/O degradation

**P2 — DBA action required**

* High CPU
* Increasing replica lag
* Long transactions
* Blocking
* Autovacuum problems
* Increasing WAL
* High DBLoad
* Increasing connection utilization
* I/O latency

**P3 — DBA review**

* Capacity trends
* Storage growth
* Maintenance
* Configuration changes
* Backup/DR warnings
* Performance trends

**Important:** Don't configure every CloudWatch metric as a critical alarm. For Aurora PostgreSQL, **DBLoad, wait events, connections, memory, replication lag, transaction age, locks, vacuum health, and failover events** generally provide much more actionable DBA signals than simply alerting on CPU/IOPS.

If building this for an **enterprise production Aurora PostgreSQL environment**, below are the **complete alert matrix with CloudWatch metric name → threshold → evaluation period → severity → SNS topic → DBA action → SQL query**, including the PostgreSQL-side queries that CloudWatch cannot detect.

Below is the **production-grade Aurora PostgreSQL alert matrix** I would use as a baseline. The important design point is that **CloudWatch alone is not enough**: combine AWS infrastructure metrics/events, CloudWatch Database Insights, PostgreSQL catalog checks, and log-based alarms.

AWS publishes Aurora metrics to CloudWatch at 1-minute intervals, and current AWS guidance uses **CloudWatch Database Insights** for database-load and wait/lock analysis. ([AWS Documentation][1])

## 1. P1 — Immediate DBA / On-call alerts

| Area         | Metric / Check               |                                P1 trigger | Evaluation | Action                                                   |
| ------------ | ---------------------------- | ----------------------------------------: | ---------- | -------------------------------------------------------- |
| Availability | DB instance state            |                            `!= available` | Immediate  | Investigate outage/failover                              |
| Availability | Writer change/failover       |                   Any unexpected failover | Immediate  | Verify new writer + application connectivity             |
| Availability | Cluster event                |                    Critical cluster event | Immediate  | DBA + AWS investigation                                  |
| Replica      | `AuroraReplicaLag`           |                             > **300 sec** | 5 min      | Remove/stop routing traffic to unhealthy reader          |
| Memory       | `FreeableMemory`             |                                 < **10%** | 5 min      | Investigate workload/connections; scale if required      |
| CPU          | `CPUUtilization`             |                                 > **90%** | 15 min     | Identify top SQL/waits; scale if sustained               |
| Connections  | `DatabaseConnections`        |              > **90% of max_connections** | 5 min      | Identify connection storm/leaks                          |
| Storage      | `AuroraVolumeBytesLeftTotal` |             < **10% of allowed capacity** | 15 min     | Capacity investigation                                   |
| DB Load      | Database Insights `DBLoad`   |                Sustained > available vCPU | 10 min     | Investigate waits/top SQL                                |
| Blocking     | PostgreSQL locks             | Blocking > **5 min** + application impact | 2–5 min    | Identify blocker and terminate only after DBA validation |
| Deadlocks    | `Deadlocks` / PostgreSQL     |                 Sudden sustained increase | 5 min      | Identify conflicting transactions                        |
| XID          | `age(datfrozenxid)`          |                                > **1.5B** | 15 min     | Immediate vacuum/XID investigation                       |
| Transactions | Old transaction              |                              > **1 hour** | 5 min      | Identify application/session                             |
| Replication  | Reader unhealthy             |               Lag continuously increasing | 5–10 min   | Investigate reader                                       |
| Logs         | Fatal/PANIC                  |                  Any sustained occurrence | Immediate  | Investigate database/application failure                 |

`AuroraReplicaLag` is reported in milliseconds, and AWS specifically recommends monitoring the individual `AuroraReplicaLag` metric on readers rather than relying only on maximum lag because temporary spikes can occur during reader lifecycle changes. ([AWS Documentation][2])

---

# 2. P2 — DBA action required

| Area              | Alert                           |                   Recommended threshold |
| ----------------- | ------------------------------- | --------------------------------------: |
| CPU               | High CPU                        |                 > **75–80% for 15 min** |
| Memory            | Memory pressure                 |                               < **20%** |
| Connections       | High connection utilization     |                            > **70–80%** |
| Replica           | Replica lag warning             |                            > **30 sec** |
| DB Load           | DBLoad approaching CPU capacity |                    > **70–80% of vCPU** |
| I/O               | Read/write latency              |        Sustained increase from baseline |
| Network           | Network utilization             |     > **70–80%** of instance capability |
| Locks             | Blocking                        |                             > **1 min** |
| Transactions      | Long-running transaction        |                            > **30 min** |
| Vacuum            | Dead tuples                     | Rapid growth / vacuum unable to keep up |
| WAL               | WAL generation                  |                > **2× normal baseline** |
| Replication slots | Retained WAL                    |              Rapid/increasing retention |
| Checkpoints       | Checkpoint pressure             |            Sustained abnormal frequency |
| Temp files        | Temp-file growth                |     Significant deviation from baseline |
| Query latency     | Slow SQL                        |                Application SLA exceeded |
| Login             | Authentication failures         |                Sudden abnormal increase |

These are **starting thresholds, not AWS-prescribed universal limits**. Aurora sizing, workload characteristics, connection pooling, query patterns and SLA should determine the final thresholds.

---

# 3. P3 — Capacity / trend alerts

These should normally create a ticket rather than page the DBA.

* CPU sustained >60–70%
* Memory declining trend
* Connection count continuously increasing
* Replica lag trend increasing
* Aurora volume consumption trend
* WAL generation trend
* Dead-tuple accumulation
* Autovacuum falling behind
* Increasing sequential scans
* Buffer-cache hit ratio deterioration
* Increasing temporary files
* Increasing query latency
* Increasing DBLoad
* Increasing lock waits
* Increasing network throughput
* Increasing read/write IOPS
* Upcoming maintenance
* Parameter-group changes
* Security/configuration changes

---

# 4. Aurora-specific storage alert

This is an important correction to traditional PostgreSQL/RDS monitoring.

**Do not treat Aurora like a normal fixed-size EBS-backed PostgreSQL server.**

Aurora storage automatically grows. Current AWS documentation says Aurora PostgreSQL storage can scale automatically up to **256 TiB for supported versions**, while earlier versions have a **128 TiB** limit. AWS specifically recommends monitoring:

`AuroraVolumeBytesLeftTotal`

when you need to understand how close the cluster is to its storage limit. ([AWS Documentation][3])

I would therefore configure:

```text
P3:
AuroraVolumeBytesLeftTotal < 25%

P2:
AuroraVolumeBytesLeftTotal < 15%

P1:
AuroraVolumeBytesLeftTotal < 10%
```

But also monitor the **rate of consumption**. A database with 40% capacity remaining but consuming several TB/day deserves attention.

---

# 5. Connection monitoring

Aurora PostgreSQL's `max_connections` is tied to the instance's available memory and parameter-group configuration. AWS also notes that every connection consumes resources, including idle connections. ([AWS Documentation][4])

### CloudWatch

```text
DatabaseConnections / max_connections
```

Recommended:

```text
> 70%  → P3
> 80%  → P2
> 90%  → P1
```

### PostgreSQL validation

```sql
SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections,
    round(
        100.0 * count(*) /
        current_setting('max_connections')::int,
        2
    ) AS connection_pct
FROM pg_stat_activity;
```

Then identify the source:

```sql
SELECT
    usename,
    application_name,
    client_addr,
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY
    usename,
    application_name,
    client_addr,
    state
ORDER BY connections DESC;
```

### Alert particularly on

```text
idle
idle in transaction
large number of connections from one application
rapid connection growth
connection failures
```

For applications with large numbers of connections, AWS recommends considering **RDS Proxy** for connection pooling. ([AWS Documentation][4])

---

# 6. Blocking / locking alert

This should be one of your **most important PostgreSQL-native alerts**.

AWS Database Insights supports lock analysis for Aurora PostgreSQL, including blocking session, blocking SQL and blocking objects. ([AWS Documentation][5])

Use PostgreSQL itself for the actual detection.

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocked.query AS blocked_query,
    now() - blocked.query_start AS blocked_duration,

    blocker.pid AS blocker_pid,
    blocker.usename AS blocker_user,
    blocker.query AS blocker_query,
    now() - blocker.query_start AS blocker_duration
FROM pg_stat_activity blocked
JOIN pg_locks blocked_lock
    ON blocked.pid = blocked_lock.pid
JOIN pg_locks blocker_lock
    ON blocker_lock.locktype = blocked_lock.locktype
    AND blocker_lock.database IS NOT DISTINCT FROM blocked_lock.database
    AND blocker_lock.relation IS NOT DISTINCT FROM blocked_lock.relation
    AND blocker_lock.page IS NOT DISTINCT FROM blocked_lock.page
    AND blocker_lock.tuple IS NOT DISTINCT FROM blocked_lock.tuple
    AND blocker_lock.virtualxid IS NOT DISTINCT FROM blocked_lock.virtualxid
    AND blocker_lock.transactionid IS NOT DISTINCT FROM blocked_lock.transactionid
    AND blocker_lock.classid IS NOT DISTINCT FROM blocked_lock.classid
    AND blocker_lock.objid IS NOT DISTINCT FROM blocked_lock.objid
    AND blocker_lock.objsubid IS NOT DISTINCT FROM blocked_lock.objsubid
    AND blocker_lock.pid <> blocked_lock.pid
JOIN pg_stat_activity blocker
    ON blocker.pid = blocker_lock.pid
WHERE NOT blocked_lock.granted
  AND blocker_lock.granted;
```

Alert:

```text
> 1 minute  → P2
> 5 minutes → P1
```

But **do not automatically kill the blocker** merely because the alarm fires. Determine whether it is a legitimate long transaction, DDL, batch operation, etc.

---

# 7. Long-running transaction alert

This is particularly important because long transactions can prevent vacuum from advancing.

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    xact_start,
    now() - xact_start AS transaction_age,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Recommended:

```text
> 15 min → P3
> 30 min → P2
> 60 min → P1
```

Adjust these for legitimate batch workloads.

---

# 8. Transaction ID / wraparound protection

This deserves a **P1-class alert** because transaction-ID exhaustion can become a database availability incident.

```sql
SELECT
    datname,
    age(datfrozenxid) AS xid_age,
    current_setting('autovacuum_freeze_max_age')::bigint
        AS freeze_max_age
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

A more useful percentage:

```sql
SELECT
    datname,
    age(datfrozenxid) AS xid_age,
    current_setting('autovacuum_freeze_max_age')::bigint AS freeze_limit,
    round(
        100.0 * age(datfrozenxid) /
        current_setting('autovacuum_freeze_max_age')::bigint,
        2
    ) AS pct_used
FROM pg_database
ORDER BY xid_age DESC;
```

Suggested operational thresholds:

```text
> 50% → P3
> 70% → P2
> 80% → P1
```

Also monitor **multixact age**, particularly for workloads with heavy row-locking/foreign-key activity.

---

# 9. Deadlock alert

Aurora exposes `Deadlocks` among its SQL-related monitoring metrics. ([AWS Documentation][6])

Don't simply configure:

```text
Deadlocks > 0
```

as a permanent P1.

Instead:

```text
1–2 isolated events → investigate
Repeated events → P2
Rapid increase / application impact → P1
```

PostgreSQL-side baseline:

```sql
SELECT
    datname,
    deadlocks
FROM pg_stat_database
ORDER BY deadlocks DESC;
```

Store the value periodically and alert on **delta**, rather than the absolute cumulative counter.

---

# 10. Autovacuum health

This is one area where **CloudWatch metrics are insufficient**.

Use PostgreSQL catalog monitoring.

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(
        100.0 * n_dead_tup /
        NULLIF(n_live_tup + n_dead_tup, 0),
        2
    ) AS dead_tuple_pct,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;
```

Starting thresholds:

```text
dead tuples > 10% → P3
dead tuples > 20% → P2
rapid dead-tuple growth → P2
autovacuum not keeping up → P1/P2 depending on impact
```

Do not blindly use one percentage for every table. High-churn tables require tighter monitoring.

---

# 11. Replication monitoring

For Aurora readers:

### CloudWatch

```text
AuroraReplicaLag
AuroraReplicaLagMaximum
AuroraReplicaLagMinimum
```

AWS documents these specifically for Aurora PostgreSQL replication monitoring. ([AWS Documentation][2])

Recommended starting point:

```text
> 10 sec → P3
> 30 sec → P2
> 60 sec → P1
> 300 sec → critical P1
```

But if your application requires:

```text
RPO/RTO < 30 seconds
```

then your alert threshold should obviously be much lower.

Also distinguish:

**Reader lag ≠ writer failure.**

A reader can have high lag while the writer is perfectly healthy.

---

# 12. Database Insights / DBLoad

I strongly recommend enabling **CloudWatch Database Insights**.

AWS defines DB Load as the level of session activity and allows you to break it down by:

* SQL
* waits
* users
* hosts
* database
* application
* blocking session
* blocking SQL

([AWS Documentation][7])

Use:

```text
DBLoad / vCPU
```

as a more meaningful signal than CPU alone.

Starting alert levels:

```text
< 0.70 × vCPU → Normal
0.70–1.00 × vCPU → P3/P2
> 1.00 × vCPU → P1 if sustained + latency impact
```

Example:

```text
8 vCPU Aurora instance

DBLoad = 2   → healthy capacity
DBLoad = 6   → investigate
DBLoad = 8   → saturated
DBLoad = 15  → severe contention
```

But don't treat DBLoad as simply "CPU". Wait states matter. AWS documents wait events such as `CPU`, `IO:DataFileRead`, `IO:XactSync`, and temporary-file I/O that can explain why DBLoad is high. ([AWS Documentation][8])

---

# 13. CloudWatch log alarms

Enable Aurora PostgreSQL PostgreSQL-log export to CloudWatch Logs. AWS provides a dedicated cluster log group under:

```text
/aws/rds/cluster/<cluster-name>/postgresql
```

([AWS Documentation][9])

Create metric filters/alarms for:

### P1

```text
PANIC
FATAL
could not write
could not extend
out of memory
database system is shutting down
database system was interrupted
```

### P2

```text
deadlock detected
canceling statement due to lock timeout
terminating connection
remaining connection slots
temporary file
duration:
```

Be careful with `duration:`. A slow-query log configuration should generate useful performance information without turning every slow query into a page.

---

# 14. RDS/Aurora EventBridge alerts

Don't rely only on CloudWatch metrics for operational events.

Aurora sends RDS events to **EventBridge** in near real time, covering cluster, instance, parameter-group, snapshot and other resource events. ([AWS Documentation][10])

I would route these to:

```text
Aurora
   ↓
EventBridge
   ↓
SNS
   ↓
Email / Teams / Slack / PagerDuty
```

### Alert on

**P1**

* Failover
* Failover started/completed
* Instance failure
* Cluster failure
* Recovery
* Unexpected restart

**P2**

* Maintenance started
* Maintenance completed
* Configuration change
* Parameter-group change
* Instance modification
* Reader added/removed
* Backup/snapshot failure

**P3**

* Scheduled maintenance notification
* Minor configuration events
* Informational lifecycle events

RDS events are best-effort, so they should complement rather than replace health metrics. ([AWS Documentation][10])

---

# 15. Network alerts

Monitor:

```text
NetworkReceiveThroughput
NetworkTransmitThroughput
```

and compare against the instance's expected capability and historical baseline.

Don't use a universal:

```text
Network > X MB/sec
```

threshold across all instance classes.

Instead:

```text
>70% sustained → P3
>80% sustained → P2
near instance/network limit + latency → P1
```

AWS specifically recommends evaluating network throughput when workload approaches instance resource limits because insufficient network bandwidth can affect processing and storage access. ([AWS Documentation][11])

---

# 16. I/O alerts

Monitor:

```text
ReadIOPS
WriteIOPS
ReadLatency
WriteLatency
ReadThroughput
WriteThroughput
```

Don't alert merely because IOPS are high.

For example:

```text
High IOPS + low latency
        ↓
Probably healthy workload

High IOPS + high latency
        ↓
Investigate

Normal IOPS + high DBLoad
        ↓
Look at CPU/locks/waits/query plans
```

That distinction prevents a huge number of false DBA alerts.

---

# 17. Backup / DR

Create alerts for:

* Backup failures
* Snapshot failures
* PITR configuration deviations
* Retention-policy changes
* Global Database replication issues, if applicable
* Cross-region replication issues
* Unexpected removal of readers
* DR topology changes

These should generally be **P1/P2 depending on whether the production recovery posture is actually compromised**.

---

# 18. Security alerts

These should come from **CloudTrail/EventBridge + AWS security tooling**, not only PostgreSQL.

Alert on:

```text
ModifyDBCluster
ModifyDBInstance
ModifyDBClusterParameterGroup
ModifyDBParameterGroup
ModifyDBSubnetGroup
ModifyDBClusterSnapshotAttribute
DeleteDBCluster
DeleteDBInstance
RestoreDBClusterFromSnapshot
CreateDBCluster
CreateDBInstance
```

Also monitor:

* Encryption configuration changes
* Security-group changes
* Public accessibility changes
* IAM authentication changes
* Parameter-group changes
* Unexpected database users/roles
* Privilege changes

These are generally **security/SOC alerts**, with severity determined by your organization's change-management policy.

---

# 19. Recommended final architecture

For an enterprise Aurora PostgreSQL environment, I would implement:

```text
                     ┌───────────────────────┐
                     │     Aurora PostgreSQL │
                     └───────────┬───────────┘
                                 │
             ┌───────────────────┼──────────────────┐
             │                   │                  │
             ▼                   ▼                  ▼
       CloudWatch            DB Insights       PostgreSQL
        Metrics               DB Load           Catalog
             │                   │                  │
             └───────────────────┼──────────────────┘
                                 │
                         Alert Evaluation
                                 │
               ┌─────────────────┼─────────────────┐
               │                 │                 │
              P1                P2                P3
               │                 │                 │
               ▼                 ▼                 ▼
            PagerDuty         DBA Queue          Email
               │
               ▼
          DBA On-Call
```

And separately:

```text
Aurora/RDS Events
       │
       ▼
  EventBridge
       │
       ▼
      SNS
       │
 ┌─────┼────────┐
 ▼     ▼        ▼
Email Teams  PagerDuty
```

---

# 20. Recommended "must-have" 20 alerts

To keep the first implementation manageable, start with these:

1. **Writer/cluster unavailable**
2. **Unexpected Aurora failover**
3. **Reader replica lag >60 sec**
4. **Reader replica lag >300 sec**
5. **CPU >90%**
6. **Freeable memory <10%**
7. **Connections >90%**
8. **AuroraVolumeBytesLeftTotal <10%**
9. **DBLoad > vCPU**
10. **Blocking >5 minutes**
11. **Long transaction >60 minutes**
12. **XID age >80%**
13. **Deadlock rate increasing**
14. **Autovacuum falling behind**
15. **WAL/replication-slot retention increasing**
16. **Critical PostgreSQL `FATAL/PANIC` log events**
17. **High I/O latency**
18. **Network saturation**
19. **Backup/DR failure**
20. **Unauthorized/unscheduled RDS configuration change**

### One important modernization point

If we're implementing this now, we would **not build a new monitoring design around the old Performance Insights terminology**. 

AWS announced the Performance Insights end-of-life date as **July 31, 2026** and has migrated users to **CloudWatch Database Insights**. 

Database Insights now provides the DB Load, wait, SQL, user, host and lock-analysis capabilities needed for this design. ([AWS Documentation][12])

This gives us a much stronger architecture than simply creating 30–40 CloudWatch CPU/memory alarms.

[1]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/monitoring-cloudwatch.html "Monitoring Amazon Aurora metrics with Amazon CloudWatch - Amazon Aurora"
[2]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html "Amazon CloudWatch metrics for Amazon Aurora - Amazon Aurora"
[3]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Performance.html "Managing performance and scaling for Aurora DB clusters - Amazon Aurora"
[4]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Managing.html "Performance and scaling for Amazon Aurora PostgreSQL - Amazon Aurora"
[5]: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights-Lock-Analysis.html "Analyzing lock trees for Amazon Aurora PostgreSQL and Amazon RDS for PostgreSQL with CloudWatch Database Insights - Amazon CloudWatch"
[6]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Monitoring.Metrics.RDSAvailability.html "Availability of Aurora metrics in the Amazon RDS console - Amazon Aurora"
[7]: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights-Database-Instance-Dashboard.html "Viewing the Database Instance Dashboard for CloudWatch Database Insights - Amazon CloudWatch"
[8]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Tuning.concepts.summary.html "Aurora PostgreSQL wait events - Amazon Aurora"
[9]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.CloudWatch.Monitor.html "Monitoring log events in Amazon CloudWatch - Amazon Aurora"
[10]: https://docs.aws.amazon.com/en_en/AmazonRDS/latest/AuroraUserGuide/working-with-events.html "Monitoring Amazon Aurora events - Amazon Aurora"
[11]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL_AnayzeResourceUsage.html "Using Amazon CloudWatch metrics to analyze resource usage for Aurora PostgreSQL - Amazon Aurora"
[12]: https://docs.aws.amazon.com/en_en/AmazonCloudWatch/latest/monitoring/Database-Insights.html "CloudWatch Database Insights - Amazon CloudWatch"

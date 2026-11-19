# **pgBackRest and PostgreSQL Failover: Why archive_mode Matters**

Advanced PostgreSQL Administration | Backup & Recovery | WAL Archiving | Streaming Replication | High Availability & Disaster Recovery

**Reference:** 
Stefan Fercot – [pgBackRest and PostgreSQL failover: why archive_mode matters](https://pgstef.github.io/2026/09/23/pgbackrest_and_postgresql_failover_why_archive_mode_matters.html)

## **1. Introduction**

In PostgreSQL production environments, backup and recovery are not simply about taking periodic full or incremental backups. A reliable recovery strategy also requires continuous Write-Ahead Log (WAL) archiving.

One frequently overlooked configuration parameter is:

```
archive_mode = on
```

This parameter becomes particularly important when a PostgreSQL standby server is promoted to become the new primary during a failover.

Consider this production scenario:

- PostgreSQL Primary (PG1) fails.
- Standby (PG2) is promoted to become the new primary.
- Another standby (PG3) continues replicating from PG2.
- The DBA attempts to run a pgBackRest backup from PG3.
- The backup fails because the promoted primary has `archive_mode=off`.

The database may be running normally, application connections may be working, and streaming replication may be healthy. However, the backup infrastructure can still be in an unsupported configuration.

**The important lesson:** Successful failover does not automatically mean that your backup and recovery architecture is healthy.

The original article demonstrates this using PostgreSQL 18, pgBackRest 2.59.1 and a three-server cascading replication topology.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://pgstef.github.io\&sz=32)

pgstef’s blog



## **2. Understanding the Core Components**

Before discussing the failure scenario, understand the responsibilities of the following PostgreSQL components.

### **2.1 Write-Ahead Logging (WAL)**

PostgreSQL uses Write-Ahead Logging to maintain database consistency and support crash recovery.

When a transaction modifies database pages:

1. PostgreSQL generates WAL records.
2. WAL records are written to WAL buffers.
3. WAL is flushed to durable storage as required.
4. Database pages are written to data files.
5. During recovery, PostgreSQL replays the necessary WAL records.

WAL is also essential for:

- Streaming replication.
- Point-in-Time Recovery (PITR).
- Crash recovery.
- Backup consistency.
- Standby synchronization.

### **2.2 WAL Archiving**

WAL archiving copies completed WAL segments to an external archive repository.

Typical configuration:

```
archive_mode = on

archive_command =
'pgbackrest --stanza=demo archive-push %p'
```

Here:

| Parameter         | Purpose                                            |
| ----------------- | -------------------------------------------------- |
| `archive_mode`    | Enables WAL archiving                              |
| `archive_command` | Specifies how completed WAL files are archived     |
| `%p`              | Full path of the WAL file                          |
| `pgbackrest`      | Backup and recovery management utility             |
| `archive-push`    | Pushes WAL segments into the configured repository |

**Important:** `archive_mode` requires a PostgreSQL restart when changed. `archive_command` can be changed using a configuration reload.

### **2.3 pgBackRest**

pgBackRest is a backup and recovery solution for PostgreSQL.

Its capabilities include:

- Full backups.
- Differential backups.
- Incremental backups.
- WAL archiving.
- Point-in-Time Recovery.
- Backup compression.
- Backup retention management.
- Parallel backup and restore.
- Backup from a standby server.

A successful backup is only one part of the recovery chain. The associated WAL files must also be available and usable.

## **3. Understanding the Three-Node Architecture**

The original experiment uses cascading streaming replication.

## **PostgreSQL HA/DR topology**

Initial Primary

**PG1**

PostgreSQL 18 · WAL archiving enabled

Streaming replication

Standby 1

**PG2**

Cascading replication source · archive_mode initially OFF

Cascading streaming replication

Standby 2

**PG3**

Backup candidate · standby of PG2

**Shared pgBackRest repository**

Stores base backups, WAL archives and recovery metadata

### **Responsibilities**

| Server     | Role              | Responsibility                          |
| ---------- | ----------------- | --------------------------------------- |
| PG1        | Primary           | Transactions, WAL generation, archiving |
| PG2        | Standby initially | Receives WAL from PG1                   |
| PG3        | Cascading standby | Receives WAL from PG2                   |
| Repository | Shared storage    | Stores backups and archived WAL         |

The important design consideration is that PG2 must be ready to become a fully functional primary, including WAL archiving.

## **4. Understanding&#x20;**`archive_mode`**: OFF, ON and ALWAYS**

PostgreSQL provides three relevant settings.

| Setting  | Behaviour                                                 |
| -------- | --------------------------------------------------------- |
| `off`    | WAL archiving is disabled                                 |
| `on`     | WAL archiving runs on the primary                         |
| `always` | WAL archiving also runs during recovery/standby operation |

### **4.1 archive_mode = off**

```
archive_mode = off
```

Consequences:

- PostgreSQL does not execute the configured archive command.
- WAL files are not automatically pushed to the archive repository.
- A standby promoted with this configuration cannot immediately begin normal WAL archiving.
- pgBackRest backup operations can fail because the primary is not configured for archiving.

### **4.2 archive_mode = on**

```
archive_mode = on

archive_command =
'pgbackrest --stanza=demo archive-push %p'
```

This is the recommended production configuration for the primary.

When a standby with this configuration is promoted:

1. It becomes the primary.
2. It starts archiving completed WAL segments.
3. pgBackRest can access the WAL archive stream.
4. New backups can be taken.
5. PITR recovery can use the archived WAL, subject to complete recovery-chain availability.

### **4.3 archive_mode = always**

```
archive_mode = always

archive_command =
'pgbackrest --stanza=demo archive-push %p'
```

This enables archiving during recovery as well as normal operation.

It can be useful in standby-based archiving architectures, but requires careful design.

**Important production warning:** Do not assume that enabling `archive_mode=always` on a standby automatically solves a broken primary archiving configuration.

Duplicate WAL archiving, checksum differences, multiple archivers and timeline management must be considered.

The original experiment demonstrates why pgBackRest protects its backup workflow with configuration checks.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://pgstef.github.io\&sz=32)

pgstef’s blog+1



## **5. Real-Time Scenario 1: Primary Failure and Standby Promotion**

Consider a banking application running on PostgreSQL.

### **Initial configuration**

| Server | IP          | Role       |
| ------ | ----------- | ---------- |
| PG1    | 10.10.10.11 | Primary    |
| PG2    | 10.10.10.12 | Standby    |
| PG3    | 10.10.10.13 | DR standby |

Assume:

- PG1 handles application transactions.
- PG2 receives streaming WAL.
- PG3 receives WAL from PG2.
- PG1 has `archive_mode=on`.
- PG2 has `archive_mode=off`.

### **Incident**

At 02:15 AM, PG1 experiences an unexpected infrastructure failure.

The DBA team promotes PG2.

```
SELECT pg_promote();
```

Check whether the server is now primary:

```
SELECT
    pg_is_in_recovery();
```

Expected result:

```
f
```

Here:

- `true` means the server is in recovery (normally a standby).
- `false` means the server is not in recovery (normally a primary).

The application reconnects successfully.

However, the DBA attempts:

```
pgbackrest --stanza=demo backup
```

The backup fails:

```
ERROR: [087]: archive_mode must be enabled
```

### **Root cause**

PG2 was originally configured with:

```
archive_mode = off
```

Promoting the standby does not automatically change this parameter.

The new primary is therefore operating without the required WAL archiving configuration.

### **Production lesson**

Do not validate HA readiness only by checking whether a standby can be promoted.

Validate whether it can also:

- Accept application connections.
- Generate and archive WAL.
- Support pgBackRest backups.
- Support PITR.
- Feed downstream replicas.
- Maintain the required recovery chain.

## **6. Real-Time Scenario 2: Taking Backups From a Standby Server**

One common DBA requirement is to offload backup workloads from the production primary.

For example:

A financial services company operates a PostgreSQL database of 4 TB.

The production primary is already handling:

- 5,000 application connections.
- Payment transactions.
- Reporting queries.
- Batch processing.
- Data integration jobs.

The infrastructure team wants to move backup processing to a standby.

pgBackRest supports standby backups.

### **Configuration**

On the backup host, configure:

```
[global]

repo1-path=/shared
repo1-retention-full=4

backup-standby=y

[demo]

pg1-path=/var/lib/pgsql/18/data

pg2-host=pg2
pg2-host-user=postgres
pg2-path=/var/lib/pgsql/18/data
```

Here, `backup-standby=y` tells pgBackRest to prefer the standby for backup processing when the required configuration is available.

The backup process needs access to:

- The standby PostgreSQL data directory.
- The primary PostgreSQL server.
- The shared repository.
- Required SSH or TLS connectivity.

Run:

```
pgbackrest --stanza=demo check
```

Take a full backup:

```
pgbackrest \
  --stanza=demo \
  --type=full \
  backup
```

### **What happens internally?**

1. pgBackRest identifies the standby.
2. It coordinates backup activity with the primary.
3. The standby reaches the required WAL replay position.
4. The backup is performed.
5. Required WAL segments are checked for archiving.
6. The backup is registered in the repository.

The advantage is reduced backup-related I/O and processing pressure on the production primary.

However, a standby backup is not a complete substitute for primary-side WAL archiving.

## **7. Understanding PostgreSQL Timelines: The Most Important Part**

This is the most critical technical concept in the original article.

Many PostgreSQL DBAs understand WAL archiving and streaming replication but overlook timeline management during failover.

### **7.1 What is a PostgreSQL timeline?**

A timeline represents a particular history of a PostgreSQL cluster's WAL progression.

When PostgreSQL promotes a standby to primary, it creates a new timeline.

For example:

## **Timeline transition during failover**

**Timeline 1**

PG1 — Original primary

PG1 failure

**Timeline 2**

PG2 — Promoted primary

Subsequent WAL generation

**Timeline 2 continues**

New transactions and archived WAL

In the original experiment:

- PG1 was on timeline 1.
- PG2 was promoted.
- PG2 created timeline 2.
- WAL archiving had not been enabled on PG2.
- The archive repository was missing part of the new timeline's recovery information.

### **7.2 Why is the timeline history file important?**

PostgreSQL creates timeline history files.

Example:

```
00000002.history
```

This file describes the relationship between timeline 2 and its parent timeline.

Example content:

```
1 0/920000A0 no recovery target specified
```

This indicates:

- Parent timeline: 1.
- Timeline switch position: `0/920000A0`.
- Timeline 2 branched from timeline 1 at that WAL position.

Why does this matter?

During PITR, PostgreSQL must understand which timeline to follow.

Without the required timeline history and WAL files, recovery across the timeline transition may not be possible.

### **7.3 What happened in the experiment?**

The original repository contained WAL from timeline 1 and WAL from timeline 2, but there was a gap around the promotion.

Conceptually:

| Repository content      | Availability            |
| ----------------------- | ----------------------- |
| Timeline 1 WAL          | Available               |
| WAL around promotion    | Missing                 |
| Timeline 2 WAL          | Available               |
| Timeline 2 history file | Missing from repository |

This is a serious DR issue.

A new backup taken after promotion might restore successfully, but that does not mean an older backup can be recovered continuously across the failover.

**Remember:** Backup success and historical recoverability are two different things.

[image](https://www.google.com/s2/favicons?domain=https://pgstef.github.io\&sz=32)

pgstef’s blog+1

## **8. Understanding&#x20;**`archive-mode-check`**&#x20;in pgBackRest**

The original article investigates whether this option can bypass the archiving validation:

```
pgbackrest \
  --stanza=demo \
  --no-archive-mode-check \
  backup
```

The experiment demonstrates an important distinction.

The option is designed to address the safety checks associated with `archive_mode=always`, not to override every archive configuration requirement.

In particular, the author found that:

```
ERROR: [087]: archive_mode must be enabled
```

still occurred when the primary had `archive_mode=off`.

### **Why does pgBackRest perform these checks?**

Consider the following situation:

- PG2 is the primary.
- PG3 is the standby.
- Both can potentially write WAL files to the same repository.
- The same logical WAL segment may be archived through different paths.
- Different checksums may be observed.

This creates repository consistency concerns.

pgBackRest validates the configuration to protect the backup and recovery process.

### **Production recommendation**

Do not modify pgBackRest source code to bypass its safety checks in a production environment.

The article demonstrates that a modified development build can complete a backup and restore it successfully. However, that experiment does not prove that all historical recovery scenarios are safe.

Use a supported configuration and test the complete recovery chain.

## **9. Real-Time Scenario 3: Banking Application With a Strict RPO**

Consider a banking organization with the following requirements:

| Requirement     | Target     |
| --------------- | ---------- |
| Database size   | 2 TB       |
| RPO             | 5 minutes  |
| RTO             | 30 minutes |
| Primary         | PG1        |
| HA standby      | PG2        |
| DR standby      | PG3        |
| Backup software | pgBackRest |

The organization wants to achieve continuous recoverability.

### **Recommended configuration**

On all promotion candidates:

```
archive_mode = on

archive_command =
'pgbackrest --stanza=prod archive-push %p'
```

On standby servers, also configure recovery-side WAL retrieval where required:

```
restore_command =
'pgbackrest --stanza=prod archive-get %f "%p"'
```

On the primary, configure:

```
wal_level = replica

max_wal_senders = 10

max_replication_slots = 10
```

These are illustrative values. Actual settings must be sized according to the number of replicas, workload and replication-slot requirements.

### **Operational flow**

1. PG1 processes banking transactions.
2. WAL is generated.
3. WAL is streamed to PG2.
4. PG2 streams WAL to PG3.
5. PG1 archives completed WAL segments.
6. pgBackRest stores full and incremental backups.
7. Monitoring checks WAL archiving delays.
8. In the event of failover, PG2 is promoted.
9. PG2 begins archiving its new timeline.
10. New backups can be taken using the updated configuration.

### **Important RPO consideration**

Streaming replication and WAL archiving are two different recovery mechanisms.

An RPO of five minutes cannot be guaranteed merely by configuring `archive_mode=on`.

You must also account for:

- Replication lag.
- WAL archive delay.
- Network outages.
- Storage availability.
- Synchronous versus asynchronous replication.
- Failure detection and promotion time.

## **10. Real-Time Scenario 4: Planned Maintenance and Switchover**

Suppose the infrastructure team needs to patch PG1.

Instead of allowing an unexpected failover, the DBA plans a controlled switchover.

### **Before switchover**

Check the current primary:

```
SELECT
    pg_is_in_recovery();
```

Check connected standbys:

```
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

Check WAL configuration:

```
SHOW archive_mode;

SHOW archive_command;

SHOW wal_level;
```

Check archive status:

```
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time,
    stats_reset
FROM pg_stat_archiver;
```

Check the standby configuration independently:

```
SHOW archive_mode;

SHOW archive_command;

SHOW restore_command;
```

### **Switchover considerations**

Before promoting PG2:

1. Verify replication health.
2. Confirm PG2 has replayed the required WAL.
3. Confirm the promotion candidate has `archive_mode=on`.
4. Confirm its `archive_command` is configured.
5. Verify repository accessibility.
6. Verify the archive command can operate after promotion.
7. Confirm the old primary will not continue accepting writes.

After promotion, verify WAL archiving and pgBackRest backup functionality.

**Do not assume that a successful switchover proves the backup infrastructure is healthy.**

## **11. Recommended PostgreSQL Configuration for Production**

For a production cluster using pgBackRest, a good starting point is:

### **postgresql.conf**

```
# WAL configuration

wal_level = replica

# WAL archiving

archive_mode = on

archive_command =
'pgbackrest --stanza=prod archive-push %p'

# WAL retention

wal_keep_size = 2GB

# Replication

max_wal_senders = 10

max_replication_slots = 10
```

### **Standby recovery configuration**

```
restore_command =
'pgbackrest --stanza=prod archive-get %f "%p"'

primary_conninfo =
'host=pg1 port=5432 user=replic_user application_name=pg2'

primary_slot_name = 'pg2_slot'
```

The `primary_slot_name` setting is optional and requires a corresponding physical replication slot on the upstream primary.

For cascading replication, PG3 should point to PG2.

### **Important configuration distinction**

| Parameter          | Restart required? |
| ------------------ | ----------------- |
| `archive_mode`     | Yes               |
| `archive_command`  | No, reload        |
| `wal_level`        | Yes               |
| `max_wal_senders`  | Yes               |
| `wal_keep_size`    | No, reload        |
| `primary_conninfo` | No, reload        |
| `restore_command`  | No, reload        |

Always confirm the parameter's context and pending-restart status for your PostgreSQL version.

## **12. Monitoring and Troubleshooting**

A PostgreSQL DBA should monitor WAL archiving continuously, not only when a backup fails.

### **12.1 Check archive configuration**

```
SELECT
    name,
    setting,
    context,
    pending_restart
FROM pg_settings
WHERE name IN
(
    'archive_mode',
    'archive_command',
    'wal_level'
);
```

This helps identify whether a parameter change requires a restart.

### **12.2 Check archiver statistics**

```
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time,
    stats_reset
FROM pg_stat_archiver;
```

Investigate:

- Increasing `failed_count`.
- An old `last_archived_time`.
- Repeated failures for the same WAL segment.
- A growing WAL backlog.

Remember that a lack of recent archived WAL does not automatically indicate a problem on an idle database.

### **12.3 Check replication status**

Run on the primary:

```
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    write_lag,
    flush_lag,
    replay_lag,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

### **12.4 Check WAL receiver status**

Run on the standby:

```
SELECT
    status,
    receive_start_lsn,
    written_lsn,
    flushed_lsn,
    latest_end_lsn,
    latest_end_time,
    slot_name,
    sender_host,
    sender_port
FROM pg_stat_wal_receiver;
```

### **12.5 Check pgBackRest repository status**

Linux shell:

```
pgbackrest --stanza=prod info
```

Validate:

- Latest full backup.
- Latest differential backup.
- Latest incremental backup.
- WAL archive minimum and maximum.
- Backup timestamps.
- Repository availability.

Run:

```
pgbackrest --stanza=prod check
```

This validates important backup and archiving prerequisites. It is not a substitute for an actual restore test.

### **12.6 Check archived WAL files**

```
pgbackrest repo-ls archive/prod/
```

Inspect timeline history files and WAL archive continuity as part of recovery investigations.

## **13. Real-Time Scenario 5: Disaster Recovery After an Accidental DELETE**

Consider an e-commerce database.

A developer accidentally executes:

```
DELETE FROM orders;
```

The transaction is committed.

The DBA must recover the database to a point immediately before the accidental deletion.

This is where PITR becomes important.

### **Recovery prerequisites**

You need:

1. A valid base backup.
2. All required WAL segments.
3. Required timeline history files.
4. Correct recovery configuration.
5. A suitable recovery target.

Example recovery configuration:

```
restore_command =
'pgbackrest --stanza=prod archive-get %f "%p"'
```

For PostgreSQL versions supporting the current recovery target settings, configure a recovery target using `recovery_target_time` in the recovery configuration.

Example:

```
recovery_target_time = '2026-11-18 10:29:00+00'

recovery_target_action = 'promote'
```

Then restore using pgBackRest:

```
pgbackrest \
  --stanza=prod \
  --type=time \
  --target="2026-11-18 10:29:00+00" \
  restore
```

The example timestamp is illustrative.

**Important:** This is a recovery operation, not a command to execute against your live production data directory without preparation. Restore into an isolated recovery environment first, validate the data, and plan the production cutover.

If the required WAL or timeline history is missing, PITR may not reach the requested recovery point.

## **14. The Important Configuration Advice: Enable archive_mode Early**

One of the most useful recommendations in the original article concerns initial cluster configuration.

The author recommends enabling:

```
archive_mode = on
```

when building the PostgreSQL cluster, even if the final archive destination is not ready yet.

Why?

Because:

- `archive_mode` requires a restart to change.
- `archive_command` can be changed using a reload.
- A standby can be promoted without needing an emergency configuration restart just to enable archive mode, provided it was enabled beforehand.

For example, during initial cluster setup:

```
archive_mode = on
archive_command = ''
```

Later, when the repository is configured:

```
archive_command =
'pgbackrest --stanza=prod archive-push %p'
```

Reload:

```
sudo systemctl reload postgresql-18
```

Verify:

```
SHOW archive_mode;

SHOW archive_command;
```

**Production warning:** An empty archive command is only an interim configuration. PostgreSQL will not archive WAL until a valid command is configured. Do not treat this as a completed backup strategy.

Also, avoid using `/bin/true` as a permanent workaround. It can make PostgreSQL treat archiving as successful without actually preserving WAL in the repository.

## **15. Production DBA Failover Checklist**

Use this checklist when building or reviewing a PostgreSQL HA/DR environment.

**Production readiness review**

0/16 completed

**Primary configuration**

archive_mode = on

**Primary configuration**

Valid archive_command configured

**Standby configuration**

archive_mode = on on every promotion candidate

**Standby configuration**

Valid archive_command configured for promotion

**Replication**

Streaming replication healthy

**Replication**

Replication slots reviewed and monitored

**Repository**

pgBackRest repository accessible

**Repository**

Repository permissions and storage capacity validated

**Backup**

Full backup completed successfully

**Backup**

Incremental/differential backup tested

**WAL**

WAL archiving verified

**Recovery**

Timeline history files available

**Recovery**

PITR tested across a timeline transition

**Failover**

Controlled promotion tested

**Failback**

Old primary safely rebuilt as standby

**Operations**

Monitoring and alerting configured

Reset checklist

## **16. Common Production Mistakes**

| Mistake                                     | Potential consequence                 |
| ------------------------------------------- | ------------------------------------- |
| `archive_mode=off` on standby candidates    | Archiving unavailable after promotion |
| No archive command configured               | WAL not preserved in repository       |
| Assuming replication equals backup          | No independent recovery capability    |
| Ignoring timeline history                   | Recovery across promotion may fail    |
| Multiple uncontrolled archivers             | Repository consistency concerns       |
| Never testing PITR                          | Undetected recovery failures          |
| Monitoring only backup job status           | WAL archive failures go unnoticed     |
| Deleting old WAL without retention analysis | Broken recovery chain                 |
| Using modified pgBackRest binaries          | Unsupported operational risk          |

## **17. Practical Lab: Reproduce the Scenario**

For your PostgreSQL administration lab, use three Rocky Linux virtual machines.

**Suggested lab environment**

| VM                | Role              | PostgreSQL |
| ----------------- | ----------------- | ---------- |
| Rocky-PG1         | Primary           | 18         |
| Rocky-PG2         | Standby           | 18         |
| Rocky-PG3         | Cascading standby | 18         |
| Shared repository | Backup storage    | pgBackRest |

Recommended learning exercise

### **Lab exercises**

| Day | Exercise                                      | Expected learning        |
| --- | --------------------------------------------- | ------------------------ |
| 1   | Install PostgreSQL and pgBackRest             | Initial configuration    |
| 2   | Configure PG1 and WAL archiving               | WAL management           |
| 3   | Configure PG2 streaming replication           | Physical replication     |
| 4   | Configure PG3 cascading replication           | Cascading architecture   |
| 5   | Configure pgBackRest full/incremental backups | Backup management        |
| 6   | Promote PG2                                   | Timeline creation        |
| 7   | Investigate archive_mode configuration        | Failover troubleshooting |
| 8   | Configure standby backup                      | Backup offloading        |
| 9   | Restore an older backup                       | Recovery validation      |
| 10  | Test PITR across failover                     | End-to-end DR            |

### **Important validation exercise**

Do not stop after receiving:

```
backup command end: completed successfully
```

Also test:

```
pgbackrest --stanza=demo info
```

Then restore the backup to a separate PostgreSQL instance and verify:

- Database consistency.
- Required WAL availability.
- Timeline history.
- Recovery target.
- Application-level data integrity.

This is what distinguishes a DBA who knows backup commands from one who understands database recoverability.

## **18. Final Technical Takeaways**

The original article demonstrates an important PostgreSQL HA/DR design principle.

1. `archive_mode=on` should be configured before a standby becomes a primary.
2. Changing `archive_mode` requires a PostgreSQL restart.
3. Changing `archive_command` requires only a reload.
4. Streaming replication does not replace WAL archiving.
5. pgBackRest can perform backups from a standby, subject to its configuration requirements.
6. `archive_mode=always` has specific use cases and must be carefully managed.
7. PostgreSQL timeline history is essential for recovery across promotions.
8. A successful backup after failover does not repair missing historical WAL.
9. Do not bypass pgBackRest safety checks in production.
10. Always test restores and PITR across timeline transitions.

**My recommendation as a production DBA:** Design your PostgreSQL environment around recoverability, not merely availability. A cluster that can promote a standby but cannot preserve a complete WAL recovery chain is not a fully prepared HA/DR solution.

## **19. Official Technical References**

These are useful references to keep alongside the original article.

1. [PostgreSQL – Write Ahead Log Configuration](https://www.postgresql.org/docs/current/runtime-config-wal.html)
2. [PostgreSQL – Continuous Archiving and Point-in-Time Recovery](https://www.postgresql.org/docs/current/continuous-archiving.html)
3. [PostgreSQL – Warm Standby and Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
4. [PostgreSQL – Administrative Functions](https://www.postgresql.org/docs/current/functions-admin.html)
5. [pgBackRest – User Guide](https://pgbackrest.org/user-guide.html)
6. [pgBackRest – Configuration Reference](https://pgbackrest.org/configuration.html)
7. [Original technical article by Stefan Fercot](https://pgstef.github.io/2026/09/23/pgbackrest_and_postgresql_failover_why_archive_mode_matters.html)


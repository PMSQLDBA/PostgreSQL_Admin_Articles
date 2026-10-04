# PostgreSQL + pgBackRest Failover: Why `archive_mode` Matters

Below is a consolidated technical article based on the supplied post, with the PostgreSQL and pgBackRest behavior cross-checked against the current official documentation. 
The original article was published by Stefan Fercot on September 23, 2026. ([pgstef’s blog][1]) citeturn1search0

---

## 1. Executive Summary

In a PostgreSQL HA/DR environment, **WAL archiving is not merely a backup feature**. 

It is a critical part of maintaining a recoverable history across **failovers, promotions, timeline changes, PITR, and standby rebuilds**.

A particularly dangerous configuration is:

```text
archive_mode = off
```

on a server that may later be promoted to primary.

You may still have:

* streaming replication,
* a healthy standby,
* successful `pgBackRest` backups in some circumstances,
* a backup that can apparently be restored.

But that does **not** necessarily mean you have a complete recoverable WAL history across a failover.

The critical distinction is:

> **A backup that can be restored is not automatically equivalent to a backup/repository that can support complete PITR across a timeline transition.**

The pgBackRest experiment demonstrates exactly this problem. ([pgstef’s blog][1])

---

# 2. The Environment

The original experiment uses three PostgreSQL 18 servers:

```text
              Streaming Replication
        ┌──────────────────────────────┐
        │                              │
        ▼                              ▼
      pg1 ───────────────► pg2 ───────► pg3
    Primary             Standby       Standby
```

Initially:

```text
pg1 = Primary
pg2 = Standby
pg3 = Standby
```

The configuration intentionally uses:

```text
pg1
 |
 +--> pg2
       |
       +--> pg3
```

This is a **cascading replication** topology.

The test environment uses AlmaLinux 10 and PostgreSQL 18, with pgBackRest storing the repository on shared storage. ([pgstef’s blog][1])

---

# 3. The Important PostgreSQL Parameters

Three settings are central to this discussion.

## 3.1 `archive_mode`

Example:

```ini
archive_mode = on
```

This controls whether PostgreSQL archives completed WAL segments.

Important:

> Changing `archive_mode` requires a PostgreSQL restart.

By contrast:

```ini
archive_command = '...'
```

can be changed with a reload. ([pgstef’s blog][1])

This distinction becomes extremely important during an emergency failover.

---

# 4. `archive_mode = on` vs `always`

PostgreSQL supports:

```text
archive_mode = off
archive_mode = on
archive_mode = always
```

The important distinction is what happens while the server is in recovery/standby mode.

### `archive_mode = on`

The archiver is active when the server is a primary.

When the server is a standby:

```text
archiver does not archive WAL received during recovery
```

After promotion, the newly promoted server starts archiving WAL it generates itself.

However, PostgreSQL documentation specifically warns that a standby promoted to primary does **not automatically archive all WAL or timeline-history files that it did not generate itself**. ([PostgreSQL][2])

### `archive_mode = always`

A standby can archive WAL segments it receives.

This is useful in some architectures where a standby maintains its own WAL archive.

But it introduces another issue:

```text
Primary ────────► Repository
      \
       └──── Standby ─────► Repository
```

Potentially, two machines may archive the same logical WAL.

That is exactly why pgBackRest has an `archive-mode-check`. ([pgBackRest][3])

---

# 5. Why `archive_mode = off` Looks Fine Initially

Suppose we configure:

```ini
pg1:
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

but configure the standby:

```ini
pg2:
archive_mode = off
```

Streaming replication still works:

```text
pg1 WAL
   │
   ▼
pg2 WAL receiver
   │
   ▼
pg2 replay
```

Everything appears healthy.

You can check:

```sql
SELECT *
FROM pg_stat_replication;
```

and see:

```text
state = streaming
```

Therefore the DBA may reasonably conclude:

> "Replication is working, so pg2 is ready for promotion."

That conclusion is incomplete.

---

# 6. The Hidden Problem

Consider:

```text
pg1
Timeline 1
   │
   │ WAL
   ▼
pg2
archive_mode=off
```

Now pg1 fails.

You promote pg2:

```sql
SELECT pg_promote();
```

PostgreSQL creates a new timeline:

```text
Timeline 1
     │
     │ failover
     ▼
Timeline 2
```

The promoted pg2 is now:

```text
Primary
Timeline 2
```

The critical problem:

```text
pg2 was not configured to archive while it was a standby.
```

Therefore, WAL that existed around the promotion may not be represented completely in the pgBackRest repository.

The original experiment found a gap between the archived WAL from timeline 1 and timeline 2. ([pgstef’s blog][1])

---

# 7. Understanding PostgreSQL Timelines

This is one of the most important concepts in PostgreSQL HA.

Imagine:

```text
Timeline 1

WAL 089
WAL 090
WAL 091
      │
      │ failover
      ▼
Timeline 2

WAL 093
WAL 094
WAL 095
```

The timeline ID changes after promotion.

So:

```text
00000001
```

becomes:

```text
00000002
```

PostgreSQL creates a timeline history file:

```text
00000002.history
```

The example contained:

```text
1 0/920000A0 no recovery target specified
```

This tells PostgreSQL that timeline 2 branched from timeline 1 at that location. ([pgstef’s blog][1])

---

# 8. Why the `.history` File Matters

A timeline history file allows PostgreSQL recovery to understand:

```text
Timeline 1
     │
     └── branch point
             │
             ▼
         Timeline 2
```

Without the necessary timeline history information in the archive, the recovery chain can become incomplete.

This is especially important for:

* PITR
* restoring older backups
* recovering through failover
* rebuilding old primary servers
* recovering to a point before/after a promotion
* following multiple failovers

PostgreSQL's standby documentation also recommends `recovery_target_timeline = latest` for HA configurations so a standby follows the timeline created by failover. ([PostgreSQL][4])

---

# 9. What Happened in the Experiment?

Before failover:

```text
pg1
Timeline 1
archive_mode = on
```

pg2:

```text
archive_mode = off
```

pg3:

```text
archive_mode = off
```

Then:

```text
pg1
  │
  ▼
pg2
  │
  ▼
pg3
```

pg1 was stopped.

pg2 was promoted:

```sql
SELECT pg_promote();
```

pg2 became:

```text
PRIMARY
Timeline 2
```

But:

```text
archive_mode = off
```

was still in effect.

Then:

```bash
pgbackrest --stanza=demo backup
```

failed:

```text
ERROR: [087]: archive_mode must be enabled
```

This is an intentional safety check. ([pgstef’s blog][1])

---

# 10. Why Doesn't `--no-archive-mode-check` Fix It?

This is a particularly important point.

A DBA might try:

```bash
pgbackrest \
  --stanza=demo \
  --no-archive-mode-check \
  backup
```

and expect pgBackRest to ignore the problem.

But the experiment demonstrated that it still failed:

```text
ERROR: [087]: archive_mode must be enabled
```

Why?

Because:

```text
archive-mode-check
```

and the primary's archive configuration checks are not exactly the same thing.

The pgBackRest source performs additional validation concerning the primary cluster and its archive configuration. ([pgstef’s blog][1])

---

# 11. What Does `archive-mode-check` Actually Protect Against?

pgBackRest documents:

```text
--archive-mode-check
```

as enabled by default.

It disallows:

```text
archive_mode=always
```

because WAL pushed from a standby can be logically equivalent to WAL pushed from the primary while having different checksums.

For example:

```text
Primary
   │
   ├── WAL 00000002...0095
   │
   └────────────► Repository
                    ▲
                    │
Standby ────────────┘
```

Both may attempt to archive the same logical WAL.

pgBackRest therefore warns that if archive-mode checks are disabled, **only one archiver should write to the repository through `archive-push`**. ([pgBackRest][3])

---

# 12. `backup-standby` Is a Different Feature

Another important distinction:

```ini
backup-standby = y
```

does **not** mean:

> "I don't need WAL archiving on the primary."

It means:

> "Perform the backup workload from a standby rather than the primary."

pgBackRest explicitly supports backup from a standby to reduce load on the primary. It requires the primary and standby hosts to be appropriately configured. ([pgBackRest][3])

Conceptually:

```text
                    ┌───────────────► WAL Repository
                    │
Primary ────────────┤
                    │
                    ▼
                 Standby
                    │
                    ▼
                 Backup
```

The standby-based backup still requires a sound WAL-archiving architecture.

---

# 13. The Correct Configuration

The article demonstrates the proper approach.

On promotion candidates:

```ini
archive_mode = on

archive_command =
'pgbackrest --stanza=demo archive-push %p'
```

This should be prepared **before the failover occurs**. ([pgstef’s blog][1])

For example:

```text
pg1:
archive_mode = on

pg2:
archive_mode = on

pg3:
archive_mode = on
```

Then:

```text
pg1 ──► pg2 ──► pg3
 │       │       │
 └───────┴───────┴────► pgBackRest Repository
```

With the appropriate topology and archiving design, the promotion candidate is already capable of performing the required archiving after becoming primary.

---

# 14. Why `archive_mode = on` Should Be Configured Before Promotion

Because:

```text
archive_mode
```

is a startup parameter.

Changing:

```ini
archive_mode = off
```

to:

```ini
archive_mode = on
```

requires:

```text
PostgreSQL restart
```

That is a major operational problem during a disaster.

Suppose:

```text
PRIMARY FAILS
     ↓
Standby promoted
     ↓
Oops! archive_mode=off
     ↓
Need restart to enable archive_mode
```

You have now introduced an additional outage/recovery operation immediately after failover.

The better approach is:

```text
Normal operation
      ↓
Promotion candidate already has
archive_mode=on
      ↓
Failure
      ↓
Promote
      ↓
Continue archiving
```

---

# 15. A Useful Bootstrap Technique

The article gives an excellent operational recommendation.

If you know that a cluster will eventually require WAL archiving but don't yet have the final archive destination configured, initialize the cluster with:

```ini
archive_mode = on
```

and temporarily use:

```ini
archive_command = '/bin/true'
```

or an equivalent no-op command.

Then later, replace the command with:

```ini
archive_command =
'pgbackrest --stanza=demo archive-push %p'
```

Changing `archive_command` requires only a reload.

Changing `archive_mode` requires a restart. ([pgstef’s blog][1])

**Important operational caveat:** `/bin/true` deliberately discards WAL from the perspective of archiving. It should therefore be treated only as a temporary bootstrap configuration, not as an acceptable DR configuration.

---

# 16. The Most Interesting Part: The Workaround

The experiment went further.

The author modified pgBackRest's source-code checks and managed to execute:

```text
archive_mode=off
```

on the primary while:

```text
archive_mode=always
```

was configured on the standby.

The backup completed.

The logs showed:

```text
WARN: archive_mode is off!
WARN: archive_command not checked!
```

and subsequently:

```text
check archive for segment(s)
...
backup command end: completed successfully
```

So technically:

> It is possible to produce a backup under this unconventional configuration.

But this is where the distinction between **"works in my test"** and **"is a sound production DR architecture"** becomes critical. ([pgstef’s blog][1])

---

# 17. Why a Successful Backup Is Not Enough

The resulting backup could be restored.

The test successfully restored the backup and rebuilt a standby.

That proves something useful:

```text
Backup
   ↓
Restore
   ↓
PostgreSQL starts
   ↓
Standby reconnects
```

But it does **not** prove:

```text
Historical backup
   ↓
Timeline 1 WAL
   ↓
Promotion
   ↓
Timeline 2
   ↓
PITR across promotion
```

is recoverable.

The earlier WAL gap remains.

That is the central lesson of the entire article. ([pgstef’s blog][1])

---

# 18. Backup Validity vs Recovery-Chain Completeness

These are different properties.

### Property 1 — Backup validity

Can I restore this backup?

```text
YES
```

### Property 2 — WAL completeness

Do I have every WAL segment required after the backup?

```text
MAYBE
```

### Property 3 — Timeline continuity

Can PostgreSQL navigate from the original timeline to the promoted timeline?

```text
MAYBE
```

### Property 4 — PITR across failover

Can I recover to an arbitrary valid point across the promotion?

```text
NOT GUARANTEED
```

Therefore:

> **Backup success is necessary, but not sufficient evidence of DR recoverability.**

---

# 19. pgBackRest's Archive Check

pgBackRest has another important check:

```text
archive-check
```

It verifies that WAL required to make the backup consistent is present in the archive before the backup completes. ([pgBackRest][5])

Typical backup flow:

```text
Backup START
     │
     ▼
Determine required WAL
     │
     ▼
Wait for WAL to be archived
     │
     ▼
Backup STOP
     │
     ▼
Verify required WAL
     │
     ▼
Backup complete
```

This is why pgBackRest logs include messages such as:

```text
check archive for prior segment
```

and:

```text
check archive for segment(s)
```

---

# 20. Why the WAL Repository Must Be Treated as a Recovery Graph

A common mistake is to think:

```text
Backup files = recovery
```

A better model is:

```text
                 ┌── WAL
                 │
Backup ──────────┼── Timeline history
                 │
                 └── Metadata
```

For PITR:

```text
Base Backup
     │
     ▼
Required WAL
     │
     ▼
Timeline history
     │
     ▼
Target recovery point
```

All components must be available.

---

# 21. Failover Creates a New Recovery Path

Consider:

```text
                 Timeline 1
                    │
                    │
             WAL 089-091
                    │
                    ▼
                 FAILOVER
                    │
                    ▼
                 Timeline 2
                 WAL 093-...
```

A DR system must preserve enough information to understand:

```text
Timeline 2
    |
    +--- branched from Timeline 1
```

The timeline history file is part of that information.

---

# 22. Why Streaming Replication Alone Isn't Enough

Streaming replication gives you:

```text
Primary WAL
      ↓
Network
      ↓
Standby WAL
      ↓
Replay
```

But streaming replication is not equivalent to:

```text
Long-term WAL archive
```

A standby may have WAL locally that never made it into your long-term archive.

If that standby is promoted and becomes the new primary, you have created a new timeline.

Your repository must then contain the information needed to traverse the recovery history.

PostgreSQL explicitly recommends configuring WAL archiving on standby servers that are intended to become primaries after failover. ([PostgreSQL][4])

---

# 23. Cascading Replication Makes This Even More Important

With:

```text
pg1 → pg2 → pg3
```

pg3 receives WAL from pg2.

Now imagine:

```text
pg1 = failed
pg2 = promoted
pg3 = standby
```

The new topology becomes:

```text
pg2
 │
 ▼
pg3
```

If pg2 had:

```ini
archive_mode = off
```

before promotion, the repository may not contain the complete WAL transition history.

Therefore, **every promotion candidate should be prepared as a future primary**, not merely configured as a read-only standby.

---

# 24. PostgreSQL's Own Recommendation

Current PostgreSQL documentation says that if a standby is being used for HA purposes, WAL archiving, connections, and authentication should be configured similarly to the primary because the standby will become the primary after failover. ([PostgreSQL][4])

This aligns directly with the practical lesson from the pgBackRest experiment.

---

# 25. Production Architecture

A robust architecture looks conceptually like this:

```text
                     ┌─────────────────────┐
                     │   pgBackRest Repo    │
                     │                     │
                     │  Backups + WAL      │
                     └──────────▲──────────┘
                                │
                       archive-push
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
          PRIMARY                             STANDBY
          pg1                                  pg2
              │                                   │
              │       Streaming Replication       │
              └──────────────────────────────────►
                                                   │
                                                   ▼
                                                pg3
```

All failover candidates should be designed with appropriate:

```ini
archive_mode = on
```

and:

```ini
archive_command = 'pgbackrest ... archive-push %p'
```

before the failure occurs.

---

# 26. Recommended pgBackRest Configuration Concept

A simplified configuration can look like:

```ini
[global]

repo1-path=/pgbackrest
repo1-retention-full=4

log-level-console=info
log-level-file=detail

compress-type=zst
start-fast=y

[demo]

pg1-path=/var/lib/pgsql/18/data

pg2-host=pg2
pg2-host-user=postgres
pg2-path=/var/lib/pgsql/18/data
```

If backups are intentionally performed from a standby:

```ini
backup-standby=y
```

pgBackRest supports:

```text
backup-standby=n
backup-standby=prefer
backup-standby=y
```

where `prefer` attempts the standby first and falls back to the primary if appropriate. ([pgBackRest][3])

---

# 27. PostgreSQL Configuration for Promotion Candidates

Recommended baseline:

```ini
archive_mode = on

archive_command =
'pgbackrest --stanza=demo archive-push %p'
```

For a standby:

```ini
primary_conninfo =
'user=replic_user host=pg1'

restore_command =
'pgbackrest --stanza=demo archive-get %f "%p"'
```

For an HA standby, configure the node so that after promotion it already has the required archiving capability. PostgreSQL's documentation explicitly recommends this model. ([PostgreSQL][4])

---

# 28. What About `archive_mode=always`?

`always` is a legitimate PostgreSQL capability.

It is particularly relevant when a standby needs to maintain its own WAL archive.

But it must be deliberately designed.

Example:

```text
                 Primary
                   │
                   │ WAL
                   ▼
                Standby
                   │
                   ▼
             Standby Archive
```

This is different from blindly having:

```text
Primary ─────► Repository
Standby ─────► Repository
```

with both independently pushing the same WAL.

pgBackRest therefore protects against this situation by default. ([pgBackRest][3])

---

# 29. Why Disabling `archive-mode-check` Is Dangerous

The option:

```bash
--no-archive-mode-check
```

should not be interpreted as:

> "pgBackRest is wrong; turn off the check."

It means you are taking responsibility for assumptions normally enforced by pgBackRest.

The official pgBackRest documentation explicitly warns that if the check is disabled, **only one archiver should write to the repository using `archive-push`**. ([pgBackRest][3])

Therefore:

```text
--no-archive-mode-check
```

is not a replacement for:

```text
Correct HA architecture
```

---

# 30. Why Modifying pgBackRest Source Code Is Not a Production Fix

The experiment demonstrates that removing the checks can make the scenario work.

But this introduces another problem:

```text
Official pgBackRest
        │
        ▼
Safety assumptions
        │
        X
Modified pgBackRest
        │
        ▼
Your organization owns the consequences
```

The original author correctly distinguishes between:

```text
Proof of concept
```

and:

```text
Supported production architecture
```

The modified build successfully generated and restored a backup, but it did not repair the WAL gap created during the earlier promotion. ([pgstef’s blog][1])

---

# 31. The Critical DR Principle

This is the most important takeaway:

> **Do not configure a standby merely to survive a failure. Configure it to become the next production primary.**

That means preparing:

* WAL archiving
* authentication
* replication
* pgBackRest
* monitoring
* backup
* restore
* timeline handling
* recovery configuration

before the failure.

---

# 32. Operational Checklist Before Promotion

Before promoting a standby, verify:

### PostgreSQL

```sql
SHOW archive_mode;
SHOW archive_command;
SELECT pg_is_in_recovery();
```

Expected:

```text
archive_mode = on
archive_command = pgBackRest archive-push
pg_is_in_recovery = true
```

Then check replication:

```sql
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    replay_lag
FROM pg_stat_replication;
```

---

# 33. Validate pgBackRest

Run:

```bash
pgbackrest --stanza=demo check
```

Then:

```bash
pgbackrest --stanza=demo info
```

Review:

```text
backup status
WAL archive min
WAL archive max
backup timestamps
WAL start/stop
repository status
```

Do not simply check:

```text
status = ok
```

Also verify that the expected WAL range and timelines are represented.

---

# 34. After Promotion

After:

```sql
SELECT pg_promote();
```

verify:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
f
```

Then:

```sql
SHOW archive_mode;
SHOW archive_command;
```

Force a WAL switch:

```sql
SELECT pg_switch_wal();
```

Then verify that the resulting WAL reaches the pgBackRest repository.

This provides a much stronger operational validation than merely confirming that PostgreSQL is accepting connections.

---

# 35. Post-Failover Validation

After promotion:

```text
1. PostgreSQL is primary
2. archive_mode = on
3. archive_command is configured
4. archive-push succeeds
5. WAL reaches repository
6. timeline history exists
7. pgBackRest check succeeds
8. backup succeeds
9. new standby can be rebuilt
10. restore test succeeds
```

---

# 36. Rebuilding the Old Primary

The old primary must **not simply be restarted as primary** after failover.

PostgreSQL documentation highlights the risk of having both old and new systems operating as primary, commonly referred to as a split-brain condition. A mechanism such as fencing/STONITH is required in HA designs to prevent this. ([PostgreSQL][6])

Typical workflow:

```text
OLD PRIMARY
     │
     ▼
Fence / stop
     │
     ▼
Rebuild as standby
     │
     ▼
Follow new primary
```

With pgBackRest:

```bash
pgbackrest --stanza=demo --type=standby restore
```

can be used as part of rebuilding the node, depending on the architecture.

---

# 37. The Recovery Test That Actually Matters

Don't stop at:

```bash
pgbackrest backup
```

Test:

```text
Backup
  ↓
Restore
  ↓
Start PostgreSQL
  ↓
Replay WAL
  ↓
Follow timeline
  ↓
Reach target recovery point
```

Even better:

```text
Simulate failure
      ↓
Promote standby
      ↓
Generate workload
      ↓
Archive WAL
      ↓
Take backup
      ↓
Destroy test instance
      ↓
Restore
      ↓
PITR
      ↓
Validate application data
```

That is an actual DR test.

---

# 38. Backup Testing vs DR Testing

These should be treated separately.

| Test                   | What it proves                                  |
| ---------------------- | ----------------------------------------------- |
| `pgbackrest backup`    | Backup can be created                           |
| `pgbackrest check`     | Repository/archive configuration is functioning |
| Restore                | Backup can be materialized                      |
| WAL replay             | Required WAL is usable                          |
| PITR                   | Recovery chain works                            |
| Timeline recovery      | Failover history is usable                      |
| Standby rebuild        | HA recovery process works                       |
| Application validation | Business recovery works                         |

A successful backup alone does not prove all of these.

---

# 39. Common DBA Mistakes

### Mistake 1

```ini
archive_mode = off
```

on every standby.

**Problem:** Promotion requires a restart before archiving can be enabled.

---

### Mistake 2

Assuming:

```text
Streaming replication = backup
```

It isn't.

---

### Mistake 3

Assuming:

```text
Backup completed successfully = PITR works
```

Not necessarily.

---

### Mistake 4

Disabling:

```text
archive-mode-check
```

without understanding why it exists.

---

### Mistake 5

Having multiple archivers write the same repository without a deliberate design.

pgBackRest explicitly warns about this scenario. ([pgBackRest][3])

---

### Mistake 6

Testing backup but never testing restoration.

---

### Mistake 7

Testing restoration but never testing failover.

---

### Mistake 8

Testing failover but never testing PITR across the timeline boundary.

---

# 40. Recommended Enterprise Standard

For PostgreSQL HA + pgBackRest, I would establish the following operational standard:

```text
                 ┌────────────────────────┐
                 │     pgBackRest Repo     │
                 │                        │
                 │ Full / Diff / Incr     │
                 │ WAL                    │
                 │ Timeline history       │
                 └───────────▲────────────┘
                             │
                       archive-push
                             │
       ┌─────────────────────┴──────────────────────┐
       │                                            │
       ▼                                            ▼
 PRIMARY                                      PROMOTION CANDIDATE
 archive_mode=on                              archive_mode=on
 archive_command configured                   archive_command configured
       │                                            │
       └──────────── streaming replication ────────►
                                                    │
                                                    ▼
                                               DR STANDBY
```

The exact archiving topology can vary, but **every node that may become primary must be capable of continuing the WAL archive without first requiring a restart**.

---

# 41. Key Difference: `archive_command` vs `archive_mode`

This deserves to be memorized.

| Parameter          | Purpose                                     |                                     Reload? | Restart? |
| ------------------ | ------------------------------------------- | ------------------------------------------: | -------: |
| `archive_mode`     | Enables WAL archiving capability            |                                           ❌ |    **✅** |
| `archive_command`  | Defines how WAL is archived                 |                                       **✅** |        ❌ |
| `archive_timeout`  | Forces WAL segment switching after interval |                                           ✅ |        ❌ |
| `primary_conninfo` | Standby connection to primary               | Usually dynamic/reload depending on setting |        — |
| `restore_command`  | Retrieves archived WAL during recovery      |                     Reload/config dependent |        — |

The operationally dangerous parameter is:

```text
archive_mode
```

because enabling it later requires a restart. ([pgstef’s blog][1])

---

# 42. The Core Failure Scenario

The entire problem can be summarized as:

```text
                  BEFORE FAILURE

        Timeline 1
             │
             ▼
        ┌─────────┐
        │   pg1   │
        │ Primary │
        │ archive │
        │   ON    │
        └────┬────┘
             │
             ▼
        ┌─────────┐
        │   pg2   │
        │ Standby │
        │ archive │
        │   OFF   │
        └─────────┘


                  FAILURE

        pg1 ───────X


                  AFTER PROMOTION

        Timeline 2
             │
             ▼
        ┌─────────┐
        │   pg2   │
        │ Primary │
        │ archive │
        │   OFF   │
        └─────────┘
```

Now the DBA discovers:

```text
pgBackRest backup
        ↓
archive_mode must be enabled
```

But enabling:

```ini
archive_mode = on
```

requires:

```text
restart
```

And the repository may already have a timeline/WAL gap.

---

# 43. The Correct Design

Instead:

```text
              BEFORE FAILURE

        ┌─────────┐
        │   pg1   │
        │ Primary │
        │ archive │
        │   ON    │
        └────┬────┘
             │
             ▼
        ┌─────────┐
        │   pg2   │
        │ Standby │
        │ archive │
        │   ON    │
        └────┬────┘
             │
             ▼
        ┌─────────┐
        │   pg3   │
        │ Standby │
        │ archive │
        │   ON    │
        └─────────┘
```

After failover:

```text
        pg1
         X

        pg2
      PRIMARY
    archive=ON
         │
         ▼
        pg3
       STANDBY
```

No configuration restart is required merely to enable WAL archiving on the newly promoted primary.

---

# 44. Final Technical Conclusions

### 1. `archive_mode=off` on a promotion candidate is a DR risk

A standby intended for HA failover should be prepared to archive after promotion. PostgreSQL documentation explicitly recommends configuring WAL archiving on HA standbys. ([PostgreSQL][4])

### 2. `archive_mode` and `archive_command` are fundamentally different

```text
archive_mode → restart
archive_command → reload
```

This makes pre-configuring `archive_mode=on` strategically important.

### 3. `backup-standby` doesn't eliminate the need for correct WAL archiving

It controls where the backup workload executes; it does not magically solve timeline/WAL archival requirements. ([pgBackRest][3])

### 4. `--no-archive-mode-check` isn't a magic bypass

It does not turn an incorrectly configured primary into a correctly archived primary.

### 5. Multiple WAL archivers require deliberate coordination

pgBackRest warns that WAL from standby and primary can be logically equivalent but have different checksums. ([pgBackRest][3])

### 6. A successful backup does not prove complete PITR

You must verify:

```text
Backup
+ WAL
+ Timeline history
+ Recovery path
```

### 7. Timeline history is essential after promotion

Failover creates a new timeline, and recovery must be able to understand the relationship between the old and new timelines.

### 8. Modifying pgBackRest to remove safety checks is not equivalent to fixing the architecture

The experiment successfully created and restored a backup, but the original WAL gap remained. ([pgstef’s blog][1])

### 9. DR should be tested as a recovery process

Not simply:

```text
"Did the backup succeed?"
```

but:

```text
"Can I recover the database to the required point after a real failover?"
```

---

# 45. DBA Golden Rules

Keep these as your operational checklist:

```text
┌────────────────────────────────────────────────────┐
│ PostgreSQL HA + pgBackRest Golden Rules            │
├────────────────────────────────────────────────────┤
│ 1. Promotion candidates: archive_mode = on         │
│ 2. Configure archive_command before failover       │
│ 3. Validate WAL reaches pgBackRest repository      │
│ 4. Do not confuse streaming replication with PITR  │
│ 5. Protect timeline history                        │
│ 6. Avoid uncontrolled multiple archivers            │
│ 7. Do not blindly disable archive-mode checks       │
│ 8. Test restore, not just backup                    │
│ 9. Test PITR across timeline changes                │
│ 10. Fence old primary after failover                │
│ 11. Rebuild old primary as a standby                │
│ 12. Perform regular end-to-end DR exercises         │
└────────────────────────────────────────────────────┘
```

## Bottom line

The most important lesson from the article is:

> **Prepare the standby for the role it will have after promotion, not merely for the role it has before promotion.**

For a PostgreSQL HA environment using pgBackRest, that means **WAL archiving must be part of the standby/promotion-candidate design before the failure occurs**. PostgreSQL's current HA documentation and pgBackRest's current documentation reinforce this architecture. ([PostgreSQL][4])

**Reference:** [Original pgBackRest and PostgreSQL failover article by pgstef](https://pgstef.github.io/2026/09/23/pgbackrest_and_postgresql_failover_why_archive_mode_matters.html)

**Official references:** [PostgreSQL 18 — Log-Shipping Standby Servers](https://www.postgresql.org/docs/current/warm-standby.html) · [PostgreSQL 18 — Failover](https://www.postgresql.org/docs/current/warm-standby-failover.html) · [pgBackRest Configuration Reference](https://pgbackrest.org/configuration.html) 

[1]: https://pgstef.github.io/2026/09/23/pgbackrest_and_postgresql_failover_why_archive_mode_matters.html "pgBackRest and PostgreSQL failover: why archive_mode matters | pgstef’s blog"
[2]: https://www.postgresql.org/docs/17/warm-standby.html "PostgreSQL: Documentation: 17: 26.2. Log-Shipping Standby Servers"
[3]: https://pgbackrest.org/configuration.html "pgBackRest - Configuration Reference"
[4]: https://www.postgresql.org/docs/current/warm-standby.html "PostgreSQL: Documentation: 18: 26.2. Log-Shipping Standby Servers"
[5]: https://pgbackrest.org/prior/2.45/command.html "pgBackRest Command Reference"
[6]: https://www.postgresql.org/docs/current/warm-standby-failover.html "PostgreSQL: Documentation: 18: 26.3. Failover"

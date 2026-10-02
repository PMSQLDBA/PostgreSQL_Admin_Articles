PostgreSQL: 
A table having 103 crore records. I want to keep latest 50 crore records and delete the remaining old records. Which is the best approach to delete ?

Proposed Resolution:
For a PostgreSQL table with **103 crore (~1.03 billion) rows**, we should **not run one massive `DELETE`** for the old ~53 crore rows.

The best approach depends primarily on whether the table has a suitable **date/time or monotonically increasing key** that defines “latest.”

### Recommended approach: Partition + drop old partitions

If the table can be partitioned by the retention column, this is by far the cleanest approach:

1. Identify the cutoff corresponding to the latest **50 crore rows**.
2. Partition the table by the date/time column.
3. Move/convert the data into the partitioned structure if necessary.
4. Keep partitions containing the latest 50 crore rows.
5. **DROP old partitions** rather than deleting individual rows.

`DROP TABLE`/`DETACH PARTITION` operates at the partition level, avoiding the huge row-by-row DELETE workload, WAL generation and dead-tuple accumulation associated with deleting hundreds of millions of rows.

---

## If partitioning is not currently possible

Use a **batched DELETE**, preferably driven by an indexed column.

For example, if you have:

```sql
created_at timestamp
id bigint
```

and `id`/`created_at` determines age:

```sql
CREATE INDEX CONCURRENTLY idx_table_created_at_id
ON dbo.mytable (created_at, id);
```

First determine the cutoff **before deleting anything**.

Then delete in controlled batches, e.g. 50,000–500,000 rows per transaction:

```sql
DELETE FROM dbo.mytable
WHERE id IN
(
    SELECT id
    FROM dbo.mytable
    WHERE created_at < :cutoff
    ORDER BY created_at, id
    LIMIT 100000
);
```

Repeat until:

```text
Rows deleted = ~530 million
```

### Do NOT do this

```sql
DELETE FROM mytable
WHERE created_at < :cutoff;
```

for 530 million rows in one transaction.

That can create substantial:

* WAL
* dead tuples
* table/index bloat
* replication lag
* I/O pressure
* autovacuum workload
* transaction/WAL retention pressure
* long-running transaction problems

---

# Even better: use keyset batching

If we have a suitable sequential primary key, we need to prefer a pattern like:

```sql
DELETE FROM mytable
WHERE id < :batch_end_id
  AND id >= :batch_start_id;
```

with controlled ranges.

For example:

```text
1–100,000
100,001–200,000
200,001–300,000
...
```

This is generally more predictable than repeatedly scanning the table to find the next batch.

---

# Important consideration: "latest 50 crore rows"

There is a major distinction between:

### A. Latest by timestamp

```text
ORDER BY created_at DESC
```

and

### B. Latest by ID

```text
ORDER BY id DESC
```

Don't assume the highest ID represents the newest row unless that is guaranteed by the application design.

For example:

```text
ID       created_at
1001     2026-09-01
1002     2026-09-03
1003     2026-08-29
```

In that situation, `id DESC` is not equivalent to newest data.

---

# My recommendation for 1.03 billion rows

If this is a production PostgreSQL database, I'd approach it in this order:

| Approach                                      |            530M-row cleanup | Recommendation   |
| --------------------------------------------- | --------------------------: | ---------------- |
| One huge DELETE                               |                   Very poor | ❌ Avoid          |
| Batched DELETE                                |                    Possible | ⚠️ Good fallback |
| Partition + DROP old partitions               |                   Excellent | ✅ Preferred      |
| Create new table with only 500M rows and swap | Excellent in some scenarios | ✅ Consider       |
| Archive old data + partition                  |                   Excellent | ✅ Often best     |

### If downtime is acceptable

Another powerful option is to build a new table containing only the required 500 million rows:

```sql
CREATE TABLE new_table AS
SELECT *
FROM old_table
WHERE ...latest 500 crore...;
```

Then recreate:

* indexes
* constraints
* triggers
* grants
* statistics

and perform a controlled cutover.

However, for a **1-billion-row production table**, this needs careful planning because you are effectively creating another copy of hundreds of millions of rows and need sufficient storage/I/O capacity.

---

## One critical point: VACUUM

After a massive batched DELETE, PostgreSQL does **not automatically return the physical table space to the operating system** through normal `VACUUM`.

Normal vacuum makes deleted space reusable inside the table.

If our objective is:

> "Delete 530 crore rows AND physically shrink the table"

then we need to consider:

```sql
VACUUM (FULL) mytable;
```

But **do not casually run `VACUUM FULL` on a 500+ million-row production table**. It requires a table rewrite and an aggressive lock.

Alternatives include:

* `pg_repack`
* partitioning
* table rebuild/swap

depending on your downtime and storage constraints.

---

### What we need to choose

For **1.03 billion → 500 million rows**, preferred architecture would be:

```text
Current 1.03B table
        │
        ├── Identify retention boundary
        │
        ├── Keep latest ~500M
        │
        └── Remove/archive ~530M
                 │
                 ▼
        Partition table by date
                 │
                 ▼
        Future retention
                 │
        DROP old partition
                 │
                 ▼
        Very fast cleanup
```

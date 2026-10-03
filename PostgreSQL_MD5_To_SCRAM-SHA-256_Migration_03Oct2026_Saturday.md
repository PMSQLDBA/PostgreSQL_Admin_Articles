The safest **zero-application-downtime** approach is **not** to immediately change every `pg_hba.conf` rule from `md5` to `scram-sha-256`.

PostgreSQL explicitly provides a transition mechanism: if `pg_hba.conf` says `md5` but the stored password is a SCRAM verifier, PostgreSQL automatically performs SCRAM authentication. 

That allows you to convert passwords first, validate all applications, and only then enforce `scram-sha-256`. ([PostgreSQL][1])

# PostgreSQL MD5 → SCRAM-SHA-256 Migration

## End-to-End Production Runbook With Zero Application Downtime

### Target architecture

```text
CURRENT

Application
    |
    | username/password
    v
PostgreSQL
    |
    +-- pg_hba.conf --> md5
    |
    +-- role password --> MD5


TRANSITION

Application
    |
    | SAME username/password
    v
PostgreSQL
    |
    +-- pg_hba.conf --> md5
    |                    |
    |                    +--> MD5 users authenticate with MD5
    |                    |
    |                    +--> SCRAM users automatically authenticate with SCRAM
    |
    +-- role password --> SCRAM


FINAL

Application
    |
    | username/password
    v
PostgreSQL
    |
    +-- pg_hba.conf --> scram-sha-256
    |
    +-- role password --> SCRAM-SHA-256
```

This staged approach is the key to avoiding an application outage.

---

# 1. Important PostgreSQL facts

PostgreSQL 10 introduced SCRAM-SHA-256 support. ([PostgreSQL][2])

In current PostgreSQL documentation:

* `scram-sha-256` is the preferred password authentication mechanism.
* MD5 password authentication is deprecated and is scheduled for removal in a future PostgreSQL release.
* `password_encryption` controls how a password is stored when `CREATE ROLE` or `ALTER ROLE` sets it.
* Current PostgreSQL defaults `password_encryption` to `scram-sha-256`.
* Existing MD5 passwords **do not automatically become SCRAM passwords**.
* The password must be set again to create a SCRAM verifier. ([PostgreSQL][3])

Most importantly:

> `md5` in `pg_hba.conf` can authenticate either MD5 or SCRAM depending on the stored password verifier.

That behavior was specifically provided to make migration easier. ([PostgreSQL][1])

---

# 2. What "zero downtime" actually means

For this migration, **zero application downtime is achievable** provided:

1. PostgreSQL remains running.
2. Existing database sessions are not terminated.
3. Application connection pools remain operational.
4. All clients that will create new connections support SCRAM.
5. The application continues using the same password.
6. You migrate the password verifier before enforcing SCRAM.
7. You do not change the application secret and database password simultaneously without coordination.

Changing `pg_hba.conf` requires only a configuration reload, not a PostgreSQL restart. PostgreSQL documents `pg_reload_conf()`/`pg_ctl reload` for rereading configuration. ([PostgreSQL][4])

---

# 3. The migration strategy I recommend

Use **four phases**.

| Phase                 | Password verifier | HBA             | Application |
| --------------------- | ----------------- | --------------- | ----------- |
| 0. Baseline           | MD5               | MD5             | Running     |
| 1. Compatibility      | MD5/SCRAM         | `md5`           | Running     |
| 2. Password migration | SCRAM             | `md5`           | Running     |
| 3. Enforcement        | SCRAM             | `scram-sha-256` | Running     |

The critical point is Phase 2.

When you convert a user's password to SCRAM while leaving the HBA rule as `md5`, PostgreSQL automatically negotiates SCRAM for that user.

Therefore:

```text
Existing MD5 users
        |
        v
     MD5 HBA
        |
        +---- MD5 verifier --> MD5 authentication
        |
        +---- SCRAM verifier --> SCRAM authentication
```

This gives you a **mixed-mode migration window**.

---

# 4. Pre-migration checklist

Before touching production, collect:

### PostgreSQL

```sql
SELECT version();
```

```sql
SHOW password_encryption;
```

```sql
SHOW hba_file;
```

```sql
SHOW config_file;
```

---

# 5. Inventory all login roles

Run:

```sql
SELECT
    rolname,
    rolcanlogin,
    rolreplication,
    rolvaliduntil
FROM pg_roles
WHERE rolcanlogin
ORDER BY rolname;
```

PostgreSQL documents `pg_roles` as the appropriate view for examining database roles. ([PostgreSQL][5])

---

# 6. Identify MD5 password users

This requires appropriate administrative privileges because password verifiers are protected.

For a privileged DBA session:

```sql
SELECT
    rolname,
    rolcanlogin,
    rolreplication,
    CASE
        WHEN rolpassword LIKE 'md5%' THEN 'MD5'
        WHEN rolpassword LIKE 'SCRAM-SHA-256$%' THEN 'SCRAM'
        WHEN rolpassword IS NULL THEN 'NO PASSWORD'
        ELSE 'OTHER'
    END AS password_type
FROM pg_authid
WHERE rolcanlogin
ORDER BY rolname;
```

Do **not** export or store `rolpassword`.

PostgreSQL explicitly states that `pg_authid` contains password information and therefore is not publicly readable; `pg_roles` hides the password field. PostgreSQL also documents the MD5 and SCRAM verifier formats. ([PostgreSQL][6])

---

# 7. Build your migration inventory

Create a controlled list such as:

```text
ROLE                  TYPE       APPLICATION
------------------------------------------------
app_orders            MD5        Orders API
app_payments          MD5        Payments API
app_reporting         MD5        Reporting
etl_user              MD5        ETL
monitoring            MD5        Monitoring
backup_user           MD5        Backup
replication_user      MD5        Replication
admin_user            MD5        DBA
```

Do not blindly change every role.

Separate:

* application users
* monitoring users
* ETL users
* reporting users
* backup users
* replication users
* administrative users
* integration users
* external vendor users

---

# 8. Inventory `pg_hba.conf`

First determine its location:

```sql
SHOW hba_file;
```

Then inspect it.

Look for:

```text
md5
```

For example:

```text
host    all    all    10.10.10.0/24    md5
```

Also look for:

```text
hostssl
host
local
replication
```

Do **not** assume that all connections use one HBA rule.

Remember:

> PostgreSQL uses the first matching `pg_hba.conf` rule.

There is no fall-through after a matching authentication rule fails. ([PostgreSQL][4])

This is extremely important during migration.

---

# 9. Validate HBA configuration before changing it

Run:

```sql
SELECT
    rule_number,
    file_name,
    line_number,
    type,
    database,
    user_name,
    address,
    auth_method,
    error
FROM pg_hba_file_rules
ORDER BY rule_number;
```

Look for:

```text
error IS NOT NULL
```

The PostgreSQL documentation specifically recommends `pg_hba_file_rules` for checking HBA configuration and identifying errors. ([PostgreSQL][4])

---

# 10. Inventory current application connections

Run:

```sql
SELECT
    usename,
    application_name,
    client_addr,
    state,
    count(*) AS connections
FROM pg_stat_activity
WHERE usename IS NOT NULL
GROUP BY
    usename,
    application_name,
    client_addr,
    state
ORDER BY usename, application_name;
```

`pg_stat_activity` exposes the database user, application name and client address associated with server processes. ([PostgreSQL][7])

This helps you identify:

```text
Which application
        |
        v
uses which PostgreSQL role
        |
        v
from which hosts
```

---

# 11. Do not forget connection poolers

If your architecture contains:

```text
Application
    |
    v
PgBouncer
    |
    v
PostgreSQL
```

you must validate the PgBouncer version/configuration and its authentication behavior separately.

Likewise check:

* PgBouncer
* JDBC
* Npgsql
* psycopg
* libpq
* Go PostgreSQL drivers
* Node.js PostgreSQL drivers
* PHP PostgreSQL drivers
* .NET applications
* Java applications
* Python applications
* ETL products
* monitoring systems
* backup tooling
* BI tools
* vendor applications

The PostgreSQL documentation explicitly warns that older client libraries may not support SCRAM. ([PostgreSQL][1])

---

# 12. Client compatibility is your primary gate

Before converting any password, establish:

```text
Application
    |
    +-- Driver/library version
    |
    +-- SCRAM capable?
    |
    +-- Connection pooler?
    |
    +-- Pooler SCRAM capable?
    |
    +-- Secret/password available?
```

Do **not** change the password verifier for an application until you have confirmed its client stack supports SCRAM.

PostgreSQL's official migration guidance specifically says to ensure all client libraries are new enough to support SCRAM before migrating. ([PostgreSQL][1])

---

# 13. Test SCRAM before production

Create a dedicated test role:

```sql
CREATE ROLE scram_test LOGIN PASSWORD 'temporary-test-password';
```

On current PostgreSQL, `password_encryption` defaults to SCRAM, but don't rely on an inherited/default setting during a controlled migration.

Explicitly use:

```sql
SET password_encryption = 'scram-sha-256';

ALTER ROLE scram_test
PASSWORD 'temporary-test-password';
```

Verify:

```sql
SELECT
    rolname,
    CASE
        WHEN rolpassword LIKE 'SCRAM-SHA-256$%' THEN 'SCRAM'
        WHEN rolpassword LIKE 'md5%' THEN 'MD5'
        ELSE 'OTHER'
    END AS password_type
FROM pg_authid
WHERE rolname = 'scram_test';
```

Expected:

```text
scram_test | SCRAM
```

---

# 14. Test using the actual application driver

Do not test only with `psql`.

For example:

```bash
psql "host=<host> port=5432 dbname=<database> user=scram_test password=<password>"
```

Then test with the **same driver/library used by the application**.

Example:

```text
Application
    |
    v
Actual JDBC/Npgsql/psycopg/etc.
    |
    v
PostgreSQL
```

This is much more meaningful than a successful `psql` test.

---

# 15. The zero-downtime migration starts here

Assume:

```text
Role: app_orders
Password: existing application password
Current verifier: MD5
Current HBA: md5
```

Do **not** change HBA yet.

First:

```sql
SHOW password_encryption;
```

Then explicitly establish SCRAM for the session:

```sql
SET password_encryption = 'scram-sha-256';
```

Now reset the role's password using the **same existing application password**:

```sql
ALTER ROLE app_orders
PASSWORD '<CURRENT_APPLICATION_PASSWORD>';
```

This is the critical operation.

The stored verifier changes:

```text
BEFORE

app_orders
    |
    +-- MD5 verifier


AFTER

app_orders
    |
    +-- SCRAM-SHA-256 verifier
```

The actual password used by the application has not changed.

---

# 16. Why existing application connections do not need to be restarted

Suppose the application currently has:

```text
Pool
 |
 +-- Connection 1 ---> PostgreSQL
 +-- Connection 2 ---> PostgreSQL
 +-- Connection 3 ---> PostgreSQL
 +-- Connection 4 ---> PostgreSQL
```

These connections are already authenticated.

Changing the role's stored password verifier does not require you to terminate those sessions.

Therefore:

```text
Existing sessions
        |
        +---- continue running
        |
        +---- no reconnect required
```

New connections, however, must authenticate against the new SCRAM verifier.

Because the HBA rule is still:

```text
md5
```

PostgreSQL automatically selects SCRAM when it sees the SCRAM verifier. ([PostgreSQL][1])

This is the mechanism that makes the migration possible without application downtime.

---

# 17. Immediately validate the converted role

Check:

```sql
SELECT
    rolname,
    CASE
        WHEN rolpassword LIKE 'SCRAM-SHA-256$%' THEN 'SCRAM'
        WHEN rolpassword LIKE 'md5%' THEN 'MD5'
        ELSE 'OTHER'
    END AS password_type
FROM pg_authid
WHERE rolname = 'app_orders';
```

Expected:

```text
app_orders | SCRAM
```

---

# 18. Test a NEW connection

This is critical.

Do not simply check existing application sessions.

Create a brand-new connection using the same:

```text
username
password
database
host
port
driver
```

The new connection must succeed.

Conceptually:

```text
Old connection
      |
      +---- already authenticated
      |
      +---- continues


New connection
      |
      +---- HBA = md5
      |
      +---- stored password = SCRAM
      |
      +---- PostgreSQL automatically uses SCRAM
      |
      +---- SUCCESS
```

---

# 19. Roll the application pool

If your application has a connection pool, you should perform a controlled pool recycle **only after confirming the new connection works**.

For example:

```text
Existing pool
     |
     +--- old connections
     |
     +--- old connections
     |
     +--- old connections

        ↓

new connections

        ↓

SCRAM authentication

        ↓

SUCCESS
```

A rolling application-instance restart is another option:

```text
App node 1 --> restart --> SCRAM
App node 2 --> restart --> SCRAM
App node 3 --> restart --> SCRAM
App node 4 --> restart --> SCRAM
```

Keep enough application instances serving traffic throughout.

The database itself does not need to restart.

---

# 20. Monitor while converting users

Run:

```sql
SELECT
    now(),
    usename,
    application_name,
    client_addr,
    state,
    count(*) AS connections
FROM pg_stat_activity
WHERE usename IS NOT NULL
GROUP BY
    usename,
    application_name,
    client_addr,
    state
ORDER BY usename, application_name;
```

Monitor application:

```text
Connection failures
Authentication failures
5xx errors
Timeouts
Connection pool exhaustion
Latency
Request rate
Database errors
```

---

# 21. Convert users one at a time

For example:

```sql
SET password_encryption = 'scram-sha-256';

ALTER ROLE app_orders
PASSWORD '<CURRENT_PASSWORD>';
```

Then:

```sql
SET password_encryption = 'scram-sha-256';

ALTER ROLE app_payments
PASSWORD '<CURRENT_PASSWORD>';
```

Then:

```sql
SET password_encryption = 'scram-sha-256';

ALTER ROLE app_reporting
PASSWORD '<CURRENT_PASSWORD>';
```

And so on.

This gives you:

```text
User 1 --> SCRAM
User 2 --> MD5
User 3 --> SCRAM
User 4 --> MD5
User 5 --> MD5
```

while:

```text
pg_hba.conf = md5
```

continues supporting both types.

---

# 22. Verify migration progress

Use:

```sql
SELECT
    CASE
        WHEN rolpassword LIKE 'SCRAM-SHA-256$%' THEN 'SCRAM'
        WHEN rolpassword LIKE 'md5%' THEN 'MD5'
        WHEN rolpassword IS NULL THEN 'NO PASSWORD'
        ELSE 'OTHER'
    END AS password_type,
    count(*)
FROM pg_authid
WHERE rolcanlogin
GROUP BY 1
ORDER BY 1;
```

You want:

```text
SCRAM       100%
MD5           0%
```

for password-authenticated login roles that are in scope.

---

# 23. Important: don't forget service accounts

Commonly missed accounts include:

```text
Monitoring
Backup
ETL
Batch jobs
Schedulers
Reporting
BI
Data integration
Replication tooling
Foreign data wrappers
Administrative scripts
Cron jobs
Shell scripts
Vendor applications
DR tooling
```

A migration can appear successful during business hours and fail at 2 AM because a forgotten batch process opens a new connection.

---

# 24. Check replication separately

If you use physical streaming replication, inspect:

```sql
SELECT
    usename,
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

Also inspect the relevant:

```text
pg_hba.conf
```

replication rules.

Do not change a replication authentication rule simply because application users have been migrated.

Treat:

```text
Application authentication
```

and

```text
Replication authentication
```

as separate migration workstreams.

---

# 25. Check backup tooling separately

For example:

```text
pgBackRest
pg_dump
pg_basebackup
custom scripts
monitoring agents
```

If they authenticate with PostgreSQL passwords, validate their connection behavior before changing those role verifiers.

---

# 26. Final HBA migration

Once:

```text
ALL required clients support SCRAM
AND
ALL in-scope passwords are SCRAM
AND
NEW connections have been tested
AND
connection pools have been validated
```

change:

```text
host    all    all    <network>    md5
```

to:

```text
host    all    all    <network>    scram-sha-256
```

Prefer more-specific HBA entries rather than unnecessarily broad rules.

For example:

```text
hostssl  appdb  app_orders  10.10.20.0/24  scram-sha-256
hostssl  appdb  app_payments 10.10.21.0/24 scram-sha-256
```

rather than blindly changing every rule.

---

# 27. Validate HBA before reload

Run:

```sql
SELECT
    rule_number,
    file_name,
    line_number,
    type,
    database,
    user_name,
    address,
    auth_method,
    error
FROM pg_hba_file_rules
ORDER BY rule_number;
```

You want:

```text
error
-----
NULL
```

for every intended rule.

---

# 28. Reload — don't restart PostgreSQL

After the HBA change:

```sql
SELECT pg_reload_conf();
```

Expected:

```text
t
```

PostgreSQL documents configuration reload via `pg_reload_conf()` or `pg_ctl reload`; a restart is not required for `pg_hba.conf` changes. ([PostgreSQL][4])

This is an important part of the zero-downtime design.

---

# 29. Verify the active HBA

Run:

```sql
SELECT
    rule_number,
    file_name,
    line_number,
    type,
    database,
    user_name,
    address,
    auth_method,
    error
FROM pg_hba_file_rules
ORDER BY rule_number;
```

You should now see:

```text
scram-sha-256
```

for the migrated rules.

---

# 30. Test a fresh application connection again

This is mandatory.

Do:

```text
Application
      |
      v
New connection
      |
      v
HBA = scram-sha-256
      |
      v
SCRAM verifier
      |
      v
SUCCESS
```

Then recycle one application instance/pod at a time if necessary.

---

# 31. Validate all applications

Your final validation matrix should look like:

| Component    | New connection | SCRAM | Result |
| ------------ | -------------: | ----: | ------ |
| Orders API   |            Yes |   Yes | PASS   |
| Payments API |            Yes |   Yes | PASS   |
| Reporting    |            Yes |   Yes | PASS   |
| ETL          |            Yes |   Yes | PASS   |
| Monitoring   |            Yes |   Yes | PASS   |
| Backup       |            Yes |   Yes | PASS   |
| Batch jobs   |            Yes |   Yes | PASS   |
| Admin tools  |            Yes |   Yes | PASS   |

---

# 32. Detect remaining MD5 users

Run:

```sql
SELECT
    rolname
FROM pg_authid
WHERE rolcanlogin
  AND rolpassword LIKE 'md5%';
```

Expected:

```text
0 rows
```

if you intend to migrate all login roles.

---

# 33. Detect remaining MD5 HBA entries

On Linux:

```bash
grep -nE '(^|[[:space:]])md5([[:space:]]|$)' "$PGDATA/pg_hba.conf"
```

But if you use included HBA files, don't rely solely on that command.

Use:

```sql
SELECT
    file_name,
    line_number,
    type,
    database,
    user_name,
    address,
    auth_method,
    error
FROM pg_hba_file_rules
WHERE auth_method = 'md5';
```

Expected:

```text
0 rows
```

for a fully enforced migration.

---

# 34. Recommended final state

Your final authentication configuration should look approximately like:

```text
                    PostgreSQL
                         |
            +------------+------------+
            |                         |
        Applications              Administrators
            |                         |
            v                         v
      SCRAM-SHA-256             SCRAM-SHA-256
            |                         |
            +------------+------------+
                         |
                  SCRAM verifier
```

No MD5 password verifiers should remain.

---

# 35. What NOT to do

## ❌ Do not do this first

```text
Change:

md5

to:

scram-sha-256
```

before converting passwords.

You could immediately break users whose password is still stored as MD5.

PostgreSQL documents that an MD5-stored password cannot authenticate using an HBA method that requires SCRAM. ([PostgreSQL][1])

---

## ❌ Don't restart PostgreSQL

You don't need:

```bash
systemctl restart postgresql
```

for this migration.

Use:

```sql
SELECT pg_reload_conf();
```

for the HBA change. ([PostgreSQL][8])

---

## ❌ Don't terminate all connections

Avoid:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
...
```

as part of the normal migration.

There is no requirement to kill existing sessions.

---

## ❌ Don't change the application password unnecessarily

The cleanest migration is:

```text
Existing application password
              |
              v
      Generate SCRAM verifier
              |
              v
      Same application password
```

rather than:

```text
Old password
      |
      v
New password
      |
      v
Update secret
      |
      v
Restart application
```

The latter introduces unnecessary operational risk.

---

# 36. If you MUST rotate the password too

Sometimes security policy requires:

```text
MD5 → SCRAM
```

**and**

```text
Password rotation
```

simultaneously.

Then use a coordinated secret rotation.

For example:

```text
              OLD
        PostgreSQL password
               |
               |
        Application running
               |
               v
        Create NEW secret
               |
               v
       Update PostgreSQL
               |
               v
       Update application
               |
               v
     Rolling application restart
```

However, this is more complicated than simply changing the verifier while retaining the existing secret.

For zero downtime, use:

```text
SCRAM conversion
```

and

```text
password rotation
```

as separate controlled changes whenever your security requirements permit it.

---

# 37. Rollback strategy

This is one of the most important parts.

Suppose:

```text
app_orders
```

was converted:

```text
MD5 → SCRAM
```

but an unexpected legacy client cannot authenticate.

If HBA is still:

```text
md5
```

you have a very useful rollback mechanism.

You can temporarily restore the user's MD5 verifier by resetting its password with:

```sql
SET password_encryption = 'md5';

ALTER ROLE app_orders
PASSWORD '<CURRENT_PASSWORD>';
```

Then verify:

```sql
SELECT
    rolname,
    CASE
        WHEN rolpassword LIKE 'md5%' THEN 'MD5'
        WHEN rolpassword LIKE 'SCRAM-SHA-256$%' THEN 'SCRAM'
        ELSE 'OTHER'
    END
FROM pg_authid
WHERE rolname = 'app_orders';
```

Expected:

```text
app_orders | MD5
```

This should be treated as a **temporary emergency rollback**, because PostgreSQL considers MD5 password support deprecated. ([PostgreSQL][3])

---

# 38. Rollback after HBA enforcement

Once you have changed:

```text
md5
```

to:

```text
scram-sha-256
```

you cannot simply restore an MD5 password and expect authentication to work.

Therefore the rollback order should be:

```text
SCRAM HBA
     |
     v
Restore HBA = md5
     |
     v
Reload
     |
     v
Restore affected password verifier if necessary
```

Do **not** reverse the password first while the HBA requires SCRAM.

---

# 39. Production change sequence

Here is the exact sequence I would use in a production environment.

### Change Window — T-7 days

```text
Inventory applications
Inventory PostgreSQL roles
Inventory HBA rules
Inventory connection pools
Inventory client libraries
Inventory service accounts
Inventory backup/monitoring/ETL
```

---

### T-5 days

Validate SCRAM support:

```text
Application
    |
    +-- Driver
    +-- Pool
    +-- Pooler
    +-- Monitoring
    +-- Backup
    +-- ETL
```

---

### T-3 days

Create test role:

```sql
CREATE ROLE scram_test LOGIN PASSWORD 'test-password';
```

Test with actual client stack.

---

### T-1 day

Back up:

```text
postgresql.conf
pg_hba.conf
included HBA files
application connection configuration
secret-manager configuration
```

Also capture:

```sql
SHOW hba_file;
SHOW config_file;
SHOW password_encryption;
```

---

# 40. Production execution

### Step 1

Keep:

```text
pg_hba.conf = md5
```

---

### Step 2

Convert first application:

```sql
SET password_encryption = 'scram-sha-256';

ALTER ROLE app_orders
PASSWORD '<existing-password>';
```

---

### Step 3

Verify:

```sql
SELECT
    rolname,
    CASE
        WHEN rolpassword LIKE 'SCRAM-SHA-256$%' THEN 'SCRAM'
        WHEN rolpassword LIKE 'md5%' THEN 'MD5'
        ELSE 'OTHER'
    END
FROM pg_authid
WHERE rolname = 'app_orders';
```

---

### Step 4

Open a new connection.

---

### Step 5

Recycle one application instance/pod.

---

### Step 6

Verify application health.

---

### Step 7

Continue to the next application.

---

### Step 8

Eventually:

```text
MD5 roles = 0
```

---

### Step 9

Change HBA:

```text
md5
```

to:

```text
scram-sha-256
```

---

### Step 10

Validate:

```sql
SELECT *
FROM pg_hba_file_rules
WHERE error IS NOT NULL;
```

Expected:

```text
0 rows
```

---

### Step 11

Reload:

```sql
SELECT pg_reload_conf();
```

---

### Step 12

Test fresh connections.

---

### Step 13

Monitor.

---

### Step 14

Declare migration complete only after:

```text
MD5 password verifiers = 0
MD5 HBA rules = 0
SCRAM connections successful
Application errors = normal
Connection pool healthy
Batch jobs successful
Monitoring healthy
Backup healthy
Replication healthy
```

---

# 41. The most important operational concept

The entire migration can be visualized as:

```text
                    PHASE 1
              Existing Production
                       |
                       |
              +--------v---------+
              |   HBA = md5     |
              +--------+---------+
                       |
               +-------+-------+
               |               |
            MD5 role       SCRAM role
               |               |
              MD5            SCRAM
               |               |
               +-------+-------+
                       |
                       v
                  Application
                       |
                       v

                    PHASE 2
                Password migration
                       |
                       v
              +------------------+
              |    HBA = md5     |
              +--------+---------+
                       |
               +-------+-------+
               |               |
            MD5 role       SCRAM role
               |               |
              MD5            SCRAM
                               ^
                               |
                         migrate users
                               |
                               v

                    PHASE 3
                 All users SCRAM
                       |
                       v
              +------------------+
              |    HBA = md5     |
              +------------------+
                       |
                  SCRAM works
                       |
                       v

                    PHASE 4
                Enforce SCRAM
                       |
                       v
              +------------------+
              | HBA=scram-sha-256|
              +------------------+
                       |
                       v
                  ALL SCRAM
```

That is the **PostgreSQL-native migration path**.

---

# 42. Why this approach provides zero application downtime

There are three separate things involved:

```text
1. Existing sessions
2. Stored password verifier
3. HBA authentication method
```

They do not have to change simultaneously.

You deliberately separate them:

```text
Existing sessions
       |
       | continue
       v

Password verifier
       |
       | MD5 → SCRAM
       v

HBA remains md5
       |
       | allows transition
       v

New sessions authenticate using SCRAM
       |
       v

Only after validation:
HBA md5 → scram-sha-256
```

This staged conversion is directly aligned with PostgreSQL's documented MD5-to-SCRAM migration guidance. ([PostgreSQL][1])

---

# 43. Final production checklist

### Discovery

* [ ] PostgreSQL version identified
* [ ] `pg_hba.conf` location identified
* [ ] All HBA files identified
* [ ] All login roles inventoried
* [ ] MD5 roles identified
* [ ] Application-to-role mapping completed
* [ ] Client versions identified
* [ ] Connection pools identified
* [ ] Poolers identified
* [ ] ETL identified
* [ ] Monitoring identified
* [ ] Backup tooling identified
* [ ] Replication authentication identified

### Compatibility

* [ ] All application drivers support SCRAM
* [ ] Pooler supports SCRAM
* [ ] Monitoring supports SCRAM
* [ ] Backup tooling supports SCRAM
* [ ] ETL supports SCRAM
* [ ] Vendor applications validated

### Migration

* [ ] HBA remains `md5`
* [ ] `password_encryption='scram-sha-256'`
* [ ] Passwords converted individually
* [ ] SCRAM verifier confirmed
* [ ] New connection tested
* [ ] Connection pool tested
* [ ] Application health verified
* [ ] No downtime observed

### Enforcement

* [ ] Zero MD5 password verifiers
* [ ] Zero required MD5 HBA rules
* [ ] `pg_hba_file_rules` clean
* [ ] HBA changed to `scram-sha-256`
* [ ] `pg_reload_conf()` successful
* [ ] New connections tested
* [ ] Applications healthy
* [ ] Batch jobs healthy
* [ ] Monitoring healthy
* [ ] Backup healthy
* [ ] Replication healthy

### Security

* [ ] MD5 removed
* [ ] SCRAM enforced
* [ ] TLS/`hostssl` requirements reviewed
* [ ] Passwords not logged
* [ ] SCRAM verifiers not exported
* [ ] Secrets handled through approved secret management

---

## Bottom line

**Do not perform a direct `md5 → scram-sha-256` HBA switch as the first step.**

The production-safe sequence is:

```text
              CURRENT
                 |
                 v
        HBA = md5
        Password = MD5
                 |
                 |
       Verify client compatibility
                 |
                 v
       password_encryption =
          scram-sha-256
                 |
                 v
       ALTER ROLE ... PASSWORD
                 |
                 v
        Password = SCRAM
                 |
                 |
          HBA STILL = md5
                 |
                 v
       PostgreSQL automatically
       negotiates SCRAM
                 |
                 v
       Validate new connections
                 |
                 v
       Migrate remaining users
                 |
                 v
       ALL passwords = SCRAM
                 |
                 v
       Change HBA to
       scram-sha-256
                 |
                 v
          pg_reload_conf()
                 |
                 v
       Validate applications
                 |
                 v
             COMPLETE
```

This approach avoids a PostgreSQL restart, avoids terminating existing application sessions, provides a mixed MD5/SCRAM transition period, and follows PostgreSQL's documented migration mechanism. PostgreSQL itself recommends setting `password_encryption` to `scram-sha-256`, resetting users' passwords, and then changing the HBA authentication method to `scram-sha-256` after client compatibility has been established. ([PostgreSQL][1])

**Official PostgreSQL references:** 

[PostgreSQL 18 — Password Authentication](https://www.postgresql.org/docs/current/auth-password.html) 
[PostgreSQL 18 — pg_hba.conf](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html) 
[PostgreSQL 18 — Connections and Authentication](https://www.postgresql.org/docs/current/runtime-config-connection.html) 
[PostgreSQL 18 — pg_roles](https://www.postgresql.org/docs/current/view-pg-roles.html)

[1]: https://www.postgresql.org/docs/17/auth-password.html "PostgreSQL: Documentation: 17: 20.5. Password Authentication"
[2]: https://www.postgresql.org/docs/10/release-10.html "PostgreSQL: Documentation: 10: E.24. Release 10"
[3]: https://www.postgresql.org/docs/current/runtime-config-connection.html "PostgreSQL: Documentation: 18: 19.3. Connections and Authentication"
[4]: https://www.postgresql.org/docs/current/auth-pg-hba-conf.html "PostgreSQL: Documentation: 18: 20.1. The pg_hba.conf File"
[5]: https://www.postgresql.org/docs/current/database-roles.html "PostgreSQL: Documentation: 18: 21.1. Database Roles"
[6]: https://www.postgresql.org/docs/12/catalog-pg-authid.html "PostgreSQL: Documentation: 12: 51.8. pg_authid"
[7]: https://www.postgresql.org/docs/15/monitoring-stats.html "PostgreSQL: Documentation: 15: 28.2. The Cumulative Statistics System"
[8]: https://www.postgresql.org/docs/current/config-setting.html "PostgreSQL: Documentation: 18: 19.1. Setting Parameters"

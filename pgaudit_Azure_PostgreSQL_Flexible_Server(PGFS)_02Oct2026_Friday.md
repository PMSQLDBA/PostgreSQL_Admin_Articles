Query: 
Implement pgaudit on Azure PostgreSQL Flexible Server (PGFS). 
Documentation or implementation guides covering the configuration, prerequisites, recommended settings, and best practices for enabling pgaudit on Azure PGFS. 

===============================================================================
Information:

**pgaudit is supported on Azure Database for PostgreSQL Flexible Server (PGFS)**, and Microsoft now has a fairly complete official implementation path. 

Used Microsoft's documentation as the baseline rather than relying on generic PostgreSQL `pgaudit` guides because Azure PGFS has some platform-specific differences. ([Microsoft Learn][1])

### Recommended implementation path

**1. Verify your PostgreSQL version and pgaudit availability**

Microsoft currently documents pgaudit for PostgreSQL **11–18**, with the packaged pgaudit version varying by PostgreSQL major version. pgaudit requires `shared_preload_libraries`. ([Microsoft Learn][2])

[Microsoft — pgaudit extension/version matrix](https://learn.microsoft.com/en-us/azure/postgresql/extensions/concepts-extensions-versions)

---

**2. Allowlist/load the extension**

Azure's extension process is:

1. Add/allowlist `pgaudit`.
2. Add it to `shared_preload_libraries` where required.
3. Restart if the parameter requires it.
4. Create the extension in the target database.

Microsoft specifically documents this process for Flexible Server. ([Microsoft Learn][3])

[Microsoft — Create extensions in PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/extensions/how-to-create-extensions)

---

**3. Configure `pgaudit` parameters**

The important parameters include:

```text
pgaudit.log
pgaudit.log_catalog
pgaudit.log_client
pgaudit.log_level
pgaudit.log_parameter
pgaudit.log_parameter_max_size
pgaudit.log_relation
pgaudit.log_rows
pgaudit.log_statement
pgaudit.log_statement_once
pgaudit.role
```

Microsoft's PGFS parameter documentation confirms these are supported. ([Microsoft Learn][4])

For example, don't automatically start production with:

```text
pgaudit.log = ALL
```

Although Microsoft uses `ALL` as a quick-start example, production configuration should normally be driven by the audit/compliance requirement because audit volume can become substantial. Microsoft also warns that high-volume logging can create performance overhead. ([Microsoft Learn][1])

A more targeted starting point might be:

```text
pgaudit.log = DDL,ROLE,WRITE
```

and add:

```text
READ
```

only where SELECT auditing is actually required.

---

### 4. Send the logs outside the PostgreSQL server

This is **very important on Azure**.

pgaudit writes audit events into the PostgreSQL logging subsystem. You then use Azure Monitor Diagnostic Settings to send PostgreSQL logs to destinations such as:

* Log Analytics
* Azure Storage
* Event Hubs
* supported partner solutions

Microsoft recommends resource-specific tables when using Log Analytics. ([Microsoft Learn][5])

The architecture is essentially:

```text
Application
     |
     v
Azure PostgreSQL Flexible Server
     |
     +--> pgaudit
     |
     v
PostgreSQL Server Logs
     |
     v
Azure Monitor Diagnostic Settings
     |
     +--> Log Analytics
     +--> Storage
     +--> Event Hub
     |
     v
Security / SIEM / Compliance
```

[Microsoft — Configure and access PostgreSQL Flexible Server logs](https://learn.microsoft.com/en-us/azure/postgresql/monitor/how-to-configure-and-access-logs)

---

### 5. Validate the audit trail

Microsoft provides a useful starting KQL query:

```kusto
AzureDiagnostics
| where Resource =~ "<flexible-server-name>"
| where Category == "PostgreSQLLogs"
| where TimeGenerated > ago(1d)
| where Message contains "AUDIT:"
```

([Microsoft Learn][1])

If using resource-specific tables, Microsoft now documents the `PGSQLServerLogs` table for PostgreSQL server logs. ([Microsoft Learn][6])

---

### 6. Microsoft Entra ID needs special consideration

If your PGFS environment uses Microsoft Entra authentication, there is an additional auditing consideration: **group-based Entra authentication can make identifying the individual user behind a group login more difficult**.

Microsoft has a dedicated article for this and recommends modifying `log_line_prefix` to include user/database/session information, followed by KQL correlation. ([Microsoft Learn][7])

For example:

```text
%t-%c-user=%u,db=%d,session=%c,sess_time=%s
```

This is particularly useful for environments where:

```text
Application/User
       |
       v
Microsoft Entra ID
       |
       v
Entra Group
       |
       v
PostgreSQL Flexible Server
```

---

## My recommended production baseline

For a typical enterprise PGFS deployment, I would structure the implementation like this:

| Area                  | Recommendation                                                        |
| --------------------- | --------------------------------------------------------------------- |
| Extension             | `pgaudit`                                                             |
| Preload               | Configure `shared_preload_libraries`                                  |
| Audit scope           | Start targeted, don't blindly use `ALL`                               |
| DDL                   | **Enable**                                                            |
| ROLE/security changes | **Enable**                                                            |
| WRITE                 | Enable where modification auditing is required                        |
| READ                  | Enable only where regulatory/security requirements justify the volume |
| `log_client`          | Keep **OFF**                                                          |
| `log_level`           | Don't use in production                                               |
| `log_catalog`         | Consider `OFF` to reduce noise                                        |
| `log_parameter`       | Assess carefully because parameters can contain sensitive data        |
| Log destination       | Azure Monitor / Log Analytics                                         |
| Retention             | Based on compliance requirement                                       |
| SIEM                  | Forward relevant audit events if required                             |
| Alerting              | Create KQL-based security/compliance alerts                           |
| Monitoring            | Monitor audit-log volume and ingestion cost                           |
| Testing               | Validate audit events before production rollout                       |
| Major upgrades        | Revalidate pgaudit configuration after upgrade                        |

One particularly important Azure-specific point: **after a major-version upgrade, Microsoft states that pgaudit is dropped and recreated, but custom pgaudit configuration such as `pgaudit.log` isn't automatically preserved.** Your upgrade runbook should therefore explicitly validate/reapply the pgaudit configuration. ([Microsoft Learn][1])

---

## Official documentation I recommend

**Primary implementation guide**

[Audit logging in Azure Database for PostgreSQL Flexible Server — Microsoft Learn](https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit)

**Extension installation**

[Create extensions in Azure Database for PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/extensions/how-to-create-extensions)

**Supported pgaudit versions**

[PostgreSQL extensions and modules by version — Microsoft Learn](https://learn.microsoft.com/en-us/azure/postgresql/extensions/concepts-extensions-versions)

**PGFS pgaudit parameters**

[Azure PostgreSQL Flexible Server parameters](https://learn.microsoft.com/en-us/azure/postgresql/server-parameters/concepts-server-parameters)

**Log Analytics / Diagnostic Settings**

[Configure and access logs — Azure PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/monitor/how-to-configure-and-access-logs)

**Microsoft Entra + pgaudit**

[Audit logging for Microsoft Entra ID principals](https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit-entra)

**Azure Policy**

Microsoft even provides an Azure Policy definition to audit whether **pgaudit is enabled across PostgreSQL Flexible Server instances**, which is useful for enterprise governance. ([Microsoft Learn][8])

[Azure Policy support for PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/security/security-azure-policy)

### If you're implementing this in an enterprise environment

Refer **production-ready PGFS pgaudit implementation runbook** covering:

**Prerequisites → Azure Portal/CLI configuration → exact parameter values → `CREATE EXTENSION` → Log Analytics Diagnostic Settings → KQL queries → alert rules → security/compliance recommendations → performance impact → sensitive-data considerations → Entra ID auditing → HA/failover → major-version upgrade checklist → rollback procedure.**

That would be more useful than simply enabling `pgaudit.log = ALL`.

[1]: https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit "Audit Logging in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[2]: https://learn.microsoft.com/fi-fi/azure/postgresql/extensions/concepts-extensions-versions "List of Extensions and Modules by Name in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/azure/postgresql/extensions/how-to-create-extensions "Create Extensions in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[4]: https://learn.microsoft.com/en-us/azure/postgresql/server-parameters/concepts-server-parameters "Parameters in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/azure/postgresql/monitor/how-to-configure-and-access-logs "Configure and Access Logs in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[6]: https://learn.microsoft.com/en-us/azure/postgresql/monitor/concepts-monitoring "Monitor Using Metrics and Logs in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[7]: https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit-entra "Audit Logging for Microsoft Entra ID Principals in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[8]: https://learn.microsoft.com/en-us/azure/postgresql/security/security-azure-policy "Azure Policy Support in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"

============================================================================================================================

Below is a **production-oriented implementation runbook for pgaudit on Azure Database for PostgreSQL Flexible Server (PGFS)**. I

Aligned the configuration with current Microsoft documentation and separated **mandatory controls, recommended controls, and workload-dependent controls**.

# Azure PostgreSQL Flexible Server — pgaudit Production Implementation Runbook

## 1. Target architecture

A good enterprise implementation should look like this:

```text
                         ┌──────────────────────┐
                         │ Application / Users  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────┐
                    │ Azure PostgreSQL Flexible   │
                    │ Server                      │
                    │                             │
                    │  PostgreSQL                 │
                    │       │                     │
                    │       ▼                     │
                    │     pgaudit                 │
                    │       │                     │
                    │       ▼                     │
                    │ PostgreSQL Server Logs       │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
                     Azure Monitor Diagnostic
                           Settings
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
              Log Analytics    Event Hub       Storage
                    │
                    ▼
             Security / SIEM
                    │
             ┌──────┴───────┐
             ▼              ▼
          Alerts         Investigation
```

Microsoft explicitly supports sending Flexible Server logs to Azure Monitor destinations, including Log Analytics, Storage and Event Hubs. 

For new implementations, Microsoft recommends **resource-specific tables** rather than the legacy `AzureDiagnostics` collection mode. ([Microsoft Learn][1])

---

# 2. Prerequisites

Before touching production, establish the following.

### Azure

* Azure Database for PostgreSQL Flexible Server
* Appropriate RBAC permissions
* Log Analytics workspace
* Azure Monitor Diagnostic Settings permission
* Optional SIEM integration
* Optional Azure Policy governance

### PostgreSQL

Confirm:

```sql
SELECT version();

SHOW shared_preload_libraries;

SHOW pgaudit.log;

SHOW log_line_prefix;
```

Also check:

```sql
SELECT *
FROM pg_available_extensions
WHERE name = 'pgaudit';
```

The currently documented PGFS parameter matrix supports `pgaudit` parameters across PostgreSQL 11–18. ([Microsoft Learn][2])

---

# 3. Decide what you actually need to audit

This is the most important design decision.

Don't start production with:

```text
pgaudit.log = ALL
```

just because it is easy.

Microsoft documents `ALL` as a quick-start configuration, but production audit scope should be based on the organization's security/compliance requirements. Excessive audit logging can generate substantial log volume. ([Microsoft Learn][1])

Think about audit requirements in these categories:

| Activity                  | Typical requirement          |
| ------------------------- | ---------------------------- |
| DDL                       | Usually audit                |
| CREATE/DROP/ALTER objects | Usually audit                |
| CREATE/ALTER ROLE         | Audit                        |
| GRANT/REVOKE              | Audit                        |
| INSERT                    | Often audit                  |
| UPDATE                    | Often audit                  |
| DELETE                    | Often audit                  |
| TRUNCATE                  | Audit                        |
| COPY                      | Audit where relevant         |
| SELECT                    | Requirement-dependent        |
| System/catalog queries    | Usually reduce noise         |
| Function execution        | Requirement-dependent        |
| Row counts                | Requirement-dependent        |
| SQL parameters            | Sensitive-data consideration |

---

# 4. Recommended starting configuration

For many enterprise systems, I'd begin with:

```text
pgaudit.log = DDL,ROLE,WRITE
```

This provides a reasonable baseline for:

```text
DDL
+
Security/role changes
+
Data modifications
```

Then evaluate whether:

```text
READ
```

is actually required.

If regulatory requirements require SELECT auditing, use:

```text
pgaudit.log = DDL,ROLE,READ,WRITE
```

### Important Azure-specific difference

Microsoft documents that on Azure Flexible Server you **cannot use the `-` subtraction shortcut** in `pgaudit.log`; explicitly specify the required statement classes. ([Microsoft Learn][1])

---

# 5. Enable pgaudit

The process is:

```text
Allowlist extension
       ↓
shared_preload_libraries
       ↓
Restart if required
       ↓
CREATE EXTENSION
       ↓
Configure pgaudit parameters
       ↓
Generate test activity
       ↓
Verify AUDIT records
```

Microsoft's current process specifically requires the extension to be **allowlisted, loaded, and created in the database where it will be used**. ([Microsoft Learn][1])

---

# 6. Configure `shared_preload_libraries`

Check:

```sql
SHOW shared_preload_libraries;
```

It should contain:

```text
pgaudit
```

If other extensions already exist, **do not replace them accidentally**.

For example:

```text
pg_stat_statements,pgaudit
```

rather than changing it to:

```text
pgaudit
```

and unintentionally removing another required library.

This is an important production change-control item.

---

# 7. Create the extension

Connect to the target database:

```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;
```

Verify:

```sql
SELECT extname,
       extversion
FROM pg_extension
WHERE extname = 'pgaudit';
```

Expected:

```text
extname | extversion
--------+-----------
pgaudit | x.x.x
```

---

# 8. Core pgaudit parameters

These are the parameters I would review individually:

```text
pgaudit.log
pgaudit.log_catalog
pgaudit.log_client
pgaudit.log_level
pgaudit.log_parameter
pgaudit.log_parameter_max_size
pgaudit.log_relation
pgaudit.log_rows
pgaudit.log_statement
pgaudit.log_statement_once
pgaudit.role
```

Microsoft's PGFS parameter documentation confirms the availability and behavior of these parameters. ([Microsoft Learn][2])

---

# 9. Recommended production parameter profile

A reasonable baseline is:

```text
pgaudit.log                  = DDL,ROLE,WRITE
pgaudit.log_catalog          = off
pgaudit.log_client           = off
pgaudit.log_parameter       = off
pgaudit.log_relation         = off
pgaudit.log_rows             = off
pgaudit.log_statement        = on
pgaudit.log_statement_once   = on
```

But **do not treat these values as universal compliance requirements**.

They should be validated against the application and regulatory requirements.

---

# 10. Why `log_catalog = off`?

Applications and tools frequently execute catalog queries.

For example:

```sql
SELECT *
FROM pg_catalog.pg_tables;
```

or queries against:

```text
pg_class
pg_attribute
pg_namespace
pg_type
...
```

Logging all of these can create considerable noise.

Microsoft specifically notes that disabling `pgaudit.log_catalog` reduces noise from tools such as `psql` and PgAdmin. ([Microsoft Learn][2])

Therefore:

```text
pgaudit.log_catalog = off
```

is generally sensible unless you have a specific requirement for catalog auditing.

---

# 11. Keep `pgaudit.log_client` OFF

Use:

```text
pgaudit.log_client = off
```

This is important.

If enabled, audit messages are also sent back to the client.

Microsoft explicitly recommends generally leaving this disabled. ([Microsoft Learn][1])

You want:

```text
Database
   ↓
Server log
   ↓
Azure Monitor
   ↓
Security platform
```

rather than unnecessarily injecting audit information into application/client sessions.

---

# 12. Be extremely careful with `pgaudit.log_parameter`

This deserves special attention.

If enabled:

```text
pgaudit.log_parameter = on
```

audit records can contain parameter values.

For example, an application may execute:

```sql
UPDATE customer
SET email = $1
WHERE customer_id = $2;
```

The audit log could potentially contain values that should not be retained in your logging environment.

This creates a security and compliance question:

```text
Database security
        +
Log security
        +
PII/PCI/PHI exposure
```

Therefore, my default production recommendation is:

```text
pgaudit.log_parameter = off
```

unless there is a documented requirement for parameter auditing.

If it must be enabled, carefully evaluate:

```text
pgaudit.log_parameter_max_size
```

Microsoft documents this parameter specifically for limiting parameter size. ([Microsoft Learn][2])

---

# 13. Be careful with SQL statement logging

There are two related concepts:

```text
pgaudit.log_statement
```

and

```text
pgaudit.log_statement_once
```

The latter controls whether statement text/parameters are logged once or repeatedly for statement/substatement combinations.

For high-volume environments:

```text
pgaudit.log_statement_once = on
```

can help reduce unnecessary duplication.

Microsoft notes that disabling it reduces verbosity but can make correlating audit entries to their originating statement more difficult. ([Microsoft Learn][2])

---

# 14. Critical password logging consideration

This is one of the most important Microsoft-specific warnings.

If you configure PostgreSQL:

```text
log_statement = DDL
```

or:

```text
log_statement = ALL
```

then commands such as:

```sql
CREATE ROLE appuser PASSWORD 'secret';
```

can result in the password appearing in PostgreSQL logs.

That is obviously undesirable.

Microsoft specifically documents this behavior and recommends using pgaudit's:

```text
pgaudit.log = DDL
```

and, where role auditing is required:

```text
pgaudit.log = ROLE
```

because `ROLE` auditing redacts the password while recording the role operation. ([Microsoft Learn][1])

This is a strong reason **not to blindly configure PostgreSQL `log_statement = ALL`**.

---

# 15. Azure Monitor configuration

Go to:

```text
Azure Portal
   ↓
PostgreSQL Flexible Server
   ↓
Monitoring
   ↓
Diagnostic settings
   ↓
Add diagnostic setting
```

Select:

```text
Destination:
Log Analytics workspace
```

Use:

```text
Destination table:
Resource specific
```

Microsoft currently recommends resource-specific tables. ([Microsoft Learn][3])

---

# 16. Enable PostgreSQL Server Logs

Select:

```text
PostgreSQL Server logs
```

This is the log stream through which your pgaudit records become available for centralized analysis.

The resource-specific table is:

```text
PGSQLServerLogs
```

Microsoft's current Azure Monitor documentation identifies `PGSQLServerLogs` as the resource-specific table for Flexible Server PostgreSQL server logs. ([Microsoft Learn][4])

---

# 17. First validation query

After generating some database activity:

```sql
CREATE TABLE audit_test
(
    id integer,
    description text
);

INSERT INTO audit_test VALUES
(1, 'test');

UPDATE audit_test
SET description = 'updated'
WHERE id = 1;

DELETE FROM audit_test
WHERE id = 1;

DROP TABLE audit_test;
```

Then query:

```kusto
PGSQLServerLogs
| where LogicalServerName == "your-flexible-server"
| where TimeGenerated > ago(1h)
| where Message contains "AUDIT:"
| order by TimeGenerated desc
```

Microsoft confirms `PGSQLServerLogs` as the resource-specific table and provides server-log query examples. ([Microsoft Learn][3])

---

# 18. If you're still using AzureDiagnostics

Some existing environments use:

```text
AzureDiagnostics
```

Then:

```kusto
AzureDiagnostics
| where LogicalServerName_s == "your-flexible-server"
| where Category == "PostgreSQLLogs"
| where TimeGenerated > ago(1h)
| where Message contains "AUDIT:"
| order by TimeGenerated desc
```

Microsoft's pgaudit documentation provides this pattern. ([Microsoft Learn][1])

For **new implementations**, however, I'd use:

```text
PGSQLServerLogs
```

rather than building new monitoring around `AzureDiagnostics`.

---

# 19. Build useful KQL queries

## All audit events

```kusto
PGSQLServerLogs
| where LogicalServerName == "your-flexible-server"
| where Message contains "AUDIT:"
| project TimeGenerated,
          LogicalServerName,
          Message,
          ProcessId,
          Statement
| order by TimeGenerated desc
```

---

## DDL activity

```kusto
PGSQLServerLogs
| where LogicalServerName == "your-flexible-server"
| where Message contains "AUDIT:"
| where Message contains "DDL"
| project TimeGenerated, Message, Statement
| order by TimeGenerated desc
```

---

## Role/security activity

```kusto
PGSQLServerLogs
| where LogicalServerName == "your-flexible-server"
| where Message contains "AUDIT:"
| where Message contains "ROLE"
| project TimeGenerated, Message, Statement
| order by TimeGenerated desc
```

---

## DELETE activity

```kusto
PGSQLServerLogs
| where LogicalServerName == "your-flexible-server"
| where Message contains "AUDIT:"
| where Message contains "DELETE"
| project TimeGenerated, Message, Statement
| order by TimeGenerated desc
```

---

# 20. Important: don't build alerts by blindly matching SQL text

For example, this is simplistic:

```kusto
Message contains "DROP"
```

It can generate false positives.

Your security monitoring should ideally parse:

```text
timestamp
database
user
session
command class
object
schema
table
statement
client
```

and then apply security logic.

---

# 21. Microsoft Entra ID users

This becomes particularly important if your PGFS environment uses Microsoft Entra authentication.

Microsoft documents a specific challenge with **Entra group authentication**:

```text
User
  ↓
Entra ID
  ↓
Entra Group
  ↓
PostgreSQL
```

The PostgreSQL username can represent the group rather than the individual person.

Microsoft's documented solution is to configure:

```text
log_line_prefix
```

to include:

```text
%t-%c-user=%u,db=%d,session=%c,sess_time=%s
```

This allows the audit events to be correlated with the Entra authentication records. ([Microsoft Learn][5])

This is highly recommended if you have Entra group-based database access.

---

# 22. Entra audit correlation architecture

Use:

```text
Entra authentication event
        │
        │ Session ID
        ▼
PostgreSQL connection
        │
        │ Session ID
        ▼
pgaudit event
```

This lets your security team answer:

> Which actual Entra user performed this database operation?

rather than only:

> Which PostgreSQL group executed it?

Microsoft provides a dedicated KQL approach for this correlation. ([Microsoft Learn][5])

---

# 23. Object-level auditing

There are two broad approaches:

### Session auditing

```text
pgaudit.log
```

Example:

```text
DDL,ROLE,WRITE
```

### Object auditing

Using:

```text
pgaudit.role
```

Object auditing allows you to target particular relations rather than auditing every object.

For example, you may have:

```text
finance
customer
payment
employee
```

and want enhanced auditing for only:

```text
payment
employee
```

This can be significantly more appropriate than globally enabling extremely verbose auditing.

Microsoft describes `pgaudit.role` as the master role for object audit logging. ([Microsoft Learn][2])

---

# 24. When should you enable READ?

This is probably the biggest architectural decision.

Suppose an application executes:

```sql
SELECT *
FROM customer;
```

100,000 times per hour.

With:

```text
pgaudit.log = READ
```

your audit volume can become enormous.

Therefore:

### General operational auditing

```text
DDL,ROLE,WRITE
```

may be sufficient.

### Strong data-access auditing

```text
DDL,ROLE,READ,WRITE
```

may be required.

### Highly sensitive tables

Consider **object-level auditing** instead of auditing every SELECT in the entire database.

---

# 25. `pgaudit.log_relation`

This controls whether session auditing creates separate entries for each relation referenced in SELECT/DML statements.

Microsoft describes it as a shortcut for exhaustive logging without using object auditing. ([Microsoft Learn][2])

I would **not enable this by default** in a high-throughput system.

Why?

Because:

```text
one SQL statement
        ↓
multiple referenced tables
        ↓
multiple audit entries
```

can dramatically increase audit volume.

---

# 26. `pgaudit.log_rows`

This adds rows retrieved or affected to audit information.

Useful for some compliance requirements.

But it can also increase audit volume substantially.

Therefore:

```text
Default:
OFF
```

unless your audit requirement explicitly requires row counts.

---

# 27. Performance considerations

pgaudit isn't free.

The impact comes from:

```text
SQL execution
     ↓
Audit event generation
     ↓
PostgreSQL logging
     ↓
Azure Monitor ingestion
     ↓
Log Analytics
```

The impact can be especially noticeable when enabling:

```text
READ
log_relation
log_rows
log_parameter
```

on a high-throughput workload.

Therefore test:

```text
TPS
CPU
I/O
latency
log volume
Log Analytics ingestion
```

before and after activation.

---

# 28. Production rollout strategy

I recommend this rollout.

### Phase 1 — Development

```text
pgaudit.log = ALL
```

temporarily if necessary to understand what is generated.

Do **not** automatically carry that configuration to production.

---

### Phase 2 — UAT

Test:

```text
DDL
ROLE
INSERT
UPDATE
DELETE
TRUNCATE
COPY
SELECT
```

Measure audit volume.

---

### Phase 3 — Production baseline

Start with something like:

```text
pgaudit.log = DDL,ROLE,WRITE
```

and:

```text
pgaudit.log_catalog = off
pgaudit.log_client = off
pgaudit.log_parameter = off
pgaudit.log_relation = off
pgaudit.log_rows = off
pgaudit.log_statement_once = on
```

Then expand based on evidence.

---

# 29. Security alerting

Create alerts around events such as:

```text
DROP DATABASE
DROP SCHEMA
DROP TABLE
ALTER TABLE
TRUNCATE
CREATE ROLE
ALTER ROLE
DROP ROLE
GRANT
REVOKE
DELETE
```

For especially sensitive systems:

```text
SELECT from sensitive tables
```

should also be considered.

---

# 30. Example suspicious DDL query

Conceptually:

```kusto
PGSQLServerLogs
| where Message contains "AUDIT:"
| where Message has_any (
    "DROP TABLE",
    "DROP SCHEMA",
    "ALTER TABLE",
    "TRUNCATE"
)
| project TimeGenerated,
          LogicalServerName,
          Message,
          Statement
| order by TimeGenerated desc
```

Use this as a starting point and tune it against your actual audit message format.

---

# 31. Example privileged-account monitoring

Create a watchlist of privileged roles:

```text
postgres
dba_admin
security_admin
application_owner
```

Then monitor changes performed by those identities.

The important principle is:

```text
Privileged identity
       +
Sensitive operation
       =
High-priority security event
```

---

# 32. Don't forget Azure Activity Log

pgaudit covers **database activity**.

It does not replace Azure Activity Log.

For example:

```text
Azure Activity Log
        ↓
Server creation
Server deletion
Scaling
Configuration changes
Networking changes
RBAC
Azure resource operations
```

while:

```text
pgaudit
        ↓
SQL/database operations
```

These are complementary controls.

Microsoft explicitly distinguishes database auditing from Azure resource-level activity logging. ([Microsoft Learn][1])

---

# 33. Governance with Azure Policy

For a large Azure estate, don't rely on DBAs manually checking every server.

Microsoft provides an Azure Policy definition:

> **Auditing with PgAudit should be enabled for PostgreSQL flexible server instances**

with an `AuditIfNotExists` effect. Microsoft also documents policies for deploying diagnostic settings to Log Analytics. ([Microsoft Learn][6])

This gives you:

```text
Azure Policy
      ↓
Identify PGFS without pgaudit
      ↓
Compliance dashboard
      ↓
Remediation
```

That's particularly useful for enterprise environments.

---

# 34. Major-version upgrade — CRITICAL

Put this in your PostgreSQL upgrade runbook.

Microsoft explicitly states that during a major-version upgrade:

```text
pgaudit extension
      ↓
dropped
      ↓
upgrade
      ↓
recreated
```

But the custom pgaudit configuration isn't automatically preserved. ([Microsoft Learn][1])

Therefore after:

```text
PG 15 → PG 16
PG 16 → PG 17
PG 17 → PG 18
```

validate:

```sql
SHOW pgaudit.log;
SHOW pgaudit.log_catalog;
SHOW pgaudit.log_client;
SHOW pgaudit.log_parameter;
SHOW pgaudit.log_relation;
SHOW pgaudit.log_rows;
SHOW pgaudit.log_statement;
SHOW pgaudit.log_statement_once;
SHOW pgaudit.role;
```

Then execute a test transaction and confirm:

```text
AUDIT:
```

events are appearing in Log Analytics.

---

# 35. HA/failover validation

After configuring pgaudit, don't stop at the primary.

Test:

```text
Primary
   ↓
Failover
   ↓
New primary
   ↓
Application reconnect
   ↓
Audit event
   ↓
Log Analytics
```

Verify:

```text
pgaudit extension
pgaudit parameters
log_line_prefix
Diagnostic Settings
audit events
Entra correlation
```

The important objective is:

> **Audit continuity after failover.**

---

# 36. Backup/restore consideration

Remember that:

```text
pgaudit configuration
```

is not equivalent to:

```text
database data
```

Therefore don't assume a database restore automatically reproduces your entire Azure server-level audit configuration.

Your recovery runbook should explicitly contain:

```text
Create/restore PGFS
       ↓
Validate server parameters
       ↓
Validate pgaudit
       ↓
Validate Diagnostic Settings
       ↓
Validate Log Analytics
       ↓
Generate test audit event
```

---

# 37. Rollback procedure

If audit volume or performance becomes unacceptable:

First reduce the scope:

```text
DDL,ROLE,READ,WRITE
```

→

```text
DDL,ROLE,WRITE
```

rather than immediately removing pgaudit.

If necessary:

```text
Remove pgaudit from shared_preload_libraries
```

and restart according to the Azure parameter requirements.

Then:

```sql
DROP EXTENSION pgaudit;
```

only if your change plan explicitly requires complete removal.

For rollback, I would generally preserve the extension and reduce the audit scope first.

---

# 38. Production checklist

### Azure

```text
☐ PGFS identified
☐ PostgreSQL version documented
☐ pgaudit supported
☐ Log Analytics workspace available
☐ Diagnostic Settings configured
☐ Resource-specific table selected
☐ PostgreSQL Server Logs enabled
☐ RBAC validated
☐ Azure Policy considered
```

### PostgreSQL

```text
☐ shared_preload_libraries reviewed
☐ pgaudit allowlisted
☐ pgaudit extension created
☐ pgaudit.log configured
☐ pgaudit.log_catalog reviewed
☐ pgaudit.log_client OFF
☐ pgaudit.log_parameter reviewed
☐ pgaudit.log_relation reviewed
☐ pgaudit.log_rows reviewed
☐ pgaudit.log_statement reviewed
☐ pgaudit.log_statement_once reviewed
☐ pgaudit.role reviewed
```

### Security

```text
☐ DDL audited
☐ ROLE changes audited
☐ GRANT/REVOKE monitored
☐ Destructive operations monitored
☐ Sensitive-table access reviewed
☐ PII/password exposure assessed
☐ Entra authentication correlation configured
☐ SIEM integration evaluated
```

### Operations

```text
☐ Audit volume baseline established
☐ CPU baseline established
☐ Storage/logging impact measured
☐ Log Analytics ingestion cost reviewed
☐ Alert rules tested
☐ Failover tested
☐ Restore procedure tested
☐ Major upgrade runbook updated
```

---

# 39. My recommended enterprise baseline

If I were designing the initial production standard for a normal enterprise PGFS environment, I would start around:

```text
pgaudit.log                = DDL,ROLE,WRITE
pgaudit.log_catalog        = off
pgaudit.log_client         = off
pgaudit.log_parameter      = off
pgaudit.log_relation       = off
pgaudit.log_rows           = off
pgaudit.log_statement      = on
pgaudit.log_statement_once = on
```

Then:

```text
                    ┌───────────────────┐
                    │ DDL / ROLE / WRITE│
                    └─────────┬─────────┘
                              │
                              ▼
                       Initial baseline
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
          Compliance need?            Performance OK?
                 │                         │
                 ▼                         ▼
           Add READ where          Increase scope
           actually required        selectively
                 │
                 ▼
       Sensitive tables?
                 │
                 ▼
          Object auditing
```

This is much safer operationally than turning on every pgaudit feature globally.

---

## 40. Official Microsoft references

**Primary pgaudit implementation guide**

[Microsoft — Audit logging in Azure Database for PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit)

**Diagnostic Settings / Log Analytics**

[Microsoft — Configure and access PostgreSQL Flexible Server logs](https://learn.microsoft.com/en-us/azure/postgresql/monitor/how-to-configure-and-access-logs)

**PGFS server parameters**

[Microsoft — Parameters in Azure Database for PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/server-parameters/concepts-server-parameters)

**Entra ID + pgaudit**

[Microsoft — Audit logging for Microsoft Entra ID principals](https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit-entra)

**Azure Monitor `PGSQLServerLogs`**

[Microsoft — PGSQLServerLogs table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/pgsqlserverlogs)

**Azure Policy**

[Microsoft — Azure Policy support for PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/security/security-azure-policy)

---

### One important recommendation

For a **real production implementation**, do not stop at enabling pgaudit. 

**Ready-to-execute Azure PGFS implementation package** containing:

1. **Azure CLI script** to configure the server
2. **SQL validation script**
3. **Production parameter baseline**
4. **Diagnostic Settings CLI script**
5. **10–15 production KQL queries**
6. **Azure Monitor alert rules**
7. **Entra ID audit correlation KQL**
8. **Sensitive-data/password audit safeguards**
9. **Performance/load-testing checklist**
10. **HA/failover validation**
11. **PG 15 → 16 → 17 → 18 upgrade checklist**
12. **Rollback script**
13. **DBA handover/runbook**
14. **Security/compliance mapping**

That would turn this from a documentation exercise into an **implementation-ready PGFS pgaudit standard**.

[1]: https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit "Audit Logging in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/azure/postgresql/server-parameters/concepts-server-parameters "Parameters in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/azure/postgresql/monitor/how-to-configure-and-access-logs "Configure and Access Logs in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[4]: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/pgsqlserverlogs "Azure Monitor Logs reference - PGSQLServerLogs - Azure Monitor | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit-entra "Audit Logging for Microsoft Entra ID Principals in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[6]: https://learn.microsoft.com/en-us/azure/postgresql/security/security-azure-policy "Azure Policy Support in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"

============================================================================================================

**Implementation-ready Azure PostgreSQL Flexible Server (PGFS) pgaudit package**. 

Important point: Microsoft’s current documentation says `pgaudit` is allowlisted through `azure.extensions`; the extension is then created in the target database. ([Microsoft Learn][1])

# Azure PostgreSQL Flexible Server — Production pgaudit Standard

## 1. Production architecture

```text
                     Azure PostgreSQL Flexible Server
                     ┌──────────────────────────────┐
                     │                              │
Users / Apps ───────►│ PostgreSQL                   │
                     │       │                      │
                     │       ▼                      │
                     │    pgaudit                   │
                     │       │                      │
                     │       ▼                      │
                     │ PostgreSQL Server Logs        │
                     └──────────────┬───────────────┘
                                    │
                                    ▼
                       Azure Monitor Diagnostic
                              Settings
                                    │
                                    ▼
                           Log Analytics
                         PGSQLServerLogs
                                    │
                     ┌──────────────┼───────────────┐
                     ▼              ▼               ▼
                  KQL/Alerts      SIEM          Retention
```

Microsoft identifies `PostgreSQLLogs` as the diagnostic category and `PGSQLServerLogs` as its resource-specific Log Analytics table. Resource-specific tables are the recommended collection mode. ([Microsoft Learn][2])

---

# 2. Variables

Use these variables in the scripts below.

```bash
export SUBSCRIPTION_ID="<subscription-id>"
export RESOURCE_GROUP="<resource-group>"
export SERVER_NAME="<pgfs-server-name>"
export DATABASE_NAME="<database-name>"
export LOG_ANALYTICS_WORKSPACE="<workspace-name>"
export LOG_ANALYTICS_RG="<workspace-resource-group>"
```

Authenticate:

```bash
az login

az account set \
  --subscription "$SUBSCRIPTION_ID"
```

Verify:

```bash
az account show \
  --query "{subscription:id,name:name}" \
  -o table
```

---

# 3. Pre-implementation inventory

Run:

```bash
az postgres flexible-server show \
  --resource-group "$RESOURCE_GROUP" \
  --name "$SERVER_NAME" \
  -o json
```

Capture:

```text
PostgreSQL version
SKU
Availability zone / HA
Storage
Backup retention
Network configuration
Authentication configuration
Maintenance window
```

Then inspect the current parameters:

```bash
az postgres flexible-server parameter show \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --name azure.extensions
```

Check:

```bash
az postgres flexible-server parameter show \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --name shared_preload_libraries
```

And:

```bash
az postgres flexible-server parameter show \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --name pgaudit.log
```

---

# 4. SQL pre-check

Connect to the database and execute:

```sql
SELECT version();

SHOW azure.extensions;

SHOW shared_preload_libraries;

SELECT name,
       default_version,
       installed_version
FROM pg_available_extensions
WHERE name = 'pgaudit';

SELECT extname,
       extversion
FROM pg_extension
WHERE extname = 'pgaudit';
```

Also capture the current audit/logging configuration:

```sql
SELECT name,
       setting,
       unit,
       context,
       source
FROM pg_settings
WHERE name IN
(
    'shared_preload_libraries',
    'log_statement',
    'log_line_prefix',
    'pgaudit.log',
    'pgaudit.log_catalog',
    'pgaudit.log_client',
    'pgaudit.log_level',
    'pgaudit.log_parameter',
    'pgaudit.log_parameter_max_size',
    'pgaudit.log_relation',
    'pgaudit.log_rows',
    'pgaudit.log_statement',
    'pgaudit.log_statement_once',
    'pgaudit.role'
)
ORDER BY name;
```

**Save this output before changing anything.**

---

# 5. Allowlist pgaudit

Microsoft's current Flexible Server procedure is:

```text
azure.extensions
        ↓
add pgaudit
        ↓
CREATE EXTENSION pgaudit
```

The Azure CLI syntax is:

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --name azure.extensions \
  --value "pgaudit"
```

However, **do not blindly replace an existing allowlist**.

If you currently have:

```text
pg_stat_statements,pgcrypto,vector
```

you need:

```text
pg_stat_statements,pgcrypto,vector,pgaudit
```

not simply:

```text
pgaudit
```

Microsoft documents `azure.extensions` as the allowlist parameter. ([Microsoft Learn][1])

---

# 6. Configure `shared_preload_libraries`

Check first:

```sql
SHOW shared_preload_libraries;
```

If pgaudit isn't already loaded, configure it using the server parameter mechanism.

For example:

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --name shared_preload_libraries \
  --value "pgaudit"
```

Again, **preserve existing libraries**.

For example:

```text
pg_stat_statements,pgaudit
```

rather than accidentally replacing:

```text
pg_stat_statements
```

with:

```text
pgaudit
```

Microsoft's extension documentation notes that extensions requiring preload libraries must also be added to the relevant preload configuration. ([Microsoft Learn][3])

---

# 7. Restart planning

Because `shared_preload_libraries` is a startup-level PostgreSQL setting, schedule the required server restart/change window if Azure indicates one is necessary.

Before restart:

```text
☐ Application team notified
☐ Connection pool behavior checked
☐ Maintenance window approved
☐ HA/failover state checked
☐ Current parameters exported
☐ Rollback parameters captured
```

---

# 8. Create pgaudit

After the required configuration is active:

```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;
```

Validate:

```sql
SELECT extname,
       extversion
FROM pg_extension
WHERE extname = 'pgaudit';
```

Also:

```sql
SHOW shared_preload_libraries;
```

You should see:

```text
pgaudit
```

or:

```text
pg_stat_statements,pgaudit
```

depending on the server.

---

# 9. Production baseline

For a general enterprise workload, I recommend starting with:

```text
pgaudit.log                  = DDL,ROLE,WRITE
pgaudit.log_catalog          = off
pgaudit.log_client           = off
pgaudit.log_parameter        = off
pgaudit.log_relation         = off
pgaudit.log_rows             = off
pgaudit.log_statement        = on
pgaudit.log_statement_once   = on
```

**This is a baseline, not a compliance certification.**

The exact setting must be mapped against the organization's audit requirement.

Microsoft specifically recommends testing the parameters and verifying the resulting behavior. It also notes that `pgaudit.log_client` should generally remain disabled. ([Microsoft Learn][4])

---

# 10. Apply the baseline with Azure CLI

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --source user-override \
  --name pgaudit.log \
  --value "DDL,ROLE,WRITE"
```

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --source user-override \
  --name pgaudit.log_catalog \
  --value "off"
```

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --source user-override \
  --name pgaudit.log_client \
  --value "off"
```

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --source user-override \
  --name pgaudit.log_parameter \
  --value "off"
```

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --source user-override \
  --name pgaudit.log_relation \
  --value "off"
```

For PG14+:

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --source user-override \
  --name pgaudit.log_rows \
  --value "off"
```

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --source user-override \
  --name pgaudit.log_statement \
  --value "on"
```

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --source user-override \
  --name pgaudit.log_statement_once \
  --value "on"
```

Microsoft documents the `az postgres flexible-server parameter set --source user-override` syntax for pgaudit configuration. ([Microsoft Learn][4])

---

# 11. Verify parameters

```sql
SELECT name,
       setting,
       source,
       pending_restart
FROM pg_settings
WHERE name LIKE 'pgaudit%'
ORDER BY name;
```

Expected baseline:

```text
pgaudit.log                  DDL,ROLE,WRITE
pgaudit.log_catalog          off
pgaudit.log_client           off
pgaudit.log_parameter        off
pgaudit.log_relation         off
pgaudit.log_rows             off
pgaudit.log_statement        on
pgaudit.log_statement_once   on
```

---

# 12. Important: don't enable `log_statement = ALL`

This is one of the most important safeguards.

Do **not** use:

```text
log_statement = ALL
```

as your primary audit mechanism.

Microsoft specifically warns that PostgreSQL's normal `log_statement = DDL` or `ALL` can put passwords from commands such as:

```sql
CREATE ROLE appuser PASSWORD 'secret';
```

into the PostgreSQL logs.

Microsoft's documented alternative is to use pgaudit, including `ROLE`, which redacts the password. ([Microsoft Learn][4])

For example:

```text
pgaudit.log = DDL,ROLE,WRITE
```

is preferable to using PostgreSQL's generic statement logging to capture role operations.

---

# 13. Configure Azure Diagnostic Settings

First discover available categories:

```bash
az monitor diagnostic-settings categories list \
  --resource-group "$RESOURCE_GROUP" \
  --resource "$SERVER_NAME" \
  --resource-namespace Microsoft.DBforPostgreSQL \
  --resource-type flexibleServers \
  --query "value[].{name:name,categoryType:categoryType}" \
  -o table
```

Microsoft currently documents `PostgreSQLLogs` as the server-log category. ([Microsoft Learn][2])

---

# 14. Find Log Analytics workspace ID

```bash
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group "$LOG_ANALYTICS_RG" \
  --workspace-name "$LOG_ANALYTICS_WORKSPACE" \
  --query id \
  -o tsv)

echo "$WORKSPACE_ID"
```

---

# 15. Create Diagnostic Setting

For audit logging, use the audit category group if available in your environment.

First inspect:

```bash
az monitor diagnostic-settings categories list \
  --resource-group "$RESOURCE_GROUP" \
  --resource "$SERVER_NAME" \
  --resource-namespace Microsoft.DBforPostgreSQL \
  --resource-type flexibleServers \
  -o json
```

Then create the diagnostic setting.

For a server-log-only implementation:

```bash
az monitor diagnostic-settings create \
  --name "pgfs-audit-to-law" \
  --resource "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.DBforPostgreSQL/flexibleServers/$SERVER_NAME" \
  --workspace "$WORKSPACE_ID" \
  --export-to-resource-specific true \
  --logs '[{"category":"PostgreSQLLogs","enabled":true}]'
```

Microsoft documents `--export-to-resource-specific true` for resource-specific collection. ([Microsoft Learn][2])

---

# 16. Validate Diagnostic Settings

```bash
az monitor diagnostic-settings list \
  --resource "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.DBforPostgreSQL/flexibleServers/$SERVER_NAME" \
  -o json
```

Check:

```text
PostgreSQLLogs = enabled
Log Analytics = correct workspace
resource-specific = enabled
```

---

# 17. Generate test audit events

Connect to the target database.

Create a controlled test table:

```sql
CREATE TABLE pgaudit_validation
(
    id          integer,
    description text
);
```

Write:

```sql
INSERT INTO pgaudit_validation
VALUES (1, 'pgaudit-test');
```

Update:

```sql
UPDATE pgaudit_validation
SET description = 'pgaudit-test-updated'
WHERE id = 1;
```

Delete:

```sql
DELETE FROM pgaudit_validation
WHERE id = 1;
```

DDL:

```sql
ALTER TABLE pgaudit_validation
ADD COLUMN test_timestamp timestamptz;
```

Finally:

```sql
DROP TABLE pgaudit_validation;
```

---

# 18. First KQL validation

Use:

```kusto
PGSQLServerLogs
| where LogicalServerName =~ "<SERVER_NAME>"
| where TimeGenerated > ago(1h)
| where Message contains "AUDIT:"
| project
    TimeGenerated,
    LogicalServerName,
    Message
| order by TimeGenerated desc
```

Microsoft documents `PGSQLServerLogs` as the resource-specific table for PostgreSQL server logs. ([Microsoft Learn][5])

---

# 19. Audit volume baseline

Before enabling `READ`, measure your audit volume.

```kusto
PGSQLServerLogs
| where LogicalServerName =~ "<SERVER_NAME>"
| where TimeGenerated > ago(24h)
| where Message contains "AUDIT:"
| summarize
    AuditEvents=count(),
    ApproxMB=sum(_BilledSize) / 1024 / 1024
```

Track this for several business cycles.

Record:

```text
Audit events/day
Approximate ingestion volume
CPU
Storage
Application latency
Log Analytics cost
```

Azure explicitly notes that external log collection, ingestion, retention and querying incur costs. ([Microsoft Learn][6])

---

# 20. DDL monitoring query

```kusto
PGSQLServerLogs
| where LogicalServerName =~ "<SERVER_NAME>"
| where Message contains "AUDIT:"
| where Message contains "DDL"
| project
    TimeGenerated,
    LogicalServerName,
    Message
| order by TimeGenerated desc
```

---

# 21. Role/security activity

```kusto
PGSQLServerLogs
| where LogicalServerName =~ "<SERVER_NAME>"
| where Message contains "AUDIT:"
| where Message contains "ROLE"
| project
    TimeGenerated,
    LogicalServerName,
    Message
| order by TimeGenerated desc
```

---

# 22. Destructive activity

```kusto
PGSQLServerLogs
| where LogicalServerName =~ "<SERVER_NAME>"
| where Message contains "AUDIT:"
| where Message has_any (
    "DROP",
    "TRUNCATE",
    "DELETE"
)
| project
    TimeGenerated,
    LogicalServerName,
    Message
| order by TimeGenerated desc
```

Treat this as a starting query; tune it against your actual audit-message format before using it for alerting.

---

# 23. GRANT / REVOKE monitoring

```kusto
PGSQLServerLogs
| where LogicalServerName =~ "<SERVER_NAME>"
| where Message contains "AUDIT:"
| where Message has_any (
    "GRANT",
    "REVOKE"
)
| project
    TimeGenerated,
    LogicalServerName,
    Message
| order by TimeGenerated desc
```

---

# 24. Sensitive operation alert

Create an Azure Monitor log-search alert for events such as:

```text
DROP TABLE
DROP SCHEMA
TRUNCATE
ALTER ROLE
DROP ROLE
GRANT
REVOKE
```

Recommended alert metadata:

```text
Severity: High
Evaluation: 5 minutes
Frequency: 5 minutes
Window: 5–10 minutes
Action Group: DBA/Security
```

Do not alert on every normal application `INSERT`/`UPDATE` unless there is a specific requirement.

---

# 25. READ auditing

If your security requirement says:

> "We must know who read customer data."

then:

```text
pgaudit.log = DDL,ROLE,READ,WRITE
```

becomes relevant.

But don't enable it without measuring the resulting volume.

The progression should be:

```text
DDL,ROLE,WRITE
        │
        ▼
Measure volume
        │
        ▼
Business/security requirement
        │
        ▼
Add READ
        │
        ▼
Measure again
```

---

# 26. Sensitive-table auditing

For particularly sensitive objects, object auditing can be preferable to global `READ`.

For example:

```text
customer
employee
payment
credit_card
medical_record
```

Conceptually:

```text
Global:
DDL,ROLE,WRITE

Sensitive objects:
READ + WRITE
```

`pgaudit.role` is the mechanism Microsoft exposes for object audit logging. ([Microsoft Learn][7])

---

# 27. Example object-audit design

Create an audit role:

```sql
CREATE ROLE audit_sensitive;
```

Then configure:

```text
pgaudit.role = audit_sensitive
```

Grant that audit role to the relevant database roles according to your object-auditing design.

For example, the application/security model can be structured around:

```text
audit_sensitive
       │
       ├── payment
       ├── customer
       └── employee
```

Test carefully in non-production before applying this model.

---

# 28. `log_parameter` decision

Default:

```text
pgaudit.log_parameter = off
```

Only enable when there is a documented requirement.

If enabled:

```text
pgaudit.log_parameter = on
```

consider:

```text
pgaudit.log_parameter_max_size
```

Microsoft documents this parameter as a mechanism to replace oversized parameter values with a placeholder. ([Microsoft Learn][7])

Even with a maximum size, **do not assume the logging destination is safe for sensitive data**.

---

# 29. Entra ID implementation

If users authenticate through Microsoft Entra groups:

```text
User A ─┐
User B ─┼──► Entra Group ──► PostgreSQL
User C ─┘
```

PostgreSQL may see the group name as the database username.

Microsoft specifically documents this audit-correlation problem. ([Microsoft Learn][8])

Configure an appropriate `log_line_prefix`, for example the Microsoft-documented pattern:

```text
%t-%c-user=%u,db=%d,session=%c,sess_time=%s
```

Then correlate:

```text
Entra authentication
       +
PostgreSQL session
       +
pgaudit event
```

This is particularly important for forensic investigations involving group-based authentication.

---

# 30. Entra correlation KQL

Start with:

```kusto
PGSQLServerLogs
| where LogicalServerName =~ "<SERVER_NAME>"
| where TimeGenerated > ago(1h)
| where Message contains "AUDIT:"
| project
    TimeGenerated,
    LogicalServerName,
    Message
| order by TimeGenerated desc
```

Then correlate the session identifiers with the Microsoft Entra authentication data available in your Azure Monitor environment.

Microsoft provides a dedicated implementation for this scenario rather than relying only on the PostgreSQL username. ([Microsoft Learn][8])

---

# 31. Do not confuse these three audit layers

This is an important enterprise architecture distinction.

### Layer 1 — Azure Activity Log

```text
Azure resource operations
```

Examples:

```text
Server created
Server deleted
Configuration changes
Scaling
RBAC
Networking
```

### Layer 2 — PostgreSQL/pgaudit

```text
Database operations
```

Examples:

```text
CREATE
ALTER
DROP
INSERT
UPDATE
DELETE
SELECT
GRANT
REVOKE
```

### Layer 3 — Microsoft Entra

```text
Identity/authentication events
```

These three complement each other. pgaudit isn't a replacement for Azure Activity Log or Entra auditing. Microsoft explicitly distinguishes Azure resource-level logging from database activity auditing. ([Microsoft Learn][4])

---

# 32. Azure Policy

For an estate with dozens/hundreds of PGFS servers, create governance rather than relying on manual DBA checks.

Microsoft provides the built-in policy:

> **Auditing with PgAudit should be enabled for PostgreSQL flexible server instances**

with `AuditIfNotExists`. ([Microsoft Learn][9])

Your governance model becomes:

```text
Azure Policy
      │
      ▼
PGFS inventory
      │
      ├── pgaudit enabled
      └── pgaudit missing
              │
              ▼
          Compliance
```

You should also govern Diagnostic Settings so that enabling pgaudit without exporting logs doesn't create a false sense of compliance.

---

# 33. Performance test

Before production:

### Capture baseline

```text
CPU
Memory
IOPS
Storage throughput
TPS
P95 latency
P99 latency
connections
log volume
```

Then enable:

```text
DDL,ROLE,WRITE
```

Repeat the workload.

Then, if required, test:

```text
DDL,ROLE,READ,WRITE
```

Repeat.

Compare:

```text
                     Before       After
CPU                  _____        _____
IOPS                 _____        _____
TPS                  _____        _____
P95                  _____        _____
P99                  _____        _____
Audit events/day     _____        _____
Log volume           _____        _____
```

This is much more defensible than claiming a fixed percentage performance impact.

---

# 34. HA/failover test

Execute:

```text
Primary
   │
   ├── pgaudit enabled
   ├── Diagnostic Settings
   └── Log Analytics
          │
          ▼
       Failover
          │
          ▼
New primary
          │
          ├── pgaudit
          ├── logging
          └── audit events
```

After failover:

```sql
SHOW shared_preload_libraries;

SHOW pgaudit.log;

SELECT extname, extversion
FROM pg_extension
WHERE extname = 'pgaudit';
```

Then execute:

```sql
CREATE TABLE pgaudit_failover_test(id int);
DROP TABLE pgaudit_failover_test;
```

Verify those events in Log Analytics.

---

# 35. Disaster recovery test

Your DR runbook should contain:

```text
Provision/restore PGFS
        ↓
Configure azure.extensions
        ↓
Configure shared_preload_libraries
        ↓
CREATE EXTENSION
        ↓
Configure pgaudit
        ↓
Configure Diagnostic Settings
        ↓
Verify Log Analytics
        ↓
Generate test audit event
```

Don't assume a database restore reproduces every Azure server-level monitoring configuration.

---

# 36. Major-version upgrade

This deserves a mandatory checklist item.

Microsoft states that during a major PostgreSQL upgrade, Azure automatically drops and recreates the pgaudit extension, but **custom pgaudit configuration isn't automatically preserved**. ([Microsoft Learn][4])

Therefore:

```text
Before upgrade
      ↓
Export pgaudit configuration
      ↓
Upgrade
      ↓
Verify extension
      ↓
Reapply configuration
      ↓
Verify Diagnostic Settings
      ↓
Generate audit event
      ↓
Verify Log Analytics
```

Capture:

```sql
SELECT name,
       setting,
       source,
       pending_restart
FROM pg_settings
WHERE name LIKE 'pgaudit%'
ORDER BY name;
```

before the upgrade.

After upgrade, run exactly the same query and compare.

---

# 37. Upgrade validation script

```sql
DO $$
BEGIN
    IF NOT EXISTS
    (
        SELECT 1
        FROM pg_extension
        WHERE extname = 'pgaudit'
    )
    THEN
        RAISE EXCEPTION 'pgaudit extension is NOT installed';
    END IF;
END
$$;

SELECT extname,
       extversion
FROM pg_extension
WHERE extname = 'pgaudit';

SELECT name,
       setting,
       source,
       pending_restart
FROM pg_settings
WHERE name LIKE 'pgaudit%'
ORDER BY name;
```

Then manually verify Azure Diagnostic Settings.

---

# 38. Rollback

If the audit volume is unexpectedly high:

### First reduce scope

```text
DDL,ROLE,READ,WRITE
```

to:

```text
DDL,ROLE,WRITE
```

Do not immediately remove pgaudit.

If pgaudit itself must be disabled:

```bash
az postgres flexible-server parameter set \
  --resource-group "$RESOURCE_GROUP" \
  --server-name "$SERVER_NAME" \
  --name pgaudit.log \
  --source user-override \
  --value "none"
```

Then follow your approved restart/configuration process if required.

---

# 39. Audit retention

Don't confuse:

```text
PostgreSQL server-log capture
```

with:

```text
Log Analytics retention
```

Azure's downloadable server-log capture has a configurable **1–7 day** retention window. That feature is useful for troubleshooting, but it should not be treated as your enterprise audit archive. ([Microsoft Learn][10])

For compliance evidence, design retention in:

```text
Log Analytics
       +
Storage/Event Hub/SIEM
       +
Organization retention policy
```

according to your requirements.

---

# 40. Recommended production control matrix

| Control                  | Baseline                                    |
| ------------------------ | ------------------------------------------- |
| pgaudit allowlisted      | **Required**                                |
| pgaudit installed        | **Required**                                |
| shared_preload_libraries | **Required**                                |
| DDL auditing             | **Recommended**                             |
| ROLE auditing            | **Recommended**                             |
| WRITE auditing           | **Recommended**                             |
| READ auditing            | Requirement-dependent                       |
| Object auditing          | Sensitive-data use cases                    |
| log_catalog              | OFF                                         |
| log_client               | OFF                                         |
| log_parameter            | OFF                                         |
| log_relation             | OFF                                         |
| log_rows                 | OFF                                         |
| log_statement            | ON / requirement-dependent                  |
| log_statement_once       | ON                                          |
| Log Analytics            | **Recommended**                             |
| Resource-specific tables | **Recommended**                             |
| Azure Policy             | Enterprise recommended                      |
| Entra correlation        | Required where group authentication is used |
| SIEM                     | Enterprise/security requirement             |
| HA validation            | **Required**                                |
| Upgrade validation       | **Required**                                |
| DR validation            | **Required**                                |

---

# 41. Final production acceptance test

We should not mark the implementation complete until all of these pass:

```text
[01] pgaudit extension installed
[02] pgaudit version documented
[03] shared_preload_libraries validated
[04] pgaudit.log validated
[05] ROLE auditing tested
[06] DDL auditing tested
[07] INSERT tested
[08] UPDATE tested
[09] DELETE tested
[10] TRUNCATE tested
[11] Audit event appears in PGSQLServerLogs
[12] Log Analytics ingestion verified
[13] KQL queries validated
[14] Destructive-operation alert tested
[15] Privileged-role monitoring tested
[16] Parameter/PII exposure reviewed
[17] Entra correlation tested where applicable
[18] Audit volume measured
[19] Performance baseline compared
[20] HA/failover tested
[21] DR procedure tested
[22] Major-version upgrade procedure documented
[23] Azure Policy compliance verified
[24] Retention verified
[25] SIEM integration tested where required
```

## The key design principle

The production design should **not** be:

```text
pgaudit = ALL
```

It should be:

```text
                    Audit Requirement
                           │
                           ▼
                    Required Events
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
       Session audit                Object audit
             │                           │
             ▼                           ▼
       DDL/ROLE/WRITE             Sensitive objects
             │
             ▼
       READ if required
             │
             ▼
      Azure Monitor
             │
             ▼
       Log Analytics
             │
        ┌────┴────┐
        ▼         ▼
      Alert      SIEM
```

That gives you a **controlled audit trail rather than simply generating a huge PostgreSQL log stream**.

### Current Microsoft references

* [Microsoft — Audit logging in Azure PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit)
* [Microsoft — Allow extensions in PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/extensions/how-to-allow-extensions)
* [Microsoft — Create extensions in PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/extensions/how-to-create-extensions)
* [Microsoft — Configure and access PGFS logs](https://learn.microsoft.com/en-us/azure/postgresql/monitor/how-to-configure-and-access-logs)
* [Microsoft — PGSQLServerLogs Azure Monitor table](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/pgsqlserverlogs)
* [Microsoft — Entra ID principal audit logging](https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit-entra)
* [Microsoft — Azure Policy for PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/security/security-azure-policy)
* [Microsoft — PGFS server parameters](https://learn.microsoft.com/en-us/azure/postgresql/server-parameters/concepts-server-parameters)

This is the baseline we need to use for an **enterprise DBA/security change record and implementation runbook**.

[1]: https://learn.microsoft.com/en-us/azure/postgresql/extensions/how-to-allow-extensions "Allow Extensions in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/azure/postgresql/monitor/how-to-configure-and-access-logs "Configure and Access Logs in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/azure/postgresql/extensions/how-to-create-extensions "Create Extensions in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[4]: https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit "Audit Logging in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/pgsqlserverlogs "Azure Monitor Logs reference - PGSQLServerLogs - Azure Monitor | Microsoft Learn"
[6]: https://learn.microsoft.com/en-us/azure/postgresql/monitor/concepts-monitoring "Monitor Using Metrics and Logs in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[7]: https://learn.microsoft.com/en-us/azure/postgresql/server-parameters/concepts-server-parameters "Parameters in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[8]: https://learn.microsoft.com/en-us/azure/postgresql/security/security-audit-entra "Audit Logging for Microsoft Entra ID Principals in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[9]: https://learn.microsoft.com/en-us/azure/postgresql/security/security-azure-policy "Azure Policy Support in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"
[10]: https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-configure-server-logs "Download PostgreSQL and Upgrade Logs in Azure Database for PostgreSQL Flexible Server - Azure Database for PostgreSQL | Microsoft Learn"

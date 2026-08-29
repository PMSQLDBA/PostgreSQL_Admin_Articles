# PostgreSQL 17 Installation on Ubuntu 26.04

## Step 1: Verify Ubuntu version

```bash
cat /etc/os-release
```

You should see something similar to:

```text
NAME="Ubuntu"
VERSION="26.04 LTS (Resolute Raccoon)"
VERSION_CODENAME=resolute
```

Ubuntu 26.04 LTS uses the codename **resolute**. ([Ubuntu Lists][2])

Check architecture:

```bash
uname -m
```

Typical Intel/AMD server output:

```text
x86_64
```

You can also check the Debian/Ubuntu architecture name:

```bash
dpkg --print-architecture
```

Expected:

```text
amd64
```

---

# Step 2: Update Ubuntu package information

```bash
sudo apt update
```

Optionally upgrade existing OS packages:

```bash
sudo apt upgrade -y
```

The Ubuntu equivalent of Oracle Linux `dnf` is mainly `apt`.

| Oracle Linux         | Ubuntu                       |
| -------------------- | ---------------------------- |
| `dnf install`        | `apt install`                |
| `.rpm`               | `.deb`                       |
| `dnf update`         | `apt update` / `apt upgrade` |
| `/var/lib/pgsql/...` | `/var/lib/postgresql/...`    |

Your original notes use `dnf` and RPM packages because they target Oracle Linux. 

---

# Step 3: Install prerequisites

Install PostgreSQL's repository-management package and certificates:

```bash
sudo apt install -y postgresql-common ca-certificates
```

Verify:

```bash
dpkg -l | grep postgresql-common
```

---

# Step 4: Add the official PostgreSQL PGDG repository

The easiest official method is:

```bash
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

The script detects:

```text
resolute
```

and configures the PGDG repository for Ubuntu 26.04.

This is currently the installation method documented by PostgreSQL for Ubuntu. ([PostgreSQL][1])

During execution, you may see the PostgreSQL repository being configured as:

```text
https://apt.postgresql.org/pub/repos/apt
resolute-pgdg
```

---

# Step 5: Verify the PGDG repository

Run:

```bash
grep -R "apt.postgresql.org" /etc/apt/sources.list.d/
```

You should see the PGDG repository.

You can also run:

```bash
apt-cache policy postgresql-17
```

You should see a candidate version coming from:

```text
apt.postgresql.org
```

The PGDG repository supports PostgreSQL **13, 14, 15, 16, 17, 18 and newer development versions** on Ubuntu 26.04 at present. ([PostgreSQL Wiki][3])

---

# Step 6: Update package information again

After adding the repository:

```bash
sudo apt update
```

This is important because Ubuntu now needs to download the package metadata from PGDG.

---

# Step 7: Install PostgreSQL 17

Install PostgreSQL 17 server:

```bash
sudo apt install -y postgresql-17
```

I also recommend installing the client and contrib package:

```bash
sudo apt install -y postgresql-client-17 postgresql-contrib
```

Or together:

```bash
sudo apt install -y \
postgresql-17 \
postgresql-client-17 \
postgresql-contrib
```

PostgreSQL currently publishes PostgreSQL 17 packages specifically built for Ubuntu 26.04. ([PostgreSQL Apt][4])

### Package purposes

| Package                    | Purpose                               |
| -------------------------- | ------------------------------------- |
| `postgresql-17`            | PostgreSQL 17 database server         |
| `postgresql-client-17`     | `psql`, `pg_dump`, `pg_restore`, etc. |
| `postgresql-contrib`       | Additional PostgreSQL extensions      |
| `postgresql-server-dev-17` | Development headers                   |

---

# Step 8: Verify PostgreSQL installation

Run:

```bash
psql --version
```

Example:

```text
psql (PostgreSQL) 17.x
```

Also:

```bash
/usr/lib/postgresql/17/bin/postgres --version
```

Example:

```text
postgres (PostgreSQL) 17.x
```

You can check the installed packages:

```bash
dpkg -l | grep postgresql
```

---

# Step 9: Understand the Ubuntu initialization difference

This is one of the most important differences from your Oracle Linux notes.

On Oracle Linux your notes explicitly run:

```bash
/usr/pgsql-17/bin/postgresql-17-setup initdb
```

which creates:

```text
/var/lib/pgsql/17/data
```



### On Ubuntu you normally DO NOT run `initdb` manually.

The package installation normally creates a default PostgreSQL cluster automatically:

```text
17/main
```

Check it:

```bash
pg_lsclusters
```

Example:

```text
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

This command is very useful on Ubuntu:

```bash
pg_lsclusters
```

---

# Step 10: Check PostgreSQL service

Run:

```bash
sudo systemctl status postgresql
```

You should see:

```text
active (exited)
```

Don't be concerned by `active (exited)` on the top-level `postgresql.service`; Ubuntu uses it as a service managing PostgreSQL clusters.

Check the actual PostgreSQL 17 cluster:

```bash
sudo systemctl status postgresql@17-main
```

You should see:

```text
Active: active (running)
```

Another easy check:

```bash
pg_lsclusters
```

Expected:

```text
17  main  5432  online
```

---

# Step 11: Enable PostgreSQL at boot

Usually installation already enables it, but verify:

```bash
sudo systemctl enable postgresql
```

Check:

```bash
systemctl is-enabled postgresql
```

Expected:

```text
enabled
```

---

# Step 12: PostgreSQL service commands

### Start

```bash
sudo systemctl start postgresql
```

or specifically PostgreSQL 17:

```bash
sudo systemctl start postgresql@17-main
```

### Stop

```bash
sudo systemctl stop postgresql@17-main
```

### Restart

```bash
sudo systemctl restart postgresql@17-main
```

### Reload configuration

```bash
sudo systemctl reload postgresql@17-main
```

### Status

```bash
sudo systemctl status postgresql@17-main
```

These are the Ubuntu equivalents of the `systemctl start/stop/status postgresql-17` commands in your Oracle Linux notes. 

---

# Step 13: Check PostgreSQL processes

Run:

```bash
ps -ef | grep postgres
```

You should see processes similar to:

```text
postgres ... /usr/lib/postgresql/17/bin/postgres
postgres ... postgres: 17/main: checkpointer
postgres ... postgres: 17/main: background writer
postgres ... postgres: 17/main: walwriter
postgres ... postgres: 17/main: autovacuum launcher
postgres ... postgres: 17/main: logical replication launcher
```

Your Oracle Linux installation shows the same PostgreSQL background-process architecture. 

A better command is:

```bash
pgrep -a postgres
```

---

# Step 14: Switch to the postgres OS user

Ubuntu automatically creates the operating-system account:

```text
postgres
```

Switch to it:

```bash
sudo su - postgres
```

Your prompt should change to something similar to:

```text
postgres@server1:~$
```

Check:

```bash
whoami
```

Expected:

```text
postgres
```

---

# Step 15: Connect to PostgreSQL

As the `postgres` OS user:

```bash
psql
```

You should get:

```text
psql (17.x)
Type "help" for help.

postgres=#
```

Your attached notes use the same `sudo su - postgres` → `psql` workflow. 

---

# Step 16: Check PostgreSQL version

Inside `psql`:

```sql
SELECT version();
```

You can also check:

```sql
SHOW server_version;
```

Example:

```text
17.x
```

---

# Step 17: Check current database and user

```sql
SELECT current_database();
```

Expected:

```text
postgres
```

Check user:

```sql
SELECT current_user;
```

Expected:

```text
postgres
```

---

# Step 18: List databases

Inside `psql`:

```text
\l
```

or:

```text
\l+
```

Initially you'll normally see:

```text
postgres
template0
template1
```

This is the same database layout shown in your existing notes. 

---

# Step 19: List PostgreSQL users/roles

```text
\du
```

You should see:

```text
postgres
```

with superuser privileges.

---

# Step 20: List schemas

```text
\dn
```

Typically:

```text
public
```

---

# Step 21: Important PostgreSQL 17 locations on Ubuntu

This is another major difference from Oracle Linux.

| Item                    | Ubuntu 26.04 / PostgreSQL 17                 |
| ----------------------- | -------------------------------------------- |
| PostgreSQL binaries     | `/usr/lib/postgresql/17/bin`                 |
| Data directory          | `/var/lib/postgresql/17/main`                |
| Configuration directory | `/etc/postgresql/17/main`                    |
| `postgresql.conf`       | `/etc/postgresql/17/main/postgresql.conf`    |
| `pg_hba.conf`           | `/etc/postgresql/17/main/pg_hba.conf`        |
| `pg_ident.conf`         | `/etc/postgresql/17/main/pg_ident.conf`      |
| Log directory           | `/var/log/postgresql`                        |
| Main log                | `/var/log/postgresql/postgresql-17-main.log` |
| Port                    | `5432`                                       |
| OS user                 | `postgres`                                   |

This differs from your Oracle Linux layout, where the notes use `/usr/pgsql-17` and `/var/lib/pgsql/17/data`. 

---

# Step 22: Confirm locations directly from PostgreSQL

Connect:

```bash
sudo -u postgres psql
```

Run:

```sql
SHOW data_directory;
```

Expected:

```text
/var/lib/postgresql/17/main
```

Then:

```sql
SHOW config_file;
```

Expected:

```text
/etc/postgresql/17/main/postgresql.conf
```

Then:

```sql
SHOW hba_file;
```

Expected:

```text
/etc/postgresql/17/main/pg_hba.conf
```

Check port:

```sql
SHOW port;
```

Expected:

```text
5432
```

Exit:

```text
\q
```

Your original document uses these same SQL commands to determine the real configuration locations rather than relying only on assumptions. 

---

# Step 23: Check PostgreSQL log

Run:

```bash
sudo tail -100 /var/log/postgresql/postgresql-17-main.log
```

Follow the log live:

```bash
sudo tail -f /var/log/postgresql/postgresql-17-main.log
```

Or use systemd:

```bash
sudo journalctl -u postgresql@17-main
```

Recent messages:

```bash
sudo journalctl -u postgresql@17-main -n 100
```

---

# Step 24: Verify PostgreSQL is listening on port 5432

Run:

```bash
sudo ss -lntp | grep 5432
```

You may initially see PostgreSQL listening only locally:

```text
127.0.0.1:5432
```

and/or:

```text
[::1]:5432
```

Check from SQL:

```bash
sudo -u postgres psql -c "SHOW port;"
```

Expected:

```text
5432
```

Your attached notes also identify PostgreSQL's default port as `5432`. 

---

# Step 25: Create a test database

Connect:

```bash
sudo -u postgres psql
```

Create database:

```sql
CREATE DATABASE testdb;
```

Check:

```text
\l
```

Connect:

```text
\c testdb
```

---

# Step 26: Create a test table

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(100)
);
```

Insert a record:

```sql
INSERT INTO employees (name, department)
VALUES ('Sitaram', 'PostgreSQL DBA');
```

Check:

```sql
SELECT * FROM employees;
```

Expected:

```text
 id |  name   |   department
----+---------+----------------
  1 | Sitaram | PostgreSQL DBA
```

This follows the same final validation test in your original document. 

---

# Step 27: Exit PostgreSQL

```text
\q
```

Exit the `postgres` Linux account:

```bash
exit
```

---

# Step 28: One-command health check

These commands are useful after installation:

```bash
pg_lsclusters
```

```bash
sudo systemctl status postgresql@17-main
```

```bash
sudo -u postgres psql -c "SELECT version();"
```

```bash
sudo -u postgres psql -c "SHOW data_directory;"
```

```bash
sudo -u postgres psql -c "SHOW config_file;"
```

```bash
sudo -u postgres psql -c "SHOW hba_file;"
```

```bash
sudo ss -lntp | grep 5432
```

If all of these work, your PostgreSQL installation is healthy.

---

## Complete command flow

For a fresh **Ubuntu 26.04 + PostgreSQL 17** server, the core installation is essentially:

```bash
# Verify OS
cat /etc/os-release
uname -m

# Update package metadata
sudo apt update

# Install repository prerequisites
sudo apt install -y postgresql-common ca-certificates

# Configure official PostgreSQL PGDG repository
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh

# Refresh repositories
sudo apt update

# Verify PostgreSQL 17 is available
apt-cache policy postgresql-17

# Install PostgreSQL 17
sudo apt install -y \
postgresql-17 \
postgresql-client-17 \
postgresql-contrib

# Check cluster
pg_lsclusters

# Check service
sudo systemctl status postgresql@17-main

# Check version
psql --version

# Check processes
ps -ef | grep postgres

# Connect
sudo -u postgres psql
```

Then inside PostgreSQL:

```sql
SELECT version();
SELECT current_database();
SELECT current_user;

SHOW data_directory;
SHOW config_file;
SHOW hba_file;
SHOW port;

\l
\du
\dn

CREATE DATABASE testdb;
\c testdb

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(100)
);

INSERT INTO employees (name, department)
VALUES ('Sitaram', 'PostgreSQL DBA');

SELECT * FROM employees;

\q
```

### The key Oracle Linux → Ubuntu differences

Your Oracle Linux procedure is essentially:

```text
dnf
↓
install PGDG RPM
↓
postgresql17-server
↓
postgresql-17-setup initdb
↓
/var/lib/pgsql/17/data
↓
postgresql-17.service
```

On Ubuntu 26.04 it becomes:

```text
apt
↓
configure PGDG APT repository
↓
postgresql-17
↓
cluster automatically created
↓
/var/lib/postgresql/17/main
↓
postgresql@17-main.service
```

So **do not run the Oracle Linux `dnf`, `.rpm`, `/usr/pgsql-17/...`, or `postgresql-17-setup initdb` commands on Ubuntu**.

The official PostgreSQL documentation currently lists Ubuntu **26.04 `resolute`** as supported by PGDG, so using the official repository for a fixed major version such as PostgreSQL 17 is appropriate here. ([PostgreSQL][1])

[1]: https://www.postgresql.org/download/linux/ubuntu/?utm_source=chatgpt.com "PostgreSQL: Linux downloads (Ubuntu)"
[2]: https://lists.ubuntu.com/archives/ubuntu-announce/2026-April/000323.html?utm_source=chatgpt.com "Ubuntu 26.04 (\"Resolute Raccoon\") LTS released"
[3]: https://wiki.postgresql.org/wiki/Apt?utm_source=chatgpt.com "Apt - PostgreSQL wiki"
[4]: https://apt.postgresql.org/pub/repos/apt/pool/main/p/postgresql-17/?utm_source=chatgpt.com "Index of /pub/repos/apt/pool/main/p/postgresql-17/"

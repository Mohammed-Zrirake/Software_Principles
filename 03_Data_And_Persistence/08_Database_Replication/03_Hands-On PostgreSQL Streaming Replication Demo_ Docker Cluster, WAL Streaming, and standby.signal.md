#Database #PostgreSQL #Replication #StreamingReplication #Docker #WAL #HandsOn #SystemDesign #DevOps #Tutorial

> [!abstract] Executive Summary
> In PostgreSQL, **Streaming Replication** operates at the physical block level by continuously transmitting Write-Ahead Log (WAL) records from a primary instance to one or more standby replicas over a dedicated TCP stream.
> 
> Starting in **PostgreSQL 12 and 13+**, the classic replication configuration underwent a major architectural overhaul: the legacy `recovery.conf` file was permanently deprecated in favor of the **`standby.signal` trigger file** and native `postgresql.conf` parameters (`primary_conninfo`).
> 
> This hands-on laboratory builds a complete, working PostgreSQL 13+ synchronous streaming replication cluster from scratch using Docker, demystifies `pg_hba.conf` and `synchronous_standby_names`, verifies health via `pg_stat_replication`, tests DDL/DML propagation, and proves read-only constraint enforcement on standby nodes.

---

## 1. Laboratory Architecture & Container Blueprint

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                DOCKER CLUSTER TOPOLOGY                                          │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘

          Host Machine (Localhost)
                     │
         ┌───────────┴───────────┐
         │                       │
   Port: 5432              Port: 5433
         ▼                       ▼
 ┌───────────────┐       ┌───────────────┐
 │   p_master    │       │   p_standby   │
 │ (Postgres 13) │       │ (Postgres 13) │
 └───────┬───────┘       └───────┬───────┘
         │                       │
         │   Persistent Volume   │   Persistent Volume
         ▼                       ▼
 [ ./p_master_data ]     [ ./p_standby_data ]
```

- **`p_master` (Primary):** Exposes port `5432`. Accepts reads, writes, and schema mutations.
- **`p_standby` (Standby Replica):** Exposes port `5433`. Operates in Hot Standby mode (read-only), streaming WAL from `p_master`.
- **Application Name:** `standby1` (identifies the replica in the master's synchronous quorum).

---

## 2. Step 1: Spin Up the Primary (`p_master`)

Open a terminal and create a dedicated workspace directory:

```bash
mkdir pg_replication && cd pg_replication
```

Launch the master PostgreSQL container with an attached local volume mount:

```bash
docker run -d \
  --name p_master \
  -p 5432:5432 \
  -v $(pwd)/p_master_data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=postgres \
  postgres:13
```

Check logs to verify the database cluster initialized cleanly:
```bash
docker logs p_master
```
*Output: `database system is ready to accept connections`.*

---

## 3. Step 2: Spin Up and Initialize the Standby (`p_standby`)

### The Golden Invariant of Replication Initialization
A standby replica **cannot start from an empty database**. It must begin with an **exact, byte-for-byte identical base copy** of the master's data directory at a known WAL checkpoint.

#### A. Start Standby Container to Allocate Environment:
```bash
docker run -d \
  --name p_standby \
  -p 5433:5432 \
  -v $(pwd)/p_standby_data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=postgres \
  postgres:13
```

#### B. Stop Both Containers for Data Cloning:
```bash
docker stop p_master p_standby
```

#### C. Clone Master Data to Standby Directory:
```bash
# Backup/clear initial standby data
mv p_standby_data p_standby_data_orig

# Copy master data directory recursively to standby volume
cp -R p_master_data p_standby_data
```

> [!NOTE] Production vs. Lab Initialization
> In this local lab, copying files while containers are stopped is the fastest demonstration technique. In production, copying a running database will corrupt data. Production clusters use **`pg_basebackup`**, which streams a consistent snapshot while the primary remains live online:
> ```bash
> pg_basebackup -h master_host -p 5432 -U repuser -D /var/lib/postgresql/data -Fp -Xs -R
> ```

---

## 4. Step 3: Configure Master Permissions (`pg_hba.conf`)

The primary node must grant permission for incoming connections requesting the **`replication`** pseudo-database protocol.

Open `p_master_data/pg_hba.conf` in your editor and append this line at the bottom:

```text
# Allow replication connections from anywhere (or specify standby IP)
host    replication     postgres        all                     md5
```

- **`replication`**: Special keyword instructing PostgreSQL that this entry applies to WAL streaming rather than standard SQL database connections.
- **`postgres`**: The user authorized to stream WAL. *(In production, use a dedicated role: `CREATE ROLE repuser REPLICATION LOGIN ...`).*
- **`all`**: Allowed client IP CIDR mask (`all` permits Docker internal bridge network IPs).
- **`md5`**: Password authentication method (or `scram-sha-256`).

---

## 5. Step 4: Configure the Standby Node (`standby.signal` & `postgresql.conf`)

### The PostgreSQL 12+ Architectural Shift
- In PostgreSQL 11 and earlier, standby configuration lived in an external file called `recovery.conf`.
- In PostgreSQL 12, 13, and newer, `recovery.conf` was eliminated. Configuration is now split into:
  1. A trigger file named **`standby.signal`**.
  2. Standard configuration parameters in **`postgresql.conf`**.

#### A. Create the Standby Trigger File
In the `p_standby_data/` directory, create an empty file named `standby.signal`:

```bash
touch p_standby_data/standby.signal
```
*Why this is required:* The presence of this empty file instructs PostgreSQL at boot time: *"Do not initialize as a normal read-write master. Enter standby recovery mode, keep the database read-only, and look for an upstream primary to stream WAL from."*

#### B. Configure `primary_conninfo` in `p_standby_data/postgresql.conf`
Open `p_standby_data/postgresql.conf` and configure the upstream primary connection parameters:

```ini
# Upstream primary connection string
primary_conninfo = 'application_name=standby1 host=host.docker.internal port=5432 user=postgres password=postgres'
```

- **`application_name=standby1`**: Case-sensitive unique name. The master identifies this standby by this exact name.
- **`host=host.docker.internal`**: Resolves to the host machine running port `5432` *(on Linux, use the Docker gateway IP like `172.17.0.1` or container network alias)*.
- **`port=5432`**: The port where `p_master` is listening.
- **`user=postgres` / `password=postgres`**: Replication credentials configured in `pg_hba.conf`.

---

## 6. Step 5: Configure Synchronous Mode on Master

To enforce synchronous replication where the master waits for `standby1` before completing a `COMMIT`:

Open `p_master_data/postgresql.conf` and update:

```ini
synchronous_standby_names = 'FIRST 1 (standby1)'
```

- **`FIRST 1 (standby1)`**: Mandates that the master must receive confirmation from at least 1 synchronous standby named `standby1` before confirming a transaction commit to the client.

---

## 7. Step 6: Start Containers and Inspect Verification Logs

Start both instances:
```bash
docker start p_master p_standby
```

### Inspect Master Logs:
```bash
docker logs p_master
```
**Expected Output:**
```text
LOG:  standby "standby1" is now a synchronous standby with priority 1
LOG:  database system was not properly shut down; automatic recovery in progress
LOG:  database system is ready to accept connections
```

### Inspect Standby Logs:
```bash
docker logs p_standby
```
**Expected Output:**
```text
LOG:  entering standby mode
LOG:  consistent recovery state reached at 0/16B2540
LOG:  database system is ready to accept read only connections
LOG:  started streaming WAL from primary at 0/1700000 on timeline 1
```

*Both instances have connected! WAL streaming is active and healthy.*

---

## 8. Step 7: Querying `pg_stat_replication` Metrics

Connect to the master via `psql` to inspect replication telemetry:

```bash
docker exec -it p_master psql -U postgres
```

Execute the system replication monitoring query:
```sql
SELECT 
    application_name,
    client_addr,
    state,
    sync_state,
    sync_priority,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS sent_lag_bytes,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

**Output:**
```text
 application_name | client_addr |   state   | sync_state | sync_priority | sent_lag_bytes | replay_lag_bytes 
------------------+-------------+-----------+------------+---------------+----------------+------------------
 standby1         | 172.17.0.1  | streaming | sync       |             1 |              0 |                0
(1 row)
```

- **`state = streaming`**: WAL chunks are actively transmitted over TCP in real time.
- **`sync_state = sync`**: The replica is participating in the master's synchronous commit quorum.
- **`lag_bytes = 0`**: The replica is 100% caught up with zero byte lag.

---

## 9. Step 8: Hands-On Verification: DDL, DML & Read-Only Testing

Open two terminal windows side-by-side:
- **Terminal 1 (Left):** Master (`docker exec -it p_master psql -U postgres`)
- **Terminal 2 (Right):** Standby (`docker exec -it p_standby psql -U postgres -p 5432`)

### Test A: DDL Table Creation
```sql
-- In Terminal 1 (Master):
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50)
);

-- In Terminal 2 (Standby):
\d employees
```
*Result:* The schema mutation appears on the standby instantly!

### Test B: DML Data Insertion
```sql
-- In Terminal 1 (Master):
INSERT INTO employees (name, department) 
SELECT 'Employee_' || i, 'Engineering' 
FROM generate_series(1, 10000) AS i;

-- In Terminal 2 (Standby):
SELECT count(*) FROM employees;
```
*Result:* `count = 10000`. All ten thousand records are immediately queryable.

### Test C: Schema Alterations & Truncation
```sql
-- In Terminal 1 (Master):
ALTER TABLE employees ADD COLUMN salary INT DEFAULT 75000;
TRUNCATE TABLE employees;

-- In Terminal 2 (Standby):
\d employees
SELECT count(*) FROM employees;
```
*Result:* Column `salary` is added, and the table count drops to `0`.

### Test D: Enforcing Read-Only Protection on Standby
Attempt to write directly to the standby replica:

```sql
-- In Terminal 2 (Standby):
CREATE TABLE test_hacker (id INT);
```
**Output:**
```text
ERROR: cannot execute CREATE TABLE in a read-only transaction
```

```sql
-- In Terminal 2 (Standby):
INSERT INTO employees (name, department) VALUES ('Ghost', 'IT');
```
**Output:**
```text
ERROR: cannot execute INSERT in a read-only transaction
```
*Result:* The standby strictly rejects all mutations, preserving data integrity and preventing cluster state divergence.

---

## 10. Production Best Practices & Hardening Blueprint

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                           PRODUCTION HARDENING RECOMMENDATIONS                                  │
├───────────────────────────────┬─────────────────────────────────────────────────────────────────┤
│ Best Practice                 │ Production Command & Rule                                       │
├───────────────────────────────┼─────────────────────────────────────────────────────────────────┤
│ **Dedicated Replication User**│ Never use superuser `postgres`. Create least-privilege role:    │
│                               │ `CREATE ROLE repuser WITH REPLICATION LOGIN PASSWORD '...';`    │
├───────────────────────────────┼─────────────────────────────────────────────────────────────────┤
│ **Use `pg_basebackup`**       │ Avoid manual file copies:                                       │
│                               │ `pg_basebackup -h master -D /data -U repuser -v -P -X stream -R`│
├───────────────────────────────┼─────────────────────────────────────────────────────────────────┤
│ **Replication Slots**         │ Use replication slots so primary does not delete old WAL files  │
│                               │ before slow replicas have consumed them:                        │
│                               │ `SELECT pg_create_physical_replication_slot('standby1_slot');`  │
├───────────────────────────────┼─────────────────────────────────────────────────────────────────┤
│ **Monitor Lag Alerts**        │ Query `pg_stat_replication` every 10 seconds; trigger pager     │
│                               │ if `replay_lag` exceeds 10 seconds or 100MB.                    │
└───────────────────────────────┴─────────────────────────────────────────────────────────────────┘
```

---

## 11. Related Notes & Conceptual Navigation

- [[01_Database Replication Fundamentals_ Master-Standby vs. Multi-Master Architectures|Database Replication Fundamentals: Master-Standby vs. Multi-Master]]
- [[02_Synchronous vs. Asynchronous Replication_ Consistency, Quorums, and the Latency Trade-Off|Synchronous vs. Asynchronous Replication]]
- [[04_Database Replication Trade-Offs_ Pros, Cons, and Global Multi-Region Architecture|Database Replication Trade-Offs & Multi-Region Architecture]]
- [[02_Hands-On Distributed Sharding Demo_ Multi-PostgreSQL Cluster with Docker|Hands-On Distributed Sharding Demo with Docker]]
- [[04_Database Deadlocks_ Detection, Resolution, and PostgreSQL Hands-On Demo|Database Deadlocks Hands-On Demo]]
